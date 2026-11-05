# Meta Andromeda: Chat Summary & Reference Notes

*Source: Meta Engineering blog, "Meta Andromeda: Powering the Next Generation of Ads Retrieval Through Model-System Co-Design", plus follow-up Q&A.*
https://engineering.fb.com/2026/10/28/ml-applications/meta-andromeda-next-gen-ads-retrieval-gpu-co-design/

**Where retrieval sits in the ads funnel:**

```
Eligible ads (100Ms) -> [Andromeda retrieval] -> shortlist (1000s) -> [Ranking] -> [Auction] -> ad shown
```

Retrieval optimizes for **recall** at very low cost per ad. Ranking uses much heavier models for **precision**. The auction then maximizes expected value (bid x predicted action rate x quality).

---

## Section 1: Prerequisites: Intro to Recommender Systems

### 1.1 Two-tower systems (query tower, candidate tower)

A **tower** is a real neural sub-network with its own parameters. It encodes one side of a (user, item) pair into a fixed-size **embedding** vector. The two towers never see each other's features.

| Tower | Input | Output |
|---|---|---|
| Query / user tower | User features (history, cohort, context) | User embedding (e.g. 128-d) |
| Candidate / item tower | Ad features (content, category, price) | Item embedding (e.g. 128-d) |

### 1.2 How they produce recommendations

1. Compute the item embeddings **offline** for the whole corpus and store them in an ANN index.
2. At request time, run the user tower **once** to get the user embedding.
3. Score each item with `score = user_emb · item_emb` (dot product or cosine).
4. Use nearest-neighbor lookup to return the top-K items.

Why it scales: the item side is precomputed, and the per-request work is one tower pass plus a cheap vector search.

---

## Section 2: Andromeda V1 (2024)

### 2.1 Problems with classical recommender systems

| Problem | Detail |
|---|---|
| **Scale** | Tens of millions of ads per request, in milliseconds |
| **Quality** | A dot product is low-capacity (see Appendix B). Ranking-grade models are too expensive to run at retrieval scale. |

### 2.2 The dense NN in V1

**Idea:** replace the dot product with an **Interaction Network**, a dense NN that takes both embeddings together.

**Why it is better:** nonlinear layers can learn cross-feature patterns (XOR-like, conditional, context-dependent). A dot product cannot represent these. Reported gains: about +6% recall and +8% ad quality.

**Inputs and outputs:**

```
raw user features -> [User Tower] -> user_embedding --+
                                                      +-> concat -> [Interaction NN] -> score in [0,1]
raw ad features   -> [Ad Tower]   -> ad_embedding   --+
```

- The **input** to the interaction NN is the two tower-output embeddings, not the raw features.
- The **output** is a scalar: the predicted probability of engagement.
- The user embedding is computed once per request. Ad and node embeddings are precomputed.

### 2.3 The new problem, and the hierarchical-tree trick

The interaction NN needs both inputs jointly, so item embeddings can't be precomputed into a plain ANN lookup. Running it on every ad is infeasible.

**Solution: a learned hierarchical index (tree) plus beam search.**

- Ads are grouped into a multi-level tree. Every node has a learned embedding.
- The **same interaction NN** scores the user against nodes at each level. The inputs are the user embedding and a node embedding, and the node can be a coarse cluster, a finer cluster, or a single ad.
- Beam search keeps the top-B nodes per level and descends into their children.
- Cost grows with about `branching x depth`, which is **sublinear** (about log N).
- Worked example: 10M ads, branching 10, beam 2, depth 7 gives about **130 NN calls**, versus 10M for brute force.
- The tree is **trained jointly** with the model (cross-attention plus residual quantization). Branches reflect the model's own notion of relevance, so good ads are less likely to be pruned early. A classical ANN index is built after training from frozen embeddings and has no such guarantee.

**Related V1 levers** (mentioned in the chat, with less detail):

- A stacked interaction network, with heavier models at the top of the tree and lighter ones near the leaves.
- "Model elasticity", which spends more compute on high-value ad segments.

