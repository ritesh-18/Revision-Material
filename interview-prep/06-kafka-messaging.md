# 06 — Kafka & Messaging (20 Questions)

You used Kafka for the SaaS notification fan-out (10K+ daily notifications, dead-letter queue, retry policy), MQTT for IoT ingestion (5K events/hour with replay), and BullMQ-via-Redis at Tibil. These cover Kafka internals, RabbitMQ, MQTT, comparisons, and the production gotchas at every layer.

---

### Q1. Walk me through Kafka's architecture. What are brokers, topics, partitions?

**Answer:**
Kafka is a distributed append-only log organized as **topics**, each split into one or more **partitions**, replicated across a cluster of **brokers**.

**Brokers** — Kafka server processes. A cluster has N (typically 3–9 in production). Each broker stores some partitions and serves clients.

**Topics** — logical streams of records ("orders", "notifications"). Producers append, consumers read.

**Partitions** — the unit of parallelism. A topic with 12 partitions can be consumed by up to 12 workers in parallel (one partition per worker in a consumer group). Within a partition, order is strictly preserved. Across partitions, ordering is not guaranteed.

**Replication** — each partition has a **leader** (handles all reads/writes) and N-1 **followers** (replicate from leader). If the leader broker fails, a follower is promoted. The **in-sync replica (ISR)** set is the followers caught up to the leader.

```
   Topic "notifications" (4 partitions, replication factor 3)
   ───────────────────────────────────────────────────────────
   Partition 0:    Broker 1 (Leader), Broker 2, Broker 3
   Partition 1:    Broker 2 (Leader), Broker 1, Broker 3
   Partition 2:    Broker 3 (Leader), Broker 1, Broker 2
   Partition 3:    Broker 1 (Leader), Broker 2, Broker 3
```

**Records (messages):**
- Key (optional but important — used for partition routing).
- Value (the payload, bytes).
- Headers (metadata).
- Timestamp.
- Producer-assigned offset within the partition (monotonically increasing per partition).

**Producer routing:** records with the same key go to the same partition (consistent hashing on key). No key = round-robin.

**Follow-up 1: Why does within-partition ordering matter so much?**
Because event ordering is often semantically required: "user updated profile then deleted account" must be processed in order. Kafka guarantees this only within a partition. If you partition by `user_id`, all events for one user land in one partition and stay ordered — that's the trick.

