# PART 3 – 100 INTERVIEW QUESTIONS WITH SOLUTIONS

> JavaScript & TypeScript — Complete reference for interview preparation.

---

## JavaScript – Fundamentals (Q1–Q20)

### Q1. What is the difference between `var`, `let`, and `const`? When would you use each?

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisting | Yes (undefined) | Yes (TDZ) | Yes (TDZ) |
| Reassignment | Yes | Yes | No |
| Redeclaration | Yes | No | No |

**Use:** `const` by default, `let` when reassignment is needed, avoid `var` in modern code.

```js
var x = 1; // function-scoped
let y = 2; // block-scoped, mutable
const z = 3; // block-scoped, immutable binding
```

---

### Q2. Explain the Temporal Dead Zone (TDZ) with an example.

TDZ is the period between entering scope and the variable's declaration where accessing `let`/`const` throws `ReferenceError`.

```js
console.log(a); // ReferenceError (TDZ)
let a = 5;
```

---

### Q3. What is hoisting? What gets hoisted and what does not?

Hoisting moves declarations to the top of scope at compile-time.

- **Hoisted (initialized):** `var` (as `undefined`), function declarations.
- **Hoisted (uninitialized → TDZ):** `let`, `const`, `class`.
- **Not hoisted:** function expressions, arrow functions assigned to variables.

```js
foo(); // works
function foo() {}

bar(); // TypeError
var bar = function() {};
```

---

### Q4. What is the difference between `null` and `undefined`?

- `undefined`: variable declared but not assigned (default value).
- `null`: intentional absence of value (assigned by developer).

```js
let a;          // undefined
let b = null;   // null
typeof a;       // "undefined"
typeof b;       // "object"
```

---

### Q5. What does `typeof null` return and why?

Returns `"object"` — a historical bug in JavaScript. In the original implementation, values were tagged with type bits, and objects had a tag of `0`. `null` was represented as a NULL pointer (also `0`), so it was misidentified as an object.

---

### Q6. Explain type coercion. Give 3 unexpected examples.

JavaScript automatically converts types in certain operations.

```js
[] + []        // "" (both become empty strings)
[] + {}        // "[object Object]"
{} + []        // 0 (block + array → number)
1 + "1"        // "11" (number → string)
"5" - 2        // 3 (string → number)
true + true    // 2 (boolean → number)
```

---

### Q7. What is the difference between `==` and `===`? When is `==` acceptable?

- `==` — loose equality with type coercion.
- `===` — strict equality, no coercion.

`==` is acceptable only for `value == null` to check both `null` and `undefined` in one expression.

```js
0 == "0"       // true
0 === "0"      // false
value == null  // true for null or undefined
```

---

### Q8. List all falsy values in JavaScript.

`false`, `0`, `-0`, `0n` (BigInt), `""`, `null`, `undefined`, `NaN`.

Everything else is truthy — including `"0"`, `"false"`, `[]`, `{}`.

---

### Q9. What is the difference between `||` and `??`?

- `||` returns the right side for any falsy left value.
- `??` returns the right side only for `null`/`undefined`.

```js
0 || 5         // 5
0 ?? 5         // 0
null ?? 5      // 5
"" || "hi"     // "hi"
"" ?? "hi"     // ""
```

---

### Q10. What is optional chaining (`?.`) and how does it differ from `&&` chaining?

`?.` short-circuits on `null`/`undefined` only. `&&` short-circuits on any falsy.

```js
user?.address?.city
// equivalent to:
user && user.address && user.address.city
// but ?. allows 0, "", false to pass through
```

---

### Q11. Difference between primitive type and reference type.

- **Primitive:** `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint` — stored by value.
- **Reference:** `object`, `array`, `function` — stored by reference (pointer to heap).

```js
let a = 1, b = a; b = 2;  // a still 1
let x = {n:1}, y = x; y.n = 2;  // x.n is also 2
```

---

### Q12. What happens when you assign an object to another variable?

The reference (pointer) is copied, not the object. Both variables point to the same heap memory.

