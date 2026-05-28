# 05 — Redis & Caching (20 Questions)

Redis powers your cache-aside layer at Tibil (40–60% latency reduction), the BullMQ job pipeline (10K+ daily jobs), and the Keycloak token cache in your SaaS project (~1s auth latency saved). These cover data structures, persistence, eviction, BullMQ, distributed locks, rate limiting, and production gotchas.

---

### Q1. What are Redis's main data structures and when do you use each?

**Answer:**

- **String** — the basic key-value type. Can hold up to 512 MB. Used for cached HTML/JSON blobs, counters (`INCR`), feature flags. Atomic ops like `INCR`, `INCRBY`, `SETNX`, `GETSET`.
- **List** — ordered, linked-list-backed. `LPUSH`/`RPUSH`/`LPOP`/`RPOP`/`LRANGE`. Used for simple FIFO/LIFO queues, recent-N items, activity feeds. Watch out for unbounded growth.
- **Hash** — field-value map within a single key. `HSET`/`HGET`/`HGETALL`. Good for storing structured objects without serializing the whole thing on every update.
- **Set** — unordered unique values. `SADD`/`SMEMBERS`/`SISMEMBER`/`SINTER`/`SUNION`/`SDIFF`. Used for tags, online-user sets, deduplication.
- **Sorted set (ZSet)** — set with a score per member; sorted by score. `ZADD`/`ZRANGE`/`ZRANGEBYSCORE`. The workhorse for leaderboards, time-ordered queues, priority queues, sliding-window rate limiters.
- **Stream** — append-only log with consumer groups (since Redis 5). Like a mini-Kafka. Used for event-streaming workloads.
- **Bitmap** — bit operations on strings. `SETBIT`/`GETBIT`/`BITCOUNT`. Compact storage for boolean per-user features ("did user N do thing X this month?"). 100M users = 12 MB.
- **HyperLogLog** — probabilistic cardinality counter. `PFADD`/`PFCOUNT`. Counts unique items with ~0.8% error in 12 KB regardless of cardinality. Good for "unique visitors this hour."
- **Geo** — lat/long indexing. `GEOADD`/`GEORADIUS`. Useful for proximity queries; internally a sorted set with geohash scores.

**Code — sorted set as a delayed job queue:**
```bash
# enqueue a job to fire at timestamp 1717890000
ZADD jobs:delayed 1717890000 "send-email:user-42"

# worker polls: any jobs due now?
ZRANGEBYSCORE jobs:delayed -inf <current-unix-ts> LIMIT 0 10
# atomically pop them with a Lua script
```

**Follow-up 1: Why is a hash often better than serializing JSON into a string?**
With JSON-as-string, you SET/GET the whole blob and can't update individual fields atomically. A hash lets you `HSET key field value` for a single field — smaller wire traffic, atomic per-field, less GC churn. Memory is also often lower because Redis ziplist-compresses small hashes.

**Follow-up 2: When is a Stream better than List + Pub/Sub?**
When you need durability, consumer groups, and replay. Lists don't have built-in consumer-group semantics; Pub/Sub doesn't persist (subscribers offline = missed messages). Streams give you both: a durable log, multiple consumer groups each with their own offset, and the ability to replay from any point.

---

### Q2. RDB vs AOF persistence — what's the difference and which do you pick?

**Answer:**

**RDB (snapshot):**
- Periodic point-in-time snapshot of the entire dataset to disk.
- `save 900 1` = save if at least 1 change in 900 seconds (default config has several such rules).
- Fast restart (just load the file).
- Crash loses everything since the last snapshot — up to minutes.

**AOF (Append-Only File):**
- Every write command appended to a log file.
- On restart, replay the log to reconstruct state.
- Three fsync policies:
  - `appendfsync always` — fsync after every command. Slow but durable.
  - `appendfsync everysec` (default) — fsync once per second. ~1s of data at risk on crash.
  - `appendfsync no` — let the OS decide. Fastest, weakest.
- AOF grows unbounded; Redis rewrites it periodically as a compact equivalent.

**Hybrid (default since Redis 7):** Use both. RDB for fast restart, AOF for low-RPO durability. AOF rewrite generates an RDB-prefixed file plus AOF tail.

**Decision matrix:**
- Cache only (data is rebuildable) → disable persistence entirely (`save ""`).
- Critical data, fast restart matters → hybrid mode with `appendfsync everysec`.
- Strict durability (financial, identity) → AOF with `always`. But consider whether Redis is the right primary store at all.

**Code — turning off persistence for a pure cache:**
```
# redis.conf
save ""
appendonly no
```

**Follow-up 1: What does "AOF rewrite" actually do?**
Redis forks a child process. The child writes the current in-memory state as a compact AOF (using the minimal commands needed to reconstruct it). Meanwhile, the parent buffers new writes. When the child finishes, the buffered writes are appended and the file swapped in. The result is a smaller AOF representing the same data.

**Follow-up 2: I lost data after a crash even with AOF `everysec` — why?**
"Everysec" means the OS fsync happens every second, so you lose up to ~1 second of writes on power loss. Also, if the OS crashes between Redis writing to the FD and the OS flushing the page cache, you can lose data. For strict durability use `always` (slower) or replicate synchronously to a slave with its own AOF.

---

### Q3. What are Redis eviction policies?

**Answer:**
When Redis hits `maxmemory`, it must evict keys to make room. The `maxmemory-policy` setting decides how.

