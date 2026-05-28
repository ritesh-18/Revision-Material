# 12 — Design Patterns & LLD (20 Questions)

Your resume calls out SOLID, Factory, Repository, Observer, Strategy, Middleware, and Chain of Responsibility. These cover the principles, the GoF patterns you'll actually use in backend work, and how to approach a low-level-design interview.

**Sections**
- Part A — SOLID & DI (Q1–Q3)
- Part B — Creational & structural patterns (Q4–Q12)
- Part C — Behavioral patterns (Q13–Q16)
- Part D — LLD approach & anti-patterns (Q17–Q20)

---

## Part A — SOLID & Dependency Injection

### Q1. Explain the SOLID principles with backend examples.

**Answer:**

**S — Single Responsibility Principle (SRP).**
A class should have one reason to change. A `UserService` that handles persistence, email sending, and password hashing has three reasons to change; split them.

**O — Open/Closed Principle (OCP).**
Open for extension, closed for modification. You should be able to add behavior without editing existing code. A notification system where adding a new channel (push) means adding a class, not editing a giant switch statement.

**L — Liskov Substitution Principle (LSP).**
Subtypes must be substitutable for their base types without breaking correctness. If `Rectangle.setWidth` and a `Square` subclass breaks the contract (setting width also changes height), you've violated LSP.

**I — Interface Segregation Principle (ISP).**
Clients shouldn't depend on methods they don't use. A fat `Repository` interface with 20 methods forces implementers to stub the ones they don't need. Split into focused interfaces.

**D — Dependency Inversion Principle (DIP).**
Depend on abstractions, not concretions. High-level modules shouldn't import low-level ones directly; both depend on interfaces. Your service depends on a `PaymentGateway` interface, not the concrete `StripeGateway`.

**Code — applying SOLID:**
```ts
// Violates SRP — does too much
class UserService {
  async register(dto) {
    const hash = await bcrypt.hash(dto.password, 12);   // hashing
    const user = await this.db.user.create({ ... });    // persistence
    await sendgrid.send({ ... });                        // email
  }
}

// Follows SRP + DIP — depends on abstractions, each does one thing
class UserService {
  constructor(
    private repo: UserRepository,        // persistence abstraction
    private hasher: PasswordHasher,      // hashing abstraction
    private mailer: Mailer,              // email abstraction
  ) {}

  async register(dto: RegisterDto) {
    const hash = await this.hasher.hash(dto.password);
    const user = await this.repo.create({ ...dto, passwordHash: hash });
    await this.mailer.sendWelcome(user.email);
    return user;
  }
}
```

**Follow-up 1: Aren't SOLID principles sometimes over-applied?**
Yes. SRP taken to extremes creates a class per method; ISP creates dozens of one-method interfaces. The principles are heuristics, not laws. Apply them when they reduce real coupling or improve testability — not dogmatically. "Three similar lines is better than a premature abstraction."

**Follow-up 2: Which SOLID principle matters most in practice?**
Dependency Inversion, because it's what makes code testable and modular. If your services depend on interfaces (injected), you can mock dependencies in tests and swap implementations. The others are refinements; DIP is foundational and is exactly what NestJS's DI container enables.

---

### Q2. Explain Single Responsibility more deeply — how do you know when to split?

**Answer:**
SRP says "one reason to change." The practical question is: what counts as a "reason"?

A useful reframing: a class should serve **one actor / one stakeholder**. If the billing team's requirements and the reporting team's requirements both force changes to the same class, that class has two responsibilities.

**Signs a class violates SRP:**
- It has methods that operate on disjoint sets of fields (one cluster uses fields A, B; another uses C, D).
- Its name contains "and" or is vague ("Manager", "Helper", "Util", "Processor").
- Changing one feature repeatedly causes merge conflicts with unrelated features.
- It's hard to name — because it does many things.
- Tests for it are huge and test unrelated behaviors.

**When NOT to split:**
- The "responsibilities" always change together. Splitting them adds indirection without reducing coupling.
- The class is small and cohesive even if it touches a couple of concerns.

**Code — a class with multiple actors:**
```ts
// Bad — serves three actors
class Employee {
  calculatePay() { ... }       // finance team's rules
  saveToDatabase() { ... }     // DBA's schema
  generateReport() { ... }     // reporting team's format
}

// Better — split by actor
class PayCalculator { calculate(emp: Employee): Money { ... } }
class EmployeeRepository { save(emp: Employee): Promise<void> { ... } }
class EmployeeReportGenerator { generate(emp: Employee): Report { ... } }
```

**Follow-up 1: How does SRP relate to microservice boundaries?**
Same principle, different scale. A microservice should have one reason to change — one bounded context, one team's responsibility. The "distributed monolith" anti-pattern is SRP violated at the service level: services that always change together should be one service.

**Follow-up 2: Doesn't splitting create more files to navigate?**
Yes, and that's a real cost. The trade-off: more files but each is simple and focused, vs fewer files each doing many things. For a large codebase with multiple teams, the focused-files approach wins (parallel work, isolated changes). For a tiny script, one file is fine. Judge by team size and change frequency.

---

### Q3. Dependency Inversion and Dependency Injection — what's the difference, and how does it enable testing?

**Answer:**

**Dependency Inversion (DIP)** is the *principle*: depend on abstractions, not concretions. High-level policy shouldn't depend on low-level details.

**Dependency Injection (DI)** is a *technique* to achieve DIP: instead of a class creating its dependencies, they're provided ("injected") from outside.

**Inversion of Control (IoC)** is the broader concept: the framework controls object creation and wiring, not your code. DI containers (NestJS, Spring) implement IoC.

**Why this enables testing:**
If a class `new`s its dependencies internally, you can't replace them in tests — you're stuck with the real database, real HTTP calls. If dependencies are injected, you pass mocks/stubs in tests.

