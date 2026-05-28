# 03 — PostgreSQL (25 Questions)

Postgres is the workhorse of your stack — Tibil's monolith, the multi-tenant SaaS, the IoT pipeline's time-series storage. These cover indexing internals, query planning, MVCC, isolation, schema design, and production ops.

**Sections**
- Part A — Indexing & query performance (Q1–Q8)
- Part B — Transactions & concurrency (Q9–Q14)
- Part C — Schema design & advanced (Q15–Q20)
- Part D — Operations (Q21–Q25)

---

## Part A — Indexing & Query Performance

### Q1. What index types does Postgres support and when do you use each?

**Answer:**
Postgres ships with five main index types, each optimized for a specific query shape:

- **B-tree** — the default. Balanced tree, supports `=`, `<`, `>`, `BETWEEN`, `IN`, `IS NULL`, `LIKE 'prefix%'`. Works on any orderable type. 90% of indexes you'll ever create.
- **Hash** — equality only (`=`). Smaller and faster than B-tree for pure lookup, but useless for ranges. Pre-PG10 it wasn't WAL-logged (crash-unsafe), so most teams just used B-tree everywhere. Now safe but rarely chosen.
- **GIN (Generalized Inverted Index)** — for "many values per row" cases: arrays (`@>`, `&&`), JSONB containment (`@>`), and full-text search (`tsvector`). Slow to update, fast to query.
- **GiST (Generalized Search Tree)** — for geometric data, ranges, and proximity searches (PostGIS uses it heavily). Good for "find rows near this point" or "find ranges overlapping this range."
- **BRIN (Block Range Index)** — extremely compact summary index. Stores min/max per block range. Useful only on physically clustered data (time-series tables where rows are appended in order). Index for a 1TB table can be a few MB.
- **SP-GiST** — partitioned space (non-balanced trees) for non-uniform data like IP addresses (`inet`). Niche.

For your IoT telemetry table appended in timestamp order, BRIN on `(recorded_at)` gives you nearly free range filtering. For the SaaS notifications query "give me all notifications tagged with X," GIN on a `tags text[]` column.

**Code:**
```sql
-- Standard B-tree (composite, your dashboard query)
CREATE INDEX users_tenant_email_idx ON users (tenant_id, email);

-- GIN for JSONB containment
CREATE INDEX events_props_gin ON events USING gin (props);
SELECT * FROM events WHERE props @> '{"action": "login"}';

-- BRIN on a time-series append-only table
CREATE INDEX telemetry_recorded_brin
  ON telemetry USING brin (recorded_at) WITH (pages_per_range = 32);
```

**Follow-up 1: When would I use GIN over B-tree on a JSONB column?**
B-tree on a whole JSONB column is mostly useless — Postgres can compare JSONB for equality but not for containment, and you almost never query "find rows where props equals this exact JSON." GIN supports `@>` (contains), `?` (key exists), `?&` / `?|` (all/any keys exist) — the operations you actually want. Trade-off: GIN indexes are large and slow to update; don't create one on a write-heavy column unless you actually query containment.

