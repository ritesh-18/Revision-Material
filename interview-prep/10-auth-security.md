# 10 — Auth & Security (20 Questions)

You used JWT + OAuth 2.0 + Keycloak with RBAC across your SaaS multi-tenant project, and applied Helmet + rate limiting at Tibil. These cover authentication, authorization, OAuth flows, OIDC, Keycloak, the OWASP Top 10, and the production security gotchas every backend engineer must know cold.

---

### Q1. Session vs JWT authentication — when do you use each?

**Answer:**

**Session-based:**
- Server stores session state (usually in Redis or a DB). Client gets an opaque session ID in a cookie.
- Every request looks up the session by ID.
- **Pros:** Easy revocation (delete the session), small cookie size, server controls everything.
- **Cons:** Requires a session store; doesn't scale across services without shared state.

**JWT (JSON Web Token):**
- Server signs a token containing claims (user ID, roles, expiration). Client sends it on every request. Server verifies signature.
- Stateless on the server side — no lookup required.
- **Pros:** Scales horizontally; works across services without shared session store; ideal for microservices.
- **Cons:** Can't revoke before expiration without extra infra; larger headers; signature verification per request.

**Decision factors:**
- **Single monolith** → sessions are simpler.
- **Microservices / SPAs / mobile** → JWT.
- **Need instant revoke** → sessions or JWT + revocation list.
- **High volume APIs** → JWT (no DB hit per request).

**Hybrid pattern (common):** JWT for stateless API access + a refresh token stored server-side that *is* revocable.

**Code — session in Express:**
```js
app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET,
  cookie: { secure: true, httpOnly: true, sameSite: 'strict', maxAge: 7 * 24 * 3600 * 1000 },
  resave: false,
  saveUninitialized: false,
}));

req.session.userId = user.id;          // login
delete req.session;                     // logout
```

**Follow-up 1: Why not just always use JWT for everything?**
JWT's revocation problem is real. If a user's token leaks and is valid for 15 minutes, that's 15 minutes of access. With sessions, you delete the row. For an admin panel or anything sensitive, sessions are often the safer default.

**Follow-up 2: Are cookies more secure than localStorage for tokens?**
Yes. Cookies with `httpOnly` can't be read by JavaScript — XSS can't steal them. localStorage is plain JS-accessible; one XSS = token gone. Trade-off: cookies are vulnerable to CSRF unless you set `SameSite=strict` or use anti-CSRF tokens. Modern best practice: cookies for browser-facing apps, Authorization header for mobile/server-to-server.

---

### Q2. Walk me through JWT structure. What's in the header, payload, signature?

**Answer:**
A JWT is `header.payload.signature`, each part base64url-encoded.

**Header:**
```json
{
  "alg": "RS256",            // signing algorithm
  "typ": "JWT",
  "kid": "key-id-2026"       // optional: which key signed it (for rotation)
}
```

**Payload (claims):**
```json
{
  "sub": "user-42",          // subject (user ID)
  "iss": "auth.example.com", // issuer
  "aud": "api.example.com",  // audience
  "exp": 1717890000,         // expiration (unix seconds)
  "iat": 1717886400,         // issued at
  "nbf": 1717886400,         // not before
  "jti": "abc-123",          // unique token ID (for revocation lists)
  // custom claims
  "tenant_id": "acme",
  "roles": ["admin"]
}
```

**Signature:**
- `HMACSHA256(base64url(header) + "." + base64url(payload), secret)` for HS256.
- `RSASHA256(...)` with the private key for RS256.

The signature is what makes the token tamperproof — if anyone modifies the header or payload, the signature no longer matches.

**Code — signing and verifying with `jsonwebtoken`:**
```js
import jwt from 'jsonwebtoken';

// Sign
const token = jwt.sign(
  { sub: user.id, tenant_id: user.tenantId, roles: user.roles },
  privateKey,
  { algorithm: 'RS256', expiresIn: '15m', issuer: 'auth.example.com', audience: 'api.example.com' }
);

// Verify
const claims = jwt.verify(token, publicKey, {
  algorithms: ['RS256'],
  issuer: 'auth.example.com',
  audience: 'api.example.com',
});
```

**Standard claims to always validate:**
- `exp` — not expired (most libs do this automatically).
- `iss` — issued by who you expect.
- `aud` — intended for you (prevents tokens issued for service A being used at service B).
- `nbf` — current time ≥ this.

**Follow-up 1: Is JWT encrypted?**
No. JWT is signed, not encrypted. The payload is base64-encoded — readable by anyone with the token. If you need to hide data, use JWE (JSON Web Encryption) or just don't put secrets in the token. Treat the payload as public info that's just been tamperproofed.

**Follow-up 2: What's `kid` for and how do you use it?**
Key ID. Lets you rotate signing keys without downtime — old tokens signed with `kid=A` still verifiable while new ones use `kid=B`. The verifier looks up the right public key by `kid` from a JWKS endpoint. Critical for any system that runs long-lived tokens or has compliance-driven key rotation.

---

### Q3. HS256 vs RS256 — which should you use?

**Answer:**

**HS256 (HMAC-SHA256):**
- Symmetric. Same secret used to sign and verify.
- Fast.
- Anyone who can verify can also sign → bad for distributed verifier services.
- Use only when issuer and verifier are the same process.

**RS256 (RSA-SHA256):**
- Asymmetric. Private key signs; public key verifies.
- Public key can be freely distributed — verifiers don't need to be trusted with signing capability.
- 5–10x slower than HS256, but still fast enough (~1 ms per verify).
- The right choice for microservices.

**EdDSA / Ed25519:**
- Modern alternative. Smaller keys (32 bytes vs 2048+ bits), faster than RSA.
- Use if your library supports it.

**For your SaaS:** RS256 is correct. Keycloak holds the private key, signs tokens. Each microservice fetches Keycloak's public key (from `/.well-known/jwks.json`) and verifies. Compromised microservice can't issue new tokens.

