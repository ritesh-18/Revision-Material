# 02 — NestJS & Express (25 Questions)

NestJS is the framework backing your Tibil and multi-tenant SaaS work. These questions cover DI, the request pipeline, lifecycle, microservices transport, and the production gotchas interviewers love to dig into.

**Sections**
- Part A — Architecture & DI (Q1–Q6)
- Part B — Request pipeline (Q7–Q14)
- Part C — Advanced features (Q15–Q22)
- Part D — Production concerns (Q23–Q25)

---

## Part A — Architecture & DI

### Q1. What is NestJS and how does it compare to plain Express?

**Answer:**
NestJS is an opinionated Node.js framework that sits on top of Express (or optionally Fastify) and adds **structure**: dependency injection, a module system, decorators for routing, and lifecycle hooks. The mental model borrows heavily from Angular and Spring Boot — controllers, services, modules, providers.

What Nest gives you that Express doesn't:
- **DI container** — you don't `new` your services; the framework wires them.
- **Module system** — explicit boundaries between feature areas with clear public APIs.
- **Decorators for routing** (`@Controller`, `@Get`) instead of `app.get(...)` calls scattered around.
- **Built-in patterns** — guards (auth), interceptors (cross-cutting concerns), pipes (validation), exception filters — all composable.
- **Microservices transport** — same controller code can serve HTTP, Kafka, Redis pub/sub, gRPC, etc.

The cost: more abstraction, a learning curve, and metadata reflection that adds startup time and some debugging complexity.

**Code — same endpoint, both frameworks:**
```ts
// Express
const router = express.Router();
router.get('/users/:id', authMiddleware, async (req, res, next) => {
  try {
    const user = await usersService.findOne(req.params.id);
    res.json(user);
  } catch (e) { next(e); }
});

// NestJS
@Controller('users')
@UseGuards(AuthGuard)
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.users.findOne(id);
  }
}
```

The Nest version is shorter because exception handling, DI, and middleware composition are framework concerns.

**Follow-up 1: When would you choose Express over NestJS?**
For very small services, scripts, or proxy/gateway-style apps where the structure cost outweighs the benefit. Also for teams that don't want the decorator/metadata magic — Nest's "where is this actually wired up?" can frustrate engineers used to explicit imports. For anything multi-module with > 10 controllers, Nest's structure pays off fast.

**Follow-up 2: Does NestJS slow down request handling?**
The per-request overhead is small (a few microseconds) — DI resolution for request-scoped providers, interceptor chain, pipe execution. For most APIs it's noise compared to DB and downstream latency. Where you'd notice: very tight, high-throughput endpoints (>20k req/sec per process). Mitigation: use default singleton-scoped providers, switch the adapter to Fastify (`@nestjs/platform-fastify`) for ~2x throughput.

---

### Q2. Explain dependency injection in NestJS. How does the container resolve dependencies?

**Answer:**
DI is the pattern of giving a class its dependencies from outside instead of constructing them itself. NestJS implements DI via a container that, at app bootstrap, builds a graph of every provider, resolves the order, instantiates each, and injects them into constructors based on the parameter types.

The mechanism:
1. You declare a class as `@Injectable()` — this stamps metadata on the class.
2. You list it in a module's `providers` array.
3. When another class declares a constructor parameter of that type, Nest's container looks up an instance, creating one if needed.
4. TypeScript's `emitDecoratorMetadata: true` puts each parameter's type into `design:paramtypes`; Nest reads this via `reflect-metadata`.

Providers are **singletons by default** — one instance per module's scope, shared across the whole app.

**Code:**
```ts
@Injectable()
export class UsersService {
  constructor(
    private readonly db: DatabaseService,    // resolved from DI
    private readonly logger: Logger,
  ) {}
}

@Module({
  providers: [UsersService, DatabaseService, Logger],
  controllers: [UsersController],
  exports: [UsersService],                    // make it available to other modules
})
export class UsersModule {}
```

**Follow-up 1: What's the difference between class providers, value providers, and factory providers?**

```ts
{
  provide: UsersService,                       // class provider (most common)
  useClass: UsersService,                       
},
{
  provide: 'CONFIG',                            // value provider — useful for constants
  useValue: { apiKey: process.env.API_KEY },
},
{
  provide: 'DB_CONNECTION',                     // factory — async or computed
  useFactory: async (config: ConfigService) => {
    return await createPool(config.get('DATABASE_URL'));
  },
  inject: [ConfigService],
}
```

Factory providers are how you wire up things that need async setup or runtime configuration.

**Follow-up 2: How does Nest handle circular dependencies?**
By breaking the type-level chain. Wrap one side with `forwardRef(() => OtherService)` and inject with `@Inject(forwardRef(...))`. This works but is a code smell — usually it means two services share too much logic and should be split into a third. Nest also detects cycles at bootstrap and gives an error pointing to the chain.

---

### Q3. What are NestJS modules and why does the module system matter?

**Answer:**
A module is a class annotated with `@Module({...})` that declares a slice of the app: controllers, providers, imports, exports. Modules are the unit of **encapsulation** — providers declared inside a module are private to it unless explicitly exported.

The `@Module` metadata has four arrays:
- `imports` — other modules whose **exports** become available here.
- `providers` — classes/values registered with the DI container for this module.
- `controllers` — HTTP/RPC entry points.
- `exports` — providers that other modules importing this one can inject.

The benefit: clear boundaries. `UsersModule` doesn't accidentally depend on `BillingModule`'s internal cache repository unless `BillingModule` explicitly exports it. This makes refactoring safer and the dependency graph readable.

**Code — module structure for your multi-tenant SaaS:**
```ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService, PasswordHasher],
  controllers: [UsersController],
  exports: [UsersService],                        // PasswordHasher stays private
})
export class UsersModule {}

@Module({
  imports: [UsersModule, ConfigModule],           // can inject UsersService here
  providers: [AuthService, JwtStrategy],
  controllers: [AuthController],
})
export class AuthModule {}
```

**Follow-up 1: What's a `@Global()` module?**
A module marked `@Global()` makes its exports available everywhere without needing to be imported into each module. Use sparingly — for truly cross-cutting concerns like logging, config, or telemetry. Overusing it defeats encapsulation; it's a sign you might just want a globally registered provider.

**Follow-up 2: When should you create a separate module vs adding to an existing one?**
Create a new module when the new code has its own coherent responsibility, its own external interface, and could plausibly be deployed independently later. Don't create modules for organizational neatness alone — a `HelpersModule` full of tiny utilities is worse than just having them as exported functions.

---

### Q4. What are provider scopes? Singleton vs Request vs Transient.

**Answer:**
By default, NestJS providers are **singletons** — one instance for the lifetime of the application, shared across all requests. This is fine and efficient for stateless services (most of your code). But sometimes you need per-request or per-injection state. That's where scopes come in.

```ts
@Injectable({ scope: Scope.DEFAULT })       // singleton — the default
@Injectable({ scope: Scope.REQUEST })       // new instance per HTTP request
@Injectable({ scope: Scope.TRANSIENT })     // new instance per injection point
```