**Follow-up 2: How do I know which index Postgres is using?**
`EXPLAIN ANALYZE <query>`. Look for `Index Scan using <name>` or `Bitmap Index Scan using <name>`. If you see `Seq Scan`, your index isn't being used — either it doesn't fit the predicate, the planner thinks a seq scan is cheaper (small table), or the query is using a function/cast that makes the index non-applicable (e.g., `WHERE lower(email) = ...` won't use an index on `email`).

---

### Q2. How do composite (multi-column) indexes work and why does column order matter?

**Answer:**
A composite index sorts rows by the first column, then by the second within ties, and so on. This means the index is useful for queries that filter on a **prefix** of the column list, in order.

Index on `(a, b, c)` helps:
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`
- `WHERE a = ? AND b > ?` (range on b, equality on a)

It does **not** help (or only marginally):
- `WHERE b = ?` (skipped `a`)
- `WHERE c = ?`
- `WHERE a > ? AND b = ?` (after a range on `a`, the `b` ordering inside the index is no longer useful)

**Rule of thumb:** order columns by selectivity for equality predicates, then put the range column last.

**Code — finding the right index for the right query:**
```sql
-- A common multi-tenant query: list this tenant's active users by email prefix
SELECT * FROM users
WHERE tenant_id = $1 AND status = 'active' AND email LIKE 'a%';

-- Right index: (tenant_id, status, email)  — equalities first, prefix-match column last
CREATE INDEX users_tenant_status_email
  ON users (tenant_id, status, email);
```

**Follow-up 1: When does an "out-of-order" index column still help?**
Bitmap index scans can combine *multiple* indexes via `BitmapAnd` / `BitmapOr`. So with separate indexes on `a` and `b`, Postgres can intersect them for `WHERE a = ? AND b = ?`. Slower than a single composite hitting both, but workable for ad-hoc analytical queries. For your high-traffic dashboard endpoint, a single well-ordered composite always wins.

**Follow-up 2: What about `INCLUDE` columns?**
`CREATE INDEX ... (a, b) INCLUDE (c, d)` adds `c` and `d` to the index leaf pages without making them part of the sort key. This enables index-only scans for queries that filter on `a, b` and only read `c, d`. The index doesn't help filter by `c` or `d`, but it avoids the heap lookup. Great for read-heavy hot paths.

---

### Q3. What are partial indexes and when should you use them?

**Answer:**
A partial index is an index that covers only the rows matching a `WHERE` clause. The index is smaller, faster to scan, and cheaper to maintain because most rows are excluded.

Classic uses:
1. **Skewed boolean columns** — `is_deleted = false` is true for 99% of rows. Index just the rare ones.
2. **Status flags with one hot value** — your job queue has a `status` column; you only ever query `pending`. Index `WHERE status = 'pending'`.
3. **Soft-deletion filters** — every query has `AND deleted_at IS NULL`. Make it part of the index condition.
4. **Multi-tenant data with archived tenants** — index only `WHERE tenant_active = true`.

**Code — your BullMQ-equivalent job queue:**
```sql
CREATE TABLE jobs (
  id          uuid PRIMARY KEY,
  queue       text NOT NULL,
  status      text NOT NULL,     -- 'pending', 'running', 'done', 'failed'
  scheduled_for timestamptz NOT NULL,
  payload     jsonb
);

-- Most jobs are 'done'. Workers only ever query for 'pending'.
CREATE INDEX jobs_pending_idx
  ON jobs (queue, scheduled_for)
  WHERE status = 'pending';

-- A 100M-row table with 99% 'done' jobs gets a tiny ~1M-row index.
-- Worker query — uses the partial index:
SELECT * FROM jobs
WHERE queue = 'emails' AND status = 'pending' AND scheduled_for <= now()
ORDER BY scheduled_for LIMIT 10;
```

**Follow-up 1: Why is a partial index sometimes faster than a regular index?**
Two reasons: (1) smaller — fewer pages in the buffer cache, fewer levels in the B-tree, fewer disk reads to traverse; (2) more focused — the planner knows that scanning this index already filters to the rows you want, often eliminating a re-check. Bonus: writes to rows that don't match the predicate skip the index entirely, which is huge for hot tables.

**Follow-up 2: What's a common partial-index footgun?**
The query's `WHERE` clause must exactly match the index predicate for the planner to use it. `WHERE status = 'pending'` matches an index `WHERE status = 'pending'`. But `WHERE status IN ('pending', 'queued')` doesn't, even though `'pending'` is one of the values. The planner is conservative — make sure to query with the same shape.

---

### Q4. What's an index-only scan, and how do covering indexes enable it?

**Answer:**
A regular index scan walks the index to find pointers (TIDs) to heap pages, then fetches each row from the heap to read the requested columns. The heap lookup is the expensive part — random I/O.

An **index-only scan** answers the query entirely from the index, without touching the heap. Postgres can do this when:
1. Every column the query *needs* (in SELECT, WHERE, ORDER BY) is in the index.
2. The visibility map says the heap pages involved are "all visible" (no concurrent transactions could see different data).

Condition #2 means index-only scans require an up-to-date visibility map, which `VACUUM` maintains. After many updates, the visibility map gets stale and index-only scans degrade.

**Covering indexes** are the way you make index-only scans possible. The `INCLUDE` clause adds non-key columns to the leaf level so the data is available without a heap fetch.

**Code:**
```sql
-- Query pattern: lookup user by id, return name and email
SELECT name, email FROM users WHERE id = $1;

-- Plain B-tree on id requires heap fetch for name/email
CREATE INDEX users_id_idx ON users (id);                           -- two-step

-- Covering index lets it stay in the index
CREATE INDEX users_id_with_data_idx ON users (id) INCLUDE (name, email);
-- EXPLAIN now shows "Index Only Scan"
```

**Follow-up 1: When does an index-only scan *not* happen even though it should?**
Stale visibility map. Run `VACUUM users` (or wait for autovacuum) and try again. You can check the visibility map state via `pg_visibility` extension. Heavy update workloads keep the map dirty; index-only scans become unreliable. For hot tables, plan around regular updates rather than relying on index-only scans.

**Follow-up 2: Why use `INCLUDE` instead of just adding the column to the index key?**
Two reasons: (1) it doesn't affect sort order — useful columns that aren't queryable as range predicates shouldn't bloat the sort tree; (2) it doesn't add to the unique constraint if the index is unique. `CREATE UNIQUE INDEX ... (email) INCLUDE (name)` is unique on email and carries name along for index-only scans without making `(email, name)` the uniqueness constraint.

---

### Q5. How do you read an EXPLAIN ANALYZE output?

**Answer:**
`EXPLAIN ANALYZE` runs the query and reports the actual execution plan with timings. The output is a tree — read top-to-bottom, but the leaves execute first, results flow upward.

Key things to look for:
- **Node type** — `Seq Scan`, `Index Scan`, `Bitmap Heap Scan`, `Hash Join`, `Nested Loop`, etc.
- **Cost** — `cost=startup..total`. Planner's estimate, in arbitrary units. Useful for *relative* comparison.
- **Actual time** — `actual time=startup..total ms`. Real execution.
- **Rows** — `rows=N` planner estimate vs `actual rows=M`. **Big mismatches indicate bad statistics** — run `ANALYZE`.
- **Loops** — for nested loop joins, the inner side is executed `loops=N` times; multiply by per-loop time.
- **Buffers** (with `BUFFERS true`) — `shared hit=X read=Y` shows cache hits vs disk reads.

**Code:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, o.total
FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.tenant_id = $1 AND o.created_at >= now() - interval '7 days';
```

A healthy plan you might see:
```
Nested Loop  (cost=0.85..125.42 rows=20 width=40) (actual time=0.04..1.23 ms rows=18 loops=1)
  Buffers: shared hit=82
  ->  Index Scan using users_tenant_idx on users u
        Index Cond: (tenant_id = $1)
        (actual rows=200 loops=1)
  ->  Index Scan using orders_user_created_idx on orders o
        Index Cond: ((user_id = u.id) AND (created_at >= ...))
        (actual rows=1 loops=200)
Planning Time: 0.5 ms
Execution Time: 1.4 ms
```

**Follow-up 1: What does a row estimate that's wildly off tell me?**
That the planner's statistics are stale or insufficient. Possible fixes: (1) `ANALYZE table_name` to refresh stats, (2) `ALTER TABLE table_name ALTER COLUMN col SET STATISTICS 1000` to sample more values for that column, (3) `CREATE STATISTICS` for cross-column correlations the default stats don't capture. Bad estimates → wrong plan choice (e.g., nested loop when it should hash join) → catastrophic slowdowns.

**Follow-up 2: How do `EXPLAIN`, `EXPLAIN ANALYZE`, and `EXPLAIN (ANALYZE, BUFFERS)` differ?**
`EXPLAIN` just prints the planner's chosen plan and estimates — fast and side-effect-free. `EXPLAIN ANALYZE` actually executes the query, so timings are real (and any side effects — INSERTs, UPDATEs — happen; wrap in a transaction and ROLLBACK to be safe). `BUFFERS` adds I/O counts so you can see cache hits vs disk reads — essential when diagnosing why a query is slow on cold cache.

---

### Q6. Explain the major join algorithms and when Postgres picks each.

**Answer:**
Postgres has three join algorithms:

- **Nested Loop** — for each row in the outer, scan the inner. Picks when the outer has very few rows and there's an index on the inner. Catastrophic if either side is large without an index (it becomes O(N×M)).
- **Hash Join** — build a hash table on the smaller side (typically inner), then probe it once for each row of the outer. Requires equality join predicate. Excellent for big-to-big joins. Memory-bound — if the hash doesn't fit in `work_mem`, spills to disk and gets slow.
- **Merge Join** — both sides sorted on the join key, then merged like sorted lists. Linear in total rows once sorted. Best when both sides are already sorted (often via index) or when one side is small enough to sort cheaply.

**Code — predicting the join:**
```sql
-- Small outer (filtered by tenant), inner has good index — Nested Loop
SELECT u.name, p.title
FROM users u JOIN posts p ON p.user_id = u.id
WHERE u.tenant_id = $1;       -- filter cuts users to ~200

-- Big to big, equality — Hash Join
SELECT u.id, COUNT(*)
FROM users u JOIN events e ON e.user_id = u.id
GROUP BY u.id;
```

**Follow-up 1: How can I force a different join type?**
You can hint with config settings — `SET enable_nestloop = off`, `SET enable_hashjoin = off`, `SET enable_mergejoin = off` — usually for diagnostics. In production, fix the root cause: better indexes, more accurate statistics, or rewriting the query. Postgres doesn't have query hints like Oracle/MSSQL by design; the planner is supposed to be smart enough given good stats.

**Follow-up 2: What is `work_mem` and how does it affect joins?**
`work_mem` is the memory budget per sort/hash *operation*. A query with multiple sorts/joins can use multiple times `work_mem`. If a hash join doesn't fit in `work_mem`, it spills batches to disk — orders of magnitude slower. Tune it carefully: set high globally and a query with 10 sorts can OOM the box; set low and OLAP queries crawl. Typical pattern: low global default (e.g., 4MB), bumped per-session for analytical queries via `SET LOCAL work_mem = '256MB';`.

---

### Q7. How does `pg_stat_statements` help find slow queries in production?

**Answer:**
`pg_stat_statements` is an extension that aggregates execution stats for every distinct query shape. Postgres replaces literal values with placeholders (`$1`, `$2`) and groups by the normalized query string. For each, you get:
- `calls` — how many times executed.
- `total_exec_time`, `mean_exec_time`, `min_exec_time`, `max_exec_time`, `stddev_exec_time`.
- `rows` — total rows returned/affected.
- `shared_blks_hit`, `shared_blks_read` — cache hits vs disk reads.

This is **the** tool for finding production hotspots. Don't profile what you think is slow — measure what's actually slow.

**Code — installation and the top-10 query:**
```sql
-- Once, by a superuser
CREATE EXTENSION pg_stat_statements;

-- In postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'

-- Top 10 by total time
SELECT
  substring(query, 1, 100) AS short,
  calls,
  round(total_exec_time::numeric, 0) AS total_ms,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

What you do with the output:
1. **Top by `total_exec_time`** — the query that, summed over all executions, eats the most CPU. Often a fast-but-frequent query.
2. **Top by `mean_exec_time`** — the slowest individual executions. Often a missing index.
3. **Top by `rows / calls`** — queries returning huge result sets, often a forgotten `LIMIT` or a "fetch all" anti-pattern.

**Follow-up 1: What did "Hunted N+1 queries via pg_stat_statements" on your resume actually look like?**
The dashboard endpoint showed 14 rows in `pg_stat_statements` like `SELECT * FROM posts WHERE user_id = $1`, each with `calls=14000`, total time ~3000ms. Same query repeating per-user inside a loop. Fix: replace with `WHERE user_id = ANY($1)` and bulk-load all posts in one query, then attach in application code. After: 1 call, ~50ms.

**Follow-up 2: What's the overhead of `pg_stat_statements`?**
A few percent CPU and a fixed-size hash table in shared memory (`pg_stat_statements.max`, default 5000 distinct queries). Effectively free in production. Reset stats periodically (`SELECT pg_stat_statements_reset();`) if you want to look at "since last reset" instead of "since restart."

---

### Q8. What is index bloat, and how do you fix it without taking the table offline?

**Answer:**
B-tree indexes in Postgres don't reclaim space when rows are deleted or updated. Pages with mostly-dead tuples take up disk and buffer cache for nothing. Over months of writes, an index can be 3x its useful size. Symptoms: queries get slower, disk usage grows, vacuum takes longer.

Three ways to fix it:

1. **`REINDEX` (blocking)** — rebuilds the index from scratch. Holds an `ACCESS EXCLUSIVE` lock — no queries until done. Fine for maintenance windows; unacceptable for hot tables.
2. **`REINDEX CONCURRENTLY` (PG12+)** — builds a new index in the background, swaps, drops the old. Doesn't block reads or writes. Same as `CREATE INDEX CONCURRENTLY` + drop dance you used to do manually.
3. **`pg_repack`** — external tool. Builds a new copy of the table without bloat, swaps. Useful for table bloat (not just index bloat).

**Code:**
```sql
-- Estimate bloat (rough heuristic — pg_stat_user_indexes has more accurate variants)
SELECT
  schemaname, tablename, indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size,
  idx_scan
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- The non-blocking way
REINDEX INDEX CONCURRENTLY users_email_idx;
```

**Follow-up 1: How do you tell if an index is bloated vs just large?**
Compare actual size to expected size. Extensions like `pgstattuple` give precise bloat percentages: `SELECT * FROM pgstattuple_approx('users_email_idx');` reports dead-tuple percent. A useful heuristic: index is bigger than `(rows * (avg key size + 12))` by more than 2x → bloat.

**Follow-up 2: Why does PG generate index bloat at all?**
MVCC. When you `UPDATE` a row, Postgres writes a new row version, marks the old one dead, and adds a new index entry pointing to the new TID. The old index entry is now dead. Autovacuum cleans up dead heap tuples but cannot reclaim B-tree pages mid-tree easily — only when a leaf becomes entirely empty. Workloads with heavy updates churn through index pages faster than vacuum can reclaim them.

---

## Part B — Transactions & Concurrency

### Q9. What does ACID actually mean? Walk me through each letter with examples.

**Answer:**

**A — Atomicity.** A transaction is all-or-nothing. If you `BEGIN; INSERT; INSERT; COMMIT`, either both inserts persist or neither does. On `ROLLBACK` or crash, partial work is undone. Postgres implements this via WAL (write-ahead log) and the transaction ID system.

**C — Consistency.** The database enforces declared invariants (constraints, foreign keys, check constraints) at transaction boundaries. A transaction that would violate a constraint is rejected, leaving the DB in a valid state. Note: "consistency" here is integrity-level (constraints hold), not the same as the "C" in CAP theorem.

**I — Isolation.** Concurrent transactions don't see each other's uncommitted changes. The degree of isolation is configurable (next question). True serializability means the outcome is equivalent to *some* serial execution of the same transactions.

**D — Durability.** Once `COMMIT` returns success, the data survives crash or power loss. Postgres flushes WAL records to disk on commit (configurable via `synchronous_commit`). The `fsync` and durable storage matter — disabling them trades durability for speed.

**Code — atomicity in action:**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- crash here? both updates are rolled back automatically
COMMIT;
```

**Follow-up 1: What does `synchronous_commit = off` give up?**
The promise that a returned `COMMIT` means data is on disk. With it off, COMMIT returns as soon as WAL is in OS buffers but before fsync — much faster, but if the OS crashes within a few hundred ms, recently committed transactions can disappear. Useful for non-critical bulk loads, dangerous for financial data. Per-session toggle is safer than global.

**Follow-up 2: How does PG ensure atomicity if the process crashes mid-transaction?**
WAL. Every change is written to the write-ahead log before being applied to data files. On restart, Postgres replays the WAL up to the last `COMMIT` record and discards anything after — the half-finished transaction's WAL entries are simply not applied. The data files might have partial writes, but the WAL is the authoritative source until replay catches up.

---

### Q10. Explain the SQL isolation levels. Which does Postgres default to?

**Answer:**
Four standard isolation levels, defined by which concurrency anomalies they allow:

| Level | Dirty reads | Non-repeatable reads | Phantom reads | Serialization anomalies |
|-------|-------------|----------------------|---------------|--------------------------|
| Read Uncommitted | possible | possible | possible | possible |
| Read Committed | no | possible | possible | possible |
| Repeatable Read | no | no | possible (SQL standard) | possible |
| Serializable | no | no | no | no |

**Postgres specifics:**
- Default is **Read Committed**.
- Postgres has **no Read Uncommitted** — it silently maps to Read Committed.
- **Repeatable Read in PG is stronger than the standard** — phantom reads are prevented, thanks to MVCC snapshots.
- **Serializable** uses SSI (Serializable Snapshot Isolation) — it does not lock; it detects conflicts and aborts one transaction at commit if they violate serializability. Retry logic is required.

**Code — setting per transaction:**
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- All SELECTs in this transaction see the same snapshot
SELECT balance FROM accounts WHERE id = 1;
-- ... other work
SELECT balance FROM accounts WHERE id = 1;     -- same value, even if someone else committed
COMMIT;
```

**Follow-up 1: When would you actually need Serializable instead of Read Committed?**
When correctness depends on a multi-row invariant that can be violated by interleaving with concurrent transactions. Classic example: "transfer $100 from A to B, but only if A's total balance across all accounts stays > $0." Two concurrent transfers could each see balance > $100 at snapshot time, but together push it negative. Serializable detects this and aborts one. At Read Committed you'd need explicit `SELECT FOR UPDATE`.

**Follow-up 2: What's a "serialization failure" and how should the app handle it?**
A Postgres-specific error `SQLSTATE 40001 serialization_failure` raised when SSI detects a conflict at commit. Standard pattern: catch it and retry the entire transaction with exponential backoff (limit retries to avoid livelock). All ORMs have helpers for this; Prisma's `interactiveTransactions` accepts a `maxWait` and `timeout`.

---

### Q11. What is MVCC and how does Postgres implement it?

**Answer:**
**MVCC (Multi-Version Concurrency Control)** is the technique of keeping multiple versions of each row so that readers and writers don't block each other. When you update a row, you don't overwrite it — you write a new version, and old versions remain visible to transactions that started before the update.

**Postgres implementation:**
- Every row has hidden columns `xmin` (the txid that inserted it) and `xmax` (the txid that deleted/updated it, or 0 if still live).
- Every transaction starts with a snapshot: a set of txids visible to it (committed before it started).
- A row is visible to a transaction if its `xmin` is in the snapshot and its `xmax` is not (or is 0).
- `UPDATE` is effectively `INSERT new row + mark old row dead` — old row keeps its xmin and gets xmax set to the updating txid.
- Old row versions are reclaimed by `VACUUM` once no transaction can possibly need them.

**Diagram:**
```
            txid=100 inserts row v1   →    [xmin=100, xmax=0]   "Alice", $50
            txid=200 updates to v2    →    v1: [xmin=100, xmax=200]   (now dead from 200 onward)
                                            v2: [xmin=200, xmax=0]   "Alice", $60

txid=150 (started between 100 and 200) sees v1
txid=250 sees v2
```

**Implications:**
- **Readers don't block writers** and vice versa. Huge for concurrency.
- **Updates are expensive** — each modification writes a new row version somewhere.
- **Disk bloat without VACUUM** — dead row versions accumulate until vacuum reclaims their space.

**Follow-up 1: How does this compare to MySQL's MVCC?**
MySQL InnoDB also uses MVCC but stores old versions in a separate "rollback segment" (undo log) instead of inline. Updates overwrite the row, with the previous version logged for older snapshots. Pros: no table bloat from updates, cheaper VACUUM-equivalent. Cons: long-running transactions can blow up the undo log size, and rollbacks of large transactions are slower.

**Follow-up 2: What's "transaction wraparound" and why should I care?**
Postgres txids are 32-bit. After ~2 billion transactions, they wrap. PG uses a "frozen txid" mechanism — rows older than a certain age get marked as visible to everyone, sidestepping the wraparound. If autovacuum falls behind, you risk the database refusing writes to protect data integrity (`MultiXact members threshold exceeded`). For high-throughput systems, monitor `pg_stat_user_tables.n_tup_upd` and tune `autovacuum_freeze_max_age`.

---

### Q12. Explain dirty reads, non-repeatable reads, and phantom reads.

**Answer:**

**Dirty read** — a transaction reads a row that another transaction has modified but not yet committed. If the other transaction rolls back, you've read data that "never existed."
```
T1: UPDATE accounts SET balance = 0 WHERE id = 1;     -- not committed
T2: SELECT balance FROM accounts WHERE id = 1;        -- reads 0 (dirty!)
T1: ROLLBACK;                                         -- the 0 was a lie
```
Postgres **never allows dirty reads**, regardless of isolation level.

**Non-repeatable read** — within one transaction, you read the same row twice and get different values because another transaction committed between your reads.
```
T1: SELECT balance FROM accounts WHERE id = 1;        -- reads 50
T2: UPDATE accounts SET balance = 100 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;        -- reads 100 — not repeatable!
```
Prevented by **Repeatable Read** and stricter.

**Phantom read** — within one transaction, you run the same range query twice and see different *rows* (not just different values) because another transaction inserted/deleted rows in your range.
```
T1: SELECT COUNT(*) FROM users WHERE age >= 18;        -- 100
T2: INSERT INTO users (age) VALUES (25); COMMIT;
T1: SELECT COUNT(*) FROM users WHERE age >= 18;        -- 101 — phantom!
```
SQL standard requires **Serializable** to prevent phantoms. Postgres prevents them at **Repeatable Read** thanks to its snapshot model.

**Follow-up 1: Why does PG's Repeatable Read prevent phantoms but the SQL standard doesn't require it?**
Because Postgres uses snapshot isolation. The snapshot is taken at the start of the transaction and includes all rows visible at that moment. New rows inserted by other transactions after the snapshot simply aren't visible — there's no "range lock" needed because the snapshot conceptually freezes the world. The SQL standard was written when range locks were the implementation strategy, and didn't anticipate snapshot-based isolation.

**Follow-up 2: Are there anomalies snapshot isolation *doesn't* prevent?**
Yes — write skew. Two transactions read overlapping data, each makes a decision based on the snapshot, and each writes a different row. Each looks consistent in isolation but together they violate an invariant. Example: on-call schedule with "at least one person must be on duty" rule — two on-callers both check "someone else is on," each removes themselves. Both see "someone else on" in their snapshot, both write. Need Serializable (SSI) to detect this.

---

### Q13. What's `SELECT FOR UPDATE`? When do you use it?

**Answer:**
`SELECT FOR UPDATE` is row-level pessimistic locking. It acquires an exclusive lock on the selected rows that's held until the transaction commits or rolls back. Other transactions that try to lock or update the same rows wait.

Common pattern: **"read-modify-write"** scenarios where you need to read state, decide based on it, and update — and you can't tolerate someone else changing the state between your read and write.

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;     -- lock the row
-- application checks balance >= 100
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;                                                     -- lock released
```

Variants:
- **`FOR UPDATE`** — exclusive, blocks all other reads-for-update and writes.
- **`FOR NO KEY UPDATE`** — weaker; doesn't block foreign-key-reference reads. Use for updates that don't change key columns.
- **`FOR SHARE`** — shared lock; multiple transactions can hold it simultaneously, but blocks `FOR UPDATE`. Use when you want to ensure a row doesn't change but multiple readers are fine.
- **`FOR KEY SHARE`** — weakest; only blocks key changes.
- **`SKIP LOCKED`** — return only rows that aren't already locked. Foundation of efficient SQL-based job queues.

**Code — SQL job queue using `SKIP LOCKED`:**
```sql
BEGIN;
-- Worker grabs the next available job without blocking on other workers
WITH next AS (
  SELECT id FROM jobs
  WHERE status = 'pending' AND scheduled_for <= now()
  ORDER BY scheduled_for
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE jobs SET status = 'running', started_at = now()
FROM next
WHERE jobs.id = next.id
RETURNING jobs.*;
COMMIT;
```

`SKIP LOCKED` means N workers can each grab a different job concurrently, no queuing.

**Follow-up 1: How does this interact with `SERIALIZABLE`?**
At Serializable, you usually don't need `FOR UPDATE` — SSI detects conflicts automatically and aborts at commit. But explicit locking can still be useful to force serialization in a known way (rather than relying on retry loops). Mixing them works but you need to handle 40001 retries.

**Follow-up 2: What's the difference between `FOR UPDATE` and `SELECT … FOR UPDATE OF table_name`?**
With multi-table joins, `FOR UPDATE` locks rows in *every* table mentioned. `FOR UPDATE OF specific_table` only locks rows in that one. Useful in joins where you want to read other tables but only write to one, avoiding unnecessary contention.

---

### Q14. What is a deadlock and how does Postgres handle it?

**Answer:**
A **deadlock** is when two or more transactions are each waiting on a lock held by another, forming a cycle. None can proceed. Classic example:

```
T1: BEGIN; UPDATE a SET x = 1 WHERE id = 1;             -- locks row a:1
T2: BEGIN; UPDATE a SET x = 2 WHERE id = 2;             -- locks row a:2
T1: UPDATE a SET x = 1 WHERE id = 2;                    -- waits for T2's lock on a:2
T2: UPDATE a SET x = 2 WHERE id = 1;                    -- waits for T1's lock on a:1 → deadlock
```

Postgres has a background deadlock detector that runs periodically (every `deadlock_timeout`, default 1s). It checks the lock wait graph for cycles; when it finds one, it picks a victim (the youngest transaction in the cycle), aborts it with error `40P01 deadlock_detected`, and the others proceed.

**Prevention strategies:**
1. **Always acquire locks in the same order.** If every transaction updates rows in ascending `id` order, no cycle is possible.
2. **Keep transactions short.** Less time holding locks → less chance of collision.
3. **Use `SELECT FOR UPDATE` early** to acquire all locks upfront rather than incrementally.
4. **Retry on deadlock.** Like serialization failures, deadlocks are transient; the standard pattern is catch-and-retry.

**Code — defensive locking pattern:**
```sql
BEGIN;
-- Lock both rows in deterministic order
SELECT id FROM accounts WHERE id IN ($1, $2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = $1;
UPDATE accounts SET balance = balance + 100 WHERE id = $2;
COMMIT;
```

The `ORDER BY id FOR UPDATE` ensures concurrent transfers between the same two accounts can't deadlock — both lock the lower id first.

**Follow-up 1: How do I diagnose a recurring deadlock?**
Enable `log_lock_waits` and `deadlock_timeout = 1s`. Postgres logs full lock waits (after `deadlock_timeout`) and deadlock detection events with the queries and PIDs involved. Look at `pg_stat_activity` and `pg_locks` for the live picture. Once you have the two queries, you can usually spot the conflicting lock-order pattern.

**Follow-up 2: Why is the victim "the youngest transaction"?**
Because aborting it usually wastes the least work. Younger transactions have done less; older transactions may have done substantial work that would have to be redone. Postgres's actual policy is "the transaction that detected the deadlock," which in practice is approximately the youngest in active waiting. You can't tune this directly.

---

## Part C — Schema Design & Advanced

### Q15. When should you denormalize? What are the trade-offs?

**Answer:**
Normalization eliminates redundancy — every fact lives in exactly one place. Denormalization deliberately duplicates data to make reads faster, at the cost of write complexity (keeping copies in sync).

**Reasons to denormalize:**
- Read paths join too many tables, plan time dominates.
- A frequently-read aggregate (count, sum) is expensive to compute every time.
- Cross-table queries need a single index that spans multiple tables.

**Common forms:**
1. **Materialized aggregates** — store `users.post_count` instead of `SELECT count(*) FROM posts WHERE user_id`. Update on each post insert/delete (trigger or app logic).
2. **Copied columns** — store `order_items.product_name` snapshot at order time. Survives the product being renamed/deleted.
3. **Materialized views** — pre-computed query results, refreshed periodically.

**Code — counter denormalization with a trigger:**
```sql
ALTER TABLE users ADD COLUMN post_count int NOT NULL DEFAULT 0;

CREATE FUNCTION update_post_count() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE users SET post_count = post_count + 1 WHERE id = NEW.user_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE users SET post_count = post_count - 1 WHERE id = OLD.user_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_count_trigger
  AFTER INSERT OR DELETE ON posts
  FOR EACH ROW EXECUTE FUNCTION update_post_count();
```

**Trade-offs:**
- Faster reads, more complex writes.
- Drift risk — what if the trigger fails or a backfill is wrong?
- Adds contention on the `users` row from every post insert.

**Default to normalized.** Denormalize when measurement shows reads dominate and the cost is real. Don't denormalize for hypothetical future scale.

**Follow-up 1: What's a materialized view and when is it better than a denormalized column?**
A materialized view is a saved query result, refreshable on demand. `CREATE MATERIALIZED VIEW ... AS SELECT ...; REFRESH MATERIALIZED VIEW CONCURRENTLY ...`. Better than columns when (a) the aggregate is complex/multi-table, (b) you can tolerate seconds-to-minutes lag (refresh interval). Worse when (a) you need real-time accuracy, (b) writes need to immediately reflect.

**Follow-up 2: How do you backfill a denormalized counter safely?**
For small tables, single update: `UPDATE users SET post_count = (SELECT count(*) FROM posts WHERE user_id = users.id);`. For huge tables, chunk by `id` ranges or by tenant; commit in batches; reconcile with the trigger by using a "running diff" approach (compute the counter at time T, apply trigger-tracked delta from T to now). Always have a "fix from source of truth" script handy in case drift is detected.

---

### Q16. JSONB columns — when to use them vs separate tables?

**Answer:**
JSONB is Postgres's binary JSON type. It stores structured data inside one column, supports rich querying (`@>`, `->`, `->>`, `jsonb_path_query`), and can be GIN-indexed. The convenience is real, but so are the costs.

**Use JSONB when:**
- The schema is genuinely dynamic — different rows have different fields (e.g., webhook payloads, per-customer custom attributes).
- The data is read as a blob most of the time, with occasional structured queries.
- You need to evolve fast without migrations for every new field.

**Don't use JSONB when:**
- The fields are well-known and queried regularly — you're sacrificing planner clarity, statistics, and proper indexing for nothing.
- You need foreign keys or referential integrity on the inner fields.
- You'll join against the inner values.

**Code:**
```sql
-- Event log table — payload shape varies by event_type
CREATE TABLE events (
  id          uuid PRIMARY KEY,
  event_type  text NOT NULL,
  user_id     uuid NOT NULL,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  payload     jsonb NOT NULL
);

CREATE INDEX events_user_idx ON events (user_id, occurred_at DESC);
CREATE INDEX events_type_idx ON events (event_type);
CREATE INDEX events_payload_gin ON events USING gin (payload jsonb_path_ops);

-- Common query: "show me all login events for this user with mobile platform"
SELECT * FROM events
WHERE user_id = $1
  AND event_type = 'login'
  AND payload @> '{"platform": "mobile"}'
ORDER BY occurred_at DESC LIMIT 50;
```

**Follow-up 1: What's the difference between `jsonb_ops` and `jsonb_path_ops` GIN operator classes?**
`jsonb_ops` (default) supports all JSONB query operators but builds a larger index — every key and every value gets an entry. `jsonb_path_ops` only supports `@>` (containment) but indexes paths, making the index smaller and queries faster — at the cost of supporting fewer operators. For containment-only workloads, use `jsonb_path_ops`.

**Follow-up 2: What's the cost of updating a single field in a JSONB column?**
The whole column rewrites — MVCC means a new row version, the entire JSONB blob is re-stored. For a 50KB JSONB column updated frequently, that's terrible. Fix: pull the hot fields out into typed columns; keep only the truly dynamic stuff in JSONB. TOAST (Postgres's mechanism for large values) helps but doesn't eliminate the cost.

