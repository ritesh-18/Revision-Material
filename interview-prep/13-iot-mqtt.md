# 13 — IoT & MQTT (10 Questions)

Your IoT projects ingested telemetry (CPU/memory/disk/temperature) over MQTT → message queue → S3, at 5K events/hour with guaranteed delivery and replay, plus time-series storage and ML-based anomaly detection. These cover the MQTT protocol, broker choices, security, time-series storage, and high-volume ingestion design.

---

### Q1. What is MQTT and why is it used for IoT instead of HTTP?

**Answer:**
**MQTT (Message Queuing Telemetry Transport)** is a lightweight publish/subscribe messaging protocol designed for constrained devices and unreliable networks. Devices publish messages to **topics** on a **broker**; subscribers receive messages for topics they've subscribed to.

**Why MQTT over HTTP for IoT:**

1. **Lightweight.** MQTT's fixed header is 2 bytes. HTTP headers are hundreds of bytes. For a battery-powered sensor sending tiny payloads, the overhead difference is huge for power and bandwidth.

2. **Persistent connection.** MQTT keeps a long-lived TCP connection. HTTP (without keep-alive tricks) sets up/tears down per request. For frequent small messages, the connection reuse saves enormous overhead.

3. **Push, not poll.** A subscriber gets messages pushed instantly. With HTTP, the device would have to poll ("any new commands?"), wasting bandwidth and adding latency.

4. **Pub/sub decoupling.** Publishers don't know subscribers. A sensor publishes telemetry; any number of backends subscribe. No point-to-point coupling.

5. **QoS levels.** Built-in delivery guarantees (0/1/2) tuned for lossy networks.

6. **Last Will & Testament.** The broker can announce a device went offline — crucial for monitoring fleets.

**When HTTP is still better:**
- Request/response semantics (device fetches config once on boot).
- Firewalls that only allow HTTP.
- Simple integrations without a broker.

```
   Devices (publishers)         Broker            Subscribers
   ┌──────────┐                ┌────────┐         ┌──────────────┐
   │ sensor 1 │──telemetry/1──▶│        │────────▶│ ingestion svc│
   ├──────────┤                │  MQTT  │         ├──────────────┤
   │ sensor 2 │──telemetry/2──▶│ broker │────────▶│ alerting svc │
   ├──────────┤                │        │         ├──────────────┤
   │ sensor 3 │──telemetry/3──▶│        │────────▶│ dashboard    │
   └──────────┘                └────────┘         └──────────────┘
```

**Follow-up 1: How does MQTT handle unreliable networks?**
Persistent sessions + QoS. With a persistent session (`cleanSession=false`), if a device disconnects, the broker queues QoS 1/2 messages for it and delivers on reconnect. QoS 1/2 add acknowledgments and retries. So a sensor on flaky cellular doesn't lose data when its connection drops.

**Follow-up 2: What's MQTT-SN and when would you use it?**
MQTT-SN (Sensor Networks) is a variant for non-TCP networks like Zigbee or low-power radio. It works over UDP, has shorter topic names (IDs instead of strings), and supports sleeping devices. Use it for ultra-constrained devices that can't run TCP/IP. Standard MQTT (over TCP) covers most IoT.

---

### Q2. Explain MQTT QoS levels in depth for IoT scenarios.

**Answer:**
(Builds on section 06 Q14 with IoT-specific reasoning.)

**QoS 0 — at most once ("fire and forget"):**
- No acknowledgment, no retry.
- Message may be lost if the connection drops mid-publish.
- **IoT use:** high-frequency redundant telemetry. If a temperature sensor reports every 5 seconds, losing one reading is fine — the next is right behind. Saves bandwidth and battery.

**QoS 1 — at least once:**
- Publisher stores the message, sends, waits for `PUBACK`. Retries if no ack.
- Possible duplicates (ack lost but broker received).
- **IoT use:** the common default. Telemetry you can't afford to lose but can tolerate occasional duplicates (your "guaranteed delivery" requirement). Downstream must be idempotent or dedupe.

