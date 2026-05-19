# 🚀 ITC Infotech – Fullstack MERN Interview Prep
### Role: Fullstack Developer (MERN) | Experience: ~2 Years

---

## 📌 Table of Contents
1. [JavaScript & ES6+](#javascript--es6)
2. [React.js](#reactjs)
3. [Node.js](#nodejs)
4. [Express.js](#expressjs)
5. [MongoDB](#mongodb)
6. [REST API & System Design](#rest-api--system-design)
7. [Version Control & Tools](#version-control--tools)
8. [HR / Behavioural Round](#hr--behavioural-round)
9. [Topics to Revise](#-topics-to-revise)

---

## JavaScript & ES6+

**Q1. What is the difference between `var`, `let`, and `const`?**
> `var` is function-scoped and hoisted with `undefined`. `let` and `const` are block-scoped; `let` can be reassigned, `const` cannot. Use `const` by default, `let` when reassignment is needed.

**Q2. Explain closures with an example.**
> A closure is a function that retains access to its outer scope even after the outer function has returned.
```js
function counter() {
  let count = 0;
  return () => ++count;
}
const inc = counter();
inc(); // 1
inc(); // 2
```

**Q3. What is the event loop? How does Node.js handle async operations?**
> JS is single-threaded. The event loop processes the call stack, then the microtask queue (Promises), then the macrotask queue (setTimeout, setInterval). Node.js offloads I/O to libuv, which uses worker threads internally, then pushes callbacks to the event loop.

**Q4. What is the difference between `Promise` and `async/await`?**
> Both handle async code. `async/await` is syntactic sugar over Promises, making code look synchronous and easier to read/debug. Errors are caught via `try/catch` with async/await.

**Q5. What are higher-order functions? Give examples.**
> Functions that take another function as an argument or return a function. Examples: `map`, `filter`, `reduce`.
```js
const doubled = [1, 2, 3].map(n => n * 2); // [2, 4, 6]
```

**Q6. Explain `this` keyword in JavaScript.**
> `this` refers to the context in which a function is called. In regular functions, it depends on the caller. In arrow functions, `this` is lexically inherited from the enclosing scope.

**Q7. What is destructuring? Give an object and array example.**
```js
const { name, age } = { name: 'Arjun', age: 25 };
const [first, , third] = [1, 2, 3];
```

**Q8. What is the difference between `==` and `===`?**
> `==` does type coercion before comparison. `===` checks both value and type without coercion. Always prefer `===`.

**Q9. What is a Promise chain? What problem does it solve?**
> Chaining `.then()` calls avoids callback hell, making async code sequential and readable.

**Q10. What are generators and iterators?**
> A generator function uses `function*` and `yield` to pause execution and return values lazily. Useful for custom iteration logic.

---

## React.js

**Q11. What is the Virtual DOM and how does React use it?**
> React maintains a lightweight copy of the real DOM (Virtual DOM). On state changes, it diffs the new VDOM with the old one (reconciliation) and applies only the minimal real DOM updates.

**Q12. Explain the difference between controlled and uncontrolled components.**
> Controlled: form value is controlled by React state via `value` and `onChange`. Uncontrolled: form value is managed by the DOM itself via `ref`.

**Q13. What are React hooks? Name and explain the most commonly used ones.**
- `useState` – manages local state
- `useEffect` – side effects (API calls, subscriptions)
- `useContext` – consume context without prop drilling
- `useRef` – persist values across renders or access DOM nodes
- `useMemo` – memoize expensive calculations
- `useCallback` – memoize function references

**Q14. What is the difference between `useMemo` and `useCallback`?**
> `useMemo` memoizes the **return value** of a function. `useCallback` memoizes the **function itself**. Both help avoid unnecessary re-renders.

**Q15. Explain React's component lifecycle with hooks.**
> - Mount: `useEffect(() => { ... }, [])` — runs once after mount
> - Update: `useEffect(() => { ... }, [dep])` — runs when `dep` changes
> - Unmount: return a cleanup function from `useEffect`

**Q16. What is prop drilling? How do you avoid it?**
> Passing props through many layers of components that don't need them. Avoided using React Context, Redux, or Zustand.

**Q17. What is `useEffect` dependency array and what happens if you omit it?**
> If omitted, the effect runs after every render. Empty `[]` means run once on mount. Specific deps `[val]` means run when `val` changes.

**Q18. What is lazy loading in React?**
> Loading components only when needed using `React.lazy()` and `Suspense`. Reduces initial bundle size.
```js
const Dashboard = React.lazy(() => import('./Dashboard'));
```

**Q19. What is Redux? Explain the data flow.**
> Redux is a state management library. Flow: UI dispatches an **action** → **reducer** processes it → **store** updates → UI re-renders via `useSelector`.

**Q20. What is the difference between `useContext` and Redux?**
> `useContext` is built-in and good for simple/small apps. Redux is better for large apps with complex state, middleware needs (like async via Redux Thunk/Saga), and DevTools support.

**Q21. How do you optimise React performance?**
> - `React.memo` to prevent unnecessary re-renders
> - `useMemo` / `useCallback`
> - Code splitting with `React.lazy`
> - Virtualization for long lists (`react-window`)
> - Avoid inline functions/objects in JSX

**Q22. What is the difference between `key` prop in a list and its importance?**
> React uses `key` to identify which list items changed, added, or removed during reconciliation. Keys should be unique and stable (avoid using index as key if list can reorder).

---

## Node.js

**Q23. What is Node.js? How is it different from browser JavaScript?**
> Node.js is a server-side runtime for JavaScript built on Chrome's V8 engine. Unlike browser JS, it has no DOM but has access to the file system, OS, networking, and modules like `fs`, `http`, `path`.

**Q24. What is the difference between synchronous and asynchronous code in Node?**
> Synchronous code blocks the thread until complete. Asynchronous code (callbacks, Promises, async/await) is non-blocking and allows other operations to continue.

**Q25. What is middleware in Node.js/Express?**
> A function that has access to `req`, `res`, and `next`. It can modify the request/response, end the cycle, or pass control to the next middleware.

**Q26. Explain `require` vs ES Modules (`import/export`).**
> `require` is CommonJS (synchronous, dynamic). `import/export` is ES Module (static, tree-shakeable). Node supports both; use `.mjs` or `"type": "module"` in `package.json` for ESM.

**Q27. What is the purpose of `package.json`?**
> Contains project metadata, dependencies, devDependencies, scripts, and the entry point. `npm install` uses it to set up the project.

**Q28. How do you handle errors in async Node.js code?**
> Using `try/catch` with `async/await`, or `.catch()` on Promises. Always use a centralized Express error-handling middleware for unhandled errors.

**Q29. What is clustering in Node.js and why is it used?**
> Node is single-threaded; clustering creates multiple worker processes (one per CPU core) using the `cluster` module, allowing horizontal scaling on the same machine.

**Q30. What is the difference between `process.nextTick()` and `setImmediate()`?**
> `process.nextTick()` runs before the next event loop iteration (highest priority). `setImmediate()` runs in the check phase of the current event loop iteration — after I/O callbacks.

---

## Express.js

**Q31. How do you structure a scalable Express project?**
```
src/
├── controllers/
├── routes/
├── models/
├── middleware/
├── config/
└── app.js
```

**Q32. How do you handle CORS in Express?**
```js
const cors = require('cors');
app.use(cors({ origin: 'http://localhost:3000' }));
```

**Q33. What is the difference between `app.use()` and `app.get()`?**
> `app.use()` matches any HTTP method and is used for middleware. `app.get()` matches only GET requests on a specific path.

**Q34. How do you validate request data in Express?**
> Using libraries like `express-validator` or `joi`. Always validate on the server side even if client-side validation exists.

**Q35. How do you implement JWT authentication in Express?**
> 1. On login, sign a JWT with a secret: `jwt.sign({ userId }, SECRET, { expiresIn: '1d' })`
> 2. Send token to client (store in httpOnly cookie or localStorage)
> 3. On protected routes, verify token in middleware: `jwt.verify(token, SECRET)`

**Q36. What is rate limiting and how do you implement it?**
> Restricting the number of requests from a client in a time window. Use `express-rate-limit`:
```js
const rateLimit = require('express-rate-limit');
app.use(rateLimit({ windowMs: 15 * 60 * 1000, max: 100 }));
```

---

## MongoDB

**Q37. What is MongoDB? How is it different from SQL databases?**
> MongoDB is a NoSQL document database storing data as BSON (JSON-like) documents. Unlike SQL, it has no fixed schema, uses collections instead of tables, and scales horizontally easily.

**Q38. Explain indexing in MongoDB.**
> Indexes speed up queries by creating a data structure (B-tree) on a field. Use `db.collection.createIndex({ field: 1 })`. Without indexes, MongoDB does a full collection scan (COLLSCAN).

**Q39. What is Mongoose and why use it?**
> Mongoose is an ODM (Object Document Mapper) for MongoDB in Node.js. It adds schema validation, middleware (pre/post hooks), and a cleaner API for queries.

**Q40. What is the difference between `find()` and `findOne()`?**
> `find()` returns a cursor of all matching documents. `findOne()` returns the first matching document.

**Q41. How do you perform aggregation in MongoDB?**
> Using the aggregation pipeline with stages like `$match`, `$group`, `$sort`, `$project`, `$lookup`.
```js
db.orders.aggregate([
  { $match: { status: 'completed' } },
  { $group: { _id: '$userId', total: { $sum: '$amount' } } }
]);
```

**Q42. What is `$lookup` in MongoDB?**
> It performs a left outer join with another collection, similar to SQL JOIN.

**Q43. What are Mongoose virtuals and middleware?**
> **Virtuals**: computed properties not stored in DB (e.g., `fullName` from `firstName + lastName`).
> **Middleware (hooks)**: `pre` and `post` hooks on operations like `save`, `find`, `remove`.

**Q44. How do you handle relationships in MongoDB?**
> Two approaches:
> - **Embedding**: store related data inside the same document (good for 1-to-few)
> - **Referencing**: store ObjectId and use `$lookup`/`populate()` (good for 1-to-many or many-to-many)

**Q45. What is the difference between `updateOne` and `findOneAndUpdate`?**
> `updateOne` updates and returns the operation result. `findOneAndUpdate` updates and returns the modified (or original) document.

---

## REST API & System Design

**Q46. What are REST principles?**
> Stateless, Client-Server, Cacheable, Uniform Interface, Layered System, Code on Demand (optional). Each request must contain all info needed to process it.

**Q47. What are HTTP status codes you commonly use?**
| Code | Meaning |
|------|---------|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Internal Server Error |

**Q48. How would you design a URL shortener system?**
> - Generate a short unique hash for each URL (nanoid or base62 encoding)
> - Store `{ shortCode, originalUrl, userId, createdAt }` in MongoDB
> - On redirect, look up by shortCode and send 301/302
> - Add TTL index for expiry, caching with Redis for hot URLs

**Q49. What is the difference between authentication and authorization?**
> **Authentication**: verifying who you are (login with JWT/session).
> **Authorization**: verifying what you're allowed to do (role-based access control).

**Q50. How would you design a basic MERN e-commerce app?**
> - **Frontend**: React + Redux (cart state), React Router (pages)
> - **Backend**: Express REST APIs (products, orders, users)
> - **DB**: MongoDB with collections: `users`, `products`, `orders`
> - **Auth**: JWT with refresh tokens
> - **Payment**: Razorpay/Stripe webhook integration

---

## Version Control & Tools

**Q51. What is the difference between `git merge` and `git rebase`?**
> `merge` creates a merge commit preserving history. `rebase` rewrites history by replaying commits on top of another branch — cleaner linear history.

**Q52. How do you resolve merge conflicts in Git?**
> Open conflicted files, choose which changes to keep (or combine), then `git add` and `git commit`.

**Q53. What is `.env` file and why is it important?**
> Stores environment variables (DB URI, JWT secret, API keys) outside the codebase. Never commit `.env` to version control; use `.env.example` instead.

**Q54. What is Docker and have you used it?**
> Docker packages an application and its dependencies into a container, ensuring consistent environments across dev/staging/prod. Common commands: `docker build`, `docker run`, `docker-compose up`.

---

## HR / Behavioural Round

**Q55. Tell me about yourself.**
> Structure: Current role → key skills → notable project → why ITC Infotech.

**Q56. Describe a challenging bug you solved.**
> Use STAR format: Situation → Task → Action → Result. Mention debugging tools (Chrome DevTools, Postman, console logs, MongoDB Compass).

**Q57. How do you manage deadlines and prioritise tasks?**
> Mention: breaking tasks into subtasks, daily standups, communicating blockers early, using tools like Jira/Trello.

**Q58. Where do you see yourself in 2–3 years?**
> Growing into a senior full-stack role, leading small feature teams, contributing to architecture decisions.

**Q59. Why ITC Infotech?**
> Research their tech stack, domain (enterprise digital transformation), and mention alignment with your career growth goals.

**Q60. Do you have any questions for us?**
> Good questions to ask:
> - What does a typical sprint look like for this team?
> - What tech stack is the team currently using?
> - What are opportunities for learning and growth here?

---

## 🔴 Deep Dive: High Priority Topics

### JS — Prototype Chain

**Q61. What is the prototype chain in JavaScript?**
> Every JS object has an internal `[[Prototype]]` link to another object. When you access a property, JS looks on the object itself, then walks up the chain until it finds it or reaches `null`.
```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function () {
  return `${this.name} makes a sound.`;
};

const dog = new Animal('Rex');
console.log(dog.speak()); // Rex makes a sound.
console.log(dog.__proto__ === Animal.prototype); // true
console.log(Animal.prototype.__proto__ === Object.prototype); // true
```
> Chain: `dog` → `Animal.prototype` → `Object.prototype` → `null`

**Q62. What is the difference between `__proto__` and `prototype`?**
> - `prototype` is a property on **constructor functions** used to set up inheritance.
> - `__proto__` is an internal link on **every object instance** pointing to its constructor's prototype.
```js
function Foo() {}
const f = new Foo();
f.__proto__ === Foo.prototype; // true
```

**Q63. How does `Object.create()` differ from using `new`?**
> `Object.create(proto)` creates a new object with `proto` as its `[[Prototype]]`, without calling a constructor.
```js
const animal = { speak() { return 'sound'; } };
const dog = Object.create(animal);
dog.speak(); // 'sound' — inherited via prototype chain
```

---

### JS — Event Loop (Deep)

**Q64. What is the order of execution in this code? (tricky output)**
```js
console.log('1');
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
console.log('4');
```
> **Output: 1, 4, 3, 2**
> - `1` and `4` are synchronous — run first.
> - Promise `.then` is a **microtask** — runs before macrotasks.
> - `setTimeout` is a **macrotask** — runs last.

**Q65. What are microtasks vs macrotasks?**
| | Microtasks | Macrotasks |
|---|---|---|
| Examples | `Promise.then`, `queueMicrotask`, `MutationObserver` | `setTimeout`, `setInterval`, `setImmediate`, I/O |
| Priority | Runs after current task, before any macrotask | Runs after microtask queue is empty |

**Q66. What are Node.js Streams and why use them?**
> Streams process data piece-by-piece (chunks) instead of loading everything into memory. Types: Readable, Writable, Duplex, Transform.
```js
const fs = require('fs');
const readable = fs.createReadStream('large-file.txt');
const writable = fs.createWriteStream('output.txt');
readable.pipe(writable); // Memory-efficient file copy
```
> Use case: reading large files, video streaming, HTTP request/response bodies.

---

### React — Performance Optimization

**Q67. What is `React.memo` and when should you use it?**
> `React.memo` is a HOC that prevents a functional component from re-rendering if its **props haven't changed** (shallow comparison).
```jsx
const UserCard = React.memo(({ name, age }) => {
  console.log('rendered');
  return <div>{name} - {age}</div>;
});
// Only re-renders if `name` or `age` changes
```
> ⚠️ Don't overuse — the comparison itself has a cost. Use when re-renders are expensive.

**Q68. Demonstrate `useMemo` preventing expensive recalculation.**
```jsx
import { useState, useMemo } from 'react';

function App() {
  const [count, setCount] = useState(0);
  const [input, setInput] = useState('');

  // Only recalculates when `count` changes, not on every `input` keystroke
  const expensiveValue = useMemo(() => {
    console.log('computing...');
    return count * 1000;
  }, [count]);

  return (
    <>
      <input value={input} onChange={e => setInput(e.target.value)} />
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <p>Result: {expensiveValue}</p>
    </>
  );
}
```

**Q69. Demonstrate `useCallback` preventing child re-renders.**
```jsx
import { useState, useCallback } from 'react';

const Button = React.memo(({ onClick, label }) => {
  console.log('Button rendered');
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // Without useCallback, a new function is created on every render,
  // causing Button to re-render even on `text` change
  const handleClick = useCallback(() => {
    setCount(c => c + 1);
  }, []); // stable reference

  return (
    <>
      <input value={text} onChange={e => setText(e.target.value)} />
      <Button onClick={handleClick} label={`Count: ${count}`} />
    </>
  );
}
```

**Q70. What is code splitting and how does `React.lazy` + `Suspense` work?**
```jsx
import React, { Suspense, lazy } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```
> Each page is split into a separate JS chunk — loaded only when the route is visited.

---

### React — Context API

**Q71. Implement a theme toggle using Context API.**
```jsx
// ThemeContext.js
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext();

export function ThemeProvider({ children }) {
  const [dark, setDark] = useState(false);
  return (
    <ThemeContext.Provider value={{ dark, toggle: () => setDark(d => !d) }}>
      {children}
    </ThemeContext.Provider>
  );
}

export const useTheme = () => useContext(ThemeContext);

// App.jsx
function Navbar() {
  const { dark, toggle } = useTheme();
  return (
    <nav style={{ background: dark ? '#111' : '#fff' }}>
      <button onClick={toggle}>Toggle Theme</button>
    </nav>
  );
}

function App() {
  return (
    <ThemeProvider>
      <Navbar />
    </ThemeProvider>
  );
}
```

---

### Redux — Thunk Middleware

**Q72. What is Redux Thunk and why is it needed?**
> Redux actions must be plain objects. Thunk middleware allows action creators to return **functions** instead, enabling async logic (API calls) inside actions.
```js
// Without Thunk — plain action (sync)
const setUser = (user) => ({ type: 'SET_USER', payload: user });

// With Thunk — async action creator
const fetchUser = (id) => async (dispatch) => {
  dispatch({ type: 'FETCH_START' });
  try {
    const res = await fetch(`/api/users/${id}`);
    const data = await res.json();
    dispatch({ type: 'SET_USER', payload: data });
  } catch (err) {
    dispatch({ type: 'FETCH_ERROR', payload: err.message });
  }
};

// In component
dispatch(fetchUser(123));
```

**Q73. Show a complete Redux Toolkit slice example.**
```js
// userSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

export const fetchUser = createAsyncThunk('user/fetch', async (id) => {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
});

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, loading: false, error: null },
  reducers: {
    clearUser: (state) => { state.data = null; }
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => { state.loading = true; })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false;
        state.data = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  }
});

export const { clearUser } = userSlice.actions;
export default userSlice.reducer;
```

---

### MongoDB — Transactions

**Q74. What are MongoDB transactions and when do you use them?**
> Transactions ensure multiple operations either **all succeed or all fail** (ACID). Required when updating multiple documents that must be consistent (e.g. transfer funds).
```js
const session = await mongoose.startSession();
session.startTransaction();
try {
  await Account.findByIdAndUpdate(senderId,
    { $inc: { balance: -amount } }, { session });
  await Account.findByIdAndUpdate(receiverId,
    { $inc: { balance: +amount } }, { session });
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```
> ⚠️ Transactions require a **replica set** (not standalone MongoDB).

**Q75. What is Mongoose `populate()` and how does it work?**
```js
// Models
const PostSchema = new mongoose.Schema({
  title: String,
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' }
});

// Query with populate
const post = await Post.findById(id).populate('author', 'name email');
// `post.author` is now a full User object, not just an ObjectId
```
> `populate()` makes a second query behind the scenes to fetch the referenced documents. For complex joins, prefer `$lookup` in aggregation.

---

## 🟠 Deep Dive: Medium Priority Topics

### Security

**Q76. What is XSS (Cross-Site Scripting) and how do you prevent it?**
> XSS is when an attacker injects malicious scripts into a web page that other users see.
> **Prevention:**
> - Sanitize/escape user input before rendering (use libraries like `DOMPurify` on client)
> - Use `helmet` in Express to set security headers
> - React escapes JSX by default — avoid `dangerouslySetInnerHTML`
> - Set `Content-Security-Policy` headers

**Q77. What is CSRF and how do you prevent it?**
> CSRF (Cross-Site Request Forgery) tricks a user's browser into making unwanted requests to your API using their existing session/cookies.
> **Prevention:**
> - Use `SameSite=Strict` or `SameSite=Lax` on cookies
> - Use CSRF tokens for state-changing requests
> - Use `Authorization: Bearer <JWT>` header-based auth (CSRF doesn't affect headers, only cookies)

**Q78. What is NoSQL Injection in MongoDB and how do you prevent it?**
> Attackers send query operators in user input to manipulate queries.
```js
// Vulnerable — user sends: { "username": { "$gt": "" } }
User.findOne({ username: req.body.username });

// Safe — sanitize or use mongoose schema types which enforce type coercion
// Use express-mongo-sanitize middleware
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize());
```

**Q79. How do you store passwords securely in Node.js?**
```js
const bcrypt = require('bcrypt');

// Hashing on register
const hash = await bcrypt.hash(plainPassword, 10); // 10 = salt rounds

// Comparing on login
const isMatch = await bcrypt.compare(plainPassword, hash);
```
> Never store plain text passwords. Always hash with bcrypt (not MD5/SHA1).

**Q80. What does the `helmet` package do in Express?**
> `helmet` sets various HTTP security headers to protect against common attacks:
```js
const helmet = require('helmet');
app.use(helmet());
// Sets: X-Content-Type-Options, X-Frame-Options, X-XSS-Protection,
//       Content-Security-Policy, Strict-Transport-Security, etc.
```

---

### Git — Advanced

**Q81. What is `git cherry-pick` and when would you use it?**
> Applies a specific commit from one branch onto another without merging the whole branch.
```bash
git checkout main
git cherry-pick abc1234   # apply only this commit from feature branch
```
> Use case: a hotfix was committed to a feature branch — cherry-pick it to `main` without merging unfinished features.

**Q82. Explain Gitflow branching strategy.**
> Gitflow defines these branches:
> - `main` — production-ready code only
> - `develop` — integration branch for features
> - `feature/xyz` — new features, branched from `develop`
> - `release/x.x` — pre-release QA, branched from `develop`
> - `hotfix/xyz` — urgent prod fixes, branched from `main`

```bash
# Feature workflow
git checkout -b feature/login develop
# ... work ...
git checkout develop
git merge --no-ff feature/login
git branch -d feature/login
```

---

## 🟢 Deep Dive: Good to Know Topics

### TypeScript Basics

**Q83. What is TypeScript and why use it over plain JS?**
> TypeScript is a statically typed superset of JavaScript that compiles to plain JS. Benefits: catches type errors at compile time, better IDE autocomplete, self-documenting code, safer refactoring.

**Q84. What is the difference between `interface` and `type` in TypeScript?**
```ts
// Interface — extendable, used for objects/classes
interface User {
  id: number;
  name: string;
}
interface Admin extends User {
  role: string;
}

// Type — more flexible, can do unions/intersections
type ID = string | number;
type AdminUser = User & { role: string };
```
> Use `interface` for object shapes, `type` for unions, intersections, and primitives.

**Q85. What are generics in TypeScript?**
> Generics allow writing reusable, type-safe functions/components.
```ts
function identity<T>(value: T): T {
  return value;
}
identity<string>('hello'); // typed as string
identity<number>(42);      // typed as number

// Generic React component
interface ListProps<T> {
  items: T[];
  render: (item: T) => React.ReactNode;
}
function List<T>({ items, render }: ListProps<T>) {
  return <ul>{items.map(render)}</ul>;
}
```

**Q86. What are common TypeScript utility types?**
```ts
interface User { id: number; name: string; email: string; }

Partial<User>       // All fields optional
Required<User>      // All fields required
Readonly<User>      // All fields read-only
Pick<User, 'id' | 'name'>  // Only id and name
Omit<User, 'email'>         // Everything except email
Record<string, number>      // { [key: string]: number }
```

---

### WebSockets / Socket.IO

**Q87. What is the difference between HTTP and WebSockets?**
| | HTTP | WebSocket |
|---|---|---|
| Connection | Request-response, closes after | Persistent, full-duplex |
| Direction | Client initiates | Both can send anytime |
| Use case | REST APIs, fetching data | Chat, live notifications, games |

**Q88. Implement a basic Socket.IO chat server + client.**
```js
// Server (Node.js)
const { Server } = require('socket.io');
const io = new Server(3001, { cors: { origin: '*' } });

io.on('connection', (socket) => {
  console.log('user connected:', socket.id);

  socket.on('send_message', (data) => {
    io.emit('receive_message', data); // broadcast to all
  });

  socket.on('disconnect', () => console.log('user left'));
});
```
```jsx
// Client (React)
import { useEffect, useState } from 'react';
import { io } from 'socket.io-client';

const socket = io('http://localhost:3001');

function Chat() {
  const [messages, setMessages] = useState([]);
  const [text, setText] = useState('');

  useEffect(() => {
    socket.on('receive_message', (msg) => {
      setMessages(prev => [...prev, msg]);
    });
    return () => socket.off('receive_message');
  }, []);

  const send = () => {
    socket.emit('send_message', { text, time: new Date() });
    setText('');
  };

  return (
    <div>
      {messages.map((m, i) => <p key={i}>{m.text}</p>)}
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={send}>Send</button>
    </div>
  );
}
```

---

### Testing

**Q89. Write a Jest unit test for a utility function.**
```js
// utils/math.js
function add(a, b) { return a + b; }
function divide(a, b) {
  if (b === 0) throw new Error('Cannot divide by zero');
  return a / b;
}
module.exports = { add, divide };

// utils/math.test.js
const { add, divide } = require('./math');

describe('add', () => {
  test('adds two numbers', () => expect(add(2, 3)).toBe(5));
  test('handles negatives', () => expect(add(-1, 1)).toBe(0));
});

describe('divide', () => {
  test('divides correctly', () => expect(divide(10, 2)).toBe(5));
  test('throws on divide by zero', () => {
    expect(() => divide(10, 0)).toThrow('Cannot divide by zero');
  });
});
```

**Q90. Write a React Testing Library test for a component.**
```jsx
// LoginForm.jsx
function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  return (
    <form onSubmit={() => onSubmit(email)}>
      <input
        placeholder="Email"
        value={email}
        onChange={e => setEmail(e.target.value)}
      />
      <button type="submit">Login</button>
    </form>
  );
}

// LoginForm.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import LoginForm from './LoginForm';

test('calls onSubmit with email value', () => {
  const mockSubmit = jest.fn();
  render(<LoginForm onSubmit={mockSubmit} />);

  fireEvent.change(screen.getByPlaceholderText('Email'), {
    target: { value: 'test@example.com' }
  });
  fireEvent.click(screen.getByText('Login'));

  expect(mockSubmit).toHaveBeenCalledWith('test@example.com');
});
```

---

### Docker Fundamentals

**Q91. Write a `Dockerfile` for a Node.js/Express app.**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Install dependencies first (layer caching optimization)
COPY package*.json ./
RUN npm ci --only=production

# Copy source
COPY . .

EXPOSE 5000
CMD ["node", "src/app.js"]
```

**Q92. Write a `docker-compose.yml` for a MERN stack.**
```yaml
version: '3.8'
services:
  mongo:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  server:
    build: ./server
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=mongodb://mongo:27017/myapp
      - JWT_SECRET=supersecret
    depends_on:
      - mongo

  client:
    build: ./client
    ports:
      - "3000:3000"
    depends_on:
      - server

volumes:
  mongo-data:
```

---

### System Design

**Q93. Design a real-time notification system.**
> **Requirements**: Users get notified when someone likes their post.
>
> **Architecture:**
> - **Frontend**: React + Socket.IO client, shows toast/bell icon
> - **Backend**: Express + Socket.IO server
> - **Flow**:
>   1. User A likes User B's post → POST `/api/posts/:id/like`
>   2. Server saves like to MongoDB
>   3. Server emits `notification` event to User B's socket room: `socket.to(userBSocketId).emit('notification', { msg: 'User A liked your post' })`
>   4. Client receives event and shows toast
> - **Persistence**: Save notifications in MongoDB `notifications` collection
> - **Scale**: Use Redis Pub/Sub + multiple Node instances behind a load balancer for horizontal scaling

**Q94. How would you implement pagination in a MERN app?**
```js
// Backend — cursor/offset based
router.get('/posts', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  const [posts, total] = await Promise.all([
    Post.find().skip(skip).limit(limit).sort({ createdAt: -1 }),
    Post.countDocuments()
  ]);

  res.json({
    posts,
    currentPage: page,
    totalPages: Math.ceil(total / limit),
    totalItems: total
  });
});
```
```jsx
// Frontend
const [page, setPage] = useState(1);
const { data } = useFetch(`/api/posts?page=${page}&limit=10`);

<button onClick={() => setPage(p => p - 1)} disabled={page === 1}>Prev</button>
<button onClick={() => setPage(p => p + 1)} disabled={page === data.totalPages}>Next</button>
```

---

## 📚 Topics to Revise

### 🟡 High Priority
- [ ] JavaScript: Closures, Promises, async/await, Event Loop, Prototype chain
- [ ] React: Hooks (useState, useEffect, useMemo, useCallback, useRef), Component lifecycle
- [ ] Redux: Actions, Reducers, Store, Thunk middleware
- [ ] MongoDB: Aggregation pipeline, Indexing, Mongoose population
- [ ] REST API Design: Status codes, CRUD, JWT auth flow
- [ ] Node.js: Event loop, Streams, Middleware pattern
- [ ] Express.js: Routing, Error handling, Authentication middleware

### 🟠 Medium Priority
- [ ] React: Performance optimization (memo, lazy, Suspense), Context API
- [ ] MongoDB: Schema design (embed vs reference), Transactions
- [ ] Node.js: Clustering, Worker threads, `process.nextTick` vs `setImmediate`
- [ ] Security: XSS, CSRF, SQL injection equivalents in Mongo, rate limiting, CORS
- [ ] Git: Rebase vs Merge, cherry-pick, branching strategy (Gitflow)

### 🟢 Good to Know
- [ ] TypeScript basics (types, interfaces, generics)
- [ ] Docker fundamentals
- [ ] CI/CD concepts
- [ ] WebSockets / Socket.IO basics
- [ ] System design basics (URL shortener, chat app)
- [ ] Testing: Jest unit tests, React Testing Library

---

> 💡 **Tip**: For ITC Infotech, expect a mix of theoretical questions and practical coding rounds. Be ready to write small code snippets on paper or a shared editor. Practice explaining your projects clearly — they often ask "walk me through your project architecture."

---
*Prepared for ITC Infotech | MERN Fullstack | 2 YOE | Good luck! 🎯*
