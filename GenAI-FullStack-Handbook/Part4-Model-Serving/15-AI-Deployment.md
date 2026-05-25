# Chapter 15 — AI Deployment

## 15.1 Concept Explanation

Deployment is where AI systems meet reality. Models that work in notebooks fail in production for reasons that have nothing to do with the model: container size, networking, autoscaling, secrets, ingress, observability. This chapter is the deepest in the book because deployment is where the most engineering hours and the most production failures live.

We will move from the bottom up: containers, orchestration, GPU specialization, scaling, traffic management, networking, deployment strategies, multi-cloud, CI/CD, and observability hooks. By the end, you should be able to design a production-grade AI deployment from scratch and explain every layer.

## 15.2 Docker

Docker packages an application and its dependencies into a portable container image. For AI, this is essential: model weights, Python versions, CUDA libraries, system libraries, and inference servers all need to be consistent across environments.

Key practices:
- **Multi-stage builds.** Separate build environment (with compilers) from runtime environment (lean image).
- **CUDA base images.** Start from NVIDIA's CUDA images that include the right toolkit versions.
- **Image size discipline.** A 20 GB image takes minutes to pull and inflates registry costs. Strip caches, use slim bases, avoid bundling model weights in the image (mount them at runtime).
- **Pinned versions.** Pin every dependency; "latest" is how stable deployments break.
- **Health checks.** Both readiness (ready to serve traffic) and liveness (still alive).

For inference servers, vendor-published images (vLLM, TGI, Triton) are usually the right starting point — they bake in CUDA, the inference engine, and reasonable defaults.

## 15.3 Docker Compose

Docker Compose orchestrates multiple containers on a single host. For AI, it is useful in development (model + vector DB + redis on a laptop) and in small single-node deployments. For production multi-node systems, you want Kubernetes.

## 15.4 Kubernetes — The Orchestration Layer

Kubernetes is the de facto orchestration system for containerized workloads. It manages container lifecycle, scaling, networking, secrets, storage, and health across many nodes.

Key concepts for AI deployments:

- **Pod** — the smallest deployable unit, usually one or more containers running together.
- **Deployment** — declarative spec for stateless workloads, manages replica count and rolling updates.
- **StatefulSet** — for stateful workloads (databases, some vector DBs).
- **Service** — stable network endpoint for a set of pods.
- **Ingress** — HTTP routing into the cluster.
- **ConfigMap / Secret** — non-sensitive and sensitive configuration.
- **PersistentVolume / PersistentVolumeClaim** — storage abstractions.
- **HorizontalPodAutoscaler (HPA)** — scales replicas based on metrics.
- **Node** — a machine in the cluster.
- **Namespace** — logical isolation within a cluster.

Kubernetes is complex. The reward for learning it is operational uniformity across cloud providers and on-premises.

## 15.5 Helm and Kustomize

Raw manifest files become unwieldy fast. Two tools dominate:

**Helm** is a package manager. Helm charts are reusable templates with values you override per environment. Most popular software (Postgres, Redis, Kafka, vLLM) ships official Helm charts. Strength: powerful templating, broad ecosystem. Weakness: templates can become hard to read.

**Kustomize** layers patches on a base manifest. Strength: simple, no templates. Weakness: less flexible for complex deployments.

Many teams use both: Helm for third-party software, Kustomize for in-house services.

## 15.6 GPU Operators and Device Plugins

Kubernetes does not know about GPUs by default. The NVIDIA GPU Operator (a collection of Kubernetes operators) installs drivers, the device plugin, DCGM exporter, MIG manager, and the container runtime configuration needed to schedule GPU workloads.

The GPU Operator handles the awkward parts: matching driver versions, exposing GPUs to pods, managing MIG partitions. On managed Kubernetes (EKS, GKE, AKS), the cloud provider often pre-installs much of this.

Workloads request GPUs in their pod spec; the scheduler places them on nodes with available GPUs.

## 15.7 Autoscaling

Three levels of autoscaling in Kubernetes:

**Horizontal Pod Autoscaler (HPA)** — scales pod replicas based on metrics (CPU, memory, custom). For AI, scale on GPU utilization, queue depth, or tokens-per-second, not raw CPU.