**Follow-up 2: How many partitions should a topic have?**
Enough to support your peak parallelism (you can't have more consumers in a group than partitions), but not so many that broker overhead dominates. Practical guidance: start with a multiple of your broker count, monitor consumer lag, scale up. Repartitioning is non-trivial — better to overestimate by 2–3x than to be stuck under-partitioned.

---

### Q2. How do consumer groups and offsets work?

**Answer:**
A **consumer group** is a set of consumers that collectively read a topic. Kafka assigns partitions to consumers in the group such that each partition is read by exactly one consumer. With 4 partitions and 4 consumers, each consumer reads one partition. With 2 consumers, each reads 2 partitions. With 8 consumers, 4 are idle.

**Multiple consumer groups** read the same topic independently. Each group has its own offset position. This is how you fan out the same event stream to multiple services (notifications, analytics, audit log) — each is a separate group.

**Offsets** — Kafka tracks how far each consumer group has read in each partition. Stored in the `__consumer_offsets` internal topic. Consumers commit offsets back to Kafka after processing.

**Commit strategies:**
- **Auto-commit (default in some clients)** — commit every N ms automatically. Risk: commit may happen *before* you finish processing → message loss on crash. **Avoid for important workloads.**
- **Manual commit after processing** — `consumer.commit()` after the message is done. At-least-once semantics: guaranteed delivery, possible duplicate processing on crash.
- **Manual commit per batch** — commit after every N messages.

**Code (kafkajs in Node):**
```js
const consumer = kafka.consumer({ groupId: 'notifications-service' });
await consumer.subscribe({ topic: 'orders', fromBeginning: false });

await consumer.run({
  autoCommit: false,                          // we control commits
  eachMessage: async ({ topic, partition, message }) => {
    try {
      await processOrder(JSON.parse(message.value.toString()));
      // Commit after success
      await consumer.commitOffsets([{
        topic, partition, offset: (Number(message.offset) + 1).toString(),
      }]);
    } catch (err) {
      logger.error({ err, offset: message.offset }, 'failed; will retry');
      // Don't commit — message will be redelivered after restart
    }
  },
});
```

**Follow-up 1: What happens when a consumer joins or leaves the group?**
A **rebalance** — Kafka pauses the group, redistributes partitions among the new set of consumers, and resumes. During rebalance, no messages are processed. Frequent rebalances kill throughput. We'll cover causes/mitigations in Q7.

**Follow-up 2: What's the difference between "at-least-once" and "exactly-once" delivery in this model?**
At-least-once is the natural Kafka semantic: messages may be delivered multiple times if a consumer crashes after processing but before committing. Exactly-once requires either (a) idempotent processing (the consumer's effect is the same whether or not the message is processed twice), or (b) Kafka's transactional API (Q6) plus careful coordination with the downstream system.

---

### Q3. Why partitions, and how do you choose partition count?

**Answer:**
Partitions exist for **parallelism** and **ordering guarantees**.

- More partitions = more parallelism = higher throughput.
- Within a partition, order is preserved.
- Across partitions, order is best-effort (no global ordering guarantee).

**Sizing partitions:**
- **Target throughput / per-partition throughput** = required partitions. Per-partition throughput depends on consumer processing speed and producer write rate. A consumer processing 1K msg/sec, target 10K/sec → at least 10 partitions.
- **Future-proofing** — partitions can be added but not removed; adding partitions breaks key-based ordering for existing keys (because the hash mod changes). Provision generously upfront.
- **Broker capacity** — too many partitions per broker increases controller overhead, leader-election time, file descriptors. Rule of thumb: ≤ 2000–4000 partitions per broker.
- **Consumer count cap** — you can never have more useful consumers in a group than partitions. Pick partitions ≥ max consumers you'd ever want.

**Practical starting point:** 12–24 partitions per topic on a 3-broker cluster. Adjust based on observed lag and CPU.

**Code — checking partition count:**
```bash
kafka-topics.sh --bootstrap-server kafka:9092 --describe --topic orders
```

**Follow-up 1: What's the cost of having too many partitions?**
Per-partition overhead on brokers (file handles, memory for indexes, replication threads). Each partition is a directory with multiple files; tens of thousands of partitions across a cluster strain the OS. Also, leader election after broker failure takes longer with more partitions. Big production deployments cap at ~200K partitions total cluster-wide.

**Follow-up 2: Can I increase partitions on a live topic? What breaks?**
Yes (`kafka-topics --alter --partitions N`), but key-based ordering breaks: a key that used to land on partition 3 may now land on partition 7. Messages already in partition 3 stay there; new messages for that key go to 7. If your consumers rely on per-key ordering, this causes brief disorder. Plan partition counts upfront when possible.

---

### Q4. What do producer `acks` settings mean?

**Answer:**
`acks` controls how many replicas must acknowledge a write before the producer considers it successful.

- **`acks=0`** — producer doesn't wait. Highest throughput, lowest durability. Message can be lost in flight. Used for fire-and-forget telemetry where loss is acceptable.
- **`acks=1`** — wait for leader to write. If the leader fails before replicating, the write is lost. Default for older Kafka; not safe for important data.
- **`acks=all` (or `-1`)** — wait for all in-sync replicas. Strongest durability. The leader doesn't acknowledge until min.insync.replicas have written. **The right default for important data.**

`acks=all` works with the `min.insync.replicas` topic config — if fewer than that many replicas are in sync, the producer gets an error rather than silently weakening durability.

**Code (kafkajs):**
```js
const producer = kafka.producer({
  allowAutoTopicCreation: false,
  idempotent: true,                 // also forces acks=all (next question)
});
await producer.send({
  topic: 'orders',
  messages: [{ key: orderId, value: JSON.stringify(order) }],
  acks: -1,                          // all (default with idempotent)
});
```

**Throughput trade-off:**
- `acks=0`: very fast, unsafe.
- `acks=1`: fast, partial safety.
- `acks=all`: slower (RTT × ISR), strong safety. Mitigated by batching.

**Follow-up 1: What happens if `min.insync.replicas` isn't met with `acks=all`?**
The producer gets `NOT_ENOUGH_REPLICAS` or similar. The write isn't attempted. This is by design — better to fail loudly than silently lose durability. Typical setting: `min.insync.replicas = 2` for `replication.factor = 3`, meaning you can lose one broker and still write.

**Follow-up 2: How does compression interact with acks?**
Compression (`gzip`, `snappy`, `lz4`, `zstd`) shrinks batches sent over the wire. With `acks=all`, smaller batches mean lower RTT to confirm. Most teams use `lz4` (good ratio, fast) or `zstd` (better ratio, slightly slower). Big wins for textual payloads (JSON), minimal for already-compressed (Avro/Protobuf).

---

### Q5. What's an idempotent producer?

**Answer:**
Without idempotence, retrying a producer send on a transient failure (timeout, broker hiccup) can produce duplicates: the broker received the original, the producer didn't get the ack, retry succeeds, two copies stored.

**Idempotent producer** (Kafka 0.11+, `enable.idempotence=true`):
- Producer assigns each batch a **sequence number**.
- Broker tracks the highest seen sequence per (producer ID, partition).
- A retry with the same sequence is detected and ignored — no duplicate.
- Free; no application changes; should be default.

Requires:
- `acks = all` (idempotence implies this).
- `retries > 0`.
- `max.in.flight.requests.per.connection ≤ 5`.

**Code (kafkajs):**
```js
const producer = kafka.producer({
  idempotent: true,                  // turns on producer-side dedup
  maxInFlightRequests: 5,
});
```

**Limits:**
- **Per producer instance** — restart producer = new producer ID = sequence numbers reset; can produce duplicates across restarts.
- **Per partition** — duplicates across partitions are still possible (e.g., from re-keying).
- **Within a single session.**

For cross-session exactly-once, you need **transactional producers** (Q6).

**Follow-up 1: How does the broker know which sequences came from the same producer after a network blip?**
The producer gets a `producer.id` (PID) from the broker on startup, plus a monotonic epoch. The broker stores `(PID, epoch, sequence)` per partition. A retry with the same `(PID, sequence)` is silently de-duplicated. A new producer instance gets a new PID; the broker treats it as a fresh entity.

**Follow-up 2: Are duplicates from the **consumer** side covered by idempotent producers?**
No. Idempotence is producer-side. A consumer that crashes after processing but before committing still sees the message redelivered — that's a consumer-side duplicate. Solve it with idempotent consumer logic (using message key or a transaction-id field in your own DB to detect repeats).

---

### Q6. Explain Kafka's transactional API and exactly-once semantics.

**Answer:**
Kafka transactions (KIP-98, Kafka 0.11+) let a producer write to multiple partitions/topics atomically, and tie that to its consumer offset commits. The combination enables **read-process-write** loops with exactly-once semantics.

**The classic exactly-once pattern:**
1. Consume from input topic.
2. Process the message; produce derived events to output topic(s).
3. Commit consumer offset and produced messages **atomically**.

Either all of (output messages + offset commit) succeed, or none. No duplicates downstream even on crash.

**Code (kafkajs):**
```js
const producer = kafka.producer({
  idempotent: true,
  transactionalId: 'notifications-processor-v1',  // stable, unique per processor
});
await producer.connect();

// In the consumer loop, for each batch:
const tx = await producer.transaction();
try {
  await tx.send({ topic: 'emails', messages: emailMessages });
  await tx.send({ topic: 'audit', messages: auditMessages });
  await tx.sendOffsets({
    consumerGroupId: 'notifications-processor',
    topics: [{ topic: 'orders', partitions: [{ partition, offset }] }],
  });
  await tx.commit();
} catch (err) {
  await tx.abort();
  throw err;
}
```

**Consumer side:** to see only committed messages, set `isolation.level=read_committed`. Otherwise consumers see uncommitted (aborted) messages too.

**Limits:**
- Exactly-once is **within Kafka only**. If you write to a database in the middle of your processing, you need a different mechanism (outbox pattern, transactional outbox to Kafka).
- Throughput drops 10–30% vs non-transactional.
- `transactionalId` must be stable across restarts to fence off zombie producers from a previous instance.

**Follow-up 1: What's the outbox pattern and when is it needed?**
When you need to update a database AND publish to Kafka atomically. The trick: in one DB transaction, you update the business state and insert into an `outbox` table. A separate process reads the outbox and publishes to Kafka, then marks rows as sent. The DB transaction is the atomic boundary; Kafka's at-least-once becomes "publish each outbox row once or more, ignore duplicates downstream." This handles the dual-write problem cleanly.

**Follow-up 2: Why is `transactionalId` stability so important?**
The broker uses it to fence "zombie" producers — old instances that the system thinks are dead but are still alive. When a new producer starts with the same `transactionalId`, the broker bumps the epoch; the old producer's writes are rejected. If you randomize `transactionalId` per startup, you lose this fencing — duplicate processing on race conditions becomes possible.

---

### Q7. Consumer rebalancing — what triggers it and how do you minimize disruption?

**Answer:**
A **rebalance** is when Kafka recalculates partition assignments within a consumer group. During rebalance, all consumers stop processing — throughput drops to zero.

**Triggers:**
- A consumer joins the group (deploy, scale-up).
- A consumer leaves the group (shutdown, crash).
- A consumer fails to heartbeat within `session.timeout.ms`.
- Partition count changes.

**Old protocol (eager):** Every rebalance revokes ALL partitions from all consumers, then reassigns. Very disruptive.

**New protocol (cooperative / incremental rebalance):** Only partitions that need to move are revoked. Other consumers keep processing. Available in modern Kafka clients; opt in via `partition.assignment.strategy = CooperativeStickyAssignor`.

**Code — cooperative rebalance in kafkajs:**
```js
const consumer = kafka.consumer({
  groupId: 'orders-processor',
  sessionTimeout: 30000,
  heartbeatInterval: 3000,
  partitionAssigners: [PartitionAssigners.cooperativeSticky],
});
```

**Reducing rebalance frequency:**
- Tune `session.timeout.ms` and `heartbeat.interval.ms` so brief blocking doesn't trigger rebalance.
- Don't do long synchronous work in your message handler — use `max.poll.interval.ms` correctly. If processing one message takes > 5 min, the consumer is considered dead and rebalance triggers.
- Sticky/cooperative assignors reduce churn — partitions tend to stay with the same consumer across rebalances.

**Follow-up 1: My consumer is being kicked out — why?**
Likely `max.poll.interval.ms` exceeded. The default is 5 minutes. Your message handler must finish (or call `poll` again) within that. For long-running work, either bump the interval or move the work to a background queue and acknowledge fast. Use the heartbeat thread (kafkajs handles automatically) to keep the session alive.

**Follow-up 2: Can I do "static membership" to avoid rebalances on rolling restarts?**
Yes (Kafka 2.3+). Set `group.instance.id` to a stable ID per consumer. On restart, the consumer keeps its assignment if it returns within `session.timeout.ms`. Rolling restarts of a 10-consumer group cause near-zero rebalance activity. Very useful in Kubernetes where pods restart routinely.

---

### Q8. How does replication work? What are leaders, ISR, and unclean elections?

**Answer:**
Each partition has a configured **replication factor** (e.g., 3). One replica is the **leader** — handles all reads and writes for that partition. The others are **followers** that replicate the leader's log.

**ISR (In-Sync Replicas)** — the set of replicas that are currently caught up to the leader within `replica.lag.time.max.ms` (default 30s). The leader is always in the ISR. Followers that fall behind are removed from the ISR until they catch up.

**Failure handling:**
- Leader fails → controller elects a new leader from the ISR.
- Follower fails → drops from ISR, but cluster continues serving.
- If `min.insync.replicas` isn't met, producers with `acks=all` get errors.

**Unclean leader election (`unclean.leader.election.enable`):**
- If `true`: when all ISR replicas are unavailable, an out-of-sync replica can be elected leader. **Data loss** — messages that were only on the failed ISR are gone.
- If `false` (recommended): the partition becomes unavailable until an ISR member returns. **Availability cost, durability preserved.**

**For most production use cases:** `replication.factor=3, min.insync.replicas=2, unclean.leader.election.enable=false`. Tolerates one broker loss without availability or data loss.

**Follow-up 1: Why might a follower fall out of ISR?**
Slow network, slow disk, GC pause, or producer surge that the follower can't keep up with. The follower lags by more than `replica.lag.time.max.ms` and drops from ISR. Monitor `UnderReplicatedPartitions` JMX metric — non-zero means followers are lagging.

**Follow-up 2: What's the impact of `unclean.leader.election.enable=true`?**
Increased availability — partition stays serving even if all ISR replicas are down. Cost: any messages that were on the ISR but not on the elected (out-of-sync) replica are lost. For analytics streams you might accept this; for financial transactions, never.

---

### Q9. What's the difference between retention and compaction?

**Answer:**
Two ways Kafka manages the size of topics.

**Retention (`cleanup.policy=delete`)** — the default. Delete messages older than `retention.ms` (time-based) or once the partition exceeds `retention.bytes` (size-based), whichever hits first. Used for event streams where old data isn't needed: logs, click streams, audit trails.

**Compaction (`cleanup.policy=compact`)** — keep only the **latest message per key** in a partition. Old values for the same key are removed during background compaction. Used for "state snapshot" topics where you want the latest state of each entity:
- Configuration changes keyed by config ID.
- User profile updates keyed by user ID.
- Materialized views keyed by entity ID.

Both can be combined (`cleanup.policy=compact,delete`) for time-bound state — e.g., "latest state per key, but drop anything older than 90 days."

**Tombstones** — to delete a key from a compacted topic, produce a message with that key and `null` value. Compaction picks up the tombstone and eventually removes both the tombstone and prior values.

**Code:**
```bash
kafka-configs.sh --bootstrap-server kafka:9092 --alter \
  --entity-type topics --entity-name user-state \
  --add-config cleanup.policy=compact,segment.ms=86400000
```

**Follow-up 1: What's `__consumer_offsets` and how is it stored?**
The internal topic Kafka uses to track consumer offsets. It's compacted — only the latest offset per (group, topic, partition) is retained. Tombstones remove offsets when a consumer group is deleted. This is also how transactional state is stored (`__transaction_state` internal topic).

**Follow-up 2: When would I use a compacted topic vs storing state in a database?**
For event-sourced read models or change-data-capture sinks where you want Kafka to be the source of truth for entity state. Bootstrapping a new consumer = read the full compacted topic and build a local view. Compaction means even ancient state is available without infinite retention. For OLTP state with complex queries, use a database; for "fan-out current state to many subscribers," compacted topics shine.

---

### Q10. What is a Schema Registry and why do you need one?

**Answer:**
Kafka doesn't enforce schemas — payloads are just bytes. Without a contract, producers and consumers drift: producer adds a field, consumer parses old schema, error. A **Schema Registry** (Confluent's, Karapace) stores schemas (Avro, Protobuf, JSON Schema) versioned per topic.

**Workflow:**
1. Producer serializes message with a schema; sends `[magic-byte, schema-id, payload]`.
2. Registry assigns the schema an ID; producer caches it.
3. Consumer reads the message, sees the schema ID, fetches schema from registry (cached), deserializes.

**Compatibility modes:**
- **BACKWARD** (default) — new schemas can read old data. Lets you upgrade consumers first.
- **FORWARD** — old consumers can read data written with new schemas. Lets you upgrade producers first.
- **FULL** — both.
- **NONE** — anything goes (don't).

**Avro vs Protobuf vs JSON Schema:**
- **Avro** — compact binary, schema travels separately, dynamic deserialization. Most common in Kafka ecosystem.
- **Protobuf** — even more compact, typed generated code, multi-language. Popular when gRPC is also in the stack.
- **JSON Schema** — human-readable, larger payloads, simpler ops. Fine for low-volume topics.

**Code — Avro with confluent-kafka:**
```js
const schemaRegistry = new SchemaRegistry({ host: 'http://registry:8081' });
const id = await schemaRegistry.register({ type: SchemaType.AVRO, schema: orderSchema });

await producer.send({
  topic: 'orders',
  messages: [{ key: orderId, value: await schemaRegistry.encode(id, order) }],
});

// Consumer
const decoded = await schemaRegistry.decode(message.value);
```

**Follow-up 1: Why care about backward compatibility?**
Because services deploy independently. If a producer adds a required field, every existing consumer breaks. Backward compatibility (Avro: default values for new fields, deprecate but don't remove old fields) lets producers move first without coordination.

**Follow-up 2: What's the difference between JSON-as-message and JSON Schema with a registry?**
Plain JSON has no contract — every consumer must trust the producer to keep shape. JSON Schema registers the contract, validates on produce/consume, and enforces compatibility rules. The wire payload is the same JSON; the registry adds discipline.

---

### Q11. How do you handle poison messages and dead-letter queues?

**Answer:**
A **poison message** is one that consistently fails processing — usually a malformed payload, a hard business-rule violation, or a downstream dependency that's permanently broken for that record. Without handling, the consumer retries forever, blocking progress.

**Dead-letter queue (DLQ):** a separate topic where bad messages go after N failed attempts. A human (or a separate process) reviews/reprocesses them.

**Pattern (your SaaS notification pipeline):**
1. Consumer pulls message from `notifications`.
2. Try to send notification.
3. On failure: increment a retry counter (stored in DB or message header).
4. If counter < N: republish with delay (back to topic) or to a dedicated retry topic with longer poll interval.
5. If counter ≥ N: publish to `notifications.dlq` with the original payload + failure context.
6. ack the original message and move on.

**Code:**
```js
async function handle(message) {
  try {
    await sendNotification(message);
    return;
  } catch (err) {
    const attempts = parseInt(message.headers['attempts']?.toString() ?? '0') + 1;
    if (attempts < 5) {
      // Republish with incremented attempt count and longer delay
      await producer.send({
        topic: 'notifications.retry',
        messages: [{
          key: message.key,
          value: message.value,
          headers: { ...message.headers, attempts: attempts.toString() },
        }],
      });
    } else {
      // DLQ
      await producer.send({
        topic: 'notifications.dlq',
        messages: [{
          key: message.key,
          value: message.value,
          headers: { error: err.message, lastAttempt: new Date().toISOString() },
        }],
      });
    }
  }
}
```

**Follow-up 1: How do you distinguish transient failures from poison messages?**
By the nature of the error. Transient: timeout, 503 from downstream, network error. Permanent: 400 (bad payload), 404 (target doesn't exist), parse error. Retry transient with backoff; DLQ permanent immediately. The retry counter is your last-resort filter — if everything else lets it through, repeated failures eventually DLQ.

**Follow-up 2: How do you reprocess from a DLQ?**
Two patterns: (1) manual — a tool that lets ops inspect DLQ messages, fix the underlying issue (e.g., add the missing tenant in the DB), and re-publish to the original topic. (2) Automated — a periodic process that retries DLQ messages after a config-driven cool-off period, in case the failure was a long-but-transient outage.

---

### Q12. Kafka vs RabbitMQ — when do you choose each?

**Answer:**

**Kafka:**
- Distributed log; messages stay around for retention period.
- High throughput (100k+ msg/sec per broker).
- Strong ordering within partition.
- Consumer groups for parallel processing.
- Replay from any offset.
- Best for: event streaming, analytics pipelines, audit logs, source-of-truth event store.

**RabbitMQ:**
- Traditional message broker; messages consumed and gone (or routed via exchanges).
- Moderate throughput (10k–50k msg/sec).
- Rich routing: direct, topic, fanout, header exchanges.
- Per-message acknowledgment, requeue, priority queues.
- Easier mental model for "task queue."
- Best for: work distribution, RPC patterns, complex routing, classic message-broker workloads.

**Key differences:**
| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Consumed messages persist? | Yes (until retention) | No (deleted after ack by default) |
| Throughput | Very high | High |
| Routing | Topic/partition only | Rich exchange-based |
| Ordering | Per partition | Per queue |
| Replay | Yes (rewind offsets) | No (gone after ack) |
| Message size | Small (<1MB recommended) | Larger OK |
| Setup complexity | Higher (Zookeeper/KRaft, JVM tuning) | Lower |

**Choosing:**
- "Event stream consumed by multiple services" → Kafka.
- "Task queue where work is distributed and forgotten" → RabbitMQ (or BullMQ).
- "Both" → both, in different roles.

**For your SaaS notification fan-out at 10K/day**, either would work. You chose Kafka — gives you future-proof replay and the ability to add analytics/audit consumers without disturbing the email pipeline.

**Follow-up 1: Why is "message persists after consumption" a big deal?**
Because new consumers can read history. Add an analytics service six months later — it reads all events from the beginning. Find a bug in your processor — fix it, reset offsets, re-process. With RabbitMQ, consumed-and-gone means you need a separate archive (S3, DB) for replay.

**Follow-up 2: When does RabbitMQ's per-message ack beat Kafka's offset-commit?**
When messages within a "queue" are independent and may finish in any order. RabbitMQ acks one message at a time — slow message N doesn't block fast message N+1. Kafka's offset is monotonic per partition — to commit message 100, you must finish 0–99. For "task queue" workloads, RabbitMQ's model fits better.

---

### Q13. Kafka vs Redis Streams — when does Streams win?

**Answer:**

**Redis Streams (since 5.0):**
- Append-only log inside Redis.
- Consumer groups with similar semantics to Kafka.
- Single-server (Streams don't span Cluster nodes naturally; one stream lives on one node).
- Backed by Redis memory + persistence.

**Streams wins:**
- Your team already uses Redis. Adding a Kafka cluster is huge operational overhead; Streams is one new command.
- Low-to-medium throughput (≤ 10K msg/sec on a single Redis).
- Short retention (hours/days). Memory cost grows with retention.
- Latency-sensitive small payloads — Redis sub-millisecond is hard to beat.

**Kafka wins:**
- Very high throughput (100k+ msg/sec).
- Long retention (weeks/months/forever-with-compaction).
- Multi-region replication, cross-DC.
- Existing Kafka tooling ecosystem (Connect, KSQL, Schema Registry).
- Need to fan out to many independent consumer applications.

**Practical heuristic:** Streams for "internal eventing inside one product," Kafka for "company-wide event backbone."

**Follow-up 1: How does throughput compare?**
Single Redis with Streams can sustain ~30–50K msg/sec for small messages on modern hardware. Single Kafka broker easily handles 100K+ msg/sec, and you scale by partitions across brokers. For sub-100K workloads, Streams is fine; beyond, Kafka.

**Follow-up 2: What about durability differences?**
Kafka's replication and `acks=all` give multi-broker durability; data is on multiple disks across machines. Redis Streams persist via AOF (with the fsync caveats) and replicate to replicas, but replication is async and replicas don't write to disk before acknowledging by default. Kafka is the safer choice for durability-critical streams.

---

### Q14. Explain MQTT's QoS levels.

**Answer:**
MQTT is the pub/sub protocol you used for IoT ingestion (sensors → broker → backend). It has three **Quality of Service (QoS)** levels, each with different delivery guarantees:

**QoS 0 — "at most once":**
- Fire and forget. Publisher sends; no acknowledgment.
- Fastest, but messages can be lost on network blips or broker restarts before persistence.
- Used for high-volume, low-stakes telemetry where loss is acceptable.

**QoS 1 — "at least once":**
- Publisher sends; broker acks (PUBACK). Publisher retries if no ack.
- Duplicates possible if the ack is lost but the broker did receive.
- The most common choice — durable enough, simple semantics.

**QoS 2 — "exactly once":**
- Four-message handshake (PUBLISH → PUBREC → PUBREL → PUBCOMP).
- Guarantees no duplicates and no loss.
- Slower (more round-trips) and more state-heavy on broker.
- Used for high-stakes single events (commands, financial messages).

```
QoS 0:  Pub ──msg──▶ Sub                      (no ack)
QoS 1:  Pub ──msg──▶ Sub ──PUBACK──▶ Pub       (retry until ack)
QoS 2:  Pub ──msg──▶ Sub ──PUBREC──▶ Pub
        Pub ──PUBREL──▶ Sub ──PUBCOMP──▶ Pub   (two-phase commit)
```

**For your IoT pipeline at 5K events/hour with guaranteed delivery + replay:** QoS 1 with broker-side persistence (Mosquitto/EMQX configured to write to disk). Replay support means the queue retains messages until acknowledged by the downstream consumer (your Node ingester).

**Follow-up 1: When is QoS 0 actually OK?**
For metrics with high redundancy — if you publish CPU usage every 10s, missing one sample is fine; the next one is right behind. Also for "hot path" telemetry where end-to-end latency matters more than completeness. Don't use it for state changes ("device went offline") or commands.

**Follow-up 2: Retained messages and Last Will — what are these?**
**Retained**: a flag on a publish that tells the broker to remember this message; any new subscriber gets it immediately on subscribe. Good for "last known state" semantics. **Last Will**: a message the broker publishes on the client's behalf if the client disconnects ungracefully — used for "device went offline" detection.

---

### Q15. How does RabbitMQ's exchange/queue/binding model work?

**Answer:**
RabbitMQ separates **producers**, **exchanges**, **queues**, and **consumers**.

- **Producer** publishes to an **exchange**, not directly to a queue.
- An **exchange** routes messages to **queues** based on **bindings**.
- **Consumers** read from queues.

**Exchange types:**

- **Direct** — routes to queues whose binding key exactly matches the message's routing key. Used for simple "send to specific worker pool" patterns.
- **Topic** — wildcards in binding keys. Routing key `order.created.us` matches binding `order.*.us` or `order.#`. Used for hierarchical routing.
- **Fanout** — broadcasts to every bound queue. Used for pub/sub.
- **Headers** — routes based on message headers (no routing key). Rare.

**Code:**
```js
// Setup
await channel.assertExchange('orders', 'topic', { durable: true });
await channel.assertQueue('email-orders', { durable: true });
await channel.bindQueue('email-orders', 'orders', 'order.created.*');

// Publish
channel.publish('orders', 'order.created.us', Buffer.from(JSON.stringify(order)), {
  persistent: true,                    // store on disk
  contentType: 'application/json',
});

// Consume
await channel.consume('email-orders', async msg => {
  await sendEmail(JSON.parse(msg.content.toString()));
  channel.ack(msg);
}, { noAck: false });
```

**Key features:**
- Per-message acknowledgment (ack/nack/requeue).
- Priority queues.
- TTL per queue or per message.
- Dead-letter exchanges.
- Quorum queues (Raft-based replication, RabbitMQ 3.8+) for HA.

**Follow-up 1: What's a "quorum queue" and why prefer it?**
Old "classic mirrored queues" had data-loss edge cases. Quorum queues use Raft consensus — every write is acknowledged by a majority before client gets confirm. Safer for important data. Slightly higher write latency. Default choice for new RabbitMQ deployments.

**Follow-up 2: How does RabbitMQ handle the "slow consumer" problem?**
Prefetch count (`channel.prefetch(N)`) limits how many unacked messages a consumer can hold. Without prefetch, RabbitMQ pushes all queued messages to whoever subscribes first — fast consumer eats them all, slow consumer gets nothing. Set prefetch to a small N (1–10 for slow handlers, 100+ for fast) for fair distribution.

---

### Q16. How do you tune Kafka producer throughput?

**Answer:**
Throughput is mostly about **batching** and **compression**.

**Batching:**
- `batch.size` — max bytes per batch (default 16KB, often raise to 64–256KB).
- `linger.ms` — how long to wait for more messages before sending (default 0). Raise to 5–100ms to let batches fill up.
- The producer holds messages until either `batch.size` or `linger.ms` is hit. Bigger batches = fewer round-trips = higher throughput.

**Compression:**
- `compression.type=zstd` (or `lz4`, `gzip`, `snappy`).
- Compresses entire batches — bigger batches compress better.
- Significantly reduces network bytes; zstd typically gives 3–10x compression on JSON.

**Other settings:**
- `acks=all` is required for durability; with idempotence on, this is automatic.
- `max.in.flight.requests.per.connection ≤ 5` (with idempotence) keeps batches in pipeline without breaking ordering.
- `buffer.memory` — total bytes the producer can buffer; raise for high-throughput producers.

**Code:**
```js
const producer = kafka.producer({
  idempotent: true,
  compression: CompressionTypes.ZSTD,
  // kafkajs internals; these are tunable
});

// When sending
await producer.send({
  topic: 'orders',
  messages: bigBatchOfMessages,           // batch yourself for big wins
  compression: CompressionTypes.ZSTD,
  timeout: 10000,
});
```

**Consumer-side throughput:**
- `fetch.min.bytes` — wait until N bytes available before returning.
- `fetch.max.bytes` — max per fetch.
- `max.poll.records` — limit batch size to your processing budget.

**Follow-up 1: My producer is fast in benchmarks but slow in prod — why?**
Common culprits: (1) too many small partitions — overhead per partition adds up; (2) low `linger.ms` — every send is a tiny batch; (3) network bandwidth — check ingress to brokers; (4) replication lag on followers — high write rate causes followers to fall behind, causing producer backpressure with `acks=all`.

**Follow-up 2: Should producers be async or sync?**
Async always. `producer.send()` is async by default — returns a promise. Use `await Promise.all(messages.map(m => producer.send(m)))` for batching, or rely on the internal batcher with `linger.ms`. Synchronous awaiting per-message kills throughput.

---

### Q17. Consumer lag — how do you measure and address it?

**Answer:**
**Consumer lag** = (latest offset in partition) − (last committed consumer offset). Tells you how far behind a consumer group is.

**Measurement:**
```bash
kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group notifications-processor
```
Output shows per-partition lag. Modern monitoring (Burrow, Cruise Control, Datadog Kafka integration) graphs this in real time.

**Causes of lag growth:**
1. Producer rate > consumer rate.
2. Consumer is processing slowly (downstream API latency, GC pauses).
3. Consumer count < partition count (some partitions idle).
4. Rebalances (during a rebalance, no consumption happens).
5. Errors causing retries within consumer.

**Addressing lag:**
- **Add consumers** (up to partition count) for parallel processing.
- **Add partitions** if you've hit the consumer-count ceiling. (Caveat: breaks key ordering for existing keys.)
- **Make handler faster** — async I/O, batching downstream calls.
- **Move slow work elsewhere** — consumer hands work off to a background queue and acks immediately. Trade-off: weaker delivery semantics for the secondary queue.
- **Increase `max.poll.records` and process in batches.**

**Code — batched consumer:**
```js
await consumer.run({
  eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
    // Process all messages in batch in parallel
    await Promise.all(batch.messages.map(m => handle(m)));
    // Mark all as processed
    for (const m of batch.messages) resolveOffset(m.offset);
    await heartbeat();
  },
});
```

**Follow-up 1: When is some lag actually OK?**
Always, in steady state. Real-time systems usually run with a few hundred ms of lag — that's the producer-to-consumer pipeline filling. What matters is whether lag grows unbounded. Alarm on rate-of-change, not on absolute number: "lag growing > X messages per minute for > 5 minutes" is the signal.

**Follow-up 2: What's the difference between lag in messages vs lag in time?**
Lag-messages tells you offset distance; lag-time tells you wall-clock delay (using message timestamps). Lag-time is what users feel — "events are 5 minutes stale." Lag-messages can mislead: 10K messages of lag is great if you process 1M/sec, terrible if 100/sec.

---

### Q18. Briefly explain Kafka Connect and Streams.

**Answer:**

**Kafka Connect:**
- Framework for moving data between Kafka and other systems without writing code.
- Two kinds: **Source connectors** (DB → Kafka) and **Sink connectors** (Kafka → DB/S3/etc.).
- Examples: Debezium (CDC from Postgres/MySQL/Mongo to Kafka), JDBC sink, S3 sink, Elasticsearch sink.
- Distributed mode: runs as a fleet of workers, auto-balances connector tasks.

**Use case for your stack:** Debezium connector tailing the Postgres WAL emits row-level change events to Kafka topics. Downstream services consume those events for cache invalidation, search indexing, analytics — without you writing any code.

**Kafka Streams:**
- A Java library for stream-processing applications on top of Kafka.
- Stateful operations (joins, aggregations, windows) with state stored in RocksDB locally + change-logged to Kafka for fault tolerance.
- Example: real-time aggregation, joining two streams, materializing a view.
- Node ecosystem alternative: ksqlDB (SQL on streams), or stream-processing in your own consumer with manual state.

**Follow-up 1: How does Debezium handle the dual-write problem?**
It doesn't write to your DB — it *reads* the WAL/binlog/oplog. Your app updates the DB normally; Debezium tails the DB's replication log and emits each row change to Kafka. Since the DB's transactional commit is the atomic point, there's no dual-write — the event is guaranteed once the row is committed.

**Follow-up 2: When would I avoid Kafka Streams?**
When your stack is Node/Python and you don't want JVM ops. Kafka Streams is Java-only. Node alternatives: `kafkajs` consumers with manual state, ksqlDB (separate service), or a stream-processing framework like Apache Flink (more powerful but heavier).

---

### Q19. Choosing the right message system — give me the decision tree.

**Answer:**

```
START
  │
  ├─ Need durability and replay? ──── No ──▶ Pub/Sub (Redis Pub/Sub, gRPC streaming, EventBridge)
  │              │
  │              Yes
  │              │
  ├─ Throughput  ─── < 5K msg/sec ──▶ Redis Streams or RabbitMQ or BullMQ (Node)
  │              │
  │              5K - 100K msg/sec
  │              │
  ├─ Multiple consumers want the same events? ── Yes ──▶ Kafka
  │              │
  │              No (work queue)
  │              │
  ├─ Need rich routing (priorities, headers, fanout, etc)? ── Yes ──▶ RabbitMQ
  │              │                                            No
  │              │                                            │
  │              │                                          BullMQ / SQS / RabbitMQ
  │              │
  └─ > 100K msg/sec or cross-region replication ──▶ Kafka (with proper sizing)
```

**Real-world choices for your projects:**
- **IoT ingestion** (5K events/hour, durability, replay) → **MQTT** for the device side, then bridged to a durable store.
- **SaaS notification fan-out** (10K/day, multiple channels, DLQ, retry) → **Kafka** because it gives you replay + multi-consumer + audit story for free.
- **Tibil's BullMQ jobs** (background work, retries, scheduled jobs) → **Redis BullMQ** — perfect fit for work queue without standing up Kafka.

**Follow-up 1: Can I switch from RabbitMQ to Kafka later?**
Yes, but it's not free. The semantics differ (per-message ack vs offset commit, no replay vs replay, routing complexity). Plan for an abstraction layer in your producer/consumer code so the swap doesn't ripple. Most teams don't switch — they pick once and live with it.

**Follow-up 2: What about SQS or Google Pub/Sub?**
Managed alternatives. **SQS**: simple message queue, infinite scale, at-least-once, no ordering by default (FIFO queues add ordering). Great if you're already on AWS and don't need Kafka's features. **Google Pub/Sub**: pub/sub with persistence; similar story for GCP. Both trade flexibility for operational simplicity — usually the right call if managed fits.

---

### Q20. What are the most common Kafka/messaging production gotchas?

**Answer:**

1. **No idempotent producer.** Retries silently duplicate. Always `enable.idempotence=true`.

2. **Auto-commit consumer offsets.** Default in some clients; commits before processing. Crash = data loss. Manual commit after processing.

3. **Single partition for ordered topics, scaled too late.** "We picked 1 partition for simple ordering, now we can't parallelize." Partition by a meaningful key (user_id, tenant_id) from day one.

4. **No `min.insync.replicas`.** A single-broker ISR survives one broker loss but a second loss either drops data or stops writes. Set explicitly.

5. **Massive messages.** Kafka has a default 1 MB limit. Producer-side and broker-side both need raising. But really — don't send huge payloads through Kafka. Put them in S3, send a reference.

6. **Consumer doing slow work in `eachMessage`.** Trips `max.poll.interval.ms`, triggers rebalance. Either make it fast or async-offload.

7. **No DLQ.** Poison messages block the partition. Implement DLQ from day one even if it's empty for years.

8. **Producer using `acks=1` (or 0) for important data.** Silent loss on broker failure. Default to `acks=all`.

9. **Consumer group ID collisions.** Two services sharing a `groupId` will share partitions — each gets half the messages. Use distinct, descriptive group IDs.

10. **Forgetting `transactionalId` stability.** Random transactional IDs break exactly-once fencing.

11. **Schema drift without registry.** Producer adds a field, downstream parser blows up. Use a schema registry or at least strict Avro/Protobuf contracts.

12. **No backup of consumer offsets.** A misconfigured ops command can reset offsets; recovery requires knowing what they were.

13. **Inadequate monitoring.** You need lag, ISR shrinkage, under-replicated partitions, request rates, GC times. Without them, you find out about problems from users.

**Follow-up 1: What's the cheapest, highest-leverage observability for Kafka?**
Two metrics: (1) **consumer group lag** per partition — exposes throughput problems before users see them. (2) **under-replicated partitions** — exposes broker/network problems before failover. Burrow (LinkedIn) handles lag alerting beautifully; broker JMX metrics handle the rest.

**Follow-up 2: How do you handle a "noisy neighbor" tenant on a shared Kafka cluster?**
Quotas. Kafka supports producer and consumer quotas per client/user, throttling them to a configured byte rate. Without quotas, one tenant's bug can starve everyone else. For multi-tenant SaaS, set quotas at signup and monitor.

---

*End of section 06. Next: Microservices & System Design (25 questions).*
