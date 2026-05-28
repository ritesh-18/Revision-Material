# 01 — Node.js, JavaScript & TypeScript (40 Questions)

These cover the runtime, async model, streams, modules, GC, and TypeScript that everything else in your stack sits on top of. If you can't answer these crisply, NestJS / Postgres / Redis answers won't land either.

**Sections**
- Part A — JavaScript fundamentals (Q1–Q10)
- Part B — Async, streams, concurrency (Q11–Q18)
- Part C — Node.js internals & ecosystem (Q19–Q26)
- Part D — TypeScript (Q27–Q34)
- Part E — Express, profiling, production (Q35–Q40)

---

## Part A — JavaScript Fundamentals

### Q1. What is the event loop in Node.js and how does it actually work?

**Answer:**
The event loop is the runtime mechanism that lets Node.js handle thousands of concurrent operations on a single JavaScript thread. JavaScript itself has no concept of concurrency — it's a single-threaded interpreter (V8). Node wraps it with **libuv**, which provides the event loop plus a thread pool (default 4 threads) for genuinely blocking operations: filesystem, DNS resolution, some crypto, `zlib`. Everything else — network sockets, timers, signals — is non-blocking at the OS level via `epoll` (Linux), `kqueue` (macOS), or IOCP (Windows). libuv just bridges those kernel events back into JS callbacks.

The loop runs in **phases**, each with its own callback queue:

```
   ┌───────────────────────────┐
┌─▶│           timers          │  setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │     pending callbacks     │  deferred TCP errors, etc.
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
│  │       idle, prepare       │  internal libuv bookkeeping
│  └─────────────┬─────────────┘     ┌──────────────────┐
│  ┌─────────────▼─────────────┐     │   incoming I/O:  │
│  │            poll           │◀────│  sockets, files  │
│  └─────────────┬─────────────┘     └──────────────────┘
│  ┌─────────────▼─────────────┐
│  │            check          │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────▼─────────────┐
└──┤      close callbacks      │  socket.on('close'), etc.
   └───────────────────────────┘
```

Between every callback (since Node 11), the runtime drains the **microtask queue**: first all `process.nextTick` callbacks, then all Promise reactions. That's why `Promise.resolve().then(...)` runs before the next `setTimeout(fn, 0)` callback. A long synchronous JS computation blocks the entire loop — the famous "don't block the event loop" rule.

**Code:**
```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');

// Output:
//   sync
//   nextTick
//   promise
//   timeout    (timeout vs immediate order is non-deterministic outside I/O)
//   immediate
```

**Follow-up 1: What's the difference between macrotasks and microtasks?**
Macrotasks are the per-phase queues (timers, I/O, setImmediate, close). Microtasks (`process.nextTick`, Promise callbacks) drain *between* every macrotask — completely emptied each time. So one macrotask can fan out hundreds of microtasks and they all run before the loop moves forward. This is why recursive `process.nextTick` calls can starve the loop: the timer phase never fires.

**Follow-up 2: How would you detect a blocked event loop in production?**
Use `perf_hooks.monitorEventLoopDelay()`. Sample its `.max` and `.percentile(99)` every few seconds and emit as a metric. Healthy Node services keep p99 delay under 10ms. Spikes over 100ms mean someone is doing CPU work on the main thread — a sync `JSON.parse` on a huge payload, a regex with catastrophic backtracking, or a hot loop. `clinic doctor` and `--prof` confirm offline.

---

### Q2. Explain `process.nextTick()` vs `setImmediate()` vs `setTimeout(fn, 0)`.

**Answer:**
All three look like "run this later" but they queue into very different places.

- **`process.nextTick(fn)`** — runs at the end of the *current* operation, before the event loop continues to the next phase. It's not part of the event loop at all; it's a microtask handled by Node itself. Fires before any Promise callbacks too (Node-specific behavior).
- **`Promise.resolve().then(fn)`** — also a microtask, but in the *Promise* microtask queue. Drains right after `process.nextTick` queue.
- **`setImmediate(fn)`** — runs in the **check** phase of the event loop, right after `poll`. Designed for "run on the next iteration, after I/O."
- **`setTimeout(fn, 0)`** — runs in the **timers** phase. Has a minimum effective delay of ~1ms (libuv clamps it).

Ordering between `setTimeout(fn, 0)` and `setImmediate(fn)` is non-deterministic at the top level. But *inside an I/O callback*, `setImmediate` always wins because the loop just exited the poll phase and check runs next.

**Code:**
```js
// Top-level — non-deterministic
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));

// Inside I/O — setImmediate always first
require('fs').readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));   // logged first
});
```

**Follow-up 1: When should I actually use `process.nextTick`?**
Use it to defer something until the *current synchronous chunk* finishes but before any I/O. The classic use is making a constructor that emits an event: if you `this.emit('connect')` inside the constructor, no one has had a chance to attach a listener yet, so you `process.nextTick(() => this.emit('connect'))` to push it past the current frame.

**Follow-up 2: What's the danger of recursive `process.nextTick`?**
Microtasks drain completely before the loop advances. If a `nextTick` callback queues another `nextTick`, the loop is stuck in microtask drain forever — timers don't fire, I/O isn't polled, the server hangs. Same risk with `Promise.then` chains that synchronously schedule more promises.

---

### Q3. How do Promises work under the hood? Explain the three states and chaining.

**Answer:**
A Promise is a wrapper around a future value with three possible states: **pending**, **fulfilled** (resolved with a value), or **rejected** (resolved with a reason). State transitions are one-way: pending → fulfilled or pending → rejected, and once settled, the state cannot change.

When you call `.then(onFulfilled, onRejected)`, you don't run the callbacks immediately even if the promise is already settled — they're scheduled as microtasks. `.then()` returns a *new* promise that resolves with whatever the callback returns. If the callback returns another promise, the outer promise waits for it ("unwrapping"). If the callback throws, the new promise rejects.

This is why `.catch(fn)` is just shorthand for `.then(undefined, fn)` and why errors propagate down a chain: each `.then` returns a new promise, and a rejection skips all the fulfillment handlers until it finds a rejection handler.

**Code:**
```js
Promise.resolve(1)
  .then(x => x + 1)              // 2
  .then(x => Promise.resolve(x * 10))  // unwraps inner promise → 20
  .then(x => { throw new Error('boom'); })
  .then(x => console.log('skipped'))    // skipped
  .catch(e => console.log('caught:', e.message));  // caught: boom
```

**Follow-up 1: What's the difference between returning a value vs returning a promise inside `.then`?**
Returning a plain value wraps it in `Promise.resolve(value)` for the next `.then`. Returning a promise causes the outer chain to *adopt* that promise's state — wait for it to settle, then continue with its resolved value. This is called "assimilation" and is why nested `.then`s flatten into a chain.

**Follow-up 2: Why does an unhandled promise rejection sometimes silently disappear?**
If a promise rejects and no `.catch` (or second arg to `.then`) is ever attached, Node fires the `unhandledRejection` event. In Node 15+, the default behavior changed to terminate the process. Always attach a handler — or use `await` inside `try/catch`, which converts the rejection into a thrown exception you can't ignore.

---

### Q4. How does `async/await` work? What's the relationship to Promises?

**Answer:**
`async/await` is pure syntactic sugar over Promises — no new mechanism, no new runtime feature. An `async` function always returns a Promise. If the function returns a value, the promise resolves with it; if it throws, the promise rejects.

`await expr` does three things:
1. If `expr` is not a Promise, wraps it in `Promise.resolve(expr)`.
2. Pauses the function and yields control back to the event loop.
3. When the promise settles, resumes the function with the resolved value (or throws if rejected).

Under the hood, the JS engine transforms `async` functions into state machines that resume execution via microtasks. This is why `await` introduces a microtask boundary — code right after `await` runs in a future tick, not synchronously.

**Code:**
```js
async function fetchUser(id) {
  try {
    const res = await fetch(`/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (err) {
    // rejection from fetch OR from json() OR our throw — all caught here
    throw new Error(`fetchUser failed: ${err.message}`);
  }
}
```

**Follow-up 1: Why is `return await promise` sometimes considered a smell?**
Inside a non-try/catch context, `return promise` and `return await promise` behave the same to the caller — both produce a promise that resolves to the same value. But `return await` adds an extra microtask hop and historically broke stack traces. *Inside a try/catch, however, you need `return await`* — without it, a rejection escapes the try block because the function returns before the promise settles.

**Follow-up 2: How do you run async operations in parallel with async/await?**
Don't `await` them sequentially. Kick off all promises first, then `await Promise.all([...])`. Example: `const [a, b] = await Promise.all([fetchA(), fetchB()]);`. If you write `const a = await fetchA(); const b = await fetchB();`, B doesn't start until A finishes — serial, not parallel.

---

### Q5. Explain closures with a real-world Node.js example.

**Answer:**
A closure is a function bundled with references to its surrounding lexical scope. When you create a function inside another function, the inner function "remembers" the outer function's variables even after the outer function has returned. Those variables are kept alive on the heap as long as the closure references them.

Closures are everywhere in Node: every callback you pass to `fs.readFile`, `setTimeout`, or an Express route handler captures variables from where it was defined. They're how you build private state in factory functions and how IIFE-based module patterns worked before ES modules.

**Code — a real-world rate limiter built with a closure:**
```js
function createRateLimiter(maxPerSecond) {
  const calls = [];                       // private state, captured by closure

  return function tryAcquire() {
    const now = Date.now();
    while (calls.length && now - calls[0] > 1000) calls.shift();
    if (calls.length >= maxPerSecond) return false;
    calls.push(now);
    return true;
  };
}