**Policies:**
- **`noeviction`** — refuse writes with an error. Use when Redis is the source of truth and losing data is unacceptable.
- **`allkeys-lru`** — evict least-recently-used keys from the entire keyspace. The "cache" default.
- **`allkeys-lfu`** — least-frequently-used (since 4.0). Better for caches where some keys are persistently hot regardless of recency.
- **`allkeys-random`** — random eviction; only useful for benchmarks.
- **`volatile-lru`** — LRU among keys with a TTL set. Untagged keys (no TTL) are never evicted.
- **`volatile-lfu`** / **`volatile-random`** — same families restricted to keys with TTL.
- **`volatile-ttl`** — evict keys with the shortest remaining TTL first.

**For your cache-aside layer at Tibil**, you'd typically use `allkeys-lru` or `allkeys-lfu` with `maxmemory` set to about 75% of the box's RAM, leaving headroom for replication buffers, command processing, and OS.

**Code:**
```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lfu
maxmemory-samples 10        # how many keys Redis samples for LRU/LFU decisions
```

Redis's LRU isn't exact — it samples N random keys and evicts the oldest. `maxmemory-samples = 10` (default 5) gives near-exact LRU at small cost.

**Follow-up 1: Why is `allkeys-lfu` often better than `allkeys-lru` for a real cache?**
LRU evicts based on *when* a key was last touched; one access promotes it back to the front. A rarely-used but recently-touched key kicks out a frequently-used key. LFU tracks access frequency over a longer window, so a hot key remains hot even if it wasn't touched in the last minute. For a cache where some queries are perennially popular (homepage, top sellers), LFU's hit rate is noticeably better.

**Follow-up 2: I'm getting OOM errors despite `maxmemory` being set — what gives?**
`maxmemory` only counts the keyspace. Replication backlog, client output buffers, AOF rewrite memory, and Lua script memory aren't counted. Under heavy fan-out (pub/sub) or replication churn, these auxiliary buffers can grow to multi-GB. Configure `client-output-buffer-limit` and monitor `INFO memory` regularly.

---

### Q4. Explain the cache-aside pattern. What did you implement at Tibil?

**Answer:**
**Cache-aside (lazy loading):** the application is responsible for managing the cache.

```
   ┌──────────┐    1. read         ┌──────────┐
   │   App    │ ──────────────────▶│  Redis   │
   │          │ ◀──────────────────│  (hit)   │
   └──────────┘    return cached   └──────────┘
        │
        │ 2. on miss
        ▼
   ┌──────────┐    3. fetch        ┌──────────┐
   │ Postgres │ ◀──────────────────│   App    │
   │          │ ──────────────────▶│          │
   └──────────┘    return row      └──────────┘
                                        │
                                        │ 4. populate cache
                                        ▼
                                   ┌──────────┐
                                   │  Redis   │
                                   └──────────┘
```

**Code:**
```js
async function getUser(id) {
  const key = `user:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const user = await db.user.findById(id);
  if (user) await redis.setex(key, 300, JSON.stringify(user)); // TTL 5 min
  return user;
}

// Invalidate on write
async function updateUser(id, patch) {
  const updated = await db.user.update(id, patch);
  await redis.del(`user:${id}`);
  return updated;
}
```

**Pros:** simple, works with any DB, only caches what's actually read.
**Cons:** first read after a miss is slow (cold cache), and stale data is possible between DB update and cache invalidation if you `update-then-invalidate` ordering is wrong.

**Your 40–60% latency reduction at Tibil** comes from two effects: (a) keeping working-set data out of Postgres entirely (~90% of dashboard queries served from Redis with <1ms latency), (b) reducing connection pressure on Postgres so the queries that *do* hit it are faster.

**Follow-up 1: Should you invalidate or update the cache on writes?**
Invalidate. Updating creates a race: two concurrent updates can write inconsistent values to the cache, and a stale read between DB write and cache write can repopulate the cache with old data. Just delete the key — next reader fetches fresh and repopulates. This is "write-around with invalidation."

**Follow-up 2: What's the order: update DB then invalidate, or invalidate then update?**
Update DB first, then invalidate. Reversed order has a window: invalidate, then concurrent read repopulates from old DB state, then your DB update happens — cache is now stale for TTL duration. Update-then-invalidate has its own (smaller) window but is generally accepted with short TTLs as a safety net.

---

### Q5. Write-through vs write-behind vs cache-aside — when do you use each?

**Answer:**

**Cache-aside** (Q4) — app manages cache, populates on read miss. Default choice.

**Write-through:**
- Writes go to the cache, which synchronously writes to the DB.
- Reads always find the data in cache (or load on demand).
- Pros: cache and DB always in sync; reads are simple.
- Cons: every write pays cache + DB latency; cache must implement DB knowledge.

**Write-behind (write-back):**
- Writes go to the cache and are asynchronously flushed to the DB by a background process.
- Pros: writes return fast; can batch writes to the DB.
- Cons: data loss window on cache crash; complex consistency.

**Read-through:**
- Like cache-aside, but the cache (or a wrapper) handles the DB fetch on miss, not the app.
- Cleaner app code, but you need a cache layer that can talk to your DB.

**For your stack:**
- Tibil's dashboard cache → cache-aside with invalidation on write.
- A counter that's expensive to write to DB but read often → write-behind, flush every N seconds.
- A configuration cache that must reflect a write immediately → write-through.

**Code — write-behind counter:**
```js
async function incrementView(postId) {
  await redis.incr(`views:${postId}`);   // hot path: in-memory inc
  // Background job flushes accumulated counts to Postgres every 60s
}