**JWKS (JSON Web Key Set):**
A standard endpoint that lists public keys. Verifiers fetch and cache; rotation is handled by adding a new key with a new `kid` while keeping the old one for a transition period.

**Code — fetching JWKS for verification:**
```js
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: 'https://keycloak.example.com/realms/myrealm/protocol/openid-connect/certs',
  cache: true,
  cacheMaxAge: 10 * 60 * 1000,
});

function getKey(header, cb) {
  client.getSigningKey(header.kid, (err, key) => cb(err, key.getPublicKey()));
}

jwt.verify(token, getKey, { algorithms: ['RS256'] }, (err, claims) => { ... });
```

**Follow-up 1: Why is the "alg: none" attack still talked about?**
JWT spec includes `alg: none` (no signature). A buggy verifier that doesn't pin allowed algorithms could accept an unsigned token as "valid." Mitigation: always pass `algorithms: ['RS256']` explicitly to verification — never trust the token's own `alg` field.

**Follow-up 2: What's "algorithm confusion" attack?**
Send a token with `alg: HS256` to a server that uses RS256. If the server uses the *public key* as the HS256 secret, the attacker can forge tokens with the publicly-known key. Fix: never auto-detect algorithm; pin it.

---

### Q4. JWT pitfalls and best practices.

**Answer:**

**Common mistakes:**

1. **Not pinning `algorithms`.** Verifier accepts any alg the token claims, including `none`. Fix: explicit list.

2. **No expiration.** Tokens live forever; one leak = permanent access. Fix: short `exp` (15 min for access tokens).

3. **No revocation strategy.** User logs out / changes password — old tokens still valid. Fix: refresh-token-only revocation, or `jti` blocklist.

4. **Putting secrets in payload.** Payload is base64, not encrypted. Anyone with the token reads it. Fix: only put IDs and minimal claims; lookup details server-side.

5. **Token in URL / referer.** Token in query string ends up in server logs, browser history. Fix: Authorization header only.

6. **Not validating `aud`.** Token issued for service A can be replayed at service B. Fix: every service validates audience.

7. **Storing tokens in localStorage.** XSS = stolen tokens. Fix: httpOnly cookies, or short-lived tokens + secure refresh flow.

8. **JWT for sessions instead of access tokens.** "Why does my logged-in session randomly expire?" Because you used JWT exp as session lifetime. Use refresh tokens to keep sessions alive while access tokens expire frequently.

9. **Forgetting clock skew.** Server-A's clock is 5s ahead of server-B's; tokens validated on B fail "nbf" check. Fix: allow a few seconds skew (`clockTolerance` in libs).

10. **Reusing the same JWT for multiple purposes** — API access, email verification, password reset. Don't. Use single-purpose, short-lived tokens for state-changing flows.

**Best-practice access token:**
- Short-lived (15 min).
- Signed with asymmetric algorithm (RS256/EdDSA).
- Minimal payload (sub, tenant, roles, iat, exp).
- HTTPS-only transport.
- Validated against `iss`, `aud`, `exp`, and signature on every request.

**Follow-up 1: What's the right access-token lifetime?**
Short. 5–15 minutes is the common range. The shorter, the smaller the leak window. Pair with a longer-lived refresh token (days to weeks) that's revocable. Users don't notice — refresh happens transparently.

**Follow-up 2: How do you handle "log out everywhere"?**
With pure JWTs you can't, until tokens expire. Solutions: (1) refresh-token blocklist + short access tokens — within max-15-min the access is gone; (2) `jti` blocklist for active access tokens — costs a Redis check per request; (3) Keycloak revokes the user's sessions — same as (1) but server-side.

---

### Q5. Refresh tokens — how do they work and how do you rotate them?

**Answer:**

**Refresh tokens** are long-lived credentials that can be exchanged for a fresh access token, without the user re-authenticating.

**Why two tokens?**
- Access token = short-lived, used per-request. Tiny leak window.
- Refresh token = long-lived, used only at the auth server. Stored more carefully (httpOnly cookie or secure storage); never sent to API services.

**Standard flow:**
1. User logs in → server returns access (15 min) + refresh (30 days).
2. Client uses access token for API calls.
3. Access token expires; client sends refresh token to `/auth/refresh`.
4. Server validates refresh token, issues a new access token (and optionally a new refresh).
5. On logout / suspected compromise, server invalidates the refresh token.

**Rotation:**
- Issue a new refresh token on each use, invalidating the old one.
- Detects theft: if an old refresh token is presented after rotation, someone has it but the legitimate user has the newer one. Invalidate the entire chain.

**Code — refresh endpoint:**
```ts
@Post('refresh')
async refresh(@Body() body: { refreshToken: string }) {
  const stored = await db.refreshTokens.findUnique({
    where: { token: hash(body.refreshToken) },
  });
  if (!stored || stored.expiresAt < now() || stored.revokedAt) {
    throw new UnauthorizedException();
  }
  if (stored.usedAt) {
    // Token reuse detected — someone has the old one
    await db.refreshTokens.updateMany({
      where: { userId: stored.userId, revokedAt: null },
      data: { revokedAt: now() },
    });
    throw new UnauthorizedException('reuse detected');
  }
  // Mark old as used, issue new pair
  await db.refreshTokens.update({ where: { id: stored.id }, data: { usedAt: now() } });
  const newRefresh = await issueRefreshToken(stored.userId);
  const newAccess = signAccessToken(stored.userId);
  return { accessToken: newAccess, refreshToken: newRefresh };
}
```

**Storage:**
- Server-side: hash the refresh token; store the hash + user ID + expiration + status. Treat like a password.
- Client-side: secure storage — httpOnly cookie for web, Keychain/Keystore for mobile.

**Follow-up 1: Why hash refresh tokens server-side?**
Same reason as passwords: if your DB is dumped, attacker has a list of hashed tokens, not the tokens themselves. Brute-forcing them is hard (tokens are long random strings). Treat refresh tokens with the same care as passwords.

