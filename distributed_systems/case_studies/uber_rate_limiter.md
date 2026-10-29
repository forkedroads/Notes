# Uber Rate Limiting System — Design Notes

## 1. The Problem

Uber's service-oriented architecture handles **hundreds of millions of RPCs/second** across **thousands of microservices**. Without strong overload protection, a single misbehaving caller or traffic spike could cascade into system-wide failures. The need: a way to **limit traffic per caller/procedure** without requiring every team to build custom logic.

---

## 2. The Old Architecture (Pre-GRL)

Before a unified system existed, rate limiting was **fragmented and ad hoc**:
- Teams built their own throttling — embedded in business logic, custom middleware, or **Redis-based counters**.
- Each request would increment/read a shared counter (e.g., in Redis); once a threshold was crossed, requests were rejected.

### Limitations of the Old Approach

**a) Operational & Scalability Overhead**
- Centralized counters (Redis) require a **network round-trip per request** to increment/check state.
- At Uber's scale (hundreds of thousands of hosts, multi-region traffic), this introduces unacceptable latency and cross-region consistency problems.
- Scaling Redis to match demand would require hundreds of clusters — adding operational complexity and new failure modes.

**b) Fragmentation & Inconsistency**
- Different teams implemented limiting differently → inconsistent behavior even for identical quotas.
- Many smaller services had **no rate limiting at all**.
- Updating limits often required **redeploying code**, slowing response to traffic changes.
- Poor observability — no unified way to tell if a failure was due to rate limiting or something else.

**Conclusion:** A fully distributed architecture — local enforcement using aggregated load data instead of a central counter — was the only way to achieve both low latency and global scale. This led to the creation of **GRL (Global Rate Limiter)**, built into Uber's service mesh.

---

## 3. GRL Phase 1: Token Bucket Algorithm

When the GRL project began, the team's **first enforcement mechanism** was a **token-bucket algorithm**, implemented on each proxy within the service mesh.

- Tokens refill at a configured rate into a "bucket"; each request consumes a token.
- If tokens run out, requests are throttled/rejected.
- This worked well when a rate limit was enforced for a **single caller**.

### Limitations
There are two. 

#### Unfair Caller Distribution
- In practice, a single limit was often **shared across multiple callers** (e.g., 10 different services all calling the same downstream procedure under one quota).
- The token bucket has **no concept of "who" is consuming tokens** — it hands tokens out first-come-first-served.
- A high-volume or aggressive caller could **monopolize most of the tokens**, starving other, well-behaved callers of their fair share — even though all callers were nominally subject to the "same" limit.
- This fairness gap meant the token bucket could not **equitably divide capacity** once the number of distinct callers grew.

#### Under utilization
- Another issue was instance-level spikes. Individual hosts with bursty traffic could start dropping requests prematurely, even when overall load was still below the global limit.

These limitations drove the team to move away from shared token accounting toward a model that treats each caller/procedure combination independently.

---

## 4. GRL Phase 2: The Probabilistic Drop Model (Final Solution)

To solve the fairness problem — and simplify the hot path — GRL moved to a **stateless, probabilistic sampling model**, replacing per-request token accounting.

### Three-Tier Feedback Loop (Unchanged Architecture)
1. **Clients** (in service mesh) — enforce decisions locally, report request counts to aggregators.
2. **Aggregators** (per zone) — collect/compute zone-level usage, forward to controllers.
3. **Controllers** (regional/global) — compute global utilization, push down **drop ratio directives** per bucket (caller/procedure).

### The Hot-Path Logic
Instead of tracking tokens, each client applies a **stateless, probabilistic decision** per request:

```
drop_ratio = cached_directive_for_bucket   // e.g., 0.30 (pushed down periodically)
random_value = uniform_random(0, 1)         // fresh draw per request

if random_value < drop_ratio:
    drop request
else:
    forward request
```

**How the "bias" works:** The RNG itself is unbiased/uniform — the drop ratio simply defines a "zone" on the [0,1] range. Since draws are uniform, the probability of landing in a zone of width `drop_ratio` equals `drop_ratio` exactly — no counting needed. Over high request volume, the actual drop rate converges to the target ratio (law of large numbers).

### Why This Solves Both Problems
- **Solves fairness:** Each caller/procedure bucket gets its **own independently-enforced ratio** rather than competing for a shared token pool — no caller can "steal" capacity from another.
- **Solves performance/scale:** No local token accounting, no shared mutable state, no locks or contention under concurrency.
- **No per-request communication** with the control plane — only the drop ratio value itself is pushed down periodically (every few seconds), not fetched per request.
- **Decisions are made entirely in-process** — just a random draw + comparison → extremely cheap, low-latency.
- **Fail-open** — if the control plane is unreachable, clients default to forwarding traffic rather than risking self-inflicted outages.

### Reference
https://www.uber.com/us/en/blog/ubers-rate-limiting-system/
