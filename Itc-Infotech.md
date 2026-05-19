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
