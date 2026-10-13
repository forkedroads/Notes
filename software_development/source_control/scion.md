
# Redesigning Source Control at Anthropic — Notes

---

## 1. Motivation

### The core problem
- Claude's ability to write code autonomously removed "writing code" as the bottleneck.
- The new bottleneck: developer infrastructure built for **human-scale** usage — source control, builds, CI.
- Merge volume growth: ~2,500 merges/day (May 2026) → 30,000+ merges/day (Sept 2026), a 10x+ increase in a few months, with no sign of slowing (removing one bottleneck just reveals the next).

### Why git strained
- Git is **decentralized**: every clone is a complete, self-contained repository (full history + ability to compute diffs/blame/checkouts fully offline).
- Anthropic's monorepo at scale:
  - ~100 GB of history on main branch alone
  - ~25 GB compressed working copy
  - ~25 million checkouts/day for CI alone (before counting dev environments, agent sandboxes, research jobs)
  - Naive git traffic at this checkout volume: **~600 PB/day**
- Git's own scaling workarounds were used, but hit limits:
  - **Shallow clones / sparse checkouts / promisor remotes** — reduce data pulled, but operations like diff/blame can break or get awkward on partial clones.
  - **Git mirrors** (several hundred, at peak) — full read-only replicas distributing read load off one canonical server. But: write capacity (merges) is still centralized to one server, and mirrors must continually re-sync; as merge rate rose, keeping mirrors fresh became an operational burden ("harder to operate").
  - **Seeds** (pre-packaged repo snapshots in blob storage) — a static blob you unpack instead of cloning via git protocol. These fell further behind the tip of main as merge velocity increased, since refresh frequency couldn't keep up.

### The key realization
- Nearly all of Anthropic's clients are **agents and automated jobs**, which are online by definition.
- Git's defining strengths (decentralization, offline capability) were *actively working against* scalability for a client population that never needed those properties in the first place.
- Question that triggered the redesign: *"If we designed source control assuming clients are always online, what would it look like?"*

### What they built
- **Scion** — in-house version control system (full monorepo migration from git in 5 months).
- Supporting systems built in parallel: **Antique** (code review surface), code search, a source-control IAM system, new mechanisms for landing large volumes of reviewed code.
- Prior art drawn on: Google's **Piper**, Meta's **Sapling** — both built for extreme-scale monorepos.
- Note: Anthropic has stated Scion is **not** intended as a general-purpose git replacement or public release — it's bespoke to their scale and tightly coupled to their other internal tooling.

---

## 2. Design Principles

### 2.1 Clients are always online
- Git assumes clients may be offline and need a complete local copy to function.
- Scion assumes the server is always reachable; clients fetch data on demand rather than holding a full copy.
- **Why offline-first mattered historically (even for connected users), for context:**
  - Speed (zero network round-trip for local ops like diff/blame/log)
  - Server can stay "dumb"/cheap (all compute is client-side)
  - Resilience to server outages
  - True decentralization (forks, peer-to-peer workflows, no single point of failure for data)
  - Safe local experimentation (rewrite history, branch freely, no network risk)
- **Why these don't matter as much for Anthropic's actual clients:**
  - Agents/CI jobs are inherently networked already (they need connectivity for other reasons, e.g. calling Claude).
  - No decentralized/forking workflow needed — one canonical source of truth.
  - Cost of giving every ephemeral client a 100GB+ full copy far outweighs the marginal speed/resilience benefit.

### 2.2 Client-server asymmetry
- **Git**: every client is a fully capable server — any clone can compute diffs, blame, merges, rebases, serve as a remote.
- **Scion**: deliberate split of responsibilities:
  - **Server**: answers history/blame queries, creates commits, performs merges/rebases — owns all computation requiring full repo knowledge.
  - **Client**: maintains a *bounded* local object cache, lays out the working copy on disk, watches for filesystem changes, provides ergonomic UX — computes very little itself.
- Tradeoff: client **can't** compute blame/rebase on its own (lacks the data) — but in exchange needs far less local storage/data, decoupling client resource needs from total repo size.
- **Analogy**: Git = every employee carries a full photocopy of the entire filing cabinet. Scion = a thin desk client that calls the central records office for anything beyond what's immediately on your desk.