---

### Q17. How do foreign keys work? What's the performance cost of enforcement?

**Answer:**
A foreign key is a constraint that says "this column's value must exist as a primary/unique key in another table." Postgres enforces it on insert/update of the child row (must reference an existing parent) and on delete/update of the parent (depending on `ON DELETE` / `ON UPDATE` action).

Enforcement requires a lookup in the parent table on every child write. For high-throughput writes, this adds latency. The cost is usually small (parent index lookup) but compounds.

**`ON DELETE` actions:**
- **`NO ACTION`** (default) — error if a referencing row exists.
- **`RESTRICT`** — like NO ACTION but checked immediately, not deferred.
- **`CASCADE`** — delete referencing rows automatically.
- **`SET NULL`** — set referencing column to NULL.
- **`SET DEFAULT`** — set to column default.

**Code:**
```sql
CREATE TABLE orders (
  id      uuid PRIMARY KEY,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE
);

-- The FK creates an implicit lookup on every INSERT INTO orders
-- but does NOT create an index on orders.user_id.
-- ALWAYS index the child side too:
CREATE INDEX orders_user_idx ON orders (user_id);
```

**Gotcha:** when you delete a user, Postgres needs to find all referencing orders. Without an index on `orders.user_id`, this is a seq scan. Always index FK columns on the child side.