const limiter = createRateLimiter(5);
if (limiter()) doWork();
```

`calls` is never exposed to the outside — only the returned function can touch it. This is the closure-as-encapsulation pattern.

**Follow-up 1: How can closures cause memory leaks?**
A closure holds a reference to its entire enclosing scope, even variables it doesn't use. If a long-lived object (an event listener, a cached function) closes over a giant array or DOM-like object, that object can't be garbage-collected. Classic Node leak: registering listeners inside a request handler that close over the `req`/`res` objects — if the listener outlives the request, the request stays in memory.

**Follow-up 2: What's the difference between closures and the `this` keyword?**
Closures capture lexically scoped variables (resolved at definition time). `this` is dynamically bound (resolved at call time) based on how the function is invoked. Arrow functions ignore `this` rebinding and inherit it from the enclosing scope — which is itself a closure-like behavior over `this`. That's why `arr.map(x => this.foo)` works inside a class method but `arr.map(function(x){ return this.foo; })` doesn't.

---

### Q6. Explain `this` binding in JavaScript. What are the four rules?

**Answer:**
`this` is determined by *how* a function is called, not where it's defined. The rules, in priority order:

1. **`new` binding** — `new Foo()` creates a fresh object and binds `this` to it.
2. **Explicit binding** — `.call(ctx, ...)`, `.apply(ctx, [...])`, or `.bind(ctx)` set `this` directly.
3. **Implicit binding** — calling as a method: `obj.foo()` binds `this` to `obj`.
4. **Default binding** — plain function call `foo()` binds `this` to `undefined` in strict mode, or the global object (`global` in Node) otherwise.

**Arrow functions break the rules entirely**: they don't have their own `this`. They inherit it from the enclosing lexical scope, which is what makes them ideal for callbacks inside methods.

**Code:**
```js
const obj = {
  name: 'A',
  regular() { return this.name; },
  arrow: () => this.name,            // `this` is module-level, NOT obj
};
obj.regular();                       // 'A'  (implicit binding)
obj.arrow();                         // undefined
const f = obj.regular;
f();                                 // undefined in strict mode (default binding)
f.call({ name: 'B' });               // 'B'  (explicit binding)
```

**Follow-up 1: Why is `this` `undefined` inside a class method passed as a callback?**
Extracting a method (`const fn = instance.method`) loses the implicit binding. When the callback fires, it's called as a plain function — default binding gives `undefined` (classes are always strict mode). Fix: `fn.bind(instance)` or define the method as an arrow function class field: `method = () => { ... }`.

**Follow-up 2: How does `new` interact with arrow functions?**
You can't `new` an arrow function — they don't have a `[[Construct]]` internal method. Trying throws `TypeError: X is not a constructor`. They also don't have an `arguments` object or a `prototype` property. Use them for callbacks, not for constructable types.

---

### Q7. How does prototypal inheritance work? How does it compare to class inheritance?

**Answer:**
Every JS object has an internal `[[Prototype]]` link (accessible via `Object.getPrototypeOf(obj)` or the legacy `__proto__`). When you access a property and the object doesn't have it, JS walks up the prototype chain until it finds the property or reaches `null`. That's inheritance — lookup by chain, not by copying.

`class` syntax (ES2015) is sugar over the prototype model. `class Foo extends Bar { method() {} }` is conceptually:
- `Foo.prototype = Object.create(Bar.prototype)`
- `Foo.prototype.method = function() {}`
- `new Foo()` returns an object whose `[[Prototype]]` is `Foo.prototype`.

The big difference from classical OOP: prototypes are *live* objects. You can mutate `Array.prototype` and every array on the planet immediately sees the change (which is why you don't do that).

**Code:**
```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}
class Dog extends Animal {
  speak() { return `${this.name} barks`; }
}

const d = new Dog('Rex');
Object.getPrototypeOf(d) === Dog.prototype;            // true
Object.getPrototypeOf(Dog.prototype) === Animal.prototype; // true
d.speak();                                              // 'Rex barks'
```

**Follow-up 1: What's the difference between `__proto__` and `prototype`?**
`prototype` is a property of *constructor functions* — it's the object that becomes the `[[Prototype]]` of instances created via `new`. `__proto__` is the legacy accessor for any object's `[[Prototype]]` link. So `dog.__proto__ === Dog.prototype` is true. Use `Object.getPrototypeOf(dog)` in modern code.

**Follow-up 2: How would you implement `Object.create` in one line?**
`Object.create(proto) === { __proto__: proto }`. Or with a helper:
```js
function create(proto) {
  function F() {}
  F.prototype = proto;
  return new F();
}
```
That's literally how it was polyfilled before ES5.

---

### Q8. Explain hoisting for `var`, `let`, `const`, and function declarations.

**Answer:**
"Hoisting" is the conceptual model that variable and function *declarations* are processed before any code in the scope runs. The actual mechanism: the engine scans the scope, allocates bindings, and only then executes line by line.

- **`var`** — declaration hoisted to the top of the function scope; initialized to `undefined`. Reading it before the `var` line gives `undefined`, not an error.
- **`let` / `const`** — declaration hoisted to the top of the *block* scope, but the binding stays in the **temporal dead zone (TDZ)** until the actual `let`/`const` line executes. Reading it before that throws `ReferenceError`.
- **Function declarations** — fully hoisted: both the binding and the function value. You can call the function above its declaration line.
- **Function expressions** (`const f = function(){}` or `const f = () => {}`) — only the variable is hoisted by `let`/`const` rules (TDZ); the function value isn't assigned until that line runs.

**Code:**
```js
console.log(a);     // undefined
console.log(b);     // ReferenceError: Cannot access 'b' before initialization
var a = 1;
let b = 2;

foo();              // 'works' — function declaration hoisted
function foo() { console.log('works'); }

bar();              // TypeError: bar is not a function
var bar = () => {};
```

**Follow-up 1: Why is the TDZ useful?**
It catches bugs. With `var`, you could use a variable above its declaration and get silent `undefined` behavior. TDZ makes that a loud error, encouraging declare-before-use. It also makes `const` actually mean "never observable in an uninitialized state."

**Follow-up 2: Are `class` declarations hoisted?**
Yes, but with TDZ semantics like `let`. You can't reference a class before its declaration line. Also, classes are always strict mode and never auto-coerce to global.

---

### Q9. `==` vs `===` — what coercion rules cause the biggest gotchas?

**Answer:**
`===` is strict equality: same type, same value. No coercion.
`==` is "loose equality": tries to coerce operands to a comparable type first, following a specific algorithm.

The coercion rules that bite people:
- `null == undefined` → `true` (but `null == 0` is `false`).
- `0 == ''` → `true` (both coerce to `0`).
- `0 == '0'` → `true`. `'' == '0'` → `false` (no coercion needed — different strings).
- `[] == false` → `true`. `[] == 0` → `true`. `[0] == false` → `true`.
- `NaN == NaN` → `false`. Always.

Modern code: use `===` everywhere except one specific pattern — `x == null` to check for *both* `null` and `undefined` in one expression. Even that's becoming uncommon with nullish-coalescing `??`.

**Code:**
```js
0 == false        // true
0 == ''           // true
0 == '0'          // true
false == ''       // true
false == '0'      // true
null == undefined // true
null == 0         // false   <-- inconsistent with the rest
NaN == NaN        // false   <-- use Number.isNaN()
```

**Follow-up 1: How does `Object.is()` differ from `===`?**
`Object.is` is like `===` except `Object.is(NaN, NaN) === true` and `Object.is(+0, -0) === false`. Use it when you need to distinguish `+0` from `-0` (rare — float math) or check for NaN.

**Follow-up 2: Why does `[] == ![]` evaluate to `true`?**
`![]` is `false` (empty array is truthy, so its negation is false). Now compare `[] == false`. Loose equality converts `false` to `0`, then converts `[]` to `''` (array toString) then to `0`. So `0 == 0` is `true`. This is the canonical example of why nobody uses `==`.

---

### Q10. What causes memory leaks in Node.js applications? How would you find one?

**Answer:**
A memory leak is when objects stay reachable from a GC root after they should be discardable. In Node, the common causes are:

1. **Global accumulators** — appending to a module-level array forever (caches, logs).
2. **Unbounded caches** — a `Map` keyed by user ID with no eviction.
3. **Closures retaining large objects** — handlers that close over big payloads.
4. **Event listeners never removed** — `emitter.on('data', fn)` without a corresponding `off`, especially on long-lived emitters like the HTTP server or a Redis client.
5. **Timer leaks** — `setInterval` callbacks that hold references to objects that should be freed.
6. **Promise chains that never resolve** — pending promises hold their resolution callbacks alive.

**Detection workflow:**
1. Reproduce a leak: load-test for 10 minutes, watch `process.memoryUsage().heapUsed` climb without coming back down after load drops.
2. Take three heap snapshots from `--inspect` Chrome DevTools at different times.
3. Diff snapshots, sort by "Retained Size," and look at the constructors that grew.
4. For each top offender, view the retainer chain — that tells you *what reference* is keeping it alive.

**Code — a textbook leak:**
```js
const cache = new Map();   // never evicted