- **`Scope.DEFAULT` (singleton)** — shared instance. Use unless you have a reason not to.
- **`Scope.REQUEST`** — Nest creates a new instance for each incoming request, then disposes it. Useful for request-scoped context (current user, tenant ID, request ID). Has performance cost: the entire dependency chain that touches a request-scoped provider also becomes request-scoped.
- **`Scope.TRANSIENT`** — fresh instance every time it's injected. Rare.

**Why request scope is expensive:**
```
ControllerA (request)  ──┐
                          ├──▶ ServiceB (request because A is) ──▶ Repo (request because B is)
```
Every layer above a request-scoped provider becomes request-scoped. For a high-throughput API, this means N times more allocations per second.

**Better alternative:** use `AsyncLocalStorage` (or `nestjs-cls`) to propagate request context through singleton services. You get per-request data without per-request instantiation.

**Follow-up 1: Why does request scope break some patterns?**
Because the lifecycle is tied to the HTTP request. Background jobs, scheduled tasks, and message consumers don't have an HTTP request — injecting a request-scoped provider into a scheduled task throws or gives undefined behavior. You'd need a separate "scope context" mechanism for non-HTTP entry points.

**Follow-up 2: How do you access the current request inside a service without making it request-scoped?**
Use `nestjs-cls` (Continuation Local Storage), which uses `AsyncLocalStorage` under the hood:
```ts
@Injectable()
export class UsersService {
  constructor(private readonly cls: ClsService) {}

  findAll() {
    const tenantId = this.cls.get('tenantId');
    return this.db.query('...').where({ tenantId });
  }
}
```
The middleware populates the CLS at request start; all singleton services read from it.

---

### Q5. What are controllers and how do route decorators work?

**Answer:**
A **controller** is a class that maps HTTP routes (or microservice patterns) to handler methods. The `@Controller(prefix)` decorator declares the base path; method-level decorators (`@Get`, `@Post`, `@Put`, `@Patch`, `@Delete`) declare the verb and sub-path. Parameter decorators (`@Param`, `@Query`, `@Body`, `@Headers`, `@Req`, `@Res`) extract pieces from the request.

At bootstrap, Nest scans every controller, reads the metadata stamped by these decorators, and registers routes on the underlying Express (or Fastify) app.

**Code:**
```ts
@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get()                                            // GET /users
  findAll(@Query('limit') limit = 20) {
    return this.users.findAll(limit);
  }

  @Get(':id')                                       // GET /users/:id
  findOne(@Param('id', ParseUUIDPipe) id: string) { // pipe validates UUID
    return this.users.findOne(id);
  }

  @Post()
  @HttpCode(201)
  create(@Body() dto: CreateUserDto) {
    return this.users.create(dto);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() dto: UpdateUserDto) {
    return this.users.update(id, dto);
  }
}
```

The controller is thin by design — its only job is to bind HTTP to service calls. All business logic stays in services. This makes controllers easy to swap (HTTP → gRPC, REST → GraphQL) and easy to test (mock the service, assert the wiring).

**Follow-up 1: What's the difference between `@Res()` and returning a value?**
By default, you return a value from the handler and Nest serializes it to JSON with status 200. If you inject `@Res() res` and use it (`res.send(...)`), you've taken control of the response — Nest no longer applies interceptors or serializes the return value. Use `@Res({ passthrough: true })` if you need to set headers but still want Nest's auto-serialization.

**Follow-up 2: How do you handle file uploads?**
`@nestjs/platform-express` ships with Multer integration. Use `@UseInterceptors(FileInterceptor('file'))` on the handler and `@UploadedFile() file: Express.Multer.File` to access it. Configure storage (memory, disk, S3 multipart) in the interceptor options. Validate size and MIME type with a custom pipe to avoid storing 4GB malware.

---

### Q6. Explain the difference between middleware, guards, interceptors, pipes, and exception filters.

**Answer:**
These are five overlapping but distinct extension points. The right one depends on **what** you're doing and **when** in the request lifecycle.

The execution order for a request:

```
   Incoming HTTP request
         │
         ▼
   ┌──────────────┐
   │  middleware  │  Express-level. Knows nothing of routes yet.
   └──────┬───────┘  Use for: low-level concerns (CORS, compression, raw logging).
          ▼
   ┌──────────────┐
   │    guards    │  After routing; can read decorator metadata.
   └──────┬───────┘  Use for: auth/authz (return true/false/throw).
          ▼
   ┌──────────────┐
   │ interceptors │  Wrap the handler call. Can run code before AND after.
   │   (before)   │  Use for: logging timing, cache, response transform.
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │    pipes     │  Per-parameter. Validate and transform inputs.
   └──────┬───────┘  Use for: DTO validation, parse-int, type coercion.
          ▼
   ┌──────────────┐
   │   handler    │  Your @Get / @Post method body.
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │ interceptors │
   │   (after)    │  Map / catch / tap on the response stream.
   └──────┬───────┘
          ▼
   exception filter (if anything threw)
          ▼
   serialized response
```

**Use the right tool for the job:**
- **Middleware** — generic HTTP concerns that don't need Nest's context.
- **Guards** — "is this request allowed?" → boolean.
- **Pipes** — "is this *parameter* valid? transform it." → throws BadRequest on failure.
- **Interceptors** — "wrap the handler with cross-cutting logic." → logging, caching, response shape.
- **Exception filters** — "translate this error into an HTTP response." → centralized error handling.

**Follow-up 1: Why can't I just put everything in middleware like Express?**
Middleware runs before Nest has resolved the route handler, so it can't access decorator metadata (`Reflector.get(...)`). For auth that depends on a `@Roles('admin')` decorator on the handler, you need a guard, not middleware. Middleware is also tied to the underlying framework — switch from Express to Fastify and Express-style middleware needs adjustment.

**Follow-up 2: In what order do multiple guards/interceptors run?**
Guards: global → controller → method, all checked left-to-right; any returning false stops the chain. Interceptors: same scope order, but they nest like an onion — controller-level wraps method-level, global wraps everything. Pipes: parameter-by-parameter in declaration order. Exception filters: most specific wins (method-level > controller-level > global), and the first matching `@Catch(...)` handles it.

---

## Part B — Request Pipeline

### Q7. How do guards work? When use them vs middleware?

**Answer:**
A guard is a class implementing `CanActivate` with a single method `canActivate(context): boolean | Promise<boolean> | Observable<boolean>`. Return `true` to let the request through, `false` (or throw) to block it. Guards run after routing has happened, so they have full access to:
- The handler reference and its metadata (via `Reflector`).
- The class (controller) reference and its metadata.
- The Nest execution context (request type: HTTP, RPC, WS, GraphQL).

This is what makes guards strictly more powerful than middleware for auth. With a guard, you can check "does this handler have `@Roles('admin')`?" and act accordingly. Middleware has no idea what route it's about to hit.

**Code — role-based auth guard:**
```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>('roles', [
      ctx.getHandler(),     // method-level @Roles
      ctx.getClass(),       // controller-level @Roles
    ]);
    if (!required) return true;

    const req = ctx.switchToHttp().getRequest();
    return required.some(role => req.user?.roles?.includes(role));
  }
}

// usage
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

@Controller('admin')
@UseGuards(JwtAuthGuard, RolesGuard)
export class AdminController {
  @Get('users')
  @Roles('admin')
  listUsers() { ... }
}
```

