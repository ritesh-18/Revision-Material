# 09 — AWS & DevOps (20 Questions)

You deployed on AWS (EC2 + S3 + IAM) at Tibil and the IoT project, automated via GitHub Actions with multi-stage Docker builds (reduced deploy time 30 → 15 min), and used Nginx as the reverse proxy. These cover compute, storage, IAM, Docker, Nginx, CI/CD, networking, and production gotchas.

---

### Q1. Talk through EC2 — instance types, AMIs, security groups.

**Answer:**
**EC2 (Elastic Compute Cloud)** = virtual machines on AWS.

**Instance types** — families optimized for different workloads:
- **General purpose (t, m)** — balanced CPU/RAM. T-series is "burstable" (cheap, accumulates CPU credits, throttles when exhausted). M-series is sustained.
- **Compute-optimized (c)** — high CPU/RAM ratio. For CPU-bound work.
- **Memory-optimized (r, x)** — large RAM. For databases, caches, in-memory analytics.
- **Storage-optimized (i, d)** — large local NVMe. For high IOPS, search indexes.
- **Accelerated (g, p)** — GPUs for ML/graphics.

Choose based on bottleneck. Most web/API workloads start on `t3.medium` or `m5.large` and scale based on observed CPU/memory.

**AMI (Amazon Machine Image)** = pre-built OS + packages. Two patterns:
- **Generic AMI** (Amazon Linux, Ubuntu) + bootstrap via user-data or config management. Flexible but slow boot.
- **Baked AMI** with your app pre-installed (built via Packer). Fast boot; immutable infra.

**Security groups** = stateful firewall rules per instance. Inbound/outbound rules by protocol+port+source. Default-deny inbound; default-allow outbound.
- Stateful — return traffic for an allowed connection passes automatically.
- Source can be CIDR, another security group, or prefix list.
- Modern practice: minimal exposure (SSH only from bastion's SG; app port only from load balancer's SG).

**Code — a typical web app SG setup:**
```
LB SG:
  inbound 443 from 0.0.0.0/0
  outbound 3000 to App SG

App SG:
  inbound 3000 from LB SG       (only LB can reach app)
  inbound 22 from Bastion SG
  outbound any
```

**Follow-up 1: What's the difference between security groups and NACLs?**
Security groups are stateful, attached to ENIs (instance level). NACLs are stateless, attached to subnets — must allow inbound and outbound separately. NACLs are coarser and rarely needed; SGs handle 95% of cases. Reach for NACLs for subnet-wide deny rules or compliance requirements.

**Follow-up 2: Spot vs On-Demand vs Reserved — when each?**
On-Demand: pay-as-you-go, no commitment, ~baseline price. Spot: ~70-90% off, but instance can be reclaimed with 2-minute notice — for fault-tolerant batch work. Reserved (or Savings Plans now): commit to 1 or 3 years for 30-70% off — for predictable baseline capacity. Mix: reserved for baseline, on-demand for surges, spot for batch.

---

### Q2. Walk me through S3 — storage classes, signed URLs, multipart upload.

**Answer:**
**S3** = object storage. Buckets contain objects; each object has a key (string path), value (bytes), metadata.

**Storage classes:**
- **Standard** — default. High availability, low latency. ~$0.023/GB/month.
- **Standard-IA (infrequent access)** — cheaper storage, charges for retrieval. ~half the price. For data accessed monthly-ish.
- **One Zone-IA** — IA in only one AZ. Cheaper still; risk if the AZ fails.
- **Glacier Instant Retrieval** — milliseconds retrieval, very cheap. For archives accessed quarterly.
- **Glacier Flexible Retrieval / Deep Archive** — minutes to hours retrieval. Cheapest. For compliance / disaster recovery.
- **Intelligent Tiering** — S3 moves objects between tiers based on access patterns; you pay a small monitoring fee.

**Lifecycle rules** automate tier transitions: "objects older than 30 days → Standard-IA; older than 90 → Glacier."

**Pre-signed URLs:**
- Generate a temporary URL with embedded auth (signed by your AWS credentials).
- Used for direct browser uploads/downloads without proxying through your server.
- Time-limited (e.g., 15 min); permission-scoped.

```js
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const url = await getSignedUrl(
  new S3Client({}),
  new PutObjectCommand({ Bucket: 'uploads', Key: 'user-42/avatar.jpg', ContentType: 'image/jpeg' }),
  { expiresIn: 900 }
);
// Browser uploads directly to this URL with PUT
```

**Multipart upload:**
- Required for objects > 5 GB; recommended for any > 100 MB.
- Split file into parts, upload in parallel, S3 reassembles.
- Each part can be retried independently.

```js
import { Upload } from '@aws-sdk/lib-storage';

await new Upload({
  client: s3,
  params: { Bucket: 'logs', Key: 'huge.log', Body: fs.createReadStream('huge.log') },
  partSize: 1024 * 1024 * 10,
  leavePartsOnError: false,
}).done();
```

**Follow-up 1: How do you avoid the "publicly accessible bucket" disaster?**
By default, S3 buckets are private. Don't change that. For public assets, use CloudFront with an Origin Access Control (OAC) — the bucket stays private, CloudFront has signed access. For uploaded user files, use pre-signed URLs for both upload and download. Bucket policies should never include `Principal: "*"` for object access.

**Follow-up 2: Versioning, encryption, and replication — what's the production checklist?**
- **Versioning ON** for any bucket with mutable objects. Lets you recover from accidental delete or overwrite.
- **Default encryption** with SSE-S3 (free) or SSE-KMS (audit-grade). Object-level encryption is a separate flag and matters for compliance.
- **Block Public Access** at the account level — overrides individual bucket settings; safety net.
- **Cross-region replication** for critical buckets — replicates writes to a different region.
- **Lifecycle rules** to expire intermediate files and tier cold data.

