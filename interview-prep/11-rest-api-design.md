# 11 — REST API Design (10 Questions)

You built 20+ REST APIs with clean architecture, Swagger/OpenAPI docs, and idempotency keys for payment flows. These cover REST principles, idempotency, versioning, pagination, error handling, caching, and the design decisions interviewers probe.

---

### Q1. What are the core principles/constraints of REST?

**Answer:**
REST (Representational State Transfer) is an architectural style defined by Roy Fielding. The constraints:

1. **Client-server** — separation of concerns; client and server evolve independently.
2. **Stateless** — each request contains all info needed; server holds no client session state between requests. Enables horizontal scaling.
3. **Cacheable** — responses declare whether they're cacheable; clients/intermediaries can cache.
4. **Uniform interface** — resources identified by URLs, manipulated via standard methods (GET/POST/PUT/PATCH/DELETE), self-descriptive messages.
5. **Layered system** — client can't tell if it's talking to the origin server or an intermediary (LB, cache, gateway).
6. **Code on demand (optional)** — server can send executable code (rarely used).

**The Richardson Maturity Model** describes levels of REST adoption:
- **Level 0** — HTTP as a tunnel for RPC (one endpoint, POST everything).
- **Level 1** — Resources (multiple URLs, but still POST for everything).
- **Level 2** — HTTP verbs (GET/POST/PUT/DELETE used semantically, proper status codes). **Most "REST" APIs live here.**
- **Level 3** — HATEOAS (responses include links to related actions). Rarely fully implemented.

Most production APIs are pragmatically Level 2: resource URLs, correct verbs, correct status codes. That's a reasonable target.

**Resource-oriented design:**
- Nouns, not verbs: `/users/42/orders`, not `/getUserOrders?id=42`.
- Collections and items: `/orders` (collection), `/orders/123` (item).
- Hierarchy for relationships: `/users/42/orders`.

**Follow-up 1: Is "stateless" violated if I use a database?**
No. Statelessness refers to *session* state — the server doesn't remember anything about the client between requests. Each request carries its own auth (token) and context. The server using a shared database is fine; that's application state, not session state. The point: any server instance can handle any request.

**Follow-up 2: Why does HATEOAS rarely get implemented?**
Because clients usually hardcode the API structure anyway (they know `/orders/{id}` exists), so embedding links adds payload size without much benefit for typical SPAs and mobile apps. HATEOAS shines for truly evolvable, machine-navigable APIs (rare). Most teams find the cost/benefit doesn't justify full HATEOAS.

---

### Q2. How do HTTP methods relate to idempotency and safety?

**Answer:**
Two properties matter:

- **Safe** — doesn't modify server state (read-only).
- **Idempotent** — multiple identical requests have the same effect as one.

| Method | Safe | Idempotent | Typical use |
|--------|------|------------|-------------|
| GET | Yes | Yes | Read a resource |
| HEAD | Yes | Yes | Read headers only |
| OPTIONS | Yes | Yes | Capabilities / CORS preflight |
| PUT | No | Yes | Replace a resource (full update) |
| DELETE | No | Yes | Remove a resource |
| POST | No | No | Create / non-idempotent actions |
| PATCH | No | No (usually) | Partial update |

**Why it matters:**
- Idempotent methods can be safely retried by clients, proxies, browsers.
- POST is not idempotent — retry can create duplicates. This is why payment endpoints (POST) need idempotency keys (section 07 Q15).
- PUT is idempotent: `PUT /users/42 {name: 'Alice'}` repeated gives the same result. POST `/users {name: 'Alice'}` repeated creates multiple users.

**PATCH nuance:**
- PATCH *can* be idempotent if it sets absolute values (`{status: 'active'}`).
- PATCH is *not* idempotent if it does relative ops (`{$inc: {count: 1}}`).
- The spec doesn't require PATCH to be idempotent; design yours to be when possible.