```js
const a = { x: 1 };
const b = a;
b.x = 99;
console.log(a.x); // 99
```

---

### Q13. How does `const` work with objects and arrays?

`const` makes the **binding** immutable, not the value. The object/array contents can still be mutated.

```js
const arr = [1, 2];
arr.push(3);      // works
arr = [4, 5];     // TypeError
```

To freeze contents: `Object.freeze(arr)`.

---

### Q14. Function declaration vs function expression.

```js
// Declaration — hoisted fully
function foo() {}

// Expression — only variable is hoisted
const bar = function() {};

// Arrow expression
const baz = () => {};
```

Declarations are hoisted with body; expressions are not.

---

### Q15. Can you call a function before it's declared?

Yes for function declarations (fully hoisted), no for expressions (only variable hoisted, value is undefined at call time).

```js
foo(); // OK
function foo() { console.log("ok"); }

bar(); // TypeError: bar is not a function
var bar = function() {};
```

---

### Q16. What is an IIFE?

Immediately Invoked Function Expression — runs as soon as defined. Used to create private scope (before ES6 modules).

```js
(function() {
  const secret = 42;
  console.log(secret);
})();
```

---

### Q17. What is a pure function?

A function with no side effects, returning the same output for the same input.

```js
// Pure
const add = (a, b) => a + b;

// Impure (mutates external state)
let total = 0;
const addTo = (n) => total += n;
```

---

### Q18. Difference between `arguments` and rest parameters.

| `arguments` | Rest `...args` |
|---|---|
| Array-like, not array | Real array |
| Not in arrow functions | Works in arrows |
| No array methods | Has `.map`, `.filter` |

```js
function legacy() { console.log(arguments); }
const modern = (...args) => args.map(x => x * 2);
```

---

### Q19. What is currying? Write `add(1)(2)(3)`.

Currying transforms `f(a,b,c)` into `f(a)(b)(c)`.

```js
const add = a => b => c => a + b + c;
add(1)(2)(3); // 6

// Generic curry
const curry = (fn) => {
  return function curried(...args) {
    return args.length >= fn.length
      ? fn(...args)
      : (...next) => curried(...args, ...next);
  };
};
```

---

### Q20. What does the `in` operator do? How is it different from `hasOwnProperty`?

`in` checks if a property exists **anywhere on the prototype chain**. `hasOwnProperty` checks only the **own** properties.

```js
const obj = { a: 1 };
"a" in obj;                       // true
"toString" in obj;                // true (inherited)
obj.hasOwnProperty("toString");   // false
```

---

## JavaScript – Core Concepts (Q21–Q50)

### Q21. Explain the scope chain with a nested function.

Inner functions can access variables from outer scopes by walking up the chain.

```js
const a = 1;
function outer() {
  const b = 2;
  function inner() {
    const c = 3;
    console.log(a, b, c); // 1 2 3
  }
  inner();
}
```

---

### Q22. What is a closure? Write a counter.

A closure is a function that remembers variables from its lexical scope, even when executed outside that scope.

```js
const createCounter = () => {
  let count = 0;
  return () => ++count;
};

const counter = createCounter();
counter(); // 1
counter(); // 2
```

---

### Q23. Difference between `.call()`, `.apply()`, and `.bind()`.

- `.call(this, a, b)` — invokes immediately with args list.
- `.apply(this, [a, b])` — invokes immediately with args array.
- `.bind(this, a)` — returns a new function with `this` bound.

```js
function greet(g, p) { return `${g}, ${this.name}${p}`; }
greet.call({name:"Alex"}, "Hi", "!");      // "Hi, Alex!"
greet.apply({name:"Alex"}, ["Hi", "!"]);   // "Hi, Alex!"
const bound = greet.bind({name:"Alex"});
bound("Hey", ".");                          // "Hey, Alex."
```

---

### Q24. How does `this` behave in regular vs arrow functions?

- **Regular:** `this` depends on how the function is **called** (caller).
- **Arrow:** `this` is **lexically inherited** from the enclosing scope at definition time.