**Follow-up 1: When would you deliberately skip foreign keys?**
At very high write throughput where every microsecond matters (e.g., metrics ingest), and where you can guarantee referential integrity at the app layer or accept eventual cleanup of orphans. Also in event-sourced systems where the "true" state lives elsewhere. Most apps should keep FKs — they're cheap insurance.

**Follow-up 2: What's a deferrable foreign key?**
`REFERENCES ... DEFERRABLE INITIALLY DEFERRED` — the constraint is checked at commit time, not on each statement. Useful when you need to insert a circular reference (A references B, B references A) within a single transaction. Without deferral, the first insert would fail because the target doesn't yet exist. Rarely needed in practice.

---

### Q18. UUID vs serial vs ULID for primary keys — what are the trade-offs?

**Answer:**
Three common choices, each with distinct trade-offs:

**`serial` / `bigserial` (auto-incrementing integer):**
- Pros: tiny (4 or 8 bytes), great locality (B-tree leaf inserts at the tail), monotonically increasing — friendly to caching and indexing.
- Cons: predictable (security/enumeration risk in URLs), coordinated allocation prevents some distributed scenarios, leaks ordering info (you can tell how many users exist).

**`uuid` (random — `gen_random_uuid()`):**
- Pros: globally unique without coordination — can be generated client-side. Unpredictable. Multi-master safe.
- Cons: 16 bytes (4x serial). Random insertion order causes B-tree fragmentation and bad cache locality — every insert touches a different leaf page.