**Vertical Pod Autoscaler (VPA)** — adjusts pod resource requests. Less common for AI because GPU allocation is integer-sized.

**Cluster Autoscaler** — adds or removes nodes when pods cannot be scheduled or nodes are underutilized. Critical for GPU workloads because GPU nodes are expensive.

**Karpenter** (AWS) and similar provisioners offer more flexible node provisioning, often faster than the classic cluster autoscaler.

## 15.8 Inference Autoscaling

LLM inference autoscaling has unique challenges:

- Cold start is slow (model load takes minutes).
- Replicas are expensive (GPUs cost dollars per hour, not cents).
- Traffic is bursty.
- Right metrics are non-obvious.

Patterns:
- **Always-warm replicas.** Keep a minimum number running even at low traffic.
- **Predictive scaling.** Scale based on historical patterns, not just current load.
- **Tiered model serving.** Cheap models for small loads, expensive models for surges.
- **Queue-based autoscaling.** Scale on queue depth, not CPU.
- **KEDA** — Kubernetes event-driven autoscaler, supports many event sources (Redis queues, Kafka, custom).
- **Knative Serving** — scale-to-zero for spiky workloads (acceptable when cold starts are tolerable).

## 15.9 Serverless AI

Serverless platforms (AWS Lambda, GCP Cloud Run, Azure Functions, Modal, Replicate, RunPod Serverless, Banana) handle scaling and idle-cost automatically. You pay per invocation.

Suitable for: spiky traffic, low-traffic features, prototypes, batch jobs.

Unsuitable for: latency-critical chat (cold starts), high-volume continuous traffic (per-invocation pricing exceeds reserved capacity).

Modal and Replicate are notable for GPU-aware serverless with reasonable cold-start mitigation.

## 15.10 Edge AI Deployment

Edge AI runs models on or near the user — on the device (phone, laptop), on a regional edge node (Cloudflare Workers AI, AWS Wavelength), or in retail/industrial appliances.

Drivers: latency, offline operation, privacy, cost (no per-token bill).

Constraints: limited memory, no GPU or weak GPU, intermittent connectivity. Models are heavily quantized (INT4, INT8) and often pruned or distilled.

The trend: more and more inference moves to the edge as model efficiency improves. A 1-3B model running locally can handle a large share of consumer AI workloads.

## 15.11 Deployment Strategies

**Rolling deployment.** Replace old pods with new ones gradually. Default in Kubernetes. Risk: bad versions reach users during the rollout.

**Blue-green deployment.** Run old (blue) and new (green) in parallel; switch traffic atomically. Fast rollback. Cost: double capacity during the switch.

**Canary deployment.** Send a small fraction of traffic to the new version; expand if metrics hold; roll back if not. Industry standard for risky changes. Tools: Argo Rollouts, Flagger.

**Shadow deployment.** Send a copy of production traffic to the new version without serving its responses to users. Compare outputs offline. Excellent for testing model changes before user exposure.

**Feature flags.** Toggle behavior without redeployment. Essential for AI because prompt and model changes carry quality risk.

For AI, blue-green or canary with shadow traffic is the gold standard. Roll forward with shadow first, canary with small percentage, then full.

## 15.12 Ingress, Load Balancers, and API Gateways

Traffic from the internet to your cluster passes through:

1. **CDN / DNS** — global routing, caching.
2. **Cloud load balancer** — regional entry point, TLS termination.
3. **Ingress controller** — Kubernetes-level HTTP routing (nginx, Traefik, Istio Gateway, Contour).
4. **API gateway** — auth, rate limiting, transformation (Kong, Tyk, cloud-managed gateways).
5. **Service** — Kubernetes service routes to pods.

For AI specifically, an **LLM gateway** (Chapter 6) often sits between the API gateway and the inference pods to handle model routing, caching, and provider failover.

Timeouts at each layer must accommodate long LLM responses. The default 30-60 second timeout on load balancers kills long completions. Audit timeouts top to bottom.

## 15.13 Service Mesh

A service mesh handles service-to-service communication: mTLS, retries, timeouts, circuit breakers, observability, traffic shifting. Two dominate:

**Istio** — feature-rich, complex, the enterprise default.

**Linkerd** — simpler, lightweight, built in Rust.