```js
const obj = {
  name: "A",
  reg() { return this.name; },     // "A"
  arr: () => this.name             // undefined (outer this)
};
```

---

### Q25. How do you fix the "lost `this`" problem in callbacks?

Three options:

```js
// 1. Arrow function (preserves outer this)
btn.addEventListener("click", () => this.handle());

// 2. .bind()
btn.addEventListener("click", this.handle.bind(this));

// 3. Save reference
const self = this;
btn.addEventListener("click", function() { self.handle(); });
```

---

### Q26. What is the prototype chain?

Every object has an internal `[[Prototype]]` link. When you access a property, JS walks up the chain until found or `null`.

```js
const animal = { eats: true };
const rabbit = Object.create(animal);
rabbit.jumps = true;
rabbit.eats;  // true (from prototype)
```

---

### Q27. Prototypal vs classical inheritance.

- **Classical (Java, C++):** classes inherit from classes; objects are instances.
- **Prototypal (JS):** objects inherit directly from other objects. Classes in JS are syntactic sugar over prototypes.

---

### Q28. What does `Object.create(null)` give you?

An object with **no prototype** — no inherited methods like `toString`, `hasOwnProperty`. Useful as a pure dictionary/map to avoid key collisions.

```js
const dict = Object.create(null);
dict.toString = "hi"; // safe — no inherited toString
```

---

### Q29. `Object.freeze()` vs `Object.seal()`.

- `freeze`: no add, no delete, **no modify** existing.
- `seal`: no add, no delete, **but can modify** existing values.

Both are shallow.

---

### Q30. How do you deep clone? Limitations of `JSON.parse(JSON.stringify())`?

```js
const clone = structuredClone(obj); // best modern way
```

Limitations of `JSON.parse(JSON.stringify())`:
- Loses `undefined`, functions, `Symbol`.
- Converts `Date` to string.
- Throws on circular references.
- Loses `Map`, `Set`, `RegExp` semantics.

---

### Q31. `for...in` vs `for...of`.

- `for...in` — iterates over **enumerable keys** (including inherited).
- `for...of` — iterates over **values of iterables** (arrays, strings, Maps, Sets).

```js
for (const key in {a:1,b:2}) {}      // "a", "b"
for (const val of [10,20]) {}         // 10, 20
```

---

### Q32. `Map` vs `Object` trade-offs.

| Feature | Map | Object |
|---|---|---|
| Key types | any | string/symbol |
| Iteration order | insertion | mixed |
| Size | `.size` | manual |
| Performance | better for frequent add/delete | better for static keys |

Choose `Map` for dynamic key-value collections.

---

### Q33. What is a `WeakMap`?

Like `Map`, but:
- Keys must be objects.
- Keys are **weakly referenced** — garbage-collected when no other refs exist.
- Not iterable, no `.size`.

Used for private data and caches tied to object lifetime.

---

### Q34. What is a `Symbol`?

A primitive that creates a unique, non-string identifier. Useful as object keys to avoid name collisions.

```js
const ID = Symbol("id");
const user = { [ID]: 123 };
// ID won't conflict with any string key
```

---

### Q35. What is the Event Loop?

A mechanism that processes async tasks. Key parts:
1. **Call Stack** — synchronous code execution.
2. **Microtask Queue** — Promises, `queueMicrotask`, `MutationObserver`.
3. **Macrotask Queue** — `setTimeout`, `setInterval`, I/O, UI events.

Loop: empty call stack → drain all microtasks → run 1 macrotask → repeat.

---

### Q36. Order of: `setTimeout(fn,0)`, Promise `.then()`, sync code?

```js
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");

// Output: 1, 4, 3, 2
```

Sync → microtasks → macrotasks.

---

### Q37. Microtasks vs macrotasks.

- **Microtasks:** `Promise.then`, `queueMicrotask`, `MutationObserver`. All drained before next macrotask.
- **Macrotasks:** `setTimeout`, `setInterval`, `setImmediate` (Node), I/O, UI rendering.

