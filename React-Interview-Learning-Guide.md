# The Complete React Interview & Learning Guide

*A roadmap-style guide from zero to senior React engineer. Covers concepts, internals, interview questions with deep answers, system design, and resources.*

---

## Table of Contents

1. [How to Use This Guide](#1-how-to-use-this-guide)
2. [The Full React Skill Tree](#2-the-full-react-skill-tree)
3. [Phase 1 — JavaScript Prerequisites](#3-phase-1--javascript-prerequisites)
4. [Phase 2 — React Fundamentals](#4-phase-2--react-fundamentals)
5. [Phase 3 — Hooks Deep Dive](#5-phase-3--hooks-deep-dive)
6. [Phase 4 — Component Patterns](#6-phase-4--component-patterns)
7. [Phase 5 — State Management](#7-phase-5--state-management)
8. [Phase 6 — Routing & Data Fetching](#8-phase-6--routing--data-fetching)
9. [Phase 7 — Forms](#9-phase-7--forms)
10. [Phase 8 — Performance](#10-phase-8--performance)
11. [Phase 9 — Testing](#11-phase-9--testing)
12. [Phase 10 — TypeScript with React](#12-phase-10--typescript-with-react)
13. [Phase 11 — Next.js & SSR](#13-phase-11--nextjs--ssr)
14. [Phase 12 — React Internals](#14-phase-12--react-internals)
15. [Phase 13 — Modern React (Server Components, Suspense, Concurrent)](#15-phase-13--modern-react)
16. [Phase 14 — Frontend System Design](#16-phase-14--frontend-system-design)
17. [Interview Questions — 150+ with Deep Answers](#17-interview-questions)
18. [Coding Challenges to Practice](#18-coding-challenges-to-practice)
19. [Behavioral & Project Discussion Prep](#19-behavioral--project-prep)
20. [Resources Library](#20-resources-library)
21. [Study Plan](#21-study-plan)
22. [Common Mistakes](#22-common-mistakes)

---

## 1. How to Use This Guide

There are two modes:

**Learning mode.** Go phase by phase. For each topic, learn the *why* before the *what*. Build something using the concept before moving on. Resist the urge to skip to advanced sections.

**Interview prep mode.** Skim phases 1-7 to confirm gaps, then drill phases 8, 12, 14, and section 17 (interview questions). Practice section 18 daily for 2-4 weeks.

Either way, the test of understanding is: can you teach it back to someone in 60 seconds with a concrete example?

---

## 2. The Full React Skill Tree

```
React Engineer
│
├── 1. JavaScript Prerequisites
│   ├── Closures, scope, hoisting
│   ├── Promises, async/await, event loop
│   ├── Prototypes, this binding
│   ├── Destructuring, spread, rest
│   ├── ES Modules
│   └── Functional concepts (map, filter, reduce, immutability)
│
├── 2. React Fundamentals
│   ├── JSX and virtual DOM
│   ├── Components (function vs class)
│   ├── Props and prop drilling
│   ├── State (useState)
│   ├── Events (synthetic events)
│   ├── Conditional rendering
│   ├── Lists and keys
│   └── Component composition
│
├── 3. Hooks Deep Dive
│   ├── useState, useEffect, useLayoutEffect
│   ├── useRef, useImperativeHandle
│   ├── useMemo, useCallback
│   ├── useContext, useReducer
│   ├── useTransition, useDeferredValue, useId
│   ├── useSyncExternalStore, useInsertionEffect
│   ├── Rules of hooks
│   └── Custom hooks
│
├── 4. Component Patterns
│   ├── Container/Presentational
│   ├── Compound components
│   ├── Render props
│   ├── Higher-order components (HOC)
│   ├── Provider pattern
│   ├── Controlled vs uncontrolled
│   └── Slots / children-as-function
│
├── 5. State Management
│   ├── Local state best practices
│   ├── Context API and its pitfalls
│   ├── Redux Toolkit
│   ├── Zustand
│   ├── Jotai / Recoil
│   ├── TanStack Query (server state)
│   ├── SWR
│   └── When to use which
│
├── 6. Routing & Data Fetching
│   ├── React Router (v6+)
│   ├── TanStack Router
│   ├── Loaders and actions
│   ├── REST integration
│   ├── GraphQL (Apollo, urql, Relay)
│   └── Streaming and Suspense for data
│
├── 7. Forms
│   ├── Controlled vs uncontrolled
│   ├── React Hook Form
│   ├── Formik
│   ├── Zod / Yup validation
│   └── File uploads
│
├── 8. Performance
│   ├── React DevTools Profiler
│   ├── memo, useMemo, useCallback
│   ├── Virtualization (react-window, react-virtual)
│   ├── Code splitting and lazy loading
│   ├── Bundle analysis
│   ├── Suspense and concurrent rendering
│   ├── Image optimization
│   └── Web Vitals (LCP, INP, CLS)
│
├── 9. Testing
│   ├── React Testing Library
│   ├── Jest, Vitest
│   ├── Playwright, Cypress (e2e)
│   ├── MSW for API mocking
│   └── Visual regression (Chromatic, Percy)
│
├── 10. TypeScript with React
│   ├── Typing props and state
│   ├── Generic components
│   ├── Polymorphic components
│   ├── Discriminated unions
│   └── Utility types
│
├── 11. Next.js & SSR
│   ├── App Router vs Pages Router
│   ├── Server Components
│   ├── Server Actions
│   ├── Streaming SSR
│   ├── Caching layers
│   ├── Edge runtime
│   └── Vercel deployment
│
├── 12. React Internals
│   ├── Reconciliation and Fiber
│   ├── Render phase vs commit phase
│   ├── Hooks implementation
│   ├── Batching
│   ├── Lanes and priority
│   └── Concurrent features
│
├── 13. Modern React
│   ├── React 18+ features
│   ├── React 19 features
│   ├── Server Components
│   ├── Suspense everywhere
│   ├── Transitions
│   └── Actions and useActionState
│
└── 14. Frontend System Design
    ├── Component library design
    ├── Design system architecture
    ├── Micro-frontends
    ├── Real-time updates (WebSocket, SSE)
    ├── Offline-first
    ├── Accessibility (a11y)
    └── Internationalization (i18n)
```

---

## 3. Phase 1 — JavaScript Prerequisites

You cannot be senior at React without being solid at JavaScript. Most "React bugs" are JavaScript misunderstandings.

### 3.1 What to Master

**Scope and closures.** A closure is a function that remembers the variables from where it was created. Closures are *why* `useCallback` exists, why stale state happens in `useEffect`, why event handlers capture old values. If you do not understand closures, you will fight React for years.

**Event loop and async.** The browser has one main thread. The event loop processes tasks one at a time. Promises (microtasks) run before timers (macrotasks). When you call `setState` and read state on the next line, the state is *not yet updated* — the re-render is queued.

**Prototypes and `this`.** Less critical for hooks-era React, but still appears in class components and in interview "explain this" questions.

**Destructuring, spread, rest, optional chaining, nullish coalescing.** Daily-use syntax.

**ES Modules.** `import`, `export`, dynamic `import()`, named vs default exports. Affects code splitting and tree shaking.

**Immutability.** Arrays and objects in React state should be replaced, not mutated. `arr.push()` does not trigger re-render; `setArr([...arr, item])` does.

**Equality.** `===` vs `==`. Object reference equality (`{} !== {}`). Why two component renders create different objects and that breaks `useEffect` deps.

### 3.2 Test Yourself

Without looking anything up, can you:
- Explain a closure with a small example?
- Predict the order of console logs in a snippet mixing `setTimeout`, `Promise.resolve()`, and synchronous code?
- Explain why `[1,2,3] === [1,2,3]` is `false`?
- Implement a debounce function from scratch?
- Implement a throttle function from scratch?
- Write `Promise.all` from scratch using promises?

If yes, move on. If no, spend 1-2 weeks shoring up.

### 3.3 Resources
- **"You Don't Know JS Yet"** by Kyle Simpson (free on GitHub)
- **JavaScript.info** — the modern JS reference; read end to end
- **ThePrimeagen's "FullStack" content** on YouTube and Frontend Masters
- **MDN Web Docs** — for any specific API

---

## 4. Phase 2 — React Fundamentals

### 4.1 What React Actually Is

React is a library for building user interfaces by describing them as a tree of components. You write what the UI *should look like* given some state, and React figures out how to update the DOM to match. The mental shift from imperative DOM manipulation ("find the div, change its text") to declarative ("the UI is a function of state") is the core skill.

### 4.2 JSX and the Virtual DOM

**JSX** is HTML-like syntax that compiles to `React.createElement(...)` calls. Those calls return plain JavaScript objects describing what the UI should be.

**Virtual DOM** is the in-memory tree of those objects. React compares the new VDOM with the previous VDOM (reconciliation) and applies only the necessary changes to the real DOM.

The win: you describe what; React handles the how. The catch: it is not magic — bad code patterns force React to re-render too much.

### 4.3 Components

**Function components** are the default. A component is a function that takes props and returns JSX.

**Class components** exist in older codebases. You should be able to read them but not write new ones.

**Props** are inputs. They flow down one direction (parent → child). Components do not modify their own props.

**State** is internal data that, when changed, triggers a re-render of the component.

### 4.4 The Render Cycle

1. State or props change.
2. React calls the component function again ("re-render").
3. The component returns new JSX.
4. React compares (diffs) the new JSX with the previous.
5. React updates only the changed parts of the real DOM ("commit").

Understanding this cycle is the foundation of every advanced topic that follows.

### 4.5 Keys

Lists need stable, unique keys. **Never use array index as key** when the list reorders or items can be inserted/removed — it causes bugs where state from one item shows up on another.

### 4.6 Events

React wraps native events in **SyntheticEvent** for cross-browser consistency. `onClick`, `onChange`, etc. attach at the React root (delegation), not directly on DOM nodes.

### 4.7 Conditional Rendering

Three idioms:
- Ternary: `condition ? <A/> : <B/>`
- Short-circuit: `condition && <A/>`
- Variable: assign JSX to a variable, return it.

Watch out: `0 && <Component/>` renders `0`. Use explicit booleans.

---

## 5. Phase 3 — Hooks Deep Dive

Hooks are the operating system of modern React. Master them.

### 5.1 The Rules

- Call hooks only at the **top level** of a component or another hook.
- Never call hooks inside loops, conditions, or nested functions.

Why: React tracks hooks by **call order**. Changing the order between renders breaks everything.

### 5.2 useState

Stores a value across renders. Returns `[value, setValue]`.

Key behaviors:
- State updates are **asynchronous and batched**.
- Calling `setValue(newValue)` does not immediately update `value` in the current render — it schedules a re-render with the new value.
- If new state depends on previous state, use the **functional form**: `setValue(prev => prev + 1)`. This avoids stale-closure bugs.
- React uses `Object.is` to compare. If you set state to the same value (or same reference), React skips the re-render.

### 5.3 useEffect

Runs **after** render commits. Used for side effects — data fetching, subscriptions, manual DOM mutations, integrations with non-React code.

Key behaviors:
- Runs after every render *by default*.
- The **dependency array** controls when it runs: `[]` runs once after mount; `[a, b]` runs when `a` or `b` changes.
- The **cleanup function** runs before the next effect and on unmount. Critical for subscriptions, timers, abort controllers.
- In Strict Mode (development), effects run twice on mount to catch missing cleanup.

Common pitfalls:
- **Stale closures.** An effect captures variables at the time it ran. If you forget a dependency, the effect sees old values.
- **Infinite loops.** An effect that updates state on every render with that state in deps.
- **Race conditions.** Fast user input triggers multiple fetches; the slow first one resolves last and overwrites correct data. Fix with AbortController.

### 5.4 useLayoutEffect

Same API as `useEffect` but runs *synchronously* after DOM mutations and *before* the browser paints. Use only for measuring DOM and triggering re-renders before paint (avoid layout flicker). Otherwise prefer `useEffect`.

### 5.5 useRef

Returns a mutable object `{ current: value }` that persists across renders. Two uses:

1. **Access DOM nodes:** attach with `ref={myRef}` on a JSX element.
2. **Store mutable values that don't trigger re-renders:** timers, last-seen values, instance flags.

Updating `ref.current` does NOT trigger a re-render. That's the point.

### 5.6 useMemo

Caches the result of a computation between renders.

```
const expensive = useMemo(() => heavyCalc(a, b), [a, b]);
```

When to use:
- The calculation is genuinely expensive (measure first).
- The value is used in another hook's dependency array (preserves reference identity).

When NOT to use: trivial computations. The bookkeeping cost can exceed the savings.

### 5.7 useCallback

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`. Returns the *same function reference* across renders as long as deps don't change.

Useful when:
- Passing the callback to a memoized child (otherwise the child re-renders because the prop "changes").
- The callback is in another hook's dependency array.

Otherwise, do not wrap every callback.

### 5.8 useContext

Reads a value from a Context provider higher in the tree. Used for global-ish data (theme, auth user, locale).

Pitfall: every consumer re-renders when the context value changes, even if they only care about part of it. Mitigations:
- Split the context into smaller contexts.
- Use a state library (Zustand) for fine-grained subscriptions.
- Memoize the provider value.

### 5.9 useReducer

Like `useState` but for complex state with multiple actions. Useful when:
- State has many sub-fields that change together.
- Update logic is non-trivial.
- You want a predictable, testable transition function.

The pattern is the same as Redux at component scope.

### 5.10 useTransition and useDeferredValue (React 18+)

`useTransition` marks an update as non-urgent. React keeps the UI responsive by interrupting the transition if a more urgent update (like typing) arrives.

```
const [isPending, startTransition] = useTransition();
startTransition(() => {
  setExpensiveState(newValue);
});
```

`useDeferredValue` is similar but applied to a value rather than an update.

Both are concurrent-rendering features that fix the "typing in search input feels laggy because results are heavy" class of bugs.

### 5.11 useId

Generates a stable unique ID. Useful for accessibility (linking labels to inputs) and SSR (server and client must produce the same ID).

### 5.12 useSyncExternalStore

Subscribes a component to an external store (Redux, Zustand, browser APIs). Replaces the old `useEffect` + `forceUpdate` pattern with concurrent-mode-safe behavior.

You will rarely call this directly; state libraries use it internally.

### 5.13 useInsertionEffect

Runs before all DOM mutations. Designed for CSS-in-JS libraries to inject styles. Almost never use it in app code.

### 5.14 Custom Hooks

A custom hook is a function whose name starts with `use` and which uses other hooks. Used to extract reusable stateful logic.

Good examples:
- `useDebounce(value, ms)` — returns a debounced value.
- `useLocalStorage(key, initial)` — syncs state with localStorage.
- `useFetch(url)` — wraps fetch with loading/error states.
- `useMediaQuery(query)` — reactive matchMedia.

Custom hooks share *logic*, not *state*. Two components using `useCounter()` get two independent counters.

---

## 6. Phase 4 — Component Patterns

### 6.1 Controlled vs Uncontrolled

A controlled component has its value tied to React state. An uncontrolled component manages its own value in the DOM (read via ref).

- **Controlled forms** are the standard. Better validation, easier control.
- **Uncontrolled forms** can be faster for huge forms but harder to validate.

### 6.2 Compound Components

A parent component exposes children that work together (Tabs, Accordion, Select). The parent holds shared state via Context; children consume it.

Example shape:
```
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Trigger value="a">Tab A</Tabs.Trigger>
    <Tabs.Trigger value="b">Tab B</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="a">...</Tabs.Content>
  <Tabs.Content value="b">...</Tabs.Content>
</Tabs>
```

Used in Radix UI, shadcn/ui. Mastering this pattern unlocks writing reusable component libraries.

### 6.3 Render Props

A component takes a function as a child and calls it with internal state.

```
<Mouse>
  {(position) => <Cursor at={position} />}
</Mouse>
```

Mostly replaced by hooks for sharing logic, but still useful for *rendering control*.

### 6.4 Higher-Order Components (HOC)

A function that takes a component and returns a new component with extra behavior. Older pattern; hooks replace most use cases.

### 6.5 Provider Pattern

Wrap part of your tree in a Context provider to share data. Common for theme, auth, i18n.

### 6.6 Slots and Children-as-Function

A flexible component accepts named children (`children`, `header`, `footer`) or a function child. Used in layout primitives.

---

## 7. Phase 5 — State Management

### 7.1 Categorize Your State

Before choosing a library, classify each piece of state:

| Category | Where it lives | Tool |
|---|---|---|
| Local UI state | One component | `useState` |
| Shared UI state | A few related components | Lift up; Context |
| Global UI state | Many unrelated components | Zustand, Redux Toolkit |
| Server state | Fetched from API | TanStack Query, SWR |
| URL state | In the URL | React Router, search params |
| Form state | An input form | React Hook Form |

**The biggest mistake** is putting server state into Redux. Server state has caching, revalidation, optimistic updates, and request deduplication — TanStack Query handles all of that; Redux does not.

### 7.2 Context API

Built-in. Good for low-frequency global values (theme, auth user).

Pitfall: every consumer re-renders on every value change. Don't put high-frequency data in Context.

### 7.3 Redux Toolkit

The modern, opinionated Redux. Solves the boilerplate complaints.

Use when:
- Complex client state with many actions.
- You need time-travel debugging.
- Team is already familiar.

Otherwise, Zustand is often a better choice today.

### 7.4 Zustand

Minimal, fast, no boilerplate. Becoming the modern default for global state.

- Store is just a hook.
- Selectors avoid unnecessary re-renders.
- Works outside React (good for stores accessed by non-component code).

### 7.5 Jotai and Recoil

Atom-based state — each piece of state is an "atom." Compose derived atoms from base atoms. Great for highly granular state.

### 7.6 TanStack Query (formerly React Query)

The gold standard for server state. Handles:
- Caching
- Background refetching
- Stale-while-revalidate
- Optimistic updates
- Pagination and infinite queries
- Request deduplication
- Devtools

If you take one tool from this guide, take this one.

### 7.7 SWR

Vercel's alternative. Simpler API, fewer features than TanStack Query. Often paired with Next.js apps.

---

## 8. Phase 6 — Routing & Data Fetching

### 8.1 React Router

The dominant client-side router. Version 6+ uses a declarative tree:

- `<Routes>` and `<Route>`
- Nested routes with `<Outlet/>`
- `useNavigate`, `useParams`, `useSearchParams`, `useLocation`
- Loaders and actions (newer pattern for data + mutations)
- Protected routes via wrapping components

### 8.2 TanStack Router

Modern alternative. Type-safe routes, file-based optional, integrated with TanStack Query. Gaining adoption.

### 8.3 Data Fetching Patterns

- **Fetch on render** — useEffect-based. Easiest, but causes waterfalls.
- **Fetch then render** — fetch at the route level (loaders), render when ready. Faster.
- **Render as you fetch** — Suspense for data. Stream UI as data arrives.

Modern apps use TanStack Query for fetch-on-render with caching, or route loaders for fetch-then-render.

### 8.4 GraphQL

Less universal than REST but still common. Major libraries:
- **Apollo Client** — most popular; heavyweight; powerful cache.
- **urql** — lighter alternative.
- **Relay** — Facebook's; strict but very powerful at scale.

---

## 9. Phase 7 — Forms

### 9.1 Controlled vs Uncontrolled Recap

Controlled = React state holds value. Uncontrolled = DOM holds value.

### 9.2 React Hook Form

The modern default. Uncontrolled by default for performance, with controlled escape hatches.

Strengths:
- Minimal re-renders.
- Built-in validation hooks.
- Easy integration with Zod / Yup schemas.

### 9.3 Formik

Older but still common. More controlled-by-default; more re-renders.

### 9.4 Schema Validation

- **Zod** — TypeScript-first, runtime + static types from one schema. Now the default.
- **Yup** — older, still common.
- **Valibot** — newer, smaller bundle.

### 9.5 File Uploads

Common interview topic.
- Show progress with `XMLHttpRequest` or fetch with streams.
- Use presigned URLs for direct-to-S3 uploads (avoid server bandwidth).
- Validate type and size client-side, re-validate server-side.

---

## 10. Phase 8 — Performance

This is where senior interviews focus.

### 10.1 The Mental Model

Slow React apps are caused by:
1. **Too many re-renders** — components updating when they don't need to.
2. **Expensive renders** — a component doing too much work each render.
3. **Large bundles** — too much JavaScript downloaded and parsed.
4. **Wrong-time renders** — heavy work blocking user input.

Fix in order.

### 10.2 Profiling

Use **React DevTools Profiler** to find slow components.
- "Flame graph" shows render duration.
- "Ranked" shows which components took longest.
- "Why did this render?" shows the cause (props change, state change, parent re-render).

### 10.3 React.memo

Wraps a component to skip re-rendering when props are shallow-equal.

```
const Item = React.memo(function Item({ name }) {
  return <li>{name}</li>;
});
```

Pitfalls:
- Pass functions via `useCallback` and objects via `useMemo`, or the memo is defeated by new references each render.
- Don't `memo` everything — bookkeeping has cost.

### 10.4 useMemo and useCallback

Use sparingly. Defaults to NOT using them unless:
- The computation is measurably expensive.
- The value/function is used as a dep elsewhere.
- A child component is memoized.

### 10.5 Virtualization

For long lists (1000+ items), render only the visible rows. Tools:
- **react-window** — lightweight, classic.
- **@tanstack/react-virtual** — modern, flexible.

Without virtualization, 10,000 rows means 10,000 DOM nodes. With virtualization, ~20.

### 10.6 Code Splitting

`React.lazy` + `Suspense` to load components on demand:

```
const Heavy = React.lazy(() => import('./Heavy'));
```

Combine with route-level splitting so each page loads only what it needs.

### 10.7 Bundle Analysis

- **Vite:** rollup-plugin-visualizer
- **webpack:** webpack-bundle-analyzer
- **Next.js:** `@next/bundle-analyzer`

Look for: large libraries (date-fns vs dayjs, lodash vs lodash-es with tree shaking), duplicate dependencies, source maps in production.

### 10.8 Concurrent Rendering

React 18+ can pause and resume rendering. Used via `useTransition` and Suspense.

The mental model: not all updates are equally urgent. Typing in an input is urgent; recalculating a chart in the background is not. `startTransition` lets React keep the input responsive.

### 10.9 Image Optimization

- Use modern formats (WebP, AVIF).
- Lazy-load below the fold.
- Use `srcset` for responsive images.
- In Next.js, use `<Image>` component.

### 10.10 Web Vitals

Google's user-experience metrics:
- **LCP (Largest Contentful Paint)** — when the main content appears. Target: under 2.5s.
- **INP (Interaction to Next Paint)** — responsiveness. Target: under 200ms.
- **CLS (Cumulative Layout Shift)** — visual stability. Target: under 0.1.

Measure with Lighthouse, web-vitals library, or PageSpeed Insights.

### 10.11 React Compiler (React 19+)

Automatically memoizes components and values at build time. Reduces the need for manual `memo`/`useMemo`/`useCallback`. Adoption is starting; learn the patterns it enables.

---

## 11. Phase 9 — Testing

### 11.1 React Testing Library (RTL)

The standard for component tests. Philosophy: test the way users use the app.

Principles:
- Query by accessible role (`getByRole`), then by label, then by text. Never by class or test ID first.
- Test behavior, not implementation. Don't assert state shape; assert what the user sees.

### 11.2 Jest vs Vitest

- **Jest** — established, slower, dominant in older codebases.
- **Vitest** — fast, Vite-native, the modern default for new projects.

API is nearly identical.

### 11.3 MSW (Mock Service Worker)

Mock API requests at the network layer, not the function level. Tests look like real usage. Works in browsers too — great for Storybook and demos.

### 11.4 End-to-End Testing

- **Playwright** — Microsoft's; modern default. Fast, cross-browser.
- **Cypress** — older default. Strong DX but slower and limited cross-browser.

### 11.5 Visual Regression

- **Chromatic** (by Storybook team)
- **Percy**
- **Playwright** can also do basic snapshots.

Catches CSS regressions before users do.

---

## 12. Phase 10 — TypeScript with React

### 12.1 Why

Most senior React jobs are TypeScript jobs. Refusing to learn TS limits your career.

### 12.2 Essentials

- Type props with `interface` or `type`.
- Type children with `React.ReactNode`.
- Type events: `React.ChangeEvent<HTMLInputElement>`, `React.MouseEvent<HTMLButtonElement>`.
- Type refs: `useRef<HTMLDivElement>(null)`.
- Generic components: `function List<T>({ items }: { items: T[] }) { ... }`.

### 12.3 Discriminated Unions for Props

Express "if `variant` is `'link'`, `href` is required":

```
type Props =
  | { variant: 'link'; href: string }
  | { variant: 'button'; onClick: () => void };
```

TypeScript narrows based on `variant`. Cleaner than optional everything.

### 12.4 Utility Types You'll Use Daily

- `Partial<T>`, `Required<T>`, `Pick<T,K>`, `Omit<T,K>`
- `ReturnType<T>`, `Parameters<T>`
- `ComponentProps<typeof Button>` — borrow another component's prop types

### 12.5 Polymorphic Components

A component that can render as different elements (`as="button" | "a" | "div"`). Common in design systems. Genuinely difficult to type well; study Radix UI / Chakra source.

---

## 13. Phase 11 — Next.js & SSR

### 13.1 Why Next.js Dominates

Next.js is the default React framework for production apps:
- SSR and SSG for SEO and performance
- Built-in routing, image optimization, font optimization
- Server components and Server Actions
- Edge runtime
- Vercel deployment with zero config

### 13.2 App Router vs Pages Router

- **Pages Router** — older, file-based, `getServerSideProps` / `getStaticProps`.
- **App Router** — newer (Next 13+), based on React Server Components, layouts, streaming.

App Router is the future. New projects should use it. Some teams still maintain Pages Router apps.

### 13.3 Server Components

Render on the server, send HTML (and a small payload) to the client. Cannot use state or effects. Can fetch directly from databases and APIs.

Game changer: most components don't need to be interactive. Render them on the server, save JS bundle.

### 13.4 Client Components

Marked with `'use client'` at the top. Have state, effects, event handlers. Hydrated in the browser.

### 13.5 Server Actions

Server-side functions you call from the client like normal functions. Forms work with `action={myAction}`. Eliminates much of the typical API layer.

### 13.6 Caching Layers (App Router)

Next.js has four caching layers:
1. Request Memoization (per request)
2. Data Cache (persistent across deploys with revalidation)
3. Full Route Cache (static rendering)
4. Router Cache (client-side)

Understanding when each kicks in is non-trivial; expect interview questions on this.

### 13.7 Streaming

Render parts of the page as they're ready. Wrap in `<Suspense>` for instant initial paint while data loads.

---

## 14. Phase 12 — React Internals

This separates seniors from mid-level engineers in interviews.

### 14.1 The Two Phases

**Render phase.** React calls components, builds a tree of "fibers" (in-memory representation), and figures out what changed. Can be interrupted in concurrent mode.

**Commit phase.** React applies DOM changes synchronously. Effects run after commit.

### 14.2 Fiber Architecture

A **fiber** is a JavaScript object representing a unit of work. Each component instance has a fiber. React keeps two trees: the current tree (committed) and the work-in-progress tree (being built).

Why: enables interruption, prioritization, and concurrent rendering.

### 14.3 Reconciliation

The diffing algorithm:
- Different types at the same position → unmount old, mount new.
- Same type → reuse the DOM, update props.
- Lists → match by key.

This is why keys matter so much.

### 14.4 Hooks Under the Hood

Hooks are stored in a linked list on the fiber. Each call to `useState`/`useEffect` advances a pointer.

This is why hooks must be called in the same order every render — the pointer can't find the right slot otherwise.

### 14.5 Batching

React batches multiple state updates within an event into a single re-render. React 18 expanded this to include async contexts (timers, promises). You can force synchronous render with `flushSync` (rarely needed).

### 14.6 Lanes and Priority

React 18+ assigns "lanes" (bitmask priorities) to updates. Urgent updates (typing, clicks) get high priority lanes. Transitions get lower lanes that can be interrupted.

### 14.7 Strict Mode

In development, React intentionally double-invokes components and effects to surface bugs (missing cleanup, impure renders). Annoying first, lifesaver after.

---

## 15. Phase 13 — Modern React

### 15.1 React 18 Features

- **Automatic batching** in async contexts.
- **Concurrent rendering** (`useTransition`, `useDeferredValue`).
- **Suspense for data fetching** (in frameworks).
- **`useId`** for accessible IDs.
- **`useSyncExternalStore`** for external stores.

### 15.2 React 19 Features

- **React Compiler** — automatic memoization.
- **Actions** — async functions in transitions; built-in pending/error states.
- **`useActionState`** — state hooks tied to actions.
- **`useFormStatus`** — pending state for forms.
- **`use()`** hook — read promises and context directly in render.
- **Server Components stable** in non-framework setups.
- **Ref as a prop** — no more `forwardRef` boilerplate.
- **Document metadata** (`<title>`, `<meta>`) directly in components.

### 15.3 The Mental Shift

Modern React is moving toward:
- Less manual optimization (compiler handles it).
- More server-side rendering by default.
- Forms as a first-class concept (actions, useActionState).
- Suspense as the data-loading paradigm.

Senior engineers should understand and use these patterns.

---

## 16. Phase 14 — Frontend System Design

### 16.1 The Frontend System Design Interview

In senior loops, you'll be asked to design:
- Twitter feed
- Google Docs (collaborative editing)
- Photo gallery (Pinterest)
- Chat (Slack / WhatsApp)
- E-commerce product page
- Real-time dashboard
- Component library / Design system
- Autocomplete search

### 16.2 The Framework

1. **Clarify requirements** — features, scale, devices, accessibility, performance budget.
2. **High-level architecture** — components, data flow, state management, API contracts.
3. **Detailed component design** — for 1-2 key components.
4. **Data layer** — fetching strategy, caching, real-time updates.
5. **Performance** — virtualization, code splitting, image strategy.
6. **Accessibility** — keyboard, ARIA, screen readers.
7. **Internationalization** — RTL, translations.
8. **Edge cases** — empty, error, loading, offline.

### 16.3 Component Library / Design System

Modern senior interviews emphasize this.

- Token system (color, spacing, typography).
- Primitive components (Button, Input).
- Compound components (Tabs, Accordion).
- Themability (CSS variables, ThemeProvider).
- Accessibility built-in.
- Documentation (Storybook).
- Versioning (semver, changesets).
- Distribution (npm, monorepo).

Study: Radix UI, shadcn/ui, Chakra, MUI, Mantine. Read their source.

### 16.4 Real-time Updates

- **WebSocket** — bidirectional, persistent.
- **SSE** — server-to-client streaming.
- **Long polling** — fallback.
- **Polling** — simple but wasteful.
- **Conflict resolution** — CRDTs (Yjs, Automerge) for collaborative editing.

### 16.5 Offline-First

- Service workers and the Cache API.
- IndexedDB for structured storage.
- Background sync.
- Frameworks: Workbox, RxDB, Replicache, ElectricSQL.

### 16.6 Accessibility (a11y)

- Semantic HTML first.
- ARIA only when semantic HTML isn't enough.
- Keyboard navigation for every interactive element.
- Focus management (modals, route changes).
- Color contrast (WCAG AA 4.5:1 for normal text).
- Screen reader testing (VoiceOver, NVDA).
- Tools: axe-core, eslint-plugin-jsx-a11y, Lighthouse.

### 16.7 Internationalization (i18n)

- **react-intl, react-i18next, lingui, FormatJS.**
- Plurals, dates, numbers, currencies.
- RTL languages (Arabic, Hebrew) — layout flips.
- Locale-aware sorting and casing.

### 16.8 Micro-Frontends

Splitting a large app into independently deployable frontends.
- **Module Federation** (webpack/rspack).
- **Single-spa.**
- **Iframes** for hard isolation.

Useful for large orgs with many teams. Overkill for most apps. Know the pattern; flag when it's appropriate.

---

## 17. Interview Questions

150+ questions organized by topic. Each has a complete, senior-level answer.

### 17.1 JavaScript Fundamentals

**Q1. Explain closures with an example.**
A closure is a function that captures variables from its lexical scope. When you create a function inside another function, the inner function "remembers" the variables of the outer scope, even after the outer function returns. Example: `function makeCounter() { let count = 0; return () => ++count; }`. Each call to `makeCounter()` returns a new function with its own `count`. Closures are why React's `useCallback` exists — to keep stable function references that close over current state values.

**Q2. What is the event loop?**
JavaScript runs on a single main thread. The event loop processes a queue of tasks. Synchronous code runs first. Then microtasks (promise callbacks) run. Then macrotasks (timers, I/O callbacks). The event loop picks the next task only when the current one finishes. This is why a long synchronous computation freezes the UI — there's nothing else the loop can do.

**Q3. Difference between `var`, `let`, `const`?**
`var` is function-scoped, hoisted, and re-declarable. `let` and `const` are block-scoped and don't hoist usefully (temporal dead zone). `const` prevents reassignment of the binding but doesn't make the value immutable. In modern code, use `const` by default; `let` when you reassign; never `var`.

**Q4. Explain `this` binding rules.**
Four rules in order: 1) `new` binding — `this` is the new instance. 2) Explicit binding — `call`, `apply`, `bind`. 3) Implicit binding — `obj.method()` makes `this` equal `obj`. 4) Default — global object (`window`) or `undefined` in strict mode. Arrow functions have no `this`; they inherit it from the enclosing scope.

**Q5. What is hoisting?**
Function declarations and `var` declarations are moved to the top of their scope before execution. `function foo()` is callable above its declaration; `let`/`const` are not (they exist but can't be accessed — the temporal dead zone). Hoisting is a runtime behavior, not a code transformation.

**Q6. Difference between `==` and `===`?**
`==` performs type coercion before comparison. `===` does not. `0 == false` is true; `0 === false` is false. Use `===` unless you have a specific reason — coercion rules surprise people.

**Q7. What's the difference between `null` and `undefined`?**
`undefined` is the default for uninitialized variables and missing properties. `null` is an explicit "no value" you assign. They're loosely equal (`null == undefined`) but not strictly. Modern code uses `null` for intentional empty values and `undefined` for unset.

**Q8. Explain promises and async/await.**
A promise is an object representing a future value. It can be pending, fulfilled, or rejected. `.then()` and `.catch()` schedule callbacks. `async`/`await` is syntactic sugar — `await promise` pauses the function until the promise settles, returning the value or throwing the error. Easier to read than chained `.then()`s.

**Q9. What does `Promise.all` do? What about `Promise.allSettled`, `Promise.race`, `Promise.any`?**
- `Promise.all([a,b])` — resolves when all resolve; rejects on first rejection.
- `Promise.allSettled([a,b])` — resolves when all settle (success or fail); array of results.
- `Promise.race([a,b])` — settles with the first to settle.
- `Promise.any([a,b])` — resolves with the first to resolve; rejects only if all reject.

**Q10. What is the difference between microtasks and macrotasks?**
Microtasks (promise callbacks, queueMicrotask) run after the current task and before the next macrotask. Macrotasks (setTimeout, setInterval, I/O) run one per loop iteration. Promises always run before timers, even with timer delay 0.

**Q11. Implement debounce.**
A debounce wraps a function so that it only fires after no calls have been made for a delay. Useful for search inputs, resize handlers. Each call clears a pending timer and sets a new one. If you describe: "store a timer ID; on each call, clearTimeout the old timer and setTimeout the new one for the delay; when the delay passes without new calls, fire the wrapped function with the last arguments" — that's it.

**Q12. Implement throttle.**
A throttle wraps a function so that it fires at most once per interval. On the first call, fire immediately and start a cooldown. During cooldown, ignore (or remember the last) calls. After cooldown ends, fire if there were ignored calls. Used for scroll handlers, mousemove.

**Q13. Difference between debounce and throttle?**
Debounce waits for a quiet period (fires last). Throttle enforces a rate (fires periodically). Use debounce for search-on-type. Use throttle for scroll and resize.

**Q14. What is currying?**
Transforming `f(a, b, c)` into `f(a)(b)(c)`. Useful for partial application and composition. `const add = a => b => a + b;`. Not used heavily in idiomatic React but appears in functional libraries.

**Q15. Explain shallow copy vs deep copy.**
Shallow copy duplicates the top level only — nested objects are shared by reference. `{...obj}` and `Object.assign({}, obj)` are shallow. Deep copy duplicates everything recursively. `structuredClone(obj)` is the modern built-in. For React state, prefer shallow copies plus explicit copies of nested fields you change.

### 17.2 React Core

**Q16. What is the virtual DOM?**
An in-memory tree of plain JavaScript objects describing what the UI should look like. React compares the new VDOM with the previous one ("diffing") and applies minimal changes to the real DOM. The win is declarative code and predictable updates, not raw speed — manual DOM updates can be faster if perfectly tuned. The VDOM is "fast enough" while being much easier to reason about.

**Q17. What is JSX?**
HTML-like syntax that compiles to `React.createElement(type, props, ...children)`. JSX is optional but standard. Babel or SWC transforms it at build time.

**Q18. Function components vs class components?**
Function components are the modern default. They use hooks for state and lifecycle. Class components have `this`, lifecycle methods (`componentDidMount`, etc.), and harder reuse patterns. Functions are simpler, more composable, and the only style that supports modern features (Server Components, Suspense data fetching).

**Q19. What are keys and why do they matter?**
Keys tell React's reconciliation algorithm which list items correspond between renders. Without stable keys, React can't tell that "the third item moved to first" — it might destroy and recreate components, losing focus and state. Use stable, unique keys (database IDs). Avoid array index unless the list is static and never reordered.

**Q20. What's the difference between props and state?**
Props are inputs to a component, set by the parent, read-only inside. State is internal data the component owns and updates. Changing state triggers a re-render. Lift state up when multiple components need it; pass via props or context.

**Q21. Why must we not mutate state?**
React detects state changes via reference equality. If you mutate the existing object (`state.x = 1; setState(state)`), the reference is unchanged, React skips the re-render, and the UI looks frozen. Always create new references for changed state.

**Q22. What is "lifting state up"?**
Moving state from a child to the nearest common ancestor when two siblings need it. The ancestor passes the value and an updater down via props. This is the basic answer for "how do I share state between siblings."

**Q23. What is prop drilling and how do you avoid it?**
Passing props through many layers just so a deep descendant can use them. Solutions: Context API for global-ish data; composition (pass JSX as children); a state library (Zustand, Redux) for app-wide state.

**Q24. Difference between controlled and uncontrolled components?**
Controlled: React state is the single source of truth (`<input value={x} onChange={e => setX(e.target.value)}/>`). Uncontrolled: DOM is the source of truth, accessed via refs. Controlled is the norm; uncontrolled is a perf optimization for big forms.

**Q25. What is reconciliation?**
The process by which React figures out what changed between renders. It diffs the previous VDOM with the new one and applies minimal DOM updates. The algorithm is O(n) thanks to two assumptions: different types mean different subtrees, and keys match list items.

### 17.3 Hooks

**Q26. What are React hooks?**
Functions that let function components have state, lifecycle, and other React features. Introduced in 16.8. They replaced most reasons to write class components.

**Q27. Why can't hooks be called conditionally?**
React tracks hooks by call order using a linked list on the fiber. If the order changes between renders, the pointer can't find the right slot, causing wrong values or crashes. The "rules of hooks" exist to keep the call order stable.

**Q28. How does `useState` work internally?**
Each `useState` call corresponds to a slot in the fiber's hook list. The first call creates the slot with the initial value. Subsequent calls read the slot. The setter, when called, marks the fiber dirty, schedules a re-render, and on the next render returns the new value.

**Q29. What does `useEffect` do?**
Runs side effects after the component commits to the DOM. Used for data fetching, subscriptions, manual DOM mutations, integrating non-React code. The cleanup function runs before the next effect and on unmount. Dependencies control when it re-runs.

**Q30. What's the difference between `useEffect` and `useLayoutEffect`?**
Both have the same API. `useEffect` runs asynchronously after the browser paints. `useLayoutEffect` runs synchronously after DOM mutations and before paint. Use `useLayoutEffect` only for measuring DOM and triggering re-renders before paint (avoiding flicker); otherwise prefer `useEffect`.

**Q31. When does `useEffect` run?**
After the component renders and commits. With no dependency array: every render. With empty `[]`: once after first render. With deps `[a]`: when `a` changes. In Strict Mode, twice on mount (to surface missing cleanups).

**Q32. What is a stale closure?**
An effect, callback, or async function captures variables at the time it was created. If those variables change but the function doesn't re-create, it sees old values. Common with `useEffect([])` capturing a state variable, or with `setTimeout` inside a handler. Fix: include the variable in deps, use the functional updater form of `setState`, or store in a ref.

**Q33. Why does my `useEffect` run infinitely?**
The effect updates state that's in its own dependency array. Every render creates new objects/arrays/functions; if those are in deps, the effect sees "new" deps every time. Fix: move stable values, memoize objects, use functional updaters, or remove the dep.

**Q34. What does `useRef` do?**
Returns a mutable object `{ current: value }` that persists across renders without triggering re-renders when mutated. Two uses: accessing DOM nodes (via `ref` JSX prop) and storing mutable values (timers, last-seen values, instance flags).

**Q35. Difference between `useRef` and `useState`?**
`useState` triggers re-renders on update; `useRef` does not. State changes are part of the render output; ref changes are side effects.

**Q36. When should you use `useMemo`?**
When a computation is expensive enough that the bookkeeping cost is justified, or when the value needs reference equality for downstream hooks/memoized children. Don't wrap trivial computations.

**Q37. When should you use `useCallback`?**
When passing a callback to a memoized child (otherwise the child re-renders), or when the callback is in another hook's deps. Otherwise unnecessary.

**Q38. What's the React Compiler and how does it change `useMemo`/`useCallback`?**
The React Compiler (stable in React 19) automatically memoizes components and values at build time, reducing the need for manual `memo`/`useMemo`/`useCallback`. As adoption grows, you'll write fewer manual memoizations.

**Q39. What is `useReducer` and when do you use it?**
A state hook for complex update logic. Takes a reducer function and dispatches actions. Use when state has many sub-fields that change together, or when update logic is non-trivial enough to benefit from a centralized function (easier testing, debugging).

**Q40. What is `useContext`?**
Reads a value from a Context provider higher in the tree. Used for global-ish data. Pitfall: every consumer re-renders when the value changes.

**Q41. How do you optimize Context to avoid unnecessary re-renders?**
Split the context into multiple smaller contexts. Memoize the provider value. Use `useMemo` for the value object. Or use Zustand/Redux for fine-grained subscriptions.

**Q42. What is `useTransition`?**
Marks an update as non-urgent. React can interrupt the transition if a more urgent update arrives. Used to keep the UI responsive during expensive renders (filtering large lists, switching tabs with heavy content).

**Q43. What is `useDeferredValue`?**
Similar to `useTransition` but applied to a value. The component renders with the old value first, then re-renders with the new value when React has time.

**Q44. What is `useId`?**
Generates a stable unique ID. Useful for accessibility (linking labels to inputs) and SSR (server and client must generate the same IDs).

**Q45. What is `useSyncExternalStore`?**
Subscribes a component to an external store (Redux, Zustand, browser APIs) in a way that's safe with concurrent rendering. State libraries use it internally; rarely written directly.

**Q46. How do you write a custom hook?**
A function whose name starts with `use` and which uses other hooks. Common pattern: encapsulate stateful logic that multiple components share. Examples: `useDebounce`, `useLocalStorage`, `useFetch`, `useMediaQuery`.

**Q47. Why must custom hooks start with `use`?**
React's linter detects hook calls by the `use` prefix. Without it, the linter can't enforce the rules of hooks for your custom hook's internal calls.

**Q48. What's the difference between `use()` and `useContext()` in React 19?**
The new `use()` hook can read promises *and* context, and it can be called inside conditionals and loops (unlike other hooks). It's a more flexible alternative to `useContext` and integrates with Suspense.

### 17.4 Component Patterns & Architecture

**Q49. Explain compound components.**
A parent and its children work together via shared context, exposed as `Parent.Child`. Example: `<Tabs><Tabs.List/><Tabs.Trigger/><Tabs.Content/></Tabs>`. Used in Radix, shadcn/ui. Flexible composition, but more setup than props.

**Q50. What are render props?**
A component takes a function as a child and calls it with internal state. Used to share rendering control with consumers. Largely replaced by hooks for logic sharing but still useful for rendering flexibility.

**Q51. What is a Higher-Order Component (HOC)?**
A function that takes a component and returns a new component, usually with extra props or behavior. `withAuth(MyComponent)` returns `MyComponent` wrapped with auth logic. Modern code uses hooks instead, but HOCs persist in libraries.

**Q52. When would you choose HOC over hook?**
Rarely. Use HOCs when you need to wrap a class component, modify the rendered output (not just provide values), or work with a library that returns components. Otherwise prefer hooks.

**Q53. What is composition over inheritance?**
React favors building UIs by composing small components rather than extending a class hierarchy. A `Button` doesn't inherit from `Element`; it composes children, renders a `<button>`, applies styles. Composition is more flexible and avoids deep class hierarchies.

**Q54. How do you build a reusable component library?**
Design tokens (color, spacing, typography), accessible primitives (Button, Input), compound components (Tabs, Dialog), themability (CSS variables), TypeScript types, Storybook documentation, tests, semver versioning, monorepo with changesets, published to npm.

**Q55. What is the children prop?**
A special prop that holds the JSX passed between a component's tags. `<Card>Hello</Card>` makes `'Hello'` the `children`. Used for composition.

**Q56. What is `React.Fragment` and why use it?**
A wrapper that doesn't render a DOM element. Used to return multiple elements from a component without an extra wrapper div. Short syntax: `<></>`.

**Q57. What is `React.cloneElement`?**
Clones a React element and lets you add or override props. Used in compound components and slot-like APIs. Modern alternatives (context, render props) often replace it.

**Q58. What is `React.Portal`?**
Renders children into a different DOM node outside the parent hierarchy. Used for modals, tooltips, dropdowns that must escape `overflow: hidden` or z-index contexts.

### 17.5 State Management

**Q59. When should you use Context vs Redux vs Zustand?**
Context: small global-ish data (theme, current user, locale). Redux Toolkit: complex client state with many actions, time-travel debugging, established team familiarity. Zustand: modern global state — minimal boilerplate, fine-grained subscriptions, the default for new projects in 2024+.

**Q60. Why is server state different from client state?**
Server state is cached locally but the source of truth lives elsewhere. It needs caching, revalidation, stale-while-revalidate, optimistic updates, retries, deduplication. Client state has no such needs. Tools like TanStack Query solve server state; Redux does not.

**Q61. What does TanStack Query do?**
Manages server state: caches by query key, deduplicates concurrent requests, refetches in the background, supports stale-while-revalidate, optimistic updates, pagination, infinite queries, devtools. The gold standard for fetching in React.

**Q62. What's stale-while-revalidate?**
A caching strategy: serve the stale (cached) value immediately, fetch a fresh value in the background, then update. Users see instant data and eventually get fresh data. Pattern used by TanStack Query, SWR, and HTTP caches.

**Q63. Explain optimistic updates.**
Update the UI immediately as if a mutation succeeded, send the request, and roll back if it fails. Makes apps feel instant. Risks: must handle failures gracefully. TanStack Query has first-class support.

**Q64. When would you choose Redux Toolkit today?**
Large teams already using Redux. Complex client state machines with many actions. Need for redux-devtools time-travel debugging. Otherwise, Zustand or TanStack Query are often better fits.

**Q65. How does Zustand differ from Redux?**
Zustand is minimal: no providers required, no reducers/actions ceremony, fine-grained subscriptions via selectors. Just a hook. Redux is more opinionated and structured but heavier.

**Q66. What is immer and where does it fit?**
A library that lets you write "mutating" code that actually produces immutable copies. Redux Toolkit uses it internally — you can `state.users.push(...)` in a reducer and get a new state object. Reduces boilerplate around deep updates.

### 17.6 Performance

**Q67. How do you profile a slow React app?**
Open React DevTools, switch to the Profiler tab, record an interaction. Look at the flame graph for slow renders. Use "Highlight updates" to see which components render on each interaction. Then Lighthouse / Web Vitals for browser-level metrics (LCP, INP, CLS).

**Q68. What causes unnecessary re-renders?**
Parent re-renders (children re-render by default). Props that are new references each render (inline objects, arrays, functions). Context value changes. State that updates but doesn't change. Fixes: `React.memo`, `useMemo`, `useCallback`, stable references, split contexts.

**Q69. When does `React.memo` actually help?**
When the component is expensive to render *and* its parent re-renders frequently with the same props. If props change every time anyway, `memo` only adds cost. Always profile before applying.

**Q70. Why might `React.memo` not work as expected?**
Because props include new references each render: inline objects, arrays, or functions. The memo's shallow comparison treats them as new. Wrap with `useMemo`/`useCallback` or move them out of the render path.

**Q71. How do you implement list virtualization?**
Render only the rows visible in the viewport (and a small buffer). Calculate which rows are visible from scroll position and item height. Libraries: react-window, @tanstack/react-virtual. For 10,000 rows, you render ~20 DOM nodes instead of 10,000.

**Q72. What is code splitting?**
Splitting your bundle into multiple chunks loaded on demand. `React.lazy(() => import('./Heavy'))` lazy-loads a component. Route-level splitting is the most impactful pattern.

**Q73. What is tree shaking?**
The bundler removes unused exports from your bundle. Requires ES modules (not CommonJS) and that libraries support it (declare `sideEffects: false` in package.json). Make sure imports are specific: `import { debounce } from 'lodash-es'` not `import _ from 'lodash'`.

**Q74. What is Suspense for?**
Originally for code splitting (`React.lazy`). In React 18+, also for data fetching in frameworks. Wrap a component that "suspends" (throws a promise) in `<Suspense fallback={...}>` to show a fallback while it waits.

**Q75. What is the React Compiler?**
A build-time tool (stable in React 19) that automatically inserts memoization where beneficial. Reduces the need for manual `memo`/`useMemo`/`useCallback`. Opts in per file or per project.

**Q76. How do you reduce bundle size?**
- Analyze with a bundle visualizer.
- Replace heavy libraries (moment → dayjs, lodash → es-toolkit).
- Tree-shake aggressively.
- Code-split at route boundaries.
- Lazy-load below-the-fold components.
- Use dynamic imports for rarely-used features.
- Remove dead code.

**Q77. What are Web Vitals?**
Google's user-experience metrics: LCP (largest contentful paint, target <2.5s), INP (interaction to next paint, target <200ms), CLS (cumulative layout shift, target <0.1). Measured in real users via the web-vitals library; lab-measured via Lighthouse.

**Q78. How do you fix a high INP?**
INP measures interaction responsiveness. High INP means long tasks blocking the main thread on user input. Fixes: break up long renders with `useTransition`; virtualize lists; move computation to web workers; reduce JavaScript on the critical path.

**Q79. What is concurrent rendering?**
React's ability to interrupt and resume rendering. Enables `useTransition`, `useDeferredValue`, Suspense for data, and selective hydration. Not "multi-threaded React" — it still runs on one thread, but it yields back to the browser between work units.

### 17.7 React Internals

**Q80. What is React Fiber?**
The internal data structure representing units of work. Each component instance corresponds to a fiber. Fibers form a tree (the work-in-progress) alongside the current committed tree. The fiber architecture (since React 16) enables incremental rendering, concurrent features, and better scheduling.

**Q81. What is the difference between render phase and commit phase?**
Render phase: React calls components, builds the work-in-progress fiber tree, figures out what changed. Can be interrupted. Commit phase: React applies DOM changes synchronously, runs `useLayoutEffect`, then `useEffect`. Cannot be interrupted.

**Q82. How does React batch updates?**
React groups multiple state updates within an event into a single re-render. React 18 expanded batching to async contexts (setTimeout, promises). You can opt out with `flushSync(() => setState(x))` (rarely needed).

**Q83. What is hydration?**
The process of attaching event handlers to server-rendered HTML on the client. The server sends HTML; React on the client "walks" it, attaches handlers, and turns it into a live React tree. Hydration mismatches (different output between server and client) cause warnings.

**Q84. What is selective hydration?**
React 18+ can hydrate parts of the page out of order, prioritizing what the user interacts with first. Improves perceived load time on large SSR pages.

**Q85. What is the difference between `useEffect` and `componentDidMount`?**
`componentDidMount` runs once after mount, synchronously after DOM commit. `useEffect` runs *asynchronously* after commit (the browser may paint between commit and effect). For pre-paint work, use `useLayoutEffect`.

**Q86. Why does React run effects twice in Strict Mode?**
To surface bugs where effects don't clean up properly. If your effect breaks when mounted twice, it would break in production scenarios like fast remounts or future concurrent features. The double-invocation only happens in dev.

### 17.8 TypeScript & Modern React

**Q87. How do you type a component's props?**
With `type` or `interface`. Prefer `type` for unions and intersections, `interface` when extending. Children type: `React.ReactNode`. Click handler: `React.MouseEventHandler<HTMLButtonElement>` or the more general `(e: React.MouseEvent) => void`.

**Q88. What is a generic component in TypeScript?**
A component that takes a type parameter, like `List<T>`. Useful for components that work with arbitrary item types while preserving type information for callbacks (e.g., `onSelect(item: T)`).

**Q89. What is a polymorphic component?**
A component that can render as different elements (`as` prop). Example: `<Box as="a" href="..."/>` vs `<Box as="button"/>`. Typing one well in TypeScript is non-trivial; libraries like Radix UI offer reference implementations.

**Q90. What's new in React 19?**
- React Compiler stabilized (auto-memoization).
- Actions in transitions (built-in pending/error states).
- `useActionState` and `useFormStatus`.
- `use()` hook reads promises and context anywhere.
- Server Components stable in framework-less setups.
- Ref as a prop (no more `forwardRef`).
- Document metadata as components (`<title>`, `<meta>`).

**Q91. What are Server Components?**
Components that render on the server only. They send HTML and a serialized payload to the client. No state, no effects, no event handlers. Can directly call databases and APIs. Reduce JS bundle and improve initial load.

**Q92. What are Server Actions?**
Server-side functions you call from the client as if they were local functions. Forms can use them directly with `action={myAction}`. Eliminates boilerplate around POST endpoints. Available in Next.js App Router and standalone via React's `'use server'` directive.

### 17.9 Forms

**Q93. Why is React Hook Form preferred over Formik?**
Performance. RHF uses uncontrolled inputs by default, drastically reducing re-renders. For large forms, this is significantly faster. RHF also has smaller bundle, simpler API, and excellent Zod integration.

**Q94. How do you validate forms with Zod and React Hook Form?**
Define a Zod schema. Pass it via `zodResolver(schema)` to RHF's `useForm`. RHF uses the schema for both runtime validation and TypeScript types. One source of truth.

**Q95. How do you handle file uploads with progress?**
Use `XMLHttpRequest` (the only API with progress events natively) or fetch with a custom progress reader. Better: use presigned URLs to upload directly to S3/GCS, bypassing your server. Show progress with a controlled `<progress>` element.

### 17.10 Testing

**Q96. What's the philosophy of React Testing Library?**
Test the way users use the app. Query by role, label, text — not by class or test-id. Test behavior, not implementation. Assertions check what the user sees.

**Q97. When do you use unit tests vs integration tests vs e2e tests?**
- Unit: pure functions, custom hooks, isolated component logic.
- Integration: component + its store + API mocks. Most React tests.
- E2E: critical user flows across pages. Slowest but highest confidence.

**Q98. What is MSW and why use it?**
Mock Service Worker intercepts network requests at the service-worker level. Tests look identical to real usage; the same mocks work in dev, in tests, and in Storybook. Better than mocking fetch directly.

### 17.11 Senior / System Design

**Q99. Design Twitter's feed in React.**
- Components: Feed (list), Tweet, Composer, Sidebar.
- Virtualization for the feed (TanStack Virtual).
- TanStack Query infinite query for pagination.
- Optimistic posting (composer adds the tweet immediately).
- Real-time updates via WebSocket merging into the query cache.
- Image lazy loading.
- Pre-fetch on hover.
- Accessibility: keyboard navigation through tweets.

**Q100. Design Google Docs collaborative editing.**
- CRDT-based state (Yjs).
- WebSocket transport.
- Local-first updates with conflict-free merging.
- Cursor positions and presence via awareness protocol.
- Content rendered via a contenteditable or a rich-text framework (TipTap, ProseMirror).
- Offline support via IndexedDB.
- Server stores snapshots periodically.

**Q101. Design an autocomplete search.**
- Debounced input.
- Cancel previous requests when new ones fire.
- TanStack Query with keepPreviousData for smooth UI.
- Keyboard navigation (arrow keys, Enter).
- Highlight matches in results.
- Empty / loading / error states.
- Accessibility: `aria-autocomplete="list"`, `aria-activedescendant`.

**Q102. Design a real-time chat UI.**
- Components: ChatList, MessageList, Composer, MessageItem.
- WebSocket connection with reconnect.
- Optimistic message sending.
- Virtualized message list (anchored to bottom).
- Read receipts, typing indicators (debounced).
- File uploads with progress.
- Message edit/delete.
- Markdown rendering for messages.

**Q103. Design a design system.**
- Token layer: CSS variables for colors, spacing, typography.
- Primitive components: Button, Input, Stack, Box.
- Compound components: Tabs, Dialog, Tooltip.
- Theming via CSS variables (`data-theme="dark"`).
- Accessibility: keyboard, ARIA, focus management.
- Documentation in Storybook.
- TypeScript types for all props.
- Distributed via npm, versioned with semver, automated with changesets.

**Q104. Design an offline-first app.**
- Local state via IndexedDB (Dexie or RxDB).
- Service worker for asset caching.
- Background sync for outgoing writes.
- Conflict resolution strategy (last-write-wins, CRDTs).
- Optimistic UI everywhere.
- Visual indicator of online/offline status.
- Replicache or ElectricSQL as a sync framework.

**Q105. How do you implement accessibility (a11y) in a React app?**
- Semantic HTML first (`<button>`, `<nav>`, `<main>`).
- ARIA only when semantic HTML isn't enough.
- Keyboard navigation for every interactive element.
- Focus management (modals trap focus, route changes move focus).
- Color contrast (4.5:1 WCAG AA).
- Screen reader testing (VoiceOver, NVDA).
- Tools: axe-core, eslint-plugin-jsx-a11y, Lighthouse.

**Q106. How do you internationalize a React app?**
- A library (react-intl, react-i18next, lingui).
- Externalize strings into translation files (JSON).
- Handle plurals, dates, numbers, currencies via Intl API.
- RTL support (Arabic, Hebrew) via CSS logical properties.
- Test with pseudo-locales (expanded text) to catch overflow.

**Q107. How do you secure a React frontend?**
- Sanitize user-generated HTML (DOMPurify).
- Avoid `dangerouslySetInnerHTML` unless required.
- Use HttpOnly cookies for auth tokens (not localStorage).
- CSP headers to limit script sources.
- CSRF tokens for cookie-based auth.
- Never trust the client — re-validate server-side.

**Q108. How do you handle errors gracefully?**
- Error boundaries around route subtrees and major components.
- Per-request error states in TanStack Query.
- Global toast on network errors.
- Retry strategies for transient failures.
- Track errors with Sentry.
- Show actionable error messages, not raw stack traces.

**Q109. What is an error boundary?**
A class component implementing `componentDidCatch` and/or `getDerivedStateFromError`. Catches errors from descendants during render, lifecycle, and constructors. Does *not* catch errors in event handlers, async code, or itself. Use `react-error-boundary` for a hook-friendly API.

**Q110. How do you implement dark mode?**
- CSS variables for colors.
- `data-theme="dark"` attribute on `<html>`.
- Toggle stored in localStorage and respecting `prefers-color-scheme` on first load.
- Avoid flash of wrong theme by setting the attribute before React hydrates (inline script in `<head>`).

### 17.12 Next.js Specific

**Q111. App Router vs Pages Router?**
App Router (Next 13+): based on React Server Components, layouts, streaming, server actions, advanced caching. The future. Pages Router (classic): `getServerSideProps`/`getStaticProps`, simpler mental model, still maintained.

**Q112. What are the four caching layers in App Router?**
1. Request Memoization — dedupes identical fetches within one request.
2. Data Cache — persistent across requests and deploys, configurable revalidation.
3. Full Route Cache — static rendering at build/request time.
4. Router Cache — client-side cache of navigated routes.

Understanding when each applies is a senior-level skill.

**Q113. What's the difference between `'use client'` and `'use server'`?**
`'use client'` at the top of a file marks it (and its imports) as client components — they ship JS and hydrate in the browser. `'use server'` marks a function as a server action — it runs on the server even when called from a client component.

**Q114. When should a component be a Server Component vs Client Component?**
Default to Server Component. Convert to Client Component only when you need state, effects, event handlers, browser APIs, or third-party client-only libraries. The deeper the client boundary, the smaller your JS bundle.

**Q115. What is streaming SSR?**
The server sends HTML in chunks as it's ready, instead of waiting for everything. With Suspense boundaries, the user sees a fast first paint while slower data loads stream in. Improves perceived performance dramatically.

**Q116. How do you handle authentication in Next.js?**
- Session cookies set by the server (HttpOnly, Secure, SameSite).
- Auth check in middleware for protected routes.
- NextAuth (now Auth.js) for OAuth providers.
- Or roll your own with iron-session or jose.

### 17.13 Behavioral / Senior Topics

**Q117. How do you decide whether to use a third-party library or build something?**
- How well does it fit my use case (no over-fitting)?
- How well maintained is it (recent commits, open issues, weekly downloads)?
- What's the bundle cost?
- What's the API stability story?
- Can I replace it later without rewriting everything?
- Could I build the minimal version in a day? Then I should.

**Q118. How do you handle a complex component that's become unmanageable?**
- Identify the seams (data fetching, state, rendering, side effects).
- Extract custom hooks for stateful logic.
- Split sub-components by responsibility.
- Replace prop drilling with context or a state library.
- Write tests as you refactor so behavior is preserved.

**Q119. How do you onboard a new developer to a React codebase?**
- Architecture diagram showing the major boundaries.
- A 30-minute walkthrough of "what happens when a user does X."
- Conventions doc (component structure, naming, testing).
- A "starter ticket" that touches multiple parts of the codebase.
- Pair programming for the first week.

**Q120. How do you handle a disagreement with a designer over implementation feasibility?**
- Understand what they're actually trying to achieve (the design's intent).
- Propose alternatives that achieve the same outcome with less complexity or cost.
- If the design is necessary as-is, scope the effort honestly.
- Don't say no — say "yes, and here's what it costs; is there a faster path that still works?"

---

## 18. Coding Challenges to Practice

Build each from scratch. No googling for the first attempt. Each tests a specific skill.

### Easy
1. **Counter with increment/decrement** — `useState` basics.
2. **Todo list with add/remove/edit/toggle** — list updates, immutability.
3. **Toggle visibility** — conditional rendering.
4. **Tabs component** — controlled state, ARIA roles.
5. **Modal/Dialog** — Portals, focus trap.
6. **Tooltip** — positioning, mouse events.
7. **Star rating** — controlled component, hover state.
8. **Light/dark theme toggle** — CSS variables, persistence.

### Medium
9. **Search with debounce** — `useEffect`, `useCallback`, cleanup.
10. **Autocomplete** — keyboard nav, ARIA, async.
11. **Pagination** — useState, derived data.
12. **Infinite scroll list** — IntersectionObserver, useEffect.
13. **Virtualized list (10k items)** — measure, render windowed range.
14. **Drag and drop** — pointer events, state.
15. **Multi-step form wizard** — state machine, validation.
16. **Stopwatch** — useEffect timer, cleanup.
17. **Image carousel** — keyboard nav, indicators.
18. **Accordion** — compound component.
19. **Type-ahead with cancellation** — AbortController, race conditions.
20. **Toast notification system** — context + reducer + portal.

### Hard
21. **Spreadsheet (rows, columns, formula cells)** — virtualization, dependency tracking.
22. **Rich text editor with toolbar** — contenteditable, commands.
23. **Trello-like board** — drag-drop columns and cards, optimistic updates.
24. **Calendar with month/week view** — date math, virtualization.
25. **Real-time chat with WebSocket** — connection management, optimistic updates.
26. **Component library primitives (Dialog, Popover, Tooltip)** — focus management, accessibility.
27. **GraphQL client (subset)** — caching, normalization.
28. **State management library (Zustand subset)** — `useSyncExternalStore`.
29. **Routing library (subset)** — history API, route matching, nested routes.
30. **Server Component renderer (subset)** — async rendering, streaming.

---

## 19. Behavioral & Project Prep

Prepare 8-10 concrete stories. Use STAR format (Situation, Task, Action, Result).

### Story Topics to Have Ready
- A challenging bug you debugged
- A performance issue you fixed (with metrics)
- A disagreement with a teammate and how you resolved it
- A feature you owned end-to-end
- A time you said no to a stakeholder
- A time you simplified an over-engineered system
- A time you missed a deadline
- A time you mentored someone
- A time you championed a technology decision
- A time you learned something new fast

### When Discussing Your Projects
For each, be ready to describe:
- The problem and constraints.
- Why React (vs alternatives).
- Architecture choices and trade-offs.
- Performance considerations.
- What you'd do differently with hindsight.
- Specific challenges and how you solved them.

---

## 20. Resources Library

### Books
- **"React: Up & Running"** — Stoyan Stefanov
- **"Learning React"** — Alex Banks, Eve Porcello
- **"Fluent React"** — Tejas Kumar (modern, internals-focused)
- **"Designing Data-Intensive Applications"** — Martin Kleppmann (not React, but mandatory for senior interviews)
- **"You Don't Know JS Yet"** — Kyle Simpson (free on GitHub)

### Official Docs (Read End-to-End)
- **react.dev** — the new official docs; outstanding
- **nextjs.org/docs** — Next.js
- **tanstack.com/query** — TanStack Query
- **react-hook-form.com** — React Hook Form

### Courses
- **Epic React** — Kent C. Dodds (definitive)
- **Frontend Masters paths** — React, Advanced React, Next.js
- **"The Joy of React"** — Josh Comeau (gentle, deep)
- **Total TypeScript** — Matt Pocock (mandatory for TS)
- **ByteByteGo** — system design including frontend
- **Hello Interview** — frontend interview prep

### YouTube Channels
- **Theo - t3.gg** — modern stack, opinionated
- **Jack Herrington** — depth on modern React patterns
- **Web Dev Simplified** — clear explanations
- **Lee Robinson** (VP of Vercel) — Next.js + DX
- **ThePrimeagen** — broader engineering culture
- **Fireship** — quick concept videos
- **Matt Pocock** — TypeScript depth
- **Josh Comeau** — visualizations and CSS
- **Cosden Solutions** — React patterns

### Newsletters & Blogs
- **react.dev/blog** — official
- **overreacted.io** — Dan Abramov (deep dives)
- **joshwcomeau.com** — explanatory pieces
- **kentcdodds.com/blog** — practical patterns
- **leerob.io** — Next.js and Vercel
- **swyx.io** — broader frontend trends
- **Bytes** newsletter
- **This Week in React**

### Communities
- **r/reactjs** (Reddit)
- **React Discord** (large, active)
- **DEV.to** React tag
- **Twitter/X** — follow Dan Abramov, Sebastian Markbåge, Andrew Clark, Sophie Alpert, Rick Hanlon, Josh Story, Ricky, Tanner Linsley, Lee Robinson, Theo, Wes Bos

### Reference Code to Read
- **Radix UI** — accessibility-first primitives
- **shadcn/ui** — composable components on Radix
- **TanStack Query** — server state mastery
- **Zustand** — minimal state lib
- **React Router v6/v7** — modern routing
- **Next.js source** — framework-level patterns

---

## 21. Study Plan

### Beginner (0-3 months)
- Daily: 1 hour JavaScript fundamentals
- Daily: 1 hour building small React apps
- Weekly: 1 mini project (counter → todo → tabs → modal → search)
- End of phase: build a todo app with localStorage, dark mode, filtering

### Intermediate (3-6 months)
- Daily: 1 hour reading react.dev or working through Epic React
- Daily: 1 hour building features for a single bigger project
- Weekly: solve 3-5 React coding challenges
- End of phase: build a complete CRUD app with auth, routing, TanStack Query, forms with Zod, tested with RTL

### Advanced (6-9 months)
- Daily: 1 hour internals/system design study
- Daily: 1 hour deep-dive project (real-time chat, design system, etc.)
- Weekly: 1 system design exercise
- Weekly: 2-3 interview-style coding challenges
- End of phase: portfolio project — a production-grade React app deployed, monitored, performance-tuned

### Interview Prep (final 4-8 weeks)
- Weekly: 5 LeetCode mediums focused on arrays/strings/hashmaps
- Weekly: 3 frontend system designs (whiteboard out loud)
- Daily: review one section of this doc and 10 interview questions
- Weekly: 1 mock interview (Pramp, Hello Interview, friend)

---

## 22. Common Mistakes

- **Skipping JavaScript foundations.** You'll fight React forever without them.
- **Using `useEffect` for everything.** Many "effects" should be derived state or event handlers.
- **Putting server state in Redux.** Use TanStack Query.
- **Premature optimization.** Profile before reaching for `memo`/`useMemo`/`useCallback`.
- **Memoizing everything.** The bookkeeping is not free.
- **Array index as key.** Causes bugs in dynamic lists.
- **Mutating state.** React skips the re-render because reference didn't change.
- **Ignoring accessibility.** Senior interviews check this.
- **Skipping TypeScript.** Most senior jobs require it.
- **Building only side projects, never reading source code.** Reading Radix, TanStack, and Zustand source teaches more than another tutorial.
- **Never running Lighthouse.** Real performance work needs measurement.
- **Treating Next.js like React.** App Router has its own mental model — caching, server components, server actions.

---

## Closing Notes

React is large but learnable. The path from beginner to senior is 12-24 months of focused work for most people. The investment compounds — every senior frontend role builds on the same foundation.

The single best predictor of interview success: building one polished, deployed, performance-tuned React project end to end. Three projects is better. Reading source code and writing about what you learned is a force multiplier.

Bookmark this file. Open it weekly. Mark off items as you complete them. Update it as React itself evolves.

Good luck.
