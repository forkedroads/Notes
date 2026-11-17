# Baseten Router: Blog Notes

Source: https://www.baseten.co/blog/introducing-baseten-router

> **Confidence key.** Items marked **(post)** come from the blog text. Items marked **(inferred)** were reasoned out during my Q&A and are not stated in the post. Verify these before quoting them in the blog.

---

## 1. Core idea

Baseten Router (part of Baseten Grid, public preview) decides, per coding task, whether to send the task to an expensive frontier model (Opus 5.5) or a cheap open-weight model (GLM 5.3 Flash). The goal is near-frontier quality while paying frontier prices only when the task needs it.

**Decision rule (post):** for each candidate model `m` and task `x`, predict success probability `p̂_m(x)` and USD cost `ĉ_m(x)`, then pick the model with the higher score:

```
score_m(x) = p̂_m(x) − λ · ĉ_m(x)
```

`λ` is a dial. Higher `λ` favors the cheap model. Lower `λ` favors the accurate model. Sweeping `λ` gives the Cost / Balanced / Intelligence tiers and the whole Pareto frontier from one deployment.

**Headline results (post)**

| Metric | Result |
|---|---|
| Pass rate vs. always-Opus 5.5 | Within 0.8 points |
| Cost vs. always-Opus 5.5 | 72% lower |
| Beats random routing at the same cost? | Yes, statistically significant. The post says it is the only router evaluated (vs. OpenRouter, FireRouter, Together AI, Applied Compute) that does. |
| Router latency | p50 ~273 ms, p95 ~429 ms |
| Test set | Three held-out benchmarks: DeepSWE 1.1, SWE Atlas QnA, Terminal-Bench 2.1 |

---

## 2. The two types of noise

Benchmarking a router has two separate problems. Each noise type addresses one.

| | Label noise | Selection noise |
|---|---|---|
| Question it answers | How much can I trust each task's measured pass rate? | Could a random router have scored this well by luck? |
| Source of randomness | Models are non-deterministic. The same task passes on one run and fails on the next. | Which tasks happen to land on which model. |
| Fix | Repeat runs and average (k_O = 4 Opus, k_G = 8 GLM). | Compare against a band of 10,000 random routers. |
| Output | A trusted per-task pass-rate and cost table. | A gray band that a real router must clear. |

### 2.1 Label noise

**Setup.** Task `i` has a true (unobservable) pass-rate gap between the models:

```
δ_i = p_O,i − p_G,i
```

- **Signal `τ²`** = `Var_i(δ_i)`. This is how much the Opus-vs-GLM gap varies from task to task. If `τ²` is near zero, no task-specific pattern exists, so a router has nothing to learn.
- **Noise `σ²`** = run-to-run variance of a single pass/fail outcome, about `p(1−p)` (Bernoulli).

**How SNR is computed (mechanics)**

| Step | What you do | Formula |
|---|---|---|
| 1 | Run Opus `k_O` times and GLM `k_G` times per task. Compute empirical pass rates. | `p̂_O,i = (#Opus passes)/k_O`, `p̂_G,i = (#GLM passes)/k_G` |
| 2 | Estimate noise per task with the plug-in `p̂(1−p̂)`, then average over all N tasks. Tasks with p̂ = 0 or 1 contribute zero. Tasks near 50/50 contribute most (max 0.25). | `σ̂²_O = mean_i[p̂_O,i(1−p̂_O,i)]`, same for GLM |
| 3 | Compute the observed gap per task and its sample variance across tasks. This is signal and leftover noise mixed together. | `δ̂_i = p̂_O,i − p̂_G,i`, `Var(δ̂) = 1/(N−1) · Σ(δ̂_i − δ̄)²` |
| 4 | Subtract the known noise to isolate the signal (method of moments, as in meta-analysis). Clip to 0 if negative. | `τ̂² = Var(δ̂) − σ̂²_O/k_O − σ̂²_G/k_G` |
| 5 | Take the ratio. | `SNR = τ² / (σ²_O/k_O + σ²_G/k_G)` |

**Key point.** `τ²` is never observed directly. It is the total spread of measured gaps minus the portion explained by run-to-run noise. Averaging `k` runs shrinks the noise by `1/k` but leaves the signal unchanged.

*(inferred)* A caveat for the blog: `p̂(1−p̂)` computed from a small number of runs underestimates the true variance (it is biased low by a factor of `(k−1)/k`). The post may or may not correct for this.

**Reported numbers (post) and a reproduction check**

| Benchmark | τ² | σ²_O | σ²_G | SNR (k=1) |
|---|---|---|---|---|
| DeepSWE 1.1 | 0.068 | 0.097 | 0.137 | 0.29 |
| SWE Atlas QnA | 0.030 | 0.097 | 0.108 | 0.15 |
| Terminal-Bench 2.1 | 0.063 | 0.040 | 0.097 | 0.46 |
| **Pooled** | **0.054** | **0.082** | **0.116** | **0.27** |