// Flush worker
async function flush() {
  const keys = await scanKeys('views:*');
  for (const key of keys) {
    const count = await redis.getdel(key);
    if (count) await db.postViews.increment(key.split(':')[1], parseInt(count));
  }
}
setInterval(flush, 60_000);
```

**Follow-up 1: When does write-behind risk data loss?**
Between the cache write and the DB flush. If Redis crashes (no persistence or AOF lag) during that window, the writes are lost. Mitigations: AOF with `appendfsync everysec`, replication, or short flush intervals. For truly critical data, use write-through or write to DB first.

**Follow-up 2: Why do most apps default to cache-aside instead of write-through?**
Because cache-aside is implementation-cheap: the cache is "dumb" (just key-value), the app already knows how to read the DB on miss. Write-through requires the cache layer to know your DB schema and connection details — coupling that few teams want. Read-through provides similar ergonomics with a library wrapper without coupling.

---

### Q6. What are the main cache invalidation strategies?

**Answer:**
Phil Karlton: *"There are only two hard things in computer science: cache invalidation and naming things."* Here are the strategies:

1. **TTL expiration** — set an expiration on every key. Simplest. Data is at most TTL-stale.
2. **Explicit invalidation on write** — delete keys when source data changes. Most accurate but requires knowing which keys to delete.
3. **Versioned keys** — embed a version in the key. Bump the version to invalidate the whole namespace: `user:42:v3` → `user:42:v4`. Old keys age out via TTL. Great for "we just changed the schema."
4. **Tag-based invalidation** — track which keys belong to which "tag" (e.g., `tenant:42`). On write to the tag, delete all keys in it. Implementation-heavy.
5. **Stale-while-revalidate** — serve stale data, asynchronously refresh in the background. Good for content that tolerates seconds of staleness for a latency win.

**Code — versioned cache namespace:**
```js
// global version stored in Redis
const ver = await redis.get('user:ver') ?? '1';
const key = `user:${ver}:${id}`;

// invalidate everyone at once
await redis.incr('user:ver');
```

**Code — tag-based:**
```js
async function setWithTags(key, value, tags) {
  const tx = redis.multi();
  tx.set(key, value, 'EX', 3600);
  for (const t of tags) tx.sadd(`tag:${t}`, key);
  await tx.exec();
}

async function invalidateTag(tag) {
  const keys = await redis.smembers(`tag:${tag}`);
  if (keys.length) await redis.del(...keys, `tag:${tag}`);
}
```

**Follow-up 1: How do you handle invalidating a query result cache when many rows could affect it?**
The hard case. Two common patterns: (1) version the namespace and bump on any write — coarse but correct; (2) tag the cache entry with every entity it touched (e.g., a `tenant:X` tag), and bust by tag. Choose based on write/read ratio — heavy writes favor coarse invalidation, heavy reads favor fine.

**Follow-up 2: Why is "set then delete" sometimes correct order in distributed caches?**
Read-after-write consistency from a single user's perspective: if their write triggers invalidation, then they read again, they shouldn't see the old value. Update-then-delete ensures the DB has the new value before cache is cleared. Delete-then-update has the inverse risk: a concurrent reader could repopulate cache with the not-yet-updated value.

---

### Q7. What's a cache stampede and how do you prevent it?

**Answer:**
A **cache stampede** (also "thundering herd") happens when a popular key expires and many concurrent readers all miss simultaneously, each trying to populate the cache. The result: 100 concurrent expensive DB queries instead of 1, often crashing the backend.

```
   t=0:  100 requests/sec hitting cache hit (key X exists)
   t=60: key X expires
   t=60.001: 100 simultaneous misses → 100 simultaneous DB queries → DB melts
```

**Prevention strategies:**

1. **Probabilistic early expiration** — refresh the key *before* TTL with some probability that scales as TTL approaches zero. Spreads the refresh load.

2. **Lock-based recompute** — first thread that misses acquires a lock and recomputes; others wait or return stale.
```js
async function getWithLock(key, ttl, fetch) {
  const v = await redis.get(key);
  if (v) return JSON.parse(v);

  const lockKey = `lock:${key}`;
  const got = await redis.set(lockKey, '1', 'NX', 'EX', 5);
  if (!got) {
    await sleep(50);
    return getWithLock(key, ttl, fetch);   // retry
  }
  try {
    const v2 = await redis.get(key);
    if (v2) return JSON.parse(v2);
    const fresh = await fetch();
    await redis.setex(key, ttl, JSON.stringify(fresh));
    return fresh;
  } finally {
    await redis.del(lockKey);
  }
}
```

3. **Stale-while-revalidate** — return the expired value; refresh asynchronously. Keep two TTLs: "stale-after" and "expire-after." If the data is stale but not yet expired, return it AND trigger a background refresh.

4. **Jittered TTLs** — set TTL with random jitter (e.g., `300 + rand(0, 60)` seconds) so similar keys don't expire in lockstep.

**Follow-up 1: When does cache stampede *not* matter?**
When the source is fast and cheap (a small in-memory computation, a single indexed Postgres query). The 100 concurrent DB queries finish in 5ms each and nobody notices. Stampede protection adds complexity; only use it when the cost of recomputation actually matters.

**Follow-up 2: What's the difference between stale-while-revalidate and lock-based recompute?**
SWR returns stale data immediately — no caller waits. Lock-based recompute serializes the refresh — most callers wait briefly. SWR has better latency at the cost of brief staleness; lock-based has stronger freshness at the cost of latency. Pick based on whether your users can tolerate a few seconds of stale.

---

### Q8. Redis Pub/Sub vs Streams — when do you use each?

**Answer:**

**Pub/Sub:**
- Fire-and-forget broadcast. Publisher emits, all currently-connected subscribers receive.
- **No persistence.** Subscribers that aren't connected miss messages.
- **No backpressure.** Slow subscribers can be disconnected.
- O(N) work per publish (N = number of subscribers).
- Good for: cache invalidation broadcasts, real-time notifications where missing one is OK.

**Streams:**
- Append-only log with offsets. Subscribers track their own position.
- **Persistent.** Reconnecting subscribers resume from their last ID.
- **Consumer groups.** Multiple consumers can share the workload of one stream, each getting unique messages.
- Acknowledgment, pending-message tracking, replay.
- Good for: event-driven workflows, durable job queues, audit logs.

**Code — Pub/Sub for cache invalidation:**
```js
// Publisher (after writing to DB)
await redis.publish('user:invalidated', userId);