**Follow-up 1: When *should* I use middleware instead of a guard?**
For things that don't depend on route metadata and that you want to run on a per-path basis: request logging, CORS handling (though `app.enableCors()` does this), compression, raw-body parsing for webhook signatures. Middleware is also useful when you need to mutate the request object before Nest's pipes see it.

**Follow-up 2: How do you pass data from a guard to a handler?**
Attach it to the request object: `request.user = decodedJwt;` inside the guard. Then in the handler, use a custom parameter decorator to extract it cleanly:
```ts
export const CurrentUser = createParamDecorator(
  (_, ctx) => ctx.switchToHttp().getRequest().user
);

@Get('me')
me(@CurrentUser() user: User) { return user; }
```

---

### Q8. What are interceptors and what are the common use cases?

**Answer:**
Interceptors implement `NestInterceptor` with `intercept(context, next): Observable<any>`. The `next.handle()` call invokes the rest of the chain (other interceptors and finally the handler) and returns an RxJS Observable of the response. You can do work **before** calling `next.handle()` and use RxJS operators to do work **after**.

This makes interceptors the right tool for **cross-cutting wrap-around concerns** — logging request timing, caching responses, transforming output shape, retrying on failure, adding tracing spans.

**Code — common patterns:**

Logging timing:
```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const start = Date.now();
    const req = ctx.switchToHttp().getRequest();
    return next.handle().pipe(
      tap(() => console.log(`${req.method} ${req.url} ${Date.now() - start}ms`)),
    );
  }
}
```

Response transformation (wrap every response in `{ data: ... }`):
```ts
@Injectable()
export class WrapResponseInterceptor implements NestInterceptor {
  intercept(_ctx, next: CallHandler) {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```

Caching:
```ts
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(@Inject(CACHE_MANAGER) private cache: Cache) {}
  async intercept(ctx, next: CallHandler) {
    const key = ctx.switchToHttp().getRequest().url;
    const cached = await this.cache.get(key);
    if (cached) return of(cached);
    return next.handle().pipe(tap(v => this.cache.set(key, v, 60)));
  }
}
```

**Follow-up 1: How is an interceptor different from middleware that runs before and after?**
Middleware can run code before `next()`, but to run code after the response is sent it has to listen on the `res` `finish` event — clunky. Interceptors are built on Observables, so the "after" path is a normal `tap`, `map`, or `catchError`. Plus interceptors see the *return value* of the handler before serialization; middleware only sees raw response bytes.

**Follow-up 2: Can interceptors short-circuit the handler?**
Yes — just don't call `next.handle()`. Instead return your own Observable. This is how the caching interceptor above bypasses the handler on a cache hit. Be careful: if you skip `next.handle()`, downstream interceptors and the handler don't run, and side effects like database writes don't happen.

---

### Q9. What are pipes? Show validation and transformation.

**Answer:**
Pipes run on **individual parameters** of a handler before the handler executes. They have two jobs:
1. **Validation** — ensure the parameter is well-formed; throw `BadRequestException` if not.
2. **Transformation** — convert the parameter to the right type or shape (string → number, plain object → DTO instance).

Built-in pipes: `ValidationPipe`, `ParseIntPipe`, `ParseUUIDPipe`, `ParseBoolPipe`, `ParseArrayPipe`, `ParseEnumPipe`, `DefaultValuePipe`.

The most-used is `ValidationPipe`, which works with `class-validator` decorators on DTO classes.

**Code:**
```ts
import { IsEmail, IsInt, Min, IsOptional, IsEnum } from 'class-validator';

export class CreateUserDto {
  @IsEmail() email: string;
  @IsInt() @Min(13) age: number;
  @IsOptional() @IsEnum(['admin', 'user']) role?: string;
}

// Apply globally — every controller benefits
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,                   // strip unknown properties
  forbidNonWhitelisted: true,        // throw if unknown properties present
  transform: true,                   // convert plain JSON to class instance
  transformOptions: { enableImplicitConversion: true },  // "13" -> 13
}));

@Post()
create(@Body() dto: CreateUserDto) {   // dto is now validated AND a CreateUserDto instance
  return this.users.create(dto);
}
```

For single-value parameters, use named pipes:
```ts
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) { ... }

@Get()
list(@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number) { ... }
```

**Follow-up 1: How do you write a custom pipe?**
Implement `PipeTransform<T, R>`:
```ts
@Injectable()
export class TrimPipe implements PipeTransform<string, string> {
  transform(value: string) {
    if (typeof value !== 'string') throw new BadRequestException();
    return value.trim();
  }
}
```
Custom pipes are great for normalization (lowercase emails, trim whitespace) and for domain-specific parsing (parse an ISO date string into a Date).

**Follow-up 2: Why does `transform: true` matter for `class-validator`?**
Without `transform`, the parameter stays a plain object — `req.body` is the literal JSON. The validation decorators still work, but the value isn't actually a `CreateUserDto` instance. With `transform: true`, it's converted via `class-transformer`, so `instanceof` checks work and any methods defined on the class are callable. Crucial when combined with `enableImplicitConversion`, which coerces `"42"` from query strings to `42` for `@IsInt()` fields.

---

### Q10. How do exception filters work? Write a custom one.

**Answer:**
Exception filters catch errors thrown anywhere in the request pipeline (controllers, services, pipes, guards, other interceptors) and translate them into HTTP responses. Nest ships with a default filter that handles `HttpException` and its subclasses (`NotFoundException`, `BadRequestException`, etc.), turning them into proper JSON responses with the right status code.

You write custom filters when you need to:
- Catch domain-specific error classes and map them to HTTP responses.
- Add logging or tracing context to every error.
- Format errors consistently across the API (e.g., always `{ error: { code, message, details } }`).

**Code — domain error filter:**
```ts
// Domain layer throws plain TS errors, not HTTP exceptions
export class UserNotFoundError extends Error {
  constructor(public readonly userId: string) {
    super(`User ${userId} not found`);
  }
}

@Catch(UserNotFoundError)
export class UserNotFoundFilter implements ExceptionFilter {
  catch(err: UserNotFoundError, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    res.status(404).json({
      error: { code: 'USER_NOT_FOUND', message: err.message, userId: err.userId },
    });
  }
}

// Apply globally so every controller benefits
app.useGlobalFilters(new UserNotFoundFilter());
```

This pattern keeps your service layer **HTTP-agnostic** — services throw domain errors, the filter translates them at the edge. Same service can be reused from a queue consumer or CLI tool without HTTP coupling.

**Follow-up 1: How do you catch all unhandled errors generically?**
Use `@Catch()` with no argument — it catches everything:
```ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(err: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse<Response>();
    if (err instanceof HttpException) {
      return res.status(err.getStatus()).json(err.getResponse());
    }
    // Log and return generic 500 — don't leak details
    logger.error({ err }, 'unhandled');
    res.status(500).json({ error: 'INTERNAL' });
  }
}
```

**Follow-up 2: What's the order if multiple filters match an exception?**
Most specific wins. A filter with `@Catch(UserNotFoundError)` beats `@Catch(Error)` beats `@Catch()`. Within the same specificity, method-level filters beat controller-level, which beat global. Once a filter handles an exception, no further filters run.

---