app.get('/data/:id', (req, res) => {
  if (!cache.has(req.params.id)) {
    cache.set(req.params.id, heavyComputation(req.params.id));
  }
  res.json(cache.get(req.params.id));
});
// After 1M unique IDs, cache eats all your RAM. Use lru-cache.
```

**Follow-up 1: What's the difference between `WeakMap` / `WeakSet` and `Map` / `Set` for caching?**
`WeakMap` keys are weakly held — when the key object is garbage-collected elsewhere, the entry vanishes automatically. This makes them perfect for attaching metadata to objects you don't own. But they don't help for typical caches keyed by strings (primitives can't be weak keys).

**Follow-up 2: How do you tell a leak from normal growth?**
Force a GC between samples (`node --expose-gc` and `global.gc()`) so you compare *post-GC retained* memory. If retained memory grows monotonically across forced GCs, it's a leak. If it grows during load and drops after, it's just working-set memory.

---

## Part B — Async, Streams, Concurrency

### Q11. Explain the four types of streams in Node. When do you use each?

**Answer:**
Streams are Node's abstraction for reading/writing data incrementally — without holding the whole payload in memory. There are four types, all built on `EventEmitter`:

- **Readable** — produces data you can consume. Examples: `fs.createReadStream`, `http.IncomingMessage` (the request body), a TCP socket on the read side.
- **Writable** — accepts data you push to it. Examples: `fs.createWriteStream`, `http.ServerResponse` (`res`), a TCP socket on the write side.
- **Duplex** — both readable and writable, independent. A TCP socket is a duplex stream.
- **Transform** — a duplex where the output is a function of the input. `zlib.createGzip()`, encryption streams, line splitters.

Use streams when payloads might exceed memory: log processing, video transcoding, large file uploads, CSV exports, or — as in your IoT pipeline — bridging an MQTT subscriber into S3 multipart upload without buffering the full batch.

**Code — uploading a file to S3 without ever loading it into memory:**
```js
const fs = require('fs');
const { S3Client, Upload } = require('@aws-sdk/lib-storage');
const s3 = new S3Client({});

await new Upload({
  client: s3,
  params: { Bucket: 'logs', Key: 'big.log', Body: fs.createReadStream('big.log') }
}).done();
```

**Follow-up 1: What's the difference between flowing and paused mode?**
A Readable stream starts in *paused* mode — data sits in an internal buffer until you call `.read()`. Attaching a `data` listener (or calling `.resume()`) switches to *flowing* mode where data is pushed at you as it arrives. `pipe()` and `for await (const chunk of stream)` handle the mode transitions for you.

**Follow-up 2: Why do you almost always use `pipeline()` instead of `pipe()`?**
`pipe()` doesn't forward errors. If the source errors mid-stream, the destination won't know — you can get partial writes and zombie streams. `stream.pipeline(src, transform, dest, cb)` (or `pipeline.promises`) wires up error propagation and cleanup correctly.

---

### Q12. What is backpressure in streams and how is it handled?

**Answer:**
Backpressure is the situation where a producer creates data faster than a consumer can handle it. Without flow control, the producer overflows the consumer's buffer — memory grows unbounded, the process eventually crashes.

Node's stream implementation handles this via the `highWaterMark` and the return value of `writable.write(chunk)`:
- `write()` returns `true` if the buffer is below the high-water mark (safe to keep writing).
- It returns `false` once the buffer is full. You should stop writing and wait for the `'drain'` event before resuming.

`pipe()` and `pipeline()` implement this dance automatically: when the destination's `write()` returns `false`, they call `source.pause()`; when `'drain'` fires, they call `source.resume()`. That's the mechanical backpressure loop.

```
producer ──write(chunk)──▶ internal buffer ──drain──▶ consumer
                              │
                       highWaterMark
                       (default 16KB for buffers, 16 objects for object mode)
```

**Code — manual backpressure handling:**
```js
function writeMany(writable, items) {
  return new Promise((resolve, reject) => {
    let i = 0;
    function write() {
      while (i < items.length) {
        const ok = writable.write(items[i++]);
        if (!ok) return writable.once('drain', write);   // wait for drain
      }
      resolve();
    }
    writable.on('error', reject);
    write();
  });
}
```

**Follow-up 1: What happens if I ignore the return value of `write()`?**
The stream still buffers — it doesn't drop data. But the buffer grows beyond `highWaterMark`, defeating its purpose. For small writes you'll likely never notice; for high-throughput data (logs, telemetry — your IoT pipeline), memory will balloon and the process gets OOM-killed.

**Follow-up 2: How does backpressure work over HTTP/TCP?**
Same principle, applied at the kernel socket buffer level. When the receiver isn't ack'ing fast enough, the OS shrinks the TCP window, the sender's `socket.write()` starts returning `false`, and Node propagates backpressure up through the stream chain. This is why streaming responses through `res.write()` is fine for slow clients — TCP backpressure naturally throttles your upstream reader.

---

### Q13. What's a Node `Buffer` and when do I use it instead of a string?

**Answer:**
A `Buffer` is a fixed-size chunk of raw binary memory allocated outside V8's heap. It's Node's pre-ES2017 answer to "how do I work with bytes" — strings in JS are always UTF-16 and assume textual data. Buffers don't assume anything; they're just bytes.

You use Buffers when:
- Reading/writing binary protocols (MQTT frames, Protocol Buffers, image bytes).
- Processing files without assuming encoding.
- Talking to native modules / crypto / compression.
- Streaming: most Node streams emit Buffers unless you set an encoding.

Since Node 4+, `Buffer.from(...)` replaces the deprecated `new Buffer(...)` (which had a footgun: `new Buffer(50)` allocated uninitialized memory containing whatever was previously there).

**Code:**
```js
const buf = Buffer.from('hello', 'utf8');     // <Buffer 68 65 6c 6c 6f>
buf.length;                                    // 5 (bytes, not chars)
buf.toString('hex');                           // '68656c6c6f'
buf.toString('base64');                        // 'aGVsbG8='

// Concatenate streams of chunks
const chunks = [];
req.on('data', c => chunks.push(c));
req.on('end', () => {
  const body = Buffer.concat(chunks).toString('utf8');
});
```

**Follow-up 1: What's the difference between `Buffer` and `Uint8Array`?**
`Buffer` is a subclass of `Uint8Array` since Node 4. Same underlying bytes, but `Buffer` adds Node-specific conveniences: encoding-aware `toString()`, `Buffer.concat()`, base64/hex helpers. In modern code you can pass a Buffer anywhere a Uint8Array is expected (Web Crypto, fetch bodies).

**Follow-up 2: How do `Buffer.allocUnsafe()` and `Buffer.alloc()` differ?**
`alloc(n)` returns zeroed memory — safe but slower. `allocUnsafe(n)` returns whatever happened to be in memory — fast but you must overwrite every byte before reading, or you leak previous request data. Use `allocUnsafe` only when you immediately fill it (e.g., copying bytes into it from a source).

---

### Q14. Worker threads vs cluster vs child_process — when do you use each?

**Answer:**
All three are ways to use multiple cores from Node, but they're fundamentally different.

- **`child_process`** — spawns a separate OS process running any command (not necessarily Node). Communication via stdio pipes or IPC channel if both are Node. Heavyweight; full memory isolation. Use it for shelling out to ffmpeg, ImageMagick, Python ML scripts.
- **`cluster`** — built on `child_process.fork`, but specifically for forking your *own Node app* and sharing a server port via round-robin load balancing in the master. Each worker has its own V8 instance, its own memory. Use it for traditional "1 process per core" web servers — though PM2 / containers usually do this for you now.
- **`worker_threads`** — true threads within the same Node process, each with their own V8 isolate but the ability to share memory via `SharedArrayBuffer` and transfer ownership of buffers cheaply. Lighter than processes. Use them for **CPU-bound work** that's part of a request lifecycle: image resizing, JSON parsing of huge payloads, ML inference.

**Code — offloading CPU work to a worker thread:**
```js
// main.js
const { Worker } = require('worker_threads');
const w = new Worker('./resize.js', { workerData: { path: 'big.jpg' } });
w.on('message', result => console.log('done', result));

// resize.js
const { parentPort, workerData } = require('worker_threads');
const result = doExpensiveResize(workerData.path);
parentPort.postMessage(result);
```

**Follow-up 1: Why don't worker threads share normal JavaScript objects?**
Each worker is a separate V8 isolate — they don't share a heap. So you can't pass arbitrary objects; you pass copies via `postMessage` (structured clone) or transfer ownership of `ArrayBuffer` / `MessagePort`. This is by design — no shared mutable state means no race conditions on JS objects.

**Follow-up 2: Should I use cluster or a process manager like PM2?**
For new code, just run multiple Node processes via PM2, Docker, or Kubernetes — they handle restarts, log aggregation, and zero-downtime reloads. `cluster` is fine if you want self-contained scaling without external tools, but most production setups have moved on.

---

### Q15. How does `EventEmitter` work? What is the "max listeners" warning?

**Answer:**
`EventEmitter` is the base class for almost every async producer in Node — streams, HTTP servers, sockets, `process` itself. It maintains an internal map of `event name → array of listener functions`. When `emit('name', ...args)` fires, all listeners for that name are called synchronously in registration order.

Listeners are synchronous by default. If a listener throws and there's no `error` listener, the process crashes. The `'error'` event is special — if emitted with no listener, Node throws.

`EventEmitter` warns when more than 10 listeners are attached to the same event on the same emitter (default `defaultMaxListeners = 10`). This isn't an error, it's a leak hint: "you probably forgot to remove an old listener." Bump it with `emitter.setMaxListeners(n)` if you really need more, or fix the leak.

**Code:**
```js
const { EventEmitter } = require('events');
const bus = new EventEmitter();