// Subscriber (each app instance, on startup)
sub.subscribe('user:invalidated');
sub.on('message', (_chan, userId) => localCache.delete(userId));
```

**Code — Stream as a job queue:**
```js
// Producer
await redis.xadd('jobs', '*', 'type', 'send-email', 'to', 'user@x.com');

// Consumer group setup (once)
await redis.xgroup('CREATE', 'jobs', 'workers', '$', 'MKSTREAM');

// Worker loop
while (true) {
  const msgs = await redis.xreadgroup(
    'GROUP', 'workers', 'worker-1',
    'COUNT', 10, 'BLOCK', 5000,
    'STREAMS', 'jobs', '>'
  );
  for (const [, entries] of msgs ?? []) {
    for (const [id, fields] of entries) {
      await process(fields);
      await redis.xack('jobs', 'workers', id);
    }
  }
}
```

**Follow-up 1: Why would I use Pub/Sub over Kafka for events?**
For ephemeral, low-stakes events where Kafka's operational weight isn't justified. Cache invalidation, presence updates, "user N is typing" — Pub/Sub is one line of code, zero infra beyond Redis you already have. Kafka becomes worthwhile when you need durability, replay, or consumer parallelism.

**Follow-up 2: How does Stream's consumer group compare to Kafka's?**
Conceptually identical: workers in a group divide messages among themselves; each message goes to one worker. Differences: Redis Streams have explicit pending-entries lists you can query and reclaim (useful for crashed-worker recovery via `XCLAIM`); Kafka does partition rebalancing more automatically. Streams scale to lower throughput than Kafka but with less ops cost.

---

### Q9. How does BullMQ work internally? Walk me through your "10K+ jobs/day" pipeline.

**Answer:**
**BullMQ** is a Node job queue built on Redis. Producers add jobs to a queue; workers pull and process them. Underneath, BullMQ uses several Redis data structures:

- **`wait` list** — newly added jobs, FIFO.
- **`active` set** — jobs currently being processed (per worker, with timestamp).
- **`delayed` zset** — scheduled-for-later jobs, scored by execute-at timestamp.
- **`completed` / `failed` zsets** — outcome history.
- **`prioritized` zset** — when priorities are used.

Worker flow:
1. Move a job from `delayed` to `wait` if its time has come (Lua script).
2. Atomically pop from `wait`, push to `active`.
3. Execute the job handler.
4. On success, move to `completed`; on failure, move to `failed` or retry (back to `wait` with backoff via `delayed`).

Heavy reliance on Lua scripts for atomicity — the move-job-between-lists operation is a single script that holds Redis's single-threaded execution model accountable.

**Code — basic BullMQ usage:**
```js
import { Queue, Worker } from 'bullmq';

const emails = new Queue('emails', { connection: redisConn });

// Producer
await emails.add('send-welcome', { userId: 'u1' }, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 1000 },
  removeOnComplete: 1000,            // keep last 1000
  removeOnFail: 5000,
});

// Worker
new Worker('emails', async job => {
  if (job.name === 'send-welcome') {
    await sendEmail(job.data.userId);
  }
}, { connection: redisConn, concurrency: 10 });
```

**For your 10K jobs/day pipeline:** queues for email, webhooks, and report-generation, each with workers tuned for concurrency × duration. Reports run with concurrency 2 (CPU-heavy); emails run with concurrency 50 (I/O-bound). Total worker count chosen so that p95 wait time is under 30s during peak.

**Follow-up 1: Why doesn't a job get lost if a worker crashes mid-process?**
BullMQ uses a "stalled job" mechanism. Each worker periodically refreshes a heartbeat on its active jobs. If a worker crashes, the heartbeat expires; a stalled-job checker finds the orphan and either retries or fails it. Configurable via `stalledInterval` and `maxStalledCount`.

**Follow-up 2: What happens at very high job throughput (>1K/sec)?**
You may hit Redis limits — particularly if jobs are large (payload size) or the queue has heavy contention on the same Lua scripts. Mitigations: split into multiple queues per type, use a Redis cluster, or move to a dedicated queue service (Kafka, SQS) for ultra-high throughput. BullMQ comfortably handles 1K–5K jobs/sec on a single Redis with modest specs; beyond that, architect around it.

---

### Q10. How would you implement a distributed lock with Redis?

**Answer:**
The naïve version: `SET lock NX EX 30`. If the SET succeeds, you hold the lock; release with `DEL`. This works for single-instance Redis with low contention.

**The right way (single instance):**
```js
const lockId = crypto.randomUUID();
const acquired = await redis.set('lock:user:42', lockId, 'NX', 'EX', 30);
if (!acquired) return false;