### Q11. How do DTOs work with `class-validator` and `class-transformer`?

**Answer:**
DTO (Data Transfer Object) is the typed shape of incoming request bodies / outgoing responses. In NestJS, you define DTOs as classes and decorate fields with validation rules from `class-validator`. When combined with `ValidationPipe`, the framework parses, validates, and instantiates them automatically.

Why classes and not interfaces? Because interfaces are erased at compile time — they don't exist at runtime. Decorators need to attach metadata to *something*, and that something has to survive to runtime. Hence classes.

**Code — multi-tenant SaaS user invite DTO:**
```ts
import { IsEmail, IsEnum, IsOptional, IsString, MaxLength } from 'class-validator';
import { Type } from 'class-transformer';

export class InviteUserDto {
  @IsEmail()
  email: string;

  @IsEnum(['admin', 'manager', 'viewer'])
  role: string;

  @IsOptional()
  @IsString()
  @MaxLength(500)
  message?: string;

  @IsOptional()
  @Type(() => Date)           // class-transformer parses ISO string to Date
  expiresAt?: Date;
}

@Post('invites')
invite(@Body() dto: InviteUserDto) { ... }
```

With `transform: true` in the global `ValidationPipe`:
- Request body is parsed.
- All fields validated; any failure → `BadRequestException` with detailed error per field.
- The plain object is converted to an `InviteUserDto` instance — `expiresAt` is a real `Date`.

**Follow-up 1: How do you exclude internal fields from outgoing responses?**
Use `@Exclude()` from `class-transformer` and a serialization interceptor (`ClassSerializerInterceptor`):
```ts
export class User {
  id: string;
  email: string;
  @Exclude() passwordHash: string;
}

app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));
```
Now `passwordHash` never appears in JSON output. Important: only works if you return class instances, not plain objects.

**Follow-up 2: How do you reuse a DTO with slight variations (create vs update)?**
Use mapped-type helpers from `@nestjs/mapped-types`:
```ts
export class CreateUserDto { @IsEmail() email: string; @IsInt() age: number }
export class UpdateUserDto extends PartialType(CreateUserDto) {}   // all fields optional
export class PublicUserDto extends OmitType(User, ['passwordHash'] as const) {}
```
This avoids duplicating decorators and keeps types in sync.

---

### Q12. What are custom decorators? Show a parameter decorator and a metadata decorator.

**Answer:**
NestJS exposes two main custom-decorator helpers:

**`createParamDecorator`** — extracts a value to inject as a handler parameter. Useful for "pull X out of the request" patterns.

**`SetMetadata`** — attaches metadata to a class or method, read later by a guard, interceptor, or filter via `Reflector`. Useful for declarative annotations like `@Roles`, `@Public`, `@RateLimit`.

**Code — parameter decorator:**
```ts
export const CurrentUser = createParamDecorator(
  (data: keyof User | undefined, ctx: ExecutionContext) => {
    const req = ctx.switchToHttp().getRequest();
    const user = req.user;
    return data ? user?.[data] : user;
  },
);

// usage
@Get('me')
me(@CurrentUser() user: User) { return user; }

@Get('my-id')
myId(@CurrentUser('id') id: string) { return id; }
```

**Metadata decorator (the `@Roles` pattern from Q7):**
```ts
export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// Guard reads it
const required = this.reflector.getAllAndOverride<string[]>(ROLES_KEY, [
  ctx.getHandler(), ctx.getClass(),
]);
```

**Composed decorators** combine multiple decorators into one for ergonomics:
```ts
import { applyDecorators } from '@nestjs/common';

export function Auth(...roles: string[]) {
  return applyDecorators(
    UseGuards(JwtAuthGuard, RolesGuard),
    Roles(...roles),
    ApiBearerAuth(),                              // Swagger doc
  );
}

@Controller('admin')
export class AdminController {
  @Get('users')
  @Auth('admin')                                  // one decorator, three behaviors
  listUsers() { ... }
}
```

**Follow-up 1: When should I write a custom param decorator vs just reading from `@Req()`?**
Whenever the same extraction happens in 3+ places. Hand-rolled `req.user.tenantId` access scattered across the codebase is fragile — change the request shape and you hunt down every reference. A `@TenantId()` decorator centralizes the contract: change the implementation once, all consumers stay correct.

**Follow-up 2: Why use `getAllAndOverride` instead of `get`?**
`get(key, target)` returns metadata from one target. `getAllAndOverride(key, [handler, class])` returns the *first* match in the chain — so method-level decorators override class-level ones. There's also `getAllAndMerge` for combining (e.g., union of class-level and method-level roles). Choose based on whether class metadata should be inherited or overridden.

---

### Q13. Walk me through the entire NestJS request lifecycle, end to end.

**Answer:**
For an HTTP request, here's exactly what happens:

```
1.  Express/Fastify receives the TCP request.
2.  Express middleware runs in registration order.
        ├─ body parsers (express.json, urlencoded)
        ├─ cookie parser, CORS, helmet
        └─ Nest-bound middleware (configured via MiddlewareConsumer)
3.  Nest router matches the route to a controller method.
4.  Global guards → controller guards → method guards run (in order).
        ├─ Any guard returning false → ForbiddenException short-circuit
        └─ Throwing → exception filter
5.  Global interceptors → controller interceptors → method interceptors enter
    (BEFORE next.handle()). Stack pushes.
6.  Global pipes → controller pipes → method pipes → parameter pipes run
    against each handler parameter:
        ├─ @Body() runs ValidationPipe on the DTO
        ├─ @Param('id') runs ParseUUIDPipe
        └─ Throwing → BadRequestException → exception filter
7.  The handler method executes.
8.  Return value bubbles up through interceptors AFTER next.handle()
    (stack pops). Each can map / tap / catch.
9.  Exception filter catches any error thrown anywhere in 3–8 and writes a response.
10. Response is serialized (JSON) and sent.
```

The `BEFORE` and `AFTER` halves of interceptors mean they nest like a Russian doll: outer interceptors see the result of inner ones.

**Code — observing the lifecycle:**
```ts
@Injectable()
export class TraceInterceptor implements NestInterceptor {
  intercept(_ctx, next: CallHandler) {
    console.log('--> handler entering');
    return next.handle().pipe(
      tap(() => console.log('<-- handler exited successfully')),
      catchError(err => { console.log('<-- handler threw', err); throw err; }),
    );
  }
}
```

