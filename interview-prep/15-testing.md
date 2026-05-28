# 15 — Testing (10 Questions)

You used Jest for unit testing and shipped features with testing as part of the delivery cycle. These cover the testing pyramid, test doubles, integration vs unit, async/DB testing, coverage, and the gotchas that make test suites either valuable or a maintenance burden.

---

### Q1. Explain the testing pyramid (and the trophy). What's the right mix?

**Answer:**
The **testing pyramid** (Mike Cohn) describes the ideal proportion of test types:

```
        ╱╲
       ╱E2E╲          few   — slow, brittle, expensive, high confidence
      ╱──────╲
     ╱ Integ. ╲       some  — moderate speed, test real interactions
    ╱──────────╲
   ╱   Unit     ╲     many  — fast, cheap, isolated
  ╱──────────────╲
```

- **Unit tests** (many) — test one function/class in isolation. Fast (ms), no I/O. Run thousands in seconds.
- **Integration tests** (some) — test how units work together with real dependencies (DB, queue). Slower, more realistic.
- **E2E tests** (few) — test the whole system through its interface (HTTP, UI). Slow, brittle, but highest confidence.

**The testing trophy** (Kent C. Dodds) revises this for modern apps, emphasizing **integration tests** as the highest-value layer:

```
      ╱────────╲
      │   E2E   │     few
      ├─────────┤
      │  Integ. │     MOST  ← biggest payoff
      ├─────────┤
      │  Unit   │     some
      ╰────┬────╯
       Static (types, lint)   ← cheapest, catches a class of bugs free
```

**The argument for the trophy:** integration tests catch the most real bugs per effort because they test actual interactions (a service + a real DB), where most bugs hide. Pure unit tests on code that mostly orchestrates other code often just test that mocks were called — low value.

**Right mix depends on the code:**
- Pure logic (parsers, calculators, validators) → unit tests.
- Service-layer code that coordinates DB/external calls → integration tests.
- Critical user flows → a few E2E tests.
- Everything → static checking (TypeScript, ESLint) as the free first layer.

**Follow-up 1: Why does the trophy de-emphasize unit tests?**
Because in typical web backends, most code coordinates other code (service calls repo, repo calls DB). Unit-testing it means mocking everything, so you test "did I call the mock correctly" — which passes even when the real integration is broken. Integration tests with a real DB catch the bugs that matter (wrong query, bad migration, transaction issue). Unit tests stay valuable for genuinely complex pure logic.

**Follow-up 2: What's the cost of too many E2E tests?**
E2E tests are slow (seconds to minutes each), flaky (timing, environment, network), and expensive to maintain (UI changes break them). A suite that's 80% E2E takes hours to run, fails randomly, and developers learn to ignore failures. Keep E2E for a handful of critical paths; push detail down to integration/unit where tests are fast and stable.

---

### Q2. What should a unit test actually test? What makes a good one?

**Answer:**
A unit test verifies one unit of behavior in isolation, with dependencies replaced by test doubles.

**Characteristics of good unit tests (FIRST):**
- **Fast** — milliseconds. No I/O, no network, no real DB.
- **Isolated** — independent of other tests; can run in any order; no shared state.
- **Repeatable** — same result every run (no reliance on time, randomness, environment without controlling them).
- **Self-validating** — pass/fail is unambiguous; no manual inspection.
- **Timely** — written alongside (or before) the code.

**What to test:**
- **Behavior, not implementation.** Test "given X input, returns Y output / has Z effect" — not "calls method foo then bar." Implementation-coupled tests break on every refactor.
- **Edge cases** — empty input, nulls, boundaries, max values, error conditions.
- **The contract** — what the function promises to callers.