**`ulid` / `uuidv7` (time-sortable identifiers):**
- Pros: same uniqueness as UUID, but sortable by creation time. Better B-tree insert locality than v4. Often base32-encoded for URL friendliness.
- Cons: minor — leaks creation order (similar to serial). Needs library support outside Postgres (or `uuidv7` from PG17+).

**For your stack:**
- Internal joins, high write rate → `bigserial`.
- Public IDs, multi-region writes → `uuidv7` / `ulid`.
- Compromise: bigserial PK + a separate uuid column for public exposure.

**Code:**
```sql
-- Modern Postgres has uuid_generate_v4 via pgcrypto (or gen_random_uuid()).
CREATE TABLE users (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email      text NOT NULL UNIQUE,
  created_at timestamptz NOT NULL DEFAULT now()
);

-- ULID via a helper extension, or generate from the app
INSERT INTO users (id, email) VALUES ('01HMQ8...', 'a@b.com');
```

**Follow-up 1: Why is UUIDv4's random insert order a problem?**
B-tree indexes are sorted by key. With sequential keys, inserts cluster at the rightmost leaf — a hot page that stays in cache. With random keys, every insert hits a different leaf, evicting other pages. The B-tree gets bigger, autovacuum works harder, write performance suffers. On a write-heavy table this can be 2-3x slower than sequential.

