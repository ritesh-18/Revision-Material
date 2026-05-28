# 16 — DSA, Backend Flavor (5 Questions)

You have 500+ LeetCode solves and GATE-level CS fundamentals. Rather than re-listing every algorithm, these 5 questions connect core data structures to the backend problems you actually solve — the framing senior interviewers use to separate "memorized LeetCode" from "understands when to apply it."

---

### Q1. How does Big-O complexity actually show up in backend systems?

**Answer:**
Big-O describes how runtime/memory grows with input size. In backend work it's not abstract — it's the difference between a service that scales and one that falls over at 10x traffic.

**Where it bites in real systems:**

- **O(N) per request that should be O(1)** — fetching all rows then filtering in app code instead of a `WHERE` clause with an index. Fine at 100 rows, fatal at 10M.
- **O(N²) hidden in loops** — the N+1 query problem (section 03 Q20) is O(N) network round-trips when it should be O(1). Nested loops over collections that grow.
- **O(N) memory** — loading a whole table/file into memory instead of streaming. Works in dev, OOMs in prod (section 01 Q11–Q12).
- **O(log N) is your friend** — B-tree index lookups, binary search. This is why indexed queries stay fast as tables grow.
- **O(1) amortized** — hash map lookups, why Redis `GET` and dedup-by-key are cheap.

**The key backend insight:** the constant factors and the *I/O* often matter more than the asymptotic class at typical scale. An O(N) in-memory scan of 1000 items beats an O(log N) query that makes a network round-trip. But as N grows, asymptotic complexity wins — and you must know where your N comes from (rows? users? requests? items per request?).

**Worked example — spot the complexity bug:**
```ts
// O(N * M) — for each order, scan all products. 1000 orders × 5000 products = 5M ops
function enrichOrders(orders: Order[], products: Product[]) {
  return orders.map(o => ({
    ...o,
    product: products.find(p => p.id === o.productId),   // O(M) scan per order
  }));
}

// O(N + M) — build a lookup map once, then O(1) per order
function enrichOrders(orders: Order[], products: Product[]) {
  const byId = new Map(products.map(p => [p.id, p]));     // O(M)
  return orders.map(o => ({ ...o, product: byId.get(o.productId) }));  // O(N)
}
```

**Follow-up 1: Why is "O(N) but in-memory" often better than "O(log N) but a DB query" at small N?**
Because the constant factor for a network round-trip (~1ms) dwarfs the asymptotic difference at small N. Looking up 50 items: an in-memory scan is microseconds; 50 indexed DB queries are 50ms of round-trips (the N+1 problem). The fix isn't "use a better algorithm" — it's "do one query, build a map, look up in memory." Know where the cost actually is: usually I/O, not CPU.

**Follow-up 2: How do you reason about complexity when the bottleneck is the database, not your code?**
Profile what dominates. Most backend latency is I/O (DB, network, disk), not CPU. So the relevant "complexity" is often "how many queries" and "how many rows scanned," not your loop structure. `EXPLAIN ANALYZE` (section 03 Q5) tells you if a query is O(rows) seq scan vs O(log rows) index scan. Optimize the I/O complexity first; micro-optimizing CPU-bound loops rarely matters when you're waiting on Postgres.

---

### Q2. Why is the hash map the backend workhorse? Solve a dedup problem with it.

**Answer:**
The hash map (object/Map in JS, dict in Python, HashMap in Java) gives O(1) average insert/lookup/delete. It's behind an enormous amount of backend infrastructure:

