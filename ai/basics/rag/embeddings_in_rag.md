#  Embeddings, Updates, and RAG (Notion-style)

This document summarizes what I found out when digger a little deeper into how embeddings are used in RAG when looking at an article on how notion designed their QnA system
The following topics were explored using a llm tool like co-pilot: 
- **how embeddings are produced and stored**
- **how they are updated efficiently (especially with partial edits)**
- **how they power a RAG workflow while staying independent from the choice of LLM**.

The Notion-specific mechanisms (span hashing, DynamoDB page-state cache, metadata-only patching, and Ray Serve query-embedding) are drawn from Notion’s engineering [blog](https://www.notion.com/blog/two-years-of-vector-search-at-notion).

---

## 1) Embeddings

### 1.1 How embedding is done in practice

Embeddings are created by running text through an **embedding model** (typically a transformer encoder or encoder-like architecture) to produce a fixed-length vector per indexed unit.

#### A) Chunk creation

**Why chunk?**
- Long documents/pages are split into smaller “chunks/spans” because models have context limits, retrieval works better on semantically coherent units, and **incremental updates** become feasible (re-embed only impacted spans).

**What a chunk looks like (conceptually)**
- A chunk is a contiguous span of text such as a paragraph, heading+paragraph, or a sliding token window.
- Common approaches:
  - **Structure-aware chunking**: preserve headings/sections (e.g., “Security Deposits”).
  - **Token-window chunking**: fixed token length with overlap.

Notion chunks long pages into **spans** and embeds each span for indexing.

---

#### B) Embedding these chunks (emphasis on independence)

**Key idea: each chunk/span is embedded independently.**

- For span *i*: `E_i = Embed(span_i_text)`
- The embedding for span *i* does **not** attend to other spans because they are not included in its input.

This is a deliberate trade-off:
- ✅ Enables partial updates (re-embed only changed spans)
- ✅ Keeps compute bounded by span length, not whole-document length
- ❌ Sacrifices global, whole-document contextualization *during embedding time* (the LLM can recover coherence later when it reads retrieved text).

Notion uses span-level embedding and stores metadata (authors/permissions) per span in the vector DB.

---

#### C) How semantic meaning is infused into embeddings (encoders and pooling)

This resolves the “is embedding just a lookup?” confusion.

**What *is* just a lookup (cheap)**
- Token IDs map to initial token vectors via a learned embedding table.

**What makes embeddings semantic (compute-intensive)**
- The stored embedding is produced by running a **full forward pass** through the model:
  - Self-attention layers contextualize token meanings.
  - MLP/FFN layers transform representations.
  - A pooling step produces one vector per span (e.g., mean pooling or [CLS]).

**Why it’s compute-intensive**
- Attention and MLP layers are dominated by large matrix multiplications.
- Attention cost grows ~quadratically with sequence length within the span.
- Embedding generation resembles the **prefill-like** cost profile: a full pass over all tokens in the input span, producing a representation (but unlike chat generation, there’s no KV-cache reuse afterward).

---

## 2) How embeddings are updated

### 2.1 Validity rule

Embeddings remain valid until either:
- The span’s **text** changes, or
- The **embedding function** changes (model/tokenizer/pooling/chunking version).

---

### 2.2 Example: partial updates (logical/text-based)

Assume a “Local Leasing Laws” document chunked as:
- **Span A**: Rent Increases
- **Span B**: Security Deposits
- **Span C**: Evictions

Initially:
- `E_A = Embed(A)`
- `E_B = Embed(B)`
- `E_C = Embed(C)`

Now only **Span B** changes (e.g., “max deposit is one month” → “two months”).

Update behavior:
- Recompute: `E_B' = Embed(B')`
- Keep: `E_A` and `E_C` unchanged

**Why this works despite attention intuition**
- Attention is global **within the span**, not across the entire document, because spans are embedded independently.
- Spans that did not change do not need recomputation.

---

### 2.3 Notion: identifying changed spans using DynamoDB page state

Notion observed that previously any edit caused them to **re-chunk, re-embed, and re-upload all spans** of a page—even for tiny changes.

They introduced a “Page State” approach:

**What they store**
- Two hashes per span:
  - **Text hash** (span text)
  - **Metadata hash** (permissions/authors/etc.)
- Stored as **one DynamoDB record per page**, containing all spans and their hashes.

**Update logic**
1. Chunk the current page into spans.
2. Fetch previous page state from DynamoDB.
3. Compare hashes:
   - **Case 1: Text changes** → re-embed and re-load only changed spans.
   - **Case 2: Metadata changes only** → skip embedding; issue a metadata-only PATCH/update to the vector DB (cheaper).

**Outcome**
- Notion reports a **70% reduction in data volume**, saving on embedding costs and vector DB write costs.

---

## 3) RAG (assume similarity-based retrieval)

### 3.1 How embeddings are used

Embeddings power **retrieval**, not generation:
- Each span is stored as `(vector, span_text, metadata)`.
- At query time:
  1. Embed the user query using the **same embedding model**.
  2. Run similarity search to retrieve top-K spans.
  3. Provide the retrieved **text** to the LLM for answering.

Notion notes they must embed queries “on the fly” before searching, and this is latency sensitive.

---

### 3.2 Retrieval (similarity search)

Example user question:
> “Is it legal for a landlord to charge two months’ rent as a deposit?”

Steps:
1. Compute query embedding: `q = Embed(question)`
2. Vector DB returns nearest spans (e.g., “Security Deposits” section).
3. Gather retrieved text + metadata.
4. Build an LLM prompt with:
   - System instructions
   - Retrieved passages (with citations)
   - User question
5. LLM generates a grounded answer.

---

### 3.3 ASCII diagram: 2-stage pipeline (Ingest + Query)

```text
STAGE 1: INGEST / INDEX (offline + incremental)
------------------------------------------------
  Corpus of Docs
       |
       v
  Parse / Clean Text
       |
       v
  Chunk into spans (independent units)
       |
       v
  Hash spans (text hash + metadata hash)
       |
       +--> Compare vs cached page state (DynamoDB); skip unchanged
       |
       v
  Embed each NEW/CHANGED span  --->  [GPU/Encoder forward pass]
       |
       v
  Upsert into Vector DB (e.g., turbopuffer)
       |
       v
  Store updated "page state" (span hashes) in DynamoDB


STAGE 2: QUERY / ANSWER (online, latency-sensitive)
---------------------------------------------------
  User Question
       |
       v
  Embed Query  --------------------> [same embedding model]
       |
       v
  Similarity Search in Vector DB  -> top-K relevant spans
       |
       v
  Build Prompt:
    - System instructions
    - Retrieved span TEXT (+ citations/metadata)
    - User question
       |
       v
  LLM Generation (OpenAI / Anthropic / other)
       |
       v
  Answer + citations
```

---

### 3.4 Why embeddings are independent of the choice of LLMs

**Separation of concerns**
- Retrieval layer: embedding model + vector DB + similarity search.
- Generation layer: LLM reads **retrieved text**, not vectors.

**What is coupled**
- Document embeddings and query embeddings must be in the **same embedding space**, i.e., use the same embedding model/version.

**What is decoupled**
- The choice of LLM provider (OpenAI vs Anthropic vs others) can change without requiring re-indexing, because the LLM consumes the retrieved text.

---

## One-screen takeaways

- **Embedding is not just a lookup**: semantic embeddings require a full encoder forward pass.
- **Chunk embeddings are intentionally independent**: enables incremental updates.
- **Notion’s Page State** uses DynamoDB + per-span text/metadata hashes to avoid unnecessary re-embedding and allow cheaper metadata-only updates.
- **RAG is two-stage**: (1) build/maintain index, (2) query-time retrieval + LLM answer.
- **Embeddings are independent of the LLM**: you can swap LLM providers as long as retrieval remains stable.