For AI deployments, service meshes are valuable when you have many internal services (vector DB, embedding server, reranker, LLM router, tool services). They enforce zero-trust networking and give consistent observability without per-service code.

Cost: meshes add latency (small but real) and complexity. Adopt when the service count justifies it.

## 15.14 CI/CD for GenAI

The pipeline for AI deployments has extra stages versus typical web apps:

- **Source.** Code, prompts, model config, tool definitions.
- **Build.** Container images, model artifacts.
- **Eval.** Run regression evals against the new prompts and models.
- **Test.** Standard unit/integration tests, plus AI-specific safety tests.
- **Staging deploy.** Canary or shadow traffic.
- **Production deploy.** Progressive rollout with auto-rollback on metric regression.

Eval gates between build and deploy are essential. A PR that worsens eval scores by 5% should fail the build, not be reviewed in PR comments.

## 15.15 CI/CD Tools

- **GitHub Actions** — broadly used, simple, integrates with GitHub.
- **GitLab CI** — full DevOps platform.
- **Jenkins** — legacy but ubiquitous.
- **CircleCI, Buildkite, Drone, Tekton** — alternatives.
- **ArgoCD** — GitOps continuous delivery for Kubernetes. Cluster state matches Git state.
- **FluxCD** — alternative GitOps tool.

ArgoCD or Flux is the modern default for Kubernetes deployments — declare desired state in Git, the controller reconciles.

## 15.16 Multi-Cloud Deployment

Reasons to deploy across clouds: disaster recovery, vendor leverage, regional GPU availability, regulatory residency.

Patterns:
- **Active-active.** Traffic served from multiple clouds simultaneously, routed by GeoDNS.
- **Active-passive.** One cloud serves; the other is warm standby.
- **Per-region selection.** Choose the cheapest GPU provider in each region.

Tools:
- **Crossplane** — Kubernetes-native multi-cloud infrastructure.
- **Terraform** — declarative infrastructure across providers.
- **Pulumi** — Terraform alternative in mainstream languages.

The hidden cost of multi-cloud is operational. Each cloud has its own quirks. Most teams should master one cloud first.

## 15.17 EKS, GKE, AKS Deployment

Managed Kubernetes services from AWS, Google, and Azure respectively. All three offer GPU node pools, integrated monitoring, and IAM integration. Choice typically follows your broader cloud commitment.

EKS strengths: largest ecosystem, deep AWS integration, Karpenter for fast autoscaling. GKE strengths: best-of-breed Kubernetes, strong networking, TPU support. AKS strengths: tight Azure integration, OpenAI proximity.

## 15.18 On-Prem Deployment

Reasons: data residency, regulatory, cost at scale, control over hardware. Stacks:
- **OpenShift** — Red Hat's enterprise Kubernetes.
- **Rancher** — Kubernetes management.
- **Vanilla Kubernetes** — kubeadm or kops on your own hardware.
- **VMware Tanzu** — for vSphere-based environments.

On-prem AI requires owning the full stack: physical GPUs, networking, cooling, storage, identity, observability. The total cost of ownership is rarely better than cloud at small scale; it becomes competitive at large, steady-state scale.

## 15.19 Networking — DNS, SSL, CDN, WAF

- **DNS.** Route 53, Cloudflare, NS1, GCP Cloud DNS. Use health checks and weighted routing for failover.
- **SSL/TLS.** Use Let's Encrypt or your cloud's certificate manager. Terminate TLS at the load balancer or ingress; mTLS internally via service mesh.
- **CDN.** Cloudflare, Fastly, CloudFront, Akamai. For chat APIs, CDN benefit is mostly in static asset delivery and DDoS protection.
- **WAF (Web Application Firewall).** Blocks common attack patterns. Cloudflare WAF, AWS WAF, GCP Cloud Armor. For AI APIs, add custom rules around prompt injection patterns and abusive token-volume requests.

## 15.20 Zero Trust Architecture

Zero trust assumes no implicit trust based on network location. Every service-to-service call is authenticated and authorized.

For AI: identity-aware proxies (Cloudflare Access, Pomerium), mTLS via service mesh, short-lived service tokens, OIDC for human users, fine-grained RBAC for tools. Especially important for agent systems with many tool integrations — the agent should never have ambient privileges.