try {
  await criticalSection();
} finally {
  // Release only if we still own it (check-and-delete via Lua)
  await redis.eval(`
    if redis.call('GET', KEYS[1]) == ARGV[1] then
      return redis.call('DEL', KEYS[1])
    else
      return 0
    end`,
    1, 'lock:user:42', lockId,
  );
}
```

The random `lockId` and Lua release prevent the classic bug: A acquires lock, A's process pauses for 30s (GC, OS hibernate), lock expires, B acquires, A wakes up and `DEL`s B's lock.

**The Redlock algorithm (multi-instance):** Antirez's protocol for redundancy across N Redis instances. Acquire on the majority within a time bound. Pros: tolerates N/2-1 instance failures. Cons: Martin Kleppmann's well-known critique — under clock skew or pause, correctness can fail. Use with care for non-critical mutexes; don't bet financial correctness on it.

**For most apps:**
- Use single-Redis SET-NX with a token and Lua release for "mostly-correct" locks.
- For correctness-critical locks (financial transactions), use Postgres `SELECT FOR UPDATE` instead — strictly correct, predictable.

**Follow-up 1: What's wrong with `SETNX` followed by `EXPIRE` as two commands?**
If your process crashes between SETNX and EXPIRE, the lock has no expiration — held forever. Use `SET key value NX EX seconds` as a single atomic command.

**Follow-up 2: How do you handle lock renewal for long-running operations?**
Periodically refresh the TTL while the work continues — a "watchdog" timer in the worker. If the worker dies, the watchdog dies with it, and the lock expires naturally. BullMQ's job heartbeat is exactly this pattern. Caveat: if your work pauses (GC, network stall) longer than the TTL, someone else can acquire — design idempotent operations.

---

### Q11. How would you implement rate limiting with Redis?

**Answer:**
Three classic algorithms, each implementable in Redis:

**Fixed window:**
- `INCR rate:user:42:202605271200; EXPIRE 60` — count requests per minute.
- Simple, but boundary issue: 100 requests at 11:00:59 + 100 at 11:01:00 = 200 in two seconds, but each window sees only 100.

**Sliding window log:**
- Sorted set of timestamps per user. `ZADD rate:user:42 <now> <now>; ZREMRANGEBYSCORE ... -inf <now-60s>; ZCARD ...`
- Exact, but stores N timestamps per user — memory-heavy for high limits.

**Sliding window counter (approximate):**
- Track current window count + previous window count; estimate by weighted blend. Cheap and reasonably accurate.

**Token bucket:**
- Each user has a bucket of N tokens that refills at R tokens/sec. Each request consumes a token; fail if empty.
- Allows bursts within a cap.
- Implementable with Lua to avoid races.

**Code — sliding window log with Lua:**
```lua
-- Lua script atomically counts and adds
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)
local count = redis.call('ZCARD', key)
if count < limit then
  redis.call('ZADD', key, now, now)
  redis.call('EXPIRE', key, window)
  return 1
else
  return 0
end
```

In Node, use `rate-limiter-flexible` — it implements all of these with Redis backends.

**Follow-up 1: How does Redis-based rate limiting work across multiple app instances?**
That's the whole point — they all increment the same key. As long as time is synchronized (NTP), every instance sees the same counter. This is why in-memory rate limiting on each instance is broken: a user spreading requests across 5 instances effectively gets 5x the limit.

**Follow-up 2: What's the performance cost of Lua-scripted rate limiting?**
Negligible. Lua scripts run atomically inside Redis's single thread. A token-bucket script is a handful of operations; thousands per second per Redis is easy. The limit is usually Redis's CPU on the sorted set ops, around 100K ops/sec on a modest box.

---

### Q12. What are Redis pipelines and transactions? When use each?

**Answer:**

**Pipelining** — send multiple commands to Redis in one batch without waiting for replies in between. The client reads all replies after sending all commands.
- Reduces network round-trips dramatically.
- Not atomic — other clients' commands can interleave.
- Use when you have many independent commands and care about throughput.

**Transactions (MULTI/EXEC)** — queue commands and execute them atomically as a block.
- Atomic — no other commands run between MULTI and EXEC.
- Not rollback-capable — if a command errors, others still execute.
- Combined with `WATCH` for optimistic locking on specific keys.

**Lua scripts** — the modern alternative to MULTI/EXEC. Atomic, support conditionals, single round-trip.

**Code:**
```js
// Pipeline — fast, not atomic
const pipe = redis.pipeline();
for (const id of userIds) pipe.get(`user:${id}`);
const results = await pipe.exec();      // [[err, val], [err, val], ...]

// Transaction — atomic, fewer use cases
const multi = redis.multi();
multi.incr('counter');
multi.incr('counter');
const [r1, r2] = await multi.exec();    // both ran atomically