Check: pooled `0.054 / (0.082 + 0.116) = 0.27`, which matches. With k_O = 4 and k_G = 8: `0.054 / (0.082/4 + 0.116/8) = 0.054 / 0.035 ≈ 1.5`, which matches the reported 1.51.

| Runs per task | SNR | Reading |
|---|---|---|
| k_O = 1, k_G = 1 | 0.27 | Noise is about 3.7x the signal. A single-run benchmark is mostly noise. |
| k_O = 4, k_G = 8 | 1.51 | Signal exceeds noise by about 50%. GLM gets more runs because it is cheaper. |

**What "Pooled" means.** The pooled row is not an average of the three SNRs. The `τ²` and `σ²` values are pooled first (as task-count-weighted averages across the merged test set), then divided. Ratios do not pool linearly, so the simple average of 0.29, 0.15 and 0.46 (about 0.30) does not equal 0.27.

- The 92.7% / 2.9% / 4.4% split in the post is the **training data mix** (software engineering / questions / terminal). It is **not** the weighting for this table, and using it as weights gives pooled τ² ≈ 0.067, which does not match the reported 0.054.
- *(inferred, unconfirmed)* Weighting by benchmark task counts reproduces the pooled row closely. The counts I used (113 / 124 / 89) came from my own recall, not from the post. Check them before quoting.

### 2.2 Selection noise

**The worry.** A router that is a coin flip could, by luck, send the easy tasks to GLM and the hard ones to Opus. Its measured pass rate would then look good even though no intelligence was involved. The test asks: *could a dice roll plausibly have produced a result this good at the same cost?*

**Experimental setup**

| Step | What happens |
|---|---|
| 1 | Pick a split `f` (fraction of tasks sent to GLM), for example 30%. |
| 2 | Randomly draw 30% of the N tasks (without replacement) to go to GLM. The rest go to Opus. |
| 3 | For each task, look up the pre-computed per-task pass rate and cost for whichever model it was assigned to. These are the `k_O = 4` / `k_G = 8` averages. No models are re-run. |
| 4 | Average over all tasks. This gives one (cost, pass rate) point, which is one gray dot. |
| 5 | Repeat 10,000 times at this split, with a different random subset each time. |
| 6 | Repeat across a grid of splits from 0% to 100%. |

**How the gray band is drawn (mechanics)**

| Element | How it is computed |
|---|---|
| Grid | Discretize the split axis (for example 0%, 5%, ..., 100%). |
| Percentiles | At each split, sort the 10,000 pass rates. The 5th percentile (rank ~500) is the lower edge. The 95th percentile (rank ~9,500) is the upper edge. |
| Cost axis | Mean simulated cost at that split, or `f·cost_GLM + (1−f)·cost_Opus`. |
| Curves | Connect the upper points into a top curve and the lower points into a bottom curve. |
| Fill | Shade between the two curves (the same technique as a Monte Carlo confidence ribbon). |
| Dotted line | The mean of the simulations. It is the straight line `f·pass_GLM + (1−f)·pass_Opus` between the two single-model endpoints. |

**Why the band is an ellipse.** At 0% and 100% there is nothing to randomize, so the band pinches to a point at each single-model endpoint. At about 50% there is the most freedom in which tasks go where, so the band is widest.

**Outcome (post).** A router point above the top edge of the band beats at least 95% of random routers at that cost. That is a p ≈ 0.05 style significance threshold. A point inside the band is indistinguishable from random routing. In the post's illustrative figure, "Router X" is above the band and "Router Y" is inside it. The text calls it an illustrative example, and I could not view the image, so I'm not certain whether X and Y are generic stand-ins or real routers.

### 2.3 How label noise feeds selection noise

- The simulations reuse the same per-task table built with `k_O = 4` and `k_G = 8` averaging. The post states that every test-set pass rate is an average over those runs.
- That table is **not** noise-free. SNR is 1.51, not infinite. There is no "denoised ground truth" anywhere in the pipeline.
- Every router (the 10,000 random ones, Baseten, and the competitors) is scored against the same table, so the comparison is fair even though absolute numbers carry residual noise.
- *(inferred)* The two noise types are not cleanly separable in the final chart. The band's width mixes real task-to-task variation with leftover label noise. I earlier claimed this makes the test conservative. That claim is plausible but I haven't verified it, so I'd leave it out of the blog.

---

## 3. How the two noise analyses validate the router

| Stage | Role in the argument | Concrete output |
|---|---|---|
| Label noise (SNR) | Justify how many repeats per task are needed to trust the measurements. | Use 4 Opus runs and 8 GLM runs per task. SNR goes from 0.27 to 1.51. |
| Per-task table | Shared input to everything downstream. | Averaged pass rate and cost for each task, for each model. |
| Selection noise (random-router simulation) | Provide a "what luck alone can do" reference. | The gray band at every cost level. |
| Router evaluation | Plot Baseten and competitors on the same cost vs. pass-rate axes. | Baseten's points sit above the band at every tested threshold. The others do not (per the post). |