bus.on('order', order => console.log('handler 1', order.id));
bus.once('order', order => console.log('handler 2 — fires once', order.id));

bus.emit('order', { id: 42 });   // both fire
bus.emit('order', { id: 43 });   // only handler 1 fires
```

**Follow-up 1: How do you wait for an event using async/await?**
Use the `events.once()` helper:
```js
const { once } = require('events');
const [data] = await once(socket, 'data');
```
It returns a Promise that resolves with the event args. Internally it just attaches a one-shot listener. Combine with `Promise.race` for timeouts.

**Follow-up 2: What's the difference between `on` and `prependListener`?**
`on` appends to the listener array; `prependListener` puts the new listener at the front. Useful when you have an existing listener you can't modify but need to inspect the event first (e.g., logging middleware around a third-party handler).

---

### Q16. How do you handle `uncaughtException` and `unhandledRejection`?

**Answer:**
These are last-resort safety nets, not error-handling tools. Both indicate a bug — a thrown synchronous error that wasn't caught, or a rejected promise with no `.catch`. Production apps should catch errors *where they happen*; these handlers exist to log diagnostics and shut down cleanly.

The accepted pattern: log the error, then crash. **Do not** try to keep running. The process is in an unknown state — memory might be corrupted, half-finished transactions might be in flight, file handles might be leaked. Let it die and let your process supervisor (PM2, systemd, Kubernetes) restart it.

```js
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'uncaught exception');
  // give logger 1s to flush, then die
  setTimeout(() => process.exit(1), 1000).unref();
});

process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'unhandled rejection');
  setTimeout(() => process.exit(1), 1000).unref();
});
```

Since Node 15, unhandled rejections crash the process by default — which is the right behavior.

**Follow-up 1: Why is "log and continue" bad?**
Because the error indicates a bug in your code's invariants. If you swallow it and keep serving requests, future requests might write inconsistent data, return wrong results, or leak resources. Crashing forces you to fix the root cause and restores a known-good state via restart.

**Follow-up 2: How is `domain` different and why was it deprecated?**
`domain` was an early Node API for grouping async operations and catching their errors centrally. It was deprecated because it leaked across async boundaries unpredictably and gave a false sense of safety — the underlying state corruption issue remained. Modern replacement: `AsyncLocalStorage` for *context propagation* (request IDs), and `try/catch` around `await` for error handling.

---

### Q17. `Promise.all` vs `Promise.allSettled` vs `Promise.race` vs `Promise.any` — when to use each?

**Answer:**
All four take an iterable of promises and return one combined promise. They differ in semantics:

- **`Promise.all([...])`** — resolves with an array of results when *all* resolve. Rejects immediately if any one rejects (other promises still run but their results are ignored). Use when you need every result and any failure means abort.
- **`Promise.allSettled([...])`** — always resolves with an array of `{status, value | reason}` objects. Use when you want every result, success or failure (e.g., "send notifications to 100 users and report which failed").
- **`Promise.race([...])`** — resolves or rejects with the first one to settle. Use for timeouts: `Promise.race([fetchData(), sleep(5000).then(() => Promise.reject(new Error('timeout')))])`.
- **`Promise.any([...])`** — resolves with the first one to *fulfill*. Only rejects if *all* reject (with an `AggregateError`). Use for redundant requests: query 3 mirrors, take the first success.

**Code:**
```js
// Send notifications to many users, don't fail the whole batch on one bad address
const results = await Promise.allSettled(users.map(u => sendEmail(u)));
const failures = results.filter(r => r.status === 'rejected');
log.warn({ count: failures.length }, 'send failures');

// Timeout pattern
const withTimeout = (p, ms) => Promise.race([
  p,
  new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms)),
]);
```

**Follow-up 1: With `Promise.all`, do the other promises stop when one rejects?**
No. JavaScript has no cancellation. The other promises continue running to completion; their results are discarded. If those promises consume resources (DB connections, HTTP requests), you've wasted them. Use `AbortController` to propagate cancellation when supported (modern `fetch` accepts a signal).

**Follow-up 2: How do you limit concurrency? `Promise.all` with 1000 items hammers your downstream.**
Use a concurrency limiter like `p-limit` or `p-map`:
```js
const pLimit = require('p-limit');
const limit = pLimit(10);              // max 10 in flight
await Promise.all(items.map(i => limit(() => fetchItem(i))));
```
Without a limiter, `Promise.all` fires off all promises immediately — fine for 10, catastrophic for 10,000.

---

### Q18. What are generators and async iterators? When are they useful?

**Answer:**
A **generator** is a function declared with `function*` that can pause execution at `yield` and resume on `.next()`. Each call to `next()` runs until the next `yield` and returns `{ value, done }`. Generators implement the iterator protocol, so they work with `for...of`.

An **async iterator** is the async version: methods return Promises of `{ value, done }`. Created with `async function*` and consumed with `for await...of`. This is what lets you write streaming code that *looks* synchronous.

Use cases:
- Lazy sequences (pagination cursors, infinite streams).
- Reading large files line-by-line without loading them.
- Consuming Node streams ergonomically.
- Implementing custom iterables for domain objects.

**Code — paginated API consumption with an async generator:**
```js
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next).then(r => r.json());
    for (const item of res.items) yield item;
    next = res.next;
  }
}

for await (const item of paginate('/api/users?cursor=0')) {
  await processItem(item);
}
```

This consumes one page at a time, never holds more than one page in memory, and reads as if it were a regular `for` loop.

**Follow-up 1: How do async iterators relate to Node streams?**
Readable streams implement the async iterator protocol since Node 10. You can do `for await (const chunk of fs.createReadStream(path)) { ... }` instead of attaching `data` and `end` listeners. This makes stream consumption much easier and integrates with `try/catch`.

**Follow-up 2: When would you *not* use a generator?**
When you need parallelism. Generators yield one value at a time — by design serial. If you want N items in flight at once, use `Promise.all` or a worker pool, not a generator.

---

## Part C — Node.js Internals & Ecosystem

### Q19. CommonJS vs ES Modules — what's the difference and why does it matter?

**Answer:**
**CommonJS (CJS)** — Node's original module system. `require()` is a synchronous function that resolves a path, loads the file, executes it, and returns its `module.exports`. Variables `module`, `exports`, `require`, `__dirname`, `__filename` are injected into every file. Resolution is dynamic — you can `require(varName)` at runtime.

**ES Modules (ESM)** — the JS standard since ES2015, supported natively in Node since v12+. `import` is *static* — declarations must be at the top, with literal paths. Loading is asynchronous and supports top-level `await`. Bindings are *live* — importing a value gets a reference, not a copy.

Practical differences that cause pain:
- ESM has no `__dirname` — use `import.meta.url` and `url.fileURLToPath`.
- ESM imports must include the file extension (`./foo.js`, not `./foo`).
- You can `import` a CJS module (its `module.exports` shows up as the default export), but you generally can't `require` an ESM module from CJS — you need dynamic `import()`.
- Package's `"type": "module"` in `package.json` decides the default; `.mjs` / `.cjs` extensions override.

**Code:**
```js
// CJS
const fs = require('fs');
module.exports = { foo };

// ESM
import fs from 'fs';
export const foo = () => {};
import { readFile } from 'node:fs/promises';
```

**Follow-up 1: Why are ESM imports "live bindings"?**
Because the spec says so — and it matters for circular imports. In CJS, a circular `require` returns whatever was on `module.exports` at the moment of the cycle (often incomplete). In ESM, both files share live references that fill in as initialization completes. ESM still has gotchas with circular imports but they're more predictable.

**Follow-up 2: How does Node decide whether a file is CJS or ESM?**
By extension first: `.cjs` is always CJS, `.mjs` is always ESM. For `.js`, Node looks at the nearest `package.json`'s `"type"` field — `"module"` means ESM, `"commonjs"` (or absent) means CJS.

---

### Q20. How does `require()` resolution work in Node?

**Answer:**
When you call `require('foo')`, Node walks a specific algorithm:

1. **Core module check** — if `foo` is a built-in (`fs`, `http`, `node:fs`), return it.
2. **Path-like?** — if the argument starts with `./`, `../`, or `/`, resolve relative to the current file. Try the literal path, then `.js`, `.json`, `.node`. If it's a directory, look for `index.js` or its `package.json`'s `main` field.
3. **`node_modules` walk** — otherwise, look in `./node_modules/foo`, then `../node_modules/foo`, then `../../node_modules/foo`, walking up to filesystem root.
4. **Caching** — once loaded, the result is cached in `require.cache` keyed by resolved absolute path. Subsequent `require`s of the same path return the cached `module.exports`.

The cache is why singletons in Node "just work" — every file that requires the same path gets the same object. It's also why you should be careful with mutable module-level state.

**Code:**
```js
// myService.js
let counter = 0;
module.exports = { increment: () => ++counter };