**Follow-up 2: How does reuse detection actually work?**
When you rotate, mark the old refresh token as "used but not revoked." If anyone presents a "used" token, you know: legitimate user has the new one; this presenter has the old one (because the response with the new one was intercepted, or because the legitimate user has been compromised). Invalidate everything for that user; force re-login. Best-practice; OAuth 2.1 codifies this.

---

### Q6. What are the main OAuth 2.0 flows and when do you use each?

**Answer:**
OAuth 2.0 defines several flows ("grant types") for different scenarios.

**Authorization Code (with PKCE) — the main one:**
- For web apps and SPAs/mobile.
- User redirects to auth server, logs in, redirects back with a code, app exchanges code for tokens.
- PKCE (Proof Key for Code Exchange) prevents code interception in public clients.

**Client Credentials:**
- For machine-to-machine.
- Client authenticates with client ID + secret; gets a token.
- No user involved.

**Resource Owner Password Credentials (DEPRECATED):**
- User gives username+password to the client app; app sends to auth server.
- Defeats the point of OAuth (third-party app has the password).
- Don't use except for legacy migration.

**Implicit Flow (DEPRECATED):**
- For SPAs before PKCE existed.
- Token returned directly in URL fragment.
- Replaced by Authorization Code + PKCE.

**Device Code:**
- For devices without a browser (TVs, IoT).
- Device shows a code; user enters it on a separate device's browser.

**Refresh Token:**
- Not a primary flow; used to refresh an access token.

**Decision tree:**
- Browser app, server-side rendered → Authorization Code (PKCE optional).
- SPA, mobile, any public client → Authorization Code + PKCE.
- Service ↔ service → Client Credentials.
- Smart TV / CLI tool → Device Code.

**Follow-up 1: What's a "public" vs "confidential" client?**
A confidential client can keep a secret (server-side backend). A public client cannot (mobile app, SPA — the secret would be visible in code/binary). Public clients use PKCE; confidential clients can use their secret instead. The distinction drives security choices.

**Follow-up 2: What's the difference between OAuth and OpenID Connect?**
OAuth 2.0 is *authorization* — "this client is allowed to access this resource." It says nothing about identity. OpenID Connect (OIDC) is built on top of OAuth — it adds an `id_token` (a JWT with user identity claims) and standardizes user info endpoints. When people say "log in with Google," they mean OIDC, not pure OAuth.

---

### Q7. Walk me through Authorization Code with PKCE step by step.

**Answer:**

**PKCE (Proof Key for Code Exchange)** secures the Authorization Code flow for public clients. The client generates a secret per flow; only the client that started the flow can complete it.

**Step by step:**

1. **Client generates a code verifier** — random 43–128 char string.
2. **Client computes code challenge** — `SHA256(verifier)` base64url-encoded.
3. **Client redirects user to auth server** with the code challenge:
```
GET https://auth.example.com/authorize?
  response_type=code&
  client_id=spa-client&
  redirect_uri=https://app.example.com/cb&
  scope=openid profile email&
  state=random_csrf_token&
  code_challenge=BASE64URL(SHA256(verifier))&
  code_challenge_method=S256
```
4. **User authenticates and consents** at the auth server.
5. **Auth server redirects back** with a code:
```
https://app.example.com/cb?code=ABC123&state=random_csrf_token
```
6. **Client validates `state`** matches what it sent (CSRF protection).
7. **Client exchanges code for tokens**, including the original verifier:
```
POST https://auth.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=ABC123&
redirect_uri=https://app.example.com/cb&
client_id=spa-client&
code_verifier=ORIGINAL_VERIFIER
```
8. **Auth server hashes the verifier**, compares to the stored challenge. Matches → issue tokens. Doesn't → reject.

**Why this is secure:**
- An attacker who intercepts the auth code can't use it without the verifier.
- The verifier never leaves the client.
- The state parameter prevents CSRF attacks on the redirect.

**Follow-up 1: Why is PKCE now recommended even for confidential clients?**
OAuth 2.1 (the simplification spec) extends PKCE to all clients. Reason: PKCE provides an extra layer beyond client secret. Even if the client secret leaks (logs, source code, environment), PKCE still requires the original verifier — closing the gap.