---

### Q38. Generator functions and `yield`.

A generator pauses and resumes execution.

```js
function* gen() {
  yield 1;
  yield 2;
  return 3;
}
const g = gen();
g.next(); // {value:1, done:false}
g.next(); // {value:2, done:false}
g.next(); // {value:3, done:true}
```

---

### Q39. How do you make a custom iterable?

Implement `[Symbol.iterator]()` returning `{ next() }`.

```js
const range = {
  from: 1, to: 3,
  [Symbol.iterator]() {
    let cur = this.from, last = this.to;
    return {
      next() {
        return cur <= last
          ? { value: cur++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
};
[...range]; // [1,2,3]
```

---

### Q40. `Promise.all` vs `allSettled` vs `race` vs `any`.

| Method | Resolves when | Rejects when |
|---|---|---|
| `all` | All fulfill | Any rejects |
| `allSettled` | All settle (never rejects) | — |
| `race` | First settles (either way) | First rejects |
| `any` | First fulfills | All reject |

---

### Q41. What if you forget `await`?

You get the Promise object instead of the resolved value. Errors become unhandled rejections.

```js
const result = fetchUser(); // Promise<User>, not User
result.id; // undefined
```

---

### Q42. Why doesn't `await` inside `forEach` work?

`forEach` doesn't await callbacks; they run in parallel and `forEach` returns before they finish.

```js
// Broken
arr.forEach(async (x) => await save(x));

// Fixed (sequential)
for (const x of arr) await save(x);

// Fixed (parallel)
await Promise.all(arr.map(save));
```

---

### Q43. Sequential vs parallel async.

```js
// Sequential (slow)
const a = await fetchA();
const b = await fetchB();

// Parallel (fast)
const [a, b] = await Promise.all([fetchA(), fetchB()]);
```

---

### Q44. How to handle errors in async/await?

```js
try {
  const data = await fetchData();
} catch (err) {
  console.error(err);
} finally {
  cleanup();
}

// Or wrap in safe helper
const safe = (p) => p.then(d => [null,d]).catch(e => [e,null]);
const [err, data] = await safe(fetchData());
```

---

### Q45. What is the `Proxy` object?

Wraps an object to intercept operations (`get`, `set`, `has`, `deleteProperty`, etc.).

```js
const log = new Proxy({}, {
  get(t, k) { console.log("get", k); return t[k]; },
  set(t, k, v) { console.log("set", k, v); t[k]=v; return true; }
});
log.name = "Alex"; // logs "set name Alex"
```

Used for validation, observability, reactivity (Vue 3 uses this).

---

### Q46. Implement Observer/EventEmitter.

```js
class EventEmitter {
  constructor() { this.events = {}; }
  on(e, fn)   { (this.events[e] ??= []).push(fn); }
  off(e, fn)  { this.events[e] = (this.events[e]||[]).filter(f=>f!==fn); }
  emit(e, ...a){ (this.events[e]||[]).forEach(fn => fn(...a)); }
}
```

---

### Q47. Common causes of memory leaks.

1. **Forgotten timers/intervals** — clear them.
2. **Detached DOM nodes held by JS refs.**
3. **Closures holding large data unnecessarily.**
4. **Event listeners not removed.**
5. **Global variables.**

---

### Q48. CommonJS vs ES Modules.

| CommonJS | ES Modules |
|---|---|
| `require` / `module.exports` | `import` / `export` |
| Synchronous | Asynchronous (static) |
| Dynamic | Static analyzable |
| Node default (old) | Browser + Node (modern) |

---

### Q49. What is tree-shaking?

Dead-code elimination by bundlers. Works because ES Modules are statically analyzable — the bundler knows which exports are used. CommonJS is dynamic and harder to tree-shake.

---

### Q50. `structuredClone()` vs spread clone.

- Spread (`{...obj}`) — **shallow**, top-level only.
- `structuredClone(obj)` — **deep**, handles cycles, `Date`, `Map`, `Set`, `ArrayBuffer`.