---

### Q3. IAM principles — users, roles, policies, least privilege.

**Answer:**
**IAM (Identity and Access Management)** controls who can do what in AWS.

**Principals:**
- **User** — a long-lived identity, usually a human, with credentials.
- **Role** — a set of permissions that can be **assumed** by users, services, or other AWS accounts. The right way for code/services.
- **Group** — collection of users (for organizing permissions).

**Policies** — JSON documents listing allowed/denied actions on resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/uploads/*",
    "Condition": { "StringEquals": { "aws:RequestedRegion": "us-east-1" } }
  }]
}
```

**Policy types:**
- **Identity-based** — attached to a user/role/group; says what *that identity* can do.
- **Resource-based** — attached to a resource (S3 bucket policy, KMS key policy); says who can access *that resource*.
- **Permissions boundaries** — max permissions a role can ever have, regardless of policies attached.

**Least privilege:**
- Grant only the actions/resources needed for the role's job.
- Avoid `*` wildcards in `Action` or `Resource`.
- Use Conditions to constrain further (IP range, time window, MFA-required).

**Roles for services:**
- EC2 instances assume an "instance profile" — no static credentials needed.
- Lambda functions have an execution role.
- Cross-account access via `sts:AssumeRole`.

**Code — checking a role in Node:**
```js
import { STSClient, GetCallerIdentityCommand } from '@aws-sdk/client-sts';

const sts = new STSClient({});
const caller = await sts.send(new GetCallerIdentityCommand({}));
// caller.Arn tells you which role/user you're currently running as
```

**Follow-up 1: What's the difference between assuming a role and being a user?**
A user has long-lived credentials (access key + secret). A role has temporary credentials minted on assume; expire after a configurable duration. Roles are vastly safer — no static keys to leak. AWS best practice: users only for humans (and even then, prefer SSO with role assumption); services always use roles.

**Follow-up 2: How do you handle "this lambda needs S3 read for one bucket"?**
Create a role for the lambda. Attach a policy: `Allow s3:GetObject on arn:aws:s3:::specific-bucket/*`. No wildcards in resource. Lambda's execution role is the principal; the policy is identity-based. Test with `iam policy simulator` before deploying.

---

### Q4. Talk me through your IoT pipeline architecture on AWS.

**Answer:**
The pipeline: devices → MQTT broker → backend service → S3 archival. Here's the AWS-flavored breakdown:

**Devices (edge):**
- IoT sensors publish MQTT messages with telemetry (CPU, memory, temperature).
- TLS encrypted; client certificates for authentication.
- Could use **AWS IoT Core** (managed MQTT broker with device shadow, rules engine) or self-hosted Mosquitto/EMQX on EC2.

**Broker:**
- AWS IoT Core if you want managed; otherwise EMQX on EC2 behind an NLB.
- QoS 1 with persistence so messages aren't lost if downstream is briefly unavailable.

**Ingestion service:**
- Node.js (Express) on EC2 (or ECS/Fargate for elasticity) subscribed to the broker.
- Validates messages; routes to short-term DB (Postgres or DynamoDB) for live queries + Kinesis/Kafka for downstream processing.

**Archival:**
- Periodic writer streams batched messages to S3 as parquet/JSON files, partitioned by date/device.
- Lifecycle rule moves old partitions to Glacier after 90 days.

**ML / analytics:**
- Python service reads S3 (or Athena queries S3 directly) for predictive analytics.

**Wired up:**
```
   Devices ──MQTT/TLS──▶  IoT Core / EMQX
                              │
                              ▼
                       Ingestion (EC2/ECS)
                       ┌──────┴───────┐
                       ▼              ▼
                 Postgres/RDS      Kinesis Stream
                                       │
                                       ▼
                                  Firehose
                                       │
                                       ▼
                                      S3
                                       │
                                       ▼
                            Athena / Glue / ML
```

**Code — ingestion subscriber sketch:**
```js
const client = mqtt.connect('mqtts://broker:8883', { /* TLS opts */ });
client.subscribe('telemetry/+/+', { qos: 1 });

client.on('message', async (topic, payload) => {
  const [_, deviceId, metric] = topic.split('/');
  const data = JSON.parse(payload.toString());
  await pg.query('INSERT INTO telemetry (...) VALUES (...)', [...]);
  await kinesis.putRecord({ /* archival */ });
});
```

**Follow-up 1: Why archive to S3 if you're already in a DB?**
Cost. Postgres storage for "last 90 days of telemetry at full fidelity" is expensive at scale. S3 is 10x cheaper, Glacier is 100x. Keep hot data in DB for queryability; archive cold data to S3 for compliance / ML training / occasional analytics via Athena.

**Follow-up 2: How would AWS IoT Core compare to self-hosted EMQX?**
IoT Core: managed (you don't run brokers), built-in device shadow, rules engine routes messages to AWS services directly (Lambda, DDB, Kinesis), per-message pricing. EMQX: more control, predictable cost at high volume, but you run/scale brokers. For < 1M messages/month, IoT Core wins on ops cost. Above that, self-hosted EMQX is cheaper.

---

### Q5. Docker fundamentals — layers, images, containers.

**Answer:**

**Image** = the immutable template. A stack of read-only layers built from a Dockerfile.

**Container** = a running instance of an image. Has a writable layer on top of the image's read-only layers.

**Layers** — every Dockerfile instruction creates a layer. Layers are cached: rebuilding only re-runs instructions whose inputs changed.

```
   Dockerfile               Layers
   ──────────              ─────────
   FROM node:20           Layer 1: node:20 base
   COPY package.json .    Layer 2: package.json
   RUN npm install        Layer 3: node_modules
   COPY . .               Layer 4: source code
   CMD ["node", "."]      (metadata; no FS change)