**Code — the difference:**
```ts
// No DI — untestable, tightly coupled
class OrderService {
  private db = new PostgresClient();              // hardcoded concretion
  private payments = new StripeGateway();         // can't mock in tests

  async placeOrder(dto) {
    await this.payments.charge(dto.amount);       // hits real Stripe in tests!
    await this.db.orders.insert(dto);
  }
}

// With DI + abstractions — testable
interface PaymentGateway { charge(amount: number): Promise<Receipt>; }
interface OrderRepository { insert(order: Order): Promise<Order>; }

class OrderService {
  constructor(
    private payments: PaymentGateway,    // injected abstraction
    private repo: OrderRepository,
  ) {}

  async placeOrder(dto: PlaceOrderDto) {
    const receipt = await this.payments.charge(dto.amount);
    return this.repo.insert({ ...dto, receiptId: receipt.id });
  }
}

// In tests — inject mocks
const fakePayments = { charge: jest.fn().mockResolvedValue({ id: 'r1' }) };
const fakeRepo = { insert: jest.fn().mockImplementation(o => Promise.resolve(o)) };
const service = new OrderService(fakePayments, fakeRepo);
```

**Follow-up 1: Constructor injection vs property injection vs method injection?**
Constructor injection (dependencies as constructor params) is the default — makes dependencies explicit and the object valid on construction. Property injection (set after construction) is for optional dependencies or breaking circular deps. Method injection (pass per-call) for dependencies that vary per operation. Prefer constructor injection; it's the clearest and what NestJS uses by default.

**Follow-up 2: Doesn't DI just move the coupling to the wiring/config?**
Yes — the composition root (where you wire everything) knows the concrete types. But that's one place, isolated and explicit, versus coupling scattered through every class. The DI container (NestJS modules) centralizes this. The benefit: every other class depends only on abstractions and is independently testable.

---

## Part B — Creational & Structural Patterns

### Q4. Explain the Factory pattern. When do you use it?

**Answer:**
The **Factory** pattern centralizes object creation, so callers don't need to know which concrete class to instantiate or how.

**Factory Method** — a method that returns objects of a common interface; subclasses or config decide the concrete type.

**Abstract Factory** — a factory that creates families of related objects.

**When to use:**
- Object creation involves logic (choosing a type based on input).
- You want to decouple callers from concrete classes.
- Creation requires setup that shouldn't be duplicated.

**Code — your notification system's channel factory:**
```ts
interface NotificationChannel {
  send(to: string, message: string): Promise<void>;
}

class EmailChannel implements NotificationChannel { async send(to, msg) { /* SendGrid */ } }
class SmsChannel implements NotificationChannel { async send(to, msg) { /* Twilio */ } }
class PushChannel implements NotificationChannel { async send(to, msg) { /* FCM */ } }

class NotificationChannelFactory {
  constructor(
    private email: EmailChannel,
    private sms: SmsChannel,
    private push: PushChannel,
  ) {}

  create(type: 'email' | 'sms' | 'push'): NotificationChannel {
    switch (type) {
      case 'email': return this.email;
      case 'sms':   return this.sms;
      case 'push':  return this.push;
      default: throw new Error(`unknown channel: ${type}`);
    }
  }
}

// Usage — caller doesn't know concrete classes
const channel = factory.create(notification.channel);
await channel.send(notification.to, notification.body);
```

This is exactly the "Factory (per-tenant service instances)" and "notification channel" pattern from your SaaS project.

**Follow-up 1: How does this relate to the Open/Closed Principle?**
Adding a new channel (WhatsApp) means adding a class + a case, not rewriting the callers. Better still, register channels in a map so the factory doesn't even need a switch — fully open for extension. The factory localizes the "which type?" decision so the rest of the code stays closed to modification.

**Follow-up 2: Isn't this just a glorified switch statement?**
For simple cases, yes — and that's fine. The value is centralization: the switch lives in one place, callers are decoupled, and you can evolve creation logic (add caching, pooling, config-driven selection) without touching callers. When creation is trivial (`new Foo()`), skip the factory — it's only worth it when creation has logic or you need decoupling.

---

### Q5. Explain the Strategy pattern.

**Answer:**
The **Strategy** pattern defines a family of interchangeable algorithms behind a common interface, letting you swap behavior at runtime.

**When to use:**
- Multiple ways to do the same thing (different algorithms, policies).
- You want to choose the behavior at runtime (config, user choice).
- Avoid large conditionals selecting behavior.

**Code — your tenant resolution + notification channel strategies:**
```ts
interface TenantResolutionStrategy {
  resolve(req: Request): string;
}

class SubdomainStrategy implements TenantResolutionStrategy {
  resolve(req) { return req.hostname.split('.')[0]; }
}
class HeaderStrategy implements TenantResolutionStrategy {
  resolve(req) { return req.headers['x-tenant-id'] as string; }
}
class JwtClaimStrategy implements TenantResolutionStrategy {
  resolve(req) { return req.user?.tenantId; }
}

class TenantResolver {
  constructor(private strategy: TenantResolutionStrategy) {}
  setStrategy(s: TenantResolutionStrategy) { this.strategy = s; }
  resolve(req: Request) { return this.strategy.resolve(req); }
}

// Configure at startup based on deployment mode
const resolver = new TenantResolver(
  config.tenantMode === 'subdomain' ? new SubdomainStrategy() : new HeaderStrategy()
);
```

Another classic: pricing strategies (regular, member discount, bulk discount), shipping cost calculators, retry strategies.

**Follow-up 1: Strategy vs Factory — what's the distinction?**
Factory is about *creating* objects; Strategy is about *swapping behavior*. They often combine: a factory creates the right strategy. Factory answers "which object?"; Strategy answers "which algorithm?". The strategy object is often created by a factory.

**Follow-up 2: How does Strategy relate to first-class functions?**
In JavaScript, you often don't need the full class-based Strategy pattern — a function passed as an argument IS a strategy. `array.sort(comparator)` is Strategy: the comparator is the swappable algorithm. The OOP pattern matters when strategies have state or multiple methods; for single-function strategies, just pass a function.

---

### Q6. Explain the Observer pattern.

**Answer:**
The **Observer** pattern lets one object (subject) notify many dependents (observers) of state changes, without tight coupling between them.

**When to use:**
- One change should trigger reactions in multiple places.
- The set of reactors is dynamic or unknown to the subject.
- Event-driven designs.

**Code — Node's EventEmitter is the Observer pattern:**
```ts
import { EventEmitter } from 'events';

class OrderService extends EventEmitter {
  async placeOrder(dto: PlaceOrderDto) {
    const order = await this.repo.create(dto);
    this.emit('order.placed', order);    // notify all observers
    return order;
  }
}

const orders = new OrderService();
// Observers subscribe independently
orders.on('order.placed', o => emailService.sendConfirmation(o));
orders.on('order.placed', o => inventoryService.reserve(o));
orders.on('order.placed', o => analyticsService.track(o));
```

