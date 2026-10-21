# Compression Deep Dive: Parquet vs. ClickHouse

## 1. Parquet Compression Fundamentals

Parquet is a **columnar format** — storing data column-by-column rather than row-by-row. This matters because similar values end up adjacent to each other, making them far easier to compress than mixed row data. Parquet supports several *encodings* (which transform/reshape data) that are applied **before** a general-purpose compressor like ZSTD squeezes out the remaining redundancy.

The most relevant encoding here is **`DELTA_BINARY_PACKED`**, which combines three ideas: **Delta encoding → Zigzag encoding → Bit packing (with Frame of Reference)**.

### Delta Encoding

Instead of storing raw values, store the **difference between each value and the previous one**:

```
Raw:    1000000, 1000001, 1000002, 1000003, 1000004
Delta:  1000000,       1,       1,       1,       1
```

The first value is stored as-is; everything after becomes a small number. This is the foundation for everything else below.

> **Problem:** Deltas can be *negative* (e.g., GPS values can fluctuate up and down), and small negative numbers in standard two's-complement binary look like *huge* numbers (`-1` as a 32-bit int is `0xFFFFFFFF`), which bit-packs terribly.

### Zigzag Encoding — Fixing the Negative Number Problem

Zigzag encoding remaps signed integers into unsigned integers by **interleaving** positive and negative values so that small values (regardless of sign) stay small:

```
Signed delta:    0   -1    1   -2    2   -3    3
Zigzag encoded:  0    1    2    3    4    5    6
```

The mapping rule is roughly: `zigzag(n) = (n << 1) ^ (n >> 31)` — it alternates between encoding positive and negative numbers so the *magnitude* of the result stays proportional to the *magnitude* of the original delta, not its raw bit pattern. This keeps small negative deltas small after encoding, which is essential for the next step (bit packing) to work well.

### Frame of Reference (FOR) + Bit Packing

Once you have small, zigzag-encoded deltas, you don't need a full 32 bits to store each one. **Bit packing** finds the *minimum number of bits* needed to represent the **largest** value in a block, and packs every value using only that many bits.

**Frame of Reference (FOR)** goes one step further: instead of bit-packing the raw deltas, it first subtracts the **minimum value in the block** (the "frame of reference") from every value, so you're packing the *smallest possible range* of numbers.

**Example:**
```
Zigzag deltas in a block: 4, 7, 3, 9, 5
Minimum (FOR):             3
Offsets from FOR:          1, 4, 0, 6, 2   → max value is 6, needs only 3 bits
```
Instead of storing each value with 32 bits, you store: the reference value (`3`) once, plus each offset using just **3 bits** — e.g., `001, 100, 000, 110, 010`. For a block of thousands of rows, this adds up to massive savings.