Spread cannot deep-clone nested objects; `structuredClone` can.

---

## JavaScript – Advanced (Q51–Q60)

### Q51. `Object.keys/values/entries/fromEntries`.

```js
const obj = { a: 1, b: 2 };
Object.keys(obj);      // ["a","b"]
Object.values(obj);    // [1,2]
Object.entries(obj);   // [["a",1],["b",2]]
Object.fromEntries([["a",1],["b",2]]); // {a:1,b:2}
```

---

### Q52. How `reduce` works. Rewrite `map` and `filter`.

```js
// Map
const map = (arr, fn) =>
  arr.reduce((acc, x, i) => (acc.push(fn(x, i)), acc), []);

// Filter
const filter = (arr, fn) =>
  arr.reduce((acc, x, i) => (fn(x, i) && acc.push(x), acc), []);
```

---

### Q53. What is `flatMap`?

`flatMap(fn)` = `map(fn).flat(1)` but in one pass and more efficient.

```js
[1,2,3].flatMap(x => [x, x*2]); // [1,2,2,4,3,6]
```

Difference: `flatMap` only flattens **one level**; `map().flat(Infinity)` flattens deeply.

---

### Q54. Implement memoize.

```js
const memoize = (fn) => {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
};
```

---

### Q55. Singleton with private static field.

```js
class Db {
  static #instance;
  constructor() {
    if (Db.#instance) return Db.#instance;
    Db.#instance = this;
  }
}
new Db() === new Db(); // true
```

---

### Q56. Composition vs inheritance.

- **Inheritance:** is-a (`Dog extends Animal`). Tight coupling, fragile base class problem.
- **Composition:** has-a (`Dog has Legs, Tail`). Reusable behaviors, easier to test.

Composition is favored because it's more flexible and avoids deep hierarchies.

```js
const canEat = (s) => ({ eat: () => `${s.name} eats` });
const dog = { name: "Rex", ...canEat({name:"Rex"}) };
```

---

### Q57. Debounce vs throttle.

- **Debounce:** wait for N ms of silence before firing (e.g., search input).
- **Throttle:** fire at most once per N ms (e.g., scroll handler).

```js
const debounce = (fn, ms) => {
  let t;
  return (...a) => { clearTimeout(t); t = setTimeout(() => fn(...a), ms); };
};

const throttle = (fn, ms) => {
  let last = 0;
  return (...a) => {
    const now = Date.now();
    if (now - last >= ms) { last = now; fn(...a); }
  };
};
```

---

### Q58. `setTimeout(fn,0)` vs `setImmediate(fn)` in Node.

- `setTimeout(fn, 0)` — runs in the **timers** phase.
- `setImmediate(fn)` — runs in the **check** phase, right after I/O.

Inside an I/O callback, `setImmediate` fires before `setTimeout(fn, 0)`.

---

### Q59. `process.nextTick` vs Promise microtasks.

- `process.nextTick` — Node-specific, drained **before** Promise microtasks. Higher priority.
- `Promise.then` — standard microtasks.

Both run before next macrotask.

---

### Q60. Simple pub/sub.

```js
const pubsub = {
  subs: {},
  subscribe(topic, fn) {
    (this.subs[topic] ??= []).push(fn);
    return () => this.unsubscribe(topic, fn);
  },
  unsubscribe(topic, fn) {
    this.subs[topic] = (this.subs[topic]||[]).filter(f => f !== fn);
  },
  publish(topic, data) {
    (this.subs[topic]||[]).forEach(fn => fn(data));
  }
};
```

---

## TypeScript – Fundamentals (Q61–Q80)

### Q61. `any` vs `unknown`.

- `any` — opts out of type checking. Dangerous.
- `unknown` — type-safe `any`; must narrow before use.

```ts
let a: any = 1;       a.foo();   // no error (unsafe)
let b: unknown = 1;   b.foo();   // error — narrow first
if (typeof b === "string") b.toUpperCase();
```

---

### Q62. `void` vs `never`.