**Follow-up 2: When does the 16-byte UUID size actually matter?**
At high row counts (billions). 4x the key size means 4x the index storage, 4x the bytes through the buffer cache. For a 1B-row table with 5 indexes, that's tens of GB of extra cache pressure. For a 1M-row table, no one cares.

---

### Q19. How do you do safe schema migrations on a hot table?

**Answer:**
The naive `ALTER TABLE` can lock a hot table for minutes — every reader and writer blocks while the migration runs. In production, that's downtime. Safe migration patterns trade speed for non-blocking behavior.

**Cheap, non-blocking operations:**
- `ADD COLUMN` of a nullable column with no default (PG11+).
- `ADD COLUMN ... DEFAULT constant` (PG11+ optimizes this).
- `CREATE INDEX CONCURRENTLY` — builds in the background, no exclusive lock.
- `DROP INDEX CONCURRENTLY`.
- `ALTER COLUMN ... DROP NOT NULL`.

**Expensive operations and their safe alternatives:**

| Naive | Problem | Safe alternative |
|-------|---------|------------------|
| `ADD COLUMN x int NOT NULL DEFAULT 0` (pre-PG11) | Rewrites the whole table | Add nullable, backfill in chunks, then add NOT NULL |
| `ALTER COLUMN ... SET NOT NULL` on existing column | Full table scan, exclusive lock | `ADD CONSTRAINT ... CHECK (col IS NOT NULL) NOT VALID` → `VALIDATE CONSTRAINT` (lower lock) → swap |
| `ALTER COLUMN ... TYPE bigint` | Rewrites | Add new column, dual-write, backfill, swap, drop old |
| `DROP COLUMN` | Quick, but space not reclaimed | Acceptable for most; vacuum later |
| `ADD FOREIGN KEY` | Validates all rows under lock | `ADD CONSTRAINT ... NOT VALID` → `VALIDATE CONSTRAINT` |

**Code — adding a NOT NULL column on a 100M-row table:**
```sql
-- Step 1: add nullable column
ALTER TABLE users ADD COLUMN tier text;

-- Step 2: backfill in chunks (run in a script)
UPDATE users SET tier = 'free' WHERE id BETWEEN 1 AND 100000 AND tier IS NULL;
-- ... loop with VACUUM ANALYZE periodically

-- Step 3: add CHECK constraint as NOT VALID (doesn't scan)
ALTER TABLE users ADD CONSTRAINT users_tier_not_null
  CHECK (tier IS NOT NULL) NOT VALID;

-- Step 4: validate (acquires SHARE UPDATE EXCLUSIVE — readers/writers fine)
ALTER TABLE users VALIDATE CONSTRAINT users_tier_not_null;

-- Step 5: set NOT NULL (fast — Postgres uses the validated CHECK)
ALTER TABLE users ALTER COLUMN tier SET NOT NULL;
ALTER TABLE users DROP CONSTRAINT users_tier_not_null;   -- redundant now
```

