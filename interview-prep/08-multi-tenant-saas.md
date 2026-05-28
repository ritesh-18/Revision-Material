# 08 — Multi-Tenant SaaS (15 Questions)

Your job platform project implemented schema-per-tenant Postgres isolation with dynamic schema switching, Keycloak SSO + tenant-scoped RBAC, sub-2s tenant onboarding, and Kafka notification fan-out. These cover the design space (silo / pool / hybrid), tenant resolution, migrations, isolation enforcement, noisy neighbors, and the operational gotchas.

---

### Q1. What are the main multi-tenancy strategies and their trade-offs?

**Answer:**
Three classic patterns:

**Pool (shared everything):**
- One database, one schema; tenant identified by a `tenant_id` column on every table.
- Cheap per tenant. Easy to add capacity.
- Risk: a missing WHERE clause leaks data between tenants. Application-level enforcement is the only barrier.
- Best for: SMB SaaS, very high tenant count (10K+), uniform feature sets.

**Silo (one DB per tenant):**
- Each tenant gets its own database (or even its own cluster).
- Strongest isolation: separate connections, separate backups, separate failure domains, easy compliance.
- High cost per tenant (every DB has overhead). Hard to query across tenants.
- Best for: enterprise tenants paying premium for isolation, regulated industries, data sovereignty.