```

If `package.json` doesn't change, Docker reuses layer 2 *and* the cached layer 3 — no `npm install` re-run. Crucial for fast CI/CD.

**Key Dockerfile commands:**
- `FROM` — base image.
- `RUN` — execute at build time (compile, install).
- `COPY` / `ADD` — copy files into image.
- `ENV` — environment variable.
- `EXPOSE` — document which ports the container uses (advisory).
- `CMD` / `ENTRYPOINT` — what to run when container starts.
- `USER` — switch user (don't run as root in production).
- `WORKDIR` — change working directory.

**Image best practices:**
- Use a specific tag: `node:20.10-alpine`, never `latest`.
- Pin dependencies (`package-lock.json` committed).
- Smallest viable base — Alpine is tiny, `distroless` smaller still.
- Order Dockerfile instructions from least to most frequently changed (max cache hits).
- One process per container.
- Run as non-root user.
- `.dockerignore` to exclude node_modules, .git, secrets.

**Code — basic Dockerfile:**
```dockerfile
FROM node:20.10-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

**Follow-up 1: Why is layer order important?**
For build cache. If you `COPY . .` before `RUN npm install`, any change in any source file invalidates the cache for `npm install` — slow. By copying only `package*.json` first and installing deps, then copying source, you maximize cache hits on the expensive step.

**Follow-up 2: What's a "container" actually, under the hood?**
A process (or group) running in isolated Linux namespaces (PID, network, mount, user, IPC, UTS, cgroup), with resource limits via cgroups, and a unionfs-based filesystem layered on top of an image. No hypervisor — same kernel as the host. Lighter than a VM, less isolated.

---

### Q6. Explain multi-stage Docker builds with a real example.

**Answer:**
**Multi-stage** = use multiple `FROM` directives in one Dockerfile to separate build environment from runtime environment. The final image only contains runtime artifacts; build tools, source code, and test files are discarded.

**Why:**
- Smaller final images (no `gcc`, no dev dependencies, no source maps if not needed).
- Better security (no compilers or debug tools in production).
- Same Dockerfile builds with one command — no separate build scripts.

**Code — Node TypeScript with multi-stage:**
```dockerfile
# Stage 1: build
FROM node:20.10-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                            # includes dev deps for build
COPY tsconfig.json ./
COPY src ./src
RUN npm run build                     # tsc → dist/

# Stage 2: runtime
FROM node:20.10-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev                 # only production deps
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

The final image:
- Doesn't have TypeScript, ts-node, dev dependencies, source code.
- Goes from ~500 MB (full builder) to ~150 MB.

**Code — even smaller with `distroless`:**
```dockerfile
FROM node:20.10-alpine AS builder
# ... build steps ...

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["dist/main.js"]
```

Distroless has no shell, no package manager, no curl. Tiny attack surface; ~70 MB.

**For your "50% deploy time reduction"** — moving to multi-stage probably contributed: smaller images push faster to ECR, pull faster on EC2, start faster.

**Follow-up 1: What if I need to debug a distroless container?**
You can't `docker exec -it bash` — no shell. Solutions: (1) build a "debug" variant with a shell, push it on demand; (2) use ephemeral debug containers in Kubernetes (`kubectl debug`); (3) ship a sidecar with diagnostic tools for non-prod environments only.

**Follow-up 2: How do you handle native dependencies (sharp, bcrypt, etc.) in multi-stage?**
They compile against the OS in the build stage. If the runtime stage uses a different OS (build on Debian, run on Alpine = different glibc/musl), the native modules break. Either: (a) same OS for both stages, (b) rebuild native modules in the runtime stage, or (c) use prebuilt binaries (`npm install --omit=dev --prefer-offline` after copying full node_modules).

---

### Q7. Docker networking and volumes — what do you need to know?

**Answer:**

**Networking:**
Docker creates virtual networks. Containers on the same network can reach each other by name.

- **`bridge`** (default) — each container gets an IP on a shared bridge. Inter-container traffic via container names if you create a user-defined bridge.
- **`host`** — container shares host's network stack. Faster (no NAT) but no port isolation.
- **`none`** — no network. For batch jobs.
- **`overlay`** — multi-host networking for Swarm/Kubernetes.

**Port publishing:**
- `-p 8080:3000` — host port 8080 → container port 3000.
- Without publish, the port is reachable only from other containers on the same network.

**Docker Compose example:**
```yaml
services:
  app:
    build: .
    ports: ['3000:3000']
    depends_on: [postgres]
    environment:
      DATABASE_URL: postgres://app:pass@postgres:5432/app
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: pass
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

`app` reaches `postgres` by name. Both on the default Compose-created network.

**Volumes:**
- **Named volume** — managed by Docker, persists across container restarts. Use for DB data, caches.
- **Bind mount** — mounts a host directory into the container. Use for dev (live code reload).
- **tmpfs** — in-memory only. Use for secrets you don't want on disk.

**Code — using volumes correctly:**
```yaml
services:
  app:
    image: myapp
    volumes:
      - ./src:/app/src                # bind mount for development
      - app-logs:/var/log/app         # named volume for persistence
volumes:
  app-logs:
```

**Follow-up 1: What happens when a container restarts? Is data lost?**
The container's writable layer is lost when removed (not just restarted). Volumes survive container restarts AND removal — that's their purpose. Always store databases, uploads, logs on volumes; never inside the container's filesystem.

**Follow-up 2: How do you handle secrets — env vars or files?**
Avoid env vars for production secrets — they leak into logs, error reports, and `docker inspect`. Use Docker Secrets (Swarm), Kubernetes Secrets, or mount a file via volumes/tmpfs. For local dev, env vars are fine. Always exclude secret files from images (`.dockerignore`).

---

### Q8. Nginx as a reverse proxy — show me the basics.

