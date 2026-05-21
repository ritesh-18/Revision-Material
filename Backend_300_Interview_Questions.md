# 🛠️ Backend Engineer – 300 Interview Questions with Detailed Solutions

> Covers: Node.js, Express, Databases, REST, Auth, Caching, Queues, Testing, DevOps & Scenarios. Target: 2 YOE.

---

## 📌 Table of Contents
1. [Node.js – Core (Q1–Q40)](#nodejs--core)
2. [Express.js (Q41–Q70)](#expressjs)
3. [Databases – SQL (Q71–Q110)](#databases--sql)
4. [Databases – MongoDB / NoSQL (Q111–Q140)](#databases--mongodb--nosql)
5. [REST API Design (Q141–Q170)](#rest-api-design)
6. [Authentication & Security (Q171–Q200)](#authentication--security)
7. [Caching & Redis (Q201–Q220)](#caching--redis)
8. [Message Queues (Q221–Q235)](#message-queues)
9. [Testing (Q236–Q255)](#testing)
10. [DevOps & Deployment (Q256–Q275)](#devops--deployment)
11. [Scenario-Based Questions (Q276–Q300)](#scenario-based-questions)

---

## Node.js – Core

### Q1. What is the event loop in Node.js? Explain all its phases in order.

The event loop is Node.js's mechanism for non-blocking I/O. It runs single-threaded JavaScript while delegating async I/O to the `libuv` thread pool. Each iteration ("tick") visits phases in order:

1. **Timers** — `setTimeout` / `setInterval` callbacks whose threshold has elapsed.
2. **Pending callbacks** — deferred I/O callbacks (e.g., TCP errors).
3. **Idle, prepare** — internal use only.
4. **Poll** — retrieves new I/O events; runs I/O callbacks. Blocks here if nothing else to do (subject to timer thresholds).
5. **Check** — `setImmediate` callbacks.
6. **Close callbacks** — e.g., `socket.on('close')`.

Between every phase, the **microtask queue** is drained: `process.nextTick` callbacks first, then `Promise.then/catch/finally` callbacks.

```
┌───────────────────────────┐
│         timers            │
├───────────────────────────┤
│     pending callbacks     │
├───────────────────────────┤
│       idle, prepare       │
├───────────────────────────┤
│           poll            │
├───────────────────────────┤
│           check           │
├───────────────────────────┤
│      close callbacks      │
└───────────────────────────┘
```

---

### Q2. Difference between `process.nextTick()`, `setImmediate()`, and `setTimeout(0)`? Which runs first?

| API | Runs in | Priority |
|---|---|---|
| `process.nextTick()` | Microtask queue (between every phase) | Highest |
| Promise `.then()` | Microtask queue | Just after nextTick |
| `setTimeout(fn, 0)` | Timers phase | After microtasks |
| `setImmediate(fn)` | Check phase | After poll |

**Execution order (top-level script):**
```js
setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));
Promise.resolve().then(() => console.log("promise"));
process.nextTick(() => console.log("nextTick"));
console.log("sync");
// sync → nextTick → promise → timeout → immediate
```

**Inside an I/O callback**, `setImmediate` always fires **before** `setTimeout(fn, 0)` because the loop is already in poll phase and check comes next.

---

### Q3. A function using `setTimeout(fn, 0)` is called inside a Promise `.then()`. What order does it execute in?

```js
console.log("A");
Promise.resolve().then(() => {
  console.log("B");
  setTimeout(() => console.log("C"), 0);
});
console.log("D");
// Output: A, D, B, C
```

**Why:** A and D are synchronous (call stack). After sync code finishes, microtasks drain → B logs and `setTimeout` is *registered*. The timers phase later fires C.

---

### Q4. What are Node.js streams? Four types and use cases.

Streams are abstractions for processing data piece-by-piece instead of loading it all into memory.

| Type | Purpose | Example |
|---|---|---|
| **Readable** | Source of data | `fs.createReadStream`, HTTP request |
| **Writable** | Destination of data | `fs.createWriteStream`, HTTP response |
| **Duplex** | Both read + write (independent) | TCP socket |
| **Transform** | Duplex that modifies data passing through | `zlib.createGzip`, `crypto.createCipheriv` |

```js
fs.createReadStream("big.csv")
  .pipe(zlib.createGzip())        // Transform
  .pipe(fs.createWriteStream("big.csv.gz"));  // Writable
```

Use streams whenever input size is unbounded, large, or where memory must remain flat (file processing, video transcoding, log shipping).

---

### Q5. Your server reads a 2GB CSV into memory and crashes. Fix using streams.

**Crashing approach:**
```js
const data = fs.readFileSync("big.csv", "utf8"); // ❌ loads 2GB into RAM
data.split("\n").forEach(processLine);
```

**Stream fix:**
```js
const readline = require("readline");
const rl = readline.createInterface({
  input: fs.createReadStream("big.csv"),
  crlfDelay: Infinity,
});
rl.on("line", processLine);
rl.on("close", () => console.log("done"));
```

This processes one line at a time; memory stays constant (~64KB buffer). For CSV specifically, use `csv-parser` with streams.

---

### Q6. What is backpressure in streams? How to handle it?

**Backpressure** occurs when a writable consumer cannot keep up with the readable producer — data piles up in internal buffers, blowing memory.

`stream.write(chunk)` returns `false` when the internal buffer exceeds `highWaterMark` (default 16KB). The producer must pause until the writable emits `'drain'`.

```js
src.on("data", (chunk) => {
  if (!dst.write(chunk)) src.pause();
});
dst.on("drain", () => src.resume());
```

**`.pipe()` handles this automatically** — that's why it's preferred. With `pipeline()` from `stream` you also get proper error propagation and cleanup:

```js
const { pipeline } = require("stream/promises");
await pipeline(src, transform, dst);
```

---

### Q7. `require` (CommonJS) vs `import` (ESM) in Node.js.

| Feature | CommonJS (`require`) | ESM (`import`) |
|---|---|---|
| Loading | Synchronous | Asynchronous, static |
| Resolution | Runtime | Parse time |
| Tree-shaking | No | Yes |
| Top-level await | No | Yes |
| `__dirname` / `__filename` | Available | Use `import.meta.url` |
| File extension | Optional | Required (`.js`, `.mjs`) |

**Switching to ESM:** set `"type": "module"` in `package.json`, or rename files to `.mjs`. Mixed packages can use `"exports"` field with conditional entries.

---

### Q8. Circular dependencies — what happens and how to fix.

When `a.js` requires `b.js` and `b.js` requires `a.js`, the second `require` returns the **partially exported** object as it stood at the time of the cycle:

```js
// a.js
exports.foo = "foo from a";
const b = require("./b"); // b runs, requires a back
exports.bar = "bar from a";

// b.js
const a = require("./a"); // returns { foo: "foo from a" } — bar missing
console.log(a.bar); // undefined ❌
```

**Fixes:**
1. **Refactor** to remove the cycle — extract shared code to a third module.
2. **Lazy require** inside the function: `function fn() { const a = require("./a"); ... }`.
3. **Dependency injection** — pass dependencies as parameters instead of importing.

---

### Q9. Node.js `cluster` module — how it improves performance.

Node runs JS in one thread; `cluster` forks **N worker processes** (one per CPU core), each with its own event loop. The OS round-robins incoming connections (on Linux: through a master process socket-share).

```js
const cluster = require("cluster");
const os = require("os");
if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  cluster.on("exit", () => cluster.fork()); // auto-restart
} else {
  require("./server");
}
```

Workers share the same port. Each handles independent requests in parallel. In production, use **PM2** or container orchestration (k8s) instead of hand-rolling cluster.

---

### Q10. Single Node.js process maxing out one core under load. What do you do?

**Diagnose first:**
1. CPU profiler (`node --prof app.js` then `--prof-process`) or `clinic flame` to identify hot functions.
2. Check whether it's actually CPU-bound (heavy computation) or perceived (GC pressure, sync I/O).

**Solutions in order of preference:**
1. **Optimize the hot path** — algorithmic improvement, caching.
2. **Offload CPU work to worker_threads** (image processing, crypto, parsing).
3. **Scale horizontally** — cluster module or container replicas behind a load balancer.
4. **Move CPU-heavy task to a queue** — background worker processes the task asynchronously.

Don't reach for clustering before profiling — often the win is in fixing a single slow function.

---

### Q11. `worker_threads` vs clustering.

| `worker_threads` | `cluster` |
|---|---|
| In-process threads, shared memory via `SharedArrayBuffer` | Separate OS processes |
| Lightweight (<1MB each) | Heavy (~30MB each) |
| Communicate via `postMessage` | Communicate via IPC |
| Good for **CPU-bound tasks** | Good for **scaling HTTP concurrency** |

Use `worker_threads` when you need to offload a synchronous CPU-heavy computation (sharp image resize, PDF generation, parsing) without blocking the event loop. Use `cluster` to handle more concurrent HTTP requests.

```js
const { Worker } = require("worker_threads");
new Worker("./resize.js", { workerData: { file } });
```

---

### Q12. `fs.readFile` vs `fs.createReadStream`.

- `fs.readFile(path, cb)` — reads entire file into memory, then invokes callback with the buffer. Good for small files (<10MB) where you need random access to all data.
- `fs.createReadStream(path)` — streams chunks; ideal for large files, network transfer, transformations.

```js
// Small JSON config: readFile is fine
const config = JSON.parse(await fs.promises.readFile("config.json"));

// Large CSV: stream
fs.createReadStream("data.csv").pipe(parser).pipe(writer);
```

Memory footprint:
- `readFile` of 2GB = 2GB RAM.
- `createReadStream` = ~64KB regardless of file size.

---

### Q13. `process.env` and safe management across environments.

`process.env` exposes shell environment variables. Best practices:

1. **Never commit secrets** — add `.env` to `.gitignore`. Commit `.env.example` with non-sensitive placeholders.
2. **Load locally** via `dotenv`: `require("dotenv").config()` (or `node --env-file=.env` in Node 20+).
3. **Validate on boot** with a schema (Zod, Joi):
   ```js
   const env = z.object({
     DATABASE_URL: z.string().url(),
     PORT: z.coerce.number().default(3000),
     NODE_ENV: z.enum(["development","staging","production"]),
   }).parse(process.env);
   ```
4. **Production secrets** — use a secret manager (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Doppler). Never use `.env` files in production.
5. **Per-env overrides** — `.env.development`, `.env.production`, etc.

---

### Q14. How does Node.js handle uncaught exceptions?

By default, an uncaught exception crashes the process with a non-zero exit code. You can intercept globally:

```js
process.on("uncaughtException", (err) => {
  logger.fatal({ err }, "uncaughtException");
  // ⚠️ Application state may be corrupted; exit and let supervisor restart
  process.exit(1);
});
```

**Don't keep running** after `uncaughtException` — the V8 docs explicitly warn that the process is in an undefined state. Use this hook only to flush logs and shut down cleanly. Rely on a process supervisor (PM2, systemd, k8s) to restart.

---

### Q15. `UnhandledPromiseRejectionWarning` — catch and handle globally.

In Node 15+, unhandled rejections **terminate** the process by default. Always attach a global handler for logging:

```js
process.on("unhandledRejection", (reason, promise) => {
  logger.error({ reason }, "unhandledRejection");
  // optionally rethrow: throw reason;
});
```

**Real fix:** track down the source. Every `await` or `.then()` should have error handling, either via `try/catch` or `.catch()`:

```js
// ❌ rejection silently lost
someAsyncFn();

// ✅
someAsyncFn().catch(err => logger.error(err));

// ✅ in async function
try { await someAsyncFn(); } catch (err) { ... }
```

---

### Q16. `EventEmitter` — implement pub/sub.

`EventEmitter` is Node's built-in observer pattern. Most Node objects (streams, sockets) inherit from it.

```js
const { EventEmitter } = require("events");

class PubSub extends EventEmitter {
  publish(topic, data) { this.emit(topic, data); }
  subscribe(topic, handler) { this.on(topic, handler); return () => this.off(topic, handler); }
}

const bus = new PubSub();
const unsub = bus.subscribe("user:created", (u) => console.log("New user:", u.id));
bus.publish("user:created", { id: 1 });
unsub();
```

**Watch out for:**
- Default max listeners = 10. Raise with `bus.setMaxListeners(100)` or you'll get `MaxListenersExceededWarning`.
- Always remove listeners on cleanup to avoid memory leaks.

---

### Q17. Memory leaks in Node.js — common causes and detection.

**Common causes:**
1. **Forgotten timers** — `setInterval` not cleared.
2. **Event listener accumulation** — listeners added in a loop, never removed.
3. **Large module-scope caches** — Map/array growing unbounded.
4. **Closures capturing big objects** — a request handler that closes over the entire request body in a long-lived callback.
5. **Global variables** — accidental `foo = ...` in non-strict mode.

**Detection:**
- `process.memoryUsage()` over time — if `heapUsed` only grows, leak suspected.
- `--inspect` + Chrome DevTools → Memory tab → heap snapshots; diff to find growing object counts.
- `clinic doctor` or `clinic heapprofiler` for automated diagnosis.

---

### Q18. Profile and debug growing memory.

1. **Confirm the trend:** sample `process.memoryUsage().heapUsed` over hours. If RSS grows linearly, it's a leak.
2. **Take heap snapshots** at intervals (30 min apart) via Chrome DevTools (`node --inspect`) or `heapdump`/`v8.writeHeapSnapshot()`.
3. **Compare snapshots** — sort retained-size descending; look at "Objects allocated between snapshots." Common culprits: large arrays, closure scopes, listener arrays.
4. **Identify the retainer chain** — DevTools shows which references prevent GC.
5. **Reproduce in dev** by running a load test (`autocannon`) and snapshotting before/after.

Once found, fix by clearing intervals, unsubscribing listeners, bounding caches (LRU), or breaking closures.

---

### Q19. What is `libuv` and its role?

`libuv` is a C library providing Node.js's async I/O primitives. It contains:
- The **event loop** implementation.
- A **thread pool** (default 4 threads) for ops that can't be async at the OS level (`fs`, DNS lookups, some crypto).
- Cross-platform abstractions: `epoll` (Linux), `kqueue` (BSD/macOS), IOCP (Windows).
- Timers, child processes, IPC.

JavaScript runs single-threaded on the V8 thread, but `libuv` allows Node to offload blocking syscalls. Increase thread pool size via `UV_THREADPOOL_SIZE=16` if you're bottlenecked by, e.g., bcrypt or `crypto.pbkdf2`.

---

### Q20. CPU-bound vs I/O-bound — how Node handles each.

| Type | Description | Node behavior |
|---|---|---|
| I/O-bound | Waiting on disk/network/db | Excellent — non-blocking, scales to thousands of connections |
| CPU-bound | Heavy computation | Poor — blocks event loop, freezes all other requests |

**Examples:**
- I/O-bound: API gateway, real-time chat, REST CRUD app → Node ideal.
- CPU-bound: video encoding, ML inference, large JSON parsing → offload to `worker_threads` or a separate service.

**Rule of thumb:** if a task synchronously runs >50ms, move it off the main thread.

---

### Q21. Graceful shutdown — finish in-flight requests.

```js
const server = app.listen(3000);
let shuttingDown = false;

function shutdown() {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info("SIGTERM received, draining...");

  server.close(async () => {     // stops accepting new connections
    await db.close();             // close pool
    await redis.quit();
    process.exit(0);
  });

  // force exit after 30s safety net
  setTimeout(() => { logger.error("forced exit"); process.exit(1); }, 30_000).unref();
}

process.on("SIGTERM", shutdown);
process.on("SIGINT", shutdown);
```

Also add a `/healthz` endpoint that returns 503 once `shuttingDown` is true so the load balancer stops routing traffic.

---

### Q22. `SIGTERM` vs `SIGKILL`.

- **SIGTERM (15)** — polite termination request. Catchable by the process; allows cleanup. This is what Kubernetes, Docker, PM2 send first.
- **SIGKILL (9)** — forceful termination. **Cannot be caught or ignored**. Kernel kills the process immediately. Sent by k8s after `terminationGracePeriodSeconds` elapses (default 30s).

```js
process.on("SIGTERM", () => {
  // run graceful shutdown
});
// SIGKILL handler doesn't exist — the process is just gone
```

Design your shutdown to complete well within the grace period.

---

### Q23. `npm ci` vs `npm install`.

| `npm install` | `npm ci` |
|---|---|
| May modify `package-lock.json` | Fails if lockfile out of sync |
| Installs based on package.json + lockfile | Installs **exactly** from lockfile |
| Slower (resolves graph) | Faster (no resolution) |
| Allowed in dev | Recommended in CI/CD |

`npm ci` deletes `node_modules` first to guarantee a clean install — perfect for reproducible builds.

---

### Q24. Purpose of `package-lock.json` — what if you delete it?

It pins **exact versions** of every transitive dependency, ensuring everyone (and CI) installs the identical dependency tree. Without it, `npm install` re-resolves versions from `^1.2.3` ranges and may pull different patch/minor releases over time.

**If deleted:** next `npm install` recreates it from current registry state — risks introducing patch updates that break things, and breaks reproducibility across machines.

Always commit `package-lock.json`.

---

### Q25. `dependencies` vs `devDependencies` vs `peerDependencies`.

- **dependencies** — required at runtime in production (express, pg). Installed for everyone.
- **devDependencies** — needed only for local dev/build/test (jest, eslint, ts-node). Skipped with `npm install --production` or `NODE_ENV=production npm ci`.
- **peerDependencies** — declares "this library expects the host to provide X." Common in plugins (`eslint-plugin-react` peers on `eslint`). Avoids version duplication and conflicts. Installed by the *consumer*, not auto-installed.

---

### Q26. Debugging Node.js — tools.

- **`node --inspect app.js`** — exposes V8 debugger on port 9229; attach Chrome DevTools (`chrome://inspect`) or VS Code.
- **`--inspect-brk`** — pauses before first line; useful for short-running scripts.
- **VS Code launch config** — `"type": "node"`, `"request": "launch"`.
- **`console.log` + structured logging** (`pino`, `winston`) — still the most-used.
- **Profilers** — `clinic doctor` / `flame` / `bubbleprof`, `0x` for flame graphs.
- **APM** — Datadog, New Relic, Sentry for production tracing and error context.
- **`util.inspect(obj, { depth: null })`** — pretty-print circular/deep structures.

---

### Q27. `async_hooks` — use case.

`async_hooks` tracks the lifecycle of async resources, letting you correlate work across `await` boundaries. Used to implement:
- **Request-scoped context** (trace IDs, user IDs) without passing through every function.
- **AsyncLocalStorage** (built on top of async_hooks) is the modern recommended API:

```js
const { AsyncLocalStorage } = require("async_hooks");
const als = new AsyncLocalStorage();

app.use((req, res, next) => {
  als.run({ requestId: req.headers["x-request-id"] }, next);
});

logger.info(`processing for ${als.getStore().requestId}`);
```

Raw `async_hooks` has performance overhead — prefer `AsyncLocalStorage` for most cases.

---

### Q28. `ECONNRESET` intermittently — meaning and debug.

`ECONNRESET` = TCP connection forcibly closed by the peer (RST packet) — your end was still reading/writing.

**Common causes:**
1. **Keep-alive timeouts mismatch** — downstream load balancer closes idle connections before your client detects it. Fix: align `keepAliveTimeout` and `headersTimeout` in HTTP server / agent.
2. **Server crash / restart** — peer killed mid-request.
3. **Firewall / NAT idle disconnect** — long-lived idle connection dropped silently.
4. **Server overload** — backlog full, OS RSTs incoming connections.

**Debug:**
- `tcpdump` / Wireshark to confirm direction of RST.
- Check ALB/Nginx access logs at the time of error.
- For Node HTTP client: ensure `Agent` `keepAlive` true + reasonable `keepAliveTimeout`.

---

### Q29. Sync vs async errors in Node.js — propagation.

```js
// Sync error
function readSync() {
  return JSON.parse(invalid); // throws synchronously
}
try { readSync(); } catch (e) { ... }

// Async error — callback
fs.readFile("x", (err, data) => {
  if (err) return cb(err);    // explicit forwarding
  ...
});

// Async error — promise/async
async function load() {
  const data = await fs.promises.readFile("x"); // rejects
}
load().catch(handle);
```

Mixing styles is dangerous: throwing inside a callback won't be caught by an outer try/catch. With Express + async, you **must** call `next(err)` (or use a wrapper like `express-async-errors`) or async errors silently hang the request.

---

### Q30. How does Node.js `http` handle keep-alive?

HTTP/1.1 connections default to keep-alive — Node reuses the TCP socket for subsequent requests, avoiding handshake overhead.

Server side:
- `server.keepAliveTimeout` (default 5s in Node 19+) — how long to wait for the next request on an idle socket.
- `server.headersTimeout` should be **slightly greater** than `keepAliveTimeout` to avoid races.

Client side:
- `new http.Agent({ keepAlive: true, maxSockets: 50 })` — reuses sockets across requests. Critical for high-throughput outbound calls; otherwise you waste a TCP+TLS handshake per call.

---

### Q31. `--inspect` flag — attach Chrome DevTools.

```bash
node --inspect=0.0.0.0:9229 app.js
```

Then open `chrome://inspect` in Chrome → "Configure" → add `localhost:9229` → click "inspect" on the discovered target. You get the full DevTools (sources, breakpoints, profiler, memory).

For production debugging, never bind to `0.0.0.0` without auth — anyone with network access gets a remote shell. Use SSH tunnel: `ssh -L 9229:localhost:9229 prod-host`, then connect to `localhost:9229`.

`--inspect-brk` halts on the first line until the debugger attaches.

---

### Q32. `Promise.all` / `allSettled` / `race` / `any`.

| Method | Resolves | Rejects |
|---|---|---|
| `all([p1,p2])` | All fulfill → array of values | First rejection |
| `allSettled` | All settle → array of `{status,value/reason}` | Never |
| `race` | First settles (either way) | First rejects |
| `any` | First fulfills | All reject → `AggregateError` |

```js
// "best effort" — don't abort on one failure
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
results.forEach((r, i) => r.status === "fulfilled" ? use(r.value) : log(r.reason));
```

---

### Q33. Limit concurrency to 3 — 10 API calls.

Use a concurrency-limiting library or implement manually:

```js
import pLimit from "p-limit";
const limit = pLimit(3);

const tasks = urls.map(url => limit(() => fetch(url).then(r => r.json())));
const results = await Promise.all(tasks);
```

Manual (no deps):
```js
async function mapLimit(items, n, worker) {
  const results = new Array(items.length);
  let i = 0;
  async function run() {
    while (i < items.length) {
      const idx = i++;
      results[idx] = await worker(items[idx]);
    }
  }
  await Promise.all(Array.from({ length: n }, run));
  return results;
}
```

---

### Q34. `util.promisify` — use case.

Converts a Node-style callback API (`(err, value) => ...`) into a promise-returning function:

```js
const { promisify } = require("util");
const sleep = promisify(setTimeout);
await sleep(1000);

// Old DB drivers, legacy libs:
const queryAsync = promisify(connection.query.bind(connection));
const rows = await queryAsync("SELECT 1");
```

Most modern APIs already expose a promise variant (`fs.promises`, `dns.promises`), so you'll rarely need this on first-party Node modules.

---

### Q35. File uploads without a framework.

Without `multer`/`busboy`, you'd parse `multipart/form-data` manually — possible but error-prone. Here's the rough shape using `busboy` (lib, but Node-native style):

```js
const Busboy = require("busboy");
http.createServer((req, res) => {
  if (req.method === "POST") {
    const bb = Busboy({ headers: req.headers });
    bb.on("file", (name, file, info) => {
      file.pipe(fs.createWriteStream(`/tmp/${info.filename}`));
    });
    bb.on("close", () => res.end("ok"));
    req.pipe(bb);
  }
}).listen(3000);
```

Always **stream to disk** (or directly to S3/GCS); never buffer the entire upload in memory. Validate MIME type, enforce a max size (`limits`), and store with a random filename to prevent path traversal.

---

### Q36. `Buffer` — when to use over strings.

`Buffer` is a fixed-length sequence of bytes. Used for:
- **Binary protocols** — TCP, file headers, image bytes.
- **Encoding conversions** — `Buffer.from(text, "utf8").toString("base64")`.
- **Avoid string concatenation overhead** in hot loops dealing with raw bytes.
- **Crypto / hashing** — most crypto APIs return/expect Buffers.

```js
const buf = Buffer.from("hello", "utf8"); // <Buffer 68 65 6c 6c 6f>
buf.length;       // 5
buf.toString("hex"); // "68656c6c6f"
```

Strings are immutable UTF-16 — multibyte-aware. Buffers are raw bytes — encoding-agnostic.

---

### Q37. `path.join` vs string concat.

`path.join` normalises separators across platforms and resolves `..`/`.`:

```js
path.join("/users", "alex", "..", "bob");        // "/users/bob"
"/users" + "/" + "alex/../bob";                  // "/users/alex/../bob" (unnormalised, broken on Windows)
```

It also handles trailing slashes, double slashes, and platform-specific separators (`\` on Windows). Use `path.resolve` when you need an absolute path.

---

### Q38. Schedule a job daily at midnight.

**In-process (single instance):**
```js
const cron = require("node-cron");
cron.schedule("0 0 * * *", () => generateReport(), { timezone: "UTC" });
```

**Across multiple instances / production:** an in-process cron means *every instance* runs the job. Better:
- Use a job queue with cron support (`bullmq` repeatable jobs, `agenda`).
- Run a dedicated worker pod with a single replica.
- Use cloud-native: AWS EventBridge Schedule, GCP Cloud Scheduler → triggers an endpoint or Lambda.

Always make jobs idempotent so duplicate executions are safe.

---

### Q39. `throw` vs `return next(err)` in async Express middleware.

**Express 4:** async errors don't reach error middleware automatically. You **must** call `next(err)`:
```js
app.get("/x", async (req, res, next) => {
  try { res.json(await load()); }
  catch (err) { next(err); }
});
```

**Express 5 (current):** async rejections **are** forwarded — `throw` works the same as `next(err)`. Until you're on Express 5 (still beta in many setups), use `express-async-errors` or a wrapper:

```js
const asyncH = fn => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
app.get("/x", asyncH(async (req, res) => res.json(await load())));
```

`throw` in sync handlers is always caught and forwarded.

---

### Q40. Request timeouts to prevent hanging.

Set timeouts at multiple levels:

```js
// Per-request handler timeout
server.setTimeout(30_000);

// Per-route abort (Express)
app.use((req, res, next) => {
  req.setTimeout(15_000, () => res.status(503).end("timeout"));
  next();
});

// Outbound HTTP (avoid hanging external calls)
const ctl = new AbortController();
const t = setTimeout(() => ctl.abort(), 5000);
try { await fetch(url, { signal: ctl.signal }); } finally { clearTimeout(t); }
```

Also configure `keepAliveTimeout`, `headersTimeout`, and `requestTimeout` on the HTTP server. Without timeouts, a slow client can occupy a connection slot forever (Slowloris attack).

---

## Express.js

### Q41. Middleware — execution order.

Middleware functions execute in the order they're registered, like a pipeline. Each can:
- Pass to next middleware → `next()`.
- Pass to error handler → `next(err)`.
- End the response → `res.send()`, `res.json()`, `res.end()`.

```
Request → mw1 → mw2 → routeHandler → res.send → response
                ↓ error
         errorMiddleware → response
```

Order matters: `helmet()` before routes (security headers), `express.json()` before handlers that read `req.body`, error middleware last.

---

### Q42. `app.use()` vs `app.get()` vs `router.use()`.

- `app.use([path,] middleware)` — runs for **all methods** on paths matching the prefix. Used for logging, parsing, auth.
- `app.get(path, handler)` — runs only for GET on the exact path.
- `router.use(...)` — same as `app.use` but scoped to a `Router` instance (mounted via `app.use('/api', router)`).

```js
app.use(express.json());                     // global parser
app.use("/api", apiRouter);                  // mount router
app.get("/health", (req, res) => res.send("ok"));
```

---

### Q43. Async error not reaching error middleware — why and fix.

In Express 4, an async function that throws or rejects does **not** automatically call `next(err)`:

```js
// ❌ broken
app.get("/x", async (req, res) => {
  const data = await load(); // throws → unhandled rejection
  res.json(data);
});
```

**Fixes:**
1. `try/catch` and `next(err)` manually.
2. `npm i express-async-errors` and require it once — patches Express to forward.
3. Wrap handlers:
   ```js
   const asyncH = fn => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
   ```
4. Upgrade to Express 5 (handles async natively).

---

### Q44. Centralized error-handling middleware — 4 parameters.

Error middleware is identified by **arity of 4**:

```js
app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id }, "request failed");
  if (res.headersSent) return next(err); // delegate to default handler
  const status = err.status || 500;
  res.status(status).json({
    error: { code: err.code || "INTERNAL_ERROR", message: err.expose ? err.message : "Server error" }
  });
});
```

Register **last**, after all routes. Use a custom `HttpError` class to attach `status`, `code`, `expose` flag for sanitizing messages.

---

### Q45. `res.send()` vs `res.json()` vs `res.end()`.

| Method | Behavior |
|---|---|
| `res.send(body)` | Sets `Content-Type` based on body (string/buffer/object); calls `res.end`. |
| `res.json(obj)` | Sets `Content-Type: application/json`; `JSON.stringify(obj)`; calls `res.send`. |
| `res.end([data])` | Low-level: ends response without setting headers/serialising. For raw or empty responses. |

Use `res.json()` for APIs (explicit, predictable). `res.end()` for `204 No Content`, redirects, or already-piped streams.

---

### Q46. Simple request logger middleware.

```js
app.use((req, res, next) => {
  const start = process.hrtime.bigint();
  const id = req.headers["x-request-id"] || crypto.randomUUID();
  req.id = id;

  res.on("finish", () => {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    logger.info({
      id, method: req.method, url: req.originalUrl,
      status: res.statusCode, ms: ms.toFixed(1),
      ip: req.ip, ua: req.get("user-agent"),
    });
  });
  next();
});
```

Use `res.on("finish", ...)` not `res.on("close")` — close fires even on aborted connections without a response.

---

### Q47. Route hit but no response, client hangs — causes.

1. **Missing `res.send/json/end`** — common in branching code where some path forgets to respond.
2. **`await` on a never-resolving promise** — DB call hangs, no timeout set.
3. **Error swallowed** — async error not forwarded to error middleware; no response written.
4. **`next()` called without ending response** — passes to next mw which doesn't exist.
5. **Body parser failing silently** — request body pending, `req` stream never drained.
6. **Middleware loop** — two middlewares calling each other infinitely.

Debug by adding logs at each branch, set request timeouts to surface hangs (`req.setTimeout`).

---

### Q48. Validate and sanitize request bodies.

Use a schema validator — **Zod** (TypeScript-native) or **Joi**:

```js
import { z } from "zod";
const userSchema = z.object({
  email: z.string().email().toLowerCase().trim(),
  password: z.string().min(8).max(72),
  age: z.coerce.number().int().min(13).max(120).optional(),
});

app.post("/users", (req, res, next) => {
  const result = userSchema.safeParse(req.body);
  if (!result.success) return res.status(400).json({ errors: result.error.issues });
  req.validBody = result.data; // sanitized + coerced
  next();
});
```

Never trust client input. Validate types, lengths, ranges, formats. Sanitize anything rendered into HTML (XSS) or used in shell commands (injection).

---

### Q49. `express-validator` — email + password example.

```js
import { body, validationResult } from "express-validator";

app.post(
  "/login",
  body("email").isEmail().normalizeEmail(),
  body("password").isLength({ min: 8, max: 72 }).trim(),
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });
    next();
  },
  loginHandler
);
```

Personal preference: Zod's type inference is cleaner for TS projects; express-validator is fine for plain JS.

---

### Q50. Route-level vs app-level middleware.

```js
// App-level (every request)
app.use(express.json());
app.use(helmet());

// Route-level (only this route)
app.get("/admin", requireAuth, requireRole("admin"), adminHandler);

// Router-level
const apiRouter = express.Router();
apiRouter.use(authMiddleware);  // applies to all routes in this router
apiRouter.get("/me", meHandler);
app.use("/api", apiRouter);
```

---

### Q51. Purpose of `express.json()` and `express.urlencoded()`.

These body parsers populate `req.body`:
- `express.json()` — parses `Content-Type: application/json` request bodies.
- `express.urlencoded({ extended: true })` — parses `application/x-www-form-urlencoded` (HTML form submits). `extended: true` uses `qs` (supports nested objects).

Without them, `req.body` is `undefined` and your handlers can't read JSON/form data.

```js
app.use(express.json({ limit: "1mb" }));   // always set a sane limit
app.use(express.urlencoded({ extended: true, limit: "1mb" }));
```

---

### Q52. Multipart/form-data file uploads — `multer`.

```js
import multer from "multer";
const upload = multer({
  storage: multer.diskStorage({
    destination: "/tmp/uploads",
    filename: (req, file, cb) => cb(null, `${Date.now()}-${crypto.randomBytes(8).toString("hex")}`),
  }),
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    if (!["image/jpeg", "image/png"].includes(file.mimetype)) return cb(new Error("Invalid type"));
    cb(null, true);
  }
});

app.post("/avatar", upload.single("avatar"), (req, res) => {
  // req.file populated
  res.json({ filename: req.file.filename });
});
```

For production: stream directly to S3 (`multer-s3`) — never store locally on multi-instance deployments.

---

### Q53. CORS — configure for specific origins.

```js
import cors from "cors";
app.use(cors({
  origin: ["https://app.example.com", "https://admin.example.com"],
  credentials: true,                  // allow cookies
  methods: ["GET","POST","PUT","DELETE"],
  allowedHeaders: ["Content-Type","Authorization"],
  maxAge: 86400,                      // preflight cache 1 day
}));
```

For dynamic origin whitelisting:
```js
origin: (origin, cb) => cb(null, allowedSet.has(origin))
```

Never use `origin: "*"` with `credentials: true` — browsers reject it.

---

### Q54. Browser blocks `localhost:3000 → localhost:5000`.

This triggers **CORS preflight** failure. Browser sends `OPTIONS` request with `Origin: http://localhost:3000`; if your server doesn't reply with proper `Access-Control-Allow-Origin` header matching, the browser blocks the response.

**Fix:** mount `cors()` middleware before your routes, allow the dev origin:
```js
app.use(cors({ origin: "http://localhost:3000", credentials: true }));
```

Verify in Network tab: preflight `OPTIONS` should return 204 with the right headers.

---

### Q55. Rate limiting per IP.

```js
import rateLimit from "express-rate-limit";
import RedisStore from "rate-limit-redis";

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,        // 15 min
  max: 100,                         // 100 req per IP
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({ sendCommand: (...args) => redisClient.sendCommand(args) }),
});

app.use("/api", limiter);
```

Use Redis store across multiple instances — in-memory store only counts requests on the same process.

For sensitive routes (login, password reset), use stricter limits keyed on email + IP.

---

### Q56. helmet.js — 5 headers.

`helmet()` sets ~15 security headers by default:

| Header | Protects against |
|---|---|
| `Content-Security-Policy` | XSS, data injection — restricts source of scripts |
| `Strict-Transport-Security` (HSTS) | Protocol downgrade — forces HTTPS |
| `X-Frame-Options: DENY` | Clickjacking — prevents iframe embedding |
| `X-Content-Type-Options: nosniff` | MIME-sniffing attacks |
| `Referrer-Policy` | Information leakage via Referer header |
| `Cross-Origin-Opener-Policy` | Cross-origin attacks via window.opener |

```js
app.use(helmet({
  contentSecurityPolicy: { directives: { defaultSrc: ["'self'"] } }
}));
```

---

### Q57. Static files — `__dirname` vs `./`.

```js
// ❌ relative to CWD — breaks if started from different dir
app.use(express.static("./public"));

// ✅ absolute — relative to this file
app.use(express.static(path.join(__dirname, "public")));
```

`__dirname` is the absolute directory of the current module file; `./` resolves against `process.cwd()`. Always use `__dirname` (or `import.meta.url` in ESM) so behavior is independent of where the process is launched.

---

### Q58. Compression in Express.

```js
import compression from "compression";
app.use(compression({ threshold: 1024 })); // only > 1KB
```

Uses gzip/deflate by default. Cuts JSON response sizes by ~70%. Don't compress already-compressed responses (images, video).

In production, prefer **terminating compression at Nginx/CDN** — offloads CPU from Node.

---

### Q59. Structuring large Express apps (20+ routes, multiple domains).

```
src/
├── server.js              # entry — listens, graceful shutdown
├── app.js                 # express app + middleware
├── config/                # env loading, validation
├── middleware/            # auth, logging, error
├── modules/
│   ├── users/
│   │   ├── users.routes.js
│   │   ├── users.controller.js
│   │   ├── users.service.js
│   │   ├── users.repo.js
│   │   └── users.schema.js
│   ├── orders/
│   └── ...
├── lib/                   # shared utils
└── db/
```

- **Modules** group routes/controllers/services/repos by domain (vertical slice).
- **Routes** wire HTTP to controllers; controllers validate + call services; services hold business logic; repos handle DB.
- Don't mix layers — controllers shouldn't write SQL.

---

### Q60. `Router` — modularising routes.

```js
// users.routes.js
import { Router } from "express";
const router = Router();
router.get("/", listUsers);
router.post("/", createUser);
router.get("/:id", getUser);
export default router;

// app.js
import usersRouter from "./modules/users/users.routes.js";
app.use("/api/v1/users", usersRouter);
```

Routers support their own middleware, error handlers, and nested mounts. Essential for scaling beyond ~5 routes.

---

### Q61. Versioned APIs.

**URL versioning (most common):**
```js
app.use("/api/v1", v1Router);
app.use("/api/v2", v2Router);
```

**Header versioning:**
```js
app.use("/api", (req, res, next) => {
  const v = req.get("API-Version") || "1";
  return v === "2" ? v2Router(req, res, next) : v1Router(req, res, next);
});
```

**Pros & cons:** URL is discoverable & cache-friendly but pollutes URLs; header is cleaner but harder to test/debug. URL versioning wins in practice.

---

### Q62. `app.param()` — when useful.

Runs middleware whenever a specific param appears in any route:

```js
app.param("userId", async (req, res, next, id) => {
  const user = await db.users.findById(id);
  if (!user) return res.status(404).json({ error: "not found" });
  req.targetUser = user;
  next();
});

app.get("/users/:userId", (req, res) => res.json(req.targetUser));
app.put("/users/:userId", (req, res) => updateUser(req.targetUser, req.body));
```

Centralises lookup logic. Avoids duplicating "find user by id" in every handler.

---

### Q63. 404 for undefined routes.

```js
// Place AFTER all routes
app.use((req, res, next) => {
  res.status(404).json({ error: "Not Found", path: req.originalUrl });
});

// Then error handler
app.use(errorHandler);
```

For SPAs that need to serve `index.html` for unknown routes:
```js
app.get("*", (req, res) => res.sendFile(path.join(__dirname, "public/index.html")));
```

---

### Q64. HTML error pages returned to API clients.

Express's default error handler returns an HTML stack trace. Your custom error middleware must come **before** Express's default catches the error:

```js
app.use((err, req, res, next) => {
  if (req.accepts("json")) {
    return res.status(err.status || 500).json({ error: err.message });
  }
  res.status(500).send("Internal Server Error");
});
```

Also set `NODE_ENV=production` so default error pages don't leak stack traces. Ensure your error middleware is registered last and never throws itself.

---

### Q65. Response caching headers.

```js
// Static, public, cacheable for 1 day
res.set({
  "Cache-Control": "public, max-age=86400",
});

// Per-resource ETag (Express enables by default for static)
const etag = crypto.createHash("md5").update(JSON.stringify(data)).digest("hex");
res.set("ETag", `"${etag}"`);
if (req.get("If-None-Match") === `"${etag}"`) return res.status(304).end();
res.json(data);
```

- **`max-age`** — seconds to cache.
- **`public`** vs **`private`** — public allows CDN cache; private only browser.
- **`no-store`** — never cache (sensitive data).
- **`stale-while-revalidate`** — serve stale, refresh in background.

---

### Q66. `morgan` vs custom logger.

`morgan` is a tiny request-logging middleware:
```js
app.use(morgan("combined")); // Apache-style logs
```

**Custom logger advantages:**
- Structured JSON (queryable in ELK/Datadog).
- Adds request ID, user ID, custom fields.
- Integrates with your error logger.
- Logs response payload size, DB call counts.

morgan is fine for dev; custom (or `pino-http`) is preferred in production.

---

### Q67. Prevent stack disclosure.

```js
app.disable("x-powered-by");  // removes "X-Powered-By: Express"
```

Combine with:
- `NODE_ENV=production` → suppresses stack traces in error responses.
- helmet (sets generic headers).
- Custom error messages that don't include framework names.
- Generic 404/500 pages.

Attackers fingerprint by stack — fewer hints = harder reconnaissance.

---

### Q68. Request ID propagation.

```js
import { AsyncLocalStorage } from "async_hooks";
export const requestContext = new AsyncLocalStorage();

app.use((req, res, next) => {
  const id = req.get("x-request-id") || crypto.randomUUID();
  res.set("x-request-id", id);
  req.id = id;
  requestContext.run({ requestId: id }, next);
});

// In any module, deep in the call stack:
function log(msg) {
  const ctx = requestContext.getStore();
  logger.info({ requestId: ctx?.requestId }, msg);
}
```

When calling downstream services, forward the `x-request-id` header — gives end-to-end traceability.

---

### Q69. `next()` vs `next('route')`.

- `next()` — pass to **next middleware** in the chain.
- `next("route")` — skip remaining middleware for **this route** and try the next matching route (only inside route middleware).
- `next(err)` — jump to the first error-handling middleware.

```js
app.get("/x",
  (req, res, next) => req.user ? next() : next("route"),
  (req, res) => res.send("authenticated path")
);
app.get("/x", (req, res) => res.send("fallback"));
```

`next("route")` is rarely needed in practice — most teams just use `if/else`.

---

### Q70. Graceful restart without dropping connections.

Behind a load balancer:
1. New instance starts on a different port/container.
2. LB health-check picks it up.
3. Old instance receives `SIGTERM`.
4. Old instance returns 503 on `/healthz` (LB stops routing).
5. Old instance finishes in-flight requests, then `server.close()`.
6. Process exits cleanly.

For a single instance with PM2: `pm2 reload` (forks new worker, drains old). For Docker/k8s: rolling deploy with proper `preStop` lifecycle hook + `terminationGracePeriodSeconds` >> longest request.

```yaml
lifecycle:
  preStop:
    exec: { command: ["sh","-c","kill -SIGTERM 1; sleep 30"] }
terminationGracePeriodSeconds: 60
```

---

## Databases – SQL

### Q71. JOIN types — scenarios.

| JOIN | Returns |
|---|---|
| `INNER JOIN` | Rows that match in **both** tables |
| `LEFT JOIN` | All rows from left + matched from right (NULL where no match) |
| `RIGHT JOIN` | All rows from right + matched from left |
| `FULL OUTER JOIN` | All rows from both (NULL where no match) |

```sql
-- INNER: orders that have a valid user (skip orphans)
SELECT * FROM orders o INNER JOIN users u ON o.user_id = u.id;

-- LEFT: all users, with their orders (users without orders shown with NULL)
SELECT u.*, o.id AS order_id FROM users u LEFT JOIN orders o ON o.user_id = u.id;

-- FULL OUTER: reconciliation between two systems
SELECT a.id, b.id FROM systemA a FULL OUTER JOIN systemB b ON a.key = b.key;
```

---

### Q72. Users who have never placed an order.

```sql
-- Option 1: LEFT JOIN + IS NULL
SELECT u.* FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;

-- Option 2: NOT EXISTS (often fastest with proper index on orders.user_id)
SELECT u.* FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- Option 3: NOT IN (avoid if user_id can be NULL — semantics differ)
SELECT u.* FROM users u
WHERE u.id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

`NOT EXISTS` is generally the safest and most performant on PostgreSQL.

---

### Q73. Database index — how a B-tree works.

A **B-tree** is a balanced tree where each node holds sorted keys with pointers to child nodes. Lookup is O(log N).

```
        [50]
       /    \
   [20,30]   [70,90]
   / | \     /  |  \
[..rows..]
```

Properties:
- Self-balancing — depth stays roughly the same for all leaves.
- Sequential reads possible (leaves linked) → supports range queries (`BETWEEN`, `ORDER BY`).
- Inserts/deletes rebalance the tree.

PostgreSQL's default index type is B-tree. Other types: **GIN** (full-text, JSONB), **GiST** (geospatial), **Hash** (equality only, rarely used), **BRIN** (huge tables with natural ordering).

---

### Q74. 10M-row query taking 8s — diagnose and fix.

**Step-by-step:**
1. **Get the query plan**: `EXPLAIN ANALYZE <query>`.
2. **Look for `Seq Scan`** on large tables — usually means missing index.
3. **Check `rows` estimates** — if wildly off from actual, statistics are stale (`ANALYZE table`).
4. **Inspect filter selectivity** — a `WHERE status = 'active'` on a column where 99% are 'active' won't benefit from an index.
5. **Composite index ordering** — `(status, created_at)` works for filtering status + sorting time; reversed doesn't.
6. **Look at JOIN type** — nested loop on 10M rows is bad; hash/merge join much better.

**Fixes:**
- Add appropriate index (`CREATE INDEX CONCURRENTLY` in production).
- Rewrite to use `EXISTS` instead of `IN`.
- Add `LIMIT` if possible.
- Cache the result if not real-time.

---

### Q75. Clustered vs non-clustered index.

| Clustered | Non-clustered |
|---|---|
| Determines physical row order | Separate structure pointing to rows |
| One per table | Many per table |
| Faster range scans | Extra indirection (lookup row from index) |

In **MySQL InnoDB**, the primary key is the clustered index — data rows are stored in the leaf pages of the PK B-tree. Secondary indexes point to the PK, then the row is fetched via PK.

**PostgreSQL doesn't have clustered indexes** in the same sense — it uses heap storage; `CLUSTER table USING index` physically reorders rows once but doesn't maintain the order.

---

### Q76. When does an index hurt?

1. **High write workload** — every insert/update/delete must update all indexes. Too many indexes = slow writes.
2. **Low cardinality** — indexing a `gender` column (2 values) doesn't help; planner ignores it anyway.
3. **Small tables** — sequential scan is often faster (everything fits in one page).
4. **Frequently updated columns** — index churn, page splits, bloat.
5. **Wrong index order** — composite index `(a, b)` doesn't help queries filtering only by `b`.

Rule: index where the query pattern *needs* it, not preemptively. Use `pg_stat_user_indexes` to find unused indexes.

---

### Q77. `EXPLAIN ANALYZE` — what to look for.

`EXPLAIN ANALYZE` runs the query and reports actual stats. Read bottom-up.

**Red flags:**
- **`Seq Scan` on large table** — missing index or planner chose to skip it.
- **`Rows Removed by Filter` >> rows returned** — index isn't selective enough; consider partial/expression index.
- **Estimated vs actual rows wildly differ** — stale stats; run `ANALYZE`.
- **Nested Loop with high outer row count** — JOIN strategy poor; might need `SET enable_hashjoin = on` or rewrite.
- **Sort with `Memory: ... external merge`** — sort spilled to disk; increase `work_mem`.

```
Seq Scan on orders (cost=0..100000 rows=10000000 width=64)
  (actual time=0..7800 rows=10000000 loops=1)
  Filter: status = 'active'
  Rows Removed by Filter: 9000000
```

→ Missing index on `status` or partial index `WHERE status = 'active'`.

---

### Q78. Normalization — 1NF / 2NF / 3NF.

**1NF (Atomicity):** each column holds atomic values; no repeating groups.
```
❌ users (id, name, phones)   -- phones = "555-1, 555-2"
✅ users (id, name)           +  user_phones (user_id, phone)
```

**2NF (No partial dependency):** non-key columns depend on the **whole** composite key.
```
❌ order_items (order_id, product_id, product_name, qty)  -- product_name depends only on product_id
✅ order_items (order_id, product_id, qty)
   products (id, name)
```

**3NF (No transitive dependency):** non-key columns don't depend on other non-key columns.
```
❌ employees (id, name, dept_id, dept_name)  -- dept_name depends on dept_id
✅ employees (id, name, dept_id)
   departments (id, name)
```

Higher normal forms exist (BCNF, 4NF, 5NF) but 3NF is typically sufficient.

---

### Q79. When to denormalize.

**Reasons:**
- **Read-heavy workloads** — JOINs become a bottleneck.
- **Reporting/analytics tables** — pre-aggregated rollups.
- **Latency-critical paths** — store the user's display name on the post row to avoid joining users on every render.
- **Distributed databases** — cross-shard JOINs are slow.

**Costs:** data duplication, harder updates (must sync everywhere), risk of inconsistency. Mitigate with triggers, materialized views, or update fan-out via events.

Denormalize only when measurements show JOINs are the bottleneck — premature denormalization is harder to undo than premature normalization.

---

### Q80. Transactions — ACID.

A **transaction** is a unit of work executed atomically. ACID:

- **Atomicity** — all operations succeed or none. `ROLLBACK` undoes partial work.
- **Consistency** — DB moves from one valid state to another (constraints, triggers, FKs maintained).
- **Isolation** — concurrent transactions appear serial (level varies; see Q82).
- **Durability** — committed data persists across crashes (WAL/fsync).

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- both apply, or neither (on error → ROLLBACK)
```

---

### Q81. `COMMIT` vs `ROLLBACK`.

- **`COMMIT`** — finalises the transaction. Changes become visible to other transactions and durable (fsync'd to WAL).
- **`ROLLBACK`** — discards all changes made since `BEGIN`. The DB returns to the pre-transaction state.

In application code:
```js
const client = await pool.connect();
try {
  await client.query("BEGIN");
  await client.query("UPDATE ...");
  await client.query("COMMIT");
} catch (err) {
  await client.query("ROLLBACK");
  throw err;
} finally {
  client.release();
}
```

---

### Q82. Isolation levels.

From weakest (most concurrent) to strongest (most consistent):

| Level | Dirty Read | Non-repeatable | Phantom |
|---|---|---|---|
| Read Uncommitted | ✅ possible | ✅ | ✅ |
| Read Committed (PG default) | ❌ | ✅ | ✅ |
| Repeatable Read | ❌ | ❌ | ✅ (PG: ❌ due to MVCC) |
| Serializable | ❌ | ❌ | ❌ |

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Trade-off:** higher isolation → more locks/aborts → less throughput. PostgreSQL's MVCC makes Repeatable Read very effective without locking — often the sweet spot.

---

### Q83. Dirty / non-repeatable / phantom reads.

- **Dirty read** — read uncommitted data from another tx. (Imagine reading `balance=0` after a debit that gets rolled back.)
- **Non-repeatable read** — same row queried twice returns different values because another tx committed in between.
- **Phantom read** — same `WHERE` clause returns different rows because another tx inserted/deleted matching rows.

Prevention:
- Read Committed prevents dirty reads.
- Repeatable Read prevents non-repeatable reads.
- Serializable prevents phantoms (and forces conflict detection).

---

### Q84. Two transactions updating the same row.

In Read Committed (PG default), the second `UPDATE` **blocks** until the first commits or rolls back, then proceeds with the updated value (uses the latest committed version).

```
T1: BEGIN; UPDATE x SET v=v+1 WHERE id=1;  -- holds row lock
T2: BEGIN; UPDATE x SET v=v+1 WHERE id=1;  -- waits
T1: COMMIT;
T2:                                         -- now proceeds; sees T1's value
```

To enforce a consistent view: `SELECT ... FOR UPDATE` (pessimistic), or `Repeatable Read` + retry on conflict (optimistic).

---

### Q85. Deadlock — scenario and prevention.

```
T1: lock account A → tries to lock B
T2: lock account B → tries to lock A
=> deadlock
```

DB detects the cycle and aborts one transaction with `40P01` (PostgreSQL).

**Prevention:**
1. **Acquire locks in a consistent order** — e.g., always lock the lower account ID first.
2. **Use `SELECT ... FOR UPDATE NOWAIT`** — fail fast instead of waiting.
3. **Keep transactions short** — less time holding locks.
4. **Retry on deadlock** in app code with exponential backoff.

---

### Q86. Optimistic vs pessimistic locking.

**Pessimistic:** assume conflict; lock the row.
```sql
SELECT * FROM inventory WHERE id = 1 FOR UPDATE;
-- other tx blocks until commit
UPDATE inventory SET qty = qty - 1 WHERE id = 1;
```

**Optimistic:** assume no conflict; verify on commit using a version column.
```sql
SELECT qty, version FROM inventory WHERE id = 1;
-- read version=5
UPDATE inventory SET qty = qty - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- if 0 rows affected, someone else updated — retry
```

**Use optimistic** when contention is low (most rows aren't being fought over). **Pessimistic** when contention is guaranteed (single hot row, sequential workflow).

---

### Q87. Foreign key — what happens deleting parent with children.

By default (no action specified): `ON DELETE NO ACTION` — the DB raises a foreign key violation, transaction aborts.

```sql
ERROR: update or delete on table "users" violates foreign key constraint
       "orders_user_id_fkey" on table "orders"
```

You can configure behavior with `ON DELETE` clauses (see Q88).

---

### Q88. `CASCADE` / `SET NULL` / `RESTRICT`.

```sql
CREATE TABLE orders (
  user_id INT REFERENCES users(id) ON DELETE CASCADE     -- delete child rows too
);
-- or ON DELETE SET NULL  -- set the FK to NULL
-- or ON DELETE RESTRICT  -- error if children exist (default)
```

**Use cases:**
- `CASCADE`: ownership relationship — delete user → delete their posts.
- `SET NULL`: optional relationship — delete category → posts stay but become uncategorized.
- `RESTRICT`: protect against accidents — refuse to delete a customer with active orders.

Cascade can cause unexpected data loss. Prefer soft deletes for important data.

---

### Q89. Stored procedures — pros and cons.

```sql
CREATE PROCEDURE transfer(from_id INT, to_id INT, amt NUMERIC)
LANGUAGE plpgsql AS $$
BEGIN
  UPDATE accounts SET balance = balance - amt WHERE id = from_id;
  UPDATE accounts SET balance = balance + amt WHERE id = to_id;
END;
$$;
CALL transfer(1, 2, 100);
```

**Pros:** atomic encapsulation, reduced network round-trips, single source of truth for complex logic.

**Cons:**
- Hard to version-control alongside app code.
- Hard to unit-test.
- DB-specific (PL/pgSQL ≠ T-SQL ≠ PL/SQL).
- Couples business logic to DB — harder to migrate.
- Limited tooling (debuggers, observability).

Modern preference: keep business logic in the app; use SQL views/functions only for performance-critical aggregations.

---

### Q90. View vs materialized view.

- **View** — a saved query; executed every time it's read. No storage.
- **Materialized view** — query results stored on disk; refreshed manually or on a schedule.

```sql
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;

CREATE MATERIALIZED VIEW daily_revenue AS
SELECT date_trunc('day', created_at) AS day, SUM(total) AS revenue
FROM orders GROUP BY 1;

REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;  -- non-blocking
```

Use materialized views for expensive aggregations queried often (dashboards). Refresh in a background job.

---

### Q91. Recursive CTE — employees under manager.

```sql
WITH RECURSIVE subordinates AS (
  -- base case: direct reports
  SELECT id, name, manager_id, 1 AS depth
  FROM employees WHERE manager_id = 100

  UNION ALL

  -- recursive: reports of reports
  SELECT e.id, e.name, e.manager_id, s.depth + 1
  FROM employees e JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

Recursive CTEs are perfect for tree/graph traversal: org charts, category trees, comment threads. Always include a termination condition (here, employees with no further reports stop adding rows).

---

### Q92. `WHERE` vs `HAVING`.

- `WHERE` — filters **before** aggregation, on individual rows.
- `HAVING` — filters **after** aggregation, on grouped results.

```sql
-- Customers with total spending > 1000
SELECT customer_id, SUM(total) AS spent
FROM orders
WHERE status = 'completed'      -- filter rows first
GROUP BY customer_id
HAVING SUM(total) > 1000;       -- filter groups
```

You can't use aggregate functions in `WHERE` (the rows aren't grouped yet).

---

### Q93. Window function — top 3 per customer.

```sql
SELECT * FROM (
  SELECT
    o.*,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total DESC) AS rn
  FROM orders o
) ranked
WHERE rn <= 3;
```

`ROW_NUMBER()` assigns 1,2,3,... per partition. Variants:
- `RANK()` — ties get same rank, next skips (1,2,2,4).
- `DENSE_RANK()` — ties same, no skip (1,2,2,3).
- `LAG()/LEAD()` — previous/next row values.

Window functions are powerful for analytics without self-joins.

---

### Q94. `GROUP BY ROLLUP`.

`ROLLUP` adds subtotal/grand-total rows:

```sql
SELECT region, product, SUM(qty)
FROM sales
GROUP BY ROLLUP(region, product);
```

Result includes:
- Row per (region, product).
- Subtotal per region (product = NULL).
- Grand total (both NULL).

Useful for financial reports without needing `UNION ALL` of multiple aggregations. Related: `CUBE` (all combinations), `GROUPING SETS` (custom).

---

### Q95. N+1 query problem — detect and fix.

**The problem:** loading a list, then making one more query per item:
```js
const posts = await db.posts.find(); // 1 query
for (const p of posts) {
  p.author = await db.users.findById(p.userId); // N more queries
}
```

For 100 posts: 101 queries.

**Detect:**
- Log SQL with timing; spot identical queries repeated.
- ORM debug mode (`Sequelize: logging: console.log`).
- Use APM tracing — flame graph shows the loop.

**Fix:**
- Eager load with JOIN or `IN`:
  ```sql
  SELECT * FROM posts;
  SELECT * FROM users WHERE id IN (...);
  ```
- ORM-level: Sequelize `include`, Mongoose `populate` with care, Prisma `include`.
- DataLoader pattern (batches calls per request).

---

### Q96. ORM generating 101 queries.

Same as Q95. Concrete examples:

**Sequelize:**
```js
const posts = await Post.findAll({
  include: { model: User, as: "author" }   // SQL JOIN, 1 query
});
```

**Prisma:**
```js
const posts = await prisma.post.findMany({ include: { author: true } });
```

**Mongoose (separate queries, but batched):**
```js
const posts = await Post.find().populate("author"); // 2 queries, not N+1
```

**DataLoader** (works in GraphQL too):
```js
const userLoader = new DataLoader(ids => User.find({ _id: { $in: ids } }));
// .load() batches and dedupes
```

---

### Q97. Connection pooling — pool exhausted.

**What it is:** a pool of pre-opened DB connections reused across requests, avoiding TCP+auth overhead per query.

```js
const { Pool } = require("pg");
const pool = new Pool({ max: 20, idleTimeoutMillis: 30000 });
```

**When all 20 are busy:** new acquires wait in a queue. If queue exceeds threshold or `acquireTimeout` fires, requests fail.

**Causes of exhaustion:**
1. Long-running queries holding connections.
2. Transactions not committed/rolled back (connection leak).
3. Pool size too small for traffic.
4. Per-request DB calls > pool size × pool count.

**Fix:** raise pool size to a reasonable ratio (PG max ~100 connections total across all clients), use PgBouncer for connection pooling at the DB tier, profile long queries.

---

### Q98. PostgreSQL "too many connections".

PostgreSQL has a hard limit (`max_connections`, default 100). Each connection costs ~10MB RAM. Causes:

1. **No app-level pooling** — each request opens a fresh connection.
2. **Pool too large × many app instances** — 10 instances × 20 pool = 200 connections.
3. **Idle transactions** — connections stuck in `idle in transaction`.

**Solutions:**
- Use **PgBouncer** (transaction pooling mode) — 1000s of client connections multiplexed onto ~50 server connections.
- Reduce per-app pool size; tune for actual concurrency.
- Set `idle_in_transaction_session_timeout` to kill leaks.
- Monitor `pg_stat_activity`.

---

### Q99. `pg_stat_activity` — find/kill long queries.

```sql
-- Find queries running > 30s
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '30 seconds'
ORDER BY duration DESC;

-- Cancel a query (gentle)
SELECT pg_cancel_backend(12345);

-- Force-kill (if cancel doesn't work)
SELECT pg_terminate_backend(12345);
```

Common stuck states: `idle in transaction` (leak), `active` (long query), `waiting` (blocked by lock). Check `pg_locks` for blockers.

---

### Q100. Sharding — trade-offs.

**Sharding** splits data across multiple DB servers, partitioned by a key (e.g., user_id mod N, geographic region).

**Pros:**
- Horizontal scale beyond what one server can handle.
- Localised performance — queries hit only one shard.

**Cons:**
- **Cross-shard queries/joins are hard** — must scatter-gather.
- **Re-sharding** is operationally painful (rebalancing data).
- **Transactions** across shards require 2PC or saga.
- **Schema migrations** must run on every shard.
- **Hotspots** if shard key is unbalanced.

Prefer read replicas, vertical scaling, and good indexing first. Shard only when you've exhausted single-server scale.

---

### Q101. Vertical vs horizontal scaling.

| Vertical (scale up) | Horizontal (scale out) |
|---|---|
| Bigger box (more CPU/RAM/SSD) | More boxes |
| Simple, no app changes | App must support distribution |
| Hardware/cloud ceiling | Nearly unlimited |
| Single point of failure | Built-in redundancy |
| Restart for upgrade | Rolling updates |

For DBs: vertical scaling is usually first — modern cloud instances go to 1TB+ RAM. Horizontal (replicas, sharding) when read load saturates or storage exceeds one machine.

---

### Q102. Sync vs async replication.

**Synchronous replication** — primary waits for replica(s) to confirm before acknowledging the commit.
- ✅ Zero data loss on failover.
- ❌ Latency = max(primary, replica). Network blip = write timeout.

**Asynchronous replication** — primary commits immediately; replica catches up later.
- ✅ Low latency.
- ❌ Potential data loss on primary failure (replication lag).

Most setups use async with monitoring on lag. Sync for critical financial systems or quorum-based (`synchronous_commit = remote_apply` for one replica).

---

### Q103. Read replicas — routing queries.

```js
const writeDb = new Pool({ host: "primary.db" });
const readDb  = new Pool({ host: "replica.db" });

function db(req) {
  return req.method === "GET" ? readDb : writeDb;
}

// Or per-query:
const users = await readDb.query("SELECT * FROM users");
await writeDb.query("UPDATE users SET name=$1 WHERE id=$2", [name, id]);
```

**Pitfall: replication lag.** A user POSTs an update, then GETs back — read replica may not have the change yet. Mitigations:
- "Read your writes" — after a mutation, force the next read to primary for that user.
- Use `pg_last_xact_replay_lsn()` to wait for lag to close.
- Read-after-write within same request: always use primary.

---

### Q104. Zero-downtime schema migration.

**Golden rule: expand → migrate → contract.**

1. **Expand**: add new column/table; deploy code that writes to both old and new.
2. **Backfill**: copy existing data into new structure (in batches).
3. **Migrate read path**: switch app to read from new column.
4. **Contract**: stop writing old, drop old column in a later release.

Use `CREATE INDEX CONCURRENTLY` (no table lock). Avoid `ALTER TABLE ... ALTER COLUMN TYPE` that rewrites the table.

Tools: `gh-ost`, `pg-osc` for MySQL/PG online schema changes.

---

### Q105. Add NOT NULL column to 50M-row table.

**Naïve approach (will lock for minutes):**
```sql
ALTER TABLE big ADD COLUMN status text NOT NULL DEFAULT 'pending';
```

**Safe approach:**
```sql
-- 1. Add nullable column with default (PG 11+: instant, doesn't rewrite)
ALTER TABLE big ADD COLUMN status text DEFAULT 'pending';

-- 2. Backfill in batches (separate transactions)
UPDATE big SET status = 'pending' WHERE id BETWEEN 0 AND 100000;
-- repeat in batches; pause between to avoid bloat

-- 3. Add NOT NULL constraint (still locks briefly, but data is already set)
ALTER TABLE big ALTER COLUMN status SET NOT NULL;
```

PostgreSQL 11+ added fast-path for default-with-DEFAULT — column added without rewrite. Verify your version. For older versions, use the expand/backfill/contract pattern.

---

### Q106. `TRUNCATE` vs `DELETE` vs `DROP`.

| | DELETE | TRUNCATE | DROP |
|---|---|---|---|
| Removes | matching rows | all rows | table itself |
| WHERE | ✅ | ❌ | ❌ |
| Triggers fire | ✅ | ❌ (in MySQL); ✅ (in PG with FOR EACH STATEMENT) | n/a |
| Resets sequences | ❌ | ✅ (PG: `RESTART IDENTITY`) | n/a |
| Speed on large tables | Slow (per row) | Fast (drops pages) | Fastest |
| Can rollback | ✅ | ✅ (PG, in tx) | ✅ (in tx) |

```sql
DELETE FROM logs WHERE created_at < now() - interval '30 days';
TRUNCATE TABLE tmp_import RESTART IDENTITY;
DROP TABLE old_table;
```

---

### Q107. PostgreSQL sequence for auto-increment.

```sql
CREATE SEQUENCE users_id_seq;
CREATE TABLE users (
  id INT DEFAULT nextval('users_id_seq') PRIMARY KEY,
  ...
);

-- Or shorthand:
CREATE TABLE users (id SERIAL PRIMARY KEY);  -- creates the sequence implicitly

-- Modern (PG 10+): GENERATED — recommended
CREATE TABLE users (id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY);
```

`GENERATED ALWAYS` prevents accidental manual inserts into the id column. Sequences are not transactional — gaps can occur on rollback. That's by design (no contention).

---

### Q108. `UPSERT` — `ON CONFLICT`.

```sql
INSERT INTO users (id, email, login_count)
VALUES (1, 'a@x.com', 1)
ON CONFLICT (id) DO UPDATE
  SET login_count = users.login_count + 1,
      last_login = now()
RETURNING *;
```

**Variants:**
- `DO NOTHING` — silently skip duplicates.
- `DO UPDATE` — update on conflict.
- `EXCLUDED.col` references the proposed insert value.

Use for idempotent inserts, counters, dedupe. Requires a unique constraint on the conflict columns.

---

### Q109. PostgreSQL full-text search vs Elasticsearch.

**PostgreSQL FTS:**
```sql
CREATE INDEX idx_docs_fts ON docs USING GIN (to_tsvector('english', body));
SELECT * FROM docs WHERE to_tsvector('english', body) @@ to_tsquery('database & performance');
```

**When PG FTS suffices:** small-to-medium corpus (< 10M docs), basic relevance ranking, want to avoid running another service, transactional consistency with main data.

**When Elasticsearch wins:** > 100M docs, complex relevance tuning, fuzzy/typo matching, faceted search, aggregations, multi-language analysers, distributed search at scale.

Start with PG FTS; migrate to ES when you outgrow it.

---

### Q110. Backup and restore — `pg_dump`.

```bash
# Backup (custom format — compressed, supports parallel restore)
pg_dump -h prod-db -U postgres -F c -d myapp -f myapp.dump

# Backup with parallel jobs
pg_dump -h prod-db -F d -j 4 -d myapp -f myapp-dump-dir

# Restore
pg_restore -h staging-db -U postgres -d myapp_new myapp.dump

# Schema only
pg_dump --schema-only myapp > schema.sql

# Data only
pg_dump --data-only --inserts myapp > data.sql
```

For **point-in-time recovery (PITR)**: continuous WAL archiving + base backup. Tools: `pgBackRest`, `barman`, AWS RDS automated backups.

**Test restores regularly** — a backup you've never restored isn't a backup.

---

## Databases – MongoDB / NoSQL

### Q111. Embedding vs referencing — when each.

**Embed (denormalize):**
```js
{ _id, title, comments: [{ author, text, createdAt }, ...] }
```
- ✅ One query reads everything.
- ✅ Atomic updates on the whole document.
- ❌ Document size limit (16MB).
- ❌ Hard to query/index nested arrays selectively.

**Reference (normalize):**
```js
// posts collection
{ _id, title, authorId }
// users collection
{ _id, name }
```
- ✅ No size limit; data shared across documents.
- ✅ Independent updates.
- ❌ Need `$lookup` or app-level joins (extra queries).

**Rule of thumb:**
- Embed for "contained within" relationships (post + comments where comments < ~100).
- Reference for many-to-many, large unbounded arrays, or shared data (users).

---

### Q112. Blog schema — Posts/Comments/Users.

**Recommended:**
```js
// users — reference, shared
{ _id: ObjectId, name, email, avatarUrl }

// posts — embed author snapshot (denormalize displayName for read perf), reference authorId
{
  _id: ObjectId,
  title, body, tags: [],
  authorId: ObjectId("..."),
  authorName: "Alex",      // snapshot for fast list rendering
  createdAt
}

// comments — separate collection if unbounded; embed if bounded
{
  _id, postId: ObjectId("..."),
  authorId, authorName,
  body, createdAt
}
```

**Justification:** comments can grow unbounded (10k comments on a viral post would exceed 16MB), so reference. Authors are shared across posts → reference. Snapshot the display name on post for read speed; if a user renames, update via background job.

---

### Q113. Aggregation pipeline — stages.

```js
db.orders.aggregate([
  { $match: { status: "completed" } },                       // filter rows
  { $unwind: "$items" },                                     // expand array
  { $group: {                                                 // aggregate
      _id: "$items.category",
      revenue: { $sum: { $multiply: ["$items.qty","$items.price"] } }
  }},
  { $lookup: {                                               // join
      from: "categories", localField: "_id",
      foreignField: "_id", as: "category"
  }},
  { $project: { _id: 0, category: { $first: "$category.name" }, revenue: 1 }},
  { $sort: { revenue: -1 } }
]);
```

| Stage | Purpose |
|---|---|
| `$match` | Filter (like WHERE) |
| `$group` | Aggregate (like GROUP BY) |
| `$project` | Reshape output (like SELECT) |
| `$lookup` | Join with another collection |
| `$unwind` | Flatten an array field into multiple docs |
| `$sort`, `$skip`, `$limit` | Pagination |

Put `$match` and `$project` early to reduce data volume.

---

### Q114. Revenue per category from orders.

```js
db.orders.aggregate([
  { $match: { status: "completed", createdAt: { $gte: ISODate("2026-01-01") } } },
  { $unwind: "$items" },
  { $group: {
      _id: "$items.categoryId",
      revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } },
      orderCount: { $sum: 1 }
  }},
  { $sort: { revenue: -1 } },
  { $limit: 10 }
]);
```

Add an index on `{ status: 1, createdAt: 1 }` to make the `$match` fast.

---

### Q115. `$lookup` vs SQL JOIN — limitations.

```js
{ $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "user" }}
```

**Differences:**
- `$lookup` returns the matched docs as an **array** field (always), not flat columns.
- Only **left outer join**; no right or full outer.
- Performance worse than SQL JOIN on large data — Mongo isn't designed as a relational DB.
- Requires both collections in the same DB.
- Index on `foreignField` is critical.

**Limitations:**
- No cross-shard lookups in older versions (now supported with caveats).
- Can result in large documents if "as" array grows.
- Pipeline lookups (the `pipeline` variant) more flexible but slower.

For frequent multi-collection queries → reconsider schema (embed, denormalize).

---

### Q116. MongoDB index types.

| Type | Use |
|---|---|
| Single field | Most common — equality, range |
| Compound | Multiple fields, prefix matching |
| Multikey | Indexes array elements automatically |
| Text | Full-text search |
| Geospatial (2dsphere) | Location queries |
| Hashed | Sharding key with even distribution |
| Wildcard | Unknown field names (anti-pattern usually) |
| TTL | Auto-delete after expiry (see Q119) |

```js
db.users.createIndex({ email: 1 }, { unique: true });
db.posts.createIndex({ authorId: 1, createdAt: -1 });
db.places.createIndex({ location: "2dsphere" });
```

---

### Q117. Compound index — field order.

```js
db.orders.createIndex({ status: 1, createdAt: -1 });
```

**Index prefix rule:** the index supports queries on `{status}` and `{status, createdAt}`, but **not** `{createdAt}` alone.

```js
db.orders.find({ status: "open" }).sort({ createdAt: -1 });            // ✅ uses index
db.orders.find({ status: "open", createdAt: { $gt: d } });             // ✅
db.orders.find({ createdAt: { $gt: d } });                              // ❌ does NOT use this index
```

**Equality first, then sort, then range** — common heuristic. Put the most selective equality fields first.

---

### Q118. Sparse index — when.

A sparse index only indexes documents that **have the field**:

```js
db.users.createIndex({ phone: 1 }, { sparse: true, unique: true });
```

Without sparse, all docs missing `phone` are indexed as `null` — a unique index would allow only one such doc.

Use when:
- Field is optional and you want a unique constraint on docs that *do* have it.
- Reduce index size for rarely-populated fields.

In modern MongoDB, **partial indexes** are usually preferred:
```js
db.users.createIndex({ phone: 1 }, {
  unique: true,
  partialFilterExpression: { phone: { $exists: true } }
});
```

---

### Q119. TTL index — use case.

Auto-expires documents after a duration based on a date field:

```js
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 }); // 1 hour
```

Mongo runs a background job (~every 60s) deleting expired docs.

**Use cases:**
- Session storage.
- Rate-limit tracking entries.
- Verification codes (OTPs).
- Logs/events with bounded retention.

Not for precise expiry — there's a delay; use a queue if exact timing matters.

---

### Q120. Slow query on 5M docs — diagnose.

1. Run `explain("executionStats")` — check `COLLSCAN` vs `IXSCAN`.
2. Inspect `totalDocsExamined` vs `nReturned`. If examined >> returned → missing index.
3. Check `executionTimeMillis`.
4. Verify index is used (`winningPlan.inputStage.indexName`).
5. Use `db.currentOp()` to find ongoing slow queries.
6. Enable slow query log: `db.setProfilingLevel(1, { slowms: 100 })`.

**Common fixes:** create matching index, narrow `$match`, project only needed fields (`{ $project: { x: 1 } }`), avoid unindexed `$sort` on large result sets.

---

### Q121. `explain("executionStats")` — COLLSCAN vs IXSCAN.

```js
db.users.find({ email: "x" }).explain("executionStats");
```

- **COLLSCAN** — collection scan; reads every document. Bad on large collections unless intentional.
- **IXSCAN** — index scan; reads index then fetches matching docs. Good.
- **FETCH** — fetches the full doc after index match. Avoid by using **covered queries** (index includes all queried fields).

```js
// Covered: returns email only, served entirely from the index
db.users.createIndex({ email: 1, name: 1 });
db.users.find({ email: "x" }, { _id: 0, email: 1, name: 1 }).explain();
```

Key metrics: `totalKeysExamined`, `totalDocsExamined`, `nReturned`. Ideal: `totalDocsExamined ≈ nReturned`.

---

### Q122. `find()` / `findOne()` / `findById()` in Mongoose.

- `find(filter)` — returns an array (could be empty).
- `findOne(filter)` — returns one doc or `null`.
- `findById(id)` — equivalent to `findOne({ _id: id })`; auto-casts string to ObjectId.

```js
const all = await User.find({ active: true });
const one = await User.findOne({ email });
const byId = await User.findById(req.params.id);
```

Returns Mongoose documents by default (with methods). Use `.lean()` for plain JS objects (faster, no methods):
```js
const fast = await User.find().lean();
```

---

### Q123. Mongoose middleware — pre/post `save` hooks.

```js
schema.pre("save", async function (next) {
  if (!this.isModified("password")) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

schema.post("save", function (doc, next) {
  publishEvent("user.created", { id: doc._id });
  next();
});
```

Hooks fire on:
- `save`, `validate`, `remove`, `init`.
- Query: `find`, `findOne`, `findOneAndUpdate`, `update*`.

**Gotcha:** `findOneAndUpdate` does NOT trigger `save` hook (it's a query, not a save). If you rely on password hashing in pre-save, use `.save()` not `.findOneAndUpdate()`.

---

### Q124. `populate()` — performance implications.

```js
const posts = await Post.find().populate("authorId");
```

Behind the scenes: 1 query for posts, then 1 query for all needed authors via `IN`. Not N+1 — but still 2 DB roundtrips.

**Costs:**
- Extra round-trip per populated path.
- Returns full author documents (use `.populate("authorId", "name avatar")` to project).
- Cascading populate (`populate({ path: "comments", populate: "author" })`) compounds.

**Alternatives:**
- Denormalize the few fields you need (snapshot author name on the post).
- Use aggregation `$lookup` for complex joins.
- DataLoader for batching across a request.

---

### Q125. `useNewUrlParser` and `useUnifiedTopology`.

These were Mongoose options to opt-in to:
- **`useNewUrlParser`** — new MongoDB connection-string parser.
- **`useUnifiedTopology`** — new topology engine (replaced old monitoring code).

**As of Mongoose 6+, both are defaults and the options are deprecated** — you no longer need to pass them. If you see them in old code, it's safe to remove on Mongoose 6+.

---

### Q126. MongoDB connection failures — graceful handling.

```js
mongoose.connection.on("disconnected", () => logger.warn("mongo disconnected"));
mongoose.connection.on("reconnected", () => logger.info("mongo reconnected"));
mongoose.connection.on("error", (err) => logger.error({ err }, "mongo error"));

await mongoose.connect(uri, {
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  maxPoolSize: 50,
  retryWrites: true,
});
```

The driver auto-reconnects. Don't wrap every query in retry — driver handles transient failures. For long downtime, fail fast and let the orchestrator restart you.

Add a `/healthz` that returns 503 if `mongoose.connection.readyState !== 1` so LBs stop routing.

---

### Q127. MongoDB transactions — requirements.

Multi-document transactions require:
- **MongoDB 4.0+** (replica sets) or 4.2+ for sharded.
- Storage engine: WiredTiger.
- Not on standalone single-node — must be a replica set (even single-node replica set).

```js
const session = await mongoose.startSession();
session.startTransaction();
try {
  await Account.updateOne({ _id: a }, { $inc: { bal: -100 } }, { session });
  await Account.updateOne({ _id: b }, { $inc: { bal: 100 } }, { session });
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

Transactions have overhead (~5x slower writes). Design schema so most ops are single-document (which are atomic by default).

---

### Q128. Atomic inventory deduct + create order.

**Option 1 — single-document atomic operators (preferred):**
```js
const updated = await Product.findOneAndUpdate(
  { _id: productId, stock: { $gte: qty } },
  { $inc: { stock: -qty } },
  { new: true }
);
if (!updated) throw new Error("Out of stock");
await Order.create({ productId, qty, userId });
```

If the second insert fails, you have an inconsistent state — needs compensation or a transaction.

**Option 2 — transaction:**
```js
const s = await mongoose.startSession();
await s.withTransaction(async () => {
  const u = await Product.findOneAndUpdate(
    { _id: productId, stock: { $gte: qty } },
    { $inc: { stock: -qty } }, { session: s, new: true });
  if (!u) throw new Error("Out of stock");
  await Order.create([{ productId, qty }], { session: s });
});
```

`withTransaction` auto-retries on transient errors. Use this when you must update multiple docs atomically.

---

### Q129. `updateOne` / `findOneAndUpdate` / `findByIdAndUpdate`.

| | Returns | Use |
|---|---|---|
| `updateOne(filter, update)` | `{ matchedCount, modifiedCount }` | Fire-and-forget update |
| `findOneAndUpdate(filter, update, opts)` | The doc (old by default, new with `{new:true}`) | Get the updated doc back |
| `findByIdAndUpdate(id, update, opts)` | Same as above | Shorthand by _id |

```js
const updated = await User.findByIdAndUpdate(id, { name: "X" }, { new: true, runValidators: true });
```

`findOneAndUpdate` does the find + update **atomically** at the DB level — important for counters and race-free updates.

---

### Q130. `{ new: true }` in `findOneAndUpdate`.

By default, `findOneAndUpdate` returns the document **as it was before** the update (the "original"). `{ new: true }` returns the **post-update** version.

```js
// Default (old behaviour)
const old = await User.findByIdAndUpdate(id, { name: "X" });
old.name; // previous name

// Modern (usually what you want)
const updated = await User.findByIdAndUpdate(id, { name: "X" }, { new: true });
updated.name; // "X"
```

Also add `{ runValidators: true }` to apply schema validators on update (off by default!).

---

### Q131. `$set` / `$inc` / `$push` / `$pull`.

```js
// $set — set fields
await User.updateOne({ _id }, { $set: { name: "Alex", "address.city": "NYC" } });

// $inc — atomic counter (avoids read-modify-write)
await Post.updateOne({ _id }, { $inc: { views: 1 } });

// $push — append to array
await Post.updateOne({ _id }, { $push: { tags: "node" } });
// With $each + $slice (cap array length)
await Post.updateOne({ _id }, { $push: { recent: { $each: [item], $slice: -10 } } });

// $pull — remove from array
await User.updateOne({ _id }, { $pull: { roles: "admin" } });
```

These operators are **atomic at the document level** — safe under concurrency without explicit locking.

---

### Q132. Soft delete in MongoDB.

Add a `deletedAt` field instead of removing the document:

```js
const schema = new Schema({ /*...*/, deletedAt: { type: Date, default: null } });

// Soft delete
await User.updateOne({ _id }, { $set: { deletedAt: new Date() } });

// Default queries exclude deleted
schema.pre(/^find/, function () { this.where({ deletedAt: null }); });

// Hard delete after N days (TTL index)
schema.index({ deletedAt: 1 }, { expireAfterSeconds: 30 * 86400, partialFilterExpression: { deletedAt: { $ne: null } } });
```

Pros: undo possible, audit trail, FK integrity preserved.
Cons: bloats collection, must filter everywhere, may interfere with unique constraints.

---

### Q133. Capped collection — when useful.

A fixed-size collection that auto-evicts oldest documents when full:

```js
db.createCollection("logs", { capped: true, size: 100_000_000, max: 1_000_000 });
```

Properties:
- Insertion order preserved.
- Fast inserts (no fragmentation).
- Can't delete individual docs; can't grow indefinitely.
- Supports tailable cursors (real-time tail like `tail -f`).

**Use cases:** real-time logs, ring buffer, oplog itself is a capped collection. For most cases, prefer **TTL index** for time-based retention — more flexible.

---

### Q134. Oplog — role in replication.

The **oplog** (operations log) is a special capped collection (`local.oplog.rs`) that records every write on the primary. Secondaries tail the oplog and apply ops in order, eventually-consistently.

```
Primary write → write to data → append op to oplog
Secondary → tail oplog → apply op locally
```

Other uses:
- **Change Data Capture (CDC)** — apps tail the oplog to react to changes (Mongo Connector, Debezium).
- **Change Streams** — official API built on the oplog.
- **PITR / rollback** — replay oplog to a point in time.

Oplog size matters: too small → secondaries can fall too far behind and need a full resync. Sized 5% of disk by default; tune via `replSetResizeOplog`.

---

### Q135. MongoDB Atlas vs self-hosted.

Atlas (managed by MongoDB Inc.) extras:
- **Auto-scaling** storage and compute.
- **Backups** with PITR.
- **Encryption at rest + in transit** by default.
- **Network peering / private endpoints** (AWS PrivateLink, Azure, GCP).
- **Atlas Search** (Lucene-based, replaces standalone Mongo text index).
- **Atlas Charts** for visualization.
- **Online schema migration**, performance advisor, query insights.
- **Multi-region replicas** for low-latency reads.

Self-hosted is cheaper for tiny workloads but operationally expensive — backups, monitoring, upgrades, sharding rebalancing on you. Atlas is the default in most production setups today.

---

### Q136. `ObjectId` vs custom string `_id`.

**ObjectId (12 bytes):**
- Auto-generated, includes timestamp (first 4 bytes), machine id, process id, counter.
- Sortable roughly by creation time.
- Mongo native, optimal storage/index size.

**Custom string `_id`** (e.g., UUIDv4 or natural key):
- Human readable.
- Better when ID flows across systems (URL-safe UUIDs).
- More storage (UUID = 36 chars vs 24 hex chars for ObjectId).
- Random UUIDs cause index fragmentation; ULID/UUIDv7 is sortable and friendlier to B-tree inserts.

For most apps: stick with ObjectId. Use UUIDv7 or ULID if you need cross-system unique IDs that are sortable.

---

### Q137. Efficient pagination on millions of docs.

**Offset pagination (`skip`/`limit`)** — bad for deep pages:
```js
db.posts.find().sort({ _id: -1 }).skip(100000).limit(20);   // scans 100020 docs
```

**Cursor pagination (range query on indexed field):**
```js
// First page
db.posts.find({}).sort({ _id: -1 }).limit(20);

// Next page — pass the last _id from previous page
db.posts.find({ _id: { $lt: lastId } }).sort({ _id: -1 }).limit(20);
```

Always O(log N + page size) regardless of page depth. Index on the sort field (e.g., `_id` default).

---

### Q138. Cursor vs offset pagination.

| | Offset | Cursor |
|---|---|---|
| Implementation | `skip(N).limit(M)` | `find({ _id: { $lt: cursor } }).limit(M)` |
| Performance on deep pages | O(N) — scans all skipped docs | O(log N) — index seek |
| Random page access | ✅ | ❌ (sequential only) |
| Stable under concurrent inserts | ❌ (rows shift) | ✅ |

Offset is fine for shallow pages (admin tables with thousands of rows). Cursor is mandatory for infinite scroll, large datasets, or real-time feeds.

---

### Q139. Case-insensitive search.

**Option 1 — regex (no anchors, no index use):**
```js
db.users.find({ name: { $regex: /^alex$/i } });   // index NOT used due to /i
```

**Option 2 — collation (uses index):**
```js
db.users.createIndex({ name: 1 }, { collation: { locale: "en", strength: 2 } });
db.users.find({ name: "Alex" }).collation({ locale: "en", strength: 2 });
```

**Option 3 — store normalized lowercase field:**
```js
{ name: "Alex Smith", nameLower: "alex smith" }
db.users.createIndex({ nameLower: 1 });
db.users.find({ nameLower: q.toLowerCase() });
```

Option 3 is fastest and simplest for exact matches. Use collation for sorting/comparisons.

---

### Q140. PG vs Mongo for financial system.

**Choose PostgreSQL** for a financial transaction system. Reasons:

1. **Strong ACID transactions** across multiple rows/tables, mature for decades.
2. **Foreign keys, CHECK constraints** ensure referential integrity.
3. **Precise numeric types** (`NUMERIC(20,4)`) — no floating-point math errors.
4. **Standard SQL** — auditors, BI tools, ETL pipelines all understand it.
5. **Better tooling for compliance** — PITR backups, audit extensions, row-level security.
6. **Joins are first-class** — financial reports often span many tables.

Mongo can do multi-doc transactions now, but it's slower and ecosystem support for finance/auditing isn't there. Mongo shines for catalogs, content, event logs, semi-structured data.

---

## REST API Design

### Q141. REST constraints — what makes an API RESTful.

Roy Fielding's six constraints:
1. **Client–Server** — separation of concerns.
2. **Stateless** — each request contains all info needed; server holds no client session.
3. **Cacheable** — responses must indicate cacheability.
4. **Uniform interface** — resources identified by URI, manipulated via standard HTTP verbs.
5. **Layered system** — client can't tell if it's talking to origin or proxy.
6. **Code on demand** (optional) — server can send executable code.

In practice, most "REST APIs" are **RESTful-ish**: they use HTTP verbs and JSON but skip HATEOAS. The Richardson Maturity Model (level 0 → 3) describes degrees of REST-ness.

---

### Q142. `PUT` vs `PATCH`.

- **`PUT`** — replace the **entire** resource with the provided representation. Idempotent.
- **`PATCH`** — apply a **partial** update. Not necessarily idempotent.

```http
PUT /users/1
{ "name": "Alex", "email": "a@x.com", "age": 30 }    # all fields required

PATCH /users/1
{ "email": "new@x.com" }                              # only changed fields
```

`PUT` with missing fields should null them. `PATCH` semantics: JSON Merge Patch (RFC 7396) or JSON Patch (RFC 6902).

In practice many APIs use `PATCH` exclusively for updates.

---

### Q143. `POST` vs `PUT` for creation.

- **`POST /resources`** — server assigns the ID; not idempotent (each call creates a new resource).
- **`PUT /resources/{client-chosen-id}`** — client knows the ID; idempotent (same ID = same resource).

```http
POST /orders          → 201, Location: /orders/abc123
PUT /users/alex       → creates or replaces user "alex"
```

Use `POST` for typical creation. Use `PUT` when the client meaningfully owns the identifier (file uploads keyed by hash, idempotency-key flows).

---

### Q144. Idempotency.

An operation is **idempotent** if applying it multiple times has the same effect as applying it once.

| Method | Idempotent? |
|---|---|
| GET | ✅ |
| HEAD | ✅ |
| PUT | ✅ |
| DELETE | ✅ |
| OPTIONS | ✅ |
| POST | ❌ |
| PATCH | ❌ (depends on implementation) |

Matters for retries. Networks fail; clients retry. If `POST /payments` is retried after a timeout, you may charge twice. Solution: **Idempotency-Key** header — client sends a unique key; server stores result for that key and returns cached response on duplicate.

---

### Q145. Many-to-many — Users and Roles.

```http
# Get user's roles
GET /users/{userId}/roles

# Assign a role
PUT /users/{userId}/roles/{roleId}

# Remove a role
DELETE /users/{userId}/roles/{roleId}

# Replace all roles atomically
PUT /users/{userId}/roles
[{ "id": "admin" }, { "id": "editor" }]

# List all users with a role
GET /roles/{roleId}/users
```

Treat the relationship as a sub-resource. If the relationship has its own attributes (e.g., "assignedAt", "assignedBy"), model it as a first-class resource: `/user-roles/{id}`.

---

### Q146. HATEOAS — needed?

**HATEOAS** (Hypermedia as the Engine of Application State) means responses include links to related actions/resources. Without it, you're at Richardson level 2 — pragmatically "REST" enough for 99% of APIs.

```json
{
  "id": 1, "status": "pending",
  "_links": {
    "self": { "href": "/orders/1" },
    "cancel": { "href": "/orders/1/cancel", "method": "POST" }
  }
}
```

Reality: most clients are tightly coupled (mobile/SPA apps with hardcoded endpoints). HATEOAS adds complexity with rare payoff. Skip unless you have public APIs with diverse, evolving clients.

---

### Q147. API versioning options.

**URL path:**
```http
GET /api/v1/users
GET /api/v2/users
```
- ✅ Explicit, browsable, easy to cache.
- ❌ Pollutes URLs; suggests resources differ.

**Header:**
```http
GET /api/users
API-Version: 2
```
- ✅ Clean URLs.
- ❌ Harder to test in browser; needs custom tooling.

**Query parameter:**
```http
GET /api/users?v=2
```
- ✅ Simple.
- ❌ Easy to forget; clashes with filter params.

**Media type:**
```http
Accept: application/vnd.myapp.v2+json
```
- ✅ Most "RESTful".
- ❌ Verbose; tooling friction.

Most teams use URL path versioning — it's the most pragmatic.

---

### Q148. Breaking changes for unforced mobile clients.

**Strategy: never break, deprecate.**

1. **Maintain v1 indefinitely** alongside v2.
2. **Add new fields/endpoints** to v1; never remove or change semantics.
3. For breaking changes, **fork to v2**.
4. **Sunset policy** — announce v1 deprecation 12+ months ahead with usage tracking.
5. **Force-update path** — return a `426 Upgrade Required` for known-bad client versions when truly necessary (e.g., security).

Server-side flag in responses: `{ "deprecation": "v1 sunset 2027-01-01" }` so clients can warn users to update.

---

### Q149. `200` / `201` / `202` / `204`.

| Code | Meaning |
|---|---|
| 200 OK | Successful GET/PUT/PATCH; body usually present |
| 201 Created | Successful resource creation; `Location` header points to new resource |
| 202 Accepted | Request accepted for async processing; not yet done |
| 204 No Content | Success, intentionally no body (DELETE, PUT with no return) |

```http
POST /orders → 201 Created
Location: /orders/abc
{ "id": "abc", ... }

DELETE /orders/abc → 204 No Content
```

Use the right code — clients differentiate behavior on status codes.

---

### Q150. `400` vs `422`.

- **`400 Bad Request`** — malformed request (invalid JSON, missing required field, wrong content-type).
- **`422 Unprocessable Entity`** — syntactically valid but semantically wrong (email format invalid, business rule violation).

```http
# 400 — JSON parse error
{ broken json...

# 422 — JSON valid, but business rule failed
{ "email": "not-an-email" }
```

Many APIs just use `400` for both — it's acceptable. `422` adds clarity if you want clients to distinguish parse vs validation errors.

---

### Q151. `401` vs `403`.

- **`401 Unauthorized`** — actually means "**unauthenticated**". The client provided no/invalid credentials.
- **`403 Forbidden`** — authenticated, but lacks permission for this resource/action.

```http
GET /admin (no token)              → 401
GET /admin (regular user token)    → 403
```

Always include `WWW-Authenticate` header with 401 for compliance:
```http
401 Unauthorized
WWW-Authenticate: Bearer
```

---

### Q152. Consistent error responses.

Adopt a single error envelope across all endpoints. Common pattern (RFC 7807 — Problem Details):

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "detail": "email must be a valid address",
  "instance": "/users",
  "errors": [
    { "field": "email", "code": "invalid_format" }
  ],
  "requestId": "abc-123"
}
```

Centralise in error middleware. Every error → same shape. Include a `requestId` for support correlation. Never leak stack traces or internal exception messages in production.

---

### Q153. URL structure for blog posts/comments.

```
GET    /posts                       # list
POST   /posts                       # create
GET    /posts/{postId}              # read
PUT    /posts/{postId}              # full update
PATCH  /posts/{postId}              # partial update
DELETE /posts/{postId}              # delete

GET    /posts/{postId}/comments     # list comments
POST   /posts/{postId}/comments     # create
GET    /comments/{commentId}        # read (also valid as top-level)
PATCH  /comments/{commentId}        # update
DELETE /comments/{commentId}        # delete
```

Rules:
- Nouns, not verbs (`/posts`, not `/getPosts`).
- Plural collections.
- Nest only one level deep (`/posts/x/comments/y` is OK; deeper is hard to read).
- Use kebab-case in paths (`/blog-posts` over `/blogPosts`).

---

### Q154. Filters — path or query params?

**Rule of thumb:**
- **Path** identifies the resource — required, hierarchical.
- **Query** modifies or filters the response — optional.

```
GET /users/123/orders               # path: required (user id)
GET /orders?status=open&since=2026  # query: optional filters
GET /orders?customerId=123          # query alt for filter view
```

Don't put filters in the path: `/orders/status/open` is bad — `status` isn't an identifier.

---

### Q155. Filtering + sorting + pagination in one endpoint.

```
GET /products?
  category=books&
  minPrice=10&maxPrice=50&
  sort=price&order=asc&
  page=2&limit=20
```

Implementation:
```js
const { category, minPrice, maxPrice, sort = "createdAt", order = "desc", page = 1, limit = 20 } = req.query;
const where = {};
if (category) where.category = category;
if (minPrice || maxPrice) where.price = { ...(minPrice && { $gte: +minPrice }), ...(maxPrice && { $lte: +maxPrice }) };

const docs = await Product.find(where)
  .sort({ [sort]: order === "asc" ? 1 : -1 })
  .skip((page - 1) * limit)
  .limit(+limit);

res.json({ data: docs, page: +page, limit: +limit, total: await Product.countDocuments(where) });
```

Validate `sort` against an allowlist to prevent abuse (`?sort=passwordHash`).

---

### Q156. Handle `GET /users?sort=name&order=asc&page=2&limit=20`.

```js
const allowedSort = ["name", "createdAt", "email"];
const sort = allowedSort.includes(req.query.sort) ? req.query.sort : "createdAt";
const order = req.query.order === "asc" ? 1 : -1;
const page = Math.max(1, parseInt(req.query.page) || 1);
const limit = Math.min(100, parseInt(req.query.limit) || 20);  // cap

const [data, total] = await Promise.all([
  User.find().sort({ [sort]: order }).skip((page-1)*limit).limit(limit).lean(),
  User.estimatedDocumentCount()
]);

res.json({ data, pagination: { page, limit, total, totalPages: Math.ceil(total/limit) }});
```

Key defenses:
- Allowlist sort fields.
- Cap `limit` to prevent abuse.
- Use `estimatedDocumentCount` for large collections (faster than `countDocuments`).

---

### Q157. Content negotiation.

The client expresses preferences via `Accept` header; server returns the best match.

```http
GET /reports/123
Accept: application/json

→ 200 OK, Content-Type: application/json
```

```http
GET /reports/123
Accept: text/csv

→ 200 OK, Content-Type: text/csv
```

Express:
```js
res.format({
  "application/json": () => res.json(data),
  "text/csv": () => res.send(toCsv(data)),
  default: () => res.status(406).send("Not Acceptable")
});
```

Also: `Accept-Encoding` (gzip/br), `Accept-Language`.

---

### Q158. File upload in REST.

**Multipart form-data (most common):**
```http
POST /uploads
Content-Type: multipart/form-data; boundary=...
```
Works for binary + metadata fields together.

**Raw body:**
```http
POST /uploads
Content-Type: image/png
[binary bytes]
```
Cleaner for single-file uploads.

**Presigned URLs (recommended at scale):**
1. Client `POST /uploads/presign` → server returns S3 presigned PUT URL.
2. Client uploads directly to S3.
3. Client `POST /uploads/finalize` with the S3 key.

Removes upload bandwidth from your servers; safer (you never hold the bytes).

---

### Q159. REST vs GraphQL.

| | REST | GraphQL |
|---|---|---|
| Endpoints | Many resource URLs | Single endpoint |
| Over/underfetching | Common | Avoided (client specifies fields) |
| Caching | HTTP-level (easy) | App-level (harder) |
| Versioning | URL/header | Schema evolution |
| Tooling | Mature | Strong (Apollo, codegen) |
| Learning curve | Low | Medium |

**Choose REST** for: simple CRUD, public APIs, heavy caching needs, file uploads.
**Choose GraphQL** for: complex UI with varied data needs (mobile + web), aggregating multiple backends, evolving schema with many clients.

Many teams use REST internally and GraphQL at the BFF layer.

---

### Q160. gRPC vs REST for internal services.

| gRPC | REST |
|---|---|
| HTTP/2 + Protobuf binary | HTTP/1.1 or 2 + JSON |
| Strongly typed via `.proto` | Loose (or OpenAPI) |
| Streaming (bidirectional) | Server-Sent Events / WebSockets |
| Fast (5–10x JSON) | Human-readable |
| Browser support limited | Universal |

**Use gRPC** for high-throughput internal microservices, polyglot teams (codegen for many languages), streaming. **Use REST** for external/public APIs and where browsers consume directly.

---

### Q161. API Gateway — problems it solves.

A single entry point in front of your microservices. Responsibilities:

- **Routing** — `/users/*` → user service, `/orders/*` → order service.
- **Authentication / authorization** at the edge.
- **Rate limiting & quotas** per API key.
- **Request/response transformation**.
- **Caching** of cacheable responses.
- **Logging, tracing, metrics** in one place.
- **TLS termination**.
- **API versioning** routing.
- **Composition** — fan-out one request to multiple services.

Tools: Kong, AWS API Gateway, Apigee, KrakenD, Tyk, Envoy. For internal: service mesh (Istio, Linkerd).

---

### Q162. Long-running operations in REST.

**202 Accepted + polling:**
```http
POST /videos/process
→ 202 Accepted
   Location: /jobs/abc123

GET /jobs/abc123
→ 200 OK { status: "processing", progress: 0.4 }
→ 200 OK { status: "done", result: { videoUrl: "..." } }
```

**Webhooks:**
Client provides a callback URL; server POSTs results when done. Better for backend-to-backend.

**Server-Sent Events / WebSockets:**
Real-time progress push to the client.

Pick polling for simplicity; webhooks for B2B integrations; WS for live UIs.

---

### Q163. Webhook vs polling — securing.

**Webhook** = your server pushes to a client URL when an event occurs. Beats polling on latency and efficiency.

**Securing incoming webhooks:**
1. **Signature verification** — payload signed with shared secret (HMAC); reject if invalid.
   ```js
   const expected = crypto.createHmac("sha256", SECRET).update(rawBody).digest("hex");
   if (!crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(req.get("X-Signature")))) throw;
   ```
2. **Replay protection** — include timestamp in signed payload; reject if > 5 min old.
3. **Idempotency** — handle duplicate deliveries (use event ID dedup).
4. **IP allowlist** if provider publishes static IPs.
5. **HTTPS only**.

---

### Q164. Document a REST API — OpenAPI.

**OpenAPI** (formerly Swagger) is the de-facto schema for REST APIs.

```yaml
openapi: 3.1.0
paths:
  /users/{id}:
    get:
      parameters:
        - { in: path, name: id, required: true, schema: { type: string } }
      responses:
        "200":
          content:
            application/json:
              schema: { $ref: "#/components/schemas/User" }
```

Benefits:
- Auto-generate client SDKs (multiple languages).
- Swagger UI / Redoc for browsable docs.
- Validate requests/responses against the spec.
- Mock servers (Prism, Stoplight).

Tools: `swagger-jsdoc`, `tsoa`, `zod-to-openapi`, NestJS built-in.

---

### Q165. Synchronous vs asynchronous API design.

**Synchronous (default REST):** client waits, gets result in response.
- Good for fast operations (<2s).
- Easier to reason about.

**Asynchronous:** client gets a job ID, polls or receives webhook.
- Required for long operations (video processing, report generation).
- Resilient — server can retry/scale without holding open connections.
- More complex client logic.

Hybrid: many APIs are sync for the happy path, async (with 202) for heavy operations.

---

### Q166. Search across multiple fields.

```http
GET /products?q=red+leather+jacket
```

Backend:
```js
// SQL
SELECT * FROM products
WHERE to_tsvector('english', name || ' ' || description) @@ plainto_tsquery($1)
ORDER BY ts_rank(...) DESC LIMIT 20;

// Mongo
db.products.find({ $text: { $search: q } }).sort({ score: { $meta: "textScore" } });
```

For richer search (faceting, typos, synonyms): Elasticsearch / Algolia / Meilisearch. Always rank by relevance, not lexicographically.

---

### Q167. Reduce large response sizes.

1. **Compression** — gzip/br at the proxy.
2. **Pagination** — never return unbounded arrays.
3. **Field selection** — `?fields=id,name,email` (sparse fieldsets).
4. **Avoid embedding heavy related data** — return references, let client fetch.
5. **Use efficient formats** — Protobuf or MessagePack for internal APIs; JSON is verbose.
6. **HTTP/2 + header compression**.
7. **Image/asset URLs** instead of base64.

For paginated lists, return summary fields; expose detail at `/items/{id}`.

---

### Q168. Both JSON and CSV from one endpoint.

```js
app.get("/reports/sales", async (req, res) => {
  const data = await getSales();
  res.format({
    "application/json": () => res.json(data),
    "text/csv": () => {
      res.set("Content-Disposition", "attachment; filename=sales.csv");
      res.send(toCsv(data));
    }
  });
});
```

Or via query param: `?format=csv`. Stream CSV for large datasets:
```js
res.type("text/csv");
const stream = db.salesStream().pipe(csvTransformer());
stream.pipe(res);
```

---

### Q169. `ETag` — conditional requests.

`ETag` is a hash/version identifier for a response. Enables conditional GETs:

```http
# First request
GET /users/1
→ 200 OK
  ETag: "abc123"
  { ...user... }

# Subsequent request
GET /users/1
If-None-Match: "abc123"
→ 304 Not Modified              # body not resent, saves bandwidth
```

For updates (optimistic concurrency):
```http
PUT /users/1
If-Match: "abc123"
→ 412 Precondition Failed       # if ETag has changed (someone else updated)
```

Express auto-generates ETags for static + `res.send()`. Disable with `app.disable("etag")` if you handle caching elsewhere.

---

### Q170. Atomic multi-operation REST endpoint.

REST is single-resource by nature; for compound operations:

**Option 1 — coarse-grained endpoint:**
```http
POST /checkout
{ cartId, paymentMethod, shippingAddress }
```
Backend wraps everything in a DB transaction.

**Option 2 — Saga pattern (distributed):**
Step 1 → reserve inventory.
Step 2 → charge card.
Step 3 → create order.
If step 3 fails, compensate (refund + restore inventory).

**Option 3 — Idempotency + retry:**
Each step is idempotent (via idempotency keys); client retries on partial failure.

Don't expose low-level steps as separate endpoints and ask the client to orchestrate transactions — they will fail to do it right.

---

## Authentication & Security

### Q171. Authentication vs authorization.

- **Authentication (AuthN)** — proves *who* you are. "Login." Username/password, JWT, OAuth, SSO.
- **Authorization (AuthZ)** — decides *what* you can do. "Can this user delete this post?" RBAC, ABAC, policy checks.

A request typically flows: authenticate → identify user → authorize action. Confusing them is a common bug ("user is logged in, therefore can do anything").

---

### Q172. JWT — structure and what to store.

A JWT is three base64url-encoded parts separated by dots:

```
header.payload.signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.abc123
```

- **Header** — `{ alg: "HS256", typ: "JWT" }`.
- **Payload** — claims like `sub` (user id), `exp`, `iat`, `roles`.
- **Signature** — HMAC or RSA over `base64(header) + "." + base64(payload)`.

**SAFE to put in payload:** user id, email, roles, expiration. **NEVER put:** passwords, credit card data, anything sensitive — the payload is base64-encoded, **not encrypted**. Anyone with the token can read it.

---

### Q173. JWT storage — localStorage vs cookie.

| | localStorage | httpOnly Cookie |
|---|---|---|
| XSS exposure | ❌ JS can read | ✅ JS cannot read |
| CSRF exposure | ✅ requires header | ❌ automatically sent (needs CSRF token) |
| Cross-site usage | Easy | Needs `SameSite=None; Secure` |
| Mobile clients | Trivial | Awkward |

**Recommendation:** httpOnly Secure SameSite=Strict cookie for web apps. JWT in `Authorization: Bearer` header for mobile/native. Never `localStorage` for tokens — one XSS = full account takeover with no expiry.

---

### Q174. Access vs refresh tokens — rotation.

- **Access token** — short-lived (5–15 min), sent on every API request.
- **Refresh token** — long-lived (days/weeks), used only to obtain a new access token. Stored more securely (httpOnly cookie, encrypted DB).

**Rotation flow:**
1. Login → returns access + refresh tokens.
2. Client uses access token until expiry.
3. On 401, client calls `POST /auth/refresh` with refresh token.
4. Server **issues new access + new refresh**; invalidates old refresh (one-time use).
5. If old refresh is reused → suspicious; revoke entire refresh chain (token reuse detection).

This limits exposure: a leaked access token expires fast; a leaked refresh token is detected on reuse.

---

### Q175. JWT secret compromised — invalidate all tokens.

JWTs are stateless — there's no central "logout." Mitigations:

1. **Rotate the signing secret immediately.** All existing tokens become invalid.
2. Deploy with new secret; old tokens fail signature verification → users must re-login.
3. If using a `jti` (JWT ID) deny-list in Redis, push all active jtis to deny-list. Slower but precise.
4. **Lower future blast radius:** issue tokens with `kid` (key id); store keys in a JWKS endpoint; rotate keys regularly without invalidating all tokens.

This is why pure stateless JWTs are tricky for revocation — many teams keep a session-id reference in Redis alongside the JWT for selective revoke.

---

### Q176. Stateful (session) vs stateless (JWT).

| Session | JWT |
|---|---|
| Server stores session state | Stateless — token is the truth |
| Revoke = delete session row | Revoke is hard (need denylist) |
| Scales horizontally with shared store | Scales without storage |
| Smaller payload (just session id) | Larger payload (claims) |
| Requires cookie | Works in any header |

For typical web apps with one or two services: sessions are simpler. For microservices/mobile/many services: JWT or opaque tokens with introspection.

---

### Q177. OAuth 2.0 — Authorization Code flow.

For server-side apps with a backend:

```
1. User clicks "Login with Google."
2. App redirects to:
   https://accounts.google.com/o/oauth2/auth?
     client_id=APP&redirect_uri=...&response_type=code&scope=email
3. User authenticates with Google, grants consent.
4. Google redirects back: GET /callback?code=AUTH_CODE
5. Backend exchanges code for tokens:
   POST https://oauth2.googleapis.com/token
   { code, client_id, client_secret, grant_type: "authorization_code" }
6. Receives access_token (+ refresh_token).
7. Backend uses access_token to call Google APIs / fetch user info.
8. Backend creates its own session/JWT for the user.
```

The **code** vs **token** indirection prevents tokens from appearing in URLs/browser history. **PKCE** extends this for mobile/SPA (no client secret needed).

---

### Q178. OAuth 2.0 vs OpenID Connect.

- **OAuth 2.0** — authorization framework. "App X can act on behalf of user U with scope S."
- **OIDC** — identity layer on top of OAuth 2.0. Adds an **ID token** (JWT containing user identity), `/userinfo` endpoint, standard claims.

OAuth alone tells you "this token is valid." OIDC adds "this token represents user `alex@x.com`."

Use OIDC when you need to **log users in** (SSO). Use raw OAuth when you only need API access on behalf of users.

---

### Q179. RBAC in Node.js.

```js
const ROLES = {
  user:    ["post:read"],
  editor:  ["post:read", "post:write"],
  admin:   ["post:read", "post:write", "post:delete", "user:manage"]
};

function authorize(...perms) {
  return (req, res, next) => {
    const userPerms = ROLES[req.user.role] || [];
    if (perms.every(p => userPerms.includes(p))) return next();
    res.status(403).json({ error: "forbidden" });
  };
}

app.delete("/posts/:id", authMiddleware, authorize("post:delete"), deletePost);
```

For multi-role users: store an array of roles, union permissions. Externalise the role→permission map so non-engineers can edit.

---

### Q180. ABAC vs RBAC.

**RBAC** = "role determines access." User has role; role has permissions.

**ABAC** = "attributes determine access." Decisions based on user attributes (department, region) + resource attributes (owner, classification) + context (time, IP).

```js
// ABAC: only the owner OR an admin in the same org can edit
function canEdit(user, post) {
  return user.id === post.authorId ||
         (user.role === "admin" && user.orgId === post.orgId);
}
```

Use ABAC when policies depend on data (ownership, sharing, classification). RBAC suffices for simple "admin can do anything, user can do their own things." Tools: OPA (Open Policy Agent), AWS Cedar.

---

### Q181. bcrypt — salt rounds.

```js
const hash = await bcrypt.hash(password, 12); // 12 = "cost factor"
await bcrypt.compare(password, hash); // verify
```

Salt rounds are exponential — each +1 doubles the computation time. Recommendation 2026: **12** for typical web apps. Aim for **~250ms hashing** on production hardware — slow enough to defeat brute-force, fast enough not to block.

bcrypt automatically generates a per-password salt, embedded in the resulting hash. Hash uses 72-byte input cap — truncate or pre-hash with SHA-256 if you must support longer passphrases.

---

### Q182. Secure password reset flow.

```
1. POST /auth/forgot-password { email }
   - ALWAYS respond 200 (don't leak existence).
   - If email exists, generate random 32-byte token, store hash + expiry (15 min).
   - Email link: https://app/reset?token=RAW_TOKEN

2. POST /auth/reset-password { token, newPassword }
   - Hash token, find user, verify not expired.
   - Update password (bcrypt), invalidate the token (one-time use).
   - Invalidate all existing sessions / refresh tokens.
   - Optionally email "Your password was changed."
```

Key points: token in DB is hashed (so DB leak ≠ tokens); single use; short expiry; rate-limited endpoint; constant-time response to avoid timing-based email enumeration.

---

### Q183. HTTPS / TLS — handshake.

**TLS handshake (simplified):**
1. **Client Hello** — supported cipher suites, client random.
2. **Server Hello** — chosen cipher, server random, certificate.
3. **Certificate verification** — client validates cert chain against trusted CAs.
4. **Key exchange** — both derive a shared symmetric key (ECDHE, no key sent over wire).
5. **Finished** — both encrypt subsequent traffic with the symmetric key.

TLS 1.3 reduces this to **one round-trip** (1-RTT) and supports 0-RTT for resumption. Always use TLS 1.2+ in production.

---

### Q184. SSL certificate / self-signed.

- **CA-signed certificate** — issued by a trusted Certificate Authority (Let's Encrypt, DigiCert). Browsers trust it.
- **Self-signed** — you sign your own cert. Browsers show warnings.

**Self-signed acceptable:**
- Local development.
- Internal services with custom trust (mTLS within a private network).
- Pinned client expecting a specific cert.

For anything user-facing — always use Let's Encrypt (free, automated via certbot/Caddy/Traefik).

---

### Q185. CSRF — tokens and JWT.

**CSRF** = attacker tricks a logged-in user's browser into making an authenticated request to your site (via image src, form, JS).

**Why cookies are vulnerable:** browser auto-sends them.
**Why JWT in headers isn't:** browser doesn't send custom headers automatically.

**Defenses:**
1. **SameSite cookies** (`Strict` or `Lax`) — browser blocks cookie on cross-site requests. Biggest single fix.
2. **CSRF token** — random token in form / header; server verifies match.
3. **Double-submit cookie** — token in cookie AND header; attacker can't read cookie due to CORS.
4. **Custom header** (e.g., `X-Requested-With`) — preflighted, attacker can't set arbitrary headers cross-origin.

JWT in `Authorization: Bearer` header is naturally CSRF-resistant.

---

### Q186. XSS — stored vs reflected; prevention.

- **Stored XSS** — attacker payload saved in DB, served to other users.
- **Reflected XSS** — payload in URL/query, reflected back in response.
- **DOM-based XSS** — client-side JS reads user input and writes to DOM unsafely.

**Backend prevention:**
1. **Input validation** — reject obviously bad payloads (but not the only line of defense).
2. **Output encoding** — escape data based on context (HTML, attribute, JS, URL).
3. **Content-Type** — always send `application/json` for APIs (browsers won't execute).
4. **Set CSP** — restricts what scripts can load.
5. **Sanitize HTML** (DOMPurify) if you must accept user HTML (blog posts).

A Node API serving JSON is generally safe; the front-end framework (React, Vue) escapes by default. Beware `dangerouslySetInnerHTML` and `v-html`.

---

### Q187. SQL injection — vulnerable vs parameterized.

**Vulnerable:**
```js
db.query(`SELECT * FROM users WHERE email = '${email}'`);
// Attacker sends:  ' OR 1=1 --
// Resulting SQL:   SELECT * FROM users WHERE email = '' OR 1=1 --'
```

**Parameterized:**
```js
db.query("SELECT * FROM users WHERE email = $1", [email]);
// Driver sends email as bound parameter — never parsed as SQL
```

ORM queries are safe by default. Never concatenate user input into queries. If you must build dynamic SQL (column names from input), use an allowlist.

---

### Q188. NoSQL injection in MongoDB.

Mongo query operators are objects. If a user controls the *shape*, they control the query:

```js
// Vulnerable
db.users.findOne({ email: req.body.email, password: req.body.password });
// Attacker sends: { email: "a@x.com", password: { "$ne": null } }
// Equivalent to: where password != null → returns the user!
```

**Prevent:**
1. **Validate types** — `req.body.password` must be a string, not object.
2. **Use Mongoose schemas + cast** — auto-coerces, rejects objects.
3. **`express-mongo-sanitize`** middleware removes `$` keys from input.
4. Always hash passwords — never `findOne({ password })` directly.

---

### Q189. Brute-force on login.

**Defenses (layer them):**
1. **Rate limit** per IP and per username (`max 5 attempts / 15 min`).
2. **Exponential backoff / lockout** — after N failures, lock account for increasing duration.
3. **Captcha** after threshold.
4. **MFA** — second factor regardless of password.
5. **Slow hash (bcrypt cost 12)** — slows offline brute force.
6. **Notify user** of suspicious attempts.
7. **Block credential-stuffing IPs** via fail2ban / WAF.

Never reveal valid usernames in error messages: respond identically for "user not found" and "wrong password."

---

### Q190. Account enumeration.

If `/login` returns "user not found" for missing emails but "wrong password" for existing emails, an attacker can enumerate all valid accounts.

```
POST /login { email: "alex@x.com", password: "x" } → "wrong password"
POST /login { email: "notexist@x.com", password: "x" } → "user not found"
```

**Fix:** uniform error:
```http
401 Unauthorized
{ "error": "Invalid email or password" }
```

Apply same uniformity to `/forgot-password`, `/signup` (don't tell attacker the email is taken — send confirmation email instead). Use constant-time comparisons to avoid timing-based enumeration.

---

### Q191. MFA / 2FA.

Common flow with TOTP:

```
1. Setup:
   POST /mfa/setup → server generates secret, returns QR (otpauth://totp/...).
   User scans with Google Authenticator / Authy.
   User confirms with current TOTP code.
   Server stores secret (encrypted) + flips mfa_enabled = true.

2. Login:
   POST /auth/login { email, password } → returns "mfa_required: true"
   POST /auth/mfa { email, code } → verify code with `speakeasy.totp.verify`.
   On success → issue session/JWT.

3. Backup codes:
   Generate 10 one-time codes at setup.
   Store hashed; mark as used on consumption.
```

Add WebAuthn (passkeys) for stronger phishing-resistant 2FA. Always offer backup codes for device-loss recovery.

---

### Q192. Timing attack and `timingSafeEqual`.

Naïve string comparison short-circuits on first mismatch — exposes how many chars matched via tiny timing differences. Attackers measure latency across many requests to recover secrets bit-by-bit.

```js
// ❌ vulnerable
if (token === expected) { ... }

// ✅ constant time
if (crypto.timingSafeEqual(Buffer.from(token), Buffer.from(expected))) { ... }
```

`timingSafeEqual` checks every byte regardless of where they differ — same execution time. Use for token/HMAC/MAC comparisons.

---

### Q193. Secure secret storage in production.

**Don't:**
- Commit secrets to git (even private repos).
- Bake secrets into Docker images.
- Pass as command-line args (visible in `ps`).
- Log them.

**Do:**
- **Secret manager** — AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Doppler.
- Inject at runtime via env vars from the secret manager.
- Encrypt at rest (DB-level or app-level encryption).
- Rotate periodically; automate rotation.
- **Audit access** logs.
- Use **IAM roles** instead of static credentials when possible (AWS IAM roles for EC2/EKS).

---

### Q194. Principle of least privilege — DB users.

Each component gets the **minimum permissions needed**.

```sql
-- App write user
CREATE USER app_rw WITH PASSWORD '...';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_rw;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_rw;

-- Read-only user for analytics/replicas
CREATE USER app_ro WITH PASSWORD '...';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_ro;

-- Migration user (used only by CI for `prisma migrate deploy`)
CREATE USER app_migrate WITH PASSWORD '...';
GRANT ALL PRIVILEGES ON SCHEMA public TO app_migrate;
```

Never connect the app as the DB superuser. If the app is compromised, the attacker inherits exactly what the app could do — not the entire DB.

---

### Q195. Security audit log — what to log.

**Log:**
- Logins (success, failure, MFA challenges).
- Password changes, reset requests.
- Permission grants/revokes.
- Privileged actions (admin operations, user deletions).
- API key creation/use.
- Source IP, user agent, request id, user id, timestamp.

**Never log:**
- Passwords (even hashed).
- Full credit card numbers.
- API tokens / session ids.
- Full request/response bodies for auth endpoints.

Use structured JSON logs; ship to immutable storage (S3 with object lock, SIEM). Retention: typically 1–7 years for compliance.

---

### Q196. Mass assignment vulnerability.

```js
// ❌ Vulnerable
const user = await User.create(req.body);
// Attacker sends: { name, email, password, role: "admin", isVerified: true }
```

**Fix:** allowlist fields explicitly.
```js
const { name, email, password } = req.body; // pick only safe fields
const user = await User.create({ name, email, password });

// Or via Zod
const schema = z.object({ name: z.string(), email: z.string().email(), password: z.string() }).strict();
const data = schema.parse(req.body);
```

Mongoose: enable `strict: "throw"` to reject unknown fields. ORMs: never spread `req.body` into model constructors without filtering.

---

### Q197. SSRF — Server-Side Request Forgery.

The server makes an HTTP request to a URL controlled by the user — attacker can target internal services.

```js
// Vulnerable
app.get("/proxy", (req, res) => {
  axios.get(req.query.url).then(r => res.send(r.data));
});
// Attacker: ?url=http://169.254.169.254/latest/meta-data/  (AWS metadata, steal creds)
//           ?url=http://localhost:6379/                     (local Redis)
```

**Prevent:**
1. **Allowlist** of permitted destination hosts.
2. **Block private IP ranges** (10.x, 172.16–31.x, 192.168.x, 127.x, 169.254.x).
3. **Resolve DNS first** and re-check after resolution (avoid DNS rebinding).
4. **Disable redirects** or re-validate each redirect target.
5. **Use a forward proxy** with strict policies.

Library: `ssrf-req-filter`.

---

### Q198. `npm audit`.

```bash
npm audit                 # report vulnerabilities
npm audit fix             # patch automatically (semver-compatible)
npm audit fix --force     # may break (uses major bumps)
```

Reports CVEs in your dependencies. In CI, fail builds on high/critical severity (`npm audit --audit-level=high`).

Better tools:
- **Snyk**, **Dependabot**, **Renovate** — automated PRs to bump vulnerable deps.
- **OWASP Dependency-Check**.
- **SBOM** generation (CycloneDX) for supply-chain visibility.

Audit only catches known CVEs in known packages — doesn't protect against malicious typosquatted packages. Pin versions, use lockfiles, review new dependencies.

---

### Q199. CSP header.

`Content-Security-Policy` restricts where the browser can load resources from — primary mitigation for XSS.

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.jsdelivr.net;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https://images.example.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  report-uri /csp-report;
```

Even if attacker injects `<script src="http://evil.com/x.js">`, the browser refuses to load it. Start with `Content-Security-Policy-Report-Only` to catch violations without breaking the site, then enforce.

---

### Q200. API keys + JWT side by side.

For B2B API clients (machine-to-machine) alongside user JWT auth:

```js
// Middleware
async function auth(req, res, next) {
  const apiKey = req.get("X-API-Key");
  const bearer = req.get("Authorization")?.replace("Bearer ", "");

  if (apiKey) {
    const key = await ApiKey.findOne({ keyHash: hash(apiKey), active: true });
    if (!key) return res.status(401).end();
    req.apiClient = key.clientId;
    req.permissions = key.scopes;
  } else if (bearer) {
    const payload = jwt.verify(bearer, SECRET);
    req.user = await User.findById(payload.sub);
  } else {
    return res.status(401).end();
  }
  next();
}
```

API keys:
- Hashed at rest (bcrypt or SHA-256).
- Rotatable; users can revoke.
- Scoped (read-only, write, admin).
- Rate-limited per key.
- Logged on every use.
- Optional IP allowlist.

---

## Caching & Redis

### Q201. Caching strategies.

| Strategy | Flow |
|---|---|
| **Cache-aside (lazy)** | App reads cache; on miss → read DB, write cache. Most common. |
| **Read-through** | App reads cache; cache library reads DB on miss transparently. |
| **Write-through** | App writes to cache; cache writes to DB synchronously. Consistent, slower writes. |
| **Write-behind** | App writes to cache; cache writes to DB later (async). Fast, risk of loss. |
| **Refresh-ahead** | Cache proactively refreshes hot keys before expiry. |

```js
// Cache-aside
async function getUser(id) {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);
  const user = await db.users.findById(id);
  await redis.setEx(`user:${id}`, 300, JSON.stringify(user));
  return user;
}
```

---

### Q202. 500ms response → Redis caching.

1. **Identify the bottleneck** — confirm DB query is the slow part (APM, query timing).
2. **Decide cache key** — `report:${userId}:${date}` — must include all parameters that affect the result.
3. **Decide TTL** — based on freshness tolerance (1 min for live data, 1 hour for hourly snapshots).
4. **Add cache-aside layer** in the handler.

```js
const key = `dashboard:${userId}`;
let data = await redis.get(key);
if (!data) {
  data = JSON.stringify(await runHeavyQuery(userId));
  await redis.setEx(key, 60, data);
}
res.json(JSON.parse(data));
```

5. **Invalidate** on writes (e.g., when underlying data changes).
6. **Monitor hit ratio** — aim for >80% on hot endpoints.

Result: cache hits respond in <5ms; misses still 500ms.

---

### Q203. Cache invalidation — why hard.

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

Problems:
1. **Stale reads** — DB updated, cache not yet invalidated.
2. **Race conditions** — concurrent read+write can write a stale value back after delete.
3. **Fan-out** — one change might invalidate many cached entries.
4. **Distributed caches** — invalidation must propagate across nodes/regions.
5. **TTL trade-off** — short TTL = more misses; long TTL = more staleness.

Strategies: TTL (eventual consistency), write-through, event-driven invalidation (publish on change), versioned keys (`user:123:v5` — bump on write).

---

### Q204. TTL — how to choose.

Consider:
- **How fast does the data change?** Hourly metrics → 1h TTL. Real-time → seconds.
- **How critical is freshness?** Bank balance → near-zero TTL or no cache. Catalog → minutes.
- **How expensive is regeneration?** Heavy aggregation → longer TTL.
- **What's the read rate?** High-read warrants longer TTL (more hits).

Rules of thumb:
- Sessions: matches session timeout (e.g., 24h).
- User profile: 5–15 min.
- Static catalog: 1h+.
- Hot lists / leaderboards: 30s–5min.

Add **jitter** (`TTL ± 10%`) to spread expiries and avoid stampedes.

---

### Q205. Cache stampede.

When a hot key expires, many concurrent requests miss the cache simultaneously, **all** hit the DB, and **all** try to repopulate — overwhelming the DB.

**Prevention:**
1. **Lock-based regeneration** — first miss acquires a Redis lock; others wait or serve stale.
2. **Probabilistic early expiration (XFetch)** — refresh slightly before expiry randomly.
3. **`stale-while-revalidate`** — serve stale data while refreshing in background.
4. **Request coalescing** in-process (singleflight pattern) — dedupe concurrent identical loads.
5. **Pre-warm** caches on deploy.

```js
async function getWithLock(key, regenFn, ttl) {
  let val = await redis.get(key);
  if (val) return JSON.parse(val);
  const lock = await redis.set(`lock:${key}`, "1", { NX: true, EX: 10 });
  if (lock) {
    val = await regenFn();
    await redis.setEx(key, ttl, JSON.stringify(val));
    await redis.del(`lock:${key}`);
    return val;
  }
  // wait briefly and retry
  await sleep(50);
  return getWithLock(key, regenFn, ttl);
}
```

---

### Q206. Redis commands.

| Command | Data type | Use |
|---|---|---|
| `SET k v` / `GET k` | String | Basic key-value, JSON blobs |
| `HSET k f v` / `HGET k f` | Hash | User profile fields, structured |
| `LPUSH k v` / `LPOP k` | List | Queue (FIFO with `RPOP`), recent items |
| `SADD k v` / `SISMEMBER k v` | Set | Tags, unique membership |
| `ZADD k score v` | Sorted set | Leaderboards, time-ordered streams |
| `EXPIRE k sec` | — | Set TTL |
| `INCR k` | String (counter) | Atomic counters |
| `MGET k1 k2 k3` | String | Batch get |

```redis
SET session:abc123 '{"userId":1}' EX 3600
HSET user:1 name "Alex" email "a@x.com"
ZADD leaderboard 9999 "alex"
ZREVRANGE leaderboard 0 9 WITHSCORES
```

---

### Q207. Session store with Redis + Express.

```js
import session from "express-session";
import RedisStore from "connect-redis";
import { createClient } from "redis";

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient, prefix: "sess:" }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: true,        // HTTPS only
    sameSite: "lax",
    maxAge: 24 * 3600 * 1000
  }
}));
```

Each session gets a key `sess:<id>` in Redis with the session payload. Cookies hold only the session id. Sessions survive app restarts; scales across replicas.

---

### Q208. Rate limiting with Redis.

**Fixed window counter:**
```js
async function rateLimit(key, max, windowSec) {
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, windowSec);
  return count <= max;
}
// Usage: const ok = await rateLimit(`rl:${ip}`, 100, 60);
```

**Sliding window log (more accurate):**
```js
async function rateLimitSliding(key, max, windowMs) {
  const now = Date.now();
  await redis.zRemRangeByScore(key, 0, now - windowMs);
  const count = await redis.zCard(key);
  if (count >= max) return false;
  await redis.zAdd(key, { score: now, value: `${now}-${Math.random()}` });
  await redis.expire(key, Math.ceil(windowMs / 1000));
  return true;
}
```

**Token bucket** is best for burst-tolerant limits. Libraries: `rate-limiter-flexible`, `express-rate-limit` + redis store.

---

### Q209. Redis pub/sub vs message queue.

| Redis pub/sub | RabbitMQ / Kafka |
|---|---|
| Fire-and-forget | Persistent messages |
| Subscribers must be online | Messages stored until consumed |
| No delivery guarantees | At-least-once / exactly-once |
| No queue (no consumer = lost) | Durable queues |
| Sub-ms latency | Higher latency, much more reliable |
| Simple | Operationally heavier |

Use Redis pub/sub for **ephemeral notifications** (real-time updates between server instances, cache invalidation). Use a proper queue for **work** (background jobs, reliable delivery).

**Redis Streams** (since Redis 5) bridge the gap — durable, consumer groups, like a lightweight Kafka.

---

### Q210. Sorted set — leaderboard.

```js
// Update score
await redis.zAdd("leaderboard", { score: 9999, value: "alex" });
await redis.zIncrBy("leaderboard", 10, "alex");  // +10

// Top 10
const top = await redis.zRange("leaderboard", 0, 9, { REV: true, WITHSCORES: true });

// User's rank (0-based)
const rank = await redis.zRevRank("leaderboard", "alex");

// Users in score range
const inRange = await redis.zRangeByScore("leaderboard", 1000, 2000);
```

All operations are O(log N). Perfect for leaderboards, recent-N feeds (use timestamp as score), priority queues.

---

### Q211. Distributed lock — Redlock.

Single Redis node:
```js
// Acquire
const ok = await redis.set("lock:job-42", workerId, { NX: true, EX: 30 });
if (ok) {
  try { await doWork(); }
  finally { await redis.del("lock:job-42"); }
}
```

**Better — release only if we still own it (Lua script):**
```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end
```

**Redlock algorithm** (multiple Redis instances) — acquire lock on majority of N independent Redis nodes within timeout; if you fail, release all. Handles single-node failures. Library: `redlock`.

Note: Redlock is debated (Martin Kleppmann critique). For most use cases, single-instance lock with proper expiry is sufficient; for stronger guarantees, use ZooKeeper or etcd.

---

### Q212. Redis eviction policies.

When memory limit reached, Redis evicts per `maxmemory-policy`:

| Policy | Behavior |
|---|---|
| `noeviction` | Refuse writes (default) |
| `allkeys-lru` | Evict least-recently-used across all keys |
| `volatile-lru` | LRU only on keys with TTL |
| `allkeys-lfu` | Least-frequently-used (Redis 4+) |
| `volatile-lfu` | LFU on keys with TTL |
| `allkeys-random` | Random eviction |
| `volatile-ttl` | Evict keys closest to expiry |

**Session store** — `volatile-lru` (sessions have TTL; evict idle ones).
**General cache** — `allkeys-lru` or `allkeys-lfu`.
**Primary data store** — `noeviction` (don't lose data; provision enough memory).

---

### Q213. `EXPIRE` vs `EXPIREAT`.

- `EXPIRE key 60` — expires in 60 seconds from now.
- `EXPIREAT key 1735689600` — expires at Unix timestamp.

```js
redis.expire("temp", 3600);              // in 1 hour
redis.expireAt("session", Date.now() / 1000 + 86400); // exact time
```

`EXPIREAT` is useful when expiry is tied to an external event (token issued at T, expires at T+24h regardless of when stored). `PEXPIRE`/`PEXPIREAT` use milliseconds.

---

### Q214. Paginated cache.

Cache each page separately:
```js
const key = `posts:list:p${page}:l${limit}`;
let data = await redis.get(key);
if (!data) {
  data = JSON.stringify(await getPosts(page, limit));
  await redis.setEx(key, 60, data);
}
```

**Challenges:**
- Invalidation on writes: deleting a post shifts pages — must invalidate all.
- Use a key pattern + `SCAN` to delete: `redis.scanIterator({ MATCH: "posts:list:*" })`.
- Or use a version key: `posts:v${version}:p${page}` — bump version on write, old keys naturally expire.

For mostly-static content, cache aggressively; for highly dynamic, cache only first page or skip caching.

---

### Q215. RDB vs AOF.

- **RDB (snapshot)** — periodic binary dump of the dataset. Fast restart; data loss possible since last snapshot (default every 5 min).
- **AOF (append-only file)** — every write logged. Replayed on restart. Larger files; minimal data loss (depends on `appendfsync` — `everysec` is typical, ≤1s loss).

**Choose:**
- **Cache** — RDB only is fine (loss tolerable).
- **Primary data store / sessions** — both, RDB for fast restart + AOF for durability.

```conf
save 900 1
appendonly yes
appendfsync everysec
```

---

### Q216. Redis Cluster vs Sentinel.

| Sentinel | Cluster |
|---|---|
| HA for single primary | Sharded across multiple primaries |
| Auto-failover | Auto-failover per shard |
| 1 master + replicas | N masters, each with replicas |
| Same data on all nodes | Data split by key hash slots |
| Easier setup | More complex, but scales horizontally |

**Sentinel:** "I have one Redis and want it always available."
**Cluster:** "My dataset exceeds one node's RAM, or I need write throughput beyond one master."

Most apps start with a managed Redis (Elasticache, MemoryDB, Upstash, Redis Cloud) and never need to think about this.

---

### Q217. Handle Redis connection failure (fail-open).

```js
async function getCached(key, regen) {
  try {
    const v = await redis.get(key);
    if (v) return JSON.parse(v);
  } catch (err) {
    logger.warn({ err }, "redis get failed; falling back to DB");
  }
  const data = await regen();
  try {
    await redis.setEx(key, 300, JSON.stringify(data));
  } catch {}
  return data;
}
```

**Fail-open** = cache failures should not break the app; degrade to direct DB access.

Caveats:
- DB may be overwhelmed if cache is down — circuit-break if DB latency spikes.
- For rate-limiting / locks, you may need **fail-closed** (refuse the request) to prevent abuse.

---

### Q218. Cache hit ratio.

```
Hit ratio = hits / (hits + misses)
```

- **Below 50%** — cache isn't paying for itself.
- **80–95%** — typical good range.
- **>95%** — excellent; TTLs probably tuned well.

Monitor via Redis `INFO stats` (`keyspace_hits`, `keyspace_misses`) or APM. Per-key hit ratio is more useful than overall.

Improve hit ratio by:
- Longer TTL where freshness allows.
- Pre-warming on deploy.
- Bigger memory budget (fewer evictions).
- Better key design (avoid keys with too-high cardinality).

---

### Q219. Per-user vs shared cache.

**Per-user data (private):**
```js
const key = `user:${userId}:cart`;   // scoped, never shared
```

**Shared data:**
```js
const key = `products:list:p${page}`;
```

Considerations:
- Per-user keys can explode in count (1M users × 10 keys = 10M keys) — ensure adequate memory and eviction policy.
- Shared keys benefit all users — high hit ratio.
- Mixed-tenancy: include tenant id in key (`org:42:products`) to prevent leaks.
- Cache personalised data (recommendations) only if cost of regeneration justifies it.

Never cache per-user data globally — privacy and freshness both suffer.

---

### Q220. Stale data after DB update.

**Cache invalidation on write:**
```js
await db.users.update(id, { name });
await redis.del(`user:${id}`);  // invalidate
await redis.del(`user:${id}:profile`);  // anything derived from this user
```

**Pitfalls:**
- Two-phase: cache delete before or after DB commit? **After commit**, otherwise a concurrent read can repopulate stale data right before the DB commits.
- Multi-key fan-out: a user update invalidates many derived caches → use **publish/subscribe** to broadcast invalidation.
- For complex graphs, prefer **versioned keys** (`user:42:v7`) and bump the version atomically.

**Patterns:**
- Write-through cache (writes go through cache to DB).
- Event-driven (DB writes publish events, listeners invalidate).
- Short TTL + accept eventual consistency.

---

## Message Queues

### Q221. Why use a queue instead of direct calls.

Direct API call:
```
Service A → HTTP → Service B
```
- If B is down, A fails.
- If B is slow, A blocks.
- A must know B's location.

Via queue:
```
Service A → Queue → Service B (consumes asynchronously)
```

**Benefits:**
- **Decoupling** — A doesn't need B alive.
- **Smoothing load** — burst goes to queue; B processes at steady rate.
- **Retries** — failed message goes back to queue or DLQ.
- **Multi-consumer** — many B instances scale by consuming from the queue.
- **Async UX** — return 202 immediately, process in background.

Use queue for: emails, notifications, image processing, billing, analytics events. Not for: synchronous CRUD (client waits for a record).

---

### Q222. Message queue vs pub/sub.

| Queue | Pub/Sub |
|---|---|
| One message → one consumer | One message → many subscribers |
| Work distribution | Event broadcast |
| RabbitMQ work queues, SQS | Redis pub/sub, Kafka topics (with consumer groups: queue-like) |
| Competing consumers | Fan-out |

```
Queue:   Producer → [msg] → Worker1 OR Worker2 (one of them)
PubSub:  Publisher → [event] → Subscriber1 AND Subscriber2 AND Subscriber3
```

Kafka and RabbitMQ can do both modes depending on configuration.

---

### Q223. RabbitMQ — exchanges/queues/bindings.

```
Publisher → Exchange → (binding) → Queue → Consumer
```

- **Exchange** — receives messages, routes to queues based on type + binding key.
- **Queue** — buffers messages.
- **Binding** — link between exchange and queue, optionally with a routing key.

```js
await channel.assertExchange("orders.events", "topic");
await channel.assertQueue("emails");
await channel.bindQueue("emails", "orders.events", "order.created");
channel.publish("orders.events", "order.created", Buffer.from(JSON.stringify(data)));
```

This indirection lets you add new consumers (queues) without changing publishers.

---

### Q224. Exchange types.

| Type | Routing |
|---|---|
| **direct** | Routing key must exactly match binding key |
| **fanout** | Ignores routing key; sends to ALL bound queues |
| **topic** | Routing key matches binding pattern (`order.*.created`, `*.us-east.*`) |
| **headers** | Routes by message headers, not routing key (rarely used) |

**Examples:**
- `direct` — `payment.success` → only payment-success queue.
- `fanout` — broadcast cache invalidation to all instances.
- `topic` — `metrics.api.us-east.latency` matched by `metrics.api.*.latency`.

Topic is the most flexible and commonly used.

---

### Q225. Dead Letter Queue.

A DLQ is a queue for messages that couldn't be processed:

**Causes of DLQ entry:**
1. Message rejected with `requeue=false`.
2. TTL expired.
3. Queue length limit exceeded.
4. Consumer retries exhausted.

```js
await channel.assertQueue("emails", {
  arguments: {
    "x-dead-letter-exchange": "dlx",
    "x-dead-letter-routing-key": "emails.failed"
  }
});
```

**DLQ strategy:**
- Monitor DLQ depth — should be near zero in healthy systems.
- Alert on growth.
- Manual replay or automated retry with backoff.
- Inspect payloads to fix poison messages.

---

### Q226. Retry with exponential backoff.

```js
async function consume(msg) {
  const attempt = msg.properties.headers["x-retry"] || 0;
  try {
    await process(msg.content);
    channel.ack(msg);
  } catch (err) {
    if (attempt < MAX_RETRIES) {
      const delayMs = Math.min(30_000, 2 ** attempt * 1000) + Math.random() * 500;
      setTimeout(() => {
        channel.publish("retry-ex", "retry", msg.content, {
          headers: { "x-retry": attempt + 1 }
        });
      }, delayMs);
      channel.ack(msg);
    } else {
      channel.nack(msg, false, false); // → DLQ
    }
  }
}
```

Better: use a **delayed-message plugin** (RabbitMQ) or scheduled queues to avoid in-process timers. BullMQ has built-in backoff config.

---

### Q227. Message acknowledgment.

When a consumer fetches a message:
- **`ack`** — successfully processed; remove from queue.
- **`nack`/`reject`** — failed; requeue or send to DLQ.
- **No ack** — message stays in queue.

If a consumer **crashes** before acking, RabbitMQ requeues the message automatically (after the connection closes). This guarantees at-least-once delivery.

```js
channel.consume("emails", async (msg) => {
  try {
    await send(JSON.parse(msg.content));
    channel.ack(msg);
  } catch {
    channel.nack(msg, false, true); // requeue
  }
}, { noAck: false }); // must be false for manual ack
```

Set `prefetch(1)` to avoid one consumer hoarding messages while others sit idle.

---

### Q228. `ack` vs `nack` vs `reject`.

| Method | Behaviour |
|---|---|
| `ack(msg)` | Successful processing; remove from queue. |
| `nack(msg, multiple, requeue)` | Negative ack. Can target multiple; can requeue or DLQ. |
| `reject(msg, requeue)` | Same as `nack` but for single message; older API. |

```js
channel.nack(msg, false, false);  // single, no requeue → DLQ
channel.nack(msg, false, true);   // single, requeue
```

Use `nack` in modern code; `reject` exists for AMQP 0.9.1 compatibility.

---

### Q229. Competing consumers — scaling.

```
Queue: [m1] [m2] [m3] [m4] [m5]
        ↓     ↓     ↓
     Worker1 Worker2 Worker3   (round-robin distribution)
```

Each message is delivered to exactly one consumer. Add workers to scale throughput linearly (until DB or external dependency becomes the bottleneck).

**Important:** set `channel.prefetch(N)` so each worker holds at most N un-acked messages. Otherwise, one slow worker can hoard the whole queue. Tune N based on per-message processing time vs network latency.

---

### Q230. Kafka vs RabbitMQ.

| Kafka | RabbitMQ |
|---|---|
| Distributed log (append-only) | Smart broker, routing exchanges |
| Pull (consumers tail) | Push to consumers |
| Persistent (days/weeks retention) | Removed after ack |
| Replay possible | Once consumed, gone |
| Millions of msgs/sec | Tens of thousands/sec |
| Coarse routing (topic + partition) | Rich routing (exchanges, headers) |
| Best for streaming/events | Best for task queues |

Kafka is the event-streaming backbone; RabbitMQ is a smart broker for work distribution.

---

### Q231. Kafka topic/partition/consumer group.

- **Topic** — named stream of events ("orders.created").
- **Partition** — topic split for parallelism; each partition is an ordered log. Messages within a partition are strictly ordered.
- **Consumer group** — set of consumers sharing the work. Each partition is consumed by **exactly one** consumer in the group.

```
Topic "orders" with 3 partitions:
  Partition 0 → Consumer A (group X)
  Partition 1 → Consumer B (group X)
  Partition 2 → Consumer C (group X)

  Add Consumer D? Idle until rebalance gives them a partition.
```

Partition count = max parallelism. Pick by hottest expected throughput. Order is guaranteed only **within** a partition — choose partition key carefully (e.g., user_id) so related events stay ordered.

---

### Q232. Why Kafka for streaming, Rabbit for tasks.

**Kafka strengths:**
- Persistent log — replay events for new consumers / debugging.
- Horizontal scaling via partitions.
- High throughput (sequential disk writes).
- Multiple independent consumer groups read the same stream.

**RabbitMQ strengths:**
- Per-message routing flexibility (headers, topics, RPC).
- Priorities, TTL, dead lettering built-in.
- Simpler ops at low/medium scale.

Use Kafka when you have **events** to be processed by many systems (audit, search index, analytics, ML, billing). Use RabbitMQ when you have **commands/jobs** that need flexible routing and reliable single-consumer delivery.

---

### Q233. At-most / at-least / exactly-once.

- **At-most-once** — fire and forget. Lost on consumer crash. Used in metrics, logs.
- **At-least-once** — default for most queues. Acks required; duplicates possible on retry.
- **Exactly-once** — hardest. Requires idempotency + transactions. Kafka has "EoS" with idempotent producer + transactional writes.

In practice: most systems run **at-least-once with idempotent consumers**. Each message has a unique id; consumers dedup using a processed-id store.

```js
async function handle(msg) {
  const seen = await redis.set(`processed:${msg.id}`, "1", { NX: true, EX: 86400 });
  if (!seen) return;  // duplicate; skip
  await process(msg);
}
```

---

### Q234. Poison message loop.

A message that always throws — gets nacked + requeued → consumed again → throws → loop.

**Mitigations:**
1. **Retry counter** in headers. After N attempts → DLQ.
2. **Visibility timeout / lease** — message held by consumer; if not acked in time, redelivered (but with a counter).
3. **Dead-letter exchange** with TTL on retry queue.

```js
const retries = msg.properties.headers["x-death"]?.[0]?.count ?? 0;
if (retries >= 5) {
  channel.publish("dlx", "poison", msg.content);
  channel.ack(msg);
  return;
}
```

Always inspect DLQ and fix root causes — bad data formats, missing dependencies.

---

### Q235. Message ordering in distributed queues.

**Within a single Kafka partition:** strictly ordered.
**Across partitions / queues:** no global ordering.

**To preserve order for related messages:**
- **Kafka:** pick partition key = entity id (user_id, order_id). All events for that entity land in the same partition.
- **RabbitMQ:** single queue + single consumer (sequential) or consistent hash exchange plugin.

**Trade-off:** ordering requirements limit parallelism. If you need global order, you can only have one consumer at a time.

When designing, ask: do I need *global* order, or just *per-entity* order? Usually the latter.

---

## Testing

### Q236. Unit vs integration vs E2E.

| | Unit | Integration | E2E |
|---|---|---|---|
| Scope | One function/class | Multiple modules + real deps | Whole app, real infra |
| Speed | Milliseconds | Seconds | Minutes |
| Stable | Very | Mostly | Brittle |
| Example | `calcTotal(items)` returns correct sum | `POST /orders` writes to test DB | Browser → API → DB → email |

**Backend examples:**
- Unit: `formatInvoice(order) === '...'`.
- Integration: `request(app).post('/orders').send({...})` hitting Postgres-test.
- E2E: Playwright/Cypress driving the full UI + backend.

---

### Q237. Testing pyramid.

```
         /\        E2E (few)
        /--\
       /    \      Integration (some)
      /------\
     /        \    Unit (many)
    /----------\
```

- **Lots** of fast unit tests (70%).
- **Some** integration tests covering module boundaries (20%).
- **Few** E2E tests covering critical user flows (10%).

Inverted pyramids (lots of E2E) become slow and flaky. Most logic should be testable in unit form — if it isn't, refactor for testability.

---

### Q238. Unit-test a function with DB call (no real DB).

Use **dependency injection** — inject the DB client (or repository) so tests can pass a mock:

```js
// service.js
export const createOrderService = (db) => ({
  async place(items) {
    const total = items.reduce((s, i) => s + i.price * i.qty, 0);
    return db.orders.insert({ total, items });
  }
});

// service.test.js
test("places order with computed total", async () => {
  const db = { orders: { insert: jest.fn().mockResolvedValue({ id: 1 }) } };
  const svc = createOrderService(db);
  await svc.place([{ price: 10, qty: 2 }]);
  expect(db.orders.insert).toHaveBeenCalledWith({ total: 20, items: expect.any(Array) });
});
```

Pure functions + injected dependencies = trivially testable.

---

### Q239. Mock / stub / spy in Jest.

- **Mock** — replaces a function/module entirely; you control inputs/outputs.
- **Stub** — like a mock, but specifically returns canned data (subset of mocking).
- **Spy** — wraps a real function to record calls while still executing it.

```js
// Mock: full replacement
jest.mock("./email", () => ({ send: jest.fn().mockResolvedValue(true) }));

// Stub: return canned value
jest.spyOn(api, "fetch").mockReturnValue(Promise.resolve({ data: [] }));

// Spy: observe real behavior
const spy = jest.spyOn(logger, "info");
doThing();
expect(spy).toHaveBeenCalledWith("started");
spy.mockRestore();
```

---

### Q240. Test an Express route handler in isolation.

```js
import handler from "./getUser";

test("returns 404 if user not found", async () => {
  const req = { params: { id: "1" } };
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };
  const db = { users: { findById: jest.fn().mockResolvedValue(null) } };

  await handler({ db })(req, res);

  expect(res.status).toHaveBeenCalledWith(404);
  expect(res.json).toHaveBeenCalledWith({ error: "not found" });
});
```

This pattern requires writing handlers as factories that accept dependencies. Easier than mocking the entire Express stack.

---

### Q241. Integration test with supertest.

```js
import request from "supertest";
import { app } from "../src/app";

describe("POST /api/users", () => {
  beforeEach(() => db.migrate.rollback().then(() => db.migrate.latest()));

  it("creates a user", async () => {
    const res = await request(app)
      .post("/api/users")
      .send({ email: "a@x.com", password: "secret123" })
      .expect(201);

    expect(res.body).toMatchObject({ id: expect.any(String), email: "a@x.com" });
    expect(res.body).not.toHaveProperty("password");
  });

  it("rejects invalid email", async () => {
    await request(app)
      .post("/api/users")
      .send({ email: "bad", password: "secret123" })
      .expect(422);
  });
});
```

Hits the real Express app, real test DB. Slower but tests the wiring + middleware + validation + DB together.

---

### Q242. TDD — failing test first.

**Red → Green → Refactor:**

1. **Red** — write a failing test for the behavior you want.
   ```js
   test("calc tax 10%", () => expect(calcTax(100)).toBe(10));
   // Fails: calcTax doesn't exist
   ```
2. **Green** — write the minimum code to pass.
   ```js
   const calcTax = (n) => n * 0.1;
   ```
3. **Refactor** — improve structure while keeping tests green.

Benefits: forces small modular code, automatic regression protection, design pressure to keep functions pure. Not always practical (UI exploration, performance optimization), but excellent for business logic.

---

### Q243. Test error cases.

```js
test("returns 500 if DB throws", async () => {
  jest.spyOn(db.users, "findById").mockRejectedValue(new Error("boom"));

  const res = await request(app).get("/users/1");
  expect(res.status).toBe(500);
  expect(res.body.error).toBe("Internal Server Error");
});

test("retries on transient error then succeeds", async () => {
  const fn = jest.fn()
    .mockRejectedValueOnce(new Error("transient"))
    .mockResolvedValueOnce({ id: 1 });

  const result = await withRetry(fn);
  expect(result).toEqual({ id: 1 });
  expect(fn).toHaveBeenCalledTimes(2);
});
```

Errors are the easy place to break things — always test the unhappy paths.

---

### Q244. Code coverage.

Measures the % of lines/branches executed by tests:

```bash
jest --coverage
```

- **80% coverage** ≠ 80% of bugs caught. It means 80% of lines ran during tests — they may not have meaningful assertions.
- **Higher isn't always better.** Chasing 100% leads to testing trivial code (getters, console.log) and brittle tests on rapidly-changing code.
- **Aim for ~80–90%** on business logic; lower on glue/IO code that's hard to test.

Better metrics: **mutation testing** (Stryker) — change source code and check if tests fail. Real measure of test quality.

---

### Q245. Flaky tests.

A test that sometimes passes, sometimes fails. Common causes:

1. **Time dependence** — `Date.now()` rounding, timezone, DST. Use `jest.useFakeTimers()` + `setSystemTime`.
2. **Async race** — assertions before async work completes. Always `await` properly.
3. **Test order coupling** — test A leaks state that affects test B. Reset between tests (`beforeEach`).
4. **External dependencies** — network calls, real DB without cleanup. Mock or use Docker.
5. **Random data** — randomness without seeding. Use fixed seeds.
6. **Concurrency** — parallel tests touching the same shared resource.

Fix flakies immediately; don't just retry. A flaky test you ignore today erodes trust in the whole suite.

---

### Q246. Test middleware.

Middleware is just `(req, res, next) => ...`. Test directly:

```js
import { authMiddleware } from "./auth";

test("rejects missing token", () => {
  const req = { get: () => null };
  const res = { status: jest.fn().mockReturnThis(), json: jest.fn() };
  const next = jest.fn();

  authMiddleware(req, res, next);

  expect(res.status).toHaveBeenCalledWith(401);
  expect(next).not.toHaveBeenCalled();
});

test("attaches user and calls next", async () => {
  const req = { get: () => "Bearer valid" };
  const res = {};
  const next = jest.fn();
  jest.spyOn(jwt, "verify").mockReturnValue({ sub: "1" });

  await authMiddleware(req, res, next);

  expect(req.user).toEqual({ sub: "1" });
  expect(next).toHaveBeenCalled();
});
```

---

### Q247. Mock external HTTP calls.

**Option 1 — `nock`** (intercepts real HTTP):
```js
import nock from "nock";

nock("https://api.stripe.com")
  .post("/v1/charges")
  .reply(200, { id: "ch_1", status: "succeeded" });

await chargeCustomer({ amount: 100 });
```

**Option 2 — `msw` (Mock Service Worker)** — works in both Node and browser, clean handler syntax.

**Option 3 — Mock your HTTP client:**
```js
jest.mock("../lib/stripe", () => ({
  charges: { create: jest.fn().mockResolvedValue({ id: "ch_1" }) }
}));
```

Use real HTTP mocking (nock/msw) for integration tests; mock the client wrapper for unit tests.

---

### Q248. `jest.spyOn` vs `jest.fn()`.

- **`jest.fn()`** — creates a brand-new mock function with no implementation.
- **`jest.spyOn(obj, "method")`** — replaces an existing method with a mock, preserving the original (restorable).

```js
const mockFn = jest.fn().mockReturnValue(42);
mockFn(); // 42

const spy = jest.spyOn(console, "log").mockImplementation(() => {});
console.log("anything");  // no output
spy.mockRestore();         // restore original
```

`spyOn` is non-destructive when restored — use it on module exports. `jest.fn()` for injectable dependencies.

---

### Q249. Test DB setup and teardown.

```js
// Per-suite setup
beforeAll(async () => {
  await db.migrate.latest();
});

// Per-test isolation
beforeEach(async () => {
  await db.raw("TRUNCATE users, orders RESTART IDENTITY CASCADE");
});

afterAll(async () => {
  await db.destroy();
});
```

Strategies:
1. **Truncate between tests** — fast, clean.
2. **Transaction per test, rollback** — fastest, but doesn't test side effects of commit (triggers, replication).
3. **Fresh DB per test** — slowest but maximum isolation. Use **testcontainers** to spin a Postgres in Docker.

Never test against production data. Seed minimal fixtures per test.

---

### Q250. Snapshot tests — backend APIs.

```js
test("formatted invoice", () => {
  expect(formatInvoice(order)).toMatchSnapshot();
});
```

Useful for:
- Email template rendering.
- Error response structure.
- Complex object transformations.

**Pitfalls:**
- Snapshots get out-of-date silently if devs blindly press `--updateSnapshot`.
- Brittle if data includes timestamps/IDs (mock them).
- Large snapshots become unreadable; prefer inline (`toMatchInlineSnapshot`).

Use sparingly — explicit assertions are usually clearer than snapshots.

---

### Q251. Test scheduled jobs.

```js
import { job } from "./reportJob";

test("generates report at midnight", async () => {
  jest.useFakeTimers();
  jest.setSystemTime(new Date("2026-05-20T23:59:59Z"));

  const spy = jest.spyOn(reports, "generate").mockResolvedValue();
  job.start();
  jest.advanceTimersByTime(60_000); // tick past midnight
  await Promise.resolve();           // flush microtasks

  expect(spy).toHaveBeenCalled();
  job.stop();
  jest.useRealTimers();
});
```

For the job's actual work, extract it into a plain async function and test that directly. Test the scheduler wiring separately (or skip it — schedulers are usually thin).

---

### Q252. Property-based testing.

Instead of fixed examples, generate hundreds of random inputs and check invariants:

```js
import fc from "fast-check";

test("reverse(reverse(x)) === x", () => {
  fc.assert(
    fc.property(fc.array(fc.integer()), (arr) => {
      expect(reverse(reverse(arr))).toEqual(arr);
    })
  );
});
```

Property tests excel at finding edge cases (empty arrays, max ints, Unicode, etc.) you'd never think to write. Good for: parsers, math, encoding/decoding, sorters.

Library: `fast-check` (JS).

---

### Q253. Test WebSocket connections.

```js
import { io as Client } from "socket.io-client";
import { createServer } from "http";
import { Server } from "socket.io";

let server, io, port;
beforeAll((done) => {
  server = createServer();
  io = new Server(server);
  io.on("connection", (s) => s.on("ping", (cb) => cb("pong")));
  server.listen(() => { port = server.address().port; done(); });
});
afterAll(() => server.close());

test("pings get pong", (done) => {
  const client = Client(`http://localhost:${port}`);
  client.emit("ping", (msg) => {
    expect(msg).toBe("pong");
    client.close();
    done();
  });
});
```

For complex flows, use `ws` test clients with proper async/await. Test reconnection logic with a server that briefly closes.

---

### Q254. Load testing — tools and metrics.

**Tools:** `k6` (modern, JS scripts), `artillery`, `wrk`, `autocannon`, JMeter (legacy).

```js
// k6 script
import http from "k6/http";
export const options = { vus: 100, duration: "1m" };
export default () => http.get("https://api.example.com/users");
```

**Metrics to watch:**
- **Throughput (RPS)** — sustainable requests per second.
- **Latency** — p50, p95, p99 (NOT average — p99 captures the worst 1% experienced by real users).
- **Error rate** — should stay low under target load.
- **Resource saturation** — CPU, memory, DB connections.
- **Breakpoint** — at what RPS does the system start failing?

Profile **before** load testing — fix obvious issues first. Compare staging numbers against production capacity.

---

### Q255. 15-minute test suite — speed up.

1. **Parallelize** — `jest --maxWorkers=auto` runs tests in parallel processes.
2. **Sharding in CI** — split suite across N runners (`jest --shard 1/4`).
3. **Don't test what you don't need** — drop snapshot tests on rapidly changing UI, remove redundant tests.
4. **Mock heavy dependencies** — never hit real S3, third-party APIs, or sleep timers.
5. **Use SWC/esbuild for transpilation** instead of slow ts-jest.
6. **In-memory DB** for tests where appropriate (sqlite, pg-mem) — but watch for divergence from production behavior.
7. **Skip flaky/slow tests in PR runs** — gate them on nightly.
8. **Profile the suite** — `jest --verbose` shows per-test times; find the 10% taking 90% of time.

Goal: <2 min for unit, <5 min for integration. Faster feedback = more frequent runs.

---

## DevOps & Deployment

### Q256. Container vs VM.

| Container | VM |
|---|---|
| Shares host kernel | Full OS per VM |
| MBs in size | GBs in size |
| Starts in seconds | Boots in minutes |
| Process isolation (namespaces, cgroups) | Hardware-level isolation |
| Lighter for microservices | Stronger security boundary |

Both isolate workloads; containers trade some isolation for huge efficiency gains. Modern apps prefer containers; VMs for multi-tenancy, legacy OS support, or stronger security (hypervisor escape harder than container escape).

---

### Q257. Production Dockerfile for Node.js + Express.

```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --include=dev
COPY . .
RUN npm run build && npm prune --production

# Runtime stage
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
RUN apk add --no-cache tini && addgroup -S app && adduser -S app -G app
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
COPY --from=builder --chown=app:app /app/dist ./dist
COPY --from=builder --chown=app:app /app/package.json ./
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -q -O- http://localhost:3000/healthz || exit 1
ENTRYPOINT ["/sbin/tini","--"]
CMD ["node","dist/server.js"]
```

Key practices: multi-stage (smaller image), non-root user, tini as PID 1 (proper signal handling), explicit `EXPOSE`, healthcheck, prod-only deps in final stage.

---

### Q258. Multi-stage Docker build.

Stages share a single Dockerfile but only the final stage gets shipped:

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
CMD ["node","dist/server.js"]
```

**Benefits:**
- Smaller final image — no dev tools, no source, no cache layers from build.
- Faster builds — only changed stages rebuild.
- Cleaner separation: deps → build → runtime.

A typical Node image drops from 1.2GB → ~200MB this way.

---

### Q259. `docker-compose` vs Kubernetes.

| docker-compose | Kubernetes |
|---|---|
| Single-host orchestration | Multi-host cluster |
| Dev / small prod | Production at scale |
| YAML defines services | YAML defines deployments, services, ingresses |
| No auto-scaling, no self-healing | Auto-scaling, self-healing, rolling updates |
| `docker compose up` | `kubectl apply -f` |

**Use compose for:** local dev (frontend + backend + Postgres + Redis stack), CI test environments, simple single-server production.
**Use k8s for:** multi-instance production, auto-scaling, blue-green/canary deploys, complex microservice landscapes.

For most startups: managed compose alternatives (Fly.io, Railway, Render) hide k8s complexity. Reach for raw k8s when you outgrow them.

---

### Q260. Health check endpoint.

```js
app.get("/healthz", (req, res) => res.send("ok"));    // liveness

app.get("/readyz", async (req, res) => {              // readiness
  try {
    await db.query("SELECT 1");
    await redis.ping();
    res.send("ok");
  } catch (err) {
    res.status(503).send("not ready");
  }
});
```

- **Liveness** — is the process alive? Lightweight; don't check dependencies (else dependency failure cascades restarts).
- **Readiness** — is the process ready to serve traffic? Check dependencies — if DB down, return 503 so LB stops routing.

Add a `/health/deep` that touches all deps for ops dashboards; keep `/healthz` ultra-light for k8s probes.

---

### Q261. Liveness vs readiness probe.

| | Liveness | Readiness |
|---|---|---|
| Question | Is the process alive? | Can it serve traffic? |
| Failure action | Restart pod | Remove from LB rotation |
| Recommended endpoint | `/healthz` (cheap) | `/readyz` (checks deps) |

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 3000 }
  initialDelaySeconds: 10
  periodSeconds: 30
readinessProbe:
  httpGet: { path: /readyz, port: 3000 }
  periodSeconds: 10
```

If liveness fails: k8s kills and restarts. If readiness fails: k8s stops routing to the pod but leaves it alive (gives it a chance to recover).

Common bug: making liveness depend on the DB → DB hiccup causes pod restart storms. Keep liveness independent.

---

### Q262. Reverse proxy with Nginx.

```nginx
upstream nodeapp {
  server app1:3000;
  server app2:3000;
}

server {
  listen 443 ssl http2;
  server_name api.example.com;
  ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

  client_max_body_size 10m;

  location / {
    proxy_pass http://nodeapp;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 30s;
  }
}
```

Nginx terminates TLS, balances load, serves static, gzip-compresses, caches, and shields Node from slow clients (Slowloris). Node sits behind on plain HTTP.

In Express: `app.set("trust proxy", 1)` so `req.ip` reflects the real client IP.

---

### Q263. Nginx — SSL termination.

```nginx
server {
  listen 443 ssl http2;
  ssl_certificate     /etc/letsencrypt/live/api/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api/privkey.pem;
  ssl_protocols       TLSv1.2 TLSv1.3;
  ssl_ciphers         HIGH:!aNULL:!MD5;
  ssl_session_cache   shared:SSL:10m;

  location / { proxy_pass http://node:3000; }
}

# Redirect 80 → 443
server {
  listen 80;
  return 301 https://$host$request_uri;
}
```

Use Let's Encrypt + certbot for free auto-renewing certs:
```bash
certbot --nginx -d api.example.com
```

Use Mozilla SSL Configurator for modern cipher suites.

---

### Q264. Load balancer — L4 vs L7.

- **L4 (Transport)** — load-balances TCP/UDP packets. Doesn't see HTTP. Fast. Examples: AWS NLB, HAProxy in TCP mode.
- **L7 (Application)** — understands HTTP. Can route by path/header/cookie, terminate TLS, apply WAF rules. Examples: AWS ALB, Nginx, Envoy.

| | L4 | L7 |
|---|---|---|
| Protocol | TCP/UDP | HTTP/HTTPS/gRPC/WS |
| Routing rules | IP/port only | Path, host, header, cookie |
| TLS | Pass-through or termination | Termination |
| Speed | Very fast | Slightly slower |

Most modern web APIs benefit from L7 — path-based routing to microservices, header-based canary releases, WAF. L4 for raw TCP services (DBs, MQs) or when you need maximum throughput.

---

### Q265. Horizontal vs vertical for stateless backends.

**Horizontal wins for stateless backends:**
- Add more nodes instead of bigger nodes → linear scale, no ceiling.
- Built-in redundancy (one node dies, others continue).
- Cheaper (small instances + many vs one huge box).
- Rolling deploys with zero downtime.

Stateless requirement: store state externally (DB, Redis), not in process memory. Then any instance can handle any request. Use sticky sessions only as a last resort.

Vertical scaling fits stateful services (databases) where horizontal scaling requires sharding/replication overhead.

---

### Q266. CI/CD pipeline stages.

```
Push → Build → Test → Security → Package → Deploy (staging) → Smoke test → Deploy (prod)
```

**Typical stages:**
1. **Checkout** code.
2. **Install** deps (`npm ci`).
3. **Lint** + **format check**.
4. **Type-check** (`tsc --noEmit`).
5. **Unit tests** (parallel).
6. **Integration tests** (with services).
7. **Security**: `npm audit`, Snyk, secret scan.
8. **Build** artifacts (Docker image).
9. **Push** to registry.
10. **Deploy staging** + smoke test.
11. **Approval** gate.
12. **Deploy production** (rolling/canary).
13. **Notify** (Slack, email).

Tools: GitHub Actions, GitLab CI, CircleCI, Jenkins, Buildkite.

---

### Q267. Zero-downtime — rolling vs blue-green.

**Rolling update:**
- Replace pods/instances one at a time.
- Each new instance proven healthy before the next.
- No need for double capacity.
- Slower; mixed versions during deploy (consider DB compatibility).

**Blue-Green:**
- Two full environments (blue = live, green = new).
- Deploy to green, smoke-test, then swap traffic.
- Instant rollback (swap back to blue).
- Double capacity required during deploy.
- Clean cutover; no mixed-version state.

Rolling is the default for k8s `Deployment`. Blue-green is best for risky upgrades or compliance environments.

---

### Q268. Canary deployment.

Route a small % of traffic (e.g., 5%) to the new version. Monitor error rate, latency, business metrics. If healthy → ramp to 25% → 50% → 100%. If degraded → roll back.

```yaml
# Istio VirtualService example
- route:
  - destination: { host: app, subset: v1 }
    weight: 95
  - destination: { host: app, subset: v2 }
    weight: 5
```

Best for risky changes:
- Algorithm changes (recommendation engine).
- DB migrations (catch issues on small subset).
- Library upgrades.

Tools: Argo Rollouts, Flagger, LaunchDarkly (feature flag based). Always pair with strong observability — without metrics, canary is just gambling.

---

### Q269. Env-specific configs.

**Layered approach:**
1. **Base config** committed (`config/default.js`).
2. **Env overrides** committed for non-sensitive (`config/production.js`).
3. **Secrets** never committed — injected via env vars from secret manager.

```js
const env = z.object({
  NODE_ENV: z.enum(["dev","staging","prod"]),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
}).parse(process.env);
```

**Per-environment principles:**
- Same image, different config. Don't bake env into the image.
- Validate on boot — crash early on missing config.
- Single source of truth — secrets in Vault/AWS Secrets Manager, not scattered.
- Local dev uses `.env.example` as a template.

---

### Q270. Secret manager.

A centralised vault for secrets with access control + audit:

- **AWS Secrets Manager** — auto-rotation for RDS/Redshift, IAM-controlled access.
- **GCP Secret Manager** — versioned secrets, IAM integration.
- **HashiCorp Vault** — self-hosted, multi-cloud, dynamic secrets (short-lived DB creds).
- **Doppler** — developer-friendly, multi-env UI.

**Workflow:**
1. Store secret in manager.
2. App at startup fetches via SDK (or sidecar injects as env var).
3. Rotate keys periodically; SDK re-fetches.

Never store production secrets in `.env` files on disk, in source control, or in CI variables exposed to fork PRs.

---

### Q271. Structured logging.

```js
// Plain text
logger.info("User 123 logged in from 1.2.3.4 in 50ms");

// Structured
logger.info({
  event: "user.login",
  userId: 123,
  ip: "1.2.3.4",
  durationMs: 50,
  requestId: req.id,
}, "user logged in");
```

**Why structured (JSON) wins:**
- Queryable in ELK/Datadog/Loki/CloudWatch.
- Alerting on fields (`status >= 500`).
- Aggregations (count errors per endpoint per minute).
- No regex parsing of free-form text.

Use **pino** for performance, **winston** for flexibility. Always include a `requestId` and `userId` (when known) on every log line.

---

### Q272. Distributed tracing.

Each request gets a **trace id**; each operation (DB query, HTTP call) is a **span** with start/end times. Spans link to form a tree showing the full path of a request across services.

```
[Frontend] → [API Gateway] → [User Service] → [Postgres]
                          → [Order Service] → [Stripe API]
```

**Implementation:**
- **OpenTelemetry** (vendor-neutral standard).
- Instrument HTTP, DB, Redis automatically.
- Propagate trace context via `traceparent` header.
- Backends: Jaeger, Tempo, Datadog APM, Honeycomb, New Relic.

```js
import { trace } from "@opentelemetry/api";
const span = trace.getTracer("app").startSpan("processOrder");
try { /* work */ } finally { span.end(); }
```

Vital for debugging latency in microservices — without it, you're guessing.

---

### Q273. APM — metrics to track.

**APM** = Application Performance Monitoring. Tools: Datadog, New Relic, Dynatrace, Sentry (errors), Honeycomb (events).

**Golden signals (SRE):**
1. **Latency** — p50, p95, p99 response times.
2. **Traffic** — requests per second.
3. **Errors** — 5xx rate, exception count.
4. **Saturation** — CPU, memory, queue depth.

**Additional:**
- Per-endpoint breakdown.
- Database query times.
- External dependency latency (Stripe, S3).
- Cache hit ratio.
- Background job lag.
- Business metrics (signups/min, checkouts/min).

Set alerts on **SLOs** (Service Level Objectives), not raw thresholds — e.g., "99.5% of requests under 500ms over 30 days."

---

### Q274. High CPU usage — diagnose and fix.

1. **Confirm** via APM / `top` / k8s metrics — is it sustained or a spike?
2. **Identify which process** — `pidstat`, `kubectl top pod`.
3. **CPU profile** — `node --prof app.js` (run for ~30s under load), then `--prof-process` → flame graph.
4. **Modern**: `clinic flame app.js`, `0x app.js`, or attach Chrome DevTools profiler.
5. **Inspect the flame graph** — wide bars = hot functions.

**Common culprits:**
- JSON.parse/stringify on huge payloads.
- Regex catastrophic backtracking.
- Synchronous bcrypt / crypto on the main thread.
- Inefficient loops on large arrays.

**Fixes:** algorithmic improvement, move to worker_threads, cache results, batch external work. Last resort: scale horizontally.

---

### Q275. 12-factor app — 5 factors.

The [12-factor app](https://12factor.net) is a methodology for cloud-native apps. Five most-applied:

1. **Codebase** — one codebase tracked in version control; many deploys (dev/staging/prod from the same repo).
2. **Dependencies** — declared explicitly (`package.json`) and isolated (no system-wide deps).
3. **Config** — store config in environment variables, not code (per Q13/Q269).
4. **Backing services** — DBs, queues, caches are attached resources accessed via URL — swappable without code changes.
5. **Stateless processes** — share-nothing; persist state to backing services. Enables horizontal scaling.

Others: build/release/run separation, port binding, concurrency (process model), disposability (fast startup/shutdown), dev/prod parity, logs as event streams, admin processes as one-off.

Following these = trivially deployable to k8s / Heroku / Fly / Render.

---

## Scenario-Based Questions

> Use the **Identify → Diagnose → Fix → Prevent** framework for each.

### Q276. `GET /products` returns 200 with empty array, but DB has data.

**Identify:** the route works, query returns nothing for the caller — disconnect between request shape and stored data.

**Diagnose:**
1. Log the exact query being executed (Mongoose `debug: true` / pg `query` event).
2. Run the same query directly against the DB. If it returns data → query in code is different.
3. Check tenancy/filtering — is there a hidden `WHERE` (`organizationId`, `deletedAt: null`, `status`)?
4. Check connection — pointing at the wrong DB (staging vs prod, replica with lag)?
5. Auth context — does middleware scope the query by user/org? `req.user.orgId` set correctly?
6. ORM lazy population not awaited?

**Fix:** correct the filter / connection / scoping.

**Prevent:** integration tests covering happy path with seeded data; log effective queries in dev; assert connection target on boot.

---

### Q277. Intermittent 502 from Nginx + Node.

**Identify:** 502 = "Bad Gateway" — Nginx couldn't get a valid response from upstream.

**Diagnose:**
1. **Node crashed / restarting** — check `journalctl` / k8s `kubectl describe pod` for OOMKilled or crash loops.
2. **Connection refused** — port not bound (deploy still booting).
3. **`upstream prematurely closed connection`** in Nginx log — Node's `keepAliveTimeout` < Nginx's; Node closes mid-request. Fix: `keepAliveTimeout > Nginx proxy_read_timeout`.
4. **Slow request → Nginx timeout** — `proxy_read_timeout` too short; increase or fix slow endpoint.
5. **Worker thread blocked** — long sync op; check event loop lag (`@isaacs/perf_hooks`).
6. **OOM under load** — heap snapshot.

**Fix:** align timeouts (`keepAliveTimeout = 65s`, Nginx `proxy_read_timeout = 60s`), restart loop investigation, scale resources.

**Prevent:** alert on Node crash count, event-loop-lag metric, proper graceful shutdown.

---

### Q278. `POST /order` called twice due to retry — duplicate orders.

**Identify:** non-idempotent endpoint + client retry on network failure.

**Diagnose:** check whether the second request arrived (logs by request id) — if yes, server processed both. Common after timeouts or 5xx.

**Fix — Idempotency-Key pattern:**
```
Client → POST /order   Idempotency-Key: uuid-v4-from-client
Server: look up key in Redis/DB
  - If exists → return stored response
  - If not → process, store result with key, return
```

```js
async function placeOrder(req, res) {
  const key = req.get("Idempotency-Key");
  if (!key) return res.status(400).json({ error: "Idempotency-Key required" });
  const existing = await redis.get(`idem:${key}`);
  if (existing) return res.json(JSON.parse(existing));

  const lock = await redis.set(`idem:${key}:lock`, "1", { NX: true, EX: 30 });
  if (!lock) return res.status(409).json({ error: "in progress" });

  const result = await createOrder(req.body);
  await redis.setEx(`idem:${key}`, 86400, JSON.stringify(result));
  res.json(result);
}
```

**Prevent:** make every state-changing endpoint accept Idempotency-Key; document for clients; back-end retries also pass through.

---

### Q279. Error rate jumped 0.1% → 15% after deploy. Rollback.

**Immediate (< 5 min):**
1. **Stop the bleed** — initiate rollback to last known-good version.
   - k8s: `kubectl rollout undo deployment/api`.
   - Blue-green: flip traffic back to blue.
2. **Announce** in incident channel; assign incident commander.
3. **Confirm rollback** — error rate dropping in dashboards.

**Diagnose (after stabilising):**
1. Pull error logs from the bad window — group by exception type.
2. Compare diff between failing release and last good (commits, configs, migrations).
3. Reproduce locally or in staging with prod-like data.

**Fix:** patch the issue; add a test that would have caught it; re-deploy with canary + gradual rollout.

**Prevent:** smaller deploys; canary with auto-rollback on error-rate threshold; pre-deploy load tests; better staging parity.

---

### Q280. User A's data showing in User B's response intermittently.

**Identify:** classic **cache key collision** or **shared mutable state** between requests. Critical security bug.

**Diagnose:**
1. **Per-request mutable state in module scope** — most likely culprit. Variables declared at module top level shared across requests.
   ```js
   // ❌ shared across all requests
   let currentUser;
   app.use((req, res, next) => { currentUser = req.user; next(); });
   ```
2. **Wrong cache key** — `cache:user` instead of `cache:user:${id}`.
3. **CDN/proxy caching responses without `Vary: Authorization`** — same URL returns one user's data to others.
4. **Race condition in shared connection** — one DB connection holding state for two parallel requests.

**Fix:** scope all per-request data via closures, `req` object, or `AsyncLocalStorage`. Ensure cache keys are user-scoped. Set `Cache-Control: private` and `Vary` headers.

**Prevent:** code review for module-scope mutables; integration tests with parallel users; static analysis for global mutations.

---

### Q281. 100 req/s fine, 500 req/s slow — CPU fine but DB connections maxed.

**Identify:** DB connection pool is the bottleneck, not Node.

**Diagnose:**
- Check `pg_stat_activity` — many `idle in transaction`? long-running queries blocking?
- App pool: `max: 10` × N instances = X connections. Is X close to PG `max_connections`?
- Per-request DB call count — does each request open multiple connections (transaction)?

**Fix:**
1. **PgBouncer** in transaction mode — proxy that multiplexes thousands of client connections onto few server connections. Single biggest win for Postgres scaling.
2. **Increase pool size** moderately (each connection = ~10MB RAM on PG).
3. **Optimize slow queries** — add indexes, reduce per-request query count.
4. **Cache hot reads** in Redis to skip DB entirely.
5. **Read replicas** for GET endpoints.

**Prevent:** monitor pool saturation in APM; alert when waiting > X ms.

---

### Q282. Migrate `VARCHAR(50)` → `TEXT` on 50M rows, no downtime.

In modern Postgres, `VARCHAR(n) → TEXT` is metadata-only — no rewrite. But the safe approach:

```sql
-- 1. PG 9.2+ allows ALTER COLUMN TYPE to TEXT without rewrite (no length check needed going to TEXT)
ALTER TABLE big ALTER COLUMN col TYPE TEXT;
-- Quick on modern PG; takes brief ACCESS EXCLUSIVE lock — done in ms.
```

If the column actually changes storage representation (e.g., `INT → BIGINT`), use the expand-migrate-contract pattern:

1. **Add new column** `col_new TEXT`.
2. **Backfill in batches** + dual-write via trigger.
3. **Switch reads** to `col_new` (deploy).
4. **Stop writes to old column** (deploy).
5. **Drop old column** (later release).

**Tools:** `pg_repack`, `pg-osc`, `gh-ost` (for MySQL). Always test the exact migration on a production-sized clone first.

**Prevent:** size columns reasonably from day one; avoid `VARCHAR(n)` constraints unless meaningful — `TEXT` is the same speed in PG.

---

### Q283. Payment webhook hitting endpoint multiple times.

**Identify:** providers (Stripe, PayPal) retry webhooks on non-2xx. Duplicates are expected; idempotency is required.

**Diagnose:** are the payloads identical (same `event.id`)? If yes → standard retry. If different → multiple events for the same logical change.

**Fix:**
```js
app.post("/webhook/stripe", async (req, res) => {
  // 1. Verify signature
  const event = stripe.webhooks.constructEvent(req.rawBody, req.get("Stripe-Signature"), SECRET);

  // 2. Idempotency check
  const ok = await db.events.insertIgnore({ id: event.id, type: event.type, payload: event });
  if (!ok.inserted) return res.json({ ok: true });  // already processed

  // 3. Process
  await handle(event);
  res.json({ ok: true });
});
```

Acknowledge fast (return 2xx), process asynchronously if heavy. Verify signatures so attackers can't spoof.

**Prevent:** every webhook handler must dedupe on event id; store events for audit; replay support via UI.

---

### Q284. Memory leak: 200MB → 1.5GB over 6 hours.

**Identify:** classic Node memory leak — slow, monotonic growth.

**Diagnose:**
1. **Confirm pattern** — `process.memoryUsage().heapUsed` over time.
2. **Take heap snapshots** — `kill -USR2` on Node with `--inspect`, or `v8.writeHeapSnapshot()` programmatically; capture at hour 0, 2, 4.
3. **Open in Chrome DevTools** → Memory tab → "Objects allocated between snapshots." Sort by retained size.
4. **Identify suspect retainers** — likely arrays, maps, listener arrays, closure scopes.

**Common causes:**
- Unbounded in-memory cache (Map without LRU).
- Event listeners added per request, never removed.
- `setInterval` not cleared.
- Closures over large objects in long-lived callbacks.
- Promises piling up (queue without backpressure).

**Fix:** bound the growth source (LRU, `setMaxListeners`, clear timers, weak refs).

**Prevent:** load-test deploys; memory alerts; periodic process restart as a safety net.

---

### Q285. Architect 1M-email marketing campaign.

**Don't:** `for (user of users) await sendEmail(user)` — single point of failure, no retry, takes hours.

**Architecture:**
```
[Campaign service] → push 1M jobs → [Queue (SQS / RabbitMQ / Redis Streams)]
                                      ↓
                                  [Worker pool: 50 workers, each: throttle to provider limit]
                                      ↓
                                  [SES / Sendgrid / Postmark API]
                                      ↓
                                  Webhook events back → tracking DB
```

**Key points:**
1. **Queue-driven** — durable, replayable, parallelizable.
2. **Throttle workers** to stay under provider rate limit.
3. **Provider with high throughput** (SES, Sendgrid, Mailgun).
4. **Per-recipient retries** with exponential backoff; permanent fails → suppression list.
5. **Idempotency keys** (campaign_id + user_id) — no duplicate sends if worker crashes.
6. **Track engagement** via webhook events (opens, clicks, bounces).
7. **Throttle aggregate sends** to protect domain reputation (gradual ramp).
8. **Unsubscribe** processed real-time to comply with regulations.

---

### Q286. Redis keys collide between dev and prod.

**Identify:** shared Redis instance + numeric IDs (`user:123`) → dev key overwrites prod or vice versa.

**Fix:**
1. **Namespace by environment:** `dev:user:123`, `prod:user:123`. Configure prefix per env.
   ```js
   const prefix = process.env.NODE_ENV;
   await redis.set(`${prefix}:user:${id}`, ...);
   ```
2. **Separate Redis databases** — `SELECT 0` (prod), `SELECT 1` (dev). Crude but works for small setups.
3. **Best — separate Redis instances entirely** for prod vs dev. Cheap with managed Redis. Eliminates the entire class of issue.

**Prevent:** never share infrastructure between environments. Even "dev/staging Redis" should be isolated from prod.

---

### Q287. Two users booking last seat simultaneously — race condition.

**Identify:** read-then-write race. Both read `available=1`, both write `available=0`, both create bookings.

**Fix — atomic update (best):**
```sql
UPDATE seats SET booked_by = $userId, booked_at = now()
WHERE id = $seatId AND booked_by IS NULL
RETURNING *;
```
If 0 rows returned → seat already booked. Single SQL statement, no race window.

**Alternative — `SELECT FOR UPDATE` (pessimistic lock):**
```sql
BEGIN;
SELECT * FROM seats WHERE id = $1 AND booked_by IS NULL FOR UPDATE;
-- check, then UPDATE
COMMIT;
```

**Alternative — distributed lock (Redis):**
```js
const lock = await redis.set(`lock:seat:${seatId}`, userId, { NX: true, EX: 10 });
if (!lock) throw new Error("Seat being booked by another user");
// proceed, then release
```

**Prevent:** never use "read, check, write" patterns for contended resources. Use atomic conditional updates.

---

### Q288. Staging 80ms, prod 1200ms — same load.

**Possible causes:**
1. **Data volume** — prod has 10M rows, staging 10k. Queries that scan happily on staging do `Seq Scan` on prod.
2. **Cold cache** — staging recently re-seeded; prod cache hot/cold patterns differ.
3. **Indexes missing** in prod (a migration didn't run, or index in `CREATE INDEX CONCURRENTLY` stuck).
4. **Resource limits** — prod under noisy-neighbor load on shared infra; CPU steal.
5. **External dependency latency** — third-party APIs slower from prod region.
6. **Connection pool exhausted** — prod has more concurrent users.
7. **Stale statistics** — `ANALYZE` not run; planner picks bad plans.
8. **Network differences** — prod DB in different AZ, adds latency.

**Diagnose:** run `EXPLAIN ANALYZE` on slow queries in prod; check APM for which spans are slow; compare DB sizes and instance types.

**Prevent:** parity in data volume (use prod-like snapshots in staging); load tests against staging with similar concurrency.

---

### Q289. `.env` with prod DB creds committed to public GitHub repo.

**Next 10 minutes — assume credentials are already harvested by bots:**

1. **Rotate the DB password** immediately. Most managed DBs: `ALTER USER appuser WITH PASSWORD 'new'` and update secret manager.
2. **Restrict DB network access** — close to public IPs; allow only known app IPs / VPC.
3. **Rotate every other secret** in the file: API keys, JWT signing keys, third-party tokens.
4. **Check DB audit logs / connections** for unfamiliar IPs already connected.
5. **Audit recent data changes** — look for unauthorized reads/writes.

**Then:**
6. **Remove from history** — `git filter-repo --path .env --invert-paths`, force-push. Note: copies may persist in forks and GitHub caches.
7. **Add `.env` to `.gitignore`**; commit `.env.example` with placeholders.
8. **Enable secret scanning** on the repo (GitHub native, GitGuardian).
9. **Incident write-up** — root cause, timeline, mitigations.

**Prevent:** pre-commit hook (`gitleaks`, `detect-secrets`); use secret manager exclusively; CI gate on secret scans.

---

### Q290. Add required field with 3 older mobile clients live.

**Don't break old clients.** Strategies:

1. **Make the field optional at the API contract level**, even if internally required.
   ```js
   // Server fills a default if client omits it
   const value = req.body.newField ?? "default";
   ```
2. **Backfill defaults** for omitted-field requests; reject for known-good newer client versions.
3. **Header-based version routing** — old clients hit v1 (no requirement); new clients hit v2 (required).
4. **Force-upgrade prompt** in app for very old clients (last resort).

**Tracking:** look at User-Agent / client version distribution; phase deprecation when <X% on old versions.

**Prevent:** treat APIs as forever-backwards-compatible; design schemas to anticipate evolution (every field optional + server defaults).

---

### Q291. Background queue growing — consumer can't keep up.

**Identify:** queue depth growing faster than throughput. Eventually OOM/disk fills.

**Options (in order):**
1. **Scale workers horizontally** — add more consumer instances. Trivial if consumer is stateless.
2. **Increase concurrency** within each worker (process N messages in parallel) — careful with rate limits.
3. **Optimize per-message processing** — profile the consumer; common culprits: synchronous I/O, DB N+1.
4. **Batch processing** — pull 100 messages, process in one DB call.
5. **Shed load** — drop low-priority messages or older-than-X TTL.
6. **Split queue** — high-priority vs low-priority; dedicated workers per class.
7. **Async I/O upstream** — if consumer waits on a slow downstream (e.g., email provider), use that provider's batch API.

**Prevent:** monitor queue depth and consumer lag; auto-scale workers on depth thresholds; load-test pipelines.

---

### Q292. User logs in, session expires immediately. Others fine.

**Identify:** session/JWT issue isolated to one user — data-specific, not infrastructure.

**Diagnose:**
1. **Clock skew** on user's device — JWT `iat`/`exp` rejected as already expired.
2. **JWT payload** for that user contains an invalid `exp` (corrupted on issue).
3. **User's session record** in Redis exists but has TTL 0 or past.
4. **`SameSite=Strict` cookie** + redirect — cookie not sent on cross-site landing.
5. **Multiple devices** — login from device B invalidates device A (rotating refresh tokens with reuse detection).
6. **Account flagged for re-auth** — admin or security event.

**Fix:** depends on cause. Get the JWT from the user, decode, check `exp` and `iat`. Compare server time. Check session store.

**Prevent:** log session lifecycle events (issued, refreshed, invalidated, expired); have a "session debug" admin view.

---

### Q293. PDF report taking 30s — without blocking API.

```
POST /reports/generate
  → 202 Accepted, returns { jobId }
  → enqueue job in BullMQ / SQS
GET /reports/{jobId}
  → 200 { status: "processing" } | { status: "done", url: "https://s3/..." }
WebSocket / SSE
  → push "report ready" event
```

**Implementation:**
1. Validate request synchronously, persist job row (status: queued), return 202 with job id.
2. Worker picks up the job; generates PDF (Puppeteer / PDFKit / Headless Chrome).
3. Upload to S3; update job row with URL.
4. Notify client (poll, webhook, email link, WS).

**Robustness:** retries on failure, dead-letter for poison jobs, timeout per job, per-user concurrency limit to prevent abuse.

**Prevent:** never do >2s sync work in a request handler.

---

### Q294. `DELETE /users/:id` deletes wrong users due to race.

**Likely cause:** multi-step delete (delete posts → delete sessions → delete user) without isolation; concurrent request mutates IDs mid-process; missing transaction.

**Diagnose:**
- Audit log: trace which user's data deleted, by which request, in what order.
- Reproduce locally with concurrent delete calls.
- Check whether the lookup-then-delete uses stale references (`user = await find(id); ...; delete user.someRelated`).

**Fix — single atomic transaction:**
```js
await db.transaction(async (tx) => {
  await tx.posts.deleteMany({ where: { authorId: id } });
  await tx.sessions.deleteMany({ where: { userId: id } });
  await tx.users.delete({ where: { id } });
});
```

Or **cascade FKs**: `ON DELETE CASCADE` on posts/sessions referencing users.

For irreversible operations, also:
- **Soft delete** first; hard delete via background job after verification window.
- **Confirmation tokens** for destructive actions.
- **Idempotency** — repeating delete is no-op, not "delete next user."

**Prevent:** review every destructive endpoint for atomicity; audit log every deletion.

---

### Q295. External weather API limit 100/min hit by 1000 users/min.

**Identify:** rate limit downstream.

**Fix — shared cache:**
```js
// Cache results for 5 minutes (weather doesn't change second-to-second)
const key = `weather:${city}`;
let data = await redis.get(key);
if (!data) {
  data = JSON.stringify(await weatherApi.fetch(city));
  await redis.setEx(key, 300, data);
}
```

If 1000 users in 1 min all want weather for the same 50 cities → 50 unique calls/min (within limit). For more cities:

- **Singleflight** — first request fetches; others wait on the same in-flight promise (no duplicate calls).
- **Stale-while-revalidate** — serve cached even if expired; refresh in background.
- **Bulk endpoint** — if the upstream supports `GET /weather?cities=a,b,c`, fan-in.
- **Upgrade the plan** — if business justifies.
- **Backoff with circuit breaker** when 429s appear.

**Prevent:** treat all external APIs as rate-limited; always cache + circuit-break.

---

### Q296. Audit trail for every change — without touching every endpoint.

**Approach 1 — DB triggers:**
```sql
CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_log_change();
```
Trigger writes before/after row + user id (from session var) to an audit table. No app code change.

**Approach 2 — ORM hooks/middleware:**
- Mongoose `pre/post` middleware on every model.
- Prisma extensions (`query` middleware).
- Centralised — apply once at the ORM layer.

**Approach 3 — CDC (Change Data Capture):**
Debezium reads Postgres WAL and emits events for every change. Consume into an audit store. Decouples audit from app.

**Approach 4 — Event-sourcing:**
All mutations go through commands; an event log is the source of truth. Audit free, but invasive redesign.

For most apps: approach 2 (ORM middleware) is the best balance. Store `who, what, when, before, after` with diff.

**Prevent:** decide audit strategy at architecture time, not as an afterthought.

---

### Q297. Query 50ms isolated, 2s in app.

**Possible reasons:**
1. **ORM overhead** — Mongoose hydration, Sequelize instance creation on 10k rows is heavy.
2. **N+1 queries** within the function — the "one query" is 50ms but the loop runs it 40 times.
3. **Connection acquisition delay** — pool exhausted; waiting for a free connection.
4. **Network latency** — DB in another region/AZ; per-roundtrip cost.
5. **Serialization** — converting large result sets to JSON.
6. **Middleware between query and response** — extra processing, transforms.
7. **Statistics differ** — isolated query runs with hot cache; app run cold.
8. **Connection pool warm-up** — first query on a new connection slower (TLS handshake, prepare).

**Diagnose:** add timing around the query call only (`console.time`); use APM spans (`db.query` vs full request). Compare.

**Fix:** depending on cause — `.lean()` in Mongoose, eager loading, raise pool size, batch operations, co-locate DB.

---

### Q298. Feature flag system — design backend.

**Schema:**
```js
flags: { id, name, enabled, rolloutPercent, conditions: {...}, createdAt }
flag_overrides: { flagName, userId, enabled }
```

**Evaluation rule (server-side):**
```js
async function isEnabled(flagName, user) {
  // 1. User-specific override
  const override = await db.flagOverrides.findOne({ flagName, userId: user.id });
  if (override) return override.enabled;

  // 2. Conditions (org, plan, region, custom attrs)
  const flag = await cache.get(flagName) ?? await db.flags.findByName(flagName);
  if (!flag.enabled) return false;
  if (!matchesConditions(flag.conditions, user)) return false;

  // 3. Percentage rollout — stable hash so user is consistently in/out
  const hash = murmur3(`${flagName}:${user.id}`);
  return (hash % 100) < flag.rolloutPercent;
}
```

**Operational:**
- **Cache flags** in Redis; invalidate on change (sub-second propagation).
- **Pub/sub** to notify all app instances on flag update.
- **Local fallback** — if Redis down, use last-known config.
- **Audit log** of flag changes (who, what, when).
- **SDK** for use in code: `if (await flags.on("new-checkout", user))`.

**Tools:** LaunchDarkly, Unleash (open source), Flagsmith. Build only if you have specific needs.

---

### Q299. Image processing in upload request times out.

**Current (broken):**
```js
app.post("/avatar", upload.single("file"), async (req, res) => {
  const resized = await sharp(req.file.path).resize(200).toBuffer(); // slow
  // ... times out
});
```

**Redesigned (async):**
```
1. Client POST /avatar/upload-url → { presignedUrl, jobId }
2. Client uploads directly to S3 using presigned URL.
3. Client POST /avatar/process { jobId } → 202 Accepted.
4. Backend enqueues job in queue (BullMQ / SQS).
5. Worker:
   - Downloads from S3
   - Resizes, compresses, watermarks (sharp / ImageMagick)
   - Uploads variants (thumb, medium, large) to S3
   - Updates user.avatarUrl
   - Optionally notifies client (WebSocket, push notification)
6. Client polls /avatar/status/{jobId} or receives event.
```

**Benefits:**
- API responds instantly (202).
- Image processing scales independently (worker pool).
- Failures isolated; retries automatic.
- No request timeout.

**Prevent:** anything > 2s sync → make it async from day one.

---

### Q300. Onboarding to legacy Node.js codebase with no tests, no docs, many bugs — two-week plan.

**Week 1 — Understand & Stabilise:**

- **Day 1–2: Read & map.**
  - Run the app locally; trace happy paths in the debugger.
  - Sketch the architecture (services, dependencies, data flow).
  - Read `package.json`, top-level files, README.
  - Pair with someone who's used the system; record knowledge in a doc.

- **Day 3–4: Observability.**
  - Add structured logging at request boundaries if not present.
  - Capture error rates, latency by endpoint (basic APM / Sentry).
  - Identify the top 5 bug-causing endpoints from logs.

- **Day 5: Smoke / golden-path tests.**
  - Write 5–10 integration tests covering the most critical user flows (login, primary action, billing).
  - These become your safety net.

**Week 2 — Improve incrementally:**

- **Day 6–8: Fix the top 3 bugs.**
  - Reproduce, write a failing test, fix, ship.
  - Document each fix briefly.

- **Day 9–10: Foundations.**
  - Add lint + format if missing.
  - Set up CI to run the tests you wrote.
  - Add `.env.example`; document setup steps.

- **Day 10 — Knowledge sharing.**
  - Write an `ARCHITECTURE.md` summarising what you've learned.
  - Present findings to the team; identify the next 30/60/90 day improvements.

**Mindset:**
- Don't rewrite. Stabilise first.
- Touch only what you can verify with tests.
- Document as you go; future-you will thank you.
- Resist scope creep — fix bugs in scope, file tickets for the rest.

---

## 💡 Answer Template for Scenarios

For any scenario question in an interview, structure your answer as:

1. **Identify** — name the type of problem (race condition, cache miss, memory leak, etc.).
2. **Diagnose** — tools/logs/queries you'd use to confirm the root cause.
3. **Fix** — immediate mitigation + correct long-term solution.
4. **Prevent** — process/monitoring change so it doesn't recur.

---

## 🚀 Quick Revision Tracker

| Section | Range | Confident | Needs Review |
|---|---|---|---|
| Node.js Core | Q1–Q40 | ☐ | ☐ |
| Express.js | Q41–Q70 | ☐ | ☐ |
| SQL | Q71–Q110 | ☐ | ☐ |
| MongoDB | Q111–Q140 | ☐ | ☐ |
| REST API | Q141–Q170 | ☐ | ☐ |
| Auth & Security | Q171–Q200 | ☐ | ☐ |
| Caching & Redis | Q201–Q220 | ☐ | ☐ |
| Message Queues | Q221–Q235 | ☐ | ☐ |
| Testing | Q236–Q255 | ☐ | ☐ |
| DevOps | Q256–Q275 | ☐ | ☐ |
| Scenarios | Q276–Q300 | ☐ | ☐ |

---

*Backend Engineer Interview Prep | 300 Questions with detailed solutions | Good luck! 🚀*