**QoS 2 — exactly once:**
- Four-way handshake (PUBLISH/PUBREC/PUBREL/PUBCOMP).
- No loss, no duplicates.
- Slowest, most state on broker and client.
- **IoT use:** critical commands ("open the valve", "unlock the door"), billing events. Not for high-volume telemetry — the overhead doesn't scale.

**Trade-off table for IoT:**
| QoS | Bandwidth | Battery | Guarantee | Use |
|-----|-----------|---------|-----------|-----|
| 0 | Lowest | Best | May lose | Frequent redundant metrics |
| 1 | Medium | Good | At least once | Standard telemetry |
| 2 | Highest | Worst | Exactly once | Critical commands |

**Important:** QoS is negotiated separately on publish (publisher → broker) and subscribe (broker → subscriber). The effective QoS for the subscriber is `min(publish QoS, subscribe QoS)`.

**Follow-up 1: For your 5K events/hour with "guaranteed delivery and replay" — what QoS and why?**
QoS 1. Guarantees each event arrives (at least once), much cheaper than QoS 2. Combined with broker persistence (`cleanSession=false` + persistent storage), messages survive broker restarts and consumer downtime. The "replay" comes from the broker queuing undelivered messages plus your downstream archival to S3 (you can re-read from S3 to replay). True exactly-once handled by idempotent downstream processing, not QoS 2.

**Follow-up 2: Does QoS 2 actually guarantee exactly-once end-to-end?**
Only between publisher and broker, and broker and subscriber — as separate hops. True end-to-end exactly-once (publisher → final data store) requires idempotency in your processing. A QoS 2 message delivered once to your ingestion service could still be processed twice if your service crashes after processing but before acknowledging. QoS 2 reduces but doesn't eliminate the need for idempotent handlers.

---

### Q3. How do MQTT topics and wildcards work?

**Answer:**
MQTT topics are hierarchical strings, levels separated by `/`. Topics aren't pre-declared — publishing to a topic creates it implicitly.

**Topic structure example:**
```
fleet/region-us/device-42/telemetry/cpu
fleet/region-us/device-42/telemetry/memory
fleet/region-eu/device-99/status
```

**Wildcards (for subscriptions only):**
- **`+` (single level)** — matches exactly one level.
  - `fleet/+/device-42/telemetry/cpu` matches any region for device-42's CPU.
- **`#` (multi-level)** — matches all remaining levels; must be last.
  - `fleet/region-us/#` matches everything under region-us.
  - `fleet/+/+/telemetry/#` matches all telemetry from all devices.

**Topic design best practices:**
- **Hierarchical, specific-to-general or general-to-specific** consistently.
- **Put routing keys early** so wildcards are useful: `fleet/{region}/{device}/{metric}`.
- **No leading slash** (creates an empty first level).
- **Avoid spaces and `#`/`+` in actual topic names** (reserved).
- **Keep topics reasonably short** — they're sent with every message (QoS overhead).
- **Don't put high-cardinality data in topics** if it explodes subscription complexity, but DO use device IDs (that's the point).

**Code — subscriber with wildcards:**
```js
const client = mqtt.connect('mqtts://broker:8883', opts);

client.on('connect', () => {
  client.subscribe('fleet/+/+/telemetry/#', { qos: 1 });   // all telemetry, all devices
  client.subscribe('fleet/+/+/status', { qos: 1 });        // all status messages
});

client.on('message', (topic, payload) => {
  const [, region, deviceId, kind, ...rest] = topic.split('/');
  if (kind === 'telemetry') {
    handleTelemetry(region, deviceId, rest[0], JSON.parse(payload.toString()));
  } else if (kind === 'status') {
    handleStatus(region, deviceId, payload.toString());
  }
});
```

**Follow-up 1: Why design topics carefully upfront?**
Because topic structure determines what subscriptions are possible. If you put the device ID before the metric type (`device/42/cpu`), you can easily subscribe to "all of device 42's data" but not "all CPU data from all devices" without a broad wildcard + filtering. Topic hierarchy is your routing schema; changing it later means re-flashing devices. Design like you'd design a URL hierarchy.

