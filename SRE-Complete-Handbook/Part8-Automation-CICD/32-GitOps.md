# Chapter 32 — GitOps

## 32.1 Concept Explanation

GitOps is a deployment pattern where Git is the single source of truth for system state, and automated operators continuously reconcile the live system to match Git. Pull requests are the change mechanism.

The defining shift: instead of pipelines pushing changes into the cluster, the cluster pulls its desired state from Git. The cluster becomes a controller observing Git, not an endpoint receiving commands.

Four principles of GitOps:
1. **Declarative.** Desired state in code.
2. **Versioned.** Git stores all state.
3. **Auto-applied.** Operators reconcile continuously.
4. **Observable.** Drift is visible and alertable.

## 32.2 Why GitOps

Before GitOps:
- Pipelines run `kubectl apply` and hope.
- No drift detection.
- Manual changes diverge from source.
- Rollback means re-running a pipeline.
- Audit trail scattered across CI logs and cluster.

With GitOps:
- Git history is the audit.
- Drift is automatically detected and reconciled.
- Rollback is a Git revert.
- Multi-cluster is one pattern, replicated.
- Anyone can read what should be deployed where.

## 32.3 ArgoCD

The dominant GitOps tool for Kubernetes.

How it works:
- Configure Argo to watch a Git repo.
- Argo polls (or webhooks) for changes.
- When desired state changes, Argo applies to the cluster.
- Argo reports sync status (synced, out of sync, healthy, degraded).
- UI shows current state vs Git state.

Features:
- Sync policies (automatic, manual approval).
- Sync waves (ordering of resources).
- App of apps (a single app references many).
- Multi-cluster from one Argo instance.
- RBAC integration.
- Notifications.

ArgoCD is mature, widely adopted, and the default for new Kubernetes deployments.

## 32.4 Flux

The other major GitOps tool. CNCF graduated. Lighter than Argo, more components.

Differences:
- Flux is more modular (separate controllers for Helm, Kustomize, image automation).
- Argo has a richer UI; Flux is more CLI/CR-driven.
- Flux supports image automation natively (detect new images, update Git).

Pick by team preference. Both are excellent.

## 32.5 Repo Strategies

Where to put GitOps manifests:

**Monorepo.** Everything in one repo. Simple discovery. Painful at scale.

**Repo per environment.** `cluster-prod`, `cluster-staging`. Clear separation.

**Repo per team.** Each team owns its services. Argo aggregates.

**App-of-apps with shared infrastructure repo.** Modern pattern: shared infra repo for cluster-level stuff, app repos for services.

The right choice depends on org structure and scale.

## 32.6 Kustomize and Helm in GitOps

Two ways to template manifests:

**Kustomize.** Base + overlays. No templating language. Each environment is an overlay.

**Helm.** Charts with templating. Variables per environment.

ArgoCD and Flux both support both. Use Helm for third-party packages (Prometheus, ArgoCD itself); use Kustomize or plain manifests for internal services.

Some teams use both side by side.

## 32.7 The Sync Loop

```
   ArgoCD Application points at Git repo + path + branch
        |
   ArgoCD detects change (poll or webhook)
        |
   Computes diff (live state vs Git)
        |
   If auto-sync: apply changes
   If manual: surface for approval
        |
   Resources applied
        |
   Health checks evaluate
        |
   Status updated (synced + healthy / out of sync / degraded)
        |
   Notifications sent on state changes
```

The loop runs every few minutes. Continuous reconciliation.

## 32.8 Drift Detection

When live state differs from Git, that's drift. ArgoCD shows this explicitly.

Causes:
- Manual `kubectl` changes.
- Other controllers modifying resources.
- Failed applies leaving partial state.

Response:
- **Auto-sync** re-applies Git state automatically.
- **Manual** surfaces for human decision.

Auto-sync is opinionated: Git is right; the cluster is wrong. This is usually correct but occasionally surprising.

## 32.9 Secrets in GitOps

Secrets can't go in Git in plaintext. Options:

**Sealed Secrets (Bitnami).** Encrypt secrets in Git; cluster decrypts.

**External Secrets Operator.** Sync from external store (Vault) into K8s Secrets.

**SOPS** with various backends. Edit secrets locally; encrypted in Git.

**HashiCorp Vault sidecar injector.** Mount secrets into pods at runtime.

Most modern stacks use External Secrets Operator with Vault or cloud Secret Manager.

## 32.10 Multi-Cluster GitOps

ArgoCD can manage many clusters from one instance. Patterns:

**Hub and spoke.** Central ArgoCD manages many clusters.

**Per-cluster ArgoCD.** Each cluster has its own. More resilient to ArgoCD outage.

**Hybrid.** Per-region ArgoCD; each manages a few clusters.

For large organizations, per-region with cross-region failover is common.

## 32.11 Promotion Workflows

Code flows from dev to staging to prod. Common patterns:

**Branch-based.** Dev branch deploys to dev; main to prod.

**Path-based.** `environments/dev/`, `environments/prod/`. Changes flow via PRs.

**Tag-based.** Same manifests; different tag references per environment.

**Image automation.** Tools (Flux Image Automation, ArgoCD Image Updater) detect new images and create PRs to update.

The path-based approach is most common and most explicit.

## 32.12 Rollback

GitOps makes rollback trivial:
- Revert the PR (`git revert`).
- ArgoCD detects the change, applies the previous state.

This is much safer than re-running a pipeline; the previous state is exactly what was running before.

## 32.13 Real-World Use Cases

- A platform team adopted ArgoCD. Onboarded 50 services in a quarter. Engineers self-serve deploys via PRs.
- A team had drifted manually for years. ArgoCD enforcement caused initial pain (live state diverged from anyone's understanding) but stabilized after a month.
- Multi-cluster GitOps replicated a complete setup to a new region in a day, where the old method took weeks.

## 32.14 Production Architecture

```
   Code repos (per service)
        |
   CI builds image, updates Git config repo
        |
   Config repo (manifests, Helm values, Kustomize overlays)
        |
   ArgoCD watches config repo
        |
   Applies to target clusters
        |
   Multi-cluster: ArgoCD application instances per cluster
        |
   Drift detection runs continuously
        |
   Notifications to Slack on degraded apps
        |
   Audit via Git history
```

## 32.15 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| GitOps | Audit, declarative, easy rollback | Learning curve |
| Push-based CD | Familiar | Drift, harder rollback |
| ArgoCD | Rich UI, mature | Single component |
| Flux | Modular, lightweight | Less unified UX |
| Auto-sync | Continuous correctness | Surprising sometimes |
| Manual sync | Human in loop | Slower |

## 32.16 Scaling Challenges

- Large Argo instances handle thousands of apps but require tuning.
- Git repo size and clone time.
- Per-tenant or per-customer manifests at scale.
- Audit log volume.

## 32.17 Security

- Git as source of truth: protect it.
- Branch protection, required reviews.
- Signed commits (Sigstore Gitsign).
- ArgoCD RBAC integrated with org SSO.
- Secrets via External Secrets, not in Git.

## 32.18 Deployment Guide

Adopting GitOps:
1. Install ArgoCD (or Flux) in a cluster.
2. Pick repo strategy (path-based per env recommended).
3. Move one service to GitOps.
4. Onboard more services.
5. Implement secrets handling (External Secrets + Vault).
6. Enable auto-sync.
7. Set up notifications.
8. Train teams.

## 32.19 Monitoring Strategy

- App sync status.
- Drift detection events.
- Failed syncs.
- ArgoCD itself (CPU, memory, queue depth).
- Time from PR merge to deployed state.

## 32.20 Cost Optimization

- ArgoCD/Flux themselves are free (open source).
- Cloud cost for cluster running them is modest.
- Cost savings from fewer outages and easier rollbacks usually exceed setup cost.

## 32.21 Interview Questions

- *What is GitOps and why?*
- *Compare ArgoCD and Flux.*
- *How do you handle secrets in GitOps?*
- *Walk through a deploy with GitOps.*
- *Multi-cluster ArgoCD patterns?*

## 32.22 Hands-on Exercises

1. Install ArgoCD locally. Connect to a Git repo. Deploy a sample app.
2. Update the manifest in Git. Watch ArgoCD apply.
3. Manually change a deployment. Watch ArgoCD revert.
4. Set up External Secrets to sync from Vault.

## 32.23 Common Mistakes

- Auto-sync enabled on day 1 — surprising changes.
- Secrets in Git unencrypted.
- One huge ArgoCD instance for everything.
- Skipping notifications — silent failures.
- Manual `kubectl` changes against GitOps (drift war).

## 32.24 Enterprise Best Practices

GitOps as default deployment pattern. ArgoCD/Flux with per-region instances. Path-based repo strategy. External Secrets + Vault. RBAC integrated with SSO. Notifications to Slack. Audit via Git. Quarterly review of repo structure as scale grows.