**Follow-up 1: Where in the lifecycle does auth happen?**
At the guard stage (step 4) — after middleware (so body parsers have run) and before pipes (so the DTO doesn't need to validate auth headers). The order matters: putting auth in middleware misses decorator-driven role checks; putting it in a pipe runs after validation, which means unauthenticated requests waste validation cycles.

**Follow-up 2: What's the difference between throwing in a guard vs returning false?**
Returning `false` causes Nest to throw a `ForbiddenException` automatically (403). Throwing your own exception lets you control the status code and message — `throw new UnauthorizedException('Token expired')` produces a 401 with that specific message. Throwing is more informative for clients and easier to debug.

---

### Q14. How do NestJS lifecycle hooks work? When are they called?

**Answer:**
Modules and providers can implement lifecycle interfaces to run code at specific points in the application's life:

```
App startup:
  ┌─ OnModuleInit         ── Each module after all providers are instantiated
  ├─ OnApplicationBootstrap ── After every module has completed OnModuleInit
  │
App shutdown (after enableShutdownHooks()):
  ├─ OnModuleDestroy       ── On SIGTERM/SIGINT, before processes close
  ├─ BeforeApplicationShutdown ── After all OnModuleDestroy completed
  └─ OnApplicationShutdown ── Final hook before exit
```

You use these for:
- Opening / closing database connections.
- Starting / stopping background workers.
- Warming caches.
- Flushing buffered logs.
- Graceful shutdown (drain in-flight requests, close DB pool).

**Code — graceful shutdown with database cleanup:**
```ts
@Injectable()
export class DatabaseService implements OnModuleInit, OnApplicationShutdown {
  private pool!: Pool;

  async onModuleInit() {
    this.pool = await createPool(this.config.dbUrl);
    await this.pool.query('SELECT 1');     // health check at startup
  }

  async onApplicationShutdown(signal?: string) {
    console.log(`shutting down (signal=${signal}); closing DB pool`);
    await this.pool.end();
  }
}

// main.ts
const app = await NestFactory.create(AppModule);
app.enableShutdownHooks();                  // critical — wires SIGTERM/SIGINT
await app.listen(3000);
```

**Important:** `enableShutdownHooks()` is **not on by default**. Without it, your `OnApplicationShutdown` hooks never fire on SIGTERM — Kubernetes kills the container with open DB connections.

**Follow-up 1: Why is `OnModuleInit` per-module instead of one big start hook?**
Because modules may depend on each other. `DatabaseModule.onModuleInit` should complete before `UsersModule.onModuleInit` runs — otherwise UsersService might query a pool that isn't open. Nest runs them in dependency-resolution order. `OnApplicationBootstrap` is the "everyone is ready" hook for app-wide cross-module initialization.

**Follow-up 2: How do you implement graceful shutdown for a Kubernetes rolling deploy?**
1. K8s sends `SIGTERM` and waits `terminationGracePeriodSeconds` (default 30s).
2. Stop accepting new connections: `app.close()` calls `server.close()`.
3. Wait for in-flight requests to finish (or timeout).
4. Close downstream resources (DB pool, message queue, Redis).
5. Process exits.

Set Kubernetes `terminationGracePeriodSeconds` greater than the longest expected request duration (e.g., 60s if you have 30s-long requests). Don't forget to drain the readiness probe a few seconds before the grace period starts so the LB stops sending traffic.

---

## Part C — Advanced Features

### Q15. How do you create dynamic modules in NestJS? Show `forRoot` / `forFeature` patterns.

**Answer:**
A **dynamic module** is one whose configuration is determined at import time rather than declared statically. `forRoot()` configures the module's global settings; `forFeature()` registers a feature-specific subset (e.g., entities for a repository).

The convention you've seen in `TypeOrmModule.forRoot(...)`, `JwtModule.register(...)`, `MongooseModule.forRoot(...)` follows this pattern.

**Code — building your own configurable module (a Keycloak module from your SaaS project):**
```ts
// keycloak.module.ts
@Module({})
export class KeycloakModule {
  static forRoot(options: KeycloakOptions): DynamicModule {
    return {
      module: KeycloakModule,
      global: true,                                     // make exports global
      providers: [
        { provide: 'KEYCLOAK_OPTIONS', useValue: options },
        KeycloakAdminService,
        KeycloakTokenService,
      ],
      exports: [KeycloakAdminService, KeycloakTokenService],
    };
  }

  static forRootAsync(opts: {
    imports?: any[];
    useFactory: (...args: any[]) => Promise<KeycloakOptions> | KeycloakOptions;
    inject?: any[];
  }): DynamicModule {
    return {
      module: KeycloakModule,
      global: true,
      imports: opts.imports,
      providers: [
        {
          provide: 'KEYCLOAK_OPTIONS',
          useFactory: opts.useFactory,
          inject: opts.inject ?? [],
        },
        KeycloakAdminService,
        KeycloakTokenService,
      ],
      exports: [KeycloakAdminService, KeycloakTokenService],
    };
  }
}

// usage with async config from ConfigService
@Module({
  imports: [
    KeycloakModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (c: ConfigService) => ({
        url: c.get('KEYCLOAK_URL'),
        realm: c.get('KEYCLOAK_REALM'),
        clientId: c.get('KEYCLOAK_CLIENT_ID'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

**Follow-up 1: Why have both `forRoot` and `forRootAsync`?**
`forRoot` takes a literal options object — fine for static config. `forRootAsync` lets you depend on other providers (like `ConfigService`) to compute the options, which is essential when config comes from env vars validated through a config module, or from a remote secrets manager called at startup.

**Follow-up 2: When do you use `forFeature` vs `forRoot`?**
`forRoot` configures the module globally (one DB connection, one JWT secret, one Redis client). `forFeature` registers *feature-specific* providers using the global config — e.g., `TypeOrmModule.forFeature([User, Post])` creates repositories for these entities, all using the connection set up by `forRoot`. So `forRoot` once at the root, `forFeature` in every feature module that needs the resource.

---

### Q16. How does configuration management work? Show `ConfigModule` with validation.

**Answer:**
NestJS has an official `@nestjs/config` package that loads env vars, validates them at startup, and exposes them through an injectable `ConfigService`. The recommended setup:

1. Load `.env` files (and process env) into an in-memory config.
2. Validate the entire config against a schema at boot — **fail fast** if a required var is missing or malformed.
3. Inject `ConfigService` wherever you need values.

**Code:**
```ts
// config.schema.ts
import { z } from 'zod';

export const configSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  KEYCLOAK_URL: z.string().url(),
  JWT_PRIVATE_KEY: z.string().min(64),
});

export type Config = z.infer<typeof configSchema>;

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ['.env.local', '.env'],
      validate: (raw) => configSchema.parse(raw),     // throws on bad config
    }),
  ],
})
export class AppModule {}

// usage
@Injectable()
export class JwtSigner {
  constructor(private cfg: ConfigService<Config, true>) {}    // true = inferred types