**Answer:**
**Nginx** sits in front of your application(s), accepts HTTPS, routes to backend(s) over HTTP, and handles cross-cutting concerns: TLS termination, caching, compression, rate limiting, static file serving.

**Basic reverse proxy config:**
```nginx
server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    location / {
        proxy_pass http://app:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$host$request_uri;       # force HTTPS
}
```

**Key directives:**
- `proxy_pass` — where to send the request.
- `proxy_set_header` — preserve client info for the backend.
- `proxy_http_version 1.1` — enable keepalive to backend (default is 1.0 = new TCP per request).
- `proxy_buffering` — Nginx buffers backend response by default; turn off for streaming responses.

**Why use Nginx instead of going direct to Node?**
- TLS handling — Node doesn't need TLS at all if Nginx terminates.
- Connection reuse — Nginx keeps a small pool to Node; absorbs huge external concurrency.
- Static file serving — much faster than Node serving them.
- Caching, compression — Nginx has decades of optimization.
- Failover — easy to route around an unhealthy backend.

**Follow-up 1: What does `proxy_buffering` actually do?**
Nginx reads the entire backend response into a buffer before sending to the client. Good for fast backends + slow clients (frees the backend quickly). Bad for streaming responses (SSE, WebSocket-upgrade-ish patterns) — the buffer holds everything, defeating streaming. Set `proxy_buffering off` for streaming endpoints.

**Follow-up 2: When would I use Nginx vs an ALB?**
On a small VM (single EC2): Nginx is free and integrated. In AWS with multi-AZ: ALB handles TLS, health checks, scaling, integration with target groups. Many setups use both: ALB → EC2 → Nginx (for path routing, cache, compression). Don't put Nginx in front of an ALB; pointless.

---

### Q9. Nginx load balancing — strategies and health checks.

**Answer:**
Nginx can balance traffic across a pool of backends.

**Strategies:**
- **Round-robin** (default) — rotate through backends.
- **Least connections** — send to the backend with fewest active connections. Good for variable-duration requests.
- **IP hash** — same client IP always goes to the same backend. Useful for sticky sessions.
- **Weighted** — give some backends more traffic than others.

**Config:**
```nginx
upstream app_backend {
    least_conn;
    server app-1.internal:3000 weight=2;
    server app-2.internal:3000;
    server app-3.internal:3000 backup;        # only used if others are down
    keepalive 32;                              # persistent conn pool
}

server {
    location / {
        proxy_pass http://app_backend;
    }
}
```

**Health checks:**
- **Passive** (default in open source) — Nginx marks a backend as failed after `max_fails` failed requests in `fail_timeout` seconds, retries after `fail_timeout`.
- **Active** — only in Nginx Plus or OSS with `nginx_upstream_check_module`. Periodic synthetic requests.

Most teams achieve active health checks externally: a script that hits `/health` on each backend, removes failed ones via `nginx -s reload`.

**Failure handling:**
- `proxy_next_upstream` — which errors should cause Nginx to try the next backend (defaults: error, timeout).
- `proxy_connect_timeout` / `proxy_read_timeout` — when to give up.

**For your "1K req/sec horizontal scaling":**
A typical setup:
```
ALB (managed) ──▶  EC2 Nginx (3 instances)  ──▶  Node app pods
                                                 (auto-scaled)
```
ALB does L7 routing; Nginx adds path-based routing, compression, cache. Node pods scale horizontally.

**Follow-up 1: How do you do zero-downtime config reload?**
`nginx -s reload` (or `systemctl reload nginx`). It spawns new workers with the new config; old workers finish in-flight requests and exit. No dropped connections. The same technique handles re-adding backends to the upstream pool after a deploy.

**Follow-up 2: When does L7 (Nginx) vs L4 (TCP load balancer) matter?**
L7 understands HTTP — can route by path, header, method; can manipulate the request. L4 sees only TCP — faster, simpler. For HTTP services, L7 (Nginx, ALB). For pure TCP (Redis, Postgres, custom protocols), L4 (NLB).

---

### Q10. CI/CD pipeline design — what stages, what runs where?

**Answer:**
A complete CI/CD pipeline runs these stages, typically on every PR + main merge:

```
   Commit / PR
        │
        ▼
   ┌──────────────┐
   │   CI (PR)    │
   │              │
   │  - install   │
   │  - lint      │
   │  - typecheck │
   │  - unit test │
   │  - build     │
   │  - intgr tst │
   └──────┬───────┘
          │ merge to main
          ▼
   ┌──────────────┐
   │  Build & push│
   │  Docker image│
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │   Deploy     │
   │   (staging)  │
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │  E2E tests   │
   │  on staging  │
   └──────┬───────┘
          ▼
   ┌──────────────┐
   │  Deploy prod │
   │  (manual or  │
   │   automatic) │
   └──────────────┘
```

**Principles:**
- **Fast feedback first.** Lint and unit tests should fail in < 2 minutes. Long-running integration tests run after.
- **Same image promoted from dev → staging → prod.** Don't rebuild; just redeploy.
- **Immutable artifacts.** Tag image by commit SHA, not "latest."
- **Automatic deploy to staging, manual gate to prod** for most teams. Mature teams auto-deploy prod after staging passes.
- **Rollback is one click** — keep the previous image around, redeploy it.

