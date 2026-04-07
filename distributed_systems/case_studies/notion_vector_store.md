# How Notion Scaled AI Q&A to Millions of Workspaces
*A phased infrastructure evolution*

---

## 1. What Notion Started With

Notion launched AI Q&A as a retrieval‑augmented feature that allows users to
ask natural‑language questions over their workspace content and connected tools
like Slack or Google Drive. This required embedding large volumes of content and
running vector search at scale.

Demand exploded quickly, turning vector search into a large-scale,
stateful infrastructure problem involving storage growth, compute pressure,
and high write amplification.

---

## 2. Original Architecture (V1)

### Architecture Overview

- Dual ingestion pipelines:
  - **Offline**: Spark batch jobs for backfills
  - **Online**: Kafka consumers for live updates
- Vector database deployed as **pods**
- Storage and compute tightly coupled
- Sharding by `workspace_id`

Each workspace mapped to a single shard for isolation and simplicity.

### Problems Encountered

- Shards had fixed capacity
- Storage and compute scaled together
- Re-sharding required massive data movement
- Existing workspaces continued to grow, stressing shards even without new onboarding

**Key clarification:**  
Sharding by workspace ID delays pressure but does not eliminate the problem of
existing workspace growth.

---

## 3. Phase 1: Generation IDs (First Survival Optimization)

### Why This Was Needed

Onboarding demand exceeded capacity planning assumptions.
Re-sharding existing data was too slow and risky.

### Solution: Generation IDs

- Existing shards were frozen once near capacity
- New shards allocated as a new "generation"
- New workspaces assigned only to the latest generation
- Existing workspaces remained in place

### What This Solved

- No data movement
- No onboarding pauses
- Low operational risk

### What It Did Not Solve

- Existing workspaces still grew
- Storage remained shard-bounded
- Generations accumulated
- Operational complexity increased over time

---

## 4. Phase 2: Serverless Embeddings

### Why This Was Necessary

Even with generations, shards saturated early due to **compute pressure**, not storage:

- Frequent edits triggered full re-embeddings
- Embedding compute ran inside vector DB pods
- Compute saturation made shards unusable early

### What Changed

Embedding generation was moved out of pods into a **serverless embeddings layer**:

- Elastic, per-use compute
- Detached from storage lifecycle
- Burst-tolerant for onboarding and edits

### Impact

- Shard pressure became primarily storage-driven
- Existing workspaces could grow without saturating shard compute
- Fewer emergency capacity events
- Cost efficiency improved

**Important clarification:**  
This reduced shard pressure but did **not** remove the need for generations or
fixed storage limits.

---

## 5. Phase 3: turbopuffer (Root-Cause Fix)

### Remaining Problems Before turbopuffer

- Shards still had fixed storage capacity
- Generations continued growing in number
- Long-term capacity planning still required
- Re-sharding avoided, but not eliminated

### What turbopuffer Introduced

- Object-storage-backed vector search
- Decoupled storage from compute
- Logical namespaces instead of fixed shards
- Pay per byte stored instead of per running pod

### Why This Solved the Problem

- Storage no longer "fills up"
- No shard splitting required
- Generation IDs no longer needed for capacity
- Existing workspaces can grow indefinitely

### Interaction with Serverless Embeddings

Serverless embeddings continued to be used:

- turbopuffer solves **storage and indexing scalability**
- serverless embeddings solve **compute elasticity**
- Together, they eliminate both growth bottlenecks

---

## 6. Final Takeaway

Notion’s AI Q&A infrastructure evolved from a pod-based, sharded vector database
that struggled under explosive growth, through survival-driven mitigations
(generation IDs and serverless embeddings), to a fundamentally more scalable
architecture with turbopuffer. The final system removes fixed shard capacity as
a constraint, making workspace growth—new or existing—a routine, uneventful
operation.

The progression is a textbook example of:
**decoupling compute first, then decoupling storage.**

## Ascii version

```text
====================================================================
PHASE 0 / V1 — ORIGINAL SYSTEM (POD‑BASED VECTOR SEARCH)
====================================================================

                ┌──────────────┐
                │  Notion UI   │
                │  / AI Q&A    │
                └──────┬───────┘
                       │ Query
                       ▼
              ┌──────────────────┐
              │  Query Router    │
              │ (workspace_id →  │
              │    shard)        │
              └──────┬───────────┘
                     │
     ┌───────────────┼────────────────┐
     │               │                │
     ▼               ▼                ▼
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Vector  │     │ Vector  │     │ Vector  │
│ DB Pod  │     │ DB Pod  │     │ DB Pod  │
│ Shard A │     │ Shard B │     │ Shard C │
│         │     │         │     │         │
│Storage  │     │Storage  │     │Storage  │
│+Compute │     │+Compute │     │+Compute │
│+Embed   │     │+Embed   │     │+Embed   │
└─────────┘     └─────────┘     └─────────┘

Problems:
- Fixed shard capacity
- Storage + compute coupled
- Re‑sharding expensive
- Existing workspace growth fills shards


====================================================================
PHASE 1 — GENERATION IDs (SURVIVAL MODE)
====================================================================

                ┌──────────────┐
                │  Notion UI   │
                └──────┬───────┘
                       │
                       ▼
            ┌────────────────────┐
            │ Query Router       │
            │ (workspace_id +    │
            │  generation_id)    │
            └──────┬─────────────┘
                   │
        ┌──────────┴───────────┐
        │                      │
        ▼                      ▼
┌────────────────┐   ┌────────────────┐
│ Generation 1   │   │ Generation 2   │
│ (Frozen)       │   │ (Active)       │
│                │   │                │
│ ┌───────────┐  │   │ ┌───────────┐  │
│ │ Shard A   │  │   │ │ Shard D   │  │
│ │ Shard B   │  │   │ │ Shard E   │  │
│ └───────────┘  │   │ └───────────┘  │
└────────────────┘   └────────────────┘

Key idea:
- Existing workspaces stay put
- New workspaces go to new generations
- No data movement
- Storage pressure remains


====================================================================
PHASE 2 — SERVERLESS EMBEDDINGS (DECOUPLE COMPUTE)
====================================================================

                ┌──────────────┐
                │  Notion UI   │
                └──────┬───────┘
                       │
          Queries       │       Edits / Ingest
                       ▼
            ┌────────────────────┐
            │ Query Router       │
            └──────┬─────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │ Vector DB Shards          │
        │ (Storage + Index only)   │
        │                          │
        │  Generation 1 / 2 / N    │
        └──────────┬──────────────┘
                   │ vector writes
                   ▼
        ┌──────────────────────────┐
        │ Serverless Embeddings    │
        │ (Elastic compute)        │
        │                          │
        │ - Scales on demand       │
        │ - Paid per use           │
        └──────────────────────────┘

Effect:
- Compute no longer fills shards
- Existing workspaces can grow longer
- Storage still finite → generations remain


====================================================================
PHASE 3 — TURBOPUFFER (ROOT CAUSE REMOVED)
====================================================================

                ┌──────────────┐
                │  Notion UI   │
                └──────┬───────┘
                       │
                       ▼
            ┌────────────────────┐
            │ Query Router       │
            │ (workspace_id →    │
            │  namespace)        │
            └──────┬─────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │ turbopuffer              │
        │ (Vector Search Engine)   │
        │                          │
        │ - Object storage backed  │
        │ - Elastic growth         │
        │ - No shard capacity      │
        └──────────┬──────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │ Serverless Embeddings    │
        │ (still in use)           │
        └──────────────────────────┘

Final state:
- No re‑sharding
- No generation sprawl
- Workspace growth is boring (good)


```