**Code — idempotent PUT vs non-idempotent POST:**
```ts
// Idempotent — replacing
@Put(':id')
replace(@Param('id') id: string, @Body() dto: ReplaceUserDto) {
  return this.users.replace(id, dto);   // same result every time
}

// Non-idempotent — creating
@Post()
create(@Body() dto: CreateUserDto) {
  return this.users.create(dto);         // each call creates a new user
}
```

**Follow-up 1: How do you make POST idempotent for retries?**
Idempotency keys. Client generates a unique key per logical operation, sends in `Idempotency-Key` header. Server stores the key + result; retries with the same key return the stored result without re-executing. Standard for payment APIs (Stripe, your Dhee project).

**Follow-up 2: When would you use PUT vs PATCH?**
PUT replaces the entire resource — you send the full representation; missing fields are removed/defaulted. PATCH updates part — you send only the fields to change. Use PUT when the client has the full object; PATCH for partial updates (e.g., "just change the email"). PATCH is more bandwidth-efficient but slightly more complex to implement (JSON Merge Patch or JSON Patch formats).

---

### Q3. How do you version an API? Compare strategies.

**Answer:**

**URI versioning (`/v1/users`):**
- Most common, most explicit.
- Easy to test, cache, route.
- Con: "version the whole API" feel; URL changes when version bumps.