// Optimistic transaction with WATCH
await redis.watch('balance');
const current = parseInt(await redis.get('balance'));
const tx = redis.multi();
tx.set('balance', current - 100);
const result = await tx.exec();         // null if balance changed during the transaction
```

**Follow-up 1: Why doesn't Redis support rollback on transaction errors?**
By design — Antirez argued that command errors inside MULTI/EXEC are programmer errors (wrong type for operation, missing args), not transient runtime issues. Atomic rollback would add complexity to a system whose simplicity is its strength. Use the right type / right args, and you don't need rollback.

**Follow-up 2: When does pipelining go wrong?**
Two cases. (1) If commands depend on previous results, you can't pipeline — each command needs the prior reply. (2) Sending huge pipelines can blow up the server's output buffer; chunk in batches of a few thousand. Common surprise: pipeline a million commands and watch Redis OOM trying to buffer the replies.

---

### Q13. How do you use Lua scripting in Redis and when is it the right tool?

**Answer:**
Redis executes Lua scripts atomically — nothing else runs while a script is in flight. This makes Lua the right tool for **multi-step atomic operations** that can't be expressed as a single command.

**Examples where Lua wins:**
- "INCR, but only if current value < N" — read-modify-write atomically.
- Rate limiters with sliding-window logic.
- Custom data structures (e.g., bounded queue: push + trim atomically).
- Conditional updates based on existing state.

**Code — atomic "INCR with cap":**
```lua
-- KEYS[1] = counter key, ARGV[1] = cap
local v = tonumber(redis.call('GET', KEYS[1]) or '0')
if v >= tonumber(ARGV[1]) then return 0 end
redis.call('INCR', KEYS[1])
return 1
```

```js
const allowed = await redis.eval(script, 1, 'requests', 100);
```

**Caching the script** — `EVALSHA` after `SCRIPT LOAD` avoids re-sending the body each call. Most Node clients (ioredis) cache automatically with `defineCommand`:

```js
redis.defineCommand('incrCapped', { numberOfKeys: 1, lua: '...' });
await redis.incrCapped('requests', 100);
```

**Cost:**
- Long-running scripts block Redis entirely. Keep scripts under a few milliseconds.
- Script execution counts toward `slowlog`.
- Debugging is harder than client-side code.

**Follow-up 1: When should I prefer Lua over multiple round trips?**
When the multiple steps need to be atomic (no other client can interleave) or when the round-trip count is high relative to the work. A single Lua call replacing 5 round-trips can cut latency by 4× on a remote Redis.

**Follow-up 2: What's the difference between EVAL and FUNCTION?**
Functions (Redis 7+) are like persistent server-side stored procedures: load once, call by name forever (across restarts with AOF). EVAL/EVALSHA scripts disappear on Redis restart (scripts must be re-cached). Functions are the modern way for libraries that want consistent server-side helpers.

---

### Q14. How does Redis Cluster work? When do you need it?

**Answer:**
Redis Cluster shards data across multiple Redis nodes. The keyspace is divided into **16384 hash slots**; each key maps to a slot (`CRC16(key) mod 16384`), each slot belongs to one master. Clients learn the slot-to-node mapping and talk directly to the right node — no proxy.

**Topology:**
- 6+ nodes typical: 3 masters + 3 replicas (one per master).
- Each master owns a range of slots.
- Replicas can be promoted on master failure (Sentinel-style logic built-in).

**When you need Cluster:**
- Working set exceeds one machine's RAM.
- Single-node Redis is CPU-saturated.
- Need write throughput beyond one Redis instance.

**Code — connecting from ioredis:**
```js
const cluster = new Redis.Cluster([
  { host: 'redis-1', port: 6379 },
  { host: 'redis-2', port: 6379 },
  { host: 'redis-3', port: 6379 },
]);
```

**Hash tags** — to ensure related keys land on the same node (for multi-key ops):
```
{user:42}:profile     -- {user:42} is hashed; both keys land on same slot
{user:42}:sessions
```

**Limitations:**
- Multi-key ops only work if all keys are on the same slot (use hash tags).
- Transactions across slots are not supported.
- Pub/Sub broadcasts across the cluster (every node sees every publish) — doesn't scale by sharding.

**Follow-up 1: What happens during a cluster resharding?**
Slots can be migrated from one master to another. During migration, the source node sends `MOVED` or `ASK` redirects for keys being moved. Clients follow them transparently. Done well, resharding has zero downtime; done badly, it spikes latency.

**Follow-up 2: When should I use Redis Sentinel instead of Cluster?**
Sentinel is for HA without sharding — N replicas of one master, automatic failover. Use when your data fits in one node but you need high availability. Cluster is for sharding (which also provides HA via per-shard replicas). Sentinel is simpler; Cluster is more capable but more operationally complex.

---

### Q15. Talk about your Keycloak token caching pattern. How did caching cut auth latency by ~1 second?

**Answer:**
Without caching, every authenticated request to your services did this:
1. Receive request with `Authorization: Bearer <token>`.
2. Call Keycloak's `/userinfo` or `/token/introspect` endpoint.
3. Keycloak validates the token (potentially DB lookups).
4. Return user info to your service.

That's a network round-trip to Keycloak + Keycloak's own processing per request — typically 200–1000ms depending on Keycloak's load and network.

**With Redis caching:**
1. Hash the token (don't store tokens directly).
2. Cache the introspection result keyed by hash, TTL set to (token TTL - safety margin).
3. On request: check Redis; on hit, use cached claims; on miss, call Keycloak, cache result.
4. On token revocation: publish a message that clears the cache entry.

**Code:**
```js
async function verifyToken(token) {
  const hash = sha256(token);
  const cached = await redis.get(`auth:${hash}`);
  if (cached) return JSON.parse(cached);

  const claims = await keycloak.introspect(token);
  if (!claims.active) throw new UnauthorizedException();

  // TTL: token expiration minus 30s safety margin
  const ttl = claims.exp - Math.floor(Date.now() / 1000) - 30;
  if (ttl > 0) await redis.setex(`auth:${hash}`, ttl, JSON.stringify(claims));
  return claims;
}