// a.js, b.js  — both files share the same counter
const svc = require('./myService');
```

**Follow-up 1: How do you bust the require cache for hot-reloading?**
`delete require.cache[require.resolve('./module')]` and require it again. Used by tools like `nodemon` (which actually uses process restart, but the technique applies for in-process reload). Footgun: existing references to the old export object don't get updated.

**Follow-up 2: How does this compare to ESM resolution?**
ESM uses a similar `node_modules` walk but follows the package's `"exports"` field strictly — if a package defines `"exports"`, you can only import the paths it explicitly exposes. No deep-importing internal files. This is called "package encapsulation" and is one of the biggest behavioral changes from CJS.

---

### Q21. npm vs yarn vs pnpm — what's actually different?

**Answer:**
All three install packages from npm registry and manage `package.json` / lockfiles, but they handle the `node_modules` tree differently.

- **npm** — flat-by-default since v3 (hoists dependencies to the top of `node_modules` to deduplicate). Lockfile: `package-lock.json`. Sometimes confusingly nondeterministic across versions.
- **yarn (classic, v1)** — same flat hoisting model. Lockfile: `yarn.lock`, generally simpler than `package-lock.json`. Yarn 2+ ("Berry") introduced Plug'n'Play, a radical change that skips `node_modules` entirely — most teams stuck with Yarn 1 or moved to pnpm.
- **pnpm** — uses a content-addressable global store (`~/.pnpm-store`). Each project's `node_modules` is a tree of *symlinks* into that store. Saves disk space dramatically and enforces strict dependency boundaries: a package can only import what's explicitly in its `package.json` (no accidentally importing a transitive dep that happened to be hoisted).

For monorepos: pnpm and yarn both handle workspaces well; npm workspaces work but feel less polished.

**Follow-up 1: Why does pnpm prevent "phantom dependencies"?**
With npm/yarn flat hoisting, a transitive dep `lodash` might appear at the top of your `node_modules`, so your code can `require('lodash')` even if it's not in your `package.json`. The day that transitive removes lodash, your code breaks. pnpm only symlinks dependencies you declared — phantom imports throw immediately.

**Follow-up 2: When should I commit the lockfile?**
Always. For both apps *and* libraries. Even though libraries don't ship their lockfile to consumers (npm ignores it for installed packages), keeping it in source control gives you reproducible CI builds and bisectable history of dependency changes.

---

### Q22. Explain semver and the important fields in `package.json`.

**Answer:**
**Semver** (semantic versioning): `MAJOR.MINOR.PATCH`.
- MAJOR — breaking changes.
- MINOR — backward-compatible features.
- PATCH — backward-compatible bug fixes.

Version ranges in `package.json`:
- `"^1.2.3"` — compatible with 1.x.x (allows minor + patch, but not major). Most common default.
- `"~1.2.3"` — allows patch updates only (1.2.x).
- `"1.2.3"` — exact pin.
- `"*"` or `""` — anything (don't use).

**Key `package.json` fields:**
- `"name"`, `"version"` — identity.
- `"main"` — CJS entry point.
- `"module"` — ESM entry (used by bundlers).
- `"exports"` — modern replacement: explicit subpath map, blocks deep imports.
- `"type": "module"` — default `.js` files to ESM.
- `"scripts"` — `npm run <name>` shortcuts.
- `"dependencies"` — runtime deps (shipped to consumers).
- `"devDependencies"` — only needed at build/test time.
- `"peerDependencies"` — "the host app must provide this" (React plugins, ESLint plugins).
- `"engines"` — Node version requirement; npm warns or errors on mismatch.

**Follow-up 1: What's the difference between `dependencies` and `peerDependencies`?**
`dependencies` get installed into your `node_modules`. `peerDependencies` declare "the consuming app must already have this" — useful for plugins that must share a singleton instance with the host (React, Webpack loaders). Otherwise you risk two copies of React and breaking hooks.

**Follow-up 2: What does `npm ci` do that `npm install` doesn't?**
`npm ci` (clean install) requires a lockfile, fails if `package.json` and lockfile disagree, deletes `node_modules` first, and never writes to the lockfile. It's deterministic and meant for CI. `npm install` may *update* the lockfile to satisfy ranges, which is great locally but unsafe in CI.

---

### Q23. How does Node's `node_modules` resolution algorithm actually work?

**Answer:**
For non-relative imports (`require('foo')`), Node walks upward through `node_modules` directories:

```
current file: /home/me/app/src/api/users.js

look for:
  /home/me/app/src/api/node_modules/foo
  /home/me/app/src/node_modules/foo
  /home/me/app/node_modules/foo
  /home/me/node_modules/foo
  /home/node_modules/foo
  /node_modules/foo
```

The first hit wins. Then within the found directory:
1. If there's a `package.json` with `"exports"`, use it (strict mode — only listed paths importable).
2. Else if `"main"` is set, load that.
3. Else load `index.js` / `index.json` / `index.node`.

Submodule imports like `require('foo/lib/util')` follow the same walk to find `foo`, then resolve `/lib/util` relative to that package.

**Follow-up 1: What's "hoisting" and when does it bite you?**
npm/yarn flatten the tree: instead of `node_modules/a/node_modules/b/node_modules/c`, they hoist `b` and `c` to the top level. This deduplicates but means siblings can see each other's transitive deps (phantom dependencies). It also means version conflicts can force nested copies — if `a` needs `b@1` but `c` needs `b@2`, you get `node_modules/b@1` at the top and `node_modules/c/node_modules/b@2` nested.

**Follow-up 2: Why might the same `import 'lodash'` resolve to different files in different parts of a monorepo?**
Because of hoisting and version conflicts. Package A in the monorepo might pull lodash@4, package B might pin lodash@3, so each gets its own copy at different paths. Worse: if you accidentally have two copies of a stateful library (like a database client), each consumer gets its own pool. This is why `peerDependencies` and tools like pnpm's strict mode exist.

---

### Q24. `global` vs `window` vs `globalThis` — what's the right way to reference the global object?

**Answer:**
Different runtimes named the global object differently. Browsers have `window` (and `self` in workers). Node has `global`. Older non-strict code could write top-level `var` declarations to become global properties.

`globalThis` (ES2020) is the standard cross-environment reference. It points to whatever the platform's global object is. Use it in code meant to run in multiple environments (libraries) or when polyfilling.

In Node specifically, you almost never *should* attach things to `global`. Module scope is the right place. The few legitimate uses: polyfilling (`globalThis.fetch = ...` for older Node), or test-helper monkeypatching with cleanup.

**Code:**
```js
// Universal
if (!globalThis.fetch) {
  globalThis.fetch = require('node-fetch');
}

// Anti-pattern — global mutable state
global.dbClient = new Client();    // singletons should be modules, not globals
```

**Follow-up 1: Why is attaching to `global` a smell in Node?**
It hides dependencies. A file that imports the `db` module declares its dependency in code. A file that reads `global.db` could be called from anywhere — tests, refactors, parallel workers — and silently break. It also creates leaks: `global` is never GC'd.

**Follow-up 2: How would you safely polyfill `fetch` only if missing?**
```js
if (typeof globalThis.fetch !== 'function') {
  const undici = await import('undici');
  globalThis.fetch = undici.fetch;
}
```
Or simpler: in Node 18+, `fetch` is built in. Check `process.version` and skip the polyfill on newer Node.

---

### Q25. How does V8 garbage collection work? What's the young/old generation split?

**Answer:**
V8 uses a **generational garbage collector** based on the empirical observation that most objects die young. The heap is split into:

- **Young generation (new space)** — small (~1–8 MB by default). New allocations go here. Collected frequently with **Scavenge**, a copying collector: live objects are copied from the "from-space" to the "to-space," dead objects are abandoned. Fast — proportional to *live* data, not total data. Survivors of two scavenges get promoted.
- **Old generation (old space)** — large (default ~1.5 GB on 64-bit). Collected less often with **Mark-Sweep** and **Mark-Compact** algorithms. These are slower but designed to be incremental (interleaved with JS execution) and concurrent (running on a background thread) so GC pauses stay under ~10ms in well-behaved apps.

```
   ┌──────── new space ────────┐    ┌──────── old space ────────┐
   │ from │    to    │         │    │         old objects        │
   └──────┴──────────┴─────────┘    └────────────────────────────┘
        scavenge (copying)             mark-sweep / mark-compact