**Follow-up 2: What's a "shared subscription" and why does it matter for scaling?**
`$share/{group}/{topic}` lets multiple subscribers in a group load-balance messages (like Kafka consumer groups). Without it, every subscriber to a topic gets every message — you can't horizontally scale consumers. With shared subscriptions, you run N ingestion workers and the broker distributes messages among them. Supported in MQTT 5 and most modern brokers (EMQX, HiveMQ, Mosquitto 2+).

---

### Q4. What are retained messages and Last Will & Testament?

**Answer:**

**Retained messages:**
- A publish with the `retain` flag tells the broker to store this message as the "last known value" for the topic.
- Any new subscriber to that topic immediately receives the retained message, without waiting for the next publish.
- Use for "current state" semantics: a device's last reported config, status, or sensor reading.

```js
// Device publishes its current status as retained
client.publish('fleet/us/device-42/status', JSON.stringify({ online: true, fw: '1.2.3' }), {
  retain: true, qos: 1,
});
// A dashboard subscribing later immediately gets the last status, not a blank.
```

To clear a retained message, publish an empty payload with `retain: true`.

**Last Will & Testament (LWT):**
- A message the client registers when it connects. If the client disconnects *ungracefully* (network drop, crash — not a clean disconnect), the broker publishes the LWT on the client's behalf.
- Use for offline detection: device sets LWT to `{online: false}`; if it vanishes, the broker announces it's gone.

```js
const client = mqtt.connect('mqtts://broker:8883', {
  will: {
    topic: 'fleet/us/device-42/status',
    payload: JSON.stringify({ online: false }),
    qos: 1,
    retain: true,
  },
});
// On normal operation, device publishes online:true (retained).
// If it crashes, broker publishes the will (online:false, retained).
```

**Combining them:** retained status + LWT gives you reliable presence. New subscribers see the current state; the broker auto-publishes "offline" if the device dies.

**Follow-up 1: What's the difference between a clean and unclean disconnect?**
A clean disconnect = the client sends a `DISCONNECT` packet (graceful shutdown). The broker does NOT publish the LWT. An unclean disconnect = the TCP connection drops without DISCONNECT (crash, network loss, power off). The broker detects this via keepalive timeout and publishes the LWT. So LWT specifically catches failures, not graceful shutdowns.

**Follow-up 2: How does keepalive detect a dead device?**
The client sends a `PINGREQ` at the keepalive interval; the broker responds `PINGRESP`. If the broker doesn't hear from a client within 1.5× the keepalive interval, it considers the client dead, closes the connection, and publishes the LWT. Tune keepalive based on how fast you need offline detection vs how much ping traffic you can afford (shorter = faster detection, more traffic).

---

### Q5. Compare the main MQTT brokers — Mosquitto, EMQX, HiveMQ, AWS IoT Core.

**Answer:**

**Mosquitto (Eclipse):**
- Lightweight, open-source, single-node (clustering needs a bridge or external tooling).
- Great for small deployments, edge gateways, dev/test.
- Limited scale (tens of thousands of connections).
- Simple to run; minimal features.

**EMQX:**
- Open-source + commercial. Built for massive scale (millions of connections).
- Native clustering, shared subscriptions, rule engine (route messages to Kafka/DBs).
- MQTT 5 full support.
- Good choice for serious self-hosted IoT at scale.

**HiveMQ:**
- Commercial (with a community edition). Enterprise-focused.
- Strong clustering, extensibility (plugins), excellent observability.
- Used in automotive, large-scale industrial IoT.

**AWS IoT Core:**
- Fully managed. No brokers to run.
- Built-in device registry, device shadows, rules engine (route to Lambda, DynamoDB, Kinesis, S3).
- Per-message + per-connection pricing.
- Great for AWS-native stacks; cost grows with volume.

