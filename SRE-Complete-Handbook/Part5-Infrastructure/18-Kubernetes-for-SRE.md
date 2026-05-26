# Chapter 18 — Kubernetes for SRE

## 18.1 Concept Explanation

Kubernetes (K8s) is the dominant container orchestration platform. Whether you love it or hate it, you operate it. This chapter covers Kubernetes from an SRE perspective — what to know, what breaks, what to monitor, what to optimize.

A working mental model: Kubernetes is a declarative platform. You describe the desired state (5 replicas of this service, 4Gi memory each, exposed on port 80). Kubernetes continuously reconciles actual state to match desired state. When something fails, the platform self-heals to bring it back.

The complexity comes from the number of moving parts: control plane, etcd, kubelet, container runtime, scheduler, controllers, ingress, networking plugins, storage plugins, security policies. Each can be a failure point.

## 18.2 The Control Plane

The brain of Kubernetes:

**API server.** The front door. All requests go through it. Validates, authenticates, persists to etcd.

**etcd.** The state store. Distributed, strongly consistent. If etcd is unhealthy, nothing else works.

**Scheduler.** Assigns pods to nodes. Considers resources, affinities, taints, etc.

**Controller manager.** Hosts the controllers — small loops that watch resource state and act to reconcile it.

**Cloud controller manager.** Cloud-specific integrations (load balancers, persistent volumes).

In managed Kubernetes (EKS, GKE, AKS), you do not run the control plane. The provider does. In self-hosted (kubeadm, kops), you do.

## 18.3 Nodes and the Kubelet

Each node runs:
- **Kubelet** — agent that talks to the API server and runs containers.
- **Kube-proxy** — networking magic for service IPs.
- **Container runtime** — containerd (modern), CRI-O, formerly Docker.

Nodes can be bare metal, VMs, or specialized (GPU nodes, ARM nodes). The cluster autoscaler adds and removes them based on pod demand.

## 18.4 Core Objects

**Pod.** Smallest deployable unit. One or more containers sharing network and storage. Usually one container per pod.

**ReplicaSet.** Ensures N copies of a pod template exist.

**Deployment.** Wraps ReplicaSet with rolling updates and history.

**StatefulSet.** For stateful workloads — stable network IDs, ordered startup, persistent storage.

**DaemonSet.** Runs one pod per node. Used for node-level services (logging agent, metrics).

**Job / CronJob.** Run-to-completion workloads.

**Service.** Stable network endpoint pointing at a set of pods.

**Ingress.** HTTP routing into the cluster.

**ConfigMap / Secret.** Configuration and sensitive data.

**Namespace.** Logical grouping for multi-tenancy.

**PersistentVolume / PersistentVolumeClaim.** Storage.

Knowing these by name and behavior is the entry-level requirement.

## 18.5 The Pod Lifecycle

```
   Deployment / ReplicaSet creates a new Pod object
        |
   Scheduler picks a node
        |
   Kubelet on that node pulls the image
        |
   Container runtime starts the container
        |
   Pod transitions: Pending → ContainerCreating → Running
        |
   Liveness probe: is it alive?
   Readiness probe: is it ready for traffic?
   Startup probe: is initialization complete?
        |
   Service includes the pod in endpoints once Ready
        |
   Pod runs, served traffic
        |
   Pod terminates (deletion, crash, eviction)
   - SIGTERM sent
   - graceful shutdown period
   - SIGKILL if it doesn't exit
        |
   ReplicaSet creates a replacement
```

Most production debugging starts with `kubectl describe pod` to see where in this lifecycle a pod is stuck.

## 18.6 Probes

Three types:

**Liveness probe.** "Is the container still working?" Failure → restart the container.
**Readiness probe.** "Is the container ready to receive traffic?" Failure → remove from service endpoints.
**Startup probe.** "Has initialization finished?" Useful for slow-starting apps; liveness/readiness only apply after startup succeeds.

Misconfigured probes are a top cause of Kubernetes flakiness:
- Liveness too aggressive → constant restarts.
- Readiness too lax → traffic to broken pods.
- Probes hitting expensive endpoints → cascading failures.

## 18.7 Networking in Kubernetes

The Kubernetes networking model:
- Each pod gets its own IP.
- Pods can talk to each other directly (flat network).
- Services provide stable IPs (cluster-internal).
- Ingress provides HTTP routing from outside.