### 2.3 A comprehensive source control service can power novel applications
- **Git's limitation**: tools needing to answer questions like "who last changed this line" or "how are these two commits related" must either maintain a full local clone or depend on vendor-specific (e.g., GitHub) API quirks.
- **Scion's approach**: every query the client can issue is *also* exposed via API — many jobs need no local repository at all.
- **Concrete examples enabled by this:**
  - **Automated bisect** — binary-search for a bad commit via API calls (ask for commits between A/Z, checkout only what's needed, run tests) rather than materializing full working copies repeatedly.
  - **Automated revert** — bot computes diff, checks if revert applies cleanly, finds owners for routing approval — all via API, no local clone.
  - Internal tools (chat-based code Q&A, security/compliance history scans) can be thin, stateless API clients instead of maintaining synced local/mirrored copies.
- Why this matters: at Anthropic's scale, tools built around full local clones (or ad hoc mirrors kept "in sync") face the same staleness/overhead problems discussed above.

### 2.4 Everything must be horizontally scalable
- At peak, several hundred git mirrors were needed — this strained capacity and stopped scaling once the repo outgrew mirrors' disk/memory limits (mirrors are full replicas; every one needs the entire growing dataset).
- Scion: repository data is **distributed**; every component scales independently; **no fully materialized copy of the repo exists anywhere, even on the server**.
- No equivalent of **packfiles** → no repository maintenance needed.
  - *(Packfile refresher: git's compressed, delta-compressed bundles of objects used for efficient storage/transfer — generating/maintaining them is itself nontrivial CPU work that scales poorly with repo size/churn.)*
- **Why Scion scales better (contrast with git mirrors):**
  - Git's horizontal "scaling" = replicate the *entire* dataset across many full-copy nodes → cost and staleness grow with repo size and write velocity.
  - Scion's servers are more like a **large, horizontally-scalable cache layer in front of a real distributed database (Spanner)** — adding capacity means adding more stateless query/cache nodes, not duplicating a 100GB+ dataset.
  - This turns a "data distribution" problem (git's core issue) into a "query/compute scaling" problem — a much more standard, well-understood backend engineering challenge.

### 2.5 Commits must be cheap, incremental, and lock-free
- **Git**: commit locally (instant, no coordination) → push to share (this is where contention/locking on a shared branch ref can occur, especially at high concurrency).
- **Scion**: every commit/rebase/merge happens **on the server** (nothing to push) — but this means these operations must themselves be fast, since there's no "local escape hatch."
- Key constraint: cost of creating a commit (and updating downstream derived data) must scale with **size of the change**, not size of the overall repo.
- **Refs (branches/tags) are discouraged** except for `main` and release branches:
  - A ref is **global mutable shared state** that every server must agree on → inherently a coordination/locking point.
  - Minimizing refs means most work (changesets) can be created concurrently and independently, without contending over a shared pointer.
  - Moving a ref (e.g., `main`) is reduced to a **compare-and-swap on a single Spanner row** — a narrow, well-scoped point of coordination rather than a systemic bottleneck.
- (Debated point from community discussion: this sacrifices git/Jujutsu-style local, offline history-building/experimentation — a deliberate tradeoff given Scion's agent-centric, always-online client base.)

### 2.6 The working copy is optional, and sparse by design
- Because checkouts happen tens of millions of times/day, Anthropic wanted to avoid materializing full working copies whenever possible.
- A CI job needing just a diff between a commit and its merge base can **ask the server directly** — no checkout needed at all.
- Many *write* workloads can also skip checkout entirely and operate purely via API.
- When files are needed on disk:
  - **Sparse checkouts** — only contain the directories the client actually asks for.
  - **Virtualized checkouts** — materialize files on demand (lazily), so a job only "pays" for what it actually reads.

---

## 3. Implementation

### 3.1 Git's underlying data model (for contrast)
- Git objects (all content-addressed by SHA hash):
  - **Blob** — raw file content
  - **Tree** — directory listing (maps names → blob/tree hashes)
  - **Commit** — pointer to one root tree (**full snapshot** of the whole project) + parent pointer(s) + metadata
  - **Tag** — named pointer to a commit
- **Git stores full snapshots, not diffs**, as its logical model. `git diff`/`git blame` are **computed on demand** by walking/comparing trees and blobs — nothing is pre-computed.
- **Packfiles** are a separate, purely physical storage optimization:
  - Many loose objects bundled together; similar objects stored as a base + small delta, then zlib-compressed.
  - This is **invisible** to higher-level commands — reconstructing a blob's full content (undoing delta compression) happens transparently in git's object-reading layer *before* any diff/blame logic runs.
  - **Two-step process, always:**
    1. **Reconstruction** (storage layer) — resolve any delta chains back into full original content.
    2. **Computation** (command layer) — run the actual diff algorithm (e.g. Myers diff) or blame-walk on the fully reconstructed snapshots.
  - Packfile delta pairing is based on storage efficiency heuristics, **unrelated** to the specific two commits you might later diff — every diff/blame query is computed fresh regardless of how data happened to be packed.

Understood — I'll avoid tables and nested code fences this time, using plain bullets instead, which should render cleanly.Apologies for that odd tool call — that was unnecessary. Let me just give you the clean markdown directly, avoiding tables and nested code fences this time so it renders properly.

---

## 3.2 Scion's data model: the "changeset"

- Unit of work = a **changeset**, an entry in a write-ahead log.
- Unlike a git commit (which stores a pointer to a full tree), a changeset stores:
  - A **list of files that changed**, relative to its first parent
  - **Content-addressed pointers** to the new/compressed file contents in blob storage
- Changesets are created **server-side** (consistent with the client-server asymmetry principle — clients don't hold enough local state to construct a full snapshot themselves) and are available everywhere instantly.
- This design enables very high commit throughput: no packfiles to assemble, no locking or cross-server coordination, and no refs to move for most operations.
- Refs are technically supported, but moving one (e.g., `main`) is just a compare-and-swap on a single Spanner row, decoupled from changeset creation itself.

**Two independent axes worth separating:**

- **What is stored (data model):**
  - Git → full snapshot per commit (pointer to a complete tree)
  - Scion → just the diff (changeset = files changed vs. parent)
- **Where it's created (location):**
  - Git → locally, on the client machine, no network required
  - Scion → on the server, as part of client-server asymmetry

- These two choices reinforce each other: since the Scion client only keeps a *bounded* local cache (not the full repo history), it genuinely can't construct "here's my full new snapshot" the way a git client can. So instead, the client simply reports "I changed these specific files, here's their new content," and the server — which has full context — records the changeset and derives everything else from it.

---

## 3.3 Derived data — answering snapshot/history questions from diffs

**The problem this solves:** A changeset only records "what changed." On its own, it can't answer:
- "What does the full file tree look like at this changeset?" (needed to materialize/checkout files)
- "Who last touched this line?" (blame)
- "Which commits touched this file?" (path history)
- "Is commit A an ancestor of commit B?" (needed for merge-base/rebase)

**The solution: "derived data," defined recursively**

> *"Each type of derived data is defined recursively, so its value for a changeset is computed from its values for that changeset's parents."*

- **Tree reconstruction (snapshot) as derived data:**
  tree(N) = tree(parent(N)), with N's changed files patched in.
  Cost scales with the size of the *change*, not the size of the repo — matches the "cheap, incremental" commit principle.

- **Blame as derived data:**
  blame(N) for file.py = blame(parent(N)) for file.py, updated only for the lines N's diff touched. If N didn't touch the file at all, blame(N) is identical to blame(parent(N)).

- **Path history as derived data:**
  path_history(N) for file.py = path_history(parent(N)) for file.py, plus N itself if N's diff includes file.py.

- **Skip lists** for the commit graph — enable fast ancestry queries (e.g., merge-base) by jumping backward through history in large strides instead of walking commit-by-commit.

**Why recursion alone isn't enough — caching makes it tractable:**
- Without caching, computing tree(N) would require recomputing tree(parent(N)), tree(grandparent(N)), and so on back to the very first commit — as expensive as git, possibly worse.
- The recursive definition only pays off because **each result, once computed, is stored permanently** — safe forever, since content-addressed data is immutable (no cache invalidation problem, ever).
- This is classic **memoization / dynamic programming**: check if the value is already stored; if so, return it cheaply. If not, recursively fetch/compute the parent's value, apply a small patch for this changeset's diff, store the result, and return it.
- **Hot path** (recent history, near the tip of main): the parent's value is almost always already cached from someone else's recent request → cheap lookup + small patch.
- **Cold path** (old/obscure history): the server walks back recursively until it finds a cached base case, then rolls forward, computing and caching each intermediate result along the way (warming the cache for next time).
- Results are written back to **Spanner** (the authoritative persistent store) as they're computed.
- Derived data computation can be **eager** (computed immediately when a changeset is created — useful for high-value data like the tree) or **deferred/lazy** (computed only on first request, then cached) — a tunable choice per data type.
- New derived data types (e.g., copy detection, path-based search) can be added without changing the client, since this is all server-side machinery sitting behind the same API.

---

## 3.4 Server architecture and caching hierarchy

**Not classic key-based sharding — more like a cache layer in front of Spanner:**

> *"A small fleet of Scion servers provides source control APIs... All source control data is content-addressed, so a piece of data never changes once it's stored, and any server can cache it without checking with the others... They are essentially large caches."*

- Because data is immutable and content-addressed, **any server can cache any object with zero coordination** — there's no "ownership" of specific data by specific shards.
- **Spanner** (Google's globally distributed, strongly consistent database) handles actual data partitioning/replication internally — Scion doesn't need to build its own shard-routing logic.
- Scion's own engineering focus is the caching/derived-data layer sitting on top of Spanner, not solving distributed storage from scratch.

**Three-tier caching hierarchy:**
- `scn` client (on the user/job's machine)
- → `scion-daemon` (per-node/machine; stores objects for all jobs running on that node)
- → `scion-cache` (regional; streams objects from memory)
- → `scion-server` (authoritative; reads from Spanner / cloud storage on a miss)

- Each tier is a strict waterfall — a request only climbs to the next tier on a cache miss.
- **Strict tiering prevents cascading failures**: a failing or overloaded regional cache can't flood the authoritative tier above it.
- Each tier reportedly maintains a **>99% cache hit rate** and holds **no long-lived state** — trivial to scale horizontally (just add more nodes, no rebalancing or handoff needed).

**Remaining distributed-systems concerns, scoped narrowly:**
- **Cold-path cache-miss cascades**: recursive walk-back for derived data on rarely-touched history can require multiple hops before hitting a cached base case.
- **Write coordination**: creating a changeset needs a consistent view of its parent; moving a ref (e.g., `main`) requires a compare-and-swap on a single Spanner row — the one narrow point of real coordination in the system, which is why refs are minimized elsewhere.

---

## 3.5 Fetch protocol — why it's cache-friendly

- **Git's fetch/clone**: involves a stateful negotiation ("here's what I have, compute me a custom packfile diff") — hard to cache since the response is session-specific.
- **Scion's `scn` client protocol:**
  > *"Scion's negotiation doesn't tell the server what the client already has; instead, it operates exclusively on trees and blobs the client wants. Starting from a root tree, the client checks its local store for missing trees and requests them recursively, then repeats the process for blobs."*
  - Every request is for a **specific, immutable object** → the response is always identical regardless of who's asking → trivially cacheable at every tier, just simple key-to-value lookups.
  - No giant multi-gigabyte packfile response — requests are small, resumable, and efficient.
  - Sparse and virtualized checkouts need no special protocol support — the client simply never asks for trees/blobs it doesn't want.

---

## 3.6 Why caching + recursion efficiency works well in practice

- **Immutability removes invalidation entirely** — the hardest caching problem is designed away from the start.
- **Fine-grained, universal object requests** — no per-client custom computation, just simple lookups.
- **Stateless cache nodes** — trivial horizontal scaling.
- **Natural workload locality** — most traffic concentrates near the tip of `main` (recent, active history), so recursive derived-data lookups almost always resolve in one hop, since the parent's value was very likely just computed by someone else's recent request.
- **Protective tiering** — isolates failures and overload between layers.

---

## 3.7 Reported results

- Infrastructure footprint: **~200x smaller** (CPU/memory) than the previous git mirroring setup.
- CI/research job checkouts: **two orders of magnitude faster** (seconds vs. minutes).
- CI alone realized a **25% CPU capacity win**, freed up fleet-wide.
- No repository maintenance needed (no packfiles, no mirror-sync operations).
- A Claude-driven rewrite of the git-to-Scion ingestion pipeline used **20x less memory** and ran **4x faster** than the original hand-written prototype.

---

## 3.8 Related process changes (noted in broader reporting)

- A traditional **merge queue with a global lock doesn't scale** at agent volume — Anthropic shifted to **post-merge and pre-deploy automated testing**, accepting that concurrent changes can conflict (reasoning: Claude can usually fix conflicts faster than humans can even detect them).
- Code review shifted from "read every line" to **reviewing intent**, aided by tooling (Antique) that distinguishes trivial changes (e.g., comment-only edits) from sensitive logic changes.
- Fine-grained **access control** (per-repository, per-directory) preserves monorepo benefits while protecting sensitive code.