  sign(payload: object) {
    return jwt.sign(payload, this.cfg.get('JWT_PRIVATE_KEY'));
  }
}
```

**Why fail-fast validation matters:** without it, a missing `DATABASE_URL` causes a cryptic ECONNREFUSED ten minutes after deploy when the first request comes in. With validation, the app refuses to start and the deploy fails clearly.

**Follow-up 1: How do you handle different configs per environment?**
Three approaches:
1. **Per-env env files** — `.env.production`, `.env.staging`, picked by `NODE_ENV`.
2. **Single source, conditional defaults in the schema** — `MAX_REQUESTS: z.coerce.number().default(NODE_ENV === 'production' ? 1000 : 100)`.
3. **Secrets manager at startup** — load from AWS Secrets Manager / Vault, merge into config before validation.

In production, prefer #3 for secrets and env vars for non-secret config. Never commit `.env` files with real values.

**Follow-up 2: How do you make the config type-safe end-to-end?**
Use the generic in `ConfigService<T, true>` where `T` is your inferred schema type. With `strict: true` (the second generic), `cfg.get('SOME_KEY')` becomes a compile-time check — typos in key names throw at build, and the return type is the actual field's type (number, URL string, enum literal), not `any`.

---

### Q17. How do NestJS microservices work? What transports are supported?

**Answer:**
NestJS supports running the same controller-style code over **non-HTTP transports** via the `@nestjs/microservices` package. Instead of `@Get`/`@Post`, you use `@MessagePattern(pattern)` (request/response) or `@EventPattern(pattern)` (fire-and-forget). The framework routes messages from the transport layer to the right handler.

Supported transports:
- **TCP** — simple JSON-over-TCP. Useful for internal service-to-service inside a cluster.
- **Redis** — pub/sub. Good for events and lightweight RPC.
- **NATS** — high-throughput messaging.
- **RabbitMQ / AMQP** — full broker semantics (queues, exchanges, DLQ).
- **Kafka** — partitioned event streaming. What you used at Tibil and on the SaaS project.
- **gRPC** — strongly typed Protocol Buffers, HTTP/2.
- **MQTT** — your IoT pipeline.
- **Custom transport** — implement `CustomTransportStrategy` for anything else.

**Code — Kafka consumer in your notification pipeline:**
```ts
// notifications.controller.ts
@Controller()
export class NotificationsController {
  constructor(private readonly senders: NotificationSenders) {}

  @MessagePattern('notification.email')         // request/response
  async sendEmail(@Payload() msg: EmailMsg) {
    return this.senders.email(msg);             // returns delivery receipt
  }

  @EventPattern('user.signed_up')               // fire-and-forget
  async onUserSignedUp(@Payload() event: UserSignedUpEvent) {
    await this.senders.welcome(event.userId);
  }
}

// main.ts — wire up Kafka transport
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.KAFKA,
  options: {
    client: { brokers: ['kafka:9092'] },
    consumer: { groupId: 'notifications-consumer' },
  },
});
await app.listen();
```

**Hybrid apps** — you can run HTTP and microservice transports in the same process:
```ts
const app = await NestFactory.create(AppModule);
app.connectMicroservice<MicroserviceOptions>({ transport: Transport.KAFKA, options: {...} });
await app.startAllMicroservices();
await app.listen(3000);
```

**Follow-up 1: What's the difference between `@MessagePattern` and `@EventPattern`?**
`@MessagePattern` is request/response: the publisher waits for a reply. `@EventPattern` is fire-and-forget: publish and move on. Use events for notifications (downstream services react to "order_placed"); use messages when you need a result back ("validate_payment → ok/error").

**Follow-up 2: How does the same code work over Kafka vs RabbitMQ vs TCP?**
The handler shape is identical (`@MessagePattern('x')`); only the transport configuration differs. Nest's microservice layer abstracts the wire protocol. Semantic differences leak through — Kafka has consumer groups and partitions, RabbitMQ has queues and exchanges, TCP has neither — but for simple request/response patterns the code is portable. For Kafka-specific features (manual commit, partition assignment), you'd reach for `kafkajs` directly.

---

### Q18. How does the CQRS module work in NestJS?

**Answer:**
CQRS (Command Query Responsibility Segregation) splits write operations (commands) from read operations (queries) into separate code paths. `@nestjs/cqrs` provides the scaffolding: command bus, query bus, event bus, sagas.

The basic flow:
1. Controller dispatches a **command** (a plain class describing an intent).
2. A **command handler** processes it (one handler per command class).
3. The handler may emit **events** to communicate to the rest of the system.
4. **Event handlers** react to events asynchronously.
5. **Queries** read state via dedicated query handlers (often hitting denormalized read models).

CQRS shines in domain-rich apps where the write model and read model diverge (event sourcing, separate read replicas, search indexes). For simple CRUD, it's overkill.

**Code:**
```ts
// Command (write intent)
export class CreateOrderCommand {
  constructor(
    public readonly userId: string,
    public readonly items: OrderItem[],
  ) {}
}

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(
    private readonly repo: OrdersRepository,
    private readonly bus: EventBus,
  ) {}

  async execute(cmd: CreateOrderCommand) {
    const order = Order.create(cmd.userId, cmd.items);
    await this.repo.save(order);
    this.bus.publish(new OrderCreatedEvent(order.id, cmd.userId));
    return order.id;
  }
}

// Event handler reacts asynchronously
@EventsHandler(OrderCreatedEvent)
export class SendOrderConfirmation implements IEventHandler<OrderCreatedEvent> {
  constructor(private mail: MailService) {}
  handle(e: OrderCreatedEvent) {
    return this.mail.sendOrderConfirmation(e.userId, e.orderId);
  }
}

// Controller
@Post('orders')
async create(@Body() dto: CreateOrderDto, @CurrentUser() u: User) {
  const id = await this.commandBus.execute(new CreateOrderCommand(u.id, dto.items));
  return { id };
}
```

**Follow-up 1: When is CQRS *not* the right choice?**
For simple CRUD apps where the read and write models are essentially the same table. CQRS adds command/event class proliferation and indirection — for an admin panel with 5 endpoints, it's pure overhead. Apply it when business logic is complex, when reads vastly outnumber writes, or when you need an audit log via event sourcing.

**Follow-up 2: How does CQRS interact with event sourcing?**
They're often paired but separate concepts. CQRS = separate read/write paths. Event sourcing = the write side stores domain events (not state); state is rebuilt by replaying. Together, the write side appends events to a store; events propagate to event handlers that update read models. NestJS's CQRS module includes `AggregateRoot` for event-sourced aggregates and `EventStore` integration.

---

### Q19. How do you implement authentication in NestJS with Passport + JWT?

**Answer:**
NestJS integrates with Passport via `@nestjs/passport`. The pattern: define a **strategy** (how to extract and verify credentials), wrap it in a **guard** (auto-named after the strategy), apply the guard to routes.

For JWT auth, the flow:
1. Login route: validates username/password, signs and returns a JWT.
2. JWT strategy: extracts the token from the Authorization header, verifies signature, attaches the decoded payload to `req.user`.
3. JWT auth guard: applies the strategy to protect routes.

**Code:**
```ts
// jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(cfg: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: cfg.get('JWT_PUBLIC_KEY'),       // RS256 verification
      algorithms: ['RS256'],
    });
  }

  validate(payload: { sub: string; email: string; roles: string[] }) {
    // Whatever this returns becomes req.user
    return { id: payload.sub, email: payload.email, roles: payload.roles };
  }
}

// jwt-auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// auth.module.ts
@Module({
  imports: [
    JwtModule.registerAsync({
      useFactory: (cfg: ConfigService) => ({
        privateKey: cfg.get('JWT_PRIVATE_KEY'),
        publicKey: cfg.get('JWT_PUBLIC_KEY'),
        signOptions: { algorithm: 'RS256', expiresIn: '15m' },
      }),
      inject: [ConfigService],
    }),
  ],
  providers: [JwtStrategy, AuthService],
  controllers: [AuthController],
})
export class AuthModule {}