```

The implication: a workload that allocates short-lived objects (most request handlers) hits the fast new-space collector and barely notices GC. A workload that retains many objects long-term puts pressure on old space and risks longer pauses.

**Follow-up 1: What causes GC pauses to spike?**
Promoting a large number of objects to old space at once (e.g., loading a 200MB JSON), or filling old space such that a major GC is triggered. You can observe this with `--trace-gc`. Mitigation: avoid creating huge transient objects, stream where possible, increase old-space size with `--max-old-space-size`.

**Follow-up 2: How does `--max-old-space-size` interact with container memory limits?**
Node's default old-space limit is ~1.5 GB regardless of container size. If your container has 4 GB but Node doesn't know, it'll OOM-kill at 1.5 GB while the container has free memory. Always set `--max-old-space-size=$((CONTAINER_MEM_MB * 80 / 100))` (leave headroom for native, stack, code). Modern Node (16+) tries to auto-detect cgroup limits but it's not foolproof.

---

### Q26. How do you debug a Node.js OOM (out-of-memory) crash?

**Answer:**
Symptoms: process gets killed, sometimes with `JavaScript heap out of memory`, sometimes silently if the OS OOM-killer fires first.

Workflow:
1. **Confirm it's the JS heap, not native or container.** Set `--max-old-space-size` explicitly so Node throws the JS error rather than getting SIGKILL'd. The error message tells you it was V8's heap, not RSS overrun from native modules.
2. **Capture a heap snapshot on OOM.** Run with `--heapsnapshot-near-heap-limit=3` — Node writes up to 3 snapshots as it approaches the limit. Open in Chrome DevTools → Memory tab.
3. **Find the leak.** In DevTools, switch to "Comparison" view between two snapshots. Sort by "Delta" — what grew? Click into the top constructor (often `Object`, `Array`, `Buffer`) and inspect retainers — the chain of references keeping the object alive.
4. **For production**, add periodic heap sampling: `clinic heapprofile` or `--inspect` attached via SSH tunnel.

**Code — preemptive monitoring:**
```js
const v8 = require('v8');
setInterval(() => {
  const s = v8.getHeapStatistics();
  metrics.gauge('node.heap.used', s.used_heap_size);
  metrics.gauge('node.heap.total', s.total_heap_size);
  metrics.gauge('node.heap.limit', s.heap_size_limit);
}, 10_000);
```

**Follow-up 1: My RSS keeps growing but heap stays flat — what's leaking?**
Native memory: Buffers (older versions allocated outside heap), native module memory (sharp, libxml, node-canvas), C++ addons, or the libuv thread pool. Use `process.memoryUsage()` to see `external` and `arrayBuffers` separately from heap. Tools: `heaptrack`, `valgrind` (Linux), or auditing native dep changes.

**Follow-up 2: Why might the heap snapshot be misleading?**
Because the snapshot captures *only the JS heap*. Off-heap memory (Buffer pools, external resources, fs handles) doesn't show up. Also, taking a snapshot triggers a full GC first — so anything genuinely freeable disappears. If your problem only manifests under load, you may need to snapshot mid-load.

---

## Part D — TypeScript

### Q27. Why use TypeScript over plain JavaScript? What does it actually buy you?

**Answer:**
TypeScript adds a **static type system** on top of JavaScript. At compile time, it checks that function arguments match parameters, that you're not calling methods on `undefined`, that objects have the properties you think they do. At runtime, TypeScript is gone — the emitted JS is what runs.

Concrete benefits at scale:
- **Refactor confidence.** Rename a property, the compiler tells you every line that broke. Without types, this is grep-and-pray.
- **IDE intelligence.** Autocomplete, jump-to-definition, inline parameter hints all become reliable because the IDE has type info.
- **Documentation that can't go stale.** A function's signature documents its contract; if the contract changes, types update.
- **Bug class elimination.** Roughly 15% of bugs in large JS codebases are type errors (Microsoft's internal study). TS catches those at compile time.

Costs: a build step, learning curve, and occasional type-system pain (complex generics, third-party libs with bad types). For your NestJS work, TS is essentially required — NestJS leans on decorators and metadata reflection that needs type info.

**Follow-up 1: Doesn't `strict: true` slow me down?**
Initially yes, long-term no. `strict` enables `strictNullChecks`, `noImplicitAny`, and friends. It forces you to think about "could this be `null`?" and "what's the type of this parameter?" — which is exactly the thinking that prevents bugs. Disable individual flags only when you have a concrete reason.

**Follow-up 2: How is TypeScript's type system different from Java's or C#'s?**
Three big differences: (1) it's **structural** — two types are compatible if they have the same shape, not if they declare a common ancestor; (2) it's **erased** — types vanish at runtime, no reflection or runtime checks; (3) it has **flow-sensitive narrowing** — the type of a variable changes inside an `if` block based on conditions you wrote. These make TS extremely expressive but also mean you can't `instanceof` an interface or check a generic's type at runtime.

---

### Q28. `interface` vs `type` — when do you use each?

**Answer:**
Both define named types. Major differences:

- **`interface`** can be **declaration-merged** — declaring the same interface twice adds the fields. Useful for extending third-party types (e.g., adding properties to Express's `Request`).
- **`interface`** can only describe object shapes (and function/class shapes). **`type`** can alias anything: unions, intersections, primitives, tuples, mapped types, conditional types.
- **`interface`** uses `extends` for inheritance; **`type`** uses `&` (intersection).
- Error messages tend to be slightly cleaner with `interface` for nominal-ish design.

Practical rule: use `interface` for object shapes that might be extended (public API, model definitions). Use `type` for unions, intersections, tuples, and anything algebraic.

**Code:**
```ts
// interface — extensible, object shape
interface User { id: string; name: string }
interface User { email: string }            // merged: User has id, name, email

// type — algebraic, unions
type Status = 'pending' | 'active' | 'banned';
type WithId<T> = T & { id: string };

// extending Express Request
declare global {
  namespace Express {
    interface Request { user?: { id: string; role: string } }
  }
}
```

**Follow-up 1: Can interfaces represent unions?**
No. `interface` is for object-like shapes only. For unions, you must use `type`. This is the main reason teams default to `type` everywhere — one consistent tool.

**Follow-up 2: When does declaration merging cause problems?**
When you don't expect it. Two different files declaring `interface User` accidentally combine into one type. Errors can show up far from the source. With `type`, a second declaration is a hard error — louder and easier to debug.

---

### Q29. Explain generics. Give a real-world backend example.

**Answer:**
Generics let you parameterize types — write a function or class once, use it with many types while preserving type information. Without generics, you'd either lose type info (returning `any`) or duplicate the code per type.

The classic example is a typed Repository pattern: one base class that knows how to persist *some* entity, parameterized by the entity type.

```ts
abstract class Repository<T extends { id: string }> {
  constructor(private db: Knex, private table: string) {}

  async findById(id: string): Promise<T | null> {
    const row = await this.db(this.table).where({ id }).first();
    return (row ?? null) as T | null;
  }

  async create(data: Omit<T, 'id'>): Promise<T> {
    const [row] = await this.db(this.table).insert(data).returning('*');
    return row as T;
  }
}

interface User { id: string; email: string; name: string }
class UserRepository extends Repository<User> {
  constructor(db: Knex) { super(db, 'users'); }
  findByEmail(email: string): Promise<User | null> {
    return this.db('users').where({ email }).first();
  }
}
```

`Omit<T, 'id'>` is a generic itself — it produces a type with all of `T`'s fields except `id`. The Repository works for any entity that has an `id: string` field.

**Follow-up 1: What does `<T extends X>` mean?**
It's a generic constraint — T can be any type, but it must be assignable to X. Inside the function/class, T has at least X's properties. Useful for "this works on any object with an id field": `<T extends { id: string }>`.

**Follow-up 2: When should I use a generic vs `unknown` vs an overload?**
Generic when the caller's type should flow through to the return type. `unknown` when you genuinely don't know the type and want callers to narrow it. Overloads when the return type depends on the *value* of an argument (e.g., a function that returns `string` when given `'json'` and `Buffer` when given `'binary'`).

---

### Q30. Explain key utility types: `Partial`, `Pick`, `Omit`, `Record`, `ReturnType`.

**Answer:**
Utility types are built-in generic helpers for transforming existing types.

- **`Partial<T>`** — makes all properties of T optional. Common in update DTOs: `update(id, patch: Partial<User>)`.
- **`Required<T>`** — opposite; all properties required.
- **`Pick<T, K>`** — produces a type with only the properties named in K (a union of keys). E.g., `Pick<User, 'id' | 'email'>` → `{ id; email }`.
- **`Omit<T, K>`** — opposite of Pick; everything *except* K. E.g., DTO without password: `Omit<User, 'passwordHash'>`.
- **`Record<K, V>`** — an object type with keys K and values V. `Record<string, number>` is "a map from string to number." `Record<'admin' | 'user', Permission[]>` is "an object with exactly these two keys."
- **`ReturnType<F>`** — the return type of function F. Useful for typing wrappers: `type UserDTO = ReturnType<typeof userService.find>`.
- **`Awaited<P>`** — unwraps Promise. `Awaited<Promise<User>>` → `User`.

**Code — typing API request/response with utility types:**
```ts
interface User {
  id: string;
  email: string;
  passwordHash: string;
  createdAt: Date;
}

type CreateUserDto = Omit<User, 'id' | 'createdAt'>;        // input
type PublicUser    = Omit<User, 'passwordHash'>;             // output
type UpdateUserDto = Partial<Pick<User, 'email'>>;           // patch

