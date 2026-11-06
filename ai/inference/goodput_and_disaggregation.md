
# Section 1 : Goodput as a metric
## 1. Goodput > Throughput as the right metric
- **Throughput** (requests/sec processed) ignores whether users actually get acceptable latency.
- **Goodput** = requests/sec that meet SLOs (TTFT and TPOT targets simultaneously). A system can have high throughput but terrible goodput if latency requirements aren't met — this is the central reframing of the whole post.

## 2. Prefill and decode are fundamentally different workloads
- **Prefill**: compute-bound (processes the whole prompt at once, saturates GPU compute).
- **Decode**: memory-bandwidth-bound (one token at a time, repeatedly reading weights + KV cache).
- Traditional systems **colocate** both phases on the same GPU via continuous batching — this creates **interference**: a new prefill job stalls ongoing decodes (hurting TPOT), and decode jobs crowding the batch delay prefill (hurting TTFT).

## 3. The core fix: Disaggregation
- Split prefill and decode onto **separate GPUs** entirely.
- This eliminates interference and **decouples resource allocation** — each phase can be independently scaled and given its own parallelism strategy tailored to its own SLO (TTFT for prefill, TPOT for decode).

## 4. The headline numbers (the example you discussed)
- Colocated: ~1.6 rps/GPU (bottlenecked by the stricter of TTFT/TPOT: `min(3, 1.6)`).
- Disaggregated (2 prefill GPUs + 1 decode GPU): ~3.3 rps/GPU (`min(5.6×2, 10)/3`).
- **~2x goodput improvement from disaggregation alone**, before even optimizing parallelism strategy.

## 5. KV cache transfer overhead is manageable, not a bottleneck
- Moving KV cache from prefill GPU → decode GPU sounds expensive, but with proper placement and fast interconnects (NVLink, PCIe 5.0), the transfer latency is **less than one decode step** — a non-issue in practice.

## 6. "Tailored parallelism" adds further gains
- Beyond just separating hardware, each phase can use a different parallelism strategy (e.g., tensor parallelism for prefill to cut TTFT, different strategy for decode) — optimized independently since they're no longer coupled.

## 7. Real system (DistServe) validates the idea at scale
Across three distinct workloads with different SLO profiles:
| Workload | Latency Profile | Goodput Gain vs. vLLM |
|---|---|---|
| Chatbot | Tight TTFT, Medium TPOT | 2.0x–3.41x |
| Code Completion | Tight TTFT, Tight TPOT | 3.2x (1.5x tighter SLO) |
| Summarization | Loose TTFT, Medium TPOT | 4.48x (10.2x tighter SLO) |

## 8. Disaggregation vs. chunked prefill
- Chunked prefill is good for maximizing raw throughput, but when an application needs **both** TTFT and TPOT satisfied (not a tradeoff between them), **disaggregation wins**.

## 9. Broader impact (confirmed by follow-up industry adoption)
This idea — originally a research bet — has since become **industry standard**: adopted by NVIDIA Dynamo, vLLM, SGLang, TensorRT-LLM, Ray Serve, and others, and TTFT/TPOT are now universal benchmarking metrics across the LLM serving industry.

**Bottom line:** By physically separating two phases with opposite resource profiles, DistServe removes the interference tax that colocated systems pay, letting each GPU specialize and run near its theoretical max efficiency — translating directly into more requests served within strict latency guarantees, at the same or lower hardware cost.

# Section 2: Why Disaggregation Speeds Things Up: Summary

**1. Eliminates forced synchronization between mismatched workloads**
Continuous batching runs one synchronized iteration per GPU — the slowest op in the batch sets the pace for everyone. Prefill (compute-heavy, bursty) and decode (lightweight per-step, but frequent) have opposite profiles, so colocating them means decode constantly waits on prefill's expensive matmul, directly inflating TPOT (and vice versa for TTFT).

**2. Each phase can now run at its own natural hardware ceiling**
- Prefill is **compute-bound** → runs flat-out against the FLOPs/compute roofline.
- Decode is **memory-bandwidth-bound** → runs flat-out against the bandwidth roofline.
On a shared GPU, decode operates below its true ceiling because compute cycles are siphoned off by prefill. Disaggregation lets each phase hit its respective ceiling, uninterrupted.

**3. Removes asymmetric cache interference**
Prefill's large data movement (512 tokens' worth of activations) tends to evict decode's warm KV-cache entries from L2 cache, forcing costly cold reads. Decode's small footprint doesn't meaningfully disrupt prefill in return — so colocation specifically penalizes decode's cache locality. Separating phases removes this one-directional thrashing.

**4. Frees up full memory capacity for KV cache, not just bandwidth**
Decode throughput is gated by *two* ceilings: bandwidth (improved via batching) and **capacity** (KV cache size grows linearly with batch × sequence length, unlike weights which are shared). On a colocated GPU, prefill's activation buffers compete with decode for the same HBM, capping decode's batch size *before* it reaches its bandwidth-driven optimum. A decode-dedicated GPU gives 100% of HBM to KV cache, letting decode batch further before hitting the memory wall.

**5. (Secondary, lower priority) Specialized kernels per phase**
Same attention math, but different computational shape (many-queries-at-once vs. single-query-against-growing-cache) benefits from phase-specific kernel implementations — easier to apply cleanly when phases aren't interleaved in the same batch.

---

**Bottom line:** Colocation forces two workloads with opposite resource demands (compute vs. memory-bandwidth, bursty vs. steady, large-footprint vs. small-footprint) to share one synchronized execution pipeline *and* one memory budget — each phase ends up throttled by the other's needs. Disaggregation removes both forms of contention (execution-time and memory), letting each GPU specialize and run against its own true performance limit.
## Reference
https://haoailab.com/blogs/distserve/
