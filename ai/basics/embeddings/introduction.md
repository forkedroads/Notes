# Embeddings, Chunking, Pooling, and Harrier

A practical and conceptual guide to modern text embeddings, chunking strategies, pooling methods, and a concrete Harrier case study.
Some parts of the notes were generated using llms but rooted in my discussion on the topics that were of interest to me or unclear to me.
Parts well known are glossed over and included just for reference. Emphasis is on new topics that I did not know much about.


## Embedding fundamentals

### Basics
- An **embedding** is a fixed-length numeric vector that represents the semantic meaning of text.
- The vector dimension is defined by the model architecture and does **not** depend on input length.

### End-to-end pipeline

```
Raw text
  → Tokenizer (text → token IDs)
  → Token embedding lookup (+ position info)
  → Transformer layers
       • Self-attention
       • MLP / Feed‑forward layers
  → Contextual token vectors (h1..hN)
  → Pooling
  → Final embedding vector
```

### Step explanations
- **Tokenizer**: Converts text into discrete tokens.
- **Token embeddings**: Map tokens to dense vectors.
- **Self-attention**: Allows tokens to exchange contextual information.
- **MLP layers**: Apply nonlinear transformations to build semantic features.
- **Pooling**: Reduces variable-length token vectors to one fixed vector.
- **Final embedding**: Used for similarity, clustering, and retrieval.

---

## Chunking

### Concept overview
- Chunking splits large documents into smaller units for embedding and retrieval.
- It determines retrieval granularity, recall behavior, and prompt efficiency.

### Types and details

#### Character-based chunking
- Split text by character count with optional overlap.
- Simple but results in variable token counts after tokenization.

**How can chunks have different token counts?**
Token boundaries do not align with characters; tokenization happens after splitting.

---

#### Token-based chunking
- Tokenize first, then split into fixed-size token windows.
- Ensures consistent token count per chunk.

**How is chunk size decided based on tokens?**
Chunk size depends on:
- Model max input length
- Desired semantic granularity
- Overlap strategy

---

#### Semantic chunking
- Split by paragraphs, sentences, or headings.
- Preserves coherence but produces variable-sized chunks.

---

#### Hybrid chunking
- Semantic-first splitting with token limits enforced.
- Common in production RAG systems.

---

### Implementation in practice

**Typical sizes**
- 200–400 tokens: high precision, many chunks
- 400–800 tokens: balanced default
- 2k–10k+ tokens: long-context embeddings

**Overlap**
- Small chunks often use 10–20% overlap
- Large chunks often need little or no overlap

**Two-stage retrieval**

```
Coarse retrieval (large chunks)
  → Optional refinement (smaller chunks)
  → Final context
```

---

## Pooling

### Why pooling exists
- Transformers output one vector per token.
- Retrieval systems require a single fixed-size vector per chunk.

### When pooling happens
Pooling occurs after all transformer layers:

```
text
→ tokens (IDs)
→ embedding lookup (static vectors)
→ + positional encoding
→ transformer layers
     → attention happens here
     → semantics injected here
→ final token vectors (h₁...hₙ)
→ pooling
→ single embedding vector
```
Note that you have a **single embedding vector** representing the chunk at the end of it

### Types and details

#### Mean pooling
- Average all token vectors.
- Strong baseline but can blur signal in long texts.

#### Max pooling
- Takes per-dimension maximum across tokens.
- Highlights salient features, can be noisy.

#### CLS pooling
- Uses special [CLS] token representation.
- Works only if trained explicitly for this role.
- CLS is desingnated as the global representative and train the model to stuff everything important into it.

#### Attention‑Weighted Pooling (Deep Dive)

Attention‑weighted pooling is a technique for collapsing a variable number of token embeddings into a single fixed‑size embedding by **learning which tokens contribute most** to the meaning of the input text.

---

##### Core idea

After the transformer layers, we have contextual token embeddings:

```
h₁, h₂, …, hₙ    (each hᵢ ∈ ℝᵈ)
```

Instead of treating all tokens equally, attention‑weighted pooling computes a **weighted sum**:

```
embedding = Σ αᵢ · hᵢ
```

Where:
- αᵢ represents the importance of token *i*
- Σ αᵢ = 1

---

##### How the weights αᵢ are computed

###### 1. Score each token