// usage
@Controller('me')
@UseGuards(JwtAuthGuard)
export class MeController {
  @Get() me(@CurrentUser() user: User) { return user; }
}
```

**Follow-up 1: How do you implement refresh tokens?**
Issue two tokens at login: short-lived access (15 min) and long-lived refresh (7–30 days, stored as `httpOnly secure` cookie). The refresh token is opaque (or a separately-keyed JWT) and stored in DB so it can be revoked. The `/refresh` endpoint validates the refresh token, optionally rotates it, and issues a fresh access token. Refresh tokens must be one-time-use or rotated to detect theft.

**Follow-up 2: Why use RS256 over HS256?**
`HS256` uses a shared secret — anyone who can verify can also sign. With many services that *verify* tokens, that's a large blast radius if any one is compromised. `RS256` uses an asymmetric key pair: only the auth service has the private key (sign); verifiers only have the public key. You can hand out the public key freely (JWKS endpoint) without enabling impersonation.

---

### Q20. How do you test NestJS apps? Show unit and e2e patterns.

**Answer:**
NestJS embraces testing in two layers:

**Unit tests** — test a service in isolation with mocked dependencies, using `Test.createTestingModule`.
**E2E tests** — boot the entire app (or a subset) and hit it with `supertest` to make real HTTP calls.

**Code — unit test with mocked dependency:**
```ts
describe('UsersService', () => {
  let service: UsersService;
  let repo: jest.Mocked<UsersRepository>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: UsersRepository,
          useValue: { findById: jest.fn(), save: jest.fn() },
        },
      ],
    }).compile();

    service = module.get(UsersService);
    repo = module.get(UsersRepository);
  });

  it('returns null when user not found', async () => {
    repo.findById.mockResolvedValue(null);
    expect(await service.findOne('123')).toBeNull();
  });

  it('saves a normalized email', async () => {
    repo.save.mockImplementation(u => Promise.resolve(u));
    const result = await service.create({ email: 'MIXED@Case.com' });
    expect(repo.save).toHaveBeenCalledWith(expect.objectContaining({ email: 'mixed@case.com' }));
  });
});
```

**E2E test:**
```ts
describe('UsersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [AppModule],                    // real module graph
    })
      .overrideProvider(KeycloakService)
      .useValue({ verify: () => ({ sub: 'test-user' }) })
      .compile();

    app = module.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
    await app.init();
  });

  it('POST /users — validates input', async () => {
    const res = await request(app.getHttpServer())
      .post('/users')
      .send({ email: 'not-an-email' });
    expect(res.status).toBe(400);
    expect(res.body.message[0]).toMatch(/email must be an email/);
  });

  afterAll(() => app.close());
});
```

**Follow-up 1: Unit vs integration vs e2e — what's the right ratio?**
The "testing trophy" model: lots of integration tests, fewer unit tests on pure logic, a small number of e2e tests for critical flows. Integration tests (a service + a real DB in a Docker container) catch the most bugs per minute of test time. Pure unit tests on services that mostly call other services are low-value — they test that mocks work. Pure logic (parsers, validators, calculators) deserves unit tests.

**Follow-up 2: How do you handle DB state between tests?**
For integration/e2e: use a real DB (Postgres in a Docker container for CI; testcontainers library helps). Three strategies for isolation: (1) transaction-per-test that rolls back, (2) truncate all tables in `afterEach`, (3) test-per-schema. Transactions are fastest but break if the code-under-test uses its own transactions. Truncate is simple and reliable, just slower.

---

### Q21. How do you handle file uploads in NestJS?

**Answer:**
NestJS wraps Multer (for Express adapter) or fastify-multipart (for Fastify) under a unified interceptor API. The flow: an interceptor parses the multipart body, extracts file(s), and exposes them via `@UploadedFile()` / `@UploadedFiles()`.

**Code — single file with validation:**
```ts
@Controller('uploads')
export class UploadsController {
  constructor(private readonly storage: StorageService) {}

  @Post()
  @UseInterceptors(FileInterceptor('file', {
    limits: { fileSize: 5 * 1024 * 1024 },         // 5 MB cap
    fileFilter: (_req, file, cb) => {
      if (!['image/png', 'image/jpeg'].includes(file.mimetype)) {
        return cb(new BadRequestException('only png/jpeg'), false);
      }
      cb(null, true);
    },
  }))
  async upload(
    @UploadedFile(new ParseFilePipe({
      validators: [
        new MaxFileSizeValidator({ maxSize: 5 * 1024 * 1024 }),
        new FileTypeValidator({ fileType: /^image\/(png|jpeg)$/ }),
      ],
    })) file: Express.Multer.File,
    @CurrentUser() user: User,
  ) {
    const url = await this.storage.put(user.id, file);
    return { url };
  }
}
```

**For your IoT/S3 pipeline scenario** — uploading huge files without buffering them in memory — use streaming directly to S3:
```ts
import { Upload } from '@aws-sdk/lib-storage';

@Post('large')
async uploadLarge(@Req() req: Request) {
  // Skip Multer entirely; pipe the request stream to S3
  const upload = new Upload({
    client: s3,
    params: { Bucket: 'uploads', Key: nanoid(), Body: req },
  });
  await upload.done();
  return { ok: true };
}
```

**Follow-up 1: Memory storage vs disk storage vs streaming directly — when to use each?**
- **Memory** (default) — convenient, OK for small files (< 1 MB). Risk: 100 concurrent 50 MB uploads = 5 GB heap.
- **Disk** — Multer writes to tmp dir; you process from there. Better for medium files, but you must clean up.
- **Direct streaming to object storage** — no temp file, no memory blowup. Best for any file > 10 MB.

**Follow-up 2: How do you handle resumable / chunked uploads?**
For very large files (gigabytes), use multipart uploads with presigned URLs. Backend issues a presigned URL per chunk; client uploads chunks directly to S3 from the browser; backend finalizes the multipart upload. Saves bandwidth and protects your service from being a bottleneck for huge transfers.

---

### Q22. How does logging work in NestJS? How would you replace the default logger?

**Answer:**
NestJS ships with a built-in `Logger` that writes prettified messages to the console. It's fine for development but limiting in production where you want structured JSON, log levels, correlation IDs, and shipping to a log aggregator (CloudWatch, ELK, Loki).

The recommended swap: **`nestjs-pino`** (or plain `pino` integration). Pino is one of the fastest Node loggers, outputs structured JSON, and supports child loggers with bound context.

**Code — pino integration:**
```ts
// main.ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));                  // make pino the global logger
```

```ts
// app.module.ts
@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: {
        autoLogging: true,
        customLogLevel: (_req, res, err) =>
          err || res.statusCode >= 500 ? 'error'
          : res.statusCode >= 400 ? 'warn' : 'info',
        redact: ['req.headers.authorization', 'req.headers.cookie'],
        formatters: { level: (label) => ({ level: label }) },   // structured level
      },
    }),
  ],
})
export class AppModule {}
```

```ts
// Inside any service
@Injectable()
export class UsersService {
  private readonly log = new Logger(UsersService.name);

