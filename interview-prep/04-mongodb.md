# 04 — MongoDB (15 Questions)

MongoDB powered your work at Dhee Coding Lab (20+ REST APIs, ~5K users, aggregation optimization). These cover the document model, schema design choices, aggregation, indexing, replication, and the production gotchas that bite Mongo teams.

---

### Q1. When is the document model genuinely better than a relational one?

**Answer:**
MongoDB's document model embeds nested data inside a single document. The fit/not-fit question comes down to **how the data is accessed**.

**Good fit:**
- Aggregate is read/written as a whole — a `User` with `address`, `preferences`, `recentOrders[]` is usually fetched together.
- Schema is naturally variable across records — e.g., a `webhook_event` whose payload shape depends on event type.
- One-to-few embeddings where the "few" is bounded (a user has < 1000 addresses, not unbounded). Bounded embeds keep the document small.
- High write volume with a clear partition key — Mongo scales out via sharding more easily than vanilla Postgres.

**Bad fit:**
- Many-to-many relationships dominate the model.
- Strong cross-entity transactions are common (Mongo has them, but the model isn't optimized for them).
- Reports and analytics drive most queries — joins and aggregates work, but Postgres + a typed schema is usually faster and clearer.
- Schema discipline matters — Mongo doesn't enforce shape by default, so codebases get messy without strict app-level validation.

**Code — a denormalized order document:**
```js
{
  _id: ObjectId(...),
  userId: ObjectId(...),
  items: [
    { productId: ObjectId(...), name: 'Widget', priceAtPurchase: 999, qty: 2 },
    { productId: ObjectId(...), name: 'Gadget', priceAtPurchase: 1499, qty: 1 }
  ],
  shippingAddress: { line1: '...', city: '...', country: '...' },
  total: 3497,
  status: 'paid',
  createdAt: ISODate('2026-05-27T...')
}
```
Reading an order = one document fetch. Compare to relational: orders + order_items + addresses, three queries or a join.

**Follow-up 1: Why embed `priceAtPurchase` instead of joining to the products table?**
Because the price at the time of the order is a historical fact, not the current product price. Embedding snapshots the value — if the product is later renamed or repriced, the order record is unchanged. This is denormalization done right: storing the immutable historical fact.

**Follow-up 2: What's the 16MB document limit and when do you hit it?**
Mongo caps single documents at 16 MB. Hit it when you embed unbounded arrays — comment threads, event logs, telemetry inside the parent. Fix: stop embedding, switch to a separate collection with a reference. The 16 MB cap is a useful pressure to keep documents focused.

---

### Q2. Embed vs reference — what's the decision framework?

**Answer:**
Three classic patterns:

**Embed (one-to-one or one-to-few, bounded):**
```js
{ _id, user: { name, address, preferences } }
```
- Pro: single document read.
- Con: hard to query the embedded thing independently.

**Reference (one-to-many, unbounded or large):**
```js
// users collection
{ _id: 'u1', name: 'Alice' }
// orders collection
{ _id: 'o1', userId: 'u1', total: 100 }
```
- Pro: orders can grow without bloating user doc; query orders by their own fields.
- Con: need `$lookup` or two queries to fetch user + orders together.

**Subset / bucketing (one-to-many, bounded snapshot):**
```js
// user keeps the *latest* N orders embedded for fast home-page reads
{ _id: 'u1', name: 'Alice', recentOrders: [ ...latest 5 ] }
// while the full history lives in the orders collection
```

**Decision factors:**
1. **Cardinality** — embed if bounded; reference if unbounded.
2. **Access pattern** — embed if always read together; reference if one is read alone often.
3. **Volatility** — embed if data is immutable post-creation (snapshots); reference if it changes independently (current product price).
4. **Document size** — never embed where total can exceed 16 MB or grow without bound.

**Follow-up 1: What's the "Outlier pattern"?**
When 99% of your documents fit the embedded model but 1% blow past size limits. Solution: keep the small ones embedded, mark outliers with a flag, store their data in a separate collection. Avoids worst-case complexity dragging the design down for the common case.

**Follow-up 2: How do you handle a many-to-many in Mongo?**
Either store IDs in both directions (`user.followedTagIds`, `tag.followerUserIds`) — fine if both sides are bounded — or use a join collection (`tag_followers: { userId, tagId, since }`). The join-collection version is exactly the same as a relational join table; if your model is dominated by these, consider whether Postgres is a better fit.

---

### Q3. What index types does MongoDB support and when do you use each?

**Answer:**
MongoDB supports several index types, each tuned for a query pattern:

- **Single field** — the default `db.users.createIndex({ email: 1 })`.
- **Compound** — multiple fields, order matters (next question).
- **Multikey** — automatic when indexing an array field; one index entry per array element.
- **Text** — for full-text search (`$text` operator). One per collection.
- **Hashed** — for hash-based sharding or equality lookups; useless for range queries.
- **Geospatial (2dsphere)** — for `$near`, `$geoWithin` queries on GeoJSON.
- **TTL** — auto-deletes documents after the indexed time field exceeds a value. Useful for sessions, OTPs, ephemeral data.
- **Wildcard** — indexes all fields matching a path pattern (`{ "attrs.$**": 1 }`); for variable-shaped documents.
- **Partial** — like Postgres partial indexes; index only documents matching a filter.
- **Unique** — enforces uniqueness; can be partial-unique.

**Code:**
```js
// Compound for the common "list this user's recent orders"
db.orders.createIndex({ userId: 1, createdAt: -1 });

// TTL — auto-expire sessions 30 days after lastSeen
db.sessions.createIndex({ lastSeen: 1 }, { expireAfterSeconds: 60 * 60 * 24 * 30 });

// Partial — only index active accounts
db.users.createIndex(
  { email: 1 },
  { unique: true, partialFilterExpression: { active: true } }
);

// Text
db.posts.createIndex({ title: 'text', body: 'text' });
db.posts.find({ $text: { $search: 'mongodb performance' } });
```

**Follow-up 1: How does Mongo's "covered query" work?**
When the index contains every field referenced by the query (filter + projection + sort) and no field needs `_id`, Mongo answers the query entirely from the index — no document fetch. Similar to Postgres's index-only scan. Confirm with `explain()`; look for `totalDocsExamined: 0`.

**Follow-up 2: What's the cost of too many indexes?**
Each write must update every applicable index — N indexes mean N times the write work. Indexes also consume RAM (working set should fit in memory). Common smell: collection with 20 indexes, half unused. Use `db.collection.aggregate([{$indexStats:{}}])` to find unused indexes and drop them.

---

### Q4. Explain compound index column order with the "ESR" rule.

**Answer:**
ESR — **Equality, Sort, Range**. The order of fields in a compound index should be:
1. **Equality** filters first.
2. **Sort** fields next.
3. **Range** filters last.

This ordering maximizes both selectivity and the index's ability to provide an already-sorted result, avoiding an in-memory sort.

**Worked example — "this user's recent orders over $100":**
```js
db.orders.find(
  { userId: 'u1', amount: { $gt: 100 } }     // equality: userId, range: amount
).sort({ createdAt: -1 });

// ESR-correct index
db.orders.createIndex({ userId: 1, createdAt: -1, amount: 1 });
// E (userId) → S (createdAt) → R (amount)
```

If you put range first (`{ amount: 1, userId: 1, createdAt: -1 }`), Mongo would scan a wide range of amounts and then filter on userId — far more index entries scanned. With ESR, it jumps straight to the user's orders, returns them in sort order, and filters by amount as it goes.

**Follow-up 1: Does an index `{a:1, b:1, c:1}` help a query on `{b: 1}`?**
No, or only marginally. Compound indexes are useful only when the query uses a **prefix** of the fields. Querying on `b` alone forces a full index scan (better than a collection scan, but not great). Mongo does have "index intersection" for some cases, but compound prefixes are strictly faster.

**Follow-up 2: What if I have multiple sort orders for the same query?**
Use the index in *both* directions. An index `{ createdAt: 1 }` serves both `sort({createdAt:1})` (forward scan) and `sort({createdAt:-1})` (reverse scan) equally well. But a compound `{a:1, b:-1}` is **not** the same as `{a:-1, b:1}` — direction matters when there are multiple sort fields.

---

### Q5. How does the aggregation pipeline work? Why does stage order matter?

**Answer:**
The aggregation pipeline is a sequence of **stages**, each transforming documents and passing results to the next stage. Stages include `$match` (filter), `$project` (reshape), `$group` (aggregate), `$sort`, `$lookup` (join), `$unwind` (flatten array), `$facet` (parallel sub-pipelines).

**Order matters for two reasons:**
1. **Performance** — `$match` early filters out documents before later expensive stages.
2. **Index use** — `$match` and `$sort` at the *start* can use indexes; later in the pipeline they operate on the intermediate result and cannot.

**Code — your "improved read query performance by 20–30%" optimization, sketched:**
```js
// Before — slow
db.events.aggregate([
  { $group: { _id: '$userId', count: { $sum: 1 } } },
  { $match: { _id: 'u1' } },                            // filter AFTER group — wasted work
  { $sort: { count: -1 } }
]);

// After — push $match before $group
db.events.aggregate([
  { $match: { userId: 'u1' } },                         // uses index on userId
  { $group: { _id: '$userId', count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]);
```

Mongo's planner does some reordering automatically (`$match` push-down), but don't rely on it for complex pipelines.

**Other useful stages:**
- `$facet` — run multiple sub-pipelines in parallel on the same data. Great for dashboard endpoints returning multiple aggregates in one query.
- `$bucket` / `$bucketAuto` — histogram-style grouping by value ranges.
- `$graphLookup` — recursive traversal for hierarchies (org charts, comment threads).
- `$merge` / `$out` — materialize results into another collection (poor-man's materialized view).

**Follow-up 1: What's `$lookup` and what are its limits?**
`$lookup` is Mongo's join operator. It performs a left outer join from the current collection to another. Limits: foreign collection must be in the same database, the operation is uncached, and it can be slow if the joined collection lacks an index on the join field. For high-traffic endpoints, denormalize instead of relying on `$lookup`.

**Follow-up 2: How do you profile an aggregation pipeline?**
`db.collection.aggregate(pipeline, { explain: true })` returns the executed plan, including which stages used indexes, how many documents flowed through each stage, and where time was spent. Look for `IXSCAN` (good) vs `COLLSCAN` (bad) in the first stage, and check the document counts at each step — if a stage produces an order-of-magnitude more docs than the next stage filters out, there's wasted work.

---

### Q6. Explain read and write concerns. How do they affect consistency vs latency?

**Answer:**
MongoDB exposes per-operation tunables for "how durable / how consistent do you need this?"

**Write concern (`w`)** — how many replicas must acknowledge a write before the driver returns success.
- `w: 1` — primary only. Fast, but if primary fails before replication, write is lost.
- `w: 'majority'` — wait for majority of replicas. Safer; survives primary failure. **Default since 5.0.**
- `w: <number>` — specific count.
- `j: true` — wait for journal flush on the acknowledging nodes; ensures durability past process crash.

**Read concern** — how recent / how durable does the data you read need to be?
- `local` — primary's latest, regardless of whether it's replicated. May get rolled back.
- `available` — even more lenient; used for secondary reads.
- `majority` — only data committed to a majority of replicas. Won't be rolled back.
- `linearizable` — strictly ordered with respect to concurrent writes (slow; primary only).
- `snapshot` — for multi-document transactions; consistent view across the transaction.

**Trade-off:**
- `w: 1, read local` → lowest latency, weakest guarantees.
- `w: majority, read majority` → strong guarantees, +1 network RTT to majority.
- `w: majority, read linearizable` → strongest, slowest.

**Code:**
```js
// Critical write — survive primary failure
db.payments.insertOne(
  { userId, amount, txId },
  { writeConcern: { w: 'majority', j: true, wtimeout: 5000 } }
);

// Per-request read concern
db.payments.find({ userId }).readConcern('majority');
```

**Follow-up 1: When would you ever use `w:1` over `w:'majority'`?**
For low-importance writes where the latency saving matters more than durability — telemetry, analytics events, cache warming. Even then, defaulting to majority is safer; the latency cost is usually a few milliseconds.

**Follow-up 2: What's a "rollback" in Mongo and when does it happen?**
If a primary accepts a write at `w:1`, then fails before replicating, when a new primary is elected from the surviving secondaries the old primary's un-replicated writes are rolled back when it rejoins. The data goes to a rollback file. With `w:'majority'`, this can't happen for acknowledged writes because they're already on a majority.

---

### Q7. How do MongoDB transactions work? What are the limits?

**Answer:**
Since Mongo 4.0 (replica sets) and 4.2 (sharded clusters), Mongo supports multi-document ACID transactions. The API is session-based:

```js
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    await db.collection('accounts').updateOne(
      { _id: 'a' }, { $inc: { balance: -100 } }, { session }
    );
    await db.collection('accounts').updateOne(
      { _id: 'b' }, { $inc: { balance: 100 } }, { session }
    );
  }, { readConcern: 'majority', writeConcern: { w: 'majority' } });
} finally {
  await session.endSession();
}
```

`withTransaction` handles automatic retry on transient errors (write conflicts, network blips).

**Constraints:**
- Transactions on a single replica set are fast; on sharded clusters they have higher overhead due to two-phase commit.
- Default transaction time limit: **60 seconds**. Long transactions get killed.
- Documents being touched are locked — high-throughput contention can cause `WriteConflict` errors.
- Cannot run on capped collections, system collections, or do DDL (createIndex, dropCollection) inside a transaction.

**When to use transactions:**
- Truly multi-document invariants (the classic transfer-money example).
- Anti-fraud / idempotency where you need "check-then-insert" atomicity.

**When *not* to use them:**
- Single-document updates — Mongo always gives atomic single-doc operations, no transaction needed.
- Bulk inserts that can tolerate partial failure — use `insertMany` with `ordered: false`.

**Follow-up 1: Why is Mongo's single-document atomicity often enough?**
Because the document model encourages embedding related data into one document, and updates to that document are atomic. The need for multi-document transactions in Mongo often signals a schema that's too relational — if every "transaction" spans 3 collections, you might be using documents wrong.

**Follow-up 2: What's a "write conflict" inside a transaction and how do I handle it?**
If two concurrent transactions try to write the same document, the loser sees `WriteConflict` and must retry the entire transaction. `withTransaction` does this automatically with backoff. Excessive retries mean you have hot keys — consider sharding or batching writes through a queue serialized per key.

---

### Q8. How does replication work in MongoDB?

**Answer:**
A **replica set** is a group of mongod processes that maintain the same data. One node is the **primary**, others are **secondaries**. Reads/writes go to the primary by default; secondaries asynchronously replicate via the oplog (a capped collection of operations).

**Election:** If the primary fails (heartbeat timeout), surviving members hold an election. The candidate with the most up-to-date oplog wins, becomes primary. Election typically takes 10–30 seconds.

**Read preference** — clients can route reads:
- `primary` (default) — strongest consistency.
- `primaryPreferred` — primary, fall back to secondary on failure.
- `secondary` — only secondaries; eventually consistent (may lag primary).
- `secondaryPreferred` — secondaries when available; reduces primary load.
- `nearest` — lowest network latency.

**Diagram:**
```
                ┌──────────┐
   writes ────▶ │ Primary  │ ────── oplog ──────┐
                └─────┬────┘                    │
                      │heartbeat                ▼
                ┌─────▼────┐               ┌────────┐
                │Secondary │ ◀─── stream ──│ oplog  │
                └──────────┘               └────────┘
                ┌──────────┐                    │
                │Secondary │ ◀──────────────────┘
                └──────────┘
```

**Arbiter** — a member that votes in elections but doesn't hold data. Use sparingly — they reduce write durability without contributing to storage.

**Follow-up 1: What's an "oplog" and how big should it be?**
A capped collection containing every write operation, used for replication. Size determines the **replication window** — how long a secondary can be down before it can't catch up by replaying the oplog and needs a full resync. Default sizing is 5% of free disk. For write-heavy workloads, oversize it (the resync of a 1TB DB takes hours).

**Follow-up 2: When does reading from a secondary cause problems?**
Stale reads. Secondaries lag the primary by some amount (usually milliseconds, occasionally seconds under load). An app that writes and immediately reads from a secondary can miss its own write. Solutions: read with `readConcern: 'majority'` from primary, or use causal consistency (`session.startTransaction()` mode) to guarantee reads see writes from the same session.

---

### Q9. How does sharding work in MongoDB?

**Answer:**
Sharding distributes a single logical collection across multiple replica sets ("shards"). Each shard holds a subset of the data, partitioned by a **shard key**.

**Components:**
- **`mongos` router** — query router clients connect to; doesn't store data, knows the chunk distribution.
- **Config servers** — replica set holding the chunk → shard mapping.
- **Shards** — replica sets, each owning a portion of the data.

**Shard key strategies:**
- **Hashed shard key** — `db.collection.shardCollection('db.col', { userId: 'hashed' })`. Uniform distribution, no monotonic hotspots. Bad for range queries on the key.
- **Range-based** — by a sortable field. Good for range queries; risk of "hot shard" if data inserts are monotonic (timestamp shard key → all new inserts hit the last shard).
- **Compound shard key** — combines fields to balance distribution and locality.
- **Zoned sharding** — pin specific value ranges to specific shards (e.g., EU customers to EU shard).

**Chunks** are subranges of the shard key. The balancer moves chunks between shards to keep them balanced.

**Code:**
```js
// Shard the orders collection by userId hash
sh.enableSharding('app');
sh.shardCollection('app.orders', { userId: 'hashed' });
```

**Follow-up 1: How do you choose a shard key?**
Three properties matter: **cardinality** (many distinct values; otherwise you can't split into enough chunks), **frequency** (no single value dominates writes; avoids hot chunks), and **non-monotonic** (so writes spread across shards rather than piling on the last chunk). The `userId` field is usually a great shard key for user-scoped collections.

**Follow-up 2: What's "targeted" vs "broadcast" query?**
A query that includes the shard key is **targeted** — `mongos` routes it to the exact shard(s) holding the data. A query without it is **scatter-gather (broadcast)** — sent to every shard, results merged. Broadcast queries don't scale with shard count and undermine the point of sharding. Design your queries (and indexes) so common ones include the shard key.

---

### Q10. How do Mongoose schemas, validation, and middleware work?

**Answer:**
Mongoose is an ODM (Object-Document Mapper) for Node. It adds:
1. **Schemas** — explicit shape and type validation.
2. **Validation** — declarative rules with custom validators.
3. **Middleware (hooks)** — pre/post hooks on save, find, update, etc.
4. **Virtuals** — computed properties not stored in the DB.
5. **Population** — convenient join-like fetching via `.populate()`.

**Code — your "schema validation with Joi and Mongoose middleware" stack:**
```js
const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: true,
    lowercase: true,
    trim: true,
    validate: { validator: v => /^[\w-.]+@[\w-]+\.[a-z]+$/i.test(v), message: 'invalid email' },
  },
  passwordHash: { type: String, required: true, select: false },
  role: { type: String, enum: ['admin', 'user'], default: 'user' },
  createdAt: { type: Date, default: Date.now },
});

// Index
userSchema.index({ email: 1 }, { unique: true });

// Pre-save: hash password if changed
userSchema.pre('save', async function () {
  if (this.isModified('passwordHash')) {
    this.passwordHash = await bcrypt.hash(this.passwordHash, 10);
  }
});

// Virtual: full name
userSchema.virtual('display').get(function () {
  return this.email.split('@')[0];
});

const User = mongoose.model('User', userSchema);
```

**Common middleware hooks:**
- `pre/post('save')` — on `.save()` or `Model.create()`.
- `pre/post('find')`, `('findOne')`, `('findOneAndUpdate')` — on queries.
- `pre/post('remove')` — on doc-level removes.

**Important gotcha:** Mongoose query middleware (`find`, `updateOne`) does **not** trigger document middleware (`save`). If your `pre('save')` hash-passwords, an `updateOne` that sets `passwordHash` will skip it. Cover both or use one consistently.

**Follow-up 1: When would you use Joi *and* Mongoose validation?**
Joi at the request boundary (validate the DTO before any DB code runs); Mongoose as a last-line safety net. Joi gives better error structures and is portable across non-Mongo code paths. Mongoose catches data that somehow bypassed Joi (direct DB writes, migrations, scripts).

**Follow-up 2: Why is `populate` sometimes a performance problem?**
`populate` issues a follow-up query per reference field per parent document — classic N+1 hidden behind ergonomics. For lists, prefer manually loading referenced docs in bulk by ID:
```js
const orders = await Order.find({ userId });
const productIds = orders.flatMap(o => o.items.map(i => i.productId));
const products = await Product.find({ _id: { $in: productIds } });
```
Or denormalize the few fields you need into the order document.

---

### Q11. What are the most common Mongo query performance pitfalls?

**Answer:**
1. **Collection scans (`COLLSCAN`)** — missing index. Run `.explain()` and verify `IXSCAN`.
2. **Unbounded `$regex` queries** — `/.*foo.*/` can't use indexes. Anchored prefixes (`/^foo/`) can.
3. **`$ne` / `$nin`** — generally don't use indexes effectively. Restructure to positive conditions where possible.
4. **`$or` across non-indexed fields** — each branch needs its own index.
5. **Skip-and-limit pagination** — `skip(100000)` scans 100k docs before returning. Use cursor-based pagination on a sortable indexed field instead.
6. **In-memory sorts on large result sets** — sorts that can't use an index and exceed 100 MB are killed.
7. **`$lookup` on unindexed foreign field** — the joined collection must have an index on the join key.
8. **Reading without projection** — fetching entire 50-field documents when you need 3 fields wastes bandwidth and cache.

**Code — cursor-based pagination:**
```js
// Bad: skip + limit, slow for deep pages
db.orders.find().sort({ createdAt: -1 }).skip(10000).limit(20);

// Good: cursor — pass the last seen createdAt as the next page boundary
db.orders.find({ createdAt: { $lt: lastSeenDate } })
         .sort({ createdAt: -1 })
         .limit(20);
```

**Follow-up 1: What does `.explain('executionStats')` tell me?**
The actual execution plan and counters: which index was chosen, how many documents were scanned (`totalDocsExamined`), how many were returned (`nReturned`), time spent, and rejected plans. Ideal ratio: `totalDocsExamined ≈ nReturned`. If you scan 100k docs to return 10, you have a missing or bad index.

**Follow-up 2: My query is fast on small data but slow in prod — why?**
Working set has outgrown RAM. Mongo's hot data should fit in memory; once it doesn't, every read pages from disk. Check `db.serverStatus().wiredTiger.cache` — if `bytes currently in the cache` is at the configured limit and `pages read into cache` is high, you're paging. Solutions: bigger box, better indexes (smaller working set), shard.

---

### Q12. What are time-series collections in Mongo and when are they useful?

**Answer:**
Since Mongo 5.0, **time-series collections** are a special collection type optimized for sequential time-stamped data: telemetry, metrics, sensor readings, logs.

**How they work:**
- Mongo stores documents in a columnar, time-bucketed internal format.
- Massively smaller disk footprint than regular collections for the same data (10–20x compression for typical metrics).
- Specialized indexes optimized for time-range queries.
- TTL-based deletion runs at the bucket level — fast.

**Creation:**
```js
db.createCollection('telemetry', {
  timeseries: {
    timeField: 'recordedAt',
    metaField: 'deviceId',         // groups buckets per device
    granularity: 'minutes',        // 'seconds'|'minutes'|'hours'
  },
  expireAfterSeconds: 60*60*24*90, // auto-delete after 90 days
});

// Inserts look like normal documents
db.telemetry.insertOne({
  recordedAt: new Date(),
  deviceId: 'dev-42',
  cpu: 0.72,
  memory: 0.55,
  temperature: 41.2,
});

// Time-range queries are fast
db.telemetry.find({
  deviceId: 'dev-42',
  recordedAt: { $gte: ISODate('2026-05-27T00:00:00Z') }
});
```

**For your IoT predictive maintenance project**, a time-series collection on telemetry replaces hand-rolled bucketing, gives you free compression, and supports the rolling time-window queries the ML pipeline needs.

**Follow-up 1: Why is the columnar/bucket format faster?**
Because metrics queries usually scan many documents but project only a few fields. Columnar storage colocates each field's values in contiguous memory — fewer pages, better cache locality, vectorizable. A `find({ deviceId: 'X' }).limit(1000)` on a regular collection might page in 100MB of full documents; in time-series storage, ~5MB.

**Follow-up 2: What are the limitations?**
You can't update or delete individual measurements (only drop whole windows via TTL or `deleteMany` with date filters). You can't add unique indexes. Granularity is fixed at creation. For pure append-only telemetry, that's fine; for "send corrections retroactively" workloads, time-series collections aren't a fit.

---

### Q13. What are change streams and how would you use them?

**Answer:**
A **change stream** is a real-time feed of changes (inserts, updates, deletes, replaces) on a collection, database, or deployment. Built on the oplog; works on replica sets and sharded clusters.

**Code — listening for new orders to fan out notifications:**
```js
const pipeline = [
  { $match: { 'operationType': 'insert', 'fullDocument.status': 'paid' } }
];

const stream = db.collection('orders').watch(pipeline, {
  fullDocument: 'updateLookup',
});

for await (const change of stream) {
  await emitOrderPaidEvent(change.fullDocument);
}
```

**Use cases:**
- Reactive features — send notifications, invalidate caches, update read models.
- Sync to external systems — index into Elasticsearch, replicate to a warehouse.
- Audit logs — capture every change to sensitive collections.
- Event sourcing-lite — collection acts as the source of truth, change stream as the event bus.

**Resume token** — each event has a `_id` resume token. Persist it after processing; on restart, resume from where you left off. This makes change streams effectively at-least-once with idempotent handlers.

**Follow-up 1: Why not just use the oplog directly?**
You can, but change streams are higher-level: filterable with aggregation pipelines, work uniformly on replica sets and sharded clusters, handle resume after disconnect, hide oplog quirks. Direct oplog tailing was the pre-3.6 pattern; change streams are the modern API.

**Follow-up 2: What guarantees do change streams offer?**
At-least-once delivery (assuming you persist resume tokens correctly). They survive primary failovers — `mongos` and the driver re-attach to the new primary using the resume token. Ordering is per-shard; across shards, you get a partial order. Don't assume strict global ordering without explicit reasoning.

---

### Q14. How do you handle schema migrations in MongoDB?

**Answer:**
Because Mongo doesn't enforce schema, migrations come in two flavors: **app-level (lazy)** and **batch (eager)**.

**Lazy migration:**
- Code reads documents and adapts on the fly: if a field is missing, treat as default; if shape is old, upcast.
- Writes update to the new shape.
- Eventually, all documents touched by writes are migrated; the rest can be batch-migrated later.

```js
function readUser(doc) {
  if (!doc.preferences) doc.preferences = { theme: 'light' };
  if (typeof doc.role === 'string') doc.role = [doc.role];     // upgrade to array
  return doc;
}
```

**Batch migration:**
- A script iterates documents in chunks, applies the transformation, writes them back.
- Use a `schemaVersion` field per document to track progress.
- Run during low traffic; chunk by ObjectId range to spread load.

```js
const cursor = db.users.find({ schemaVersion: { $lt: 2 } }).batchSize(1000);
for await (const doc of cursor) {
  await db.users.updateOne(
    { _id: doc._id },
    { $set: { /* new shape */, schemaVersion: 2 } }
  );
}
```

**Expand-contract** is the same pattern as Postgres (Q19 in section 03): add new fields, dual-write, backfill, switch reads, remove old.

**Follow-up 1: Why use a `schemaVersion` field?**
To make migrations resumable and idempotent. Each document carries its version; the migration script targets documents below the target version. Restart-safe — interrupted runs pick up from where they left off. Also helpful for analytics on adoption.

**Follow-up 2: When does Mongo's schemaless flexibility hurt you?**
When old code paths exist alongside new ones and produce inconsistent shapes. Six months in, a collection has documents in three shapes, the application is full of "if undefined…" branches, and validation is impossible. Mitigation: use Mongoose strict mode or Mongo's `$jsonSchema` validation rules to enforce shape at the DB level from day one.

---

### Q15. What are the most common production gotchas with MongoDB?

**Answer:**

1. **Working set bigger than RAM.** Mongo's performance falls off a cliff when frequently accessed data doesn't fit in memory. Monitor `wiredTiger.cache.bytes currently in the cache` vs `bytes configured max`.

2. **Default write concern bites you on 4.x.** Pre-5.0 default was `w:1` — silent data loss on primary failure. Always set `w: 'majority'` explicitly.

3. **No indexes on `_id` lookups in `$lookup`.** Always confirm the foreign collection has an index on the join field.

4. **Connection pool exhaustion in serverless.** Each Lambda invocation opens a new pool, slamming Mongo with connections. Use connection reuse patterns or `Mongo Atlas Data API` for true serverless.

5. **Embedded arrays that grow unbounded.** Comments on a post, items in a cart — eventually documents bloat past 16 MB and panic. Cap with `$slice` updates, or split.

6. **In-place updates on documents larger than original.** Triggers a document move, breaking sorts that rely on insertion order, fragmenting the data file. Less of an issue with WiredTiger but still suboptimal.

7. **Aggregation memory limits.** Stages have a 100 MB in-memory limit. Exceed it and the query fails unless `allowDiskUse: true` (slower). Push `$match` earlier to keep intermediate sets small.

8. **No transactions until 4.0.** Code written for older Mongo often has eventual-consistency assumptions; bringing in transactions later creates surprising performance characteristics.

9. **`updateOne` doesn't trigger Mongoose `save` hooks.** Password hashing, validation, virtuals — all bypassed.

10. **Slow secondary reads under load.** Secondaries can fall behind during heavy primary writes, since they apply ops in a single thread (or limited threads with 4.0+ parallel apply). Plan reads accordingly.

**Follow-up 1: How do you detect a working-set-too-big problem before it becomes a crisis?**
Track these MongoDB metrics: `wiredTiger.cache.pages evicted by application threads` (should be ~0; nonzero means app threads are blocking on cache eviction) and `wiredTiger.cache.tracked dirty bytes in the cache`. Also watch slow-query rates by collection — first symptom is a few queries getting slow as their data falls out of cache, then more as the eviction cascade widens.

**Follow-up 2: How would you size a Mongo cluster for your Dhee Coding Lab workload (5K users, 20+ REST APIs)?**
For that scale: single primary + 2 secondaries on a modest box (4 CPU, 16 GB RAM) is plenty. Working set is maybe 5–10 GB — fits comfortably in cache. Focus on indexes and query patterns, not capacity. You only need sharding when the working set blows past one machine's RAM (typically 200+ GB of hot data) or write throughput exceeds what one primary can take.

---

*End of section 04. Next: Redis & Caching (20 questions).*