One-line version: label noise tells you the measurements are trustworthy enough, and selection noise tells you the router's result is better than luck given those measurements.

---

## 4. Router architecture

The supervised learning problem stays fixed through all iterations. Only the input features change.

- **Inputs:** features describing the task.
- **Labels:** from training traces where both Opus 5.5 and GLM 5.3 Flash were run on every task at high reasoning effort. The labels are pass/fail from the verifier (for `p̂`) and observed USD cost (for `ĉ`).
- **Output:** `p̂_m(x)` and `ĉ_m(x)` for each model.
- *(inferred)* Likely losses: binary cross-entropy for `p̂` (calibration matters because the score does arithmetic on it) and a regression loss for `ĉ`. The post does not give these.

### 4.1 Embedding route

| | Vanilla version | Customization using attributes |
|---|---|---|
| Input features | Off-the-shelf embeddings of the raw task text (Qwen3-Embedding, EmbeddingGemma 2). | Structured attributes: task scope, domain knowledge required, ambiguity. |
| Where features come from | Frozen pretrained embedding models. Baseten does not train them. | Extracted by Baseten's own decision models, elephant-1 (fine-tuned from Qwen3.5-9B) and mammoth-1 (from Qwen3.5-27B). |
| Prediction model | Small supervised heads on top of the frozen embedding. | Same kind of heads, now on interpretable attributes. |
| Evaluation | Leave-one-benchmark-out cross-validation. Predicting cost and success jointly worked best. | Same. |
| Finding | Predicting cost alone captured most of the gain. Embeddings encode task scope (size and complexity) better than "which model will succeed". | Explicit, calibrated, interpretable features that avoid the shortcut below. |
| Problem | A linear probe predicts the source benchmark from the embedding with 89% accuracy, so the router may use "which benchmark is this" as a shortcut. Nearby tasks in embedding space also had only weakly correlated Opus advantage. | None stated as a blocker. |

**Why embeddings behave this way.** Embedding models are trained with a contrastive (InfoNCE) objective that pulls semantically similar text together. That teaches semantic and stylistic similarity, not "which tasks need the stronger model". Benchmark style is easy to pick up, which explains the leakage.

**How the decision models extract attributes.** They are multiple-choice answering models. The router asks questions such as "how broad is this task's scope?" and uses the probability distribution over the options as a feature. Details from the post:


### 4.2 Activations

**Idea.** Instead of guessing cost from task text, read the model's own internal state. Run GLM 5.3 Flash on the task as a forward pass only (no response generated), grab hidden activations from an intermediate layer, and train a small probe to predict how many agent steps GLM will take if it runs to completion. More steps means more tokens and tool calls, which means higher cost.

| Aspect | Detail |
|---|---|
| Training | Supervised regression. Input is the frozen activation vector. Label is the actual step count from running the task to completion (a free byproduct of trace generation). Only the small probe head is trained. GLM's weights are untouched. |
| Quality | Spearman correlation about 0.6 with actual step count on a validation set. Informative but noisy. Spearman suits routing because ranking tasks by cost is what matters. |
| Layer finding | Signal is weak in early layers, strongest in the middle layers, and declines in the deepest layers. |
| Latency | Quantization and smaller Qwen models standing in as proxies for GLM's internals. The post says routing quality was preserved but, per my reading, gives no number for accuracy lost. |
| Why useful | You learn how expensive a task will be without paying for the expensive generation. |

**Does this generalize to other models? (my questions answered)**

| Question | Answer |
|---|---|
| Is the activation head a general proxy for cost across any model? | No. It is trained to predict GLM's own step count, so it is model-specific. |
| Does it apply to Opus? | Not directly. Activations cannot be read from a closed API. *(inferred)* |
| How is Opus cost predicted then? | Not stated in the post. *(inferred)* Most plausibly from the model-agnostic structured attributes. |
| Is the router therefore asymmetric? | *(inferred)* Yes. `ĉ_GLM` likely gets extra information (attributes plus activations) while `ĉ_Opus` has attributes only. The post does not discuss whether this makes one estimate noisier. |
| Do the two costs stay comparable? | Yes, as long as both heads output USD. The score subtracts `λ·ĉ`, so they must share a unit. Predicted steps must be converted to dollars somewhere (for example as a feature into a final regression head). The post, as I read it, does not spell out this step. *(inferred)* |
| Does it transfer to other model pairs in Grid? | Not automatically. The post says the approach "can be extended", which in practice means new traces, new probe, new validation per open-weight model. |
| Is a smaller Qwen a faithful stand-in for GLM's activations? | An unquantified assumption in the post. |

**Final router (post).** Combines the structured attributes and the activation-derived signals into the `p̂` and `ĉ` predictions that feed the scoring rule.

---