Each token embedding `hᵢ` is mapped to a scalar score:

```
scoreᵢ = f(hᵢ)
```

Common choices for `f`:
- Dot product with a learned projection vector
- A small learned MLP
- Similarity to a learned global query vector

This step answers: *“How important is this token for representing the entire sequence?”*

---

###### 2. Normalize scores

Scores are converted into weights using softmax:

```
αᵢ = softmax(scoreᵢ)
```

Properties:
- All αᵢ > 0
- Σ αᵢ = 1
- Larger score → larger contribution

---

###### 3. Compute the pooled embedding

```
embedding = Σ αᵢ · hᵢ
```

Tokens with higher semantic importance contribute more to the final embedding.

---

##### Why αᵢ values are **not** static

A crucial distinction:

- **Static parameters**
  - Projection vectors, MLP weights, or query vectors
  - Learned during training and fixed afterward

- **Dynamic weights (αᵢ)**
  - Computed at inference time
  - Depend on the actual token embeddings `hᵢ`

Because:

```
αᵢ = softmax(f(hᵢ))
```

and each `hᵢ` depends on:
- The input text
- Token interactions via self‑attention
- Document length and structure

**Different inputs → different αᵢ patterns**

This mirrors transformer self‑attention:
- The computation rule is fixed
- The resulting attention values are input‑dependent

---

##### Relationship to transformer self‑attention

| Aspect | Transformer Self‑Attention | Attention‑Weighted Pooling |
|------|-----------------------------|-----------------------------|
| Queries | One per token | One global query |
| Scope | Token → token | Tokens → sequence |
| Purpose | Build contextual representations | Summarize the sequence |
| Output | Token vectors | Single embedding |

You can think of attention‑weighted pooling as:

> **One final attention step whose only job is summarization.**

---

##### Why this pooling method works well for embeddings

- Emphasizes discriminative tokens
- Suppresses boilerplate and filler
- Adapts naturally to variable length inputs
- More expressive than mean pooling
- Less brittle than CLS‑only pooling

As a result, attention‑weighted pooling is widely used in modern retrieval‑oriented embedding models.

- 

#### Last-token pooling
- Uses final token representation.
- Common in decoder-only embedding models.

---

## Transformer layers in embeddings

### Layer structure

```
Input vectors
  → Self-attention
  → MLP / FFN
  → Output vectors
```

### Why each layer exists
- **Self-attention** routes contextual information.
- **MLP layers** build semantic abstractions via nonlinear transforms.

Attention decides *what* information is shared; MLPs decide *how* it is interpreted.

---

## Advanced topics

### Context length of embeddings

**What is it?**
- Maximum tokens processed in one embedding pass.

**Advantages of long context**
- Fewer chunks
  - This is because each chunk now has a deeper semantic meaning
  - So instead of pulling in more chunks into context, fewer more meaningful chunks can be pulled though each chunk in this case may be larger
- Preserves long-range dependencies
- Higher-quality coarse retrieval

---

### Embedding model selection

Key factors:
- Domain and language
- Context length
- Embedding dimension
- Latency and hardware constraints
- Instruction or prompt requirements

---

### Training embedding models (overview)

Typical steps:
1. Collect or generate (query, positive, negative) pairs
2. Contrastive pretraining
3. Fine-tuning on curated data
4. Hard-negative mining
5. Distillation into smaller models

**Why companies open-source embeddings**
- Foundational infrastructure
- Ecosystem adoption
- Transparency and benchmarking

**Example models**
- Harrier (Microsoft)
- OpenAI text-embedding-3
- BAAI bge-m3
- Cohere Embed
- Voyage AI embeddings

#### Contrastive Training for Embedding Models

##### What is contrastive training?

Contrastive training is a learning paradigm where a model is trained to **bring semantically similar inputs closer together in embedding space and push dissimilar inputs farther apart**.

Instead of predicting labels or tokens, the model learns a *geometry*:
- distance encodes meaning
- similarity reflects semantic relatedness

The output of training is **not a classifier**, but a representation space.

---

##### Why contrastive training is used for embeddings

Embedding models are used for:
- semantic search
- retrieval and ranking
- clustering
- nearest‑neighbor lookup

These tasks care about **relative similarity**, not absolute labels.

Contrastive training directly optimizes for this by enforcing:
- similar texts → nearby vectors
- unrelated texts → distant vectors

