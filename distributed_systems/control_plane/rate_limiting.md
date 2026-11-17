Based on the ByteByteGo "Design a Rate Limiter" chapter content on this page, here are the key takeaways:

## Why rate limiters matter
- **Prevent DoS attacks** (intentional or unintentional) by blocking excess calls that would otherwise starve resources.
- **Reduce cost** — especially relevant when calling paid third-party APIs billed per-call (credit checks, payments, health record lookups, etc.); limiting unnecessary calls directly saves money.
- **Prevent server overload** — filters out excess traffic caused by bots, retries, or misbehaving clients so backend servers aren't overwhelmed.
- Real-world precedent: large companies enforce rate limits on their public APIs — e.g., Twitter limits tweets to 300 per 3 hours, Google Docs API limits reads to 300 per user per 60 seconds.

## High-level design decisions
- **Where to put the rate limiter** is the central architectural question:
  - **Client-side**: generally discouraged — clients are unreliable/forgeable and outside your control.
  - **Server-side**, embedded directly in the API server.
  - **Middleware approach**: a separate rate limiter component sitting in front of API servers, intercepting requests before they reach backend logic (illustrated via a 2-requests-per-second example where a 3rd request within the same second is blocked and returns **HTTP 429 "Too Many Requests"**).
- In modern cloud/microservices architectures, rate limiting is commonly implemented inside an **API Gateway** — a fully managed middleware layer that also handles SSL termination, authentication, IP whitelisting, and static content serving alongside rate limiting.


| Algorithm | How it works | Best suited for | Who uses it (as commonly cited in this chapter/book) |
|---|---|---|---|
| **Token Bucket** | Bucket per key with max capacity; tokens refill at fixed rate; each request consumes a token; reject if empty. | Allowing short bursts while enforcing a steady average rate; most widely adopted general-purpose algorithm. | Cited as used by **Amazon** and **Stripe** for API rate limiting. |
| **Leaky Bucket** | Requests enter a fixed-size FIFO queue, processed at a constant outflow rate; queue overflow → reject. | Smoothing bursty input into a steady, predictable output rate — good when downstream processing needs even load rather than burst tolerance. | Cited as used by **Shopify** on their API. |
| **Fixed Window Counter** | Counter per fixed time window (e.g., per minute), incremented per request, reset each window. | Simple, cheap to implement; acceptable when occasional boundary double-bursts aren't a serious concern. | Not tied to a specific named company in the chapter; presented as the conceptually simplest baseline. |
| **Sliding Window Log** | Store a timestamp per request (e.g., sorted set); count requests within trailing window; evict stale entries. | High precision rate limiting, avoiding the boundary-burst flaw of fixed windows, at higher memory cost. | Not attributed to a specific named company in the chapter. |
| **Sliding Window Counter** | Weighted combination of current + previous fixed window counts based on how far into the current window you are. | Practical middle ground — much more accurate than fixed window, far less memory than sliding log; this is the one the chapter recommends as the best general trade-off. | Cited as the approach used by **Cloudflare** for rate limiting (this is a widely cited real-world detail associated with this chapter's content). |
| **GCRA (Generic Cell Rate Algorithm)** | Equivalent to token bucket, but stores one value (TAT) per key instead of two; accept if `now ≥ TAT − τ`. | Distributed, Redis-backed limiting where atomicity and minimal per-key storage matter. |  |

# Rate Limiting Algorithms — Implementation Comparison

| Algorithm | Data Stored | Redis Data Structure | Key Operations | Atomicity Mechanism | Memory Footprint | Allows Bursts? |
|---|---|---|---|---|---|---|
| **Token Bucket** | Current token count + last refill timestamp | String (or Hash for 2 fields) | `GET`/`HMGET` → compute elapsed-time refill → `SET`/`HMSET` | Lua script (read-compute-write must be atomic) | O(1) per user | ✅ Yes, up to bucket size |
| **Leaky Bucket (queue-based)** | Actual queued requests | List | `LPUSH` (enqueue) + `RPOP` (leak via worker) + `LLEN` (check capacity) | Natural (List ops atomic) + separate fixed-rate consumer process | O(queue size) per user | ❌ No, strictly smoothed output |
| **Leaky Bucket (counter/meter variant)** | Current "water level" + last leak timestamp | String (or Hash for 2 fields) | `GET`/`HMGET` → compute elapsed-time leak → `SET`/`HMSET` | Lua script | O(1) per user | ❌ No |
| **Fixed Window Counter** | Single counter per time window | String | `INCR` + `EXPIRE` (set once on creation) | Trivial — `INCR` is already atomic | O(1) per user | ⚠️ Boundary burst exploit (effectively yes, unintentionally) |
| **Sliding Window Log** | Timestamp of every individual request in window | Sorted Set (ZSET) | `ZREMRANGEBYSCORE` (evict old) + `ZCARD` (count) + `ZADD` (insert new) | Lua script (all 3 ops must be atomic as a group) | O(number of requests in window) per user | ❌ No — exact rolling enforcement |
| **Sliding Window Counter** | Current window count + previous window count | String (2 keys) | `GET` x2 + `INCR` + weighted-average arithmetic | Lua script (recommended for full atomicity, though simpler ops) | O(1) per user | ⚠️ Slight approximation, minimal boundary leak |

## Key Implementation Themes

| Theme | Algorithms Involved | Why |
|---|---|---|
| **Needs Lua scripting for atomicity** | Token Bucket, Leaky Bucket (meter), Sliding Window Log, Sliding Window Counter (recommended) | Multi-step read-compute-write logic can't rely on a single atomic Redis command |
| **Single atomic command suffices** | Fixed Window Counter | Logic reduces to one `INCR` |
| **Needs Sorted Set (ZSET)** | Sliding Window Log only | Only algorithm that stores *individual* timestamped entries needing range-based eviction/counting |
| **Relies on lazy/on-demand time-based computation (no timers)** | Token Bucket, Leaky Bucket (meter), Sliding Window Counter | Refill/leak/weighting calculated at request time based on elapsed time, not via background jobs |
| **Relies on a literal queue + background consumer** | Leaky Bucket (queue-based) | True traffic-shaping behavior requires an actual processing worker at fixed rate |


## Reference
https://bytebytego.com/courses/system-design-interview/design-a-rate-limiter

Also see : https://github.com/forkedroads/Notes/blob/main/distributed_systems/case_studies/uber_rate_limiter.md
