Here are my notes from the article on how cloudflare saved a lot of memory by otimizing the storage schema used in the 1.1.1.1 service

# Key Takeaways: General Principles Behind the Memory Savings

## 1. At massive scale, small per-item costs become huge aggregate costs
With 250+ billion cache entries, wasting even a single byte per entry costs 250+ gigabytes fleet-wide. This is the foundational mindset shift: optimizations that look trivial in isolation ("save 8 bytes") are worth serious engineering effort when multiplied across a large enough denominator.

## 2. Don't pay for flexibility you don't need
Generic, "growable" data structures (like resizable lists/strings) carry overhead — extra bookkeeping fields, over-allocated space for future growth — to support mutation. But once data is written to a cache, it's *never modified again*. Recognizing that a structure is effectively **write-once, read-many** means you can swap to fixed-size, tightly-allocated representations and simply drop the machinery meant for growth.

## 3. Don't store what you can cheaply recompute or reconstruct from context you already have
The "owner name" optimization is a great example: rather than storing a piece of data (the record's owner name) redundantly on every record, they recognized it's usually identical to information *already available elsewhere* (the cache key) at the point of use. Only store the exception (when it actually differs, e.g. CNAMEs); reconstruct the common case for free at read time. This is a general database/caching principle: normalize away redundant copies of derivable data.

## 4. Group related data into single contiguous allocations instead of scattering it
Every separate heap allocation costs both allocator overhead (rounding to size classes, bookkeeping) and CPU cache-locality/performance (pointer chasing). Combining multiple lists into one buffer, or multiple small records into one contiguous byte blob, reduces both the *number* of allocations and improves how efficiently the CPU can read through the data sequentially.

## 5. There's a fundamental tradeoff between "convenient structured form" and "compact raw form" — and you can choose per use case
Parsed, structured representations (typed enums, structs) are easy and fast to *manipulate* in code, but carry memory overhead (type tags, padding, pointers). Raw wire-format bytes are compact and fast to *transmit/copy*, but are harder to manipulate (must be parsed/walked sequentially, can't randomly index). The insight here was recognizing that for a cache — where the main job is "store it, then later spit it back out mostly unchanged" — the wire-format side of that tradeoff wins for most data, since you avoid both extra memory **and** redundant re-serialization work.

## 6. Measure the real layout, don't assume the "obvious" design is efficient
The improvements came from actually inspecting the in-memory size/layout of existing structures and finding that idiomatic, "default" choices (growable vectors, big enums sized to their largest variant, three separate lists instead of one) were quietly paying costs nobody was using. This is a broader principle: default/idiomatic patterns optimize for general-purpose flexibility, not for a specific workload's actual access patterns — profiling reveals the gap.

## 7. Know your workload's access pattern and skew, and optimize for the common case
Several wins came from exploiting *statistical* properties of the real traffic: most records don't need DNSSEC, most records' owner name equals the query name, most record types (A/AAAA/TXT) are small and don't need re-serialization, and the "rare, large" variants (like NAPTR) are uncommon enough that boxing them individually doesn't hurt overall footprint. Optimizing for the dominant case while keeping a (slightly less optimal) fallback path for the rare case is a broadly reusable strategy.

## 8. Memory efficiency and performance aren't always in tension — sometimes the same change improves both
Normally you'd expect a memory/speed tradeoff, but here, reducing allocations and improving locality *simultaneously* reduced memory footprint AND improved throughput/latency, because fewer allocations and better cache locality are wins on both axes at once. This suggests that many "memory-optimization" tasks are really "removing accidental waste" rather than "trading space for speed" — genuinely free wins rather than compromises.

## 9. Context-specific keying (like ECS) can silently multiply your data volume — and that multiplication effect makes per-item optimizations even more valuable
When something makes "the same logical query" fan out into many cached variants, every per-entry byte you save gets multiplied by however many variants exist. Recognizing this multiplier effect helps prioritize *which* optimizations matter most and *where* (e.g., ECS-heavy locations benefit disproportionately).


## Refernces
https://blog.cloudflare.com/dns-cache-memory-optimization-1111/