## 15.21 Full Production Architecture

```
   Internet
      |
   DNS (geo + health checks)
      |
   CDN + WAF
      |
   Cloud LB (TLS termination)
      |
   Ingress (nginx / Istio gateway)
      |
   API gateway (auth, rate limit, routing)
      |
   LLM gateway (model routing, cache, observability)
      |
   +--- Chat service (FastAPI, SSE)
   +--- Search service
   +--- Agent service
      |
   Service mesh (mTLS, retries, observability)
      |
   +--- vLLM cluster (large model, H100 nodes)
   +--- vLLM cluster (small model, A10G nodes)
   +--- Embedding server
   +--- Reranker
   +--- Vector DB
   +--- Postgres / Redis
   +--- Kafka
      |
   Observability (Prometheus, Grafana, Loki, Tempo, OTEL)
      |
   Secrets (Vault, cloud Secrets Manager)
      |
   CI/CD (GitHub Actions, ArgoCD)
      |
   Infra as Code (Terraform)
```

This is roughly the shape of a serious GenAI deployment. The pieces vary by company; the layering does not.

## 15.22 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Managed Kubernetes | Operational simplicity | Cloud bill |
| Self-managed Kubernetes | Control, cost | Ops burden |
| Serverless inference | Idle cost zero | Cold starts, per-call cost |
| Always-on replicas | Low latency | Idle cost |
| Multi-cloud | Resilience, leverage | Complexity |
| Service mesh | Uniform networking | Latency, complexity |

## 15.23 Scaling Challenges

- **GPU node pools** are slow to scale; pre-provision warm capacity.
- **Image pull time** can dominate cold start; bake images into node AMIs or use registry caches.
- **Model weights** must be loaded into GPU memory; mount from a fast, shared storage layer.
- **Network egress** out of cloud regions is expensive; route traffic regionally.
- **Burst capacity** sometimes requires reserved instances months in advance for top GPUs.

## 15.24 Security Concerns

- Secrets management — never bake keys into images, use Vault or cloud Secrets Manager with workload identity.
- mTLS internally — every service authenticates every other service.
- Image signing and SBOMs — supply chain attacks are real.
- Network policies — restrict pod-to-pod traffic to what is needed.
- Audit logging — every model call, every config change, every secret access.

## 15.25 Monitoring Strategy

Beyond app-level metrics (Chapter 19), deployment-level metrics include:
- Pod restart counts and reasons.
- Image pull duration.
- Autoscaler events.
- GPU utilization per node and per pod.
- Network throughput and latency between services.
- Ingress error rates.
- Time-to-ready for new pods.

## 15.26 Cost Optimization

- Right-size GPU nodes per workload.
- Use spot/preemptible for stateless workloads.
- Set up autoscaler aggressively but with sane minimums for latency.
- Use reserved or committed-use discounts for steady state.
- Bin-pack many small models on shared GPUs via MIG.
- Audit egress traffic monthly.

## 15.27 Interview Questions

- Walk through a request from internet to GPU.
- Compare rolling, blue-green, canary, and shadow deployments. When is each right?
- Design an autoscaling strategy for bursty LLM traffic with cold-start sensitivity.
- How do you handle GPU node pool scaling?
- Describe the security boundaries from edge to model.

## 15.28 Hands-on Exercises

1. Sketch a Kubernetes deployment for a chat product, identifying every layer in 15.21 for your case.
2. Plan a canary rollout for a new prompt that you suspect may regress quality.
3. Design a multi-region failover where the primary region runs frontier models and the failover runs smaller ones.

## 15.29 Common Mistakes

- 30-second timeouts somewhere in the stack truncating long LLM streams.
- Autoscaling on CPU when the bottleneck is GPU memory or KV cache.
- Mounting model weights from slow storage, blowing up cold start.
- Service mesh adopted prematurely — adds complexity without payback at low service count.
- Secrets baked into images.

## 15.30 Enterprise Best Practices

GitOps for everything. Eval gates in CI. Progressive rollouts by default. Standardize on one Kubernetes platform. Multi-region from day one if you serve global users. Capacity planning quarterly for GPUs. Document every layer's owner and on-call rotation. Treat the deployment platform as a product with its own SLOs.