- `void` — function returns nothing (or `undefined`).
- `never` — function **never returns** (throws or infinite loop).

```ts
function log(): void { console.log("hi"); }
function fail(): never { throw new Error("x"); }
```

`never` is the bottom type — assignable to everything; nothing assignable to it (except `never`).

---

### Q63. Tuple vs array.

- **Array:** `number[]` — any length, all numbers.
- **Tuple:** `[string, number]` — fixed length, ordered types.

```ts
const point: [number, number] = [1, 2];
const named: [name: string, age: number] = ["A", 1];
```

---

### Q64. `interface` vs `type`.

| Feature | interface | type |
|---|---|---|
| Declaration merging | Yes | No |
| Extends | `extends` | `&` |
| Unions | No | Yes |
| Primitives, tuples | No | Yes |

Prefer `interface` for object shapes you may extend; `type` for unions/aliases/mapped types.

---

### Q65. Declaration merging.

Multiple `interface` declarations with the same name combine into one.

```ts
interface User { name: string; }
interface User { age: number; }
const u: User = { name: "A", age: 1 };
```

Useful for extending third-party types (e.g., adding props to `Window`, `Express.Request`).

---

### Q66. `readonly` vs `const`.

- `const` — variable binding cannot change.
- `readonly` — property cannot be reassigned.

```ts
const arr = [1, 2];   arr.push(3);   // OK
const tup: readonly number[] = [1]; tup.push(2); // error
```

---

### Q67. Union and discriminated union.

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function area(s: Shape) {
  switch (s.kind) {                       // discriminator
    case "circle": return Math.PI * s.radius ** 2;
    case "square": return s.side ** 2;
  }
}
```

The shared literal `kind` allows TS to narrow precisely.

---

### Q68. Intersection types.

```ts
interface Loggable { log(): void; }
interface Saveable { save(): void; }
type Entity = Loggable & Saveable;

const e: Entity = { log() {}, save() {} };
```

---

### Q69. `keyof` example.

```ts
function get<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
get({ a: 1, b: "x" }, "a"); // number
```

---

### Q70. `typeof` in TS type position.

JS `typeof` returns a string at runtime. TS `typeof` extracts the **static type** of a variable.

```ts
const user = { id: 1, name: "A" };
type User = typeof user; // { id: number; name: string }
```

---

### Q71. Type inference vs explicit annotations.

TS infers types from initialization. Add explicit annotations for:
- Function parameters.
- Public API boundaries.
- When inference is too narrow/wide.
- Empty arrays (`let x: string[] = []`).

---

### Q72. Generic constraints.

```ts
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("abc", "ab");   // string
longest([1,2], [3]);    // number[]
```

---

### Q73. Generic `Stack<T>`.

```ts
class Stack<T> {
  private items: T[] = [];
  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
  get size(): number { return this.items.length; }
}
```

---

### Q74. `Partial`, `Required`, `Readonly`, `Pick`.

```ts
interface User { id: number; name: string; email?: string; }

type A = Partial<User>;        // all optional
type B = Required<User>;       // all required (email becomes required)
type C = Readonly<User>;       // all readonly
type D = Pick<User, "id"|"name">; // only id and name
```

---

### Q75. `Omit<T, K>`.

Creates a type with specified keys **removed**.

```ts
interface User { id: number; name: string; password: string; }
type PublicUser = Omit<User, "password">;
// { id: number; name: string }
```

Practical use: hide sensitive fields when sending data to clients.

---

### Q76. `ReturnType<T>`.

Extracts a function's return type.

```ts
function getUser() { return { id: 1, name: "A" }; }
type User = ReturnType<typeof getUser>;
// { id: number; name: string }
```

---

### Q77. `NonNullable<T>`.

Removes `null` and `undefined` from a type.

```ts
type Maybe = string | null | undefined;
type Defined = NonNullable<Maybe>; // string
```

Useful after API filtering: `arr.filter(Boolean) as NonNullable<T>[]`.

---

### Q78. Conditional types — unwrap a Promise.

```ts
type Unwrap<T> = T extends Promise<infer U> ? U : T;