- **Caching** — Redis is essentially a distributed hash map.
- **Deduplication** — "have I seen this ID/key before?"
- **Indexing/grouping** — group items by a key (the `groupBy` pattern).
- **Counting/frequency** — count occurrences (word counts, rate limiting).
- **Memoization** — cache function results by argument.
- **Joins in application code** — build a map from one collection, look up from another (Q1's example).

**Worked problem — deduplicate events in a stream with idempotency:**
You're processing a Kafka/MQTT stream where messages may be redelivered (at-least-once). Dedupe by message ID within a time window.

```ts
class StreamDeduplicator {
  private seen = new Map<string, number>();   // messageId → timestamp
  private windowMs: number;

  constructor(windowMs = 60_000) { this.windowMs = windowMs; }

  // Returns true if this is a NEW message, false if duplicate
  process(messageId: string, now = Date.now()): boolean {
    this.evictOld(now);
    if (this.seen.has(messageId)) return false;   // O(1) duplicate check
    this.seen.set(messageId, now);
    return true;
  }

  private evictOld(now: number) {
    for (const [id, ts] of this.seen) {
      if (now - ts > this.windowMs) this.seen.delete(id);
    }
  }
}
```

**Scaling note:** for a distributed system, this in-memory map only dedupes within one process. For cross-instance dedup, use Redis (`SET messageId 1 NX EX 60` — returns whether it was new) — the distributed hash map. For massive scale where storing every ID is too much memory, a **Bloom filter** trades a small false-positive rate for huge memory savings (Redis supports them).

**Follow-up 1: When does a hash map's O(1) degrade?**
On hash collisions. Worst case is O(N) if many keys hash to the same bucket (rare with good hash functions, but a malicious actor crafting colliding keys can cause "hash flooding" DoS — real attack on web frameworks). Also, resizing (when the map grows past its load factor) is an O(N) operation amortized across inserts. For ordered iteration or range queries, a hash map is the wrong tool — you need a tree (O(log N)).

**Follow-up 2: Hash map vs Bloom filter for "have I seen this?"**
Hash map (or Redis SET): exact, stores every key, O(key size) memory. Bloom filter: probabilistic, fixed small memory regardless of count, but can return false positives ("might have seen it") — never false negatives. Use a Bloom filter when you have billions of keys and can tolerate occasional false "yes, seen it" (e.g., a cache-warming dedup where a rare skip is acceptable). Use exact storage when correctness is required.

---

### Q3. Explain the sliding window technique. Apply it to rate limiting.

**Answer:**
The **sliding window** technique processes a moving range over a sequence — maintaining a window of recent elements and updating incrementally rather than recomputing from scratch. O(N) instead of O(N×W).

**Backend applications:**
- **Rate limiting** — count requests in the last N seconds (section 05 Q11, section 07 Q21).
- **Stream aggregations** — "average over the last 5 minutes," moving averages on telemetry.
- **Anomaly detection** — rolling statistics on sensor data (your IoT project).
- **Log analysis** — "errors in the last minute."

**Worked problem — sliding window rate limiter:**
Allow at most K requests per user in any W-second window.

```ts
class SlidingWindowRateLimiter {
  private requests = new Map<string, number[]>();   // userId → sorted timestamps

  constructor(private maxRequests: number, private windowMs: number) {}

  allow(userId: string, now = Date.now()): boolean {
    let timestamps = this.requests.get(userId) ?? [];
    // Slide the window: drop timestamps older than the window
    const cutoff = now - this.windowMs;
    timestamps = timestamps.filter(t => t > cutoff);   // remove expired

    if (timestamps.length >= this.maxRequests) {
      this.requests.set(userId, timestamps);
      return false;   // limit exceeded
    }
    timestamps.push(now);
    this.requests.set(userId, timestamps);
    return true;
  }
}
```

This is the **sliding window log** algorithm. The window "slides" as time advances — old requests fall off the back, new ones enter the front. Exact, but stores every timestamp (memory grows with the limit).

**Distributed version** (the production answer): same logic in a Redis sorted set with a Lua script for atomicity (section 05 Q11) — `ZREMRANGEBYSCORE` slides the window, `ZCARD` counts, `ZADD` records. Works across all your service instances.

**The optimization mindset:** instead of recomputing "how many requests in the last 60 seconds" from a full log each time (O(N)), you maintain the window incrementally — add new, evict old. That's the sliding window's value: amortized O(1) per element instead of O(W) per query.

**Follow-up 1: Sliding window log vs sliding window counter — what's the trade-off?**
Log stores every request timestamp — exact but memory-heavy (a user with a 10,000/hour limit stores 10,000 timestamps). Counter approximates by tracking counts in fixed sub-windows and weighting — far less memory, slightly inexact at boundaries. For high limits, use the counter; for precise low-limit enforcement, the log. The fixed-window counter is even cheaper but has the boundary burst problem (section 05 Q11).

**Follow-up 2: How does this apply to your IoT anomaly detection?**
Rolling statistics over a sliding window of telemetry: maintain the last N readings (or last T seconds), compute mean/stddev incrementally as new readings arrive and old ones expire. Flag values beyond N standard deviations from the window's mean (z-score). The window slides with each reading — you don't recompute over all history, just update the running statistics. This is the foundation of real-time anomaly detection on streams (section 13 Q9).

---

### Q4. Explain heaps/priority queues. Apply them to job scheduling and top-K.

**Answer:**
A **heap** is a tree-based structure giving O(1) access to the min (min-heap) or max (max-heap) element, with O(log N) insert and extract. A **priority queue** is the abstract data type a heap implements.

**Backend applications:**
- **Job scheduling** — process the highest-priority / earliest-scheduled job next (your BullMQ delayed/prioritized jobs).
- **Top-K queries** — "top 10 trending products," "10 slowest queries" without sorting everything.
- **Rate limiting / expiry** — process items by their expiry time.
- **Dijkstra's / pathfinding** — though less common in CRUD backends.
- **Merge K sorted streams** — combining sorted results from multiple shards/replicas.

**Worked problem — delayed job scheduler:**
Process jobs at their scheduled time; always run the earliest-due job next.

```ts
class DelayedJobScheduler {
  // Min-heap keyed by scheduledFor timestamp (conceptually)
  private heap: { jobId: string; scheduledFor: number }[] = [];

  schedule(jobId: string, scheduledFor: number) {
    this.heap.push({ jobId, scheduledFor });
    this.bubbleUp(this.heap.length - 1);          // O(log N)
  }

  // Get all jobs due by `now`, in time order
  popDue(now: number): string[] {
    const due: string[] = [];
    while (this.heap.length && this.heap[0].scheduledFor <= now) {
      due.push(this.extractMin().jobId);          // O(log N) each
    }
    return due;
  }
  // bubbleUp / extractMin / bubbleDown omitted for brevity
}
```

This is exactly how delayed-job queues work conceptually. **In practice** (your BullMQ), the "heap" is a Redis sorted set (`ZADD jobs scheduledFor jobId`, `ZRANGEBYSCORE ... LIMIT` to pop due jobs) — a distributed, persistent priority queue (section 05 Q1).

**Worked problem — top-K without full sort:**
Find the 10 slowest queries from a stream of millions, without sorting all of them.

```ts
function topK<T>(items: Iterable<T>, k: number, scoreOf: (t: T) => number): T[] {
  const minHeap = new MinHeap<T>((a, b) => scoreOf(a) - scoreOf(b));
  for (const item of items) {
    if (minHeap.size() < k) {
      minHeap.push(item);
    } else if (scoreOf(item) > scoreOf(minHeap.peek())) {
      minHeap.pop();          // drop the smallest of the current top-K
      minHeap.push(item);     // O(log K)
    }
  }
  return minHeap.toSortedArray();
}
// O(N log K) time, O(K) space — vs O(N log N) to sort everything
```

The insight: to find the top-K, you don't sort all N — you keep a min-heap of size K. Each item is O(log K), total O(N log K), using only O(K) memory. Crucial when N is huge (a stream) and K is small.

**Follow-up 1: Why a min-heap for top-K-largest (not a max-heap)?**
Counterintuitive but correct: a min-heap of size K holds the K largest seen so far, with the *smallest of those K* at the root. When a new item arrives, you compare it to the root (the weakest of your current top-K); if it's bigger, evict the root and insert. The min-heap lets you cheaply find and remove the weakest survivor. A max-heap would put the largest at the root — wrong thing to evict.

**Follow-up 2: When would you use the DB/Redis instead of an in-memory heap?**
Almost always in production. An in-memory heap lives in one process and is lost on restart. For job scheduling, use Redis sorted sets (persistent, distributed, what BullMQ does). For top-K analytics, use the DB (`ORDER BY score DESC LIMIT K` with an index) or a streaming system. The in-memory heap matters for (1) understanding the algorithm, (2) processing a single in-memory batch, (3) cases where the data genuinely fits in one process's memory and persistence isn't needed.

---

### Q5. Where do graphs and BFS/DFS show up in backend work?

**Answer:**
Graphs model relationships. BFS (breadth-first, level by level) and DFS (depth-first, follow each path to the end) traverse them. Backend applications are more common than people expect:

- **Dependency resolution** — build order, module loading, task scheduling with prerequisites (topological sort).
- **Permission/access graphs** — ReBAC (section 10 Q10): "is user connected to this resource through some chain of group memberships?"
- **Org hierarchies / threaded comments** — tree traversal (a tree is a graph).
- **Cycle detection** — circular dependencies (section 02 Q2's circular DI), foreign-key cycles, workflow loops.
- **Social/recommendation graphs** — "friends of friends," shortest connection.
- **Saga/workflow orchestration** — steps as a DAG (section 07 Q7).
- **Routing** — service mesh, network topology.

**Worked problem — topological sort for task dependencies:**
You have tasks with dependencies (task B needs A done first). Find a valid execution order, or detect a cycle.

```ts
function topologicalSort(tasks: string[], deps: [string, string][]): string[] {
  // deps[i] = [a, b] means a must run before b
  const graph = new Map<string, string[]>();
  const inDegree = new Map<string, number>();
  for (const t of tasks) { graph.set(t, []); inDegree.set(t, 0); }
  for (const [a, b] of deps) {
    graph.get(a)!.push(b);
    inDegree.set(b, inDegree.get(b)! + 1);
  }

  // Kahn's algorithm (BFS-based): start with zero-dependency tasks
  const queue = tasks.filter(t => inDegree.get(t) === 0);
  const order: string[] = [];
  while (queue.length) {
    const t = queue.shift()!;
    order.push(t);
    for (const next of graph.get(t)!) {
      inDegree.set(next, inDegree.get(next)! - 1);
      if (inDegree.get(next) === 0) queue.push(next);
    }
  }

  if (order.length !== tasks.length) throw new Error('cycle detected');  // not all sortable
  return order;
}
```

This is how build systems order compilation, how a migration runner orders dependent migrations, and how a workflow engine schedules DAG steps.

**Worked problem — permission graph (ReBAC) reachability with BFS:**
"Can user U access resource R?" — U has access if there's a path U → group → ... → R.

```ts
function canAccess(start: string, target: string, edges: Map<string, string[]>): boolean {
  const visited = new Set<string>();
  const queue = [start];
  while (queue.length) {
    const node = queue.shift()!;
    if (node === target) return true;
    if (visited.has(node)) continue;     // avoid cycles / re-visits
    visited.add(node);
    for (const next of edges.get(node) ?? []) queue.push(next);
  }
  return false;
}
```

This is the core of Google Zanzibar / OpenFGA-style authorization (section 10 Q10) — access is a reachability question on a relationship graph.

**Follow-up 1: BFS vs DFS — when do you choose which?**
BFS finds the *shortest* path and explores level by level — use for "shortest connection," "minimum steps," level-order traversal. DFS goes deep first — use for cycle detection, topological sort, exhaustive path exploration, and when the solution is likely deep. BFS uses a queue and more memory (holds a whole level); DFS uses a stack/recursion and less memory but risks stack overflow on deep graphs. For "is there any path" both work; for "shortest path" use BFS.

**Follow-up 2: How do you detect a cycle, and why does it matter in backend systems?**
DFS with a "currently in the recursion stack" set: if you reach a node already on the current path, there's a cycle. Or Kahn's algorithm (above) — if you can't sort all nodes, a cycle exists. It matters because cycles break things: circular dependencies prevent a build/DI graph from resolving (section 02 Q2), circular foreign keys prevent inserts without deferral (section 03 Q17), circular workflow steps loop forever. Detecting them early turns a runtime hang into a clear error.

---

*End of section 16 — and the complete 300-question set. Good luck with the interviews.*
