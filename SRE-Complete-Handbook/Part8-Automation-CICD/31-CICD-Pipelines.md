# Chapter 31 — CI/CD Pipelines

## 31.1 Concept Explanation

Continuous Integration (CI) is the practice of merging code changes frequently with automated validation. Continuous Delivery (CD) extends this to automated deployment to production-like environments. Continuous Deployment goes further — every passing change reaches production automatically.

A mature CI/CD pipeline is the SRE's most powerful safety mechanism. Bugs caught at PR are cheaper than bugs caught at staging, which are cheaper than bugs caught at canary, which are vastly cheaper than bugs caught in production. The pipeline shifts cost left.

## 31.2 The Pipeline Stages

A typical pipeline:

```
   Push / PR
        |
   Lint + Type-check
        |
   Unit tests
        |
   Build artifact (container image)
        |
   Security scan (image, dependencies)
        |
   Push to registry
        |
   Deploy to dev
        |
   Integration tests
        |
   Deploy to staging
        |
   E2E tests / smoke tests
        |
   Approval gate (or automated)
        |
   Canary deploy to prod (1-5%)
        |
   Soak period with metrics watch
        |
   Full prod rollout
        |
   Post-deploy verification
```

Each stage takes time and resources. The art is making the right tradeoff per stage.

## 31.3 CI Tools

**GitHub Actions** — dominant for projects on GitHub. Easy syntax. Free for public, generous free tier for private.

**GitLab CI** — full DevOps platform. Strong for GitLab-hosted repos.

**CircleCI** — third-party, mature, fast.

**Buildkite** — strong for monorepos and large pipelines.

**Jenkins** — old, ubiquitous, configurable, painful to maintain.

**Drone, TeamCity, Bamboo, AWS CodeBuild, GCP Cloud Build, Azure DevOps** — alternatives.

**Tekton, Argo Workflows** — Kubernetes-native pipelines.

Most teams pick by where their code lives. GitHub Actions is the default for GitHub-hosted projects.

## 31.4 CD Tools

**ArgoCD** — GitOps for Kubernetes. The dominant new pattern.

**Flux** — alternative GitOps tool. Lighter.

**Spinnaker** — Netflix's deployment tool. Powerful, complex.

**Octopus Deploy** — enterprise CD.

**Harness** — commercial DevOps platform.

**AWS CodeDeploy, Azure DevOps** — cloud-native.

ArgoCD is the modern default for Kubernetes deployments.

## 31.5 Build Reproducibility

A build should produce identical artifacts from identical inputs. Reasons:

- **Caching** — reuse layers across builds.
- **Debugging** — reproduce a past build's behavior.
- **Security** — verify what was built.

Practices:
- Pin all dependencies.
- Lockfiles for package managers.
- Pinned base images (not `:latest`).
- Hermetic builds where possible.

Reproducibility matters more than people realize. "It worked yesterday" usually means a non-reproducible build.

## 31.6 The Build Step

Container builds: Dockerfile + builder.

Modern practices:
- **Multi-stage builds.** Compile in one stage, copy artifacts to lean runtime stage.
- **BuildKit features.** Cache mounts, secrets, parallel stages.
- **Layer ordering.** Cache-friendly — most-changing layers last.
- **Distroless or slim bases.** Smaller, fewer vulnerabilities.
- **Non-root user.** Security baseline.

Image size matters: smaller images pull faster, scan faster, have less attack surface.

## 31.7 Testing in the Pipeline

Layers of testing:

**Unit tests.** Fast, run on every commit. Cover business logic.

**Integration tests.** Slower. Test interactions with dependencies (DB, cache).

**Contract tests.** Verify API contracts between services. Pact, Spring Cloud Contract.

**E2E tests.** Slowest. Verify user flows.

**Performance tests.** Load test with regression detection.

**Security tests.** Image scan, dependency scan, SAST, secret detection.

A good pipeline runs fast tests on every push and slower tests less frequently.

## 31.8 The Test Pyramid

Many fast unit tests; fewer integration tests; even fewer E2E tests. Inverted (lots of E2E, few unit) is slow and flaky.

For SRE-owned pipelines:
- Smoke tests after every deploy.
- E2E tests for critical user flows nightly.
- Synthetic checks running constantly.

## 31.9 Image Registries

Built images live in registries:
- **Docker Hub** — public default.
- **GitHub Container Registry (GHCR)** — bundled with GitHub.
- **AWS ECR, GCR, ACR** — cloud-native.
- **Harbor, JFrog Artifactory** — private/enterprise.

Practices:
- **Private repos** for proprietary code.
- **Tags pinned in deployments** (no `:latest`).
- **Immutable tags** — never reuse a tag.
- **Scanning at push.**
- **Image signing** (cosign).
- **Retention policy** to clean old tags.

## 31.10 Deployment Strategies

Already covered in Chapter 15. Recap for CD context:

**Rolling** — Kubernetes default. Replace pods gradually.

**Blue-green** — old and new exist simultaneously; switch atomic.

**Canary** — small percentage to new; expand if metrics hold.

**Shadow** — duplicate traffic to new version, do not serve responses.

**Feature flags** — deploy with code path disabled; enable separately.

Modern best practice: deploy via flags, canary by traffic percentage, observe metrics, expand or rollback.

## 31.11 Progressive Delivery