type A = Unwrap<Promise<string>>; // string
type B = Unwrap<number>;          // number
```

---

### Q79. Mapped type — `Nullable<T>`.

```ts
type Nullable<T> = { [K in keyof T]: T[K] | null };

interface User { id: number; name: string; }
type N = Nullable<User>;
// { id: number | null; name: string | null }
```

---

### Q80. Template literal type — getter names.

```ts
type Getter<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { name: string; age: number; }
type UserGetters = Getter<User>;
// { getName: () => string; getAge: () => number }
```

---

## TypeScript – Advanced (Q81–Q100)

### Q81. Four types of decorators.

1. **Class decorator** — `@Logger class Foo {}`
2. **Method decorator** — `@Bound method() {}`
3. **Accessor decorator** — applied to getter/setter
4. **Property decorator** — `@Required name: string`
5. **(Parameter decorator)** — `method(@Inject param)`

(Stage-2 vs stage-3 syntax differs; TS supports legacy via `experimentalDecorators`.)

---

### Q82. `experimentalDecorators` and `emitDecoratorMetadata`.

- `experimentalDecorators: true` — enables legacy decorator syntax.
- `emitDecoratorMetadata: true` — emits type info at runtime via `reflect-metadata`, used by NestJS, TypeORM for DI.

---

### Q83. `strictNullChecks`.

When `true`, `null` and `undefined` are no longer assignable to other types.

```ts
let s: string = null; // error with strictNullChecks
let s: string | null = null; // OK
```

Forces explicit handling of nullable values.

---

### Q84. The `!` non-null assertion.

Tells TS "trust me, this is not null/undefined."

```ts
const el = document.querySelector("#x")!; // HTMLElement, not null
```

**Avoid when** runtime values can actually be null — prefer narrowing.

---

### Q85. Custom type guard.

```ts
interface Dog { bark(): void; }
interface Cat { meow(): void; }

function isDog(a: Dog | Cat): a is Dog {
  return (a as Dog).bark !== undefined;
}

if (isDog(pet)) pet.bark();
```

---

### Q86. Assertion function (`asserts`).

```ts
function assertDefined<T>(v: T): asserts v is NonNullable<T> {
  if (v == null) throw new Error("Required");
}

let x: string | undefined = getValue();
assertDefined(x);
x.toUpperCase(); // string, no narrowing block needed
```

Difference: guards return `boolean`; assertions throw and narrow the outer scope.

---

### Q87. Structural vs nominal typing.

- **Structural** (TS, Go): types compatible if shapes match.
- **Nominal** (Java, C#): types compatible only if explicitly declared same.

```ts
interface Point { x: number; y: number; }
const p: Point = { x:1, y:2 }; // works even without declaring "Point"
```

To simulate nominal in TS: brands.

```ts
type UserId = string & { __brand: "UserId" };
```

---

### Q88. `infer` — first arg type.

```ts
type FirstArg<T> = T extends (a: infer A, ...rest: any[]) => any ? A : never;

type A = FirstArg<(x: number, y: string) => void>; // number
```

---

### Q89. `DeepPartial<T>`.

```ts
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

interface Config { server: { host: string; port: number; }; debug: boolean; }
type P = DeepPartial<Config>;
// { server?: { host?: string; port?: number; }; debug?: boolean; }
```

---

### Q90. `as const`.

Freezes literal types instead of widening.

```ts
const a = ["red", "green"];           // string[]
const b = ["red", "green"] as const;  // readonly ["red", "green"]
type Color = typeof b[number];        // "red" | "green"
```

---

### Q91. `Record<string, T>`.

```ts
const userScores: Record<string, number> = {};
userScores["alex"] = 99;
userScores["sara"] = 87;
```

Equivalent to `{ [key: string]: number }` but more concise.

---

### Q92. `satisfies` vs `as`.

- `as` — type assertion, can be wrong, removes safety.
- `satisfies` — checks the value conforms to a type while preserving the narrower inferred type.

```ts
const config = {
  host: "localhost",
  port: 3000,
} satisfies Record<string, string | number>;