This makes contrastive learning a natural fit for embedding objectives, unlike standard supervised classification or generative losses.

---

##### The basic training unit

A typical contrastive setup operates on *pairs* or *tuples* of texts:

- **Anchor (A)** — the reference text
- **Positive (A⁺)** — semantically related to A
- **Negative (A⁻)** — semantically unrelated to A

The training objective encourages:

```
sim(A, A⁺)  >  sim(A, A⁻)
```

Only relative relationships matter; there is no "correct" embedding vector.

---

##### How contrastive training fits the embedding pipeline

During training, the full embedding pipeline is active:

```
text
 → tokens
 → transformer layers (attention + MLP)
 → contextual token embeddings
 → pooling
 → final embedding vector
```

Embeddings are then compared using a similarity function (commonly cosine or dot product), and gradients flow **through the entire model**, including:
- attention weights
- MLP parameters
- pooling parameters
- token embeddings

Contrastive loss shapes *all* of these jointly.

---

##### What contrastive training teaches the model

Contrastive objectives implicitly train the model to:

- focus attention on discriminative tokens
- suppress boilerplate and filler text
- produce embeddings stable under paraphrase
- encode meaning in global geometry rather than local syntax

The model is rewarded when its **embedding geometry** matches semantic intuition.

---

##### Role of negatives

Negatives are critical for embedding quality.

Common negative types:
- random unrelated texts
- in‑batch negatives (other items in the same batch)
- *hard negatives* (topically similar but incorrect texts)

Hard negatives sharpen the model’s decision boundary and improve retrieval precision.

---

##### Data used in contrastive training

Contrastive datasets are typically built from:

- query–document pairs
- question–answer pairs
- paraphrases
- translations
- summaries

Modern embedding models often rely heavily on **synthetic data**, generated at scale using stronger language models, combined with smaller sets of curated high‑quality pairs.

---

##### Contrastive training vs classification training

| Aspect | Classification | Contrastive Embeddings |
|------|----------------|------------------------|
| Objective | Predict label | Shape vector space |
| Output | Probabilities | Embedding vectors |
| Supervision | Explicit labels | Relative similarity |
| Error type | Categorical | Geometric |

Contrastive training optimizes *distances*, not decisions.

---

##### Relationship to pooling and attention

Contrastive loss does **not** label individual tokens or pooling weights directly.

Instead:
- pooling strategies evolve to preserve useful semantic signal
- attention patterns adapt to support discrimination
- MLP layers learn nonlinear semantic features

Pooling, attention, and token representations are all shaped indirectly through the contrastive objective.

---

##### Why contrastive training generalizes well

Because it optimizes relative structure rather than absolute outputs, contrastive training:
- transfers across tasks
- supports unseen domains
- works with different LLMs downstream

This is why embedding models are often decoupled from generation models.

---

##### Key takeaway

> **Contrastive training teaches an embedding model what semantic similarity means by shaping the geometry of its vector space, not by predicting text or labels.**

This makes it the dominant training paradigm for modern embedding models.


---

## Harrier case study

### Why Microsoft released Harrier
- To improve grounding and retrieval for search and agents.

### Special features
- Multilingual
- Long context (~32k tokens)
- Multiple model sizes
- Decoder-only with last-token pooling

### Training notes
- Large-scale contrastive learning
- Synthetic and weak supervision
- Knowledge distillation

---

## Using embedding models: Harrier example

### Goal
Expose an embeddings API similar to OpenAI’s.

### High-level architecture

```
Client
  → /v1/embeddings API
  → Validation & batching
  → Harrier model runtime
  → JSON embeddings response
```

### Implementation checklist
1. Load model once at startup
2. Encode documents and queries
3. Normalize embeddings
4. Return vectors in a standard schema
5. Store vectors with text in a vector DB

---

## Appendix: RAG flow

```
User query
  → Query embedding
  → Vector search
  → Retrieve text
  → LLM prompt
  → Generated answer
```

Vectors are never converted back to text; original text is stored alongside embeddings.

---

## References

- Microsoft Bing Blog – Harrier announcement
- Hugging Face Harrier OSS models
- SentenceTransformers documentation
- Massive Text Embedding Benchmark (MTEB)
- OpenAI Embeddings documentation
