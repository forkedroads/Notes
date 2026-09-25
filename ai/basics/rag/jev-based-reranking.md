# CLERC Re-ranking Experiment — Summary

## 1. Dataset Setup
- Source: CLERC dataset (US court opinions), each raw row = `{query, gold passage, negative passages}`
  - **query**: an opinion excerpt with its central citation removed
  - **gold passage**: the passage the removed citation actually pointed to
  - **negative passages**: topically similar but incorrect distractor passages
- The cookbook pulls **170 CLERC rows**.
  - **40 rows** are used as the actual evaluation **queries**.
  - All passages (gold + negative) from **all 170 rows** — including the 130 rows *not* used as queries — are pooled into a single flat corpus.
- Resulting corpus: **3,565 passages** total, shared across all 40 queries (not siloed per-query).
- Passage origin (gold vs. negative, which row it came from) is irrelevant to search — it's just text in the corpus. Only one label matters per query: *which single passage is that query's gold answer.*

## 2. Fast Search (Stage 1 — BM25)
- For each of the 40 queries, BM25 scores **all 3,565 passages** in the shared corpus (real search, not restricted to that row's own candidates).
- Returns the **top-30 shortlist** per query.
- BM25 scores by: term frequency, inverse document frequency, field boosting, document length normalization — pure keyword overlap, no semantic understanding.
- **Result:** gold passage appears somewhere in the top-30 for **100% of queries** (recall is solved) — but is ranked **#1 in only 5%** of queries (ordering is poor). Top-5 = 15%, Top-10 = 38%.
- **Key takeaway:** BM25 is good at *not losing* the answer, bad at *ordering* it — especially with lexical distractors sharing legal jargon.

## 3. Re-ranking (Stage 2 — TypeSafe)
- Takes each 30-candidate shortlist and re-orders it using **one independent scoring call per (query, candidate) pair** → 40 × 30 = **1,200 calls**, run concurrently.
- Each call uses a **Noul** (TypeSafe's yes/no probability question), e.g.:
  > "Could this candidate passage be from the cited precedent?"
  - Criteria define what counts as true/false (e.g., "states the specific rule cited" vs. "only on a similar topic").
  - Returns a probability (0–1) that the answer is "yes" — this is the score.
- No general-purpose scoring scale needs to be invented; the noul is directly comparable across all 30 candidates.
- Shortlist is then **sorted by noul, highest first**.
- Each call only sees `{query, one candidate}` — never the full shortlist or other candidates (independent, parallelizable, small state).

## 4. Results
| Metric        | Fast search (BM25) | + Re-ranking (TypeSafe) |
|---------------|--------------------:|-------------------------:|
| Top-1 accuracy  | 5%  | 18% |
| Top-5 accuracy  | 15% | 35% |
| Top-10 accuracy | 38% | 62% |

- Re-ranking can only **reorder** what fast search already surfaced — it can't recover a gold passage that never made the top-30.
- Since recall@30 was already 100%, the entire performance gap here is a **ranking/precision problem**, not a recall problem — which is exactly what re-ranking is designed to fix.
- Cost: 1,200 Noul calls ≈ 1.5M input tokens, ~$0.0645 total (long legal passages → ~1,280 tokens/call).

## 5. Big Picture
```
Corpus (3,565 passages, pooled from 170 rows)
        │
        ▼
BM25 fast search (per query, over full corpus)
        │
        ▼
Top-30 shortlist  (gold passage always present, rarely first)
        │
        ▼
TypeSafe re-ranking (1 Noul call per query-candidate pair)
        │
        ▼
Shortlist sorted by noul, highest first
        │
        ▼
Top-1 accuracy: 5% → 18%
```

## Reference
https://docs.typesafe.ai/cookbooks/rerank_typesafe