CNI plugins implement this:
- **Calico** — feature-rich, network policies, BGP routing.
- **Cilium** — eBPF-based, modern, fast, increasingly the default.
- **Flannel** — simple, older.
- **AWS VPC CNI** — native AWS networking.
- **Azure CNI**, **GKE CNI** — cloud-specific.

Network debugging in Kubernetes is hard because there are many layers. Tools: `kubectl exec`, `tcpdump`, network policy debugging.

## 18.8 Services and Ingress

**Service types:**
- **ClusterIP** — internal only.
- **NodePort** — exposed on every node's port (rarely used directly).
- **LoadBalancer** — cloud LB provisioned.
- **ExternalName** — DNS alias.

**Ingress** is HTTP routing. Common ingress controllers:
- **nginx-ingress** — most popular, mature.
- **Traefik** — feature-rich, complex.
- **AWS Load Balancer Controller** — native AWS integration.
- **Istio Gateway** — when using Istio.
- **Envoy-based** — Ambassador, Contour.

**Gateway API** is the modern successor to Ingress, with richer semantics. Adoption growing.

## 18.9 Resource Requests and Limits

Every container should specify:
- **Requests** — guaranteed allocation (scheduling and QoS).
- **Limits** — maximum allowed.

Memory limit exceeded → container killed (OOMKilled).
CPU limit exceeded → throttled.

QoS classes:
- **Guaranteed** — requests = limits.
- **Burstable** — requests < limits.
- **BestEffort** — neither set.

For production, set requests and limits. BestEffort pods get evicted first under pressure.

A common pitfall: setting limits too tight. Kubernetes is aggressive about enforcing them. Headroom matters.

## 18.10 Autoscaling

**HPA (Horizontal Pod Autoscaler)** — scales pod count based on metrics (CPU default; custom metrics via adapter).

**VPA (Vertical Pod Autoscaler)** — adjusts requests/limits. Less common; requires restart.

**Cluster Autoscaler** — adds/removes nodes when pods cannot schedule.

**Karpenter** (AWS-led) — modern alternative to Cluster Autoscaler. Faster, more flexible.

**KEDA** — event-driven autoscaling (queue depth, custom metrics).

Tuning autoscalers is an SRE specialty. Common mistakes:
- Wrong metric (CPU when memory is the bottleneck).
- Too-aggressive scaling (thrashing).
- Min replicas = 0 (cold-start incidents).
- Max too low (cannot absorb spikes).

## 18.11 RBAC

Kubernetes Role-Based Access Control. Components:
- **Role / ClusterRole** — permissions.
- **RoleBinding / ClusterRoleBinding** — assign role to subject.
- **ServiceAccount** — identity for pods.

For production, RBAC is non-negotiable. Default service accounts often have too much access. Audit regularly.

## 18.12 Network Policies

By default, Kubernetes pods can talk to each other freely. Network policies restrict this.

Patterns:
- Default deny all traffic.
- Allow specific ingress/egress per pod label.
- Tenant isolation in multi-tenant clusters.

CNI must support network policies. Calico and Cilium do; some older CNIs do not.

## 18.13 Common Production Issues

**ImagePullBackOff** — cannot pull image. Check registry credentials, image name, network.

**CrashLoopBackOff** — container exits, restarted, exits again. Check container logs.

**OOMKilled** — exceeded memory limit. Increase limit or fix the leak.

**Pending pods** — scheduler cannot find a node. Check resource availability, taints, node selectors.

**Evicted pods** — node had pressure (memory, disk). Investigate node health.

**Service not reachable** — check Service endpoints, NetworkPolicy, kube-proxy.

**Ingress 502/504** — check upstream pod readiness, ingress controller logs.

Each of these has a typical investigation path. Build runbooks.

## 18.14 Operators and CRDs

**CRD (Custom Resource Definition)** lets you define new Kubernetes object types.

**Operator** is a controller that manages those custom resources, encoding operational knowledge.

Common operators:
- **Prometheus Operator** — manages Prometheus instances.
- **cert-manager** — manages TLS certificates.
- **External Secrets Operator** — syncs secrets from external stores.
- **Postgres Operator** (Zalando, CrunchyData) — Postgres on K8s.
- **Strimzi** — Kafka on K8s.

Operators turn Kubernetes into a platform for managing complex stateful workloads.

## 18.15 GPU Support