**For your Tibil pipeline (30 → 15 min):**
The optimizations that compound:
- Docker layer caching (don't rebuild deps unless they changed).
- Multi-stage builds (smaller final image, faster push).
- Parallel test runs by file or module.
- Cache dependencies (npm cache restored from prior runs).
- Skip CI on draft PRs.

**Follow-up 1: When should you NOT deploy automatically?**
For database migrations that aren't backwards-compatible — require a manual approval. For changes to authentication / billing / compliance code — security review. For changes touching shared infrastructure — coordinate with affected teams. Otherwise, automate aggressively.

**Follow-up 2: What's the right way to handle DB migrations in CI/CD?**
Run migrations as a separate step before deploying new app code. Sequence:
1. Deploy new app code that's compatible with both old and new schema.
2. Run migration.
3. Backfill data if needed.
4. Deploy app code that uses the new schema only (next release).

This is the expand-contract pattern (Q19 of section 03). Risky shortcut: run migrations and deploy code in one step — fine for small additive changes, dangerous for anything else.

---

### Q11. GitHub Actions — workflows, jobs, secrets.

**Answer:**

**Workflow** = YAML file in `.github/workflows/`. Defines triggers and jobs.
**Trigger** — on push, pull_request, schedule, workflow_dispatch (manual).
**Job** — runs on a runner (Linux/macOS/Windows). Jobs run in parallel by default; declare `needs:` for dependencies.
**Step** — runs a command or an Action.

**Code — typical Node CI workflow:**
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env: { POSTGRES_PASSWORD: pass }
        ports: ['5432:5432']
        options: >-
          --health-cmd "pg_isready" --health-interval 10s
          --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20.10'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://postgres:pass@localhost:5432/postgres

  build-and-push:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write              # for OIDC to AWS
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gh-actions-deploy
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr
      - run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} ${{ steps.ecr.outputs.registry }}/myapp:${{ github.sha }}
          docker push ${{ steps.ecr.outputs.registry }}/myapp:${{ github.sha }}
```

**Secrets** — stored in repo/org settings. Available as `${{ secrets.NAME }}`. For AWS, use **OIDC** (no long-lived secrets) — GitHub mints a JWT, AWS trusts GH's OIDC provider, role assumed without storing keys.

**Caching** — `actions/cache` for dependencies. The `cache: 'npm'` shortcut in setup-node uses package-lock as cache key.

**Follow-up 1: How do you avoid running CI on every commit during a busy WIP session?**
Use `[skip ci]` in the commit message (GitHub Actions respects it on push triggers). Or push to a draft PR — most projects skip expensive jobs on draft state (check `github.event.pull_request.draft`).

**Follow-up 2: Self-hosted runners vs GitHub-hosted — when each?**
GitHub-hosted: free for public repos, decent for private (with limits). Easy. Self-hosted: when you need access to your private network, big resources (GPUs, lots of RAM), or want to control concurrency. Cost: you run the infrastructure. Most teams stay GitHub-hosted; reach for self-hosted only at scale.

---

### Q12. Blue-green vs canary vs rolling — pick the right deployment strategy.

**Answer:**

**Rolling deploy:**
- Replace instances/pods one at a time with the new version.
- Default in Kubernetes.
- Slow but safe; load balancer routes traffic to ready pods only.
- Brief period where both versions handle traffic.

**Blue-green:**
- Two parallel environments (blue = current, green = new).
- Deploy to green; test; flip the load balancer.
- Instant cutover; instant rollback (flip back).
- Cost: double the infrastructure during deploy.

**Canary:**
- Deploy new version to a small subset of traffic (1%, 10%, 50%, 100%).
- Monitor errors/latency at each step.
- Roll forward if healthy, rollback if not.
- Slowest, safest. Good for risky changes.

**Choosing:**
- Stateless services, low-risk changes → rolling.
- Need instant rollback or blue-green is required by compliance → blue-green.
- High-risk changes, large user base → canary.
- Database migrations → none of the above; use expand-contract.

**Code — Kubernetes rolling update spec:**
```yaml
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # at most 1 pod unavailable at a time
      maxSurge: 2            # can run up to 2 extra pods during update
```

**For your "30 min → 15 min deploy cycle"**: rolling deploys on K8s/EC2 with health-check-gated cutover. Combined with smaller images and parallel CI, that gets you in the 15-minute range.

**Follow-up 1: How do you do canary in plain AWS without K8s?**
Two target groups behind one ALB. Weighted routing — start at 95% to old, 5% to new. Adjust weights via API. Monitor metrics; complete cutover when comfortable.

**Follow-up 2: What's "feature flags" and how do they help deploys?**
Decouple "code deployed" from "feature on." Deploy code dark; flip a flag to enable for selected users (1%, employees, by region). If something's wrong, flip the flag off — no rollback needed. Tools: LaunchDarkly, Unleash, Flagsmith, or homegrown.

---

### Q13. How do you do rolling deploys in Kubernetes specifically?

**Answer:**
K8s Deployments handle rolling updates natively. Update the image tag in the Deployment spec; the controller creates a new ReplicaSet with new pods, gradually scales it up while scaling the old one down.

**The dance:**
1. Apply new Deployment manifest (new image SHA).
2. K8s creates new pods one at a time (within `maxSurge` budget).
3. New pod becomes Ready (readiness probe passes).
4. Old pod is sent SIGTERM (within `maxUnavailable` budget).
5. Service routes traffic only to Ready pods.
6. Repeat until all old replaced.

**Critical settings:**
- **Readiness probe must be accurate.** "Ready" = "I can handle traffic" — typically tests DB connection.
- **Graceful shutdown** — pod handles SIGTERM, flips readiness to false, drains in-flight requests, then exits within `terminationGracePeriodSeconds`.
- **`maxUnavailable: 0`** if you need 100% capacity throughout the deploy; trade-off is slower deploys.

**Code:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: myapp }
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: app
          image: myapp:abc123
          readinessProbe:
            httpGet: { path: /readyz, port: 3000 }
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet: { path: /livez, port: 3000 }
            periodSeconds: 10
            failureThreshold: 3
```

**Rollback:**
```bash
kubectl rollout undo deployment/myapp
```

`kubectl` keeps the previous ReplicaSet around (within revision history), enabling instant rollback.