### 2.4 End-to-end flow (training and inference split)

### PART 1: TRAINING (happens offline, on historical logged data)

**Step 1: Collect labeled data from logs**

- Logged impressions: `(user_features, ad_features) -> label` (1 = clicked/converted, 0 = didn't)
- Positives: real logged clicks/conversions
- Negatives: random ads + in-batch ads (sampled, since "no click" is not a reliable negative on its own)

**Step 2: Build raw feature vectors**

```
user_raw = concat([user_id_emb, click_history_emb, cohort_emb, age, recency, ...])
ad_raw   = concat([ad_id_emb, advertiser_emb, category_emb, price, discount, ...])
```

**Step 3: Pass through towers**

```
user_embedding = UserTower(user_raw)     # e.g. 128-dim vector
ad_embedding   = AdTower(ad_raw)         # e.g. 128-dim vector
```

**Step 4: Pass through Interaction Network**

```
joint = concat([user_embedding, ad_embedding])
score = InteractionNetwork(joint)        # dense NN, nonlinear layers -> scalar in [0,1]
```

**Step 5: Compute loss, backpropagate**

```
loss = BinaryCrossEntropy(score, label)
-> gradients flow back through: Interaction Network -> Ad Tower -> User Tower
-> all weights updated together, end-to-end
```

**Step 6: Jointly train the hierarchical index (same loop)**

- Ads are organized into a tree; each node gets its own learned embedding
- An additional loss term rewards the tree for not pruning away ads that are true positives, i.e., the path from root to leaf for a truly relevant ad should score well at every level
- Node embeddings (via cross-attention + residual quantization) are updated in the same backward pass, alongside tower and interaction network weights

**End result of training:** a User Tower, an Ad Tower, an Interaction Network, and a hierarchical tree of node embeddings, all mutually consistent because they were trained together, not stitched together afterward.

### PART 2: INFERENCE / SERVING (happens live, per user request, in milliseconds)

**Step 1: Precomputation (done ahead of time, not per-request)**

- Every ad in the corpus gets its `ad_embedding` computed once via the Ad Tower and stored
- The hierarchical tree (with all node embeddings) is built and stored, sitting in GPU local memory (GH200)

**Step 2: Compute the user embedding (once per request)**

```
user_embedding = UserTower(user_raw)   # one forward pass, reused for the whole request
```

**Step 3: Traverse the tree (this is the "narrowing down" step)**

Start at the root:

```
score(user, Node) = InteractionNetwork(concat([user_embedding, Node_embedding]))
```

- Score the top-level nodes (e.g., 10 of them)
- Keep the top-B scoring nodes (beam search, not just the single best, to avoid losing good candidates to one bad early decision)
- Descend into their children, repeat the same scoring operation
- Repeat level by level, down through the tree

At every level, you're reusing the exact same Interaction Network; only the second input (which node's embedding) changes.

**Step 4: Reach the leaves**

- End up with a small set of actual ads (from the tens of millions down to a few thousand)
- Total interaction-network calls: proportional to tree depth x branching factor, which is sublinear in the total number of ads, not linear

**Step 5: Hand off to Ranking**

- This small candidate set (thousands of ads) is passed to the much more expensive, fine-grained ranking model, which makes the final selection of what's actually shown to the user

### One-paragraph summary

- **Training:** logged clicks/impressions -> towers compress raw features into embeddings -> interaction network learns nonlinear relevance patterns between user and ad embeddings -> loss backpropagates through everything, including the tree's node embeddings, so model and index are trained jointly and stay mutually consistent.
- **Inference:** user embedding computed once -> interaction network is called repeatedly, but only against a tiny number of tree nodes per level (not every ad) -> beam search narrows top-level clusters down to a small set of actual ad candidates -> those get passed on to ranking.

*Note:* the interaction network is called many times per level (once per node scored), not once per level. Total calls stay tiny compared with the corpus size.

---

## Section 3: Andromeda V2 (2026)

### 3.1 Problems with V1