**Decision matrix:**
| Need | Choice |
|------|--------|
| Small / edge / dev | Mosquitto |
| Self-hosted at scale | EMQX |
| Enterprise, mission-critical | HiveMQ |
| AWS-native, don't want to run brokers | AWS IoT Core |
| Low volume, AWS | IoT Core (managed wins) |
| High volume, cost-sensitive | EMQX self-hosted |

**For your pipeline (5K events/hour):** that's low volume — IoT Core would be cost-effective and zero-ops, or a single Mosquitto/EMQX instance on EC2 if you want control. At 5K/hour you don't need clustering.

**Follow-up 1: When does the broker become the bottleneck?**
At high connection counts (millions of devices) or high message throughput. Single-node Mosquitto tops out around tens of thousands of connections. EMQX/HiveMQ clusters scale to millions. The bottleneck is usually connection count (each persistent connection costs memory/file descriptors) before message throughput. Monitor connection count, message rate, and broker memory.

**Follow-up 2: What's a "device shadow" in AWS IoT Core?**
A persistent JSON document representing a device's last reported state plus its desired state. Apps read/write the shadow even when the device is offline; when the device reconnects, it syncs. Solves the "device is asleep but I want to change its config" problem — you write the desired state to the shadow, the device picks it up on next wake. EMQX and others have similar features.

---

### Q6. How do you secure an MQTT deployment?

**Answer:**

**Transport security:**
- **TLS** on the broker (port 8883 instead of 1883). Encrypts all traffic. Non-negotiable for production.

**Authentication:**
- **Username/password** — simplest; credentials per device. Storage and rotation matter.
- **Client certificates (mutual TLS)** — each device has a unique cert. Strong identity; revocable. The gold standard for IoT.
- **Token-based** — JWT or platform-specific tokens (AWS IoT uses SigV4 or X.509).

**Authorization:**
- **Topic ACLs** — restrict which topics each device can publish/subscribe to. Device 42 can only publish to `fleet/+/device-42/#`, not other devices' topics. Prevents a compromised device from spoofing others or eavesdropping.

**Code — client cert connection:**
```js
const client = mqtt.connect('mqtts://broker:8883', {
  key: fs.readFileSync('device-42.key'),
  cert: fs.readFileSync('device-42.crt'),
  ca: fs.readFileSync('ca.crt'),
  rejectUnauthorized: true,
  clientId: 'device-42',
});
```

**Broker ACL config (Mosquitto example):**
```
# Device can only publish to its own telemetry topics
user device-42
topic write fleet/us/device-42/#
topic read  fleet/us/device-42/commands/#
```

**Additional hardening:**
- **Unique credentials per device.** Never share one cert/password across the fleet — one leak compromises all.
- **Certificate rotation** — short-lived certs, automated renewal.
- **Rate limiting** per device — a compromised device shouldn't be able to flood the broker.
- **Disable anonymous access.** `allow_anonymous false`.
- **Network isolation** — broker in a private subnet, accessible only through controlled ingress.

**Follow-up 1: Why client certificates over username/password for IoT?**
Certs give cryptographic identity that can't be guessed or brute-forced, are individually revocable (revoke device 42 without affecting others), and don't transmit a reusable secret. Passwords can leak, be reused, be brute-forced. For a fleet of thousands of devices, per-device certs with a revocation list are far more manageable and secure.

**Follow-up 2: How do you handle a compromised device?**
Revoke its certificate (add to CRL or OCSP). The broker rejects its next connection. Then investigate: what did it access (ACLs limited its blast radius), what did it publish (check for spoofed data). This is why per-device identity + topic ACLs matter — the damage from one compromised device is bounded to its own topics, and you can cut it off individually.

---

### Q7. How do you store time-series telemetry data efficiently?

**Answer:**
Telemetry is append-heavy, queried by time range and device, rarely updated. Storage choices:

**Purpose-built time-series databases:**
- **TimescaleDB** (Postgres extension) — hypertables auto-partition by time; compression; SQL interface. Great if you already use Postgres.
- **InfluxDB** — purpose-built TSDB, optimized for metrics. Good tooling.
- **ClickHouse** — columnar OLAP, blazing for analytical time-series queries at huge scale.
- **Prometheus** — for operational metrics (not arbitrary telemetry storage; short retention).

**General databases adapted:**
- **Postgres with partitioning + BRIN indexes** (section 03 Q1, Q24) — partition by month, BRIN index on timestamp. Works well to moderate scale.
- **MongoDB time-series collections** (section 04 Q12) — columnar bucketing, compression.
- **Cassandra/DynamoDB** — wide-column, partition by device, sort by time. Scales horizontally.

**Cold storage:**
- **S3 + Parquet** — archive old data as columnar Parquet files partitioned by date/device. Query with Athena. Cheapest for "rarely accessed but must keep."

**Tiered architecture (what you'd want for your project):**
```
   Hot (last 7 days):    TimescaleDB / Postgres partitions — fast queries, dashboards
   Warm (7-90 days):     Compressed partitions or InfluxDB
   Cold (> 90 days):     S3 Parquet — Athena for occasional analysis / ML training
```

**Key techniques:**
- **Partition by time** — drop old partitions instantly instead of slow DELETEs.
- **Downsampling/rollups** — store raw data short-term, aggregated (hourly/daily averages) long-term.
- **Compression** — time-series compresses 10-20x (similar values, small deltas).
- **Appropriate indexes** — `(device_id, recorded_at DESC)` for "this device's recent data."

**Code — TimescaleDB hypertable:**
```sql
CREATE TABLE telemetry (
  device_id  uuid NOT NULL,
  recorded_at timestamptz NOT NULL,
  metric     text NOT NULL,
  value      double precision NOT NULL
);
SELECT create_hypertable('telemetry', 'recorded_at', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX ON telemetry (device_id, recorded_at DESC);

-- Continuous aggregate (rollup) — auto-maintained hourly averages
CREATE MATERIALIZED VIEW telemetry_hourly
WITH (timescaledb.continuous) AS
SELECT device_id, metric, time_bucket('1 hour', recorded_at) AS hour,
       avg(value), max(value), min(value)
FROM telemetry GROUP BY device_id, metric, hour;

-- Compression policy for data older than 7 days
ALTER TABLE telemetry SET (timescaledb.compress);
SELECT add_compression_policy('telemetry', INTERVAL '7 days');
```

**Follow-up 1: Why downsample/roll up?**
Raw per-second data for a year is enormous and rarely queried at full resolution. Dashboards showing "last year" want hourly or daily averages, not billions of points. Store raw data short-term (debugging, recent analysis), pre-aggregated rollups long-term. Saves storage and makes long-range queries fast. Your ML pipeline trains on rollups + recent raw data.

**Follow-up 2: When would you choose ClickHouse over TimescaleDB?**
ClickHouse for massive analytical scale (billions of rows, complex aggregations across huge ranges) — it's a columnar OLAP engine built for this. TimescaleDB when you want time-series capabilities within your existing Postgres (transactional consistency, joins with relational data, familiar SQL/tooling). For your project's scale (5K/hour ≈ 44M/year), TimescaleDB or even plain partitioned Postgres is plenty; ClickHouse is overkill until you're at billions of points.

---

### Q8. How do you design high-volume sensor ingestion with backpressure and replay?

**Answer:**
The challenge: devices push data faster than you can process at peak; you must not lose data; you must be able to replay.

**Architecture — buffer + decouple:**
```
   Devices ──MQTT──▶ Broker ──▶ Ingestion ──▶ Durable Queue ──▶ Processors ──▶ Storage
                                  (thin)        (Kafka/Kinesis)   (scalable)      (TSDB + S3)
```

**Key design principles:**

1. **Thin ingestion layer.** The service subscribed to MQTT does minimal work: validate, maybe enrich, then immediately push to a durable queue (Kafka/Kinesis). Don't do heavy processing inline — that creates backpressure on the broker.

2. **Durable buffer absorbs spikes.** Kafka/Kinesis holds messages; processors consume at their own pace. A spike fills the queue (lag grows) but nothing is lost. This is backpressure handled by buffering.

3. **Scale processors independently.** When lag grows, add consumers (up to partition count). They drain the backlog.

4. **Replay from the buffer.** Kafka retains messages for the retention period — re-process by resetting consumer offsets. For longer-term replay, archive to S3 and re-ingest.

5. **Backpressure at the broker.** If even the ingestion layer is overwhelmed, MQTT QoS 1 + persistent sessions queue messages at the broker. Devices with QoS 1 retry until acked.

**Idempotency for replay safety:**
- Each message has a unique ID (device + timestamp + sequence).
- Processors dedupe or use idempotent writes (UPSERT keyed by message ID).
- Replaying the same message twice produces the same result.

**Code — thin ingestion:**
```js
client.on('message', async (topic, payload) => {
  const event = parseAndValidate(topic, payload);
  // Don't process here — just durably enqueue
  await kafka.send({
    topic: 'telemetry-raw',
    messages: [{ key: event.deviceId, value: JSON.stringify(event) }],
    acks: -1,    // durable
  });
});
// Processors consume telemetry-raw, write to TSDB + S3, can replay anytime
```

**Follow-up 1: Where exactly does backpressure get handled?**
At multiple layers: (1) MQTT QoS 1 + broker persistence buffers when ingestion is slow; (2) the durable queue (Kafka) buffers when processors are slow; (3) processors scale to drain. The key insight: by inserting durable buffers between fast producers and slow consumers, you convert "drop data under load" into "lag grows, then catches up." Monitor lag as the health signal.

**Follow-up 2: How do you replay a specific time window after fixing a processing bug?**
If data is in Kafka within retention: reset the consumer group offset to the timestamp before the bug, reprocess. If beyond Kafka retention: read the archived S3 Parquet files for that window, re-publish to a replay topic, process with the fixed code. Idempotent writes ensure reprocessing doesn't create duplicates. This is exactly the "replay support" on your resume.

---

### Q9. How would you do anomaly detection / predictive maintenance on telemetry?

**Answer:**
(Your "predictive hardware maintenance" project.)

**Pipeline:**
```
   Telemetry stream ──▶ Feature extraction ──▶ ML model ──▶ Anomaly score ──▶ Alert / action
```

**Approaches, simple to sophisticated:**

1. **Threshold rules** — "alert if temperature > 85°C." Simple, interpretable, brittle (misses gradual degradation, false alarms on transient spikes).

2. **Statistical anomaly detection** — track rolling mean/stddev; flag values beyond N standard deviations (z-score), or use EWMA (exponentially weighted moving average). Catches deviations from a device's own baseline.

3. **Time-series forecasting** — predict the next value (ARIMA, Prophet, or a small neural net); flag large prediction errors as anomalies.

4. **ML classification/regression** — train on labeled failures: features (CPU trend, temperature variance, disk error rate) → failure probability. Requires historical failure data.

5. **Unsupervised** — clustering or autoencoders flag "unusual" patterns without labeled failures. Good when failures are rare/unlabeled.

**Feature engineering for hardware telemetry:**
- Rolling statistics (mean, max, variance over windows).
- Rate of change (is temperature climbing?).
- Cross-metric correlations (high CPU + high temp + fan at max = thermal problem).
- Time-since-last-event features.

**Serving:**
- **Batch** — periodic job scores recent telemetry, flags devices at risk.
- **Streaming** — score each event as it arrives (Kafka Streams, Flink, or a consumer that calls the model).
- The Python ML service reads features, returns scores; your Node backend acts on them (alert, schedule maintenance).

**Acting on predictions:**
- Don't page on every anomaly — that causes alert fatigue.
- Aggregate: "device X has elevated risk for 30 minutes" → ticket.
- The Claude/LLM angle from your resume: the model reads recent telemetry context and explains whether an anomaly matches a known failure pattern, escalating only genuine concerns — reducing false-positive pages.

**Follow-up 1: How do you avoid alert fatigue?**
Tune for precision over recall in the alerting layer — better to occasionally miss than to cry wolf constantly. Techniques: require sustained anomalies (not single spikes), correlate multiple signals, use a "confidence" threshold, batch related alerts, and add a reasoning layer (LLM or rules) that suppresses anomalies with known benign explanations. An ignored alerting system is worse than none.

**Follow-up 2: Why is unsupervised detection appealing for hardware failure?**
Because labeled failure data is scarce — hardware doesn't fail often, and when it does, you might not have clean labels. Unsupervised methods (autoencoders, isolation forests) learn "normal" from abundant healthy data and flag deviations, without needing failure examples. The trade-off: they flag *unusual*, not necessarily *failing*, so you need a triage layer to separate "interesting anomaly" from "imminent failure."

---

### Q10. What are the common IoT/MQTT production gotchas?

**Answer:**

1. **No TLS.** Plaintext MQTT (port 1883) exposed to the internet = anyone can read/inject. Always TLS (8883), always.

2. **Shared credentials across devices.** One leaked password/cert compromises the whole fleet. Per-device identity, revocable.

3. **No topic ACLs.** A compromised device can publish to other devices' topics or subscribe to everything. Restrict each device to its own topics.

4. **QoS 0 for data you can't lose.** Silent data loss on network blips. Use QoS 1 for important telemetry.

5. **Unbounded retained messages.** Every topic with a retained message stays forever; misuse creates memory growth. Clean up stale retained messages.

6. **No shared subscriptions → can't scale consumers.** Every subscriber gets every message; you can't horizontally scale ingestion. Use `$share/` groups.

7. **Heavy processing in the MQTT message handler.** Slow processing backs up the broker. Keep ingestion thin; offload to a durable queue.

8. **No persistent sessions for intermittent devices.** Device reconnects, missed all messages during downtime. Use `cleanSession=false` (MQTT 3) / session expiry (MQTT 5) for devices that need queued messages.

9. **Keepalive too long → slow offline detection.** Device dies, but you don't know for minutes. Tune keepalive to your detection SLA.

10. **Broker as a single point of failure.** One Mosquitto node down = whole fleet offline. Cluster (EMQX/HiveMQ) or managed (IoT Core) for HA.

11. **No backpressure handling.** Spike in telemetry overwhelms processors, data lost. Durable buffer (Kafka) between ingestion and processing.

12. **Time-series storage in a regular table.** Telemetry in an unpartitioned Postgres table bloats, slows down, and DELETEs are agony. Use partitioning / TSDB / time-series collections.

13. **No replay path.** Bug in processing = data lost forever. Keep raw data in Kafka (short-term) + S3 (long-term) for reprocessing.

14. **Clock skew between devices.** Device timestamps drift; ordering and windowing break. Either trust broker-receive time or sync device clocks (NTP) and validate.

15. **Storing telemetry timestamps without timezone.** Always UTC, always ISO 8601 / timestamptz.

**Follow-up 1: How do you handle devices with bad/drifting clocks?**
Two strategies: (1) use the broker's receive timestamp instead of the device's claimed time for ordering — reliable but loses true event time; (2) keep both — device time for analysis, server time for ordering — and detect drift (device time far from server time → flag). For predictive maintenance, the *relative* sequence and trends matter more than absolute time, so moderate drift is often tolerable.

**Follow-up 2: What's the single biggest scaling concern for an IoT backend?**
Connection count, then ingestion throughput, then storage volume. A million devices = a million persistent connections (broker memory/FD pressure) before message volume even matters. Then the ingestion + storage must absorb the aggregate write rate. Design: cluster the broker, use shared subscriptions to scale ingestion, buffer with Kafka, and tier storage (hot DB + cold S3). Each layer scales independently.

---

*End of section 13. Next: AI / LLM / RAG (20 questions).*