For AI workloads, Kubernetes needs:
- **NVIDIA GPU Operator** — manages drivers, device plugin, exporters.
- **Node labels** for GPU type.
- **Pod requests** for `nvidia.com/gpu`.
- **MIG** for partitioning A100/H100.
- **Time-slicing** for sharing GPUs across pods.

GPU scheduling is its own world. Covered in the GenAI handbook in depth.

## 18.16 Real-World Use Cases

- A startup runs everything on EKS. One cluster per environment. Standard pattern.
- A FAANG-scale company has hundreds of Kubernetes clusters across regions. Multi-tenant. Custom operators. Heavy investment.
- A regulated enterprise runs on-prem Kubernetes (OpenShift) with strict network policies and audit.

## 18.17 Production Architecture

```
   Cloud provider managed control plane (EKS, GKE, AKS)
        |
   Worker node pools
   +-- General-purpose
   +-- Memory-optimized
   +-- GPU
   +-- Spot/preemptible
        |
   CNI (Cilium or Calico)
        |
   Ingress controller (nginx, ALB Controller, Istio Gateway)
        |
   Workloads (Deployments, StatefulSets, DaemonSets)
        |
   Storage (EBS / persistent volumes)
        |
   Service mesh (Istio, Linkerd) optional
        |
   GitOps (ArgoCD, Flux)
        |
   Observability (Prometheus, Loki, Tempo via Helm)
```

## 18.18 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Managed K8s | No control plane ops | Premium cost |
| Self-hosted | Full control | Heavy operational burden |
| One big cluster | Resource sharing | Blast radius |
| Many small clusters | Isolation | Operational overhead |
| Cilium CNI | Modern, fast | Less mature than Calico |
| Calico CNI | Mature | Less feature-velocity |

## 18.19 Scaling Challenges

- Etcd is the limiting factor at very large clusters (5000+ nodes).
- Many small objects (Secrets, ConfigMaps) slow the API server.
- Multi-tenant clusters with noisy neighbors.
- Cross-cluster federation is not solved well.

## 18.20 Security

- RBAC: least privilege.
- Network policies: default deny.
- Pod security: no root, no privileged.
- Image scanning.
- Admission controllers (OPA, Kyverno).
- Service mesh for mTLS.
- Secrets management (External Secrets + Vault).

## 18.21 Deployment Guide

For a new cluster:
1. Choose managed (EKS/GKE/AKS) or self-hosted.
2. Pick CNI (Cilium recommended).
3. Set up node pools with appropriate instance types.
4. Install ingress controller.
5. Install observability (kube-prometheus-stack).
6. Install cert-manager.
7. Install external-secrets-operator.
8. Configure RBAC and namespaces.
9. Bring up GitOps (ArgoCD).
10. Deploy a sample app.

## 18.22 Monitoring Strategy

For Kubernetes itself:
- API server latency and error rate.
- etcd health (database size, latency, leadership changes).
- Node health (Ready status, resource usage).
- Pod health (running, restarts, evictions).
- HPA scaling events.
- Cluster autoscaler events.

## 18.23 Cost Optimization

- Right-size requests (most pods over-request).
- Use spot nodes for tolerant workloads.
- Karpenter for fast, right-sized node provisioning.
- Multi-tenant clusters (with limits).
- Resource quotas per namespace.
- Bin-packing improvements.

## 18.24 Interview Questions

- *Walk through what happens when you `kubectl apply` a Deployment.*
- *Explain liveness vs readiness probes.*
- *What is the difference between a Service and an Ingress?*
- *How do you debug a pod stuck in Pending?*
- *Compare HPA, VPA, Cluster Autoscaler, Karpenter.*

## 18.25 Hands-on Exercises

1. Stand up a kind cluster locally. Deploy nginx. Expose via Service. Curl it.
2. Add an HPA based on CPU. Generate load. Watch it scale.
3. Inject an OOMKilled scenario and observe the controller behavior.
4. Set up an Ingress with TLS via cert-manager.

## 18.26 Common Mistakes

- No resource requests/limits (BestEffort pods get evicted).
- Probes that hit expensive endpoints (cascading failures under load).
- Default service accounts with broad permissions.
- One huge namespace for everything.
- StatefulSets used where Deployments would do.
- Manual changes to live cluster (drift from declarative source).

## 18.27 Enterprise Best Practices

GitOps from day one (ArgoCD or Flux). Network policies default-deny. RBAC reviewed quarterly. Production readiness gates for new services. Image scanning in CI. Centralized observability. Backup of etcd. Disaster recovery plan including K8s itself.