// Invalidation listener (Keycloak event → Redis pub/sub)
subscriber.on('message', (_chan, payload) => {
  const { tokenHash } = JSON.parse(payload);
  redis.del(`auth:${tokenHash}`);
});
```

**Why ~1s of savings:**
- Cache hit: ~1ms Redis lookup.
- Cache miss: full Keycloak round-trip (~200–800ms on a busy Keycloak).
- With ~95%+ hit rate, average latency drops dramatically. On a service that authenticates 10K times per minute, that's >10K Keycloak calls/minute avoided.

**Follow-up 1: How do you handle revocation if cached?**
Two patterns: (1) short TTLs (5–15 minutes) — accept slight delay between revocation and expiration; (2) explicit invalidation — Keycloak emits an event on logout/revocation; a listener clears the cache. Most apps tolerate 5-minute revocation lag for the latency win.

**Follow-up 2: What if the cached claims are stale (user's roles changed)?**
Same answer: short TTL + invalidation on role change. The trade-off is the cost of staleness vs the latency win. For most apps, a 5–10 minute role-update delay is acceptable; for high-security apps, invalidate on role change explicitly.

---

### Q16. How do you keep Redis memory under control?

**Answer:**

**Per-key strategies:**
- **TTLs everywhere.** Untagged keys are forever; tag everything cacheable with `EX`/`EXPIRE`.
- **Compress values.** Gzip/brotli before storing if values are large text/JSON.
- **Hashes for small objects.** Redis ziplist-compresses hashes with few small fields — much smaller than equivalent strings.
- **Bitmaps and HyperLogLogs** for set-like data with millions of members.
- **Avoid storing huge values.** A single multi-MB value blocks Redis during reads/writes.

**Configuration:**
- `maxmemory` set to ~75% of RAM.
- `maxmemory-policy` matched to use case (Q3).
- `lazy-free` settings — let Redis free large objects asynchronously to avoid blocking on `DEL` of a 1M-element list.

**Monitoring:**
- `INFO memory` — `used_memory`, `used_memory_rss`, fragmentation ratio.
- `MEMORY USAGE <key>` — size of a specific key.
- `MEMORY STATS` — per-category breakdown.
- Watch `evicted_keys` and `keyspace_misses`/`keyspace_hits` for cache effectiveness.

**Code — finding the biggest keys:**
```bash
redis-cli --bigkeys
# Samples the keyspace, reports the largest key in each type.
```

**Follow-up 1: What's `mem_fragmentation_ratio` and when should I worry?**
RSS (memory the OS sees) divided by used_memory (what Redis allocated). Ratio > 1.5 means significant fragmentation — Redis allocates but can't release back to OS because of internal free-list structure. Mitigation: enable `activedefrag yes` for online defragmentation.

**Follow-up 2: My Redis memory keeps growing even though I'm setting TTLs — why?**
Possible causes: (1) keys without TTLs accumulating (check with `redis-cli --scan` + `TTL`); (2) replication backlog growing (large `repl-backlog-size`); (3) client output buffers for slow consumers (Pub/Sub subscribers that can't keep up); (4) Lua script memory (rare). Inspect `INFO memory` and `INFO clients` for the breakdown.

---

### Q17. How does Redis replication work and what's the failure model?

**Answer:**
**Asynchronous replication:** master sends a stream of writes to replicas. Replicas apply them in order. Replicas can serve reads but lag the master by some amount.

**Failure scenarios:**
- **Master crashes:** writes after the last replicated offset are lost. With Sentinel, a replica is promoted; clients reconnect.
- **Replica disconnects:** when it reconnects, it resumes from its last offset if within the `repl-backlog-size`. Otherwise full resync (master sends a snapshot).
- **Network partition (split-brain):** old master keeps accepting writes while new master is elected; old master's writes are lost when it rejoins (this is acknowledged in Redis docs).

**WAIT command** — block until N replicas confirm a write. Synchronous-ish replication. Doesn't guarantee durability, but trades some latency for stronger replication.

**For HA:** use **Sentinel** (small cluster of monitoring processes that detect master failure and orchestrate promotion). Modern setups use Sentinel for non-sharded HA, or Cluster for sharded HA.

**Follow-up 1: Why is "replication is async" a problem?**
Because acknowledged writes can be lost on master failure. If a client sees `OK` and immediately the master dies before replicating, the data is gone. For caches this is usually fine; for primary data, layer durable storage behind Redis (Postgres) and treat Redis as a cache, not a source of truth.

**Follow-up 2: How does WAIT differ from synchronous replication?**
`WAIT n timeout` blocks the client until at least n replicas have ACK'd the write or the timeout expires. It returns the number of ACKs received — doesn't guarantee n; just reports. Synchronous replication would refuse the write entirely if replication failed; WAIT is "best-effort confirmation."

---

### Q18. How would you choose Redis cluster mode vs Sentinel vs single-instance?

**Answer:**

**Single instance:**
- One Redis process. Backups via RDB/AOF.
- Simplest. Right for small projects, dev environments, or stateless caches that can rebuild.
- Failure = downtime.

**Sentinel:**
- One master, N replicas, M sentinel monitors.
- Sentinels detect master failure and promote a replica.
- Clients use a "Sentinel-aware" connection that asks sentinels for the current master.
- Right for production caches/queues where one machine's RAM is enough.

**Cluster:**
- Multiple masters sharding the keyspace; each master has replicas.
- Built-in failover (no separate Sentinel daemon).
- Right when one machine isn't enough — large working sets, high throughput.

**Capacity rule of thumb:**
- < 50 GB data, < 50K ops/sec, can tolerate occasional downtime → single.
- < 50 GB data, < 50K ops/sec, need HA → Sentinel.
- > 50 GB data or > 100K ops/sec → Cluster.

**Follow-up 1: What's the operational cost difference?**
Single = trivial. Sentinel = moderate (need 3+ sentinel processes, client config). Cluster = significant (resharding, hash tags, multi-key constraints, more failure modes). Don't reach for Cluster until you genuinely need it.

**Follow-up 2: Can I use managed Redis (ElastiCache, Memorystore) instead of running my own?**
Yes, and you usually should. Managed Redis takes care of patching, failover, backups, monitoring. The cost is some flexibility (you can't tune every config) and price. For most teams, the time saved on ops vastly outweighs the cost. Reach for self-hosted only when managed doesn't fit (specific Redis modules, on-prem requirements).

---

### Q19. What's the right Redis client and connection pattern in Node?

**Answer:**
**Popular clients:**
- **`ioredis`** — most common in NestJS/Node ecosystem. Supports Cluster, Sentinel, pipelining, Lua, modern features. Has good Promise API.
- **`node-redis` (v4+)** — official client. Modern Promise API since v4. Slightly less feature-rich for Cluster than ioredis.
- **`redis` (legacy v3)** — callback API, avoid for new code.

**Connection pattern:**
- **One shared client per process.** Redis multiplexes commands over a single TCP connection efficiently. Spawning a client per request is wasted overhead.
- **Separate clients for blocking ops.** `BLPOP`, subscriber connections, Streams `XREADGROUP BLOCK` all block the connection. Don't share with normal request handling.
- **Reconnection.** ioredis handles reconnect with backoff automatically; queue commands during disconnect. Configure `maxRetriesPerRequest` to avoid infinite hangs.

**Code:**
```js
import IORedis from 'ioredis';