**What NOT to test:**
- Trivial getters/setters with no logic.
- The framework / library (don't test that Express routes work — test your handler logic).
- Implementation details that should be free to change.

**Code — testing behavior, not implementation:**
```ts
// Good — tests behavior
it('normalizes email to lowercase on create', async () => {
  const repo = { save: jest.fn(u => Promise.resolve(u)) };
  const service = new UserService(repo);
  const result = await service.create({ email: 'MIXED@Case.com' });
  expect(result.email).toBe('mixed@case.com');
});

// Bad — tests implementation; breaks if you refactor internals
it('calls toLowerCase then save', async () => {
  // coupled to HOW it works, not WHAT it does
});
```

**The AAA structure:**
```ts
it('does the thing', () => {
  // Arrange — set up inputs and dependencies
  const input = {...};
  // Act — call the unit under test
  const result = doThing(input);
  // Assert — verify the outcome
  expect(result).toBe(expected);
});
```

**Follow-up 1: How do you avoid tests that break on every refactor?**
Test through the public interface, assert on outcomes (return values, observable effects), not internal calls. Avoid over-mocking — every mock is a coupling to implementation. If a refactor that preserves behavior breaks your tests, your tests were testing implementation. The goal: refactor freely, tests stay green if behavior is unchanged.

**Follow-up 2: Should you test private methods?**
No — test them through the public methods that use them. Private methods are implementation details; if they're complex enough to warrant their own tests, that's a signal to extract them into a separate unit with its own public interface. Testing privates directly couples tests to internals and resists refactoring.

---

### Q3. Explain test doubles — mocks, stubs, spies, fakes, dummies.

**Answer:**
"Test double" is the umbrella term for any stand-in for a real dependency (like a stunt double). The specific types (per Gerard Meszaros / Martin Fowler):

- **Dummy** — passed but never used. Fills a parameter slot. `new Service(dummyLogger)` where logger is never called in this test.

- **Stub** — returns canned answers. "When `findById` is called, return this user." No verification of how it's used.

- **Spy** — a stub that also records how it was called. "Return this user, AND let me check it was called with the right ID."

- **Mock** — pre-programmed with expectations; verifies it was called as expected (often fails the test if not). "Expect `send` to be called exactly once with these args."

- **Fake** — a working but simplified implementation. An in-memory database instead of a real one. Has real behavior, just not production-grade.

**In Jest, the lines blur** — `jest.fn()` is a spy/mock; `jest.mock()` auto-mocks modules.

**Code — each type:**
```ts
// Stub — canned return
const repo = { findById: jest.fn().mockResolvedValue({ id: '1', name: 'Alice' }) };

// Spy — stub + verify calls
const mailer = { send: jest.fn() };
await service.register(dto);
expect(mailer.send).toHaveBeenCalledWith('alice@example.com', expect.any(String));

// Mock — strict expectation
const gateway = { charge: jest.fn().mockResolvedValue({ id: 'ch_1' }) };
await service.checkout(order);
expect(gateway.charge).toHaveBeenCalledTimes(1);

// Fake — working in-memory implementation
class InMemoryUserRepo implements UserRepository {
  private store = new Map<string, User>();
  async save(u: User) { this.store.set(u.id, u); return u; }
  async findById(id: string) { return this.store.get(id) ?? null; }
}
```

**When to use which:**
- **Stub** when you just need the dependency to return something so the code under test can proceed.
- **Spy/Mock** when verifying the interaction IS the behavior you're testing (e.g., "an email gets sent").
- **Fake** when you need realistic behavior without the real dependency (in-memory DB for fast tests).

**Follow-up 1: When does mocking go too far?**
When you mock so much that you're testing mocks, not real behavior. "The test passes but production breaks." Over-mocking couples tests to implementation and gives false confidence. If you find yourself mocking five layers deep, consider an integration test with real components instead — it's more realistic and often simpler.

**Follow-up 2: Mock vs fake for a database?**
Mocking the DB (`jest.fn()` returning canned rows) is fast but tests nothing about your actual queries — a broken SQL query passes the test. A fake (in-memory DB) or, better, a real DB in a container (integration test) catches query bugs. For repository/query logic, prefer a real DB; for higher-level service logic where the DB is incidental, a mock or fake is fine. This is the "don't mock the database" feedback many teams adopt after getting burned.

---

### Q4. Unit vs integration vs E2E — concretely, what does each test in a NestJS app?

**Answer:**

**Unit test** — one service/class, dependencies mocked.
```ts
// Tests UsersService logic, repo mocked
const service = new UsersService(mockRepo, mockHasher);
expect(await service.create(dto)).toMatchObject({ email: 'a@b.com' });
```
Tests: business logic, validation, transformations. Fast, no I/O.

**Integration test** — multiple real components together (e.g., service + real DB).
```ts
// Real Postgres (testcontainers), real repository, real service
beforeAll(async () => { db = await startTestDb(); });
it('persists and retrieves a user', async () => {
  const created = await usersService.create({ email: 'a@b.com' });
  const found = await usersService.findById(created.id);
  expect(found.email).toBe('a@b.com');   // verifies the actual query works
});
```
Tests: real queries, transactions, migrations, how components interact. Catches the bugs unit tests miss.

**E2E test** — the whole app through HTTP.
```ts
// Boot the full NestJS app, hit it with supertest
const res = await request(app.getHttpServer())
  .post('/users')
  .set('Authorization', `Bearer ${token}`)
  .send({ email: 'a@b.com', password: 'secret123' });
expect(res.status).toBe(201);
expect(res.body.email).toBe('a@b.com');
```
Tests: routing, middleware, guards, validation pipes, serialization — the full request lifecycle. Highest confidence, slowest.

**The right boundaries:**
- A bug in your `calculateDiscount` pure function → unit test.
- A bug in your "find active subscribers" query → integration test (real DB).
- A bug in "unauthenticated requests are rejected" → E2E test (full guard chain).

**Follow-up 1: How do you set up a real DB for integration tests?**
**Testcontainers** — spins up a real Postgres/Redis/etc. in a Docker container for the test run, tears it down after. Gives you a real DB without polluting a shared one. Alternatively, a dedicated test DB in CI (a Postgres service in GitHub Actions, section 09 Q11). Run migrations against it, isolate tests (transaction rollback or truncate between tests).

**Follow-up 2: How do you handle auth in E2E tests?**
Two approaches: (1) override the auth provider with a test double (`overrideProvider(KeycloakService).useValue(...)`) so any request is "authenticated" as a test user; (2) generate a real valid test token signed with a test key. Option 1 is simpler and faster; option 2 tests the real auth path. Use 1 for testing business logic through the API, a few of 2 to verify auth itself works.

---

### Q5. How do you test asynchronous code correctly?

**Answer:**
Async code (promises, callbacks, timers, events) has specific testing pitfalls — mainly tests that pass without actually asserting, because the assertion runs after the test "finishes."

**Promises / async-await:**
```ts
// Correct — await the promise, assertion runs after resolution
it('fetches a user', async () => {
  const user = await service.findById('1');
  expect(user.name).toBe('Alice');
});

// Testing rejection
it('throws when not found', async () => {
  await expect(service.findById('missing')).rejects.toThrow(NotFoundException);
});
```

**Common bug — forgetting to await/return:**
```ts
// BAD — test passes even if the assertion would fail,
// because the test function returns before the promise resolves
it('fetches a user', () => {
  service.findById('1').then(user => {
    expect(user.name).toBe('Bob');   // never actually checked!
  });
});
```

**Timers — use fake timers instead of real waits:**
```ts
it('retries after backoff', async () => {
  jest.useFakeTimers();
  const promise = retryWithBackoff(failingFn);
  await jest.advanceTimersByTimeAsync(1000);   // fast-forward instead of waiting
  // ... assert
  jest.useRealTimers();
});
```

**Events:**
```ts
it('emits order.placed', async () => {
  const handler = jest.fn();
  service.on('order.placed', handler);
  await service.placeOrder(dto);
  expect(handler).toHaveBeenCalledWith(expect.objectContaining({ id: expect.any(String) }));
});
```

**Follow-up 1: Why use fake timers instead of real `setTimeout`?**
Real timers make tests slow (a test for "retry after 30s" would take 30 seconds) and flaky (timing-dependent). Fake timers let you control time — advance instantly to trigger timeouts/intervals deterministically. A retry-with-backoff test runs in milliseconds and is reliable. Just remember to restore real timers after.

**Follow-up 2: How do you test code that has a race condition?**
Carefully — races are hard to test deterministically. Approaches: (1) control concurrency explicitly (resolve promises in a specific order using deferred promises); (2) run the operation many times to surface flakiness (not deterministic but increases detection odds); (3) better, design the code to be race-free (locks, atomic operations) and test the safety mechanism directly. Often the best fix is making the race impossible rather than testing for it.

---

### Q6. What is TDD and when is it valuable?

**Answer:**
**Test-Driven Development** — write the test first, watch it fail, write minimal code to pass, then refactor. The "red-green-refactor" cycle:

```
   1. RED    — write a failing test for the next bit of behavior
   2. GREEN  — write the simplest code that makes it pass
   3. REFACTOR — clean up, tests stay green
   repeat
```

**Benefits:**
- **Forces clear requirements** — you can't write a test without knowing the expected behavior.
- **Designs for testability** — code written test-first tends to have good seams (DI, small units).
- **Builds a safety net** — comprehensive tests as a byproduct.
- **Prevents over-engineering** — you write only code that makes a test pass.
- **Fast feedback** — you know immediately when something breaks.

**When it's valuable:**
- Complex logic with clear input/output (algorithms, business rules, parsers).
- Bug fixes — write a test that reproduces the bug (red), fix it (green). Prevents regression.
- Well-understood requirements.

**When it's awkward:**
- Exploratory/prototype code where the design is unknown — you're discovering the requirements as you go.
- UI/visual work where "correct" is subjective.
- Code dominated by external integration where the unit test would be all mocks.

**Code — TDD a discount calculator:**
```ts
// 1. RED — write the test first
it('applies 10% discount for orders over $100', () => {
  expect(calculateDiscount(150)).toBe(15);
});
// (calculateDiscount doesn't exist yet — test fails to compile/run)

// 2. GREEN — minimal implementation
function calculateDiscount(total: number): number {
  return total > 100 ? total * 0.1 : 0;
}

// 3. Add next case (RED), extend (GREEN), refactor
it('no discount under $100', () => expect(calculateDiscount(50)).toBe(0));
```

**Follow-up 1: Is TDD always the right approach?**
No. TDD shines for logic with clear specs and for bug fixes (reproduce-then-fix). It's awkward for exploratory work where you're figuring out the design, or integration-heavy code. Many pragmatic developers use TDD selectively — strict for core logic and bug fixes, looser for glue code and prototypes. The dogma ("always TDD or you're doing it wrong") is less useful than the judgment of when it helps.

**Follow-up 2: What's the "write a failing test for the bug first" practice?**
For any bug fix: first write a test that reproduces the bug (it fails — confirming you've captured the bug). Then fix the code (the test passes). This guarantees (1) you actually understood and reproduced the bug, (2) the fix works, (3) the bug never silently returns (regression test). It's TDD applied to bug fixing — high value, widely adopted even by non-TDD teams.

---

### Q7. How do you handle test data and database state between tests?

**Answer:**
Tests that hit a real DB need isolation — one test's data mustn't affect another. Strategies:

**1. Transaction rollback per test:**
- Wrap each test in a transaction; roll back after.
- Fast (no actual writes committed). 
- Caveat: breaks if the code under test manages its own transactions (nested transaction issues).

**2. Truncate tables between tests:**
- After each test, `TRUNCATE` all tables.
- Simple, reliable, works with any transaction usage.
- Slower than rollback but robust.

**3. Fresh schema/database per test suite:**
- Each suite gets a clean DB (or schema).
- Strong isolation, slower setup.
- Good with testcontainers.

**4. Unique data per test:**
- Each test creates data with unique keys (random IDs, timestamped emails).
- No cleanup needed if data doesn't conflict.
- Risk: data accumulates; tests can interfere if they query broadly.

**Code — truncate strategy:**
```ts
afterEach(async () => {
  await dataSource.query(`
    TRUNCATE TABLE users, orders, payments RESTART IDENTITY CASCADE
  `);
});
```

**Code — transaction rollback (with TypeORM):**
```ts
let queryRunner: QueryRunner;
beforeEach(async () => {
  queryRunner = dataSource.createQueryRunner();
  await queryRunner.connect();
  await queryRunner.startTransaction();
});
afterEach(async () => {
  await queryRunner.rollbackTransaction();   // undo everything this test did
  await queryRunner.release();
});
```

**Seed data:**
- **Factories** (like `faker` + a builder) generate test entities with sensible defaults, overridable per test.
```ts
function makeUser(overrides = {}) {
  return { id: uuid(), email: faker.internet.email(), role: 'user', ...overrides };
}
```
- Avoid giant shared fixtures — they couple tests and make failures hard to diagnose. Each test creates exactly the data it needs.

**Follow-up 1: Why prefer factories over shared fixtures?**
Shared fixtures (one big "seed.sql" all tests use) create hidden coupling — change the fixture, break unrelated tests. Tests also depend on data they didn't create, obscuring what's actually being tested. Factories let each test declare exactly the data it needs inline, making tests self-contained and readable. Failures point to the specific test's data, not a shared blob.

**Follow-up 2: Transaction rollback vs truncate — which is better?**
Rollback is faster (nothing commits) and gives perfect isolation, BUT it breaks when the code under test uses its own transactions (you get nested-transaction confusion) or when you need to test commit behavior itself. Truncate is slower but works universally. Many teams use truncate for reliability; use rollback when speed matters and your code doesn't fight it. For testing transactional behavior specifically, you need real commits, so neither pure approach works — use a fresh DB.

---

### Q8. What does code coverage tell you — and what doesn't it?

**Answer:**
**Code coverage** measures what percentage of your code is executed during tests.

**Types:**
- **Line coverage** — % of lines executed.
- **Branch coverage** — % of branches (if/else paths) taken. More meaningful than line.
- **Function coverage** — % of functions called.
- **Statement coverage** — % of statements run.

**What coverage tells you:**
- **What's definitely NOT tested.** 0% coverage on a module = no tests touch it. Useful for finding gaps.
- A rough health signal across the codebase.

**What coverage does NOT tell you:**
- **Whether tests are good.** 100% coverage with no assertions tests nothing — the code ran, but nothing was verified.
- **Whether behavior is correct.** Coverage measures execution, not correctness.
- **Whether edge cases are handled.** A line can be covered by one input but fail on others.

**The trap — chasing a coverage number:**
- Mandating "90% coverage" leads to tests written to hit lines, not to verify behavior. Developers write assertion-free tests, or test trivial code, to pump the number.
- Coverage is a tool for finding *untested* code, not a goal in itself.

**Code — coverage can lie:**
```ts
function divide(a: number, b: number): number {
  return a / b;   // 100% line coverage with this test:
}
it('divides', () => { divide(10, 2); });   // no assertion! covers the line, tests nothing
// And misses the edge case: divide(10, 0) → Infinity
```

**Useful targets:**
- Aim for high coverage on **critical logic** (payment, auth, core business rules).
- Don't obsess over covering trivial code (getters, framework glue).
- Track coverage trends (is it dropping?) more than absolute numbers.
- **Branch coverage** is more meaningful than line coverage.

**Follow-up 1: What's a reasonable coverage target?**
There's no universal number. 70-80% is a common pragmatic range, with critical paths at 90%+. But the number matters less than what's covered — 80% that covers all the business logic beats 95% that covers getters while missing edge cases. Use coverage to find untested critical code, not as a KPI to game.

**Follow-up 2: What's mutation testing and how does it improve on coverage?**
Mutation testing introduces small bugs ("mutants") into your code (flip a `>` to `>=`, change a `+` to `-`) and checks whether your tests catch them. If a mutant survives (tests still pass), your tests are weak there — they execute the code but don't actually verify its behavior. It measures test *quality*, not just coverage. Tools: Stryker (JS/TS). More expensive to run, but reveals assertion-free or weak tests that coverage misses.

---

### Q9. How do you test code with external dependencies (APIs, third parties)?

**Answer:**
External dependencies (payment gateways, email providers, third-party APIs) can't be hit in tests — they're slow, cost money, have rate limits, and you can't control their responses.

**Strategies:**

**1. Dependency injection + mocking (unit level):**
- Inject the external client behind an interface; mock it in tests.
```ts
const gateway = { charge: jest.fn().mockResolvedValue({ id: 'ch_1', status: 'succeeded' }) };
const service = new CheckoutService(gateway);
```

**2. HTTP mocking (intercept network calls):**
- Libraries like `nock` (Node) intercept HTTP requests and return canned responses.
```ts
nock('https://api.stripe.com')
  .post('/v1/charges')
  .reply(200, { id: 'ch_1', status: 'succeeded' });
```
- Tests the real HTTP client code (serialization, headers, error handling) without hitting the real API.

**3. Contract testing:**
- Verify your code and the external API agree on the contract.
- **Consumer-driven contracts** (Pact) — you define expected request/response; both sides verify against it.
- Catches "they changed their API" breakage.

**4. Sandbox/test environments:**
- Many providers (Stripe, Twilio) offer test modes with fake credentials.
- Use in integration/E2E tests for realistic behavior without real charges.

**5. Recording (VCR pattern):**
- Record real API responses once, replay them in tests. Libraries: `polly.js`, `nock` with recording.

**Code — testing retry/error handling with nock:**
```ts
it('retries on 503 then succeeds', async () => {
  nock('https://api.provider.com')
    .post('/send').reply(503)              // first call fails
    .post('/send').reply(200, { ok: true }); // retry succeeds
  const result = await sendWithRetry(payload);
  expect(result.ok).toBe(true);
});
```

**Follow-up 1: How do you test that YOUR code handles the external API's failures correctly?**
Mock the failure scenarios: timeouts, 500s, malformed responses, rate-limit 429s. The most important tests aren't "happy path works" but "what happens when the provider is down/slow/wrong?" — because that's where bugs hide. With nock or injected mocks, simulate each failure mode and verify your retry/circuit-breaker/fallback logic responds correctly.

**Follow-up 2: What's the risk of mocking external APIs?**
Your mocks can drift from reality — the real API changes, but your mocks don't, so tests pass while production breaks. Mitigations: contract testing (Pact) to catch contract drift, occasional integration tests against the real sandbox, and monitoring in production. Mocks test "does my code handle this response shape" — they can't verify the shape is still accurate. Combine mocked tests (fast, comprehensive) with a few real-sandbox tests (catch drift).

---

### Q10. What are the most common testing gotchas and anti-patterns?

**Answer:**

1. **Tests with no assertions.** The code runs, nothing is verified. Covers lines, tests nothing. Always assert on outcomes.

2. **Testing implementation, not behavior.** Asserting "method X was called" instead of "the right outcome happened." Breaks on every refactor. Test the contract.

3. **Over-mocking.** Mocking so much you test mocks, not real code. Tests pass, production breaks. Prefer integration tests for interaction-heavy code.

4. **Flaky tests.** Pass sometimes, fail sometimes — timing, ordering, shared state, real network/timers. A flaky suite trains developers to ignore failures. Fix or delete flaky tests; use fake timers, isolate state.

5. **Interdependent tests.** Test B depends on test A running first (shared state). Breaks when run in isolation or parallel. Each test must be independent.

6. **Slow test suites.** Real DB/network in every unit test, no parallelization. A suite taking 30 minutes doesn't get run. Keep unit tests fast; parallelize; reserve slow tests for fewer integration/E2E.

7. **Testing the framework.** Testing that Express routes or TypeORM saves — that's the library's job. Test your logic.

8. **Giant shared fixtures.** One seed file all tests depend on. Change it, break everything. Use factories.

9. **Chasing coverage numbers.** Writing assertion-free tests to hit a target. Coverage finds gaps; it's not a quality measure.

10. **Not testing error paths.** Only happy-path tests. Most production bugs are in error handling. Test failures, edge cases, nulls, timeouts.

11. **Snapshot testing everything.** Huge auto-generated snapshots nobody reviews; developers blindly update them on failure. Use snapshots sparingly for stable, meaningful output.

12. **Mocking what you don't own without contract tests.** Mocks drift from reality. Add contract/integration tests for external dependencies.

13. **No tests for bug fixes.** Bug recurs because nothing guards against it. Always write a regression test for fixed bugs.

14. **Testing private methods directly.** Couples tests to internals. Test through the public interface.

15. **Conditional logic in tests.** `if (x) expect(...)` — tests should be deterministic; conditionals hide untested paths. Each test should have a clear, fixed expectation.

**Follow-up 1: How do you deal with an existing flaky test suite?**
Triage: quarantine flaky tests (mark them, run separately) so they don't block CI while you fix them. Then fix root causes — usually shared state, timing assumptions, real timers/network, or test ordering dependencies. Track flakiness (which tests fail intermittently). A flaky suite erodes trust in all tests; fixing it is high-priority maintenance, not optional.

**Follow-up 2: When should you delete a test?**
When it tests implementation that's been refactored away, duplicates another test, is permanently flaky with no clear fix and low value, or tests trivial framework behavior. A test that costs more to maintain than the bugs it catches is net-negative. Tests are code — they have maintenance cost. Delete dead weight, but never delete a test just because it's failing (fix the bug or the test).

---

*End of section 15. Next: DSA (Backend Flavor) — the final 5 questions.*