**Header versioning (`Accept: application/vnd.api.v2+json` or `X-API-Version: 2`):**
- Clean URLs (resource URL is stable).
- Con: harder to test (can't just paste in a browser), caching needs `Vary` header.

**Query parameter (`/users?version=2`):**
- Simple, but mixes versioning with filtering; can be missed.

**No versioning — evolve additively:**
- Add fields, never remove; add endpoints, never break existing.
- Works for a surprising amount of evolution.
- Version only when you genuinely break compatibility.

**Best practices:**
- **Version per breaking change, not per release.** Most changes are additive (new optional field, new endpoint) and don't need a new version.
- **Version individual resources** when possible, not the whole API at once. Run `/v1/users` and `/v2/orders` simultaneously.
- **Communicate deprecation** — `Sunset` header (RFC 8594), changelog, advance notice.
- **Keep old versions running** during a migration window.

**What counts as a breaking change:**
- Removing a field.
- Renaming a field.
- Changing a field's type.
- Changing validation rules to be stricter.
- Changing default behavior.
- Removing an endpoint.

**What's NOT breaking (additive):**
- Adding an optional request field.
- Adding a response field.
- Adding a new endpoint.
- Adding a new optional query parameter.

**Follow-up 1: How do you avoid maintaining two versions forever?**
Set a deprecation timeline. Announce v1 deprecation, give clients (say) 6 months, monitor v1 usage, reach out to remaining users, then remove. Don't let old versions linger indefinitely — each is maintenance burden and security surface.

**Follow-up 2: How do you handle a breaking change that affects only some clients?**
Feature negotiation rather than versioning. The client opts into the new behavior via a header or a flag. Or: deploy the new behavior behind a per-client config. This avoids a full version bump for a change that affects a subset. Trade-off: more conditional logic in the codebase.

---

### Q4. Pagination — offset vs cursor vs keyset. Which and why?

**Answer:**

**Offset pagination (`?page=3&limit=20` → `LIMIT 20 OFFSET 40`):**
- Simple, supports jumping to arbitrary pages.
- **Problem 1: slow for deep pages.** `OFFSET 100000` scans and discards 100k rows.
- **Problem 2: inconsistent under writes.** If a row is inserted while paginating, you can see duplicates or skip rows.
- Fine for small datasets and admin UIs where users rarely go deep.

**Cursor / keyset pagination (`?after=<last_id_or_value>`):**
- Use the last-seen sort value as the boundary for the next page.
- **Fast at any depth** — uses an index seek, not offset scan.
- **Consistent under writes** — you're anchored to a value, not a position.
- Con: no random page access ("jump to page 50"); only next/prev.

**Code — keyset pagination:**
```sql
-- First page
SELECT * FROM orders
WHERE tenant_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page: pass the last row's (created_at, id) as the cursor
SELECT * FROM orders
WHERE tenant_id = $1
  AND (created_at, id) < ($2, $3)       -- compound comparison
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```ts
// API response with opaque cursor
{
  "data": [...],
  "pageInfo": {
    "hasNextPage": true,
    "endCursor": "base64(2026-05-27T10:00:00Z,uuid)"   // opaque to client
  }
}
```

**Why the compound comparison `(created_at, id) < ($2, $3)`?**
Because `created_at` may not be unique. Two orders at the same timestamp need a tiebreaker (`id`). Without it, you'd skip or duplicate rows at timestamp boundaries.

**Follow-up 1: When is offset pagination actually fine?**
For small, bounded datasets (a few thousand rows max), or admin tools where users rarely page beyond the first few pages. The simplicity and "jump to page N" capability outweigh the performance concern when N is always small.

**Follow-up 2: How do you make the cursor opaque and tamper-resistant?**
Base64-encode the cursor data (sort value + id), optionally sign it (HMAC) so clients can't craft arbitrary cursors. Opaque cursors also let you change the underlying sort fields without breaking clients — they just pass back whatever you gave them.

---

### Q5. How do you design error responses? What status codes matter?

**Answer:**

**Use the right status code:**
- **2xx success** — 200 OK, 201 Created (with `Location` header), 202 Accepted (async), 204 No Content (delete).
- **4xx client errors:**
  - 400 Bad Request — malformed/invalid input.
  - 401 Unauthorized — not authenticated.
  - 403 Forbidden — authenticated but not allowed.
  - 404 Not Found — resource doesn't exist (or you don't want to reveal it exists).
  - 409 Conflict — state conflict (duplicate, version mismatch).
  - 422 Unprocessable Entity — semantically invalid (passed syntax but failed business rules).
  - 429 Too Many Requests — rate limited.
- **5xx server errors:**
  - 500 Internal Server Error — unexpected failure.
  - 502/503/504 — gateway / unavailable / timeout.

**Consistent error shape** — pick one and use it everywhere:
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "must be a valid email" },
      { "field": "age", "message": "must be at least 13" }
    ],
    "requestId": "req-abc-123",
    "timestamp": "2026-05-27T10:00:00Z"
  }
}
```

**Principles:**
- **Machine-readable code** (`VALIDATION_FAILED`) for programmatic handling, plus a human message.
- **Don't leak internals** — no stack traces, SQL errors, or file paths in production responses.
- **Include a request ID** so users can reference it in support tickets and you can find the corresponding logs/trace.
- **RFC 7807 (Problem Details)** is a standard format worth following: `application/problem+json` with `type`, `title`, `status`, `detail`, `instance`.

**Code — NestJS exception filter producing consistent errors:**
```ts
@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(err: unknown, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    const req = host.switchToHttp().getRequest();
    const status = err instanceof HttpException ? err.getStatus() : 500;
    const message = err instanceof HttpException ? err.getResponse() : 'INTERNAL';
    if (status >= 500) logger.error({ err, requestId: req.id }, 'server error');
    res.status(status).json({
      error: {
        code: this.codeFor(status),
        message: status >= 500 ? 'Internal server error' : message,
        requestId: req.id,
        timestamp: new Date().toISOString(),
      },
    });
  }
}
```

**Follow-up 1: 400 vs 422 — when each?**
400 for malformed requests (bad JSON, missing required field, wrong type) — the request itself is broken. 422 for well-formed requests that fail business validation (email already taken, age below minimum for this product). Some teams use 400 for everything; the distinction helps clients tell "fix your request format" from "your data violated a rule."

**Follow-up 2: Should a "not found" be 404 or 403?**
Depends on information disclosure. If revealing the resource exists is a security risk (e.g., another tenant's resource), return 404 — don't confirm existence. If existence is public but access is denied, 403 is more informative. For multi-tenant systems, often 404 for cross-tenant access to avoid leaking what exists.

---

### Q6. How do you design resource URLs and naming?

**Answer:**

**Conventions:**
- **Nouns, plural, lowercase, hyphenated** — `/job-applications`, not `/JobApplication` or `/getJobApplications`.
- **Collections and items** — `/orders` (collection), `/orders/{id}` (item).
- **Nesting for relationships** — `/users/{id}/orders` for "this user's orders."
- **Don't over-nest** — `/users/{id}/orders/{oid}/items/{iid}/reviews` is too deep. Flatten: `/order-items/{iid}/reviews` or `/reviews?orderItemId={iid}`.
- **Verbs only for actions that aren't CRUD** — `/orders/{id}/cancel`, `/users/{id}/reset-password`. These are RPC-ish but pragmatic.

**Query parameters for:**
- Filtering: `/orders?status=paid&minTotal=100`.
- Sorting: `/orders?sort=-createdAt` (- for descending).
- Pagination: `/orders?limit=20&after=cursor`.
- Field selection: `/orders?fields=id,total,status`.

**Examples:**
```
GET    /users                    list users
POST   /users                    create user
GET    /users/{id}               get one user
PUT    /users/{id}               replace user
PATCH  /users/{id}               update user fields
DELETE /users/{id}               delete user
GET    /users/{id}/orders        list this user's orders
POST   /users/{id}/orders        create an order for this user
POST   /orders/{id}/cancel       action: cancel (not pure CRUD)
```

**Consistency matters more than the specific convention.** Pick one (plural nouns, hyphenated, etc.) and apply it everywhere. A consistent imperfect convention beats an inconsistent "perfect" one.

**Follow-up 1: How do you handle actions that don't map to CRUD?**
Two options: (1) treat the action as a sub-resource — `POST /orders/{id}/cancellation`; (2) use a verb sub-path — `POST /orders/{id}/cancel`. Both are accepted pragmatic departures from pure REST. Choose one style and be consistent. For complex workflows, consider modeling state transitions as resources.

**Follow-up 2: Should IDs be sequential integers or UUIDs in URLs?**
UUIDs (or ULIDs) for public-facing URLs — sequential integers leak information (how many users you have, enable enumeration attacks). Internally, sequential IDs are fine for performance (section 03 Q18). Common pattern: bigserial PK internally, UUID/ULID for the public API.

---

### Q7. How do you implement filtering, sorting, and field selection?

**Answer:**

**Filtering** — query parameters mapped to WHERE clauses.
```
GET /orders?status=paid&minTotal=100&createdAfter=2026-01-01
```
```ts
const where = {};
if (q.status) where.status = q.status;
if (q.minTotal) where.total = { gte: Number(q.minTotal) };
if (q.createdAfter) where.createdAt = { gte: new Date(q.createdAfter) };
```

**Sorting** — `sort` param with field name; prefix `-` for descending.
```
GET /orders?sort=-createdAt,total
```
```ts
const allowedSort = ['createdAt', 'total', 'status'];   // allowlist!
const orderBy = q.sort.split(',').map(f => {
  const desc = f.startsWith('-');
  const field = desc ? f.slice(1) : f;
  if (!allowedSort.includes(field)) throw new BadRequestException(`cannot sort by ${field}`);
  return { [field]: desc ? 'desc' : 'asc' };
});
```

**Field selection (sparse fieldsets)** — `fields` param to return only requested fields.
```
GET /orders?fields=id,total,status
```
Reduces payload, useful for mobile. The implementation maps to a `SELECT` projection.

**Important: always allowlist** field names for sort and filter. User input naming a column can't be parameterized in SQL; an unsanitized sort field is a SQL injection vector. Validate against a known list.

**Advanced — query language:**
For complex filtering, some APIs adopt a mini-query syntax (`filter[status][eq]=paid`, RSQL, or GraphQL). Trade-off: power vs complexity. For most APIs, simple param mapping is enough.

**Follow-up 1: How do you prevent unbounded result sets?**
Always enforce a max `limit` (e.g., default 20, max 100). Reject or clamp requests asking for more. Without this, a client can request `limit=1000000` and OOM your service or hammer the DB. Pagination is mandatory, not optional.

**Follow-up 2: What's the danger of allowing arbitrary filter combinations?**
Some combinations have no supporting index → full table scans. A malicious or careless client filtering on an unindexed field at scale degrades the whole service. Defenses: (1) only allow filters on indexed columns; (2) query timeouts; (3) monitor slow queries per endpoint. For truly flexible querying, push to a search engine (Elasticsearch) rather than the OLTP DB.

---

### Q8. How does HTTP caching work — Cache-Control, ETags, conditional requests?

**Answer:**

**Cache-Control header** — directives controlling caching:
- `public` — cacheable by any cache (including CDN).
- `private` — only the end client may cache (not shared caches).
- `no-cache` — must revalidate with origin before using cached copy.
- `no-store` — don't cache at all (sensitive data).
- `max-age=3600` — cache fresh for 3600 seconds.
- `s-maxage` — max-age for shared caches (CDN) specifically.
- `immutable` — content never changes (versioned assets).

```
Cache-Control: public, max-age=31536000, immutable    # versioned static asset
Cache-Control: private, no-cache                       # user-specific data
Cache-Control: no-store                                # sensitive (don't cache)
```

**ETag (Entity Tag)** — a fingerprint of the resource (hash or version). Enables conditional requests.

**Conditional requests (revalidation):**
1. Server returns resource with `ETag: "abc123"`.
2. Client caches it. Next time, client sends `If-None-Match: "abc123"`.
3. If unchanged, server returns `304 Not Modified` (empty body — saves bandwidth).
4. If changed, server returns `200` with new content and new ETag.

```
# First request
GET /users/42
← 200 OK
  ETag: "v3-abc"
  { ...user data... }

# Subsequent request
GET /users/42
  If-None-Match: "v3-abc"
← 304 Not Modified              # no body; client uses cached copy
```

**Last-Modified / If-Modified-Since** — same idea using timestamps instead of ETags.

**Code — ETag in Express (built-in for static, manual for dynamic):**
```ts
@Get(':id')
async getUser(@Param('id') id: string, @Headers('if-none-match') etag: string, @Res() res) {
  const user = await this.users.findOne(id);
  const currentEtag = `"${user.version}"`;
  if (etag === currentEtag) return res.status(304).end();
  res.set('ETag', currentEtag);
  res.set('Cache-Control', 'private, no-cache');
  return res.json(user);
}
```

**Follow-up 1: When use ETag vs max-age?**
`max-age` for content you're confident stays fresh for a known duration (static assets, slow-changing data). ETags for content that changes unpredictably but where revalidation is cheap — you save bandwidth (304 is empty) even if you still make a request. Combine: short `max-age` + ETag for revalidation after expiry.

**Follow-up 2: What's the difference between strong and weak ETags?**
Strong ETag (`"abc"`) means byte-for-byte identical. Weak ETag (`W/"abc"`) means semantically equivalent (e.g., same data, different whitespace). Weak ETags are fine for caching but can't be used for range requests / partial content. Most APIs use weak ETags based on a version number or updated_at timestamp.

---

### Q9. How do you communicate rate limits to clients?

**Answer:**
When you rate-limit, tell clients their budget so they can back off gracefully rather than hammering and getting 429s.

**Standard response headers:**
```
X-RateLimit-Limit: 100           # max requests in the window
X-RateLimit-Remaining: 47        # how many left
X-RateLimit-Reset: 1717890000    # when the window resets (unix time)
```

Or the IETF draft standard (`RateLimit-*` without the `X-`):
```
RateLimit-Limit: 100
RateLimit-Remaining: 47
RateLimit-Reset: 60              # seconds until reset
```

**On 429 (Too Many Requests):**
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30                  # seconds to wait before retrying
Content-Type: application/json

{ "error": { "code": "RATE_LIMITED", "message": "Rate limit exceeded. Retry after 30s." } }
```

**`Retry-After`** can be seconds or an HTTP date. Well-behaved clients respect it.

**Code — setting headers in a NestJS guard:**
```ts
async canActivate(ctx: ExecutionContext): Promise<boolean> {
  const res = ctx.switchToHttp().getResponse();
  const key = this.keyFor(ctx);
  const result = await this.limiter.get(key);   // { remaining, limit, msBeforeNext }
  res.set('X-RateLimit-Limit', String(result.limit));
  res.set('X-RateLimit-Remaining', String(result.remaining));
  if (result.remaining < 0) {
    res.set('Retry-After', String(Math.ceil(result.msBeforeNext / 1000)));
    throw new HttpException('Rate limit exceeded', 429);
  }
  return true;
}
```

**Layered limits:**
- Per-IP for unauthenticated traffic (DDoS-ish protection).
- Per-user/API-key for authenticated.
- Per-tenant for multi-tenant fairness.
- Per-endpoint for expensive operations.

**Follow-up 1: Why use `Retry-After` instead of letting clients guess?**
Without it, clients retry blindly — often immediately, making the overload worse. `Retry-After` tells them exactly when capacity frees up, smoothing the retry storm. Good clients (and SDKs) honor it automatically.

**Follow-up 2: Should you return 429 or 503 when overloaded?**
429 = "you specifically exceeded your limit." 503 = "the whole service is overloaded / unavailable." 429 is per-client; 503 is system-wide. Use 429 for rate limiting, 503 (with `Retry-After`) for load shedding when the service itself is struggling regardless of individual client behavior.

---

### Q10. REST vs GraphQL vs gRPC, and the most common API design gotchas.

**Answer:**

**Quick comparison:**
- **REST** — resource-oriented, HTTP-native, universal, cacheable. Default for public APIs.
- **GraphQL** — client specifies exactly what data it needs in one query; great for diverse frontend needs; solves over/under-fetching. Cost: caching is harder, complexity, potential for expensive queries.
- **gRPC** — binary, typed, fast; best for internal service-to-service. Not browser-native (needs gRPC-Web).

**When each:**
- Public API, third-party developers → REST (familiar, tooling, cacheable).
- Complex frontend with varied data needs → GraphQL.
- Internal high-performance service calls → gRPC.

**Common REST API gotchas:**

1. **No pagination → unbounded responses.** Always paginate lists; enforce max limit.
2. **Inconsistent error shapes.** Standardize once.
3. **Wrong status codes.** 200 with `{"error": ...}` in the body confuses clients. Use the HTTP status.
4. **Leaking internals in errors.** Stack traces, SQL, file paths in production responses.
5. **No idempotency on POST.** Retries create duplicates.
6. **Versioning per release instead of per breaking change.** Version churn.
7. **N+1 in the API layer.** Endpoint that returns 100 items, each triggering a separate DB query.
8. **No rate limiting.** One client can take down the service.
9. **Chatty APIs.** Frontend needs 10 calls to render one screen. Consider aggregation / BFF.
10. **Inconsistent naming/casing.** `user_id` in one response, `userId` in another. Pick one (camelCase for JSON is common).
11. **Returning huge nested objects by default.** Use field selection / separate detail endpoints.
12. **Not documenting.** OpenAPI/Swagger spec is a contract; keep it in sync (your Swagger usage covers this).
13. **Breaking changes without notice.** Sunset headers, changelogs, migration guides.
14. **Timestamps without timezone.** Always ISO 8601 with timezone (UTC): `2026-05-27T10:00:00Z`.
15. **Exposing internal IDs that enable enumeration.** Use UUIDs for public IDs.

**Follow-up 1: How does OpenAPI/Swagger help beyond documentation?**
The spec is a machine-readable contract. From it you can: generate client SDKs, generate server stubs, run contract tests, validate requests/responses against the schema, mock the API for frontend dev before the backend exists, and power interactive docs. In NestJS, decorators generate the spec automatically (`@nestjs/swagger`), keeping docs in sync with code.

**Follow-up 2: When should I expose GraphQL on top of REST services?**
When you have multiple frontends with diverse data needs, and the over-fetching/under-fetching of REST is causing real pain (too many round trips, too much data). A GraphQL gateway can compose multiple REST/gRPC backends into one flexible query interface (BFF pattern). Don't add GraphQL for a single simple frontend — the complexity isn't justified.

---

*End of section 11. Next: Design Patterns & LLD (20 questions).*