export const redis = new IORedis(process.env.REDIS_URL, {
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  // For BullMQ in NestJS
  // maxRetriesPerRequest: null,
});

// Separate subscriber client — blocks on subscribe loop
export const subscriber = redis.duplicate();
await subscriber.subscribe('events');
subscriber.on('message', (chan, msg) => handle(chan, msg));
```

**For BullMQ specifically:** `maxRetriesPerRequest: null` because BullMQ uses its own retry/reconnect logic and the default would interfere.

**Follow-up 1: Why do I need a separate subscriber connection?**
Because the connection is in subscribe mode after `SUBSCRIBE` — Redis only accepts `(UN)SUBSCRIBE` and quit commands on it. If you tried `GET` on the same connection, it would error. Same for blocking commands like `BLPOP` — they monopolize their connection.

**Follow-up 2: How many Redis connections can I have?**
Redis defaults to `maxclients 10000`. Practically, you want far fewer — high connection counts increase Redis's per-poll overhead and consume memory. Aim for tens of connections per app instance (one main client, one or two subscribers, plus BullMQ's pool), not hundreds.

---

### Q20. What are the most common Redis production gotchas?

**Answer:**

1. **No persistence enabled when needed.** Cache restart = cold cache, but cache *primary store* = data loss. Decide deliberately.

2. **No `maxmemory` set.** Redis grows until OOM-killed. Always cap.

3. **Slow commands blocking the loop.** `KEYS *`, `SUNION` on huge sets, `SORT` on huge lists — all O(N) and run inline on Redis's single thread. Use `SCAN` instead of `KEYS`; bound your set sizes.

4. **Big keys.** A 100 MB list operated on with `LRANGE 0 -1` blocks Redis for seconds. Find and break up with `--bigkeys`.

5. **Pub/Sub assuming durability.** Subscribers offline = missed messages, no replay. Use Streams if you need durability.

6. **Replication backlog too small.** Replica falls slightly behind, can't catch up via backlog, triggers full resync — heavy on the master. Bump `repl-backlog-size` for write-heavy workloads.

7. **AOF `everysec` data loss surprise.** "Durable" is a spectrum; "everysec" still loses ~1s on crash.

8. **Eviction-friendly policy on data that shouldn't be evicted.** Mixed-use Redis (cache + queue + session) with `allkeys-lru` — queues and sessions get evicted. Either separate Redis databases or use `volatile-lru` with persistent keys having no TTL.

9. **Latency from large TLS handshakes.** TLS to Redis adds ~5ms per new connection. Use connection pooling and persistent connections.

10. **Single point of failure.** Single-instance Redis = single point of failure for everything that depends on it. Plan for it.

11. **No monitoring of `latency` metrics.** `LATENCY HISTORY` and `LATENCY GRAPH` are gold for diagnosing intermittent stalls.

12. **Misusing `EXPIRE` semantics.** `EXPIRE key 0` deletes immediately; negative values also delete. Setting TTL on a key that gets `SET` again resets the TTL (use `SET ... KEEPTTL` to preserve).

**Follow-up 1: I see periodic latency spikes in Redis — how do I find the cause?**
Enable `latency-monitor-threshold 10` (capture events > 10ms). Run `LATENCY DOCTOR` for a diagnosis. Common causes: AOF fsync on slow disk, RDB snapshot fork (memory copy-on-write), big keys deleted synchronously, network blip. Each has a specific fix.

**Follow-up 2: How do you migrate Redis without downtime?**
For single-instance: set up the new Redis as a replica of the old (`REPLICAOF host port`), wait for sync, switch clients to new, promote new with `REPLICAOF NO ONE`, decommission old. For Cluster: more involved (resharding tools), but the principle is the same — replicate, switch, promote.

---

*End of section 05. Next: Kafka & Messaging (20 questions).*