**Follow-up 1: What goes wrong if readiness probe is too lenient?**
K8s sends traffic to pods that aren't actually ready (e.g., DB pool not initialized, JIT not warm). First requests time out or 500. Looks like flaky deploys. Make readiness probe genuinely reflect "ready to serve" — including any startup work.

**Follow-up 2: How do you handle long-running deploys for slow-starting apps?**
Use `startupProbe` separately from `livenessProbe`. Startup probe runs first; once it passes, liveness takes over. Allows liveness probe to be aggressive (catch real hangs) without prematurely killing slow-starting apps.

---

### Q14. Linux basics every backend engineer should know.

**Answer:**

**Filesystem hierarchy:**
- `/etc` — config. `/var` — variable data (logs, caches). `/usr` — software. `/home` — users. `/proc`, `/sys` — kernel interfaces.

**Permissions:**
- `rwx` for owner / group / other. `chmod 755 file` = rwxr-xr-x.
- `chown user:group file` — change owner.
- Setuid/setgid bits — execute as owner/group (rare, security-sensitive).

**Useful commands:**
```bash
# Find files
find /var/log -name "*.log" -mtime -1            # logs modified in last day
# Search inside files
grep -rn "ERROR" /var/log
# Count
wc -l file.log
# Sort and unique
sort access.log | uniq -c | sort -rn | head
# Disk usage
du -sh /var/lib/postgresql
df -h
# Process list
ps aux | grep node
# Resource usage in real time
top    htop
# Network
ss -tnlp                    # listening sockets
curl -v https://example.com
# Manage files
tar -czf archive.tar.gz dir/
rsync -az dir/ remote:dir/
```

**Process management:**
- `kill PID` (SIGTERM, graceful).
- `kill -9 PID` (SIGKILL, immediate).
- `nohup cmd &` — run in background, survives logout.
- `systemctl start/stop/restart/status service` — modern service control.
- `journalctl -u service -f` — follow service logs.

**Streams:**
- stdin (0), stdout (1), stderr (2).
- `cmd > out.log 2>&1` — redirect both to file.
- `cmd1 | cmd2` — pipe stdout of cmd1 to stdin of cmd2.

**Follow-up 1: How does Linux handle backpressure / OOM?**
The OOM killer chooses a process to kill when system memory is exhausted. It uses an "oom_score" based on memory usage and other factors. Critical services can be protected with `oom_score_adj`. In containers, cgroup memory limits trigger OOM within the container before the host runs out.

**Follow-up 2: What's the difference between SIGTERM and SIGKILL?**
SIGTERM (15) is a polite request: the process can clean up. SIGKILL (9) is unstoppable: kernel removes the process immediately, no cleanup. Always SIGTERM first; SIGKILL only if it didn't respond. K8s does this by default: SIGTERM, wait `terminationGracePeriodSeconds`, then SIGKILL.

---

### Q15. Linux processes, signals, and systemd basics.

**Answer:**

**Processes:**
- Every process has a PID, parent PID, user, working directory, environment, open FDs.
- `fork()` creates a child (copies parent). `exec()` replaces the process image with a new program.
- Zombie process: child finished but parent hasn't reaped it (called `wait()`). Stays in process table.
- Daemon process: detached from terminal, runs in background. Typically owned by init (PID 1).

**Signals** are messages sent to processes. Common:
- SIGHUP (1) — terminal closed. Often used to reload config.
- SIGINT (2) — Ctrl+C.
- SIGTERM (15) — polite stop.
- SIGKILL (9) — force kill, uncatchable.
- SIGCHLD — child process changed state.
- SIGPIPE — broken pipe (writing to closed socket).
- SIGUSR1, SIGUSR2 — user-defined. Apps can use for custom triggers (Node.js heap dump on SIGUSR2).

**systemd:**
- Init system (PID 1) on most modern Linux distros.
- Manages services via unit files in `/etc/systemd/system/`.

**Code — basic systemd service:**
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target postgresql.service

[Service]
Type=simple
User=app
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/node dist/main.js
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment="NODE_ENV=production"
EnvironmentFile=/etc/myapp.env

[Install]
WantedBy=multi-user.target
```

Operations:
```bash
sudo systemctl daemon-reload                # reload unit files
sudo systemctl enable --now myapp            # start at boot + now
sudo systemctl restart myapp
sudo systemctl status myapp
sudo journalctl -u myapp -f                  # live logs
```

**Follow-up 1: When would I run a service via systemd vs Docker?**
systemd for traditional Linux deployments (no containers), or for managing the Docker daemon itself. Docker for portability, environment isolation, and consistent dev/prod. Most modern stacks use containers; systemd manages the orchestrator (Kubernetes nodes, Docker daemon, monitoring agents).

**Follow-up 2: What if your Node process needs to handle SIGTERM gracefully?**
```js
const server = app.listen(3000);

process.on('SIGTERM', async () => {
  console.log('SIGTERM received, draining...');
  server.close(() => console.log('HTTP server closed'));
  await db.close();
  await redis.quit();
  process.exit(0);
});
```

systemd or K8s sends SIGTERM, your handler stops accepting new traffic, drains in-flight, exits cleanly. Without this handler, requests die mid-flight on restart.

---

### Q16. SSH, key management, and bastion patterns.

**Answer:**

**SSH** = encrypted remote shell. Identity via public/private key pairs.

**Generating and using keys:**
```bash
# Generate ed25519 (modern, fast)
ssh-keygen -t ed25519 -C "me@example.com"

# Public key goes to server's ~/.ssh/authorized_keys
ssh-copy-id user@server

# Use a specific key
ssh -i ~/.ssh/prod_key user@server
```

**Config file (`~/.ssh/config`):**
```
Host bastion
    HostName bastion.example.com
    User opsadmin
    IdentityFile ~/.ssh/bastion_key