- Generative-AI ad creation exploded the candidate pool to hundreds of millions of ads.
- Quality still short of ranking-level.
- **One user vector is a bottleneck.** One embedding must compress heterogeneous signals (sequences, cohort stats, LLM semantics, cross-surface behavior) and serve every possible ad. This causes information loss and negative transfer between signal types.
- **Bottleneck shift at billion-scale throughput:** memory and network bandwidth, not FLOPs. Separate kernels and CPU/accelerator round-trips saturate memory and network links.

### 3.2 The MoE trick and scoring every ad

**Retrieval Mixture-of-Experts (MoE):**

| Component | V2 design |
|---|---|
| User side | **Ensemble of expert towers.** Examples: ad-engagement sequence, user cohort, LLM-semantic, cross-surface co-engagement. Each produces its own embedding, computed once per request. |
| Ad side | **Context-aware ad tower.** Static content is precomputed. Real-time serving context (placement, time, session) is fused in live. |
| Interaction | **Ad-aware router plus attention.** The router looks at the ad embedding and picks the top 1-2 experts. Attention between the ad embedding and only those experts yields the score. |

**Intuition for splitting the user into experts:**

1. **Specialists beat one generalist.** Each expert focuses on one signal type, which avoids negative transfer.
2. **Different ads need different views of the same user.** A tent ad routes to engagement-sequence and cross-surface experts. A credit-card ad routes to cohort and LLM-semantic experts.
3. **Sparsity.** Capacity grows with the number of experts, but compute only grows with the number activated (K). This is where the roughly **300x fewer inference FLOPs** comes from.

**Scoring every ad:** V2 does **not** prune with a tree. The router, attention and score run for **every eligible ad**, in parallel on the GPU. Narrowing happens **after** scoring, via top-K.

**Clarification (from the chat):** the ad embedding step is cheap because it is a single tower pass and uses no experts. The expert-sparsity saving applies to the *interaction* step.

### 3.3 End-to-end flow (training and inference split)

#### Training (offline)

> The blog does not detail V2 training. The following is inferred from the architecture and the general MoE recipe, so treat it as a reasonable reconstruction rather than a sourced description.

1. **Same supervision as V1:** logged clicks and conversions as positives, sampled negatives, binary cross-entropy.
2. **Forward pass:** raw user features go to *each* expert tower to give expert embeddings. Ad content plus context goes to the context-aware ad tower to give the ad embedding. The router scores the experts from the ad embedding, the top-K experts are selected, attention produces the score, and the loss is computed against the label.
3. **Backprop** updates all expert towers, the ad tower, the router and the attention layers jointly. MoE training usually adds a load-balancing term, so that routing doesn't collapse onto one expert. The chat did not cover this.
4. There is **no hierarchical-index training**, since the tree is not the narrowing mechanism in V2.

#### Inference (live, per request)

```
Request arrives (compact: user features + targeting query) -> single GPU server
  |
  v
1. INDEX MATCHING on GPU (hard targeting-rule filter) -> eligible ad pool
  |
  v
2. USER EXPERT EMBEDDINGS (once per request, reused for all ads)
  |
  v
3. PER-AD FUSED KERNEL (for EVERY eligible ad, in parallel; one GPU kernel, nothing leaves registers/shared memory):
     a) ad_embedding     = AdTower(static_content, live_context)
     b) router_scores    = Router(ad_embedding)
     c) selected_experts = top_K(router_scores)      # 1-2 of N
     d) score            = AttentionInteraction(ad_embedding, selected_experts)
  |
  v
4. TOP-K SELECTION over all scores (same GPU server)
  |
  v
Only the winners leave the server -> Ranking
```

Co-locating targeting, scoring and top-K on one server cuts network traffic by about 100x. Fusing the per-ad steps into one kernel avoids writing intermediate tensors to HBM.

Per-ad loop, in short:

```
FOR EACH eligible ad:
    ad_emb  = AdTower(static, context)
    experts = top_K(Router(ad_emb))
    score   = Attention(ad_emb, experts)
AFTER all ads scored: top_K_ads = TopK(scores)
```

---