A generation beyond canary: tools that automate the canary → full process with metric checks.

- **Argo Rollouts** — Argo-native, integrates with metrics for auto-progress and auto-rollback.
- **Flagger** — similar, integrates with various service meshes.

These tools watch SLI metrics during canary. If error rate spikes, they roll back automatically. The team is notified but not on the critical path.

## 31.12 GitOps

A specific CD pattern:
- **Git is source of truth** for desired state.
- **Operators continuously reconcile** actual state to Git state.
- **PRs are the deploy mechanism.**

Benefits:
- Audit trail (Git history).
- Reproducible (re-apply at any point).
- Self-service (devs make PRs, not tickets).
- Easy rollback (revert the PR).

Tools: ArgoCD, Flux, Jenkins X.

Covered in Chapter 32.

## 31.13 Secrets in CI/CD

CI runners need access to deploy. Practices:
- Workload identity for cloud access (GitHub OIDC + AWS IAM).
- Short-lived tokens.
- Secret scanning to catch leaks in code.
- Restricted access to CI secrets.

Long-lived static credentials in CI are a serious risk; they have caused breaches.

## 31.14 Pipeline Performance

Slow pipelines kill productivity. Optimizations:

- **Parallelism.** Tests, builds in parallel.
- **Caching.** Dependencies, build layers.
- **Incremental** — only build/test what changed (relevant in monorepos).
- **Distributed test runners.**
- **Selective execution** — only run affected tests.

Target: under 10 minutes for full pipeline. Above 30 minutes, engineers context-switch and lose state.

## 31.15 Pipeline as Code

Pipelines defined in version-controlled YAML. Every change is reviewed. No clicking in UI.

This is non-negotiable for production-grade pipelines.

## 31.16 Real-World Use Cases

- A team's deploys took 45 minutes. After cache and parallelism work: 8 minutes. Deploys went from 2/day to 10/day.
- A pipeline lacked image scanning. A vulnerability shipped to production. After: scanning blocking on critical CVEs.
- A team adopted Argo Rollouts. Auto-rollback caught and reverted a bad deploy in 90 seconds, before users noticed.

## 31.17 Production Architecture

```
   Developer pushes code or opens PR
        |
   GitHub Actions runs:
   +-- Lint, type-check, unit tests
   +-- Container build (cached)
   +-- Security scan
   +-- Push image to registry
        |
   Merge to main
        |
   ArgoCD detects change in deployment manifest
        |
   Deploys to dev cluster
        |
   E2E tests run against dev
        |
   Promote to staging (manifest update via PR or automation)
        |
   ArgoCD deploys to staging
        |
   Promote to prod
        |
   Argo Rollouts canaries (10% → 50% → 100% with metric checks)
        |
   Full deploy
        |
   Post-deploy synthetic checks
```

## 31.18 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Continuous deployment | Velocity | Risk |
| Manual approval gate | Safety | Slower |
| Heavy test suite | Catches bugs | Slow pipeline |
| Light test suite | Fast | Misses bugs |
| Self-hosted runners | Control | Maintenance |
| Hosted CI | No ops | Cost, less control |

## 31.19 Scaling Challenges

- Monorepo pipelines explode in size.
- Many services × many environments = pipeline matrix.
- Test data management.
- Flaky tests pollute trust.

## 31.20 Security

- Image scanning blocking on critical CVEs.
- Dependency scanning (Snyk, Dependabot).
- Secret scanning.
- SAST (static application security testing).
- Image signing.
- SBOM generation.
- Supply chain attestations (SLSA).

## 31.21 Deployment Guide

A starter pipeline:
1. GitHub Actions for CI.
2. Lint + unit tests on every PR.
3. Container build + scan.
4. Push to private registry.
5. ArgoCD for CD to dev/staging.
6. Canary for prod with Argo Rollouts.
7. Synthetic checks post-deploy.

## 31.22 Monitoring Strategy

- Pipeline duration (p50, p95).
- Failure rate.
- Flaky test frequency.
- Time from commit to production.
- Rollback rate.
- Deploys per day.

DORA metrics:
- Deployment frequency.
- Lead time for changes.
- Mean time to restore.
- Change failure rate.

## 31.23 Cost Optimization

- Cache aggressively.
- Self-hosted runners for high-volume.
- Spot instances for CI workers.
- Don't run heavy tests on every PR — risk-based.

## 31.24 Interview Questions

- *Walk through your CI/CD pipeline.*
- *Compare blue-green and canary.*
- *What is GitOps?*
- *Explain progressive delivery.*
- *How do you handle secrets in CI?*

## 31.25 Hands-on Exercises

1. Set up GitHub Actions for a small project: lint, test, build, push.
2. Add image scanning that blocks merges on critical CVEs.
3. Configure ArgoCD to deploy from a Git repo.
4. Add a canary rollout via Argo Rollouts.

## 31.26 Common Mistakes

- Long pipelines that engineers ignore.
- Tests that have been flaky for months.
- Manual deploys "for emergencies" (becomes the norm).
- Latest tag in production.
- Secrets as long-lived CI secrets instead of workload identity.

## 31.27 Enterprise Best Practices

GitOps. Workload identity for CI. Mandatory security scanning. Pipeline-as-code. DORA metrics tracked. Progressive delivery default. Postmortems for pipeline-caused incidents. Periodic pipeline review and optimization.