The `OrderService` doesn't know or care who's listening. Add a new reaction (loyalty points) by subscribing — no change to `OrderService`.

**At scale**, the Observer pattern generalizes into event-driven architecture: Kafka/RabbitMQ are distributed observers. The publisher emits an event; consumers (observers) react independently. Your SaaS notification fan-out is Observer at the infrastructure level.

**Follow-up 1: What's the downside of Observer?**
Implicit flow. When you emit an event, it's not obvious from the code what happens next — observers are registered elsewhere. Debugging "why did X happen?" requires finding all listeners. Mitigations: naming conventions, central event registry, good logging. The decoupling that makes Observer powerful also makes flow harder to trace.

**Follow-up 2: Synchronous vs asynchronous observers?**
Node's EventEmitter calls listeners synchronously by default — a slow listener blocks the emit. For async work, listeners should be fire-and-forget (`.on('event', async () => {...})` — but unhandled rejections are a risk) or you push to a queue. For cross-service observation, you're inherently async (events through a broker). Be deliberate about whether the subject waits for observers.

---

### Q7. Explain the Repository pattern. Why use it?

**Answer:**
The **Repository** pattern abstracts data access behind a collection-like interface. Business logic talks to the repository ("find user by email", "save order") without knowing the underlying storage (Postgres, Mongo, in-memory).

**Benefits:**
- **Decouples business logic from persistence.** Switch DBs or ORMs without touching domain code.
- **Testability.** Mock the repository in unit tests; no real DB needed.
- **Centralizes query logic.** All "find active users" queries live in one place.
- **Domain-oriented interface.** `findByEmail` reads better than scattered ORM calls.

**Code:**
```ts
// Domain-facing interface
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<void>;
}

// Concrete implementation — could be swapped
@Injectable()
class PostgresUserRepository implements UserRepository {
  constructor(@InjectRepository(UserEntity) private repo: Repository<UserEntity>) {}

  findByEmail(email: string) {
    return this.repo.findOne({ where: { email: email.toLowerCase() } });
  }
  save(user: User) { return this.repo.save(user); }
  // ...
}

// Service depends on the interface
class UserService {
  constructor(private users: UserRepository) {}   // abstraction, not the concrete
  async getByEmail(email: string) {
    const user = await this.users.findByEmail(email);
    if (!user) throw new NotFoundException();
    return user;
  }
}
```

**Follow-up 1: Isn't the ORM already a repository? Why add another layer?**
TypeORM and Prisma do provide repository-like APIs. The question is whether to expose them directly or wrap them. Wrapping adds value when: (1) you want domain-meaningful methods (`findActiveSubscribers` not `find({where})`), (2) you want to keep ORM types out of your domain layer, (3) you anticipate switching ORMs. For simple CRUD apps, using the ORM repository directly is fine — don't add a wrapper that just forwards calls.

**Follow-up 2: How does the Repository pattern relate to the Unit of Work pattern?**
Unit of Work tracks changes to multiple entities and commits them in one transaction. Repository handles single-entity-type access; Unit of Work coordinates across repositories. Example: `unitOfWork.run(async () => { userRepo.save(u); orderRepo.save(o); })` commits both atomically. TypeORM's `EntityManager.transaction` and Prisma's `$transaction` are Unit of Work implementations.

---

### Q8. Explain the Singleton pattern. What are its problems?

**Answer:**
The **Singleton** ensures a class has only one instance and provides global access to it.

**When legitimately useful:**
- A single shared resource: DB connection pool, config object, logger, cache client.
- In Node, modules are effectively singletons (the module cache returns the same export).

**Code — Singleton in Node (module-based):**
```ts
// db.ts — module caching makes this a singleton
let pool: Pool;
export function getPool(): Pool {
  if (!pool) pool = new Pool({ connectionString: process.env.DATABASE_URL });
  return pool;
}
// Every importer gets the same pool.
```

**In NestJS**, providers are singletons by default — the DI container ensures one instance per app. This is the *good* way to do singletons: managed by the container, injectable, testable.