**Follow-up 2: What attacks does the `state` parameter prevent?**
CSRF on the OAuth callback. Without `state`, an attacker tricks the victim into completing an OAuth flow with the attacker's account, then the victim's app is now logged in as the attacker (and posts the victim's data there). `state` is a random token the client generates, checks on callback. Standard CSRF defense.

---

### Q8. OAuth vs OIDC — what does OIDC add?

**Answer:**

**OAuth 2.0** = authorization framework. Issues access tokens. Says nothing about who the user is.

**OIDC** (OpenID Connect) = identity layer on top of OAuth 2.0. Adds:
- **`id_token`** — a JWT containing user identity claims (sub, email, name, picture, etc.).
- **`/userinfo` endpoint** — standardized way to get more profile data.
- **Standardized scopes** — `openid`, `profile`, `email`, etc.
- **Discovery endpoint** — `/.well-known/openid-configuration` describes the provider's endpoints and capabilities.

**Use OAuth alone when:**
- You only need authorization, not identity (rare in modern apps).
- e.g., a back-end service authorizing another back-end service via client credentials.

**Use OIDC when:**
- You're implementing login.
- "Log in with Google / GitHub / Microsoft" → always OIDC.

**Code — verifying an OIDC id_token:**
```js
import jwt from 'jsonwebtoken';
import jwksClient from 'jwks-rsa';

const client = jwksClient({
  jwksUri: 'https://accounts.google.com/.well-known/openid-configuration',
});

function getKey(header, cb) {
  client.getSigningKey(header.kid, (err, key) => cb(err, key.getPublicKey()));
}

const idTokenClaims = await new Promise((resolve, reject) =>
  jwt.verify(idToken, getKey, {
    algorithms: ['RS256'],
    issuer: 'https://accounts.google.com',
    audience: 'YOUR_CLIENT_ID',
  }, (err, claims) => err ? reject(err) : resolve(claims))
);

const user = await findOrCreateUser({
  email: idTokenClaims.email,
  googleId: idTokenClaims.sub,
});
```

**Follow-up 1: When can you stop after the id_token without making more calls?**
When the id_token has all the claims you need. Many providers (Google, Microsoft, Keycloak) include enough basic info (email, name, sub) that no follow-up `/userinfo` call is needed. Save the round trip.

**Follow-up 2: Should I use OIDC or implement my own login?**
Always OIDC for federated login ("sign in with X") or when you have an existing IdP (Keycloak, Okta). Roll your own only if you have no identity provider and the user base is small/simple. Modern apps almost universally use OIDC, even for first-party auth — via Keycloak, Auth0, Cognito, Okta, etc.

---

### Q9. How does Keycloak work? Walk me through your integration.

**Answer:**

**Keycloak** is an open-source identity provider. It speaks OIDC (and SAML, LDAP, Kerberos). Manages users, groups, roles, sessions; supports MFA, social login, identity federation.

**Components:**
- **Realm** — top-level isolation. One realm per environment or per major tenant.
- **Client** — an application that authenticates through Keycloak (your NestJS API, your frontend).
- **User** — an end user.
- **Role** — a permission tag (realm-level or client-level).
- **Group** — a collection of users with shared roles.
- **Authentication flow** — configurable steps (password, MFA, captcha, etc.).
- **Identity provider** — external IdPs (Google, Azure AD, SAML) to federate from.

**Your integration:**

1. **Realm setup** — one realm per environment (`prod`, `staging`).
2. **Frontend client** (public client, type `public`, with PKCE).
3. **Backend client** (confidential client for admin operations, machine-to-machine).
4. **Role hierarchy** — `admin`, `manager`, `viewer`. Mapped to user groups per tenant.
5. **Tenant claim mapper** — adds `tenant_id` to the JWT based on the user's group membership.

**Login flow:**
```
User → SPA → Keycloak (login page) → SPA receives auth code → SPA exchanges for tokens
                                                                    │
                                                                    ▼
                                                            Access + Refresh + ID tokens
SPA stores tokens; sends access token in Authorization header on every API call
Backend verifies access token (RS256 + JWKS) and reads tenant_id + roles from claims
```

**Code — NestJS verifying Keycloak token:**
```ts
@Injectable()
export class KeycloakStrategy extends PassportStrategy(Strategy, 'keycloak') {
  constructor(cfg: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKeyProvider: passportJwtSecret({
        cache: true,
        rateLimit: true,
        jwksRequestsPerMinute: 5,
        jwksUri: `${cfg.get('KC_URL')}/realms/${cfg.get('KC_REALM')}/protocol/openid-connect/certs`,
      }),
      issuer: `${cfg.get('KC_URL')}/realms/${cfg.get('KC_REALM')}`,
      algorithms: ['RS256'],
    });
  }

  validate(payload: any) {
    return {
      id: payload.sub,
      email: payload.email,
      tenantId: payload.tenant_id,
      roles: payload.realm_access?.roles ?? [],
    };
  }
}
```

**Follow-up 1: How do you handle "user provisioning" — when a user logs in via SSO and isn't in your DB yet?**
Just-In-Time (JIT) provisioning. First time a user appears in your app with a valid Keycloak token, create their record in your DB using their Keycloak `sub` as the key. Map their roles. Subsequent logins find the existing record. No manual admin onboarding step needed.

**Follow-up 2: What's the trade-off of running Keycloak vs using a managed service like Auth0/Okta?**
Keycloak: self-hosted, full control, no per-user fees (the main reason it's popular for B2B SaaS at scale). Costs: you operate it, including HA, patching, upgrades, backup. Auth0/Okta: managed, polished UX, fast to set up; per-user pricing gets expensive at scale. Choose Keycloak when you have ops capacity and significant user counts; managed for startups and prototypes.

---

### Q10. RBAC vs ABAC vs ReBAC — what's the difference?

**Answer:**

**RBAC (Role-Based Access Control):**
- Users are assigned roles; roles have permissions.
- Permissions checked against the user's roles.
- Simple: "admin can do X, viewer cannot."
- Limitation: doesn't scale to complex relationships (per-resource permissions, hierarchies).

**ABAC (Attribute-Based Access Control):**
- Permission decisions based on attributes of subject, resource, action, and environment.
- Policy: "user can edit document if user.department == document.department AND time < 18:00."
- Flexible; can express any rule.
- Complexity: policies grow large; harder to audit "who can do what?"

**ReBAC (Relationship-Based Access Control):**
- Permissions derived from relationships between entities.
- "User is owner of document X" → can edit. "User belongs to group Y which has read access" → can read.
- Used by Google (Zanzibar paper), Auth0 FGA, OpenFGA, SpiceDB.
- Powerful for complex permission graphs (sharing, groups, hierarchies).

**For your SaaS:**
- RBAC at the tenant level (admin/manager/viewer in tenant X).
- ABAC-flavored conditions for cross-tenant operations or special cases (super-admin only on production data).
- ReBAC if you build collaborative features (document sharing, team-based access).

**Code — simple RBAC check:**
```ts
function can(user, action, resource) {
  const role = user.roleInTenant(resource.tenantId);
  return PERMISSIONS[role]?.includes(action);
}
```

**Code — ABAC-style with conditions:**
```ts
function can(user, action, resource, ctx) {
  if (action === 'delete' && resource.protected) return user.hasRole('admin');
  if (action === 'read') return user.tenantId === resource.tenantId;
  // ... more rules
}
```

**Follow-up 1: When do I need ReBAC?**
When permissions are graph-shaped: "user is in team that owns folder that contains document." RBAC would explode (a role per folder per team), ABAC would need brittle rules. ReBAC stores the relationship and queries it directly. Necessary for Google-Docs-style collaborative apps; overkill for most SaaS.

**Follow-up 2: Where do you actually enforce authorization?**
Multiple places:
1. **Gateway/middleware** — coarse (is this endpoint reachable?).
2. **Route guards** — has the right role for this endpoint?
3. **Service layer** — does this user have access to *this specific resource*? (e.g., "is order X in user's tenant?")
4. **DB query** — every query includes tenant_id (or schema, in your case).

Defense in depth: a bug in any one layer doesn't cause data leakage.

---

### Q11. What does Helmet.js do? Walk me through the important headers.

**Answer:**
**Helmet** is a middleware that sets a stack of security-related HTTP headers. It defaults to sensible values; you adjust for your app's needs.

**Key headers:**

**`Content-Security-Policy` (CSP):**
- Whitelists where scripts, styles, images, etc. can load from.
- Best defense against XSS.
```
Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com; img-src 'self' data:;
```

**`Strict-Transport-Security` (HSTS):**
- Tells browsers "always use HTTPS for this domain" for N seconds.
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**`X-Content-Type-Options: nosniff`:**
- Prevents MIME-sniffing — browser uses declared Content-Type only.

**`X-Frame-Options: DENY` (or CSP `frame-ancestors`):**
- Prevents clickjacking — your site can't be embedded in iframes.

**`Referrer-Policy: strict-origin-when-cross-origin`:**
- Controls how much of the referrer URL is sent.

**`Permissions-Policy`:**
- Disable browser features by default (camera, microphone, geolocation, etc.).

**Code — Helmet in NestJS/Express:**
```js
import helmet from 'helmet';
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", 'cdn.example.com'],
      imgSrc: ["'self'", 'data:', 'images.example.com'],
      connectSrc: ["'self'", 'api.example.com'],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
}));
```

**For API-only services**, many headers are less critical (no browser rendering JS), but HSTS and X-Content-Type-Options are still worth having.

**Follow-up 1: Why is CSP so important?**
It's the only header that meaningfully reduces XSS impact. With a strict CSP, even if an attacker injects a `<script>`, the browser refuses to execute it because it's not from an allowed source. Combined with input sanitization and output encoding, CSP makes XSS exploitation extremely difficult.

**Follow-up 2: What's "unsafe-inline" and why avoid it?**
A CSP directive value that allows inline scripts/styles. Defeats the purpose of CSP. Move all JS to external files; use nonces or hashes for unavoidable inline. Same with styles. Yes, it's a pain; yes, it's worth it.

---

### Q12. How do you defend against CSRF?

**Answer:**
**CSRF (Cross-Site Request Forgery)** = attacker tricks an authenticated user's browser into making a request to your site. Because the browser sends cookies automatically, the request looks legitimate from your server's perspective.

Example: user is logged into bank.com. Attacker's malicious site has `<img src="https://bank.com/transfer?to=attacker&amount=1000">`. Browser sends the request with the user's cookies; bank processes it.

**Defenses (use multiple):**

1. **SameSite cookies:**
```
Set-Cookie: session=abc; SameSite=Strict; Secure; HttpOnly
```
`SameSite=Strict` — cookie not sent on cross-site requests. `Lax` (browser default now) — sent only for top-level navigation. Modern browsers default to Lax, which prevents most CSRF.

2. **CSRF tokens (synchronizer token):**
- Server generates a random token per session, embeds in forms.
- Client sends token in form data or custom header.
- Server validates token matches session.
- Library: `csurf` (deprecated; `csrf-csrf` or built-in framework support).

3. **Custom request header:**
- Send `X-Requested-With: XMLHttpRequest` from your SPA. Browsers don't let cross-site requests set custom headers without preflight (CORS).
- Server requires the header.

4. **Origin / Referer header check:**
- Reject requests whose `Origin` doesn't match your domain.

**For APIs using JWT in Authorization header**, CSRF is mostly moot — browser doesn't auto-send the header for cross-site requests. CSRF is a cookie-auth problem.

**Code — combining SameSite + token:**
```js
app.use((req, res, next) => {
  if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(req.method)) {
    const headerToken = req.headers['x-csrf-token'];
    if (headerToken !== req.session.csrfToken) return res.status(403).end();
  }
  next();
});

// Set the cookie correctly
app.use(cookieSession({
  name: 'session',
  secret: process.env.SESSION_SECRET,
  sameSite: 'strict',
  secure: true,
  httpOnly: true,
}));
```

**Follow-up 1: Is CSRF still a concern with modern SameSite defaults?**
Less, but yes. Default `SameSite=Lax` covers most. But: (1) form submissions via top-level navigation can still trigger Lax-cookied requests in some browsers; (2) GET requests are typically exempt and can still leak via image tags etc.; (3) older browsers without Lax default. Defense in depth: SameSite + CSRF token for sensitive operations.

**Follow-up 2: When is CSRF irrelevant?**
For APIs that use Authorization header (Bearer token) and don't accept cookies. Browsers don't auto-attach Authorization headers cross-site, so CSRF can't work. Mobile apps and SPAs that pass JWTs via header are CSRF-immune by design.

---

### Q13. CORS — what does it do and what does it NOT do?

**Answer:**
**CORS (Cross-Origin Resource Sharing)** is a browser security mechanism. By default, JavaScript can't read responses from a different origin (protocol + domain + port). CORS lets the server opt in to allowing certain cross-origin reads.

**It does:**
- Restrict which origins can call your API from a browser.
- Distinguish "simple" (GET, POST with limited content types) from "preflighted" (OPTIONS request first to check permission) requests.

**It does NOT:**
- Stop server-to-server requests (no browser, no CORS).
- Stop the request from being sent — the request *goes through*; CORS only stops the browser from giving the response to the JS.
- Protect against CSRF (separate problem).
- Prevent unauthorized API access (use real auth).

**Preflight (OPTIONS) flow:**
```
Browser → OPTIONS /api/users
            Origin: https://app.example.com
            Access-Control-Request-Method: POST
            Access-Control-Request-Headers: Content-Type, Authorization
        ←
            Access-Control-Allow-Origin: https://app.example.com
            Access-Control-Allow-Methods: GET, POST
            Access-Control-Allow-Headers: Content-Type, Authorization
            Access-Control-Allow-Credentials: true
            Access-Control-Max-Age: 86400        # cache preflight

Browser → POST /api/users
            Authorization: Bearer ...
```

**Configuration:**
```js
import cors from 'cors';
app.use(cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
}));
```

**Anti-pattern:** `Access-Control-Allow-Origin: *` with `credentials: true` — browsers reject this combination. And `*` alone means anyone's JS can call your API.

**Follow-up 1: My CORS works in dev but fails in prod — what's wrong?**
Usually one of: (1) origin list doesn't include the actual prod URL; (2) prod uses different port; (3) preflight isn't being handled (some setups bypass middleware for OPTIONS); (4) auth middleware runs before CORS and rejects the preflight (CORS first, auth second).

**Follow-up 2: How do you handle CORS for many tenant subdomains?**
Dynamic origin function. `origin` can be a function that inspects the request and returns true/false. Verify the origin matches a pattern (e.g., `*.example.com`) and allow. Don't just reflect the origin back blindly — that opens you to attacks. Whitelist patterns or look up tenants in DB.

---

### Q14. SQL injection — how does it work and how do you prevent it?

**Answer:**
**SQL injection** happens when user input is concatenated into SQL without proper escaping. Attacker crafts input that closes the intended query and adds their own.

**Classic example:**
```js
// BAD
const q = `SELECT * FROM users WHERE email = '${req.body.email}'`;
db.query(q);

// attacker sends email = "' OR '1'='1"
// Resulting SQL: SELECT * FROM users WHERE email = '' OR '1'='1'
// Returns every user.
```

Worse cases: `'; DROP TABLE users; --`. Information disclosure: `' UNION SELECT password FROM admins --`.

**Prevention:**

**1. Parameterized queries / prepared statements** — the only real fix.
```js
// Good
db.query('SELECT * FROM users WHERE email = $1', [req.body.email]);
```
The driver sends the SQL and the parameter separately; the DB binds the parameter as a literal value, never as SQL code. Attacker input is just a string, not executable.

**2. ORM / query builders** — TypeORM, Prisma, Knex all use parameterized queries by default. As long as you don't drop to raw SQL with template strings, you're safe.

**3. Never trust input** — even if you "validate" the format, treat all user input as potentially malicious; use parameters.

**Stored procedures / functions** — depending on language, often help but can still be vulnerable if they dynamically build queries inside.

**Allowlists for dynamic identifiers** — column names, table names can't be parameterized. If user input names a column for sorting, validate against an allowlist:
```js
const allowedSort = ['name', 'createdAt', 'email'];
if (!allowedSort.includes(req.query.sort)) throw new BadRequestException();
// then it's safe to interpolate
```

**Follow-up 1: ORMs claim to "prevent SQL injection." Are they really safe?**
For typical use, yes — they use parameterized queries. But raw query escapes are still a foot-gun:
- Prisma: `$queryRawUnsafe('...')` — unsafe.
- TypeORM: `.query('SELECT * FROM users WHERE name = ' + name)` — unsafe.
- The named "Raw" or "Unsafe" methods exist for power users; concatenating into them defeats the protection.

**Follow-up 2: Are NoSQL databases immune?**
No. **NoSQL injection** in Mongo: attacker passes `{ $ne: null }` as a field value to bypass checks (e.g., `{ password: { $ne: null } }` matches every user). Validate types strictly and reject objects where you expect strings.

---

### Q15. XSS — types, prevention, and real defenses.

**Answer:**
**XSS (Cross-Site Scripting)** = attacker's JavaScript runs in another user's browser, on your domain. With that, attacker can steal cookies, perform actions as the user, exfiltrate data.

**Three flavors:**

**Reflected XSS:**
- Malicious input in URL/form, server reflects it back unescaped.
- Example: `search?q=<script>...</script>` and your page shows `Results for: <script>...</script>`.
- Requires luring victim to a crafted URL.

**Stored XSS:**
- Malicious input saved in DB (comment, profile field), served to many users.
- Most dangerous; one payload affects everyone who views the page.

**DOM-based XSS:**
- Client-side JS unsafely uses URL/data: `document.write(location.hash)` or innerHTML with user data.
- Server doesn't even see the payload; it's processed entirely in the browser.

**Prevention:**

1. **Output encoding** — when rendering user data, escape based on context.
   - HTML body: `&`, `<`, `>`, `"`, `'` → entities.
   - HTML attributes: more aggressive escaping.
   - JavaScript context: completely different rules (don't put user data in JS strings without serializing as JSON).
   - URL context: percent-encoding.
   - Modern frameworks (React, Vue) do this by default — `{userInput}` is safe; `dangerouslySetInnerHTML` is not.

2. **Don't use `innerHTML` with user data.** Use `textContent` or framework binding.

3. **Sanitize HTML** if you must accept HTML input (rich text editor). Use DOMPurify; never roll your own.

4. **Content Security Policy** — last line of defense (Q11). Strict CSP blocks inline scripts entirely.

5. **HttpOnly cookies** — XSS can't steal them. Damage is contained.

6. **Validate input** — reject obviously malicious patterns. But don't rely on this alone; sophisticated payloads bypass naive filters.

**Follow-up 1: Why is `dangerouslySetInnerHTML` in React called "dangerous"?**
Because it tells React not to escape the content. If that content includes `<script>` or HTML attributes with `javascript:` URLs, you've enabled XSS. The naming is deliberate — make developers stop and think. Use only with DOMPurify-cleaned input.

**Follow-up 2: What's the most reliable XSS test?**
Try inserting `"><svg/onload=alert(1)>` into every text field, every URL parameter. If an alert pops in any rendered context, you have XSS. Modern frameworks handle this in normal rendering paths; the bugs are usually in custom rendering, raw DOM manipulation, or markdown/HTML pipelines.

---

### Q16. How do you store passwords correctly?

**Answer:**
**Never store passwords in plaintext.** Hash them. And not just hash — use a *slow* password-hashing function with a salt.

**Why slow?**
Fast hash (SHA-256) can be brute-forced billions of times per second on a GPU. A slow hash (bcrypt, argon2) deliberately takes ~100ms per check — millions of times slower, making offline brute-force impractical.

**Algorithms:**
- **bcrypt** — battle-tested, widely supported. Cost factor (default 10–12) tunes the slowness.
- **argon2** — newer, winner of the Password Hashing Competition. Three variants; argon2id is the recommended one. Resistant to GPU and ASIC attacks.
- **scrypt** — also good; memory-hard.
- **PBKDF2** — older, still acceptable with enough iterations (NIST recommends 600K+).

**Avoid:**
- Plain SHA / MD5 — too fast.
- Salted SHA — still too fast.

**Salt** = per-password random data, prevents rainbow tables. The hashing function handles it; bcrypt embeds the salt in the output.

**Code — bcrypt in Node:**
```js
import bcrypt from 'bcrypt';

// Sign up
const hash = await bcrypt.hash(password, 12);
await db.user.create({ data: { email, passwordHash: hash } });

// Log in
const user = await db.user.findUnique({ where: { email } });
const ok = await bcrypt.compare(password, user.passwordHash);
if (!ok) throw new UnauthorizedException();
```

**Argon2:**
```js
import argon2 from 'argon2';
const hash = await argon2.hash(password, { type: argon2.argon2id, memoryCost: 65536, timeCost: 3, parallelism: 4 });
const ok = await argon2.verify(hash, password);
```

**Password reset flow:**
1. User requests reset; you email a single-use, expiring token.
2. User clicks link, enters new password.
3. New password is hashed and stored.
4. Invalidate all the user's active sessions/tokens.

**Follow-up 1: What's the right bcrypt cost factor?**
Tune so one hash takes ~250ms on your hardware. Too low = vulnerable to brute force; too high = users wait long to log in / sign up. Cost 12 is a good default in 2026; revisit every 1–2 years as hardware improves.

**Follow-up 2: What if my user DB is dumped?**
With proper password hashing: attacker has hashes but can't trivially recover passwords. For weak passwords ("password123"), they'll get them (dictionary attack); for strong unique passwords, very few will fall. This is why force a minimum length (12+), check against breach databases (haveibeenpwned API), and offer/require MFA.

---

### Q17. How do you manage secrets in production?

**Answer:**

**Don't:**
- Commit secrets to git.
- Bake secrets into Docker images.
- Pass secrets via command-line args (visible in `ps`).
- Log secrets.
- Store secrets in plaintext on disk.

**Do:**
- **Secrets manager** — AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager, Azure Key Vault, Doppler, 1Password Secrets Automation.
- **Inject at runtime** via env vars (set by the orchestrator), files (mounted volumes), or fetched on startup.
- **Rotate regularly** — automated rotation reduces blast radius.
- **Audit access** — who pulled which secret when.
- **Least privilege** — each service only has access to the secrets it needs.

**Workflow:**
1. Secret created in Vault/Secrets Manager.
2. Application's IAM role has permission to read that specific secret.
3. On startup, app fetches the secret using its role credentials.
4. Cached in memory; re-fetched on rotation event or after TTL.

**Code — fetching from AWS Secrets Manager:**
```js
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const sm = new SecretsManagerClient({});
const res = await sm.send(new GetSecretValueCommand({ SecretId: 'prod/db/password' }));
const secret = res.SecretString;
```

**For Kubernetes:** External Secrets Operator syncs from a manager into K8s Secret objects, which are mounted as env vars or files. Combined with role-based access, no developer ever sees the actual secret.

**For CI/CD:** OIDC-based authentication (GitHub Actions → AWS / GCP) means CI gets short-lived credentials per run, no long-term keys stored in the repo.

**Follow-up 1: What's wrong with putting secrets in `.env` files?**
For local dev, fine. For production: (1) files often end up in git, in logs, in backups; (2) anyone with FS access can read them; (3) no rotation, no audit; (4) no fine-grained access. Secrets managers solve all of these.

**Follow-up 2: How do you handle "the secret leaked, what now?"**
Rotation. Revoke the leaked credential immediately (or as fast as possible). Issue a new one. Audit what the leaked credential could have accessed and check for misuse. If the secret was an API key, also rotate any downstream tokens it might have issued. Have a documented incident playbook; you don't want to be writing it during the breach.

---

### Q18. mTLS between services — how does it work and when do you need it?

**Answer:**
**mTLS (mutual TLS)** = both client and server present certificates to each other. Standard TLS authenticates only the server; mTLS authenticates both ends.

**Why use it:**
- **Network-level authentication** of every service-to-service call. Even if an attacker is inside your network, they can't call services without a valid cert.
- **Encryption** of internal traffic.
- **Identity binding** — each service has a unique cert; logs/audit can prove who called what.

**How:**
1. A private CA (Certificate Authority) issues certs to every service.
2. Each service is configured with its own cert+key and the CA's public cert.
3. On every call, both sides verify the other's cert is signed by the CA.

**At scale:**
- Service mesh (Istio, Linkerd) automates mTLS — sidecars handle certs, app code is unchanged.
- AWS App Mesh, Consul Connect: similar.
- Manual mTLS without a mesh is painful (cert rotation, distribution).

**SPIFFE/SPIRE** — open standard for service identity, often paired with mTLS.

**Code — Node.js mTLS server:**
```js
import https from 'https';
const server = https.createServer({
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),
  ca: fs.readFileSync('ca.crt'),
  requestCert: true,                  // require client cert
  rejectUnauthorized: true,           // reject if cert isn't valid
}, (req, res) => {
  const cert = req.socket.getPeerCertificate();
  console.log('called by', cert.subject.CN);
  res.end('ok');
});
```

**Follow-up 1: When is mTLS overkill?**
For a small monolith or simple architecture where network segmentation (VPC, security groups) already provides isolation. Adding mTLS is operational overhead — cert lifecycle, debugging, performance hit. Worth it when you have many services and need defense-in-depth, especially in regulated industries.

**Follow-up 2: How do certificates get rotated?**
With a mesh, sidecars rotate automatically (short-lived certs, e.g., 24h). Without a mesh, you script certbot or similar; each service reloads on cert change. Cert expiration causing outages is a famous failure mode — automate rotation and monitor cert expiration.

---

### Q19. What's the OWASP Top 10 — the high points?

**Answer:**
The OWASP Top 10 is a list of the most critical web security risks, updated periodically. As of 2021 (current at time of writing):

1. **Broken Access Control** — users access resources they shouldn't. Mitigations: centralized authorization, deny by default, log access decisions.

2. **Cryptographic Failures** — weak/missing encryption of sensitive data. Mitigations: TLS 1.2+, strong hashing for passwords, encrypt PII at rest, key management.

3. **Injection** — SQL, NoSQL, OS command, LDAP. Mitigations: parameterized queries, ORM, input validation.

4. **Insecure Design** — flaws in the architecture itself (e.g., no rate limiting on a password reset). Mitigations: threat modeling, secure defaults, security review during design.

5. **Security Misconfiguration** — default creds, verbose error messages, unnecessary features enabled. Mitigations: hardening, automated security scans, minimal images.

6. **Vulnerable & Outdated Components** — old dependencies with known CVEs. Mitigations: SBOM, automated dependency updates (Dependabot/Renovate), snyk scans.

7. **Identification & Authentication Failures** — broken sessions, weak passwords, no MFA. Mitigations: short-lived tokens, refresh-token rotation, MFA, secure session management.

8. **Software & Data Integrity Failures** — code/data tampering. Mitigations: signed packages, SLSA framework, code review.

9. **Security Logging & Monitoring Failures** — no detection of breaches in progress. Mitigations: centralized logs, alerts on anomalies, SIEM.

10. **Server-Side Request Forgery (SSRF)** — server fetches URLs from user input, attacker tricks it into hitting internal services. Mitigations: allowlist destinations, block internal IPs (169.254.169.254 is the AWS metadata service — popular target).

**Follow-up 1: What's the single most impactful thing a team can do for security?**
Centralized, well-tested authorization. Most breaches come from "X user accessed Y resource they shouldn't have." If every access decision goes through one well-tested module, with comprehensive tests, your blast radius is limited. Compare to scattered "if user.role === 'admin'" checks all over the codebase.

**Follow-up 2: How do you stay on top of new vulnerabilities?**
Automated scanning: GitHub Advanced Security, Snyk, Dependabot, Trivy for containers. Plus subscribe to relevant security mailing lists (Node Security WG, OWASP newsletter). The goal is "patch before exploit" — short loop between CVE disclosure and your deploy.

---

### Q20. What are the most common production security gotchas?

**Answer:**

1. **Authorization checks missed on a new endpoint.** Most common breach vector. Fix: deny-by-default, centralized authorization, mandatory tests.

2. **JWT validation missing one of: signature, exp, iss, aud, algorithm pinning.** Each is a separate footgun. Use a hardened library and test with bad tokens.

3. **`Allow all` CORS.** `origin: '*'` with credentials. Limits attack surface only to your domain.

4. **Sensitive data in URL.** Tokens, IDs, PII in query strings → leaks to logs, referers, browser history. Use headers / POST bodies.

5. **Open S3 bucket.** "It's just for static assets." Bots find it, dump contents, ransom or publish. Block Public Access, use CloudFront + signed URLs.

6. **Verbose error messages in production.** Stack traces leak code structure; SQL errors leak schema. Generic errors to clients; detailed logs internally.

7. **No rate limiting on auth endpoints.** Brute-force password guessing. Rate limit per IP and per username.

8. **Logging passwords or tokens.** Even accidentally — request body logger captures it. Use a redaction-aware logger.

9. **Outdated dependencies with known CVEs.** Update aggressively; Dependabot/Renovate auto-PRs.

10. **Secrets in container images.** Image gets pushed; everyone with pull access has the secret. Use runtime injection.

11. **TLS using weak ciphers / old versions.** Disable TLS 1.0, 1.1; use 1.2+. Configure cipher suites.

12. **No anti-CSRF on state-changing requests** for cookie-authenticated apps.

13. **Insufficient logging of security events.** When breach happens, no forensics. Log auth events, role changes, admin actions, failed access.

14. **Direct object references (IDOR).** `GET /api/orders/123` works for any user. Always check ownership / authorization on each resource.

15. **Trusting client-supplied headers like `X-User-Id` from external traffic.** Only trust them inside your network from the gateway.

**Follow-up 1: How do you test for IDOR (Insecure Direct Object Reference)?**
For every endpoint that accesses a resource by ID, test: user A's request for user B's resource should be 403 or 404. Automate this in your test suite: a "cross-tenant authorization" test set that runs on every PR.

**Follow-up 2: What's "secure by default" mean in practice?**
Defaults should be the safe choice. Examples: HTTPS only (no plain HTTP fallback), strict CORS (whitelist origins explicitly), authorization required (deny by default; opt-in to public), tokens expiring quickly (short access tokens), errors generic (detailed in logs, not responses). The configuration cost is on the developer making something less secure, not more secure.

---

*End of section 10. Next: REST API Design (10 questions).*