**Schema-per-tenant (your pattern):**
- One database, one schema per tenant. Same Postgres instance; schemas are namespaces.
- Middle ground: better isolation than pool (separate tables; missing WHERE doesn't accidentally read another tenant), much cheaper than silo (one instance, shared connection infrastructure).
- Postgres limits: rough ceiling around 5–10K schemas per cluster before catalog overhead bites.
- Best for: hundreds to a few thousand tenants, mixed-tier offering.

**Comparison:**
| Aspect | Pool | Schema | Silo |
|--------|------|--------|------|
| Cost per tenant | Lowest | Moderate | Highest |
| Isolation strength | Lowest | Moderate | Highest |
| Cross-tenant ops | Easy | Medium | Hard |
| Schema migrations | One run | Loop over tenants | Loop over tenants |
| Scale ceiling | Very high tenants | Few thousand | Few hundred |
| Data residency | Single region | Single region | Per-tenant |

**Follow-up 1: Why did you choose schema-per-tenant over pool?**
The job platform handled sensitive applicant data with per-tenant retention rules and the occasional "give us all this tenant's data and delete it from your system" compliance request. Schema isolation makes those trivial: pg_dump one schema, drop the schema. Pool would require careful WHERE-clause filtering on every operation and a multi-day delete job.

**Follow-up 2: When would I move from schema to silo?**
When (a) a tenant pays for guaranteed isolation, (b) regulatory requirements force regional / per-tenant infrastructure, (c) one tenant's load patterns hurt others (noisy neighbor), or (d) the catalog overhead of thousands of schemas starts to slow queries. Many SaaS run schema-per-tenant for SMB and silo for enterprise.

---

### Q2. Walk me through your schema-per-tenant implementation in NestJS.

**Answer:**
The core idea: per-request, set the Postgres `search_path` to the tenant's schema. All subsequent queries in that connection see the tenant's tables transparently.

**The pieces:**
1. **Tenant resolution middleware** — extracts `tenantId` from request (header, subdomain, JWT).
2. **AsyncLocalStorage / CLS** — stores tenant context for the request lifecycle.
3. **TypeORM / Prisma interceptor** — before query, sets `search_path` on the borrowed connection.
4. **Tenant schemas** in Postgres created on tenant onboarding.

**Code — middleware + CLS + interceptor:**
```ts
// Tenant middleware
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  constructor(private readonly cls: ClsService) {}
  use(req: Request, _res: Response, next: NextFunction) {
    const tenantId = req.headers['x-tenant-id'] ?? req.user?.tenantId;
    if (!tenantId) throw new BadRequestException('missing tenant');
    this.cls.set('tenantId', tenantId);
    next();
  }
}

// Query interceptor — for every DB connection borrow, set search_path
@Injectable()
export class TenantQueryInterceptor implements NestInterceptor {
  constructor(
    private readonly cls: ClsService,
    @InjectDataSource() private readonly ds: DataSource,
  ) {}
  async intercept(ctx: ExecutionContext, next: CallHandler) {
    const tenantId = this.cls.get('tenantId');
    return next.handle().pipe(
      tap(async () => {
        // (better: do this once per query runner, not per request)
        await this.ds.query(`SET search_path TO "${this.escape(tenantId)}", public`);
      }),
    );
  }
}
```

**Better pattern with TypeORM QueryRunner per request:**
```ts
@Injectable()
export class TenantDataSourceService {
  constructor(@InjectDataSource() private readonly ds: DataSource) {}

  async runWithTenant<T>(tenantId: string, fn: (m: EntityManager) => Promise<T>): Promise<T> {
    const runner = this.ds.createQueryRunner();
    await runner.connect();
    try {
      await runner.query(`SET search_path TO "${this.safeName(tenantId)}", public`);
      return await fn(runner.manager);
    } finally {
      await runner.release();
    }
  }
  private safeName(s: string) { return s.replace(/[^a-z0-9_]/gi, ''); }   // sanitize!
}
```

**Why the `safeName` matters:** the schema name goes into a literal SQL statement (`SET search_path TO ...`); parameter binding doesn't work for identifiers. SQL injection here would be devastating across tenants — escape rigorously or use an allowlist of known tenant IDs.

**Follow-up 1: Why not just include `tenant_id` in every WHERE clause like the pool model?**
You can, and many do. The schema approach has two advantages: (1) accidentally missing the WHERE in a new query simply returns zero rows from the wrong schema rather than another tenant's data, (2) per-schema migrations are easier — drop the schema, you've deleted everything. Pool requires constant vigilance.

**Follow-up 2: How do you handle shared/global data (e.g., country list, plan tiers)?**
Keep a `public` schema with shared tables. `SET search_path TO "tenant_x", public` means PG first looks in the tenant schema, falls back to public. Read works seamlessly; writes must explicitly target the right schema (`INSERT INTO public.plans` vs default).

---

### Q3. How do you do tenant resolution? Header, subdomain, path — which?

**Answer:**

**Subdomain (`acme.app.com`):**
- Pro: clean URLs, obvious to users, easy CDN/edge routing per tenant.
- Con: requires wildcard DNS + wildcard TLS (Let's Encrypt + automated renewal). Hard to share links cross-tenant.
- Good fit: B2B SaaS where tenants are organizations with their own branded URL.

**Path prefix (`app.com/acme/...`):**
- Pro: no DNS wildcard, simpler ops.
- Con: less obvious branding, URLs longer, all tenants look the same.
- Good fit: developer tools, internal-facing apps.

**Header (`X-Tenant-Id: acme`):**
- Pro: invisible to URL structure; flexible.
- Con: harder for users to bookmark / understand, no edge routing by tenant possible without inspecting body.
- Good fit: APIs, mobile apps.

**Claim in JWT (`tenant_id` in token):**
- Pro: tenant is part of identity; can't spoof if token is signed.
- Con: only works after auth (gateway routing needs another mechanism for the auth endpoint itself).
- Good fit: post-auth APIs in B2B SaaS.

**Common pattern:** subdomain for UI, header (or JWT claim) for APIs. Gateway resolves the tenant from whichever is present, normalizes into a `X-Tenant-Id` internal header.

**Code — gateway-level resolution:**
```ts
function resolveTenant(req): string {
  // Priority: explicit header > JWT claim > subdomain
  if (req.headers['x-tenant-id']) return req.headers['x-tenant-id'];
  if (req.user?.tenantId) return req.user.tenantId;
  const host = req.hostname;
  const match = host.match(/^([^.]+)\./);
  if (match && match[1] !== 'www') return match[1];
  throw new BadRequestException('cannot determine tenant');
}
```

**Follow-up 1: How do you handle "switch tenant" for users who belong to multiple?**
A user has N tenant memberships. Issue tokens scoped to one tenant at a time; a "switch tenant" UI re-issues a token for the new one. Or, more flexible: token contains all tenants the user belongs to; the request specifies which one via header. The latter is friendlier UX but requires verifying access on every request.

**Follow-up 2: How do you prevent a user from accessing another tenant by spoofing the header?**
Always validate the tenant claim against the user's allowed tenants — typically encoded in the JWT or fetched from auth. If the request asks for tenant X but the user only belongs to Y, reject. Never trust a tenant header from an external client without validation; trust it only inside your private network from the gateway.

---

### Q4. Schema migrations across thousands of tenants — how do you handle them?

**Answer:**
With shared-schema (pool), you run one migration. With schema-per-tenant, you must run it on every tenant's schema. With silo, against every tenant's DB. The mechanics differ but the same principles apply.

**Migration strategy:**
1. **Backwards-compatible by default.** Apps deploy before migrations run, and old code must work against new schemas.
2. **Schema migrations run idempotently** — re-running should be a no-op. Use migration tools (Knex migrations, TypeORM migrations, Flyway) that track applied versions per schema.
3. **Run in parallel, chunked.** Migrate 50 tenants at a time, not 5000 serially.
4. **Per-tenant version table** so the migrator knows which schemas have which migration version.

**Code sketch — your "automated migration runner":**
```ts
async function runMigrations(targetVersion: number) {
  const tenants = await db.tenant.findMany();
  const concurrency = pLimit(20);
  await Promise.all(tenants.map(t => concurrency(() => migrateOne(t, targetVersion))));
}

async function migrateOne(tenant: Tenant, target: number) {
  await ds.query(`SET search_path TO "${tenant.schema}"`);
  const cur = await getCurrentVersion(tenant.schema);
  if (cur >= target) return;
  for (let v = cur + 1; v <= target; v++) {
    await migrations[v].up();
    await setVersion(tenant.schema, v);
  }
}
```

**Onboarding new tenants:**
```ts
async function onboardTenant(name: string) {
  const schema = generateSchemaName(name);   // e.g., 't_acme_a1b2'
  await ds.query(`CREATE SCHEMA "${schema}"`);
  await migrateOne({ schema }, LATEST_VERSION);   // bring straight to current
  await db.tenant.create({ data: { name, schema } });
  return schema;
}
```

**Two-phase deploy for breaking changes:**
1. Deploy app version N+1 that handles both old and new schema (e.g., reads from both columns, writes to new).
2. Run migration adding new column.
3. Backfill data.
4. Deploy app version N+2 that only uses the new shape.
5. Run cleanup migration to drop the old column.

**Follow-up 1: What's your sub-2-second onboarding budget actually doing?**
The breakdown: `CREATE SCHEMA` (~50ms), running 80 cached migrations on the new schema (~1.5s for table creation, indexes, constraints), insert tenant record (~10ms), warm-up cache (~200ms). The trick is migrations are pre-compiled SQL strings, no parsing overhead.

**Follow-up 2: What happens if a migration fails on tenant 47 of 500?**
Mark tenant 47 as in a failed migration state; abort processing. Other tenants continue. Fix manually (rollback or fix the migration), retry. The migration runner must be resumable — failing halfway shouldn't leave the system in an unrecoverable state. Idempotency in each step is essential.

---

### Q5. How does RBAC work across tenants with Keycloak?

**Answer:**
**Keycloak** is an open-source identity provider supporting OAuth 2.0, OIDC, SAML. For multi-tenant apps, two design patterns:

**Realm-per-tenant:**
- Each tenant has its own Keycloak realm (separate users, separate roles, separate clients).
- Strongest isolation: users in tenant A can't see tenant B's users.
- Operationally heavier: managing N realms vs one.
- Login URL differs per tenant (`/realms/acme/...`).

**Single realm with tenant claims:**
- One realm; users carry a `tenant_id` claim and `tenant_roles` claim in their tokens.
- Operationally simpler.
- Risk: a misconfigured user could end up with access to multiple tenants.

**Your pattern (likely):** single realm with per-tenant client scopes. Each tenant has a client; user-to-tenant mapping enforced via group membership and a mapper that adds `tenant_id` to tokens.

**Role hierarchy at three levels:**
1. **Application-level roles** — `admin`, `manager`, `viewer` defined globally.
2. **Tenant-scoped membership** — a user has role X in tenant Y (stored as a group attribute or via Keycloak's "Authorization Services").
3. **Resource-level permissions** — controllers/methods annotated with required roles; guards enforce.

**Code — NestJS guard combining Keycloak token with tenant scope:**
```ts
@Injectable()
export class TenantRoleGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const required = this.reflector.getAllAndOverride<string[]>('roles', [
      ctx.getHandler(), ctx.getClass(),
    ]);
    if (!required) return true;

    const req = ctx.switchToHttp().getRequest();
    const tenantId = req.headers['x-tenant-id'];
    const claims = req.user;       // populated by JwtStrategy

    if (claims.tenant_id !== tenantId) return false;          // wrong tenant
    return required.some(r => claims.realm_access.roles.includes(r));
  }
}
```

Combined with declarative decorators:
```ts
@Controller('jobs')
@UseGuards(JwtAuthGuard, TenantRoleGuard)
export class JobsController {
  @Post() @Roles('admin', 'manager')
  create(@Body() dto: CreateJobDto) { ... }
}
```

**Follow-up 1: What's "zero privilege escalation incidents" actually mean?**
That your authorization layer never let one tenant or role do something they shouldn't. Achieved by: (a) centralized guards (no per-controller hand-rolled checks), (b) deny-by-default (no role declaration → no access), (c) tenant context bound to the request lifecycle so a stray query can't escape, (d) tests covering "user A tries to GET tenant B's resource — must 403."

**Follow-up 2: Why cache Keycloak token introspection?**
Each introspection is a network round-trip to Keycloak + Keycloak doing some DB work. Cache the result by token hash with TTL = (token expiration − safety margin). 95%+ cache hit rate turns sub-second auth latency into sub-millisecond. Invalidate on user logout / role change via Keycloak events.

---

### Q6. How do you onboard a new tenant programmatically?

**Answer:**
A complete onboarding flow needs:

1. **Validate inputs** — tenant name, domain, plan tier.
2. **Create the tenant record** in your main metadata DB.
3. **Provision Postgres schema** (or DB, depending on isolation strategy).
4. **Run migrations** to bring the new schema to current version.
5. **Seed default data** — default settings, sample workflows, demo job posts.
6. **Set up auth** — Keycloak client or group, initial admin user.
7. **Configure feature flags** for this tenant's plan.
8. **Warm caches** — preload common queries.
9. **Schedule recurring jobs** specific to the tenant (e.g., daily digest emails).
10. **Send welcome email / webhook.**

**Code — orchestration sketch:**
```ts
@Injectable()
export class TenantOnboardingService {
  async onboard(input: CreateTenantInput): Promise<Tenant> {
    return await this.db.$transaction(async (tx) => {
      const tenant = await tx.tenant.create({
        data: { name: input.name, plan: input.plan, schema: this.generateSchemaName(input.name) },
      });

      // Outside the transaction-safe path — these are external side effects
      // Use the outbox pattern or saga to make them recoverable
      await this.outbox.publish('tenant.creating', tenant);

      return tenant;
    });
  }
}

// Background consumer of 'tenant.creating' runs the provisioning
async function provision(tenant) {
  await pg.query(`CREATE SCHEMA "${tenant.schema}"`);
  await runMigrations(tenant.schema);
  await seedDefaults(tenant.schema, tenant.plan);
  await keycloak.createClient(tenant.id);
  await keycloak.createInitialAdmin(tenant.adminEmail);
  await sendWelcomeEmail(tenant);
  await db.tenant.update({ where: { id: tenant.id }, data: { provisioned: true } });
}
```

**For sub-2s "live" onboarding**, pre-cache migration scripts and run them in parallel. Heavy work (emailing, Keycloak setup) can be async — return success once the schema exists and minimal data is seeded.

**Follow-up 1: How do you rollback a failed onboarding?**
Compensating transaction (saga pattern). Each step's compensation: drop schema, delete Keycloak client, delete tenant record. Run compensations in reverse order. Tricky parts: side effects that can't be undone (welcome email already sent — can't unsend; just live with it).

**Follow-up 2: What's the right place to store the tenant-to-schema mapping?**
In a "control plane" database, typically a separate `tenants` table. This control plane DB is shared (pool-style) — tracks every tenant's metadata, plan, status, schema name. The control plane is the only thing that lists "all tenants"; tenant-specific data lives in their own schemas/databases.

---

### Q7. Cross-tenant operations — how do you support admin queries that span tenants?

**Answer:**
Operations like "show me total revenue across all tenants" or "find users with email X across all tenants" violate the per-tenant isolation. Handling them well:

**Option 1: Iterate tenants from the application.**
```ts
const tenants = await db.tenant.findMany();
const results = await Promise.all(tenants.map(t =>
  withSchema(t.schema, () => db.users.findMany({ where: { email } }))
));
return results.flat();
```
- Pros: explicit, isolation respected.
- Cons: N round trips; slow at scale.

**Option 2: Cross-schema query (Postgres supports it).**
```sql
SELECT 'acme' AS tenant, * FROM "acme".users WHERE email = $1
UNION ALL
SELECT 'beta' AS tenant, * FROM "beta".users WHERE email = $1
UNION ALL
... -- one branch per tenant
```
- Pros: single round-trip.
- Cons: query length grows linearly with tenants; not practical past a few hundred.

**Option 3: Denormalized cross-tenant index.**
- Maintain a `_global_user_index` table in the public schema with denormalized rows: `(tenant_id, user_id, email, ...)`.
- Populate via triggers or events on each tenant's user table.
- Query the index for cross-tenant lookups.
- Pros: single query, fast.
- Cons: extra storage, eventual consistency, write amplification.

**Option 4: Foreign data wrappers** — use Postgres FDW to query across separate databases (silo model). Slower than local, but possible.

**For your job platform**, admin operations across tenants are probably rare and tolerate latency — option 1 is fine. For frequent cross-tenant operations (e.g., billing dashboards), option 3.

**Follow-up 1: How do you audit cross-tenant access?**
Log every cross-tenant operation with elevated detail: which admin user, what query, what tenants accessed, what data returned. Treat these as security-sensitive events; alert if frequency exceeds baseline. The admin who exports tenant X's data shouldn't be able to do so without leaving a trail.

**Follow-up 2: What's the cost of supporting cross-tenant analytics in real time?**
High. Best practice: don't. Stream events to an analytics warehouse (Snowflake, BigQuery, ClickHouse) where cross-tenant aggregations run cheaply. The operational DB stays clean and isolated; the warehouse is the source of truth for analytics.

---

### Q8. How do you prevent data leakage between tenants?

**Answer:**
Data leakage = one tenant accidentally accessing another tenant's data. Defenses, layered:

**Layer 1: Database-level isolation.**
- Schema-per-tenant or DB-per-tenant: data lives in physically separate namespaces. A bug accessing the wrong schema would 404, not silently fetch wrong data.

**Layer 2: Application-level enforcement.**
- Centralized tenant context in middleware/CLS.
- Every query goes through a wrapper that asserts the current tenant and binds `search_path`.
- No direct DB access outside the wrapper — code review enforces this.

**Layer 3: Authorization.**
- Every operation checks: "does the current user have rights to this tenant?"
- Tenant ID from the request is validated against the user's claims.
- Resource-level checks: "does this resource belong to the current tenant?"

**Layer 4: Auditing and monitoring.**
- Log every request with tenant context.
- Alert on anomalies: a user querying across tenants, an unusual spike in cross-tenant queries, queries returning rows whose tenant doesn't match the request context.

**Layer 5: Testing.**
- Cross-tenant access tests: "user in tenant A receives 403/404 when trying to access tenant B's resource."
- Fuzz testing: alter `tenant_id` in requests and assert proper denial.

**Common leak vectors:**
- **Missing WHERE clause** in pool model.
- **Cached data without tenant key** — Redis cache key like `user:42` instead of `user:tenant1:42`. User 42 in two tenants would collide.
- **Background jobs without tenant context** — job processor doesn't know which tenant it's working on, defaults to the wrong one.
- **Cross-tenant relationship** — foreign keys that span tenants (don't do this).
- **Logging tenant data without redaction** — log lines that contain other tenants' data leaked into a shared log file.

**Follow-up 1: How do you cache safely across tenants?**
Always include tenant in the cache key: `user:{tenantId}:{userId}`. Or use separate Redis databases per tenant (Redis supports 16 logical DBs out of the box). For Redis Cluster where DBs don't exist, prefix everything. Never share an unprefixed cache across tenants.

**Follow-up 2: How do tests cover this?**
A "tenant isolation" test suite: create two tenants, populate each, simulate a request as user-of-A accessing resources of B. Every endpoint must reject. Run on every PR; treat any test failure as a P0 bug. The cost of writing them once is far less than the cost of a real leak.

---

### Q9. How do you handle connection pooling across many tenants?

**Answer:**
With schema-per-tenant on shared Postgres, the connection pool is shared too. With DB-per-tenant (silo), you need a more careful strategy.

**Schema-per-tenant (your model):**
- One pool to one Postgres instance.
- Each query: acquire connection, set search_path, run query, release.
- The pool itself doesn't care about tenant; it's the application's job to set the schema per use.
- Pool size: total throughput / per-query speed.

**DB-per-tenant (silo):**
- Naïve approach: pool per tenant. For 500 tenants × 10 conns = 5K connections. Postgres won't take that. Most are idle anyway.
- Better: shared pool, with connections lazily switched to whichever tenant DB the current request needs. PgBouncer in **transaction pool mode** with a per-tenant database parameter handles this.
- Or: pool of "connection slots" with the actual DB connection assigned on the fly.

**Per-tenant rate limiting on the pool:**
- One tenant shouldn't be able to exhaust connections and starve others.
- Apply per-tenant limit: each tenant can hold at most N pool connections at a time. Enforce via a semaphore keyed by tenant ID.

**Code — semaphore-style per-tenant cap:**
```ts
const tenantSlots = new Map<string, Semaphore>();

async function withTenantConn<T>(tenantId: string, fn: () => Promise<T>): Promise<T> {
  let sem = tenantSlots.get(tenantId);
  if (!sem) {
    sem = new Semaphore(20);    // max 20 concurrent connections per tenant
    tenantSlots.set(tenantId, sem);
  }
  await sem.acquire();
  try { return await fn(); }
  finally { sem.release(); }
}
```

**Follow-up 1: What's the right pool size?**
For shared-pool models: `(workers × concurrent_queries) + some headroom`. Total at most a few hundred per Postgres instance — more increases context-switch cost and lock contention. Use PgBouncer as a layer in front: app → PgBouncer (10K virtual conns) → 50 real backends → Postgres. Critical for connection-heavy workloads.

**Follow-up 2: Can I detect a "tenant hogging connections" situation in real time?**
Yes. Tag every connection acquisition with tenant ID, sample the active set periodically, alert if one tenant holds > X% of the pool. Combined with per-tenant query latency monitoring, you spot noisy neighbors before they break the service.

---

### Q10. Noisy neighbor — how do you contain one tenant's load from affecting others?

**Answer:**
A **noisy neighbor** is a tenant whose load patterns degrade service for others on shared infrastructure.

**Containment strategies:**

1. **Per-tenant rate limits.** A tenant can't issue more than N requests/sec to your API. Keeps absolute throughput controlled.

2. **Per-tenant connection limits.** Limit DB pool slots per tenant (Q9).

3. **Per-tenant CPU/memory quotas (in Kubernetes).** If you run separate pods per tier, premium tenants get more resources, free-tier shares a smaller cluster. Possible but complex.

4. **Per-tenant DB resource governor.** Postgres doesn't have native per-user quotas; use external tools or design constraints (move heavy tenants to silos).

5. **Query timeout per tenant.** A free-tier tenant's queries time out at 5s; enterprise tenants get 30s.

6. **Background job priority.** Separate BullMQ queues per tier; premium tenants' jobs processed first.

7. **Async-first design.** Cap synchronous work; push expensive operations to async queues with backpressure.

**Detecting noisy neighbors:**
- Monitor per-tenant: request rate, latency p95/p99, DB query count, queue depth.
- Compare against tenant's plan / baseline.
- Alert on outliers: "tenant X has 10x its usual rate" before pages start failing.

**Promoting tenants to silos:**
- If a tenant exceeds shared-tier resource expectations, migrate them to a dedicated DB / dedicated cluster.
- Charge accordingly.
- Common path for B2B SaaS: free/SMB on pool, premium/enterprise on silos.

**Follow-up 1: How would you detect a tenant doing a runaway query that locks tables?**
Postgres `pg_stat_activity` shows currently-running queries with `tenant_id` tagging (via `application_name` set per connection or via comments injected into queries). A monitoring job that scans `pg_stat_activity` for queries > 30s alerts you. Auto-kill at hard limits (`statement_timeout`) prevents long blockages.

**Follow-up 2: What's the trade-off of strict per-tenant quotas?**
Strict quotas reduce blast radius but can frustrate tenants who legitimately spike (campaign launches, demos). Pattern: soft quota that allows bursts, hard quota that's a multiple above. Plus elastic capacity in your auto-scaling so even a soft over-quota doesn't degrade others when the cluster can flex up.

---

### Q11. How do you do tenant-aware caching?

**Answer:**
Every cache key must include the tenant identifier — period. Without it, two tenants' data collides on the same key, leaking one's data to the other's reads.

**Patterns:**

**Key prefixing:**
```js
const key = `${tenantId}:user:${userId}`;
await redis.get(key);
```
Simple, works everywhere. Just need discipline at the wrapper.

**Tenant-scoped cache helper:**
```ts
@Injectable()
export class TenantCache {
  constructor(@Inject(REDIS) private redis: Redis, private cls: ClsService) {}

  private key(suffix: string) {
    const t = this.cls.get('tenantId');
    if (!t) throw new Error('no tenant in context');
    return `${t}:${suffix}`;
  }

  get<T>(k: string): Promise<T | null> { return this.redis.get(this.key(k)).then(JSON.parse); }
  set<T>(k: string, v: T, ttl: number) { return this.redis.setex(this.key(k), ttl, JSON.stringify(v)); }
  del(k: string) { return this.redis.del(this.key(k)); }
}
```

**Invalidation:**
- Per-tenant flush: `SCAN MATCH "tenant1:*"` then `DEL`. Slow on large keyspaces.
- Versioned namespace per tenant: `tenant1:v3:user:42`. Incrementing the version invalidates everything for that tenant atomically.

**Sizing:**
- One tenant should never be able to push every other tenant's hot keys out of the cache. Run with `allkeys-lfu` (Q3 of section 05) — frequently-accessed keys for active tenants stay hot.
- For premium tenants, consider isolated Redis instances.

**Follow-up 1: Why is forgetting the tenant prefix one of the worst SaaS bugs?**
Because user X in tenant A's cached profile gets served to user X in tenant B (if their IDs match — which they often do, since you may use integer auto-IDs). Even with UUID PKs, the impact of misrouted cache reads is severe (showing wrong tenant's data) and silent (no error, just wrong output). Defense: a code review checklist item, a lint rule, or a custom cache client that requires explicit tenant context.

**Follow-up 2: How do you cache cross-tenant aggregates (like the admin dashboard)?**
Different key namespace: `global:*` or `admin:*`. Reads bypass the tenant context. Write access tightly gated (admin only). Never shared with tenant-scoped routes.

---

### Q12. Tenant-aware logging and metrics — what's the right way?

**Answer:**

**Logging:**
- Every log line includes `tenantId` as a structured field.
- Use a context-aware logger that pulls tenant from CLS automatically.
- Log aggregator can filter by tenant for support / debugging.

```ts
function getLogger() {
  const cls = container.get(ClsService);
  return baseLogger.child({ tenantId: cls.get('tenantId'), userId: cls.get('userId') });
}
```

**Metrics:**
- **Don't tag every metric with tenant_id** — cardinality explosion (Q22 of section 07).
- Tag with **tenant_tier** (free, pro, enterprise) — low cardinality, useful for tracking each tier's behavior.
- Sample per-tenant metrics for top-N tenants — track the heaviest 50 tenants individually, group the rest.
- For per-tenant rate / latency, push to a separate analytical store (Prometheus exemplars, or just a periodic dump to a TSDB).

**Tracing:**
- Tag every span with `tenant.id` — high cardinality is fine in traces (you don't aggregate; you search). Lets you reconstruct a tenant's full request flow in incident investigation.

**Operational dashboards:**
- "Tenant health" view: per-tenant request rate, error rate, p95 latency. Heatmap by tier.
- "Top tenants by usage" — capacity-planning aid.

**Follow-up 1: How do you give a customer a usage report?**
Periodic aggregation job: read tenant's metrics from the past period, compute totals (requests, jobs run, storage used), generate a report. For real-time-ish dashboards, stream events to a per-tenant analytics store. For billing, append-only event log of metered operations, summed at the end of the billing cycle.

**Follow-up 2: What if a customer asks "how much is your service costing me?"**
You need per-tenant cost attribution. Track resource usage (CPU time, DB queries, storage, bandwidth) by tenant. Apply unit costs. Output a breakdown. Most SaaS just track usage and charge a flat tier fee; only enterprise customers usually ask for actual cost breakdowns.

---

### Q13. How do you handle data residency and compliance (GDPR, regional)?

**Answer:**
**Data residency** = data physically stored in specific geographic regions. Required by GDPR for EU citizens, by Russian / Chinese laws for those countries' citizens, and increasingly by enterprise customers' security policies.

**Architectural choices:**

**Per-region silo:** each region has its own cluster (DB, services, queues). Tenants in EU live entirely in EU infrastructure. Tenants in US live in US. No cross-region data movement.
- Strongest residency guarantee.
- Operational complexity multiplied per region.

**Per-region database with shared application:** apps run globally, but writes route to the tenant's home-region DB. Reads can stay regional if the app is region-aware.
- Moderate.

**Single-region with logical compliance:** all data in one region; satisfy compliance via legal mechanisms (Standard Contractual Clauses). Cheaper but weaker guarantee.

**GDPR-specific requirements:**
- **Right to be forgotten** — must delete a user's data on request. With schema-per-tenant, schema-level deletion is trivial; user-level deletion needs careful identification of all PII. Cross-tenant data (analytics) needs sanitization.
- **Data portability** — export user data in a machine-readable format. Build an export endpoint per tenant/user.
- **Consent and purpose limitation** — track what consent each user gave; respect it in processing.
- **Breach notification** — incident playbook with 72-hour notification SLA.

**Code — soft-delete with eventual purge:**
```ts
// User clicks "delete account"
await db.user.update({ where: { id }, data: { deletedAt: now() } });
await producer.publish('user.deletion_requested', { id, tenantId });

// Async worker, 30 days later
async function purgeOldDeletions() {
  const deletable = await db.user.findMany({ where: { deletedAt: { lt: daysAgo(30) } } });
  for (const u of deletable) {
    await db.user.delete({ where: { id: u.id } });
    await scrubFromAnalytics(u.id);
    await scrubFromBackups(u.id);    // hardest part
  }
}
```

**Follow-up 1: What's the hardest part of "right to be forgotten"?**
Backups. You can scrub from production easily, but backups are immutable for safety reasons. Two patterns: (a) shorten backup retention so old data ages out; (b) maintain a "deletion log" — on restore, re-apply scrubs. Most teams document this limitation and accept the trade-off.

**Follow-up 2: How do you handle data sovereignty when a tenant moves regions?**
Migration: export data from source region, import to destination, verify, switch routing, eventually delete from source. Complex; usually contracted as a project, not a self-service feature. Most SaaS pin a tenant to one region forever.

---

### Q14. How do you back up and restore per tenant?

**Answer:**

**With schema-per-tenant (your model):**
- **Per-tenant backup:** `pg_dump --schema=tenant_x ...`. Quick and clean.
- **Per-tenant restore:** `pg_restore` into a fresh schema; can be done while other tenants serve traffic.
- **Full-cluster backup also necessary** for the underlying instance.

```bash
# Backup one tenant's schema
pg_dump -h db -U app -n "tenant_acme" -F c app > /backups/acme-$(date +%F).dump

# Restore a tenant's schema to a fresh name (useful for "give me yesterday's data")
pg_restore -h db -U app -d app --schema=tenant_acme \
  --no-owner --create /backups/acme-2026-05-26.dump
```

**With pool model:**
- Per-tenant backup needs row-level export with `tenant_id` filter — slower, requires custom tooling.
- Restore is harder: row-by-row reinsert with conflict handling.

**With silo:**
- Per-tenant backup = backup a whole DB. Standard `pg_basebackup` per tenant.
- Restore = restore one DB. Independent.

**Use cases for per-tenant restore:**
- Tenant accidentally deleted important data — restore their schema from yesterday's backup.
- Compliance: "we need a copy of all our data as of date X."
- Tenant offboarding: hand them their data and delete from your system.

**Code — tenant export endpoint:**
```ts
@Post('export')
@Roles('admin')
async exportTenant(@CurrentUser() user: User) {
  const dumpPath = await this.backups.dumpTenantSchema(user.tenantId);
  const downloadUrl = await this.s3.uploadAndGetSignedUrl(dumpPath);
  await sendEmail(user.email, 'your-export-ready', { url: downloadUrl });
}
```

**Follow-up 1: What about point-in-time recovery per tenant?**
PITR is cluster-wide in Postgres — you can't replay WAL for just one schema. To get per-tenant PITR, you'd silo. Alternatively, take frequent schema-level dumps (every 15 minutes) and accept that the recovery point granularity is the dump interval, not seconds.

**Follow-up 2: How do you test that backups actually work?**
Periodic restore drill: take a backup, restore it to a sandbox environment, run a smoke test ("can we log in? can we list jobs?"). Many teams have backups they've never tested; the day they need it is the day they discover a flaw. Schedule monthly drills with results in your team's metrics.

---

### Q15. What are the most common multi-tenant production gotchas?

**Answer:**

1. **Missing tenant filter on a new endpoint.** Returns data from wrong tenant. Defense: centralized tenant context + tenant-scoped query wrappers; never raw queries that don't go through them.

2. **Background jobs lose tenant context.** Job created in tenant A runs in worker with no tenant set, queries the default schema (often public) or fails silently. Always serialize tenant ID into the job payload and re-establish context at start of processing.

3. **Cache keys missing tenant prefix.** User X in tenant A's data shows up for User X in tenant B.

4. **Forgotten "system" data.** Plan tiers, country lists, etc., live in `public` schema. Code that always uses `search_path` finds them, but code that explicitly uses schema names breaks. Document and enforce conventions.

5. **Catalog bloat from many schemas.** 5000+ schemas slows down `\dt`, slows Postgres's metadata operations. Approach the silo model if you cross this threshold.

6. **`SET search_path` not getting reset between requests in pooled connections.** Connection leaves the pool with the previous tenant's schema set; next user inherits it. Defense: always set search_path at the start of every request, or reset on connection release.

7. **Single Keycloak instance becoming a SPoF.** If Keycloak is down, no one can log in across all tenants. Add replicas, monitor health, cache token introspection.

8. **Per-tenant features that aren't actually per-tenant.** Feature flag service that doesn't know about tenant scope ends up applying flags inconsistently. Tenant-aware feature flag service from day one.

9. **Onboarding races.** Two onboarding requests for the same tenant name. Use unique constraints + idempotency keys.

10. **No way to "view as tenant" for support.** When a customer reports a bug, you need to reproduce against their data. Build an "impersonate tenant" admin tool with audit logging.

11. **Cross-tenant analytics joining live tables.** A heavy analytics query against the live multi-tenant DB drags down user-facing performance. Use a separate replica or warehouse for analytics.

12. **Schema names from user input without sanitization.** `tenant_'; DROP TABLE users; --` injected into `CREATE SCHEMA` is a catastrophe. Generate schema names server-side (UUID-based, not name-based).

13. **Long-running migration locks one tenant out for minutes.** Apply migrations in transactions with `lock_timeout`; do schema changes in expand/contract phases (Q19 of section 03).

14. **Compliance request not satisfiable.** Tenant asks for full data export; takes weeks of engineering because no one built it. Build it once, automate, document.

15. **No isolation testing.** Never tested that tenant A can't access tenant B. Adversarial tests on every PR.

**Follow-up 1: What does "first month of operations" usually surface?**
The shape of customers. You'll find that tenant size distribution is heavily skewed — a few enterprise tenants account for most data and load. Plan for that distribution: build noisy-neighbor defenses, consider per-tier resource pools, and have a path to upgrade a customer to a silo without a complete migration.

**Follow-up 2: When would you reconsider the entire multi-tenancy design?**
When (a) the operational cost of running thousands of schemas/DBs exceeds the engineering cost of moving to pool, (b) compliance requirements push everyone to silos anyway, (c) a major outage was caused by isolation failure and the post-mortem demands a stronger model, or (d) the product split into multiple very different offerings where one-size-fits-all isolation doesn't work.

---

*End of section 08. Next: AWS & DevOps (20 questions).*