**Putting it together (Parquet's `DELTA_BINARY_PACKED`):**
```
Raw values → Delta → Zigzag (handle negatives) → FOR (subtract block min) → Bit-pack (use only needed bits) → feed to ZSTD
```

This encoding, combined with ZSTD, is what got the blog's Parquet setup down to **~8.2 bytes/row** (vs. ~28.9 bytes/row with naive Snappy + PLAIN/dictionary encoding).

---

## 2. ClickHouse Compression

ClickHouse achieves even better results (~5.9 bytes/row) using its own codec pipeline: **`Delta`/`DoubleDelta` + `ZSTD`**.

### DoubleDelta — Building on Delta

> *"How does DoubleDelta work, and why does it work?"*

Where `Delta` stores the difference between consecutive values, **`DoubleDelta` stores the difference between consecutive *deltas*** — i.e., it takes a second derivative.

**Example — a vehicle driving on a straight highway:**

Imagine GPS longitude readings as the vehicle moves at constant speed:
```
Raw longitude_e6:     100000, 100010, 100020, 100030, 100040
Delta (1st diff):         —,     10,     10,     10,     10
DoubleDelta (2nd diff):   —,      —,      0,      0,      0
```

Visually: `Delta` stores the blue vectors (point-to-point offsets); `DoubleDelta` stores the *red* vectors — the difference between consecutive offsets. DoubleDelta effectively **predicts** that the next delta will match the previous delta, and only stores the small "correction" when that prediction is wrong. If the vehicle moves at a near-constant speed/direction (as on a highway), that correction is almost always **zero**, which is about as compressible as data gets.

**Where DoubleDelta works well:**
- Regular-interval timestamps (`time_sent` pinging every second)
- Steady, linear motion (highway driving — most of this dataset!)
- Any monotonic counter incrementing by a roughly constant step

**Where it works less well:**
- Stop-and-go city driving, turns, acceleration/deceleration — the "correction" term grows and compression suffers
- Erratic/non-monotonic data in general

**Additional Note :**
- Compression doesn't require values to be identical — it requires them to be small and statistically concentrated. Jitter adds minor spread around zero, but as long as that spread stays narrow relative to the raw data's range, both bit-packing (small bit-width) and ZSTD (skewed value frequency) still compress it very effectively — just not quite as perfectly as the noise-free, constant-velocity case.
- This means even when there is jitter and the diffs in timestamp don't exactly have same interval, the entropy is reduced which compresses well (ZStd compression as in CH). Also needs fewer bits for representation (bit packing in parquet)
### Why Delta/DoubleDelta + ZSTD Works So Well

> *"How does Delta + ZSTD make for good compression?"*

This is a **two-stage pipeline**: the codec (Delta/DoubleDelta) reshapes the data to expose redundancy, then ZSTD (a byte-level, pattern-matching compressor) exploits that redundancy.

- Raw timestamps/coordinates are large, high-entropy numbers with little byte-level repetition — ZSTD can't do much with them directly.
- After Delta/DoubleDelta, most values collapse to small numbers (often `0` or `1`), producing **long runs of repeated bytes** — exactly the pattern ZSTD's match-finding is built to exploit.

ClickHouse's advantage over Parquet here, per the blog's own measurements, is that **`DoubleDelta` avoids straddling bit-packed values across byte boundaries** the way Parquet's `DELTA_BINARY_PACKED` does — keeping cleaner byte-aligned repeating patterns for ZSTD to find. Result: `time_sent` hit **0.404 bytes/row** in ClickHouse vs. **0.976 bytes/row** in tuned Parquet — less than half the size, *despite* using a weaker ZSTD level.

### ZSTD Levels

> *"What are the levels in ZSTD, and what's computationally different between them?"*

ZSTD supports levels **1–22**, trading compression ratio for CPU cost:

- **Low levels (e.g., 1–3):** Use a smaller search window and a fast, greedy match-finding strategy — take the first "good enough" repeated pattern and move on.
- **High levels (e.g., 11–22):** Use a much larger search window and exhaustive/optimal match-finding (comparing many candidate matches, backtracking) — closer to finding the *best possible* compression, at much higher CPU cost.

**Why ClickHouse uses a low level (3) while Parquet used a high level (11):** ClickHouse's `MergeTree` engine continuously **re-merges and recompresses data in the background**, so a high ZSTD level would make every merge expensive and slow down ingestion ("write amplification"). Parquet files, by contrast, are typically written once and read many times — a "compress once, decompress repeatedly" scenario where paying the one-time cost of a high level is worth it. Remarkably, ClickHouse's `ZSTD(3)` + DoubleDelta still *beat* Parquet's `ZSTD(11)` + DELTA_BINARY_PACKED, showing the codec choice mattered more than the raw compression level.

### Gorilla Codec — An Alternative for Noisy Data

> *"What is Gorilla, and how does it compare to DoubleDelta?"*

Gorilla takes a different approach, suited to a different data shape. Instead of assuming values change by a **constant rate** (like DoubleDelta), it assumes consecutive values are simply **very similar** to one another, without necessarily following a clean trend.

It works by taking the **XOR** of consecutive (typically floating-point) values — since similar floats share most of their leading/trailing bits, XOR cancels those shared bits to zero, leaving only the small chunk of bits that actually changed.

**Example intuition:** Daily weight readings like `70.2, 70.3, 70.3, 70.1, 70.4` aren't moving at a constant rate — they jitter up and down. XOR-ing consecutive values exposes just the small "wobble," regardless of direction, which Delta-based codecs (assuming directional/constant change) handle less gracefully.

**DoubleDelta vs. Gorilla — when each wins:**

| Data pattern | Better codec | Why |
|---|---|---|
| Constant-rate motion (highway driving, regular timestamps) | **DoubleDelta** | Second derivative → ~0 |
| Noisy gauge/metric data (CPU %, temperature) bouncing around a value | **Gorilla** | XOR tolerates lack of a consistent trend |
| Erratic, large random jumps | **Neither** | Plain ZSTD may do just as well |

Neither codec is universally better — the right choice depends on matching the codec's assumptions to the actual shape of the data, which is why this blog (and ClickHouse's own docs) stress empirical testing over defaults.

## References
https://tomalard.github.io/posts/clickhouse-outcompresses-parquet-surprises-from-our-migration/