**Problems with the classic Singleton (global access):**
1. **Hidden dependencies.** A class that reaches for `Database.getInstance()` doesn't declare its dependency — you can't tell from its signature.
2. **Hard to test.** Global state persists across tests; you can't easily inject a mock.
3. **Tight coupling.** Callers depend on the concrete singleton class, not an abstraction.
4. **Concurrency issues** in multi-threaded languages (less so in Node's single thread).

**The DI alternative:** instead of a global Singleton accessed via static method, register a single instance in the DI container and inject it. Same "one instance" benefit, but explicit, testable, swappable.

**Follow-up 1: When is the global-access Singleton actually OK?**
For genuinely process-global, stateless-ish utilities: a logger, a metrics client. Even then, DI is usually cleaner. The Singleton anti-pattern reputation comes from people using it as a backdoor for global mutable state, which creates the testing and coupling problems.

**Follow-up 2: How do you test code that uses a module-level singleton?**
Hard — that's the point of the criticism. Options: (1) dependency injection so you can pass a fake; (2) `jest.mock('./db')` to replace the module; (3) a reset function the singleton exposes for tests. The cleanest is to not use module-global singletons for things you need to mock — inject them instead.

---

### Q9. Explain the Chain of Responsibility pattern.

**Answer:**
**Chain of Responsibility** passes a request along a chain of handlers. Each handler decides to process it, pass it on, or both. The sender doesn't know which handler will handle it.

**When to use:**
- Multiple handlers might process a request, decided at runtime.
- You want to decouple sender from receivers.
- Processing is a pipeline of steps, each able to stop the chain.

**Code — your permission guards as a Chain of Responsibility:**
```ts
abstract class PermissionGuard {
  private next?: PermissionGuard;
  setNext(guard: PermissionGuard): PermissionGuard {
    this.next = guard;
    return guard;   // allow chaining
  }
  async check(ctx: RequestContext): Promise<boolean> {
    if (!await this.handle(ctx)) return false;
    return this.next ? this.next.check(ctx) : true;
  }
  protected abstract handle(ctx: RequestContext): Promise<boolean>;
}

class AuthenticatedGuard extends PermissionGuard {
  protected async handle(ctx) { return !!ctx.user; }
}
class TenantMembershipGuard extends PermissionGuard {
  protected async handle(ctx) { return ctx.user.tenants.includes(ctx.tenantId); }
}
class RoleGuard extends PermissionGuard {
  protected async handle(ctx) { return ctx.user.roles.includes(ctx.requiredRole); }
}

// Build the chain
const auth = new AuthenticatedGuard();
auth.setNext(new TenantMembershipGuard()).setNext(new RoleGuard());

const allowed = await auth.check(context);   // runs all in sequence, stops on first false
```

This is exactly the "Chain of Responsibility (permission guards)" from your SaaS project, and it's also how Express middleware works (each `next()` passes to the next handler).

**Follow-up 1: How is this different from a simple list of checks?**
For simple cases, a list/array of check functions you iterate is fine — and often clearer. The full CoR pattern (linked handlers) adds value when handlers need to be dynamically composed, when each handler decides whether to continue, or when handlers are objects with their own state. Express/NestJS middleware is CoR with framework-managed chaining.

**Follow-up 2: What's the relationship to middleware?**
Middleware IS Chain of Responsibility. Each Express middleware `(req, res, next)` is a handler; calling `next()` passes to the next handler; not calling it stops the chain. NestJS guards, interceptors, and pipes are all variations. Understanding CoR helps you reason about middleware ordering and short-circuiting.

---

### Q10. Explain the Decorator pattern.

**Answer:**
The **Decorator** pattern wraps an object to add behavior without modifying the original class. Decorators implement the same interface as what they wrap, so they're transparent to callers.

**When to use:**
- Add responsibilities to objects dynamically.
- Avoid a subclass explosion (every combination of features as a subclass).
- Layer cross-cutting concerns (caching, logging, retry) around a core implementation.

**Code — decorating a repository with caching and logging:**
```ts
interface UserRepository {
  findById(id: string): Promise<User | null>;
}

class PostgresUserRepository implements UserRepository {
  async findById(id) { return this.db.users.findOne(id); }
}

// Caching decorator — same interface, adds caching
class CachedUserRepository implements UserRepository {
  constructor(private inner: UserRepository, private cache: Redis) {}
  async findById(id: string) {
    const cached = await this.cache.get(`user:${id}`);
    if (cached) return JSON.parse(cached);
    const user = await this.inner.findById(id);     // delegate
    if (user) await this.cache.setex(`user:${id}`, 300, JSON.stringify(user));
    return user;
  }
}

// Logging decorator
class LoggingUserRepository implements UserRepository {
  constructor(private inner: UserRepository, private log: Logger) {}
  async findById(id: string) {
    const start = Date.now();
    const result = await this.inner.findById(id);
    this.log.debug(`findById(${id}) took ${Date.now() - start}ms`);
    return result;
  }
}

// Compose — layers of behavior
const repo = new LoggingUserRepository(
  new CachedUserRepository(
    new PostgresUserRepository(db),
    redis,
  ),
  logger,
);
```

Each layer adds one concern. Order matters: logging-outside-caching logs total time including cache hits; caching-outside-logging logs only DB-hit time.

**Follow-up 1: How does this relate to TypeScript/NestJS decorators (`@Injectable`)?**
Confusingly, they share a name but differ. The Decorator *pattern* (above) is structural — wrapping objects. TypeScript *decorators* (`@Get`, `@Injectable`) are a language feature for metadata/annotation. Some TS decorators implement the pattern (method decorators that wrap functions), but `@Controller` just stamps metadata — not the structural pattern.

**Follow-up 2: Decorator vs inheritance for adding behavior?**
Inheritance is static (decided at compile time) and creates rigid hierarchies. Decorators compose at runtime — you can stack caching + logging + retry in any order without N×M subclasses. "Favor composition over inheritance" — the Decorator pattern is composition applied to adding behavior.

---

### Q11. Explain the Adapter pattern.

**Answer:**
The **Adapter** pattern converts one interface into another that clients expect. It's the "glue" between incompatible interfaces — wrapping a third-party API to match your domain's expected interface.

**When to use:**
- Integrating a third-party library whose interface doesn't match yours.
- Wrapping a legacy component to work with new code.
- Creating a consistent interface over multiple providers (payment gateways, SMS providers).

**Code — adapting multiple payment providers to one interface:**
```ts
// Your domain's expected interface
interface PaymentGateway {
  charge(amount: number, currency: string, source: string): Promise<{ id: string; status: string }>;
}

// Stripe's SDK has a different shape — adapt it
class StripeAdapter implements PaymentGateway {
  constructor(private stripe: Stripe) {}
  async charge(amount: number, currency: string, source: string) {
    const intent = await this.stripe.paymentIntents.create({
      amount, currency, payment_method: source, confirm: true,
    });
    return { id: intent.id, status: intent.status };    // map to your shape
  }
}

// Razorpay has yet another shape — adapt it too
class RazorpayAdapter implements PaymentGateway {
  constructor(private razorpay: Razorpay) {}
  async charge(amount: number, currency: string, source: string) {
    const payment = await this.razorpay.payments.create({
      amount: amount, currency, method: source,
    });
    return { id: payment.id, status: payment.status };
  }
}

// Your code depends only on PaymentGateway — providers are swappable
class CheckoutService {
  constructor(private gateway: PaymentGateway) {}
  async checkout(order) { return this.gateway.charge(order.total, 'usd', order.source); }
}
```

This is how your Dhee project integrated "payment gateway, SMS, email" providers behind consistent interfaces with retry/idempotency.

**Follow-up 1: Adapter vs Facade — what's the difference?**
Adapter converts one interface to *match an expected interface* (you don't control the target shape; you adapt to it). Facade *simplifies* a complex subsystem behind a cleaner interface (you design the simpler interface). Adapter is about compatibility; Facade is about simplification.

**Follow-up 2: How does this help with vendor lock-in?**
By depending on your own interface, swapping vendors is just writing a new adapter — no changes to business logic. You used Stripe; you want to add Razorpay for India — write `RazorpayAdapter`, register it for Indian tenants. The CheckoutService never knows. This is the practical payoff of the Adapter + DIP combination.

---

### Q12. Explain the Builder and Facade patterns.

**Answer:**

**Builder** — constructs complex objects step by step, separating construction from representation. Useful when an object has many optional parameters or requires multi-step setup.

```ts
class QueryBuilder {
  private parts = { select: '*', from: '', where: [] as string[], orderBy: '', limit: 0 };

  select(cols: string) { this.parts.select = cols; return this; }
  from(table: string) { this.parts.from = table; return this; }
  where(cond: string) { this.parts.where.push(cond); return this; }
  orderBy(col: string) { this.parts.orderBy = col; return this; }
  limit(n: number) { this.parts.limit = n; return this; }

  build(): string {
    let q = `SELECT ${this.parts.select} FROM ${this.parts.from}`;
    if (this.parts.where.length) q += ` WHERE ${this.parts.where.join(' AND ')}`;
    if (this.parts.orderBy) q += ` ORDER BY ${this.parts.orderBy}`;
    if (this.parts.limit) q += ` LIMIT ${this.parts.limit}`;
    return q;
  }
}

const sql = new QueryBuilder()
  .select('id, name').from('users').where('active = true').orderBy('name').limit(10)
  .build();
```

Knex, TypeORM's query builder, and Prisma's fluent API are all Builders. They avoid telescoping constructors (`new Query(a, b, c, d, e, f)`) and make optional parameters readable.

**Facade** — provides a simplified interface to a complex subsystem.

```ts
// Complex subsystem
class OnboardingFacade {
  constructor(
    private schemas: SchemaService,
    private migrations: MigrationRunner,
    private keycloak: KeycloakService,
    private seeder: DataSeeder,
    private mailer: Mailer,
  ) {}

  // One simple method hides the complex multi-step process
  async onboardTenant(input: CreateTenantInput): Promise<Tenant> {
    const schema = await this.schemas.create(input.name);
    await this.migrations.run(schema);
    await this.keycloak.createClient(input.name);
    await this.seeder.seedDefaults(schema, input.plan);
    await this.mailer.sendWelcome(input.adminEmail);
    return { id: schema, name: input.name };
  }
}

// Caller just does this
await onboarding.onboardTenant({ name: 'acme', plan: 'pro', adminEmail: 'a@acme.com' });
```

The caller doesn't deal with schemas, migrations, Keycloak, seeding — the Facade hides that complexity.

**Follow-up 1: When does the Builder pattern become overkill?**
For objects with few parameters or no optional fields, a plain constructor or object literal is clearer. Builders shine when there are many optional parameters, validation during construction, or fluent step-by-step assembly. In JS, an options object (`new Foo({ a, b, c })`) often replaces the Builder pattern with less ceremony.

**Follow-up 2: Isn't a Facade just a service?**
Often, yes — a well-designed service that coordinates other services IS a Facade. The pattern name describes intent: "simplify a complex subsystem behind one interface." Your `OnboardingService` is a Facade. The label matters less than the principle: hide complexity, expose a clean interface.

---

## Part C — Behavioral Patterns

### Q13. Explain the Command pattern.

**Answer:**
The **Command** pattern encapsulates a request as an object, letting you parameterize, queue, log, and undo operations.

**When to use:**
- Queue operations for later execution (job queues).
- Support undo/redo.
- Log operations for audit/replay.
- Decouple the invoker from the operation.

**Code — commands in a job queue (ties to your BullMQ work):**
```ts
interface Command {
  execute(): Promise<void>;
  undo?(): Promise<void>;
}

class SendEmailCommand implements Command {
  constructor(private mailer: Mailer, private to: string, private body: string) {}
  async execute() { await this.mailer.send(this.to, this.body); }
}

class ChargeCardCommand implements Command {
  constructor(private gateway: PaymentGateway, private amount: number, private card: string) {}
  private chargeId?: string;
  async execute() { this.chargeId = (await this.gateway.charge(this.amount, this.card)).id; }
  async undo() { if (this.chargeId) await this.gateway.refund(this.chargeId); }
}

// Invoker — doesn't know what the command does
class CommandBus {
  private history: Command[] = [];
  async run(cmd: Command) {
    await cmd.execute();
    this.history.push(cmd);
  }
  async undoLast() {
    const cmd = this.history.pop();
    if (cmd?.undo) await cmd.undo();
  }
}
```

NestJS's CQRS module (`@nestjs/cqrs`) is built on the Command pattern — `CreateOrderCommand`, dispatched to a `CreateOrderHandler`.

**Follow-up 1: How does Command enable the saga pattern?**
A saga is a sequence of commands with compensating commands (undos). The `undo()` method on each command is the compensation. If step 3 fails, you call `undo()` on steps 2 and 1 in reverse. The Command pattern's encapsulation of "the operation and its reversal" maps directly to saga steps and compensations (section 07 Q7).

**Follow-up 2: When is Command overkill?**
For straightforward synchronous calls where you don't need queuing, undo, logging, or decoupling — just call the method. Command adds indirection (a class per operation). Use it when the operations need to be first-class objects: queued, logged, replayed, undone. CQRS frameworks make it ergonomic; hand-rolling Command for simple CRUD is over-engineering.

---

### Q14. Explain the Proxy pattern.

**Answer:**
The **Proxy** pattern provides a placeholder/surrogate for another object to control access to it. The proxy implements the same interface and forwards calls, adding behavior around them.

**Types:**
- **Virtual proxy** — lazy initialization (create the expensive object only when first used).
- **Protection proxy** — access control (check permissions before forwarding).
- **Remote proxy** — represents an object in another process/machine (RPC stubs).
- **Caching proxy** — cache results (overlaps with Decorator).

**Code — protection proxy for a sensitive service:**
```ts
interface DocumentService {
  read(docId: string): Promise<Document>;
  delete(docId: string): Promise<void>;
}

class RealDocumentService implements DocumentService {
  async read(docId) { return this.db.docs.findOne(docId); }
  async delete(docId) { return this.db.docs.delete(docId); }
}

class ProtectedDocumentService implements DocumentService {
  constructor(private real: DocumentService, private user: User) {}
  async read(docId: string) {
    if (!this.user.can('read', docId)) throw new ForbiddenException();
    return this.real.read(docId);
  }
  async delete(docId: string) {
    if (!this.user.can('delete', docId)) throw new ForbiddenException();
    return this.real.delete(docId);
  }
}
```

JavaScript has a built-in `Proxy` object for intercepting operations (get, set, has) — used by Vue's reactivity, MobX, and ORMs for lazy loading.

```js
const lazyDb = new Proxy({}, {
  get(target, prop) {
    if (!target.connection) target.connection = expensiveConnect();
    return target.connection[prop];
  }
});
```

**Follow-up 1: Proxy vs Decorator — they look similar?**
Both wrap an object with the same interface. The intent differs: Decorator *adds* behavior/responsibilities; Proxy *controls access* (lazy loading, permission, remoting). A caching proxy and a caching decorator can look identical — the distinction is whether you're enhancing (decorator) or gating/managing (proxy). The line is fuzzy; intent is the differentiator.

**Follow-up 2: How do ORMs use proxies for lazy loading?**
When you fetch an entity with relations not eagerly loaded, the ORM returns a proxy for the relation. Accessing it triggers the actual DB query (virtual proxy). TypeORM and Hibernate do this. Gotcha: lazy loading inside a loop becomes N+1; accessing a proxy outside the session/transaction throws. Know when relations are real vs proxies.

---

### Q15. Composition over inheritance — what does it mean and why prefer it?

**Answer:**
**Inheritance** ("is-a") establishes a rigid hierarchy: a `Dog` is an `Animal`. Subclasses inherit and override behavior.

**Composition** ("has-a") builds objects from smaller parts: a `Car` has an `Engine`, has `Wheels`.

**"Favor composition over inheritance"** because:

1. **Inheritance is rigid.** Deep hierarchies are hard to change — modifying a base class ripples to all subclasses. Composition lets you swap parts independently.

2. **The fragile base class problem.** A change to the base class can break subclasses in non-obvious ways.

3. **Single inheritance limits.** Most languages allow inheriting from one class; you can compose many behaviors.

4. **Inheritance exposes implementation.** Subclasses depend on the base class's internals, breaking encapsulation.

5. **The combinatorial explosion.** Need flying + swimming + walking animals? With inheritance you get `FlyingSwimmingAnimal`, `FlyingWalkingAnimal`, etc. With composition, you compose behaviors.

**Code — the classic example:**
```ts
// Inheritance gone wrong
class Animal { move() {} }
class FlyingAnimal extends Animal { fly() {} }
class SwimmingAnimal extends Animal { swim() {} }
// A duck flies AND swims — which to extend? Multiple inheritance problem.

// Composition — behaviors as components
interface MoveBehavior { move(): void; }
class Fly implements MoveBehavior { move() { console.log('flying'); } }
class Swim implements MoveBehavior { move() { console.log('swimming'); } }

class Duck {
  constructor(private behaviors: MoveBehavior[]) {}
  move() { this.behaviors.forEach(b => b.move()); }
}
const duck = new Duck([new Fly(), new Swim()]);   // duck both flies and swims
```

This is also the **Strategy pattern** — composition is how Strategy works.

**Follow-up 1: When IS inheritance the right choice?**
For genuine "is-a" relationships with stable hierarchies and shared behavior that truly belongs to all subtypes. Framework base classes (extending `Error`, extending a NestJS base controller) are legitimate. The warning is against using inheritance for code reuse when the "is-a" relationship doesn't hold — that's where composition wins.

**Follow-up 2: How does this relate to React/frontend?**
React explicitly recommends composition over inheritance for components — pass children, use higher-order components or hooks, not class inheritance. Same principle: compose UI from smaller pieces rather than building inheritance hierarchies of components. The lesson generalizes across backend and frontend.

---

### Q16. Template Method vs Strategy — what's the difference?

**Answer:**

**Template Method** defines the skeleton of an algorithm in a base class, deferring some steps to subclasses. The overall structure is fixed; subclasses fill in specific steps.

**Strategy** encapsulates entire interchangeable algorithms behind an interface; the whole algorithm is swappable.

**Key difference:**
- Template Method uses **inheritance** — subclasses override hook methods within a fixed algorithm.
- Strategy uses **composition** — you inject a whole algorithm object.

**Code — Template Method:**
```ts
abstract class DataImporter {
  // The template — fixed structure
  async import(file: string): Promise<void> {
    const raw = await this.read(file);       // step varies
    const parsed = this.parse(raw);          // step varies
    const validated = this.validate(parsed); // shared
    await this.save(validated);              // step varies
  }
  protected validate(data: any[]) {          // shared default
    return data.filter(row => row.id != null);
  }
  protected abstract read(file: string): Promise<string>;
  protected abstract parse(raw: string): any[];
  protected abstract save(data: any[]): Promise<void>;
}

class CsvImporter extends DataImporter {
  protected async read(file) { return fs.readFile(file, 'utf8'); }
  protected parse(raw) { return parseCsv(raw); }
  protected async save(data) { await db.bulkInsert(data); }
}
```

**Code — Strategy (same goal, composition):**
```ts
interface ParseStrategy { parse(raw: string): any[]; }
class CsvParse implements ParseStrategy { parse(raw) { return parseCsv(raw); } }
class JsonParse implements ParseStrategy { parse(raw) { return JSON.parse(raw); } }

class DataImporter {
  constructor(private parser: ParseStrategy) {}   // inject the strategy
  async import(file: string) {
    const raw = await fs.readFile(file, 'utf8');
    const parsed = this.parser.parse(raw);         // delegate to strategy
    await db.bulkInsert(parsed);
  }
}
```

**When each:**
- Template Method when the algorithm structure is fixed and only steps vary, and the variation is closed (known subtypes).
- Strategy when you want to swap entire algorithms at runtime, or favor composition for flexibility.

**Follow-up 1: Why might you prefer Strategy over Template Method?**
Composition over inheritance (Q15). Strategy lets you change behavior at runtime (`importer.setParser(jsonParse)`), reuse strategies across contexts, and test strategies in isolation. Template Method locks you into the inheritance hierarchy. For most modern code, Strategy is the more flexible choice.

**Follow-up 2: Can you combine them?**
Yes. A Template Method might use Strategy objects for its variable steps: the template defines the flow, but each step delegates to an injected strategy rather than an overridden method. This combines a fixed skeleton with runtime-swappable steps — best of both for complex pipelines.

---

## Part D — LLD Approach & Anti-patterns

### Q17. How do you approach a low-level design (LLD) interview problem?

**Answer:**
LLD interviews ask you to design the classes, interfaces, and interactions for a system (parking lot, elevator, vending machine, rate limiter, notification service). A structured approach:

**1. Clarify requirements (5 min).**
- Functional: what must it do? ("Park cars, find spots, calculate fees.")
- Non-functional: scale, concurrency, extensibility.
- Constraints: explicit assumptions ("assume single location for now").
- Don't start coding before you understand the scope.

**2. Identify core entities (nouns).**
- Parking lot → `ParkingLot`, `ParkingSpot`, `Vehicle`, `Ticket`, `Payment`.
- These become your classes.

**3. Define relationships and responsibilities.**
- Who owns what? `ParkingLot` has many `ParkingSpot`s.
- What's each class responsible for? (SRP.)

**4. Define interfaces and abstractions.**
- Where will requirements vary? Those are your interfaces.
- `PaymentStrategy`, `SpotAllocationStrategy`, `Vehicle` (base for Car/Truck/Motorcycle).

**5. Apply patterns where they fit naturally.**
- Strategy for pricing/allocation.
- Factory for creating vehicles/spots.
- Observer for notifications ("spot freed").
- Don't force patterns; use them where they reduce coupling.

**6. Handle edge cases and concurrency.**
- Two cars competing for the last spot — locking, atomic allocation.
- Full lot, invalid ticket, payment failure.

**7. Write key classes** (not everything — focus on the interesting parts).

**Communication tips:**
- Think out loud; the interviewer evaluates your reasoning, not just the final design.
- State assumptions explicitly.
- Start simple, then extend ("here's the basic version; to support multiple floors, I'd...").
- Discuss trade-offs.

**Follow-up 1: How much code should you actually write?**
Enough to demonstrate the design — key classes, important methods, interfaces. Not every getter/setter. The interviewer wants to see your modeling, your use of abstractions, your handling of the interesting logic (allocation, pricing, concurrency). Pseudocode for the boring parts is fine.

**Follow-up 2: What separates a strong LLD answer from a weak one?**
Strong: clear separation of concerns, appropriate abstractions (interfaces where things vary), thoughtful handling of edge cases and concurrency, extensibility ("adding a new vehicle type is easy"), and clear communication of trade-offs. Weak: one giant class, hardcoded logic with big switch statements, no abstractions, ignoring concurrency, forcing patterns that don't fit.

---

### Q18. Design a parking lot system (LLD walkthrough).

**Answer:**

**Requirements:**
- Multiple spot types (compact, large, handicapped, motorcycle).
- Vehicles get assigned a compatible spot.
- Track entry/exit time; calculate fees.
- Handle full-lot, multiple floors.

**Core entities and design:**
```ts
enum SpotType { MOTORCYCLE, COMPACT, LARGE, HANDICAPPED }
enum VehicleType { MOTORCYCLE, CAR, TRUCK }

abstract class Vehicle {
  constructor(public licensePlate: string, public type: VehicleType) {}
  abstract canFitIn(spot: SpotType): boolean;
}

class Car extends Vehicle {
  constructor(plate: string) { super(plate, VehicleType.CAR); }
  canFitIn(spot: SpotType) { return spot === SpotType.COMPACT || spot === SpotType.LARGE; }
}

class ParkingSpot {
  private vehicle?: Vehicle;
  constructor(public id: string, public type: SpotType, public floor: number) {}
  isAvailable() { return !this.vehicle; }
  assign(v: Vehicle) { this.vehicle = v; }
  free() { this.vehicle = undefined; }
}

// Strategy for spot allocation — swappable policy
interface AllocationStrategy {
  findSpot(spots: ParkingSpot[], vehicle: Vehicle): ParkingSpot | null;
}
class NearestFirstStrategy implements AllocationStrategy {
  findSpot(spots, vehicle) {
    return spots.find(s => s.isAvailable() && vehicle.canFitIn(s.type)) ?? null;
  }
}

class Ticket {
  constructor(
    public id: string,
    public spot: ParkingSpot,
    public vehicle: Vehicle,
    public entryTime: Date,
  ) {}
  public exitTime?: Date;
}

// Strategy for pricing
interface PricingStrategy { calculate(ticket: Ticket): number; }
class HourlyPricing implements PricingStrategy {
  constructor(private ratePerHour: number) {}
  calculate(ticket: Ticket): number {
    const hours = Math.ceil((ticket.exitTime!.getTime() - ticket.entryTime.getTime()) / 3.6e6);
    return hours * this.ratePerHour;
  }
}

class ParkingLot {
  private spots: ParkingSpot[] = [];
  private activeTickets = new Map<string, Ticket>();
  private lock = new Mutex();   // concurrency for the last-spot race

  constructor(
    private allocation: AllocationStrategy,
    private pricing: PricingStrategy,
  ) {}

  async park(vehicle: Vehicle): Promise<Ticket> {
    return this.lock.runExclusive(() => {           // atomic spot allocation
      const spot = this.allocation.findSpot(this.spots, vehicle);
      if (!spot) throw new Error('Parking full');
      spot.assign(vehicle);
      const ticket = new Ticket(uuid(), spot, vehicle, new Date());
      this.activeTickets.set(ticket.id, ticket);
      return ticket;
    });
  }

  async exit(ticketId: string): Promise<number> {
    const ticket = this.activeTickets.get(ticketId);
    if (!ticket) throw new Error('Invalid ticket');
    ticket.exitTime = new Date();
    const fee = this.pricing.calculate(ticket);
    ticket.spot.free();
    this.activeTickets.delete(ticketId);
    return fee;
  }
}
```

**Design choices to mention:**
- **Strategy** for allocation and pricing — easy to add "premium pricing" or "spread-out allocation."
- **Vehicle hierarchy** with `canFitIn` — adding electric vehicles (need charging spots) extends cleanly.
- **Mutex** for the last-spot concurrency race.
- **Extensibility**: multiple floors (spot has a floor), reservations, EV charging — all additive.

**Follow-up 1: How would you handle concurrency in a distributed (multi-server) version?**
The in-process Mutex doesn't work across servers. Move spot allocation to an atomic DB operation: `UPDATE spots SET vehicle_id = $1 WHERE id = (SELECT id FROM spots WHERE available AND fits LIMIT 1 FOR UPDATE SKIP LOCKED) RETURNING id`. Or use Redis with a Lua script for atomic claim. Section 03 Q13's `SKIP LOCKED` pattern fits perfectly.

**Follow-up 2: How would you support reservations?**
Add a `Reservation` entity with a time window and a held spot. The allocation strategy must skip reserved spots during their window. This introduces a scheduling dimension — now spots have a timeline of availability, not just a boolean. Trade-off between flexibility and complexity; clarify whether reservations are in scope before adding.

---

### Q19. Design a notification system (LLD, ties to your SaaS project).

**Answer:**

**Requirements:**
- Multiple channels (email, SMS, push).
- Multiple providers per channel (SendGrid/SES for email).
- Retry on failure, dead-letter on permanent failure.
- User preferences (which channels, quiet hours).
- Templates.

**Design — applying Strategy, Factory, Adapter, Chain:**
```ts
// Channel abstraction (Strategy)
interface NotificationChannel {
  send(recipient: string, content: RenderedContent): Promise<DeliveryResult>;
}

// Provider adapters behind each channel (Adapter)
class SendGridEmailChannel implements NotificationChannel {
  async send(to, content) { /* SendGrid API → map to DeliveryResult */ }
}
class TwilioSmsChannel implements NotificationChannel {
  async send(to, content) { /* Twilio API */ }
}
class FcmPushChannel implements NotificationChannel {
  async send(to, content) { /* FCM */ }
}

// Factory for channels
class ChannelFactory {
  private channels = new Map<string, NotificationChannel>();
  register(type: string, ch: NotificationChannel) { this.channels.set(type, ch); }
  get(type: string): NotificationChannel {
    const ch = this.channels.get(type);
    if (!ch) throw new Error(`no channel: ${type}`);
    return ch;
  }
}

// Template rendering (Strategy)
interface TemplateRenderer { render(templateId: string, data: object): RenderedContent; }

// Preference filtering (Chain of Responsibility)
abstract class PreferenceFilter {
  next?: PreferenceFilter;
  setNext(f: PreferenceFilter) { this.next = f; return f; }
  check(n: Notification): boolean {
    if (!this.allow(n)) return false;
    return this.next ? this.next.check(n) : true;
  }
  abstract allow(n: Notification): boolean;
}
class ChannelOptInFilter extends PreferenceFilter {
  allow(n) { return n.user.optedInChannels.includes(n.channel); }
}
class QuietHoursFilter extends PreferenceFilter {
  allow(n) { return !isWithinQuietHours(n.user.timezone); }
}

// Orchestrator (Facade)
class NotificationService {
  constructor(
    private factory: ChannelFactory,
    private renderer: TemplateRenderer,
    private filters: PreferenceFilter,
    private queue: JobQueue,
  ) {}

  async notify(n: Notification): Promise<void> {
    if (!this.filters.check(n)) return;             // preferences
    // Enqueue for async delivery with retry (BullMQ / Kafka)
    await this.queue.add('send-notification', n, {
      attempts: 5,
      backoff: { type: 'exponential', delay: 1000 },
    });
  }

  // Worker side
  async deliver(n: Notification): Promise<void> {
    const content = this.renderer.render(n.templateId, n.data);
    const channel = this.factory.get(n.channel);
    const result = await channel.send(n.recipient, content);
    if (!result.success && !result.permanent) throw new Error('retry');   // → backoff
    if (!result.success && result.permanent) await this.toDlq(n, result);
  }
}
```

**This maps directly to your SaaS project:**
- Kafka producers publish; consumers fan-out to channels.
- Factory creates the right channel.
- DLQ + retry policy for failures.
- Strategy for channels and rendering.

**Follow-up 1: How do you guarantee a notification is sent exactly once?**
You usually can't get true exactly-once across external providers. Aim for at-least-once + idempotency: tag each notification with a unique ID, have providers dedupe (or track sent IDs in your DB). For critical notifications, accept possible duplicates over possible loss — most channels tolerate the rare duplicate better than a missed message.

**Follow-up 2: How do you scale to millions of notifications?**
Partition by user/tenant across Kafka partitions for parallel processing. Separate queues per channel (email is I/O-bound, can have high concurrency; SMS may have provider rate limits). Batch where providers support it (SendGrid batch API). Rate-limit per provider to respect their limits. Monitor per-channel delivery rates and DLQ depth. This is exactly the architecture behind your "10K+ notifications/day with dead-letter queue."

---

### Q20. What are common design pattern anti-patterns and over-engineering traps?

**Answer:**

**Over-application of patterns:**
1. **Pattern fever** — using patterns to look sophisticated rather than to solve a problem. A `FactoryFactory`, an `AbstractStrategyProviderBuilder`. If the pattern doesn't reduce real coupling or complexity, skip it.

2. **Premature abstraction.** Building interfaces and strategies for variation that doesn't exist yet. "We might need multiple payment providers someday" → don't build the abstraction until you have the second provider. YAGNI.

3. **Singleton abuse** — using Singleton as a backdoor for global mutable state. Creates hidden dependencies and untestable code.

4. **Anemic domain model** — objects with only getters/setters and no behavior; all logic in "service" classes. Often a sign you're not modeling the domain, just shuffling data.

5. **God object** — one class that does everything (the opposite of SRP). "Manager", "Helper", "Util" classes that grow without bound.

6. **Deep inheritance hierarchies** — five levels of inheritance where composition would be clearer. Fragile, hard to change.

**Specific traps:**
7. **Repository over an ORM that's already a repository** — adding a layer that just forwards calls. Only wrap if you add value.
8. **Observer making flow untraceable** — too many implicit event listeners; nobody can follow what happens when X fires.
9. **Strategy for a single algorithm** — one implementation behind an interface "for flexibility" that never materializes.
10. **Builder for two-field objects** — ceremony with no payoff.

**The meta-principle:** patterns are vocabulary for solutions to recurring problems, not goals in themselves. Reach for a pattern when you recognize the problem it solves in your actual code — not preemptively. "Three similar lines is better than a premature abstraction."

**Follow-up 1: How do you decide when an abstraction is worth it?**
The "rule of three": don't abstract until you have three concrete instances of the pattern. With one, you can't see the right abstraction; with two, you're guessing; with three, the commonality is clear. Abstracting too early locks in the wrong shape. Wait for the duplication to teach you what to abstract.

**Follow-up 2: How do you refactor an over-engineered codebase?**
Carefully and incrementally. Identify abstractions with only one implementation — inline them. Find pattern ceremony that adds no value — simplify. But don't rip out abstractions that are load-bearing (used in tests, multiple implementations). Measure: does removing this make the code simpler AND keep it correct? The goal is the simplest code that meets requirements, not maximal pattern usage — and not maximal cleverness in removing them either.

---

*End of section 12. Next: IoT & MQTT (10 questions).*