Host app-*
    User app
    ProxyJump bastion
    IdentityFile ~/.ssh/app_key
```

Now `ssh app-1` connects via bastion automatically.

**Bastion (jump host) pattern:**
- One small server is the only one with SSH exposed to the internet.
- Internal servers only accept SSH from the bastion's security group.
- Reduces attack surface; centralizes audit.
- ProxyJump (`-J`) tunnels through the bastion without storing private keys on it.

**Best practices:**
- Disable password auth; key-only.
- Disable root login.
- Use `ed25519` keys (smaller, faster, secure).
- Rotate keys periodically.
- For AWS: use SSM Session Manager — no SSH at all, IAM-authenticated, audit-logged.

**Code — AWS SSM Session Manager (preferred over SSH):**
```bash
aws ssm start-session --target i-0abc123def456
# IAM controls who can connect. No SSH port needs to be open.
```

**Follow-up 1: How do you handle SSH access for a team?**
Several approaches: (1) shared SSH config in a private repo + automated key distribution; (2) certificate-based SSH (CA signs short-lived certs for each user); (3) bastion + LDAP/Okta integration; (4) skip SSH, use SSM Session Manager with IAM SSO. Last option is increasingly the cloud default.

**Follow-up 2: What's a "port forward" and when use it?**
`ssh -L 5432:rds:5432 bastion` — local port 5432 tunnels through SSH to the remote target. Useful for connecting your local psql to an RDS that's only accessible from inside the VPC. Use sparingly; prefer VPN or SSM for sustained access.

---

### Q17. AWS networking — VPC, subnets, security groups, NACLs.

**Answer:**

**VPC (Virtual Private Cloud)** = isolated virtual network in AWS. Defined by a CIDR block (e.g., `10.0.0.0/16` = 65k IPs).

**Subnets** = subdivisions of the VPC's CIDR, tied to one AZ.
- **Public subnet** — has a route to an Internet Gateway. Instances with public IPs can reach the internet directly.
- **Private subnet** — no route to IGW. Instances reach the internet through a NAT Gateway in a public subnet.

**Typical 3-AZ layout:**
```
VPC 10.0.0.0/16
  ├─ Public subnet AZ-a: 10.0.0.0/24   ── route to IGW
  ├─ Public subnet AZ-b: 10.0.1.0/24
  ├─ Public subnet AZ-c: 10.0.2.0/24
  ├─ Private subnet AZ-a: 10.0.10.0/24  ── route to NAT GW
  ├─ Private subnet AZ-b: 10.0.11.0/24
  └─ Private subnet AZ-c: 10.0.12.0/24