  async create(dto: CreateUserDto) {
    this.log.log({ email: dto.email }, 'creating user');
    // ...
  }
}
```

**Best practices:**
- Structured logs (JSON) — searchable in any log aggregator.
- Bind request ID per request (via interceptor or pino-http auto).
- Redact secrets — `redact` paths in pino-http prevent accidental token logs.
- Use levels consistently — `error` for failures requiring action, `warn` for unusual but handled, `info` for important business events, `debug` for development.

**Follow-up 1: How do you propagate a request ID through async calls?**
Two approaches: (1) pass it explicitly as a function argument (boring but clear); (2) use `AsyncLocalStorage` to make it ambient, accessed from anywhere via a CLS service. Pino's child logger pattern lets you create per-request loggers in middleware: `req.log = logger.child({ requestId })`, and inject `req.log` (or read from CLS).

**Follow-up 2: How do you avoid logging huge objects that flood your aggregator?**
Truncate or summarize. For pino, set `serializers` per field. For arrays/objects, log `{ count: arr.length }` or `{ keys: Object.keys(obj) }`. Establish a convention: never log full request/response bodies in production — they leak PII, cost money in log volume, and slow down search.

---

## Part D — Production Concerns

### Q23. Express adapter vs Fastify adapter — when to switch?

**Answer:**
NestJS ships with two HTTP adapters. Express is the default — battle-tested, with the largest middleware ecosystem. Fastify is a modern alternative — schema-based, async-native, **about 2x faster** in raw throughput benchmarks.

**When Fastify wins:**
- You're building a high-throughput service (>10k req/sec per process).
- You don't need niche Express middleware.
- You want JSON schema-based serialization (faster than `JSON.stringify`).

**When to stay on Express:**
- Existing app with deep Express middleware integration (custom body parsers, session stores).
- Team is more comfortable with the Express ecosystem.
- Throughput isn't the bottleneck (most apps are bottlenecked on the DB).

**Code — Fastify adapter:**
```ts
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter({ logger: false }),
);
await app.listen(3000, '0.0.0.0');               // Fastify needs explicit host
```

Most NestJS code is portable across adapters — the controller/service/DI layers are identical. The differences show up in middleware, request/response object shape, and adapter-specific features (Fastify hooks, plugin system).

**Follow-up 1: What concrete differences will trip me up when switching?**
The `req` and `res` objects are different types. `req.cookies` works on both with appropriate plugins, but `req.session` (express-session) doesn't have a direct Fastify equivalent — you use `@fastify/session`. Multer is Express-only; for Fastify use `@fastify/multipart`. Some third-party Nest packages assume Express; check before committing.

**Follow-up 2: How much faster is Fastify really, in a realistic app?**
Benchmarks (e.g., TechEmpower) show 2–3x throughput on synthetic "return JSON" endpoints. In a real app where 80% of latency is the DB, the wall-clock difference is closer to 10–20% improvement. Worth it for genuinely high-traffic services; usually not worth a rewrite.

---

### Q24. How do you implement API versioning in NestJS?

**Answer:**
NestJS supports four versioning strategies out of the box: URI (`/v1/users`), header, media type, and custom. URI is by far the most common — it's explicit, cacheable, and the easiest to test.

**Code — URI versioning:**
```ts
// main.ts
app.enableVersioning({
  type: VersioningType.URI,
  defaultVersion: '1',
});

// controller
@Controller({ path: 'users', version: '1' })
export class UsersV1Controller { ... }

@Controller({ path: 'users', version: '2' })
export class UsersV2Controller { ... }

@Controller({ path: 'health' })                  // VERSION_NEUTRAL → /health works for all
export class HealthController { ... }
```

The above produces routes `/v1/users`, `/v2/users`, `/health`. Clients can be migrated incrementally; old versions are deprecated and removed on a stated timeline.

**Versioning strategies that actually scale:**
- **Don't break compatibility** — additive changes (new fields, new endpoints) don't need a new version.
- **Version only the changed endpoints**, not the whole API at once — many apps run with `/v1/users` and `/v2/orders` simultaneously.
- **Document the deprecation date** in the response (`Sunset` header per RFC 8594).

**Follow-up 1: When does breaking-change API versioning matter vs additive evolution?**
Most changes can be made additively: add a new field, accept a new optional parameter, return new endpoints. Versioning is needed for *semantic* breaks: changing the shape of a response, removing fields, changing validation rules, changing authentication. Don't version per release; version per breaking change.

**Follow-up 2: How do you avoid duplicating logic across versions?**
Push business logic into a shared service. Controllers (v1 and v2) become thin adapters that map their version-specific request/response shapes to/from the canonical service interface. If v1 and v2 diverge significantly in business logic, that's a sign they're conceptually different operations and should have different service methods, not different versions.

---

### Q25. What are the common NestJS production gotchas you've seen?

**Answer:**
Real-world traps that bite teams:

1. **Forgot `enableShutdownHooks()`.** SIGTERM kills the container instantly; in-flight requests die, DB connections leak. Fix in `main.ts`.

2. **Request-scoped providers infect the dependency graph.** One `@Injectable({ scope: REQUEST })` makes every service that consumes it (and consumers of those, transitively) also request-scoped, multiplying allocations per request. Use `AsyncLocalStorage` / `nestjs-cls` instead.

3. **Global `ValidationPipe` with default settings doesn't strip unknown fields.** Bad clients can sneak unexpected fields into your DTOs. Set `whitelist: true, forbidNonWhitelisted: true`.

4. **Circular dependency between modules.** Bootstrap hangs or throws a cryptic "Nest can't resolve" error. Use `forwardRef` short-term but redesign — the cycle usually means two services should be unified or split via a shared third.

5. **No DB pool tuning.** Default pools from `pg`/`mysql2` are small (10). Under load, requests queue waiting for a connection. Tune to `(num_workers * max_concurrent_queries)`.

6. **Logging unstructured strings in production.** No way to filter, search, or alert. Switch to pino with JSON output from day one.

7. **JWT secrets in env vars committed to repo.** Use a secrets manager (AWS Secrets Manager, Vault) and validate at startup that the secret meets minimum entropy.

8. **No graceful handler for `unhandledRejection`.** Default behavior in Node 15+ is to crash, but if you swallowed it, a memory-leaking promise hangs forever.

9. **Microservice transport has different error semantics than HTTP.** A Kafka consumer that throws is retried; an HTTP handler that throws gets one 500 to the client. Apps that share controller code across transports get this wrong easily.

10. **Heavy work in middleware blocks all requests.** Middleware runs synchronously through Express's chain — a slow middleware = head-of-line blocking. Move heavy work into the handler or into a background queue.

**Follow-up 1: How do you debug a Nest app that's slow under load but fast at low traffic?**
Profile event loop delay first (`perf_hooks.monitorEventLoopDelay`) — if delay spikes, something is blocking the loop (sync crypto, JSON.parse of huge bodies, regex backtracking). If event loop is fine, check the DB pool: are requests queuing on connection acquisition? Then check downstream service latency. Finally, profile CPU with `clinic flame` to find hot functions.

**Follow-up 2: How do you handle a memory leak in a NestJS app specifically?**
Same as Node generally (Q10/Q26 in section 01), but Nest-specific suspects: (1) request-scoped providers retaining references to closures that close over big objects, (2) `EventEmitter`-based providers without listener cleanup in `OnModuleDestroy`, (3) cached metadata maps growing unbounded (e.g., a custom decorator that caches reflected metadata per-instance instead of per-class). Heap snapshot, look at retainers, walk the chain back to the leaking provider.

---

*End of section 02. Next: PostgreSQL deep dive (25 questions).*