function createUser(input: CreateUserDto): PublicUser { /* ... */ }
```

**Follow-up 1: How would I write `Partial` myself?**
```ts
type MyPartial<T> = { [K in keyof T]?: T[K] };
```
A **mapped type** that iterates over T's keys and makes each property optional with `?`.

**Follow-up 2: What's the difference between `Record<string, T>` and `{ [k: string]: T }`?**
Identical at runtime; mostly stylistic. `Record` is slightly more readable and signals intent ("this is a map"). `{ [k: string]: T }` is a literal index signature — sometimes needed if you want to combine it with specific properties.

---

### Q31. What are decorators? Why does NestJS use them so heavily?

**Answer:**
Decorators are functions that attach metadata or modify the behavior of a class, method, property, or parameter declaration. Syntax: `@decoratorName` placed above the target. In TypeScript, decorators are a Stage-3 proposal; you enable them with `"experimentalDecorators": true` and (for metadata) `"emitDecoratorMetadata": true`.

NestJS is built around decorators because they let the framework do **dependency injection** and **route registration** purely from your class declarations:

```ts
@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get(':id')
  @UseGuards(AuthGuard)
  findOne(@Param('id') id: string) {
    return this.users.findOne(id);
  }
}
```

Under the hood, `@Controller('users')` is a function that stamps metadata onto `UsersController` (the base path). `@Get(':id')` does the same for the method (HTTP verb and sub-path). At bootstrap, Nest reflects this metadata, wires up Express routes, and resolves `UsersService` via the DI container.

`@emitDecoratorMetadata` is what makes the constructor's parameter types available at runtime — without it, Nest couldn't see "this constructor wants a UsersService." With it, TypeScript emits a `design:paramtypes` metadata entry the framework can read.

**Follow-up 1: What's the difference between class, method, property, and parameter decorators?**
Each receives different arguments. Class: `(target)`. Method: `(target, propertyKey, descriptor)`. Property: `(target, propertyKey)`. Parameter: `(target, propertyKey, parameterIndex)`. The descriptor argument on method decorators lets you wrap or replace the method.

**Follow-up 2: What is `reflect-metadata` and why does NestJS need it?**
`reflect-metadata` is a polyfill for the Reflect Metadata API — a key/value store of metadata associated with classes and their members. When TS emits `design:paramtypes`, it stores it via `Reflect.defineMetadata`. NestJS reads it back with `Reflect.getMetadata` to learn what types a constructor wants. Without `reflect-metadata` imported once at startup, those reads return `undefined` and DI breaks.

---

### Q32. What is type narrowing? Explain discriminated unions.

**Answer:**
**Type narrowing** is the compiler's ability to infer a more specific type within a code block based on checks you write. After `if (typeof x === 'string')`, x's type is `string` for the rest of the block.

Narrowing operators:
- `typeof x === '...'` for primitives.
- `instanceof Class` for class instances.
- `x in obj` for property presence.
- Equality (`===`) for literal types.
- User-defined **type guards**: `function isUser(x: unknown): x is User { ... }`.

A **discriminated (tagged) union** is a union of object types that share a common property whose literal value distinguishes them. The compiler can narrow by checking that property.

**Code:**
```ts
type Event =
  | { type: 'click'; x: number; y: number }
  | { type: 'submit'; values: Record<string, string> }
  | { type: 'error'; message: string };

function handle(e: Event) {
  switch (e.type) {
    case 'click':  return `clicked at ${e.x},${e.y}`;     // e is the click variant
    case 'submit': return `submitted ${Object.keys(e.values).length} fields`;
    case 'error':  return `oops: ${e.message}`;
    default:       const _exhaustive: never = e;          // compile error if a case is missing
                   return _exhaustive;
  }
}
```

The `never` assignment is the **exhaustiveness check**: if you add a new variant to `Event` and forget a case, the assignment fails because the remaining type isn't `never`. This is one of the most powerful TS patterns.

**Follow-up 1: When would I use a type guard function instead of `typeof`?**
When `typeof` can't distinguish what you need. To tell apart two object shapes, you write `function isAdmin(u: User): u is Admin { return u.role === 'admin'; }`. After `if (isAdmin(u))`, `u` is narrowed to `Admin`.

**Follow-up 2: How do you narrow with `unknown`?**
`unknown` is the type-safe version of `any` — you can't do anything with it until you narrow. Use `typeof`, `instanceof`, or a guard. This is the recommended pattern for parsing external input (`JSON.parse` returns `unknown` in strict configs): narrow first, then use.

---

### Q33. `unknown` vs `any` vs `never` — what's the difference?

**Answer:**
Three different concepts:

- **`any`** — opt out of the type system. You can do anything with an `any` value and pass it anywhere. Defeats the purpose of TypeScript; treat as a code smell.
- **`unknown`** — "I don't know what this is yet." Like `any`, can hold any value, but you **can't use it** until you narrow. Forces a type check before access. Use for parsed JSON, dynamic config, external input.
- **`never`** — the type of values that can never exist. Used as: the return type of functions that always throw or loop forever; the inferred type of an unreachable branch after exhaustive narrowing; the empty union.

```ts
function fail(msg: string): never { throw new Error(msg); }
function loop(): never { while (true) {} }

