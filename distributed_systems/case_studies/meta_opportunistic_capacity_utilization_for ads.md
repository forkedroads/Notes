# Summary: Turning Opportunistic AI Inference Capacity Into Better Ads Recommendations

## The Problem
Meta's AI inference fleet faces a **capacity paradox**: globally constrained, but with pockets of **locally idle capacity** due to:
- Peak-provisioning vs. variable demand
- Model development vs. hardware delivery timing mismatches
- Regional CPU/accelerator resource ratio mismatches
- Accelerator type/vendor incompatibilities

This idle capacity is **preemptible** (can be reclaimed anytime), **volatile**, and **scattered** across teams, regions, and hardware types.

## The Solution: Two Elasticity Levers

| Lever | What it does | Resources needed |
|---|---|---|
| **Rank More Ads (RMA)** | Evaluates more candidate ads per request | CPU + accelerator (candidates flow through retrieval, feature processing, *and* inference) |
| **Serve Better Models (SBM)** | Routes more requests to a stronger/more accurate model | Accelerator only (candidate volume stays constant, so CPU demand is stable) |

Because they consume different resource mixes, together they absorb a wider range of opportunistic capacity than either alone. Allocation between them is periodically rebalanced based on measured value.

## Protecting Baseline Service
**Core rule:** Losing opportunistic capacity may reduce bonus work, but must *never* overload baseline serving.

- **SBM: physical isolation** — separate replica pools for baseline vs. powerful model; on contraction, requests simply fall back to baseline.
- **RMA: logical isolation** — shared replicas, but a global budget caps total incremental work to what spare capacity can support.

## Feedback Loops (minute-scale)
- **RMA**: Central **PID controller** watches **tail latency** → sets target candidate volume → clients apply it. Latency ↓ → candidates ↑; latency ↑ → candidates ↓.
- **SBM**: Central controller sets **target coverage** (% of traffic to powerful model) → client-side router applies it per request, using a **request-value signal**.

## Load Shedding (millisecond-scale)
Since central loops react over minutes but capacity can vanish instantly, two faster safety nets exist:
- **Server-side shedding**: servers immediately drop lower-value work (RMA: fewer candidates per request; SBM: fewer requests kept on powerful-model path).
- **Client-side shedding**: clients watch live server status (not just the last controller target) and independently cut back if they detect overload.

Both use **value signals** to drop least-valuable work first, minimizing impact.

---

## Additional Details from Follow-up Discussion

**Capacity-change detection — server vs. client signal:**
- Slow loop: central controller infers capacity pressure indirectly via **server-side tail latency**.
- Fast loop: both servers (detect local overload directly) and clients (observe live server status, not just controller targets) react independently within milliseconds.

**What exactly is "value-aware shedding" dropping?**
- RMA: trims the number of extra candidates *within* an already-in-flight request.
- SBM: removes lower-value requests from the powerful-model path, falling back to baseline model.

**How SBM rerouting works mechanically:**
1. Client-side router buckets requests into **value segments**.
2. An **explore-and-exploit** process tests actual incremental benefit of the powerful model per segment (since raw value ≠ incremental benefit).
3. Central controller sets global target coverage; router matches segment-level routing to that target.
4. A **calibration loop** keeps real routing share aligned with target as traffic shifts.

**How is "request value" actually computed?**
- The article defines it as **predicted likelihood of user engagement** — essentially a pCTR/engagement-probability-style score, likely derived from a cheap pre-ranking signal already available for the request (user history, context, candidate pool).
- Critically, the system doesn't trust absolute value alone — it explicitly tests **incremental lift** from the powerful model per segment via live experimentation, since high-value requests don't always benefit most from a stronger model (an "uplift modeling" pattern, similar to medical triage — send extra resources where they'll actually change the outcome, not just where baseline value is highest).

**How is the underlying capacity pool itself maintained?**
- Not detailed in this post. Likely builds on a separate Meta system (**Dynamic Idle Resource Leasing**, SoCC 2024) that continuously scans the fleet for idle capacity and leases it out with defined SLOs — RMA/SBM appear to be consumers sitting on top of this lower-level discovery/allocation layer.

## Future Work
- Expanding RMA/SBM operating ranges (higher candidate volumes, stronger models)
- New levers: **precomputation elasticity** (work done ahead of request time)
- Better **model portability** across regions/accelerators
- An **optimization-based system** to automatically allocate capacity across levers based on measured user/advertiser value

---

## References

**Primary source:**
- [Turning Opportunistic AI Inference Capacity Into Better Ads Recommendations — Engineering at Meta](https://engineering.fb.com/2026/10/12/ml-applications/turning-opportunistic-ai-inference-capacity-into-better-ads-recommendations/)


**Related Meta infrastructure background:**
- [InfoQ — Meta Optimises AI Inference by Improving Tail Utilisation](https://www.infoq.com/news/2024/08/meta-tail-utilisation/)
- [ACM — Dynamic Idle Resource Leasing To Safely Oversubscribe Capacity At Meta (SoCC 2024)](https://dl.acm.org/doi/10.1145/3698038.3698537)
- [At Scale Conferences — Multi-Tenancy for AI Inference at Meta Scale](https://atscaleconference.com/multi-tenancy-for-ai-inference-at-meta-scale/)
- [snowchord — XFaaS: Hyperscale and Low Cost Serverless Functions at Meta (SOSP '23)](https://snowchord.com/papers/xfaas-sosp23.pdf)

**Background on Meta ads ranking / value signals:**
- [Meta AI Blog — New AI advancements drive Meta's ads system performance (Meta Lattice)](https://ai.meta.com/blog/ai-ads-performance-efficiency-meta-lattice/)
- [arXiv — Meta Lattice: Model Space Redesign for Cost-Effective Industry-Scale Ads Recommendations](https://arxiv.org/html/2512.09200v2)