config.port.toFixed(2); // works — TS knows it's a number
// With `as Record<string, string|number>`, port would be string|number
```

---

### Q93. Extending third-party types.

```ts
// add.d.ts
import "express";
declare module "express" {
  interface Request {
    user?: { id: number; role: string };
  }
}
```

Now `req.user` is typed across your project.

---

### Q94. `.d.ts` files.

Declaration files containing only types — no implementation. Created manually when:
- A JS library has no built-in types.
- You need to declare global ambient types.
- Sharing types across packages.

---

### Q95. `import type` vs regular import.

`import type` imports types only — erased at compile time, never affects runtime, helps avoid circular deps and reduces bundle size.

```ts
import type { User } from "./types";    // erased
import { fetchUser } from "./api";       // emitted
```

---

### Q96. `namespace` in modern TS.

`namespace` was used pre-modules to group types. **Not recommended** today — prefer ES modules (one file = one module). Still acceptable for ambient global type augmentation in `.d.ts`.

---

### Q97. Typing `this` in functions.

```ts
function show(this: { name: string }) {
  return this.name;
}
show.call({ name: "Alex" }); // OK
show.call({});               // error: missing name
```

---

### Q98. Class private: `#` vs `private`.

- `private` (TS keyword) — **compile-time only**; at runtime, the property is accessible.
- `#field` (ES private) — **truly private** at runtime; can't be accessed outside the class.

```ts
class A {
  private a = 1;   // accessible at runtime
  #b = 2;          // hard-private
}
```

---

### Q99. `Awaited<T>`.

Recursively unwraps `Promise<Promise<...>>` to the eventual value.

```ts
type A = Awaited<Promise<string>>;            // string
type B = Awaited<Promise<Promise<number>>>;   // number
```

Useful when typing `await` results from generic functions.

---

### Q100. Full narrowing chain from `unknown` to nested property.

```ts
function getCity(input: unknown): string | undefined {
  if (
    typeof input === "object" &&
    input !== null &&
    "address" in input &&
    typeof (input as any).address === "object" &&
    (input as any).address !== null &&
    "city" in (input as any).address &&
    typeof (input as any).address.city === "string"
  ) {
    return (input as { address: { city: string } }).address.city;
  }
  return undefined;
}

// Cleaner with a type guard + Zod-like schema:
const isUserWithCity = (v: unknown): v is { address: { city: string } } =>
  typeof v === "object" && v !== null &&
  "address" in v && typeof (v as any).address?.city === "string";

function safe(input: unknown) {
  if (isUserWithCity(input)) return input.address.city;
}
```

---

## Quick Revision Checklist

| Topic | Confident | Needs Review |
|---|---|---|
| `var` / `let` / `const` & TDZ | ☐ | ☐ |
| Closures & Scope chain | ☐ | ☐ |
| Prototype Chain | ☐ | ☐ |
| `this` binding | ☐ | ☐ |
| Event Loop / Microtasks | ☐ | ☐ |
| Promises / async-await | ☐ | ☐ |
| Generators / Iterators | ☐ | ☐ |
| Memory leaks & Modules | ☐ | ☐ |
| Reduce / Currying / Memoize | ☐ | ☐ |
| Debounce / Throttle | ☐ | ☐ |
| `any` vs `unknown` | ☐ | ☐ |
| Generics & Constraints | ☐ | ☐ |
| Utility Types | ☐ | ☐ |
| Conditional / Mapped Types | ☐ | ☐ |
| `infer` & Template literal types | ☐ | ☐ |
| Type Guards & Assertions | ☐ | ☐ |
| Decorators | ☐ | ☐ |
| `satisfies` vs `as` | ☐ | ☐ |
| Declaration merging | ☐ | ☐ |
| `.d.ts` & `import type` | ☐ | ☐ |

---

*Tip: For each question, first try answering aloud, then write a small code snippet to verify your understanding before checking the solution.*
