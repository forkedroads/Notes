
# Socrates: A Write Transaction End-to-End

**Examples: UPDATE and INSERT in Azure SQL Database Hyperscale**

> **Key idea:**
> Separate compute, log, and storage to get high performance, high availability, and massive scale.

---

## What is Socrates?

Socrates is the new SQL Server architecture for the cloud (implemented as Azure SQL Database Hyperscale). It separates compute, transaction log, and storage into independent layers, allowing large databases (100 TB), fast commits (< 0.5 ms), high availability (99.999%), and operations that don't depend on database size (O(1) scale, backup, etc.).

## Key Concepts

1. **Separation of compute, log, and storage** – each tier does one job well.
2. **Durability vs. availability** – log makes commits durable; page servers provide availability.
3. **Log is a first-class service (XLOG)** – only the primary writes the log.
4. **Asynchronous page propagation** – pages are updated after commit.
5. **GetPage@LSN** – ensures you never read a stale page.
6. **Stateless compute and page servers** – data lives in XLOG and XStore.
7. **Avoid O(data) operations** – scale, backup, restore, failover in constant time.

---

## Socrates Architecture (four tiers)

**Applications** (T-SQL, drivers, tools (same as SQL Server))
→ Read/Write → **Compute Tier** (Primary Compute)
⇢ Read-Only → **Compute Tier** (Secondary Compute(s))

### Compute Tier
- **Primary Compute** and **Secondary Compute(s)**
- Query processing, transaction management, concurrency control, security (same as SQL Server)
- Caches pages in memory + local SSD (RBPEX)

Compute → XLOG Service: **Write log (synchronous)**; **Send log (asynchronous)**

### XLOG Service (Transaction Log)
- Primary writes log
- Landing Zone (LZ) for durability (e.g., 3 replicas in Azure storage)
- LogBroker disseminates hardened log to page servers and secondaries
- Page servers pull the log

### Page Servers (Storage Tier)
- Serve pages to compute (GetPage@LSN)
- Apply log asynchronously
- Checkpoint pages to XStore
- Each server owns a partition of the database

### XStore (Azure Storage)
- Durable, scalable, cheap storage
- Stores checkpointed pages and backups
- Not on the commit critical path

---

## Walkthrough 1: UPDATE Transaction

**Example:** `UPDATE Accounts SET balance = 900 WHERE account_id = 42;`

1. **Client sends UPDATE** – The request goes to the Primary Compute.
2. **Get the page (if needed)** – Primary checks its cache. If the page (e.g., P100) isn't in memory/SSD, it requests it from the Page Server using GetPage@LSN.
3. **Modify the page** – SQL Server updates the page in the primary's memory (e.g., balance 1000 → 900).
4. **Generate log records** – Log records describing the change are created (e.g., page P100 updated, transaction T17, LSN 501).
5. **Write log (two paths)**
   - Synchronously to Landing Zone (LZ) for durability.
   - Asynchronously to XLOG (pending area).
6. **Log is hardened** – Once the log is durable in LZ (write quorum), COMMIT can succeed.
7. **XLOG distributes the log** – After hardening, XLOG moves the log to LogBroker and makes it available to the relevant Page Server(s) (and secondaries).
8. **Page Server applies the log (async)** – The Page Server for that partition pulls the log and updates its copy of the page (P100).
9. **Checkpoint to XStore (later)** – Eventually, the updated page is checkpointed to Azure Storage (XStore). This is not on the commit path.

**Flow diagram:**

```
Client
  ↓
Primary Compute
  1. Execute UPDATE
  2. Get page (if needed)
  3. Modify page
  4. Generate log
  5. Write log
  ↓
XLOG (LZ)
  ↓  6. Log hardened (COMMIT)
XLOG (LogBroker)
  ↓
Page Server
  ↓  8. Apply log (async)
XStore
```

---

## Walkthrough 2: INSERT Transaction

**Example:** `INSERT INTO Accounts(account_id, balance) VALUES (43, 500);`

1. **Client sends INSERT** – The request goes to the Primary Compute.
2. **Locate target pages** – SQL Server finds the right index/data pages (e.g., navigate B-tree). If needed, it fetches pages using GetPage@LSN.
3. **Modify page(s)** – The new row is inserted (e.g., into page P101). Index pages may also be updated.
4. **Generate log records** – Log records describing the insert (and any index changes) are created (e.g., transaction T18, LSN 510).
5. **Write log (two paths)**
   - Synchronously to Landing Zone (LZ) for durability.
   - Asynchronously to XLOG (pending area).
6. **Log is hardened** – Once the log is durable in LZ, COMMIT can succeed.
7. **XLOG distributes the log** – After hardening, XLOG makes the log available to the relevant Page Server(s).
8. **Page Server applies the log (async)** – The Page Server updates its pages (e.g., P101 now contains the new row).
9. **Checkpoint to XStore (later)** – Eventually, the updated pages are checkpointed to Azure Storage. Not on the commit path.

**Flow diagram:**

```
Client
  ↓
Primary Compute
  1. Execute INSERT
  2. Locate pages
  3. Modify page(s)
  4. Generate log
  5. Write log
  ↓
XLOG (LZ)
  ↓  6. Log hardened (COMMIT)
XLOG (LogBroker)
  ↓
Page Server
  ↓  8. Apply log (async)
XStore
```

---

## Key Timelines: What's synchronous vs. asynchronous?

### Must happen before COMMIT returns (synchronous)
- ✅ Execute the SQL operation on the Primary
- ✅ Get required page(s) (if not cached) using GetPage@LSN
- ✅ Generate transaction log records
- ✅ Write log to Landing Zone (LZ) and wait for it to be durable

### Happens after COMMIT (asynchronous)
- ✅ XLOG distributes hardened log to Page Server(s)
- ✅ Page Server(s) apply the log and update pages
- ✅ Pages are checkpointed to XStore
- ✅ Secondaries consume the log

---

## Common Questions (and Answers)

**Q: Why can COMMIT succeed before the page is updated on the Page Server?**
A: The durable log is sufficient to reconstruct the change. Pages are updated asynchronously for availability.

**Q: How do we avoid reading a stale page?**
A: The Primary uses GetPage@LSN. The Page Server waits until it has applied the log up to the requested LSN before returning the page.

**Q: What if a page is evicted and the Page Server hasn't caught up yet?**
A: The Primary includes the last known LSN when requesting the page. The Page Server will wait until it has applied that LSN, ensuring you get a fresh version.

---

## Why This Design Works

- **High performance** – Fast commits (log writes only, < 0.5 ms)
- **Massive scale** – Supports 100 TB databases
- **High availability** – 99.999% availability, fast failover
- **Elastic operations** – Scale up/down, create replicas, backup/restore in O(1) time (not proportional to database size)
- **Lower cost** – Fewer full copies, less compute and network resource to keep replicas synchronized

---

**Socrates = Same SQL Server experience, fundamentally new cloud architecture.**
Durable log. Shared pages. Separation of concerns. Built for the cloud.