const x: unknown = JSON.parse(s);
// x.foo;                  // error — must narrow first
if (typeof x === 'object' && x !== null && 'foo' in x) {
  // x is narrowed here
}
```

**Follow-up 1: Why is `any` dangerous?**
Because it silently spreads. If you have `function f(x: any): SomeType`, the return is typed but the body's operations on `x` aren't checked, so a bug in `f` can produce a wrong `SomeType` that everything downstream trusts. `unknown` avoids this — you can't use `x` without checking.

**Follow-up 2: Where do I see `never` in real code?**
The exhaustiveness check pattern (Q32) is the big one. Also: throw helpers (`function assertNever(x: never): never { throw ... }`), and as the inferred return type of `process.exit()` so the compiler knows code after it is unreachable.

---

### Q34. What are the essential `tsconfig.json` options you should know?

**Answer:**
Minimum set for a serious Node project:

```json
{
  "compilerOptions": {
    "target": "ES2022",                  // Node 18+ supports this fully
    "module": "NodeNext",                // ESM with Node-style resolution
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,                      // turns on the full strict family
    "esModuleInterop": true,             // allows `import x from 'cjs-module'`
    "skipLibCheck": true,                // skip type-check of node_modules (huge speedup)
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,           // import './data.json'
    "experimentalDecorators": true,      // for NestJS
    "emitDecoratorMetadata": true,
    "sourceMap": true                    // debugger / stack traces
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

What `"strict": true` actually enables:
- `noImplicitAny` — parameters/variables can't be implicit any.
- `strictNullChecks` — `null` and `undefined` must be explicit in types.
- `strictFunctionTypes` — function param types checked contravariantly (sound).
- `strictBindCallApply`, `strictPropertyInitialization`, `alwaysStrict`, `noImplicitThis`, `useUnknownInCatchVariables`.

**Follow-up 1: What's the difference between `target` and `module`?**
`target` is the *language version* of the output JS (which syntax features get downleveled). `module` is the *module system* (`require` vs `import`). They're independent — you can target ES2022 syntax but emit CommonJS modules.

**Follow-up 2: Why is `skipLibCheck: true` recommended?**
Without it, the compiler type-checks every `.d.ts` in `node_modules`. With hundreds of dependencies, this can add 30+ seconds to every compile. The downside is missed errors *inside libraries* — but those are the library author's problem, not yours.

---

## Part E — Express, Profiling, Production

### Q35. How does the Express middleware chain work?

**Answer:**
Express models a request as a pipeline of middleware functions, each with signature `(req, res, next)`. When a request arrives, Express runs the first middleware. The middleware can:
- End the response (`res.send(...)`) — chain stops.
- Call `next()` — pass control to the next middleware.
- Call `next(err)` — skip ahead to error-handling middleware.
- Do nothing — the request hangs (a common bug).

Middleware is registered in order via `app.use(...)` or `router.use(...)`. Order is critical: `app.use(express.json())` must come before route handlers that read `req.body`; auth middleware must come before the routes it protects.

**Code:**
```js
const app = express();

app.use(express.json());                    // body parser
app.use(requestId);                         // adds req.id
app.use((req, res, next) => {               // logging
  console.log(`${req.method} ${req.url} id=${req.id}`);
  next();
});

app.get('/users/:id', authGuard, async (req, res, next) => {
  try {
    const user = await getUser(req.params.id);
    res.json(user);
  } catch (e) { next(e); }                  // forward to error handler
});

app.use((err, req, res, next) => {          // 4-arg = error handler
  console.error(err);
  res.status(500).json({ error: 'internal' });
});
```

The "4-argument means error handler" is a quirky API — Express decides based on function arity.

**Follow-up 1: Why must async route handlers manually catch errors?**
Because Express's middleware chain pre-dates Promises. If your async handler rejects, Express doesn't see it — the request hangs until timeout. You must `try/catch` and call `next(err)`, or use a wrapper like `express-async-errors`. Express 5 (currently in beta) finally handles this natively.

**Follow-up 2: What's the difference between `app.use` and `app.get`?**
`app.use` matches *all* methods and any URL starting with the given path. `app.get`/`post`/etc. match only that method and an exact path pattern. So `app.use('/api', router)` mounts a router for any HTTP method under `/api`; `app.get('/api/users', ...)` only matches GET on that exact path.

---

### Q36. How do you implement centralized error handling in Express?

**Answer:**
Two layers:

1. **Operational errors** (validation failures, 404s, auth) — throw a custom error class with a status code, let an error-handling middleware translate it to a response.
2. **Programmer errors** (bugs, undefined access) — log and return a generic 500. Never leak stack traces to clients.

```js
class HttpError extends Error {
  constructor(public status: number, message: string, public code?: string) {
    super(message);
  }
}

// Routes throw structured errors
app.get('/users/:id', async (req, res, next) => {
  const user = await db.users.findById(req.params.id);
  if (!user) throw new HttpError(404, 'user not found', 'USER_NOT_FOUND');
  res.json(user);
});

// Single error handler at the end
app.use((err, req, res, next) => {
  if (err instanceof HttpError) {
    return res.status(err.status).json({ error: err.code, message: err.message });
  }
  logger.error({ err, reqId: req.id }, 'unhandled error');
  res.status(500).json({ error: 'INTERNAL' });
});
```

For async routes you need `express-async-errors` (a one-line import that patches Express) or you must wrap every handler in `try { ... } catch (e) { next(e); }`.

**Follow-up 1: What's the difference between operational and programmer errors?**
Operational errors are expected and handled gracefully — they're part of normal operation (bad input, network failure, not-found). Programmer errors indicate a bug — undefined access, calling a method that doesn't exist, type mismatches. Operational errors get translated to API responses; programmer errors should be logged and ideally crash the process so the supervisor restarts it.

**Follow-up 2: How does NestJS handle this differently?**
NestJS has a built-in `HttpException` base class with subclasses like `NotFoundException`, `BadRequestException`. Throwing one is automatically translated to the right HTTP response by a global exception filter. You can also write custom `@Catch(SomeError)` filters for app-specific error types. It's the same pattern, just standardized.

---

### Q37. How do you validate request parameters, query strings, and bodies?

**Answer:**
You don't trust input. Ever. Validation should happen at the edge of your service, before the handler logic runs.

**Approach:** define a schema, run input through it, get either a typed object or a validation error. Three good libraries:

- **Zod** — TypeScript-first, schema *is* the type.
- **Joi** — older, battle-tested, no TS-native inference.
- **class-validator + class-transformer** — used by NestJS via DTOs.

**Code — Zod with Express:**
```ts
import { z } from 'zod';

const CreateUser = z.object({
  email: z.string().email(),
  age: z.number().int().min(13).max(120),
  role: z.enum(['admin', 'user']).default('user'),
});

type CreateUserInput = z.infer<typeof CreateUser>;

app.post('/users', (req, res, next) => {
  const parsed = CreateUser.safeParse(req.body);
  if (!parsed.success) {
    return res.status(400).json({ errors: parsed.error.flatten() });
  }
  // parsed.data is typed as CreateUserInput
  return createUser(parsed.data).then(u => res.json(u)).catch(next);
});
```

**NestJS equivalent:**
```ts
class CreateUserDto {
  @IsEmail() email: string;
  @IsInt() @Min(13) @Max(120) age: number;
}

@Post()
@UsePipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }))
create(@Body() dto: CreateUserDto) { ... }
```

`whitelist` strips unknown fields; `forbidNonWhitelisted` rejects requests that send them. Defense in depth.

**Follow-up 1: Why validate even if the frontend already validates?**
Because anyone can hit your API with curl. Frontend validation is for UX; backend validation is for security and correctness. A malicious or buggy client can send anything — your API must defend itself.

**Follow-up 2: How do you handle sanitization vs validation?**
Validation rejects bad input. Sanitization transforms it (trim whitespace, normalize email casing, strip HTML). Do both — validate the shape and types, then sanitize/normalize before persisting. For HTML/SQL, never sanitize manually — use parameterized queries and a real HTML sanitizer like DOMPurify.

---

### Q38. How do you profile a Node.js app in production?

**Answer:**
Three things to profile, three different tools:

1. **CPU** — where is wall-time / CPU time going?
   - Local: `node --inspect` → Chrome DevTools "Performance" tab → "Record CPU Profile."
   - Production: `clinic flame` or `0x` generates flame graphs.
   - Live: capture a CPU profile on demand via `inspector` API and download.

2. **Memory** — what's allocated and what's retained?
   - `--heapsnapshot-signal=SIGUSR2` lets you `kill -SIGUSR2 <pid>` to dump a snapshot.
   - `clinic heapprofile` for allocation tracking.
   - Open snapshots in Chrome DevTools "Memory" tab.

3. **Event loop** — is the loop blocked?
   - `clinic doctor` for an overview (event loop delay, CPU, memory).
   - `perf_hooks.monitorEventLoopDelay()` for in-process metrics.

**Code — exposing a live profiling endpoint behind auth:**
```js
const inspector = require('inspector');
app.post('/_internal/cpu-profile', requireAdmin, async (req, res) => {
  const session = new inspector.Session();
  session.connect();
  session.post('Profiler.enable');
  session.post('Profiler.start');
  await sleep(30_000);
  session.post('Profiler.stop', (err, { profile }) => {
    res.json(profile);              // download .cpuprofile, open in DevTools
    session.disconnect();
  });
});
```

**Follow-up 1: What's the difference between a flame graph and a CPU profile?**
A CPU profile is the raw sampled stack trace data. A flame graph is one visualization of it — horizontal axis is sample count (proportional to time spent), vertical axis is call stack depth. Wide bars at the top = hot functions. The other useful visualization is "Bottom-Up" / "tree view" in DevTools.

**Follow-up 2: Profiling impacts performance — is it safe in prod?**
CPU profiling adds ~1–5% overhead and is generally safe under normal load. Heap snapshots pause the process briefly (proportional to heap size — for a 2GB heap, expect a few seconds of pause). Capture snapshots during low-traffic windows or on canary instances.

---

### Q39. What does PM2 do? When should you use it vs container orchestration?

**Answer:**
**PM2** is a Node-aware process manager. Core features:
- **Cluster mode** — `pm2 start app.js -i max` forks N copies of your app (one per core) and load-balances with the built-in round-robin.
- **Auto-restart** on crash, with backoff and crash-loop detection.
- **Zero-downtime reload** — `pm2 reload` restarts workers one at a time, waiting for the new one to be ready before killing the old.
- **Log aggregation, monitoring dashboard, startup scripts.**

PM2 is the right answer when you're running on a single VM or a handful of EC2 instances without container orchestration. It gives you "run my app reliably" without setting up systemd unit files or writing your own supervisor.

When you have **Kubernetes, ECS, or similar**, you usually skip PM2 entirely. The orchestrator handles restarts, scaling, health checks, and rolling deploys. Adding PM2 inside a container creates two competing supervisors — the orchestrator thinks the container is healthy as long as PID 1 (PM2) is alive, even if all your workers are crashing inside.

The general rule:
- One VM, no orchestration → PM2.
- Container + orchestrator → run Node directly as PID 1, scale by replicas.

**Follow-up 1: How does cluster mode actually load-balance?**
Node's `cluster` module shares a server socket: the master `accept()`s and round-robins connections to workers. PM2's cluster mode is built on this. On Windows the model is different (workers each `accept()` themselves), which can cause uneven load.

**Follow-up 2: How do you do zero-downtime deploys in Kubernetes if not using PM2?**
Kubernetes rolling update: spin up new pods, wait for readiness probe, kill old pods. The app needs to handle `SIGTERM` — stop accepting new connections, finish in-flight requests, then exit. In Node:
```js
const server = app.listen(3000);
process.on('SIGTERM', () => {
  server.close(() => process.exit(0));   // refuses new conns, drains existing
});
```

---

### Q40. How do you secure a Node.js application? What's the production checklist?

**Answer:**
A non-exhaustive but practical checklist:

1. **Helmet** — `app.use(helmet())`. Sets a stack of security headers: CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy. Cheap insurance.
2. **Rate limiting** — `express-rate-limit` or, better, Redis-backed (`rate-limiter-flexible`) for distributed rate limits across replicas. Apply per-IP for public endpoints, per-user for authenticated endpoints.
3. **Input validation** — every body, query, and param validated (Zod / Joi / class-validator). Reject unknown fields.
4. **Parameterized queries** — never string-concatenate user input into SQL. Use Prisma / TypeORM / Knex parameter binding. NoSQL injection in MongoDB is also real — sanitize `$`-prefixed keys.
5. **Secrets management** — never commit `.env`. Use AWS Secrets Manager / Vault / SOPS. At minimum, ensure CI fails on committed secrets (gitleaks, trufflehog).
6. **Dependencies** — `npm audit` in CI, Snyk / Dependabot for ongoing alerts. Pin lockfile.
7. **HTTPS only** — TLS terminated at the load balancer (ALB, Nginx). Set `app.set('trust proxy', 1)` so `req.ip` reflects the real client.
8. **CORS** — configure explicitly. Don't `cors()` with no options (allows everything).
9. **Authentication** — short-lived JWTs (15 min) + refresh tokens; or session cookies with `httpOnly`, `secure`, `sameSite=strict`.
10. **Logging** — never log passwords, tokens, or PII. Use a redaction-aware logger (pino with `redact` paths).
11. **Process limits** — body size limits (`express.json({ limit: '1mb' })`), request timeouts, slow-down protection against Slowloris (set `server.keepAliveTimeout` and `server.headersTimeout`).
12. **Run as non-root** in containers — `USER node` in your Dockerfile.

**Follow-up 1: How do you defend against ReDoS (regex DoS)?**
Avoid user-controlled regexes. Avoid catastrophic backtracking patterns (`/(a+)+$/` against `aaaaaaX`). Use `safe-regex` to lint your own regex. For untrusted input, use timeouts: run regex in a worker thread with `setTimeout` kill. Newer Node (v17+) has a flag for non-backtracking RegExp engine on linear-time regexes.

**Follow-up 2: How do you implement rate limiting that survives multiple replicas?**
In-memory rate limiters break across replicas — each instance counts separately, attackers just spread across them. Use Redis as the counter store: `INCR` with `EXPIRE`, or a sliding window via sorted sets. Libraries like `rate-limiter-flexible` handle this with Lua scripts for atomicity. For very high throughput, a token-bucket implemented in a Redis Lua script can do tens of thousands of ops/sec.

---

*End of section 01. Next: NestJS & Express deep dive (25 questions).*