**Follow-up 1: What's the lock taken by `CREATE INDEX CONCURRENTLY` and what does it block?**
A `SHARE UPDATE EXCLUSIVE` lock — blocks DDL on the table (other concurrent index builds, ALTERs, VACUUM FULL) but not normal reads or writes. Two phases: build the index using a snapshot, then a second pass to catch concurrent changes. Slower than blocking `CREATE INDEX` but production-safe.

**Follow-up 2: How do you handle a migration that needs to run before the new code is deployed?**
Use expand-contract (also called expand-migrate-contract):
1. **Expand** — add the new column/table; new code can write to both old and new.
2. **Migrate** — backfill data; new code reads from new.
3. **Contract** — remove old column/code.
Each step is independently deployable and reversible. Critical for rolling deploys where old and new code run simultaneously.

---

### Q20. What's the N+1 query problem and how do you detect/fix it?

**Answer:**
N+1 means: one query to fetch a list of N items, then N additional queries to fetch related data for each item. Total: N+1 queries when 1 or 2 would suffice.

Classic example in an ORM:
```js
const users = await db.user.findMany({ take: 20 });             // 1 query
for (const u of users) {
  u.posts = await db.post.findMany({ where: { userId: u.id }}); // 20 queries
}
```
21 queries for 20 users. At scale (the user's dashboard showing 100 entities), this is a thousand queries per page load.

**Detection:**
- `pg_stat_statements` — count how many calls of `SELECT * FROM posts WHERE user_id = $1` per page load. Should be 0 or 1, not 100.
- ORM debug logging — log every query with a tag for the request ID; count per request.
- APM tools (Datadog, New Relic) — show DB query count per endpoint.

**Fixes:**

1. **Bulk fetch with `IN` / `ANY`:**
```js
const users = await db.user.findMany({ take: 20 });
const ids = users.map(u => u.id);
const posts = await db.post.findMany({ where: { userId: { in: ids } } });
const byUser = groupBy(posts, p => p.userId);
users.forEach(u => u.posts = byUser[u.id] ?? []);
```
2 queries total.

2. **JOIN in one query** when the relation is 1:1 or you really need a single round trip:
```sql
SELECT u.*, p.id AS post_id, p.title
FROM users u LEFT JOIN posts p ON p.user_id = u.id
WHERE u.tenant_id = $1
LIMIT 20;
```
Application code reassembles. Be careful of cartesian explosion if multiple to-many relations.

3. **ORM `include` / `eager` loading** — Prisma's `include`, TypeORM's `relations`, Mongoose's `populate`. They issue 2 queries under the hood: one for the parent, one bulk for children. Use this by default.

**Follow-up 1: When is N+1 actually acceptable?**
When N is bounded and small (≤ 5), and the inner query is well-indexed. The constant overhead of N queries vs 1 query is real but minor at that scale. If you're choosing between a complex bulk fetch and the simple loop with N=3, the loop is sometimes more maintainable.

**Follow-up 2: What's the DataLoader pattern?**
Originally from GraphQL/Facebook — a per-request loader that batches and dedupes ID lookups. You call `loader.load(id)` for each item; it queues the IDs, executes one query at the end of the tick, returns a Promise that resolves for each caller. Solves N+1 at the framework layer without code changes per-resolver. The `dataloader` npm package or equivalent for your ORM.

---

## Part D — Operations

### Q21. Streaming replication vs logical replication — what's the difference?

**Answer:**

**Streaming (physical) replication:**
- Replicas apply WAL records byte-for-byte from the primary.
- Replica is a binary copy — same Postgres version, same OS, same architecture.
- Replicates everything: all databases, all tables.
- Read-only on the replica (hot standby).
- Failover replaces the primary; promoted replica becomes new primary.
- Use cases: HA (high availability), read scaling, disaster recovery.

**Logical replication:**
- Replicates *changes* (rows inserted/updated/deleted) as logical events, not WAL bytes.
- Replicas can have different schemas, indexes, even different Postgres versions.
- Per-table granularity — replicate only what you publish.
- Writable on the replica (subject to your application's discipline).
- Use cases: zero-downtime upgrades, sharding, data integration (CDC to Kafka via Debezium).

**Code — basic streaming replica setup:**
```bash
# On primary postgresql.conf:
# wal_level = replica
# max_wal_senders = 10
# Create replication user and pg_hba entry, then on the standby:
pg_basebackup -h primary -U repl -D /var/lib/postgres -P -R
# -R writes a recovery.signal and connection string; just start postgres
```

```sql
-- Logical replication
-- On publisher
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- On subscriber
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=primary user=repl dbname=app'
  PUBLICATION my_pub;
```

**Follow-up 1: How do you do a zero-downtime major-version upgrade?**
With logical replication: set up the new version as a logical subscriber to the old, let it catch up, switch traffic. Without it (pre-PG10 or for unsupported features), use `pg_upgrade` with downtime, or dump-and-restore.

**Follow-up 2: What's replication lag and how do you measure it?**
The delay between a commit on the primary and the replica applying it. Measure via `pg_stat_replication.replay_lag` (PG10+) or by comparing `pg_current_wal_lsn()` on primary to `pg_last_wal_replay_lsn()` on replica. Lag causes stale reads when you read from a replica — your service might write and immediately read back, getting old data. Solutions: read-your-writes routing, `synchronous_commit = remote_apply`, or wait for lag with `pg_wait_for_replay`.

---

### Q22. Why do you need a connection pooler like PgBouncer?

**Answer:**
Each Postgres backend process is heavy — it forks a process, allocates several MB of memory, has its own caches. Opening a connection costs ~10ms; holding one open with `max_connections=200` and 5 services each having 50 pools quickly exhausts the server.

A connection pooler sits between application and database, maintaining a small pool of real backend connections and multiplexing many client connections onto them.

**PgBouncer modes:**
- **Session pooling** — client gets a backend for the duration of its session (until disconnect). Mild benefit over no pooling.
- **Transaction pooling** — client gets a backend for the duration of one transaction. The default for high-throughput apps. Big win: 10,000 clients can share 100 backends.
- **Statement pooling** — backend released after each statement. Most aggressive; breaks prepared statements and many features.

**Trade-offs of transaction pooling:**
- Can't use session-level features: `SET` (per-session config), `LISTEN/NOTIFY`, prepared statements (unless protocol-level), temporary tables, advisory locks across transactions.
- Most ORMs work fine — Prisma, TypeORM, Knex support transaction-pool mode.

**Code — typical setup:**
```ini
# pgbouncer.ini
[databases]
mydb = host=postgres port=5432

[pgbouncer]
listen_port = 6432
pool_mode = transaction
max_client_conn = 10000
default_pool_size = 100
```

App connects to PgBouncer on 6432 with the same protocol — it's a transparent proxy.

**Follow-up 1: Why not just raise `max_connections`?**
Each backend uses 5–10 MB of memory minimum, more under load. `max_connections = 1000` on a server with 32 GB RAM and active work_mem usage easily OOMs. Postgres also doesn't scale linearly — high connection counts increase lock contention, planner overhead, and IPC. The rule of thumb: keep `max_connections` ≤ a few hundred, use a pooler for everything else.

**Follow-up 2: When does PgBouncer become a bottleneck?**
PgBouncer is single-threaded. At ~30k+ queries/sec on one PgBouncer instance, you saturate one CPU core. Solutions: run multiple PgBouncers (one per app instance, or sharded by database), use the multi-process fork (`pgbouncer-rr`), or switch to a multi-threaded alternative like Odyssey.

---

### Q23. What does VACUUM do and why is autovacuum important?

**Answer:**
**VACUUM** reclaims space taken by dead row versions (deleted rows and old versions from UPDATE). Recall MVCC: every UPDATE leaves the old version in place. Without VACUUM, the table grows forever even if the row count is stable.

VACUUM's jobs:
1. Mark dead tuple space as reusable inside existing pages.
2. Update the visibility map (which pages have only "all-visible" tuples).
3. Update statistics (when `VACUUM ANALYZE`).
4. Advance the relfrozenxid to prevent transaction-ID wraparound.

VACUUM doesn't return disk to the OS — for that you need `VACUUM FULL` (rewrites the table, exclusive lock — disruptive) or `pg_repack` (concurrent rewrite).

**Autovacuum** is the background process that runs VACUUM automatically based on thresholds:
- Trigger when dead tuples > `autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * table_size`.
- Default scale factor: 0.2 (20% of table dead).
- Hot tables: tune lower (`autovacuum_vacuum_scale_factor = 0.05`) so vacuum runs more often.

**Code — diagnosing vacuum problems:**
```sql
-- Tables with most dead tuples
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;

-- Long-running queries blocking vacuum (they hold an old snapshot)
SELECT pid, age(backend_xid), state, query
FROM pg_stat_activity
WHERE state != 'idle' AND backend_xid IS NOT NULL
ORDER BY age(backend_xid) DESC;
```

**Follow-up 1: What's "VACUUM FULL" and when do I run it?**
`VACUUM FULL` rewrites the table into a new file, free of dead tuples, and replaces the original. Returns space to the OS. Costs: takes an exclusive lock (no reads/writes), requires double the disk space during rewrite, can take hours. Use only when (a) you can take downtime, (b) the table is severely bloated and ordinary VACUUM can't help. Otherwise `pg_repack` is the production choice.

**Follow-up 2: My VACUUM is making things slower — what's wrong?**
A few possibilities: (1) it's competing for I/O — set `autovacuum_vacuum_cost_delay` to throttle; (2) a long-running transaction holds an old snapshot, preventing vacuum from reclaiming tuples newer than that snapshot — find and kill the long transaction; (3) the table is hot and bloat is happening faster than vacuum can keep up — tune autovacuum to run more frequently and with more workers (`autovacuum_max_workers`).

---

### Q24. How does table partitioning work in Postgres?

**Answer:**
Partitioning splits a logical table into multiple physical "partition" tables, transparent to most queries. Postgres supports:
- **Range partitioning** — by a range of values (typical for time-series: one partition per month).
- **List partitioning** — by an explicit set of values (one partition per region, or per tenant).
- **Hash partitioning** — by a hash of the partitioning key (uniform distribution; useful for shard-like balancing).

Benefits:
- **Partition pruning** — queries with predicates on the partition key skip irrelevant partitions entirely. A query for "last week's data" only scans last week's partition.
- **Easy data lifecycle** — drop a whole partition to delete old data instantly (no big DELETE, no bloat).
- **Smaller indexes per partition** — better cache locality, faster individual queries.

**Code — your IoT telemetry table partitioned by month:**
```sql
CREATE TABLE telemetry (
  device_id     uuid NOT NULL,
  recorded_at   timestamptz NOT NULL,
  metric        text NOT NULL,
  value         double precision NOT NULL
) PARTITION BY RANGE (recorded_at);

-- One partition per month
CREATE TABLE telemetry_2026_05 PARTITION OF telemetry
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE telemetry_2026_06 PARTITION OF telemetry
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Indexes are created per partition (or on the parent, which propagates)
CREATE INDEX ON telemetry (device_id, recorded_at DESC);

-- Drop old data — instant, no rebuild
DROP TABLE telemetry_2025_05;
```

**Follow-up 1: What's the difference between partitioning and sharding?**
Partitioning splits a table across multiple physical files on the *same* server. Sharding splits across *multiple* servers. Both reduce per-shard/partition size, but only sharding scales beyond one machine's capacity. Postgres partitioning + Citus extension = effectively sharding within Postgres.

**Follow-up 2: What's `pg_partman` and why use it?**
`pg_partman` is an extension that automates partition lifecycle: it creates future partitions ahead of time, retains/drops old partitions based on retention policy, and handles edge cases (default partitions, partition migrations). Without it, you write cron jobs to create next month's partition. With it, configure once and forget.

---

### Q25. How does Postgres backup and Point-in-Time Recovery (PITR) work?

**Answer:**
Two backup strategies:

**Logical backup — `pg_dump`:**
- Dumps SQL (or custom binary format) that can be restored anywhere.
- Slow for huge databases.
- Provides a consistent snapshot but doesn't continue beyond that point.
- Good for migrations, dev environments, small DBs.

**Physical backup — `pg_basebackup` + WAL archive:**
- Binary copy of the data directory.
- Combined with continuous WAL archiving, supports **Point-in-Time Recovery** — restore the base backup, then replay WAL up to any moment in time (e.g., "10:30 AM, one minute before the bad migration").
- The basis of high-availability and DR setups.

**PITR setup:**
1. Enable WAL archiving: `archive_mode = on; archive_command = 'aws s3 cp %p s3://bucket/wal/%f'`.
2. Take a base backup: `pg_basebackup -h primary -D /backup -X stream`.
3. On disaster: restore the base backup, configure `restore_command`, set `recovery_target_time = '2026-05-27 10:29:00'`, start postgres. It replays WAL from the archive up to that time.

**Code:**
```bash
# Base backup
pg_basebackup -h primary -D /backups/base -X fetch -P

# In postgresql.conf
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-wal/%f'

# Restore for PITR
# 1. Copy base backup to data dir
# 2. Create recovery.signal
# 3. In postgresql.auto.conf:
restore_command = 'aws s3 cp s3://my-wal/%f %p'
recovery_target_time = '2026-05-27 10:29:00'
recovery_target_action = 'promote'
# 4. Start postgres
```

**Follow-up 1: Why not just `pg_dump` nightly and be done?**
`pg_dump` gives you point-in-time recovery only to the dump moment. If the dump is at 2am and corruption happens at 10am, you lose 8 hours. WAL archiving + base backup gives recovery to any second since the last base backup — RPO measured in seconds, not hours.

**Follow-up 2: What's the difference between `pg_basebackup` and `WAL-G` / `pgbackrest`?**
`pg_basebackup` is Postgres's built-in tool — simple, no parallelism, no incremental backups. WAL-G and pgBackRest are production-grade external tools: parallel compression, incremental and differential backups, cloud storage integration, deduplication, encryption, and parallel restore. For anything beyond toy size, use one of these.

---

*End of section 03. Next: MongoDB (15 questions).*