## Section 4: V1 vs V2 Comparison

| Dimension | V1 (2024) | V2 (2026) |
|---|---|---|
| User side | 1 user tower | Ensemble of specialized expert towers |
| Ad embedding | Static, precomputed | Static content plus live context, fused at serving |
| Matching function | Dense interaction NN | MoE router plus attention over selected experts |
| How the pool is narrowed | **Before** scoring: tree pruning, most ads never touched | **After** scoring: top-K over all scored ads |
| Ads scored by the interaction model | Small subset (about 100s of calls) | **Every eligible ad** |
| Efficiency source | Fewer ads touched (sublinear search) | Fewer FLOPs per ad (MoE, about 300x), kernel fusion, GPU parallelism |
| Risk | Tree may prune a good ad early | Nothing skipped, only made cheaper |
| Bottleneck addressed | Compute vs. latency | Memory and network bandwidth |
| Index training | Joint hierarchical-index training | Not needed |
| Pipeline placement | CPU hosts plus remote accelerators | Targeting, scoring and top-K co-located on one GPU server |
| Philosophy | Smart pruning beats brute force | Brute force is viable once it is cheap enough |
| Reported results | +6% recall, +8% ad quality | 1B+ ads/sec per server (10B+ ads/sec model-only on one GH200), +2.4% Facebook conversions |

**Is Andromeda's top-K the same as ranking's selection? No.**

| | Retrieval top-K (Andromeda) | Ranking and auction |
|---|---|---|
| Goal | Recall: don't lose good ads | Precision, then value maximization |
| Objective | Generic relevance score | Bid x predicted action rate x quality |
| Model weight | Very light per ad | Very heavy per ad (up to roughly 1T params) |
| Input size | 100Ms -> 1000s | 1000s -> final ad |

---

## Additional Details

### A. Training the dense NN that replaces the dot product

Same recipe as an image classifier. The differences are the label source and the negatives.

| | Image classifier | Ads retrieval model |
|---|---|---|
| Input | Pixels | (user features, ad features) |
| Output | Class probabilities | Engagement probability in [0,1] |
| Labels | Human annotation | **Logged behavior** (click or convert = 1) |
| Loss | Cross-entropy | Binary cross-entropy |
| Mechanics | Forward, loss, backprop, update | Same |

**Negatives need construction:**

- **Random negatives:** random ads paired with the user.
- **In-batch negatives:** other examples' ads in the same batch (cheap and scalable).
- "Shown but not clicked" alone is biased toward the old system's choices.

**One training step:**

1. Sample a batch of positives and negatives.
2. Run the towers, then the interaction NN, to get scores.
3. Compute BCE loss.
4. Backprop through everything.
5. Update the weights (Adam or SGD).
6. Repeat continuously over billions of impressions.

### B. Limitations of the dot product (why quality is poor)

A dot product is a sum of **independent per-dimension products**:

```
score = u1*v1 + u2*v2 + ... + un*vn
```

- Dimension *i* of the user only interacts with dimension *i* of the ad.
- It cannot express conditional logic such as "this matters only when that is also true".
- It is a linear or bilinear form, so it cannot represent XOR-style patterns.

| User price-sensitive? | Ad discounted? | True relevance |
|---|---|---|
| 0 | 0 | Low |
| 0 | 1 | Low |
| 1 | 0 | Low |
| 1 | 1 | **High** |

- Relevance depends on the **combination**. A dot product treats the signals as separate additive contributions.
- The workaround is hand-engineered cross features, which don't scale to thousands of signals.
- A dense NN with a nonlinearity (ReLU, etc.) can learn the interaction automatically. A 2-layer MLP represents XOR exactly.

---

## Caveats

- Some specifics came from secondary write-ups or the assistant's reconstruction, not from the blog text directly. Examples: stacked MoNN levels, model-elasticity figures, and whether V2 keeps any pre-filtering beyond targeting rules.
- The V2 training section is inferred, as noted above.
- Treat the numbers (300x, 10B/sec, 100x less traffic, +2.4%) as reported figures from the blog and write-ups.