```

**Where things go:**
- ALB → public subnets (internet-facing).
- App servers → private subnets (behind ALB).
- DB → private subnets, accessible only from app SG.
- NAT Gateway → public subnet, used by private subnets to reach internet for outbound (e.g., calling external APIs).

**Security groups vs NACLs:**
- **SG** — stateful, per-ENI. Default deny inbound, allow outbound.
- **NACL** — stateless, per-subnet. Inbound and outbound rules must both allow. Use for subnet-wide deny lists or compliance.

**VPC peering / Transit Gateway / PrivateLink:**
- Peering — direct connection between two VPCs.
- Transit Gateway — hub-and-spoke for many VPCs.
- PrivateLink — expose a service in one VPC to another via interface endpoint (no full network connectivity).

**Follow-up 1: Why use private subnets at all?**
Defense in depth. An attacker who breaches a public-subnet server (the ALB) still can't reach the DB directly. Even if app code has a vulnerability, the blast radius is limited. Also, AWS billing — instances in private subnets without public IPs are slightly cheaper, and outbound NAT is cheaper per byte than per-instance internet.

**Follow-up 2: What's a VPC Endpoint and when use it?**
A way to reach AWS services (S3, DynamoDB, etc.) without traversing the internet. Saves money (no NAT data transfer) and improves security (traffic stays in AWS network). Two flavors: Gateway endpoints (free, S3 + DDB only) and Interface endpoints (paid, most services).

---

### Q18. RDS basics — what does AWS take care of, what's still on you?

**Answer:**
**RDS (Relational Database Service)** is managed Postgres/MySQL/MariaDB/Oracle/SQL Server.

**AWS handles:**
- OS and DB engine patching.
- Backups (automated, point-in-time recovery within retention).
- Multi-AZ failover (sync replica in a different AZ; promotion on primary failure).
- Read replicas (async, up to 5 per primary).
- Monitoring metrics.
- Snapshots (manual or automated).
- Storage scaling (auto-scaling up to a limit).

**You still handle:**
- Schema, indexes, query performance.
- Application-level connection pooling (RDS has its own connection limits).
- Configuration tuning (parameter groups for Postgres-specific settings).
- Cost optimization (right-sizing instance, choosing storage type).

**Instance types:**
- **db.t3 / db.t4g** — burstable, cheap, ok for dev.
- **db.m5 / db.m6i** — general purpose.
- **db.r5 / db.r6i / db.x2** — memory-optimized, for big workloads or in-memory caches.
- **Serverless v2** — auto-scaling Aurora.

**Aurora vs RDS Postgres:**
- Aurora = AWS's redesigned storage engine for Postgres/MySQL. 6-way replication, separate storage layer, faster failover, larger max size. More expensive per hour but often cheaper per workload.

**Code — connection from Node:**
```js
const pool = new Pool({
  host: 'mydb.abc.us-east-1.rds.amazonaws.com',
  port: 5432,
  user: 'app',
  password: process.env.DB_PASSWORD,
  database: 'app',
  ssl: { rejectUnauthorized: true, ca: rdsCaCert },
  max: 20,
  idleTimeoutMillis: 30_000,
});
```

**Follow-up 1: How do you scale reads with RDS?**
Read replicas — async copies of the primary. App routes write queries to primary, read queries to replicas. Mind the replication lag (usually < 1s but can spike). Don't read your own writes from a replica.

**Follow-up 2: How does Multi-AZ failover actually work?**
RDS maintains a synchronous standby in a different AZ. On primary failure, DNS flip routes traffic to standby (now primary). Takes 30s–2 min, during which writes fail. Your app needs reconnection logic. Use Multi-AZ for production; the cost (~2x) is worth the availability.

---

### Q19. Observability on AWS — CloudWatch, X-Ray, log groups.

**Answer:**

**CloudWatch Metrics:**
- Time-series metrics from AWS services (EC2 CPU, RDS connections, ALB requests, Lambda errors).
- Custom metrics from your app: `cloudwatch.putMetricData(...)`.
- Alarms trigger on thresholds (e.g., CPU > 80% for 5 min).
- Dashboards.

**CloudWatch Logs:**
- Log groups → log streams → log events.
- App writes via the AWS SDK or CloudWatch agent.
- Containers: log driver `awslogs` ships stdout/stderr directly.
- **Insights** for ad-hoc queries with a SQL-like syntax.

**X-Ray:**
- AWS's distributed tracing.
- Auto-instrumentation for AWS SDK calls; manual for your own spans.
- Service map shows dependencies and latencies.
- Largely overlaps with OpenTelemetry; X-Ray is AWS-native, OTel is open-standard.

**EventBridge:**
- Event bus for AWS service events and custom events.
- Use for cross-service notifications, scheduled tasks.

**Code — sending custom CloudWatch metrics:**
```js
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const cw = new CloudWatchClient({});
await cw.send(new PutMetricDataCommand({
  Namespace: 'MyApp',
  MetricData: [{
    MetricName: 'OrdersProcessed',
    Value: 1,
    Unit: 'Count',
    Dimensions: [{ Name: 'TenantTier', Value: 'pro' }],
  }],
}));
```

**For full observability**, most teams combine: CloudWatch (infra metrics, log archive), OpenTelemetry → managed Prometheus / Grafana / Datadog (app metrics and traces), Sentry (error tracking). The AWS-native option (CloudWatch + X-Ray) works but is typically less ergonomic than dedicated tools.

**Follow-up 1: How do you avoid CloudWatch bill surprises?**
Logs are the biggest cost driver. Practices: (1) set retention policies (default is "never expire" — expensive); (2) avoid logging huge payloads; (3) use sampling for high-volume logs; (4) export old logs to S3 with lifecycle to Glacier; (5) metrics with high cardinality cost more — keep label sets small.

**Follow-up 2: How does CloudWatch Logs Insights compare to ELK / Loki?**
Insights is convenient (no infra), works on existing CloudWatch logs, has decent query syntax. Limits: query latency, cost at scale, less rich than Kibana. For low-volume / simple needs, fine; for ad-hoc deep investigation, move logs to a dedicated log search tool.

---

### Q20. What are the most common AWS / DevOps production gotchas?

**Answer:**

1. **Secrets in env vars committed to git.** Use AWS Secrets Manager / SSM Parameter Store + IAM. Scan repo with gitleaks.

2. **S3 buckets accidentally public.** Block Public Access at the account level; use CloudFront + OAC for public assets.

3. **Static credentials in CI.** Use OIDC for GitHub Actions → AWS; never store long-lived access keys.

4. **No backup / restore drill.** RDS backs up automatically, but have you ever restored? Periodic restore tests.

5. **Single-AZ deployments.** When AZ fails, service dies. Always Multi-AZ for production.

6. **Forgetting NAT Gateway costs.** $0.045/hour + $0.045/GB processed. Heavy outbound traffic via NAT gets expensive. Use VPC Endpoints for S3/DDB; cache external API calls.

7. **Default security groups left wide open.** "Default allow all" SG attached to instances. Audit and tighten.

8. **RDS connection limit hit.** Default `max_connections` on RDS is per instance class, often low. Use connection pooling (PgBouncer) and right-size the instance.

9. **No health checks → bad pods serving traffic.** Liveness/readiness probes properly set; ALB health checks aligned with reality.

10. **Auto-scaling not tuned.** Either never scales (alarm thresholds wrong) or scales too aggressively (thrashing). Validate by load testing.

11. **Log explosion.** Verbose logging in production = $$$. Use structured logs with levels; sample DEBUG.

12. **Forgetting to set `terminationGracePeriodSeconds` long enough.** Pods killed mid-request during deploys.

13. **No "deployable from local"** — only CI/CD can deploy, but CI/CD is the only thing tested. When CI/CD breaks, no one can ship. Have a tested manual fallback.

14. **Missing CloudTrail.** Audit log of every API call in AWS. Enable on day one; investigate any anomaly.

15. **No alerting on cost spikes.** A bug that pumps 10M messages to SQS goes unnoticed until the bill arrives. Budget alarms at 50%, 80%, 100% of expected.

**Follow-up 1: What's the cheapest, highest-leverage thing to do on day one of a new AWS account?**
Enable AWS Organizations / SSO + CloudTrail + Cost Explorer + budgets. Set Block Public Access on S3 at account level. Use SSO instead of root account. These cost nothing and save you from the most common disasters.

**Follow-up 2: What does "infrastructure as code" mean in practice?**
Define your infrastructure (VPCs, RDS, ECS, IAM roles) in code (Terraform, CloudFormation, CDK, Pulumi), version it, review it like application code. Benefits: reproducible, reviewable, undoable. Drawbacks: a learning curve. Most teams should adopt it from day one — even simple Terraform beats clicking around the console.

---

*End of section 09. Next: Auth & Security (20 questions).*
