# Chapter 30 — Configuration Management

## 30.1 Concept Explanation

Configuration management is the practice of defining, storing, distributing, and changing the runtime configuration of systems. Code is what your software does; configuration is how it does it in this environment.

The discipline matters because configuration is a frequent cause of outages — perhaps the most frequent. A typo in a YAML, a flag flipped globally, a stale value never updated: these cause incidents at rates rivaling actual code bugs. Treating configuration with engineering rigor cuts that rate dramatically.

## 30.2 Types of Configuration

Configuration falls into rough categories:

**Static config.** Doesn't change at runtime. Database URLs, feature lists, API endpoints. Baked into deployments.

**Dynamic config.** Changes without redeploy. Feature flags, rate limits, behavior toggles. Read at runtime.

**Secrets.** Credentials, keys. Covered in Chapter 26.

**Schema config.** Database migrations, API versions. Has data implications.

Each type has different storage, change, and rollback needs.

## 30.3 The Twelve-Factor Config Principle

The classic guidance: strict separation of config from code. Same code, different configs for different environments. Config in environment variables, not files baked into images.

The principle still holds, modernized:
- Container images are environment-agnostic.
- Configuration injected at deploy time (env vars, mounted files, config services).
- No environment names in code ("if env == 'prod'" is an anti-pattern).

## 30.4 Configuration Sources

**Environment variables.** Native to all runtimes. Simple. Limited types (strings only). Visible in process listings.

**Config files.** Mounted into containers. Support structure (YAML, TOML). Reload requires restart unless app supports hot-reload.

**Config services.** Centralized stores (Consul KV, etcd, AWS App Config, GCP Runtime Config). Dynamic. Auditable.

**Feature flag services.** Specialized for flags (LaunchDarkly, Unleash, Flagsmith, Statsig). Cover more use cases than simple key-value.

**Kubernetes ConfigMaps.** K8s-native. Mounted as files or env. Limited dynamics.

Most teams use multiple: env vars for connection strings, ConfigMaps for app settings, a flag service for feature flags.

## 30.5 Feature Flags

A boolean (or richer) value that controls feature behavior at runtime. The single most valuable configuration pattern.

Use cases:
- **Kill switches** — disable features fast without redeploy.
- **Gradual rollouts** — enable for 1%, then 10%, then 100%.
- **A/B tests** — different behavior for different users.
- **Long-lived feature work** — merge in progress; enable when ready.
- **Customer-specific** — enable for one customer's tenant.

Without feature flags, every change is high-stakes. With them, most are reversible.

## 30.6 Flag Lifecycle

Flags are not free. Each one is conditional code that complicates testing and reasoning.

Lifecycle:
1. **Created** with clear purpose.
2. **Rolled out** progressively.
3. **Validated** at full rollout.
4. **Removed** — code cleaned up, flag deleted.

Stale flags accumulate. Six-month-old flags pointing at code paths no one tests are tech debt. Quarterly cleanup is essential.

## 30.7 Flag Targeting

Modern flag services support rich targeting:
- By user ID (specific users, percentages).
- By tenant.
- By geography.
- By user attributes (paid plan, beta tester).
- By time window.

Targeting enables sophisticated rollouts (canary by tenant, beta by opt-in) without writing custom code.

## 30.8 LaunchDarkly, Unleash, and Friends

**LaunchDarkly** — feature-rich, expensive, the established leader.

**Unleash** — open-source, self-hostable, capable.

**Flagsmith** — open-source, growing.

**Statsig** — strong on experimentation in addition to flags.

**ConfigCat, Split.io, PostHog Feature Flags** — alternatives.

**Cloud-native** — AWS AppConfig, GCP Cloud Config.

Pick by feature needs, cost, and self-hosting preference.

## 30.9 Configuration as Code

Like infrastructure, configuration benefits from being in source control:
- Versioned.
- Reviewable.
- Auditable.
- Replicable.

The pattern: a Git repo (or set of repos) contains config files. CI validates. CD applies to config stores. Changes are PRs.

This avoids "someone changed something somewhere" mysteries.

## 30.10 Configuration Validation

Configuration errors are easy to make and hard to catch. Validation defenses:

- **Schema** — JSON Schema, Cue, Open Policy Agent — to enforce structure.
- **Type-checked configuration** — strong typing in application code.
- **Linting** — yamllint, hadolint for Dockerfiles.
- **Dry runs** — preview changes without applying.
- **Canary** — apply to a subset before all.

The mantra: never trust a config file without validation.

## 30.11 Configuration Rollout Strategies

A bad config in production is worse than a bug — bugs require a deploy to ship, configs can apply instantly.

Patterns:
- **Gradual rollout** — apply to one instance, then more.
- **Percentage rollout** — to 5% of pods, then 25%, then 100%.
- **Canary by environment** — staging → 1% prod → 100% prod.
- **Auto-rollback** — if metrics degrade, revert automatically.

The slower the rollout, the safer. Critical configs deserve hour-or-day rollouts.

## 30.12 Drift and Reconciliation

When live config differs from declared, that's drift. Common causes:
- Manual changes.
- Failed apply leaving partial state.
- Side effects from other tools.

Tools that continuously reconcile (ArgoCD, Flux, Kubernetes controllers) reduce drift by re-applying continuously.

## 30.13 Configuration Audit

Who changed what, when?

- **Git history** for IaC and code-stored config.
- **Config service audit logs** for runtime stores.
- **Feature flag audit** — LaunchDarkly and others log every toggle.

Audit logs are evidence for postmortems and compliance.

## 30.14 Real-World Use Cases

- A config change set a timeout to 0. Every request immediately failed. Found because canary rollout caught it before full deploy.
- A feature flag for a new pricing model was enabled accidentally for all customers. Refunds were issued. Lesson: targeting + gradual rollout.
- A team's prod and staging configs diverged silently over a year. A "staging passed, prod failed" outage finally caught it. Lesson: enforce consistency.

## 30.15 Production Architecture

```
   Config repo (Git)
        |
   PR review + CI validation
        |
   Merge triggers CD
        |
   Apply to config store:
   +-- Kubernetes ConfigMaps
   +-- Vault for secrets
   +-- LaunchDarkly for flags
   +-- Consul for service discovery
        |
   Services read at runtime
        |
   Audit logs to SIEM
        |
   Drift detection compares actual vs declared
```

## 30.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Env vars | Simple | No structure, no dynamics |
| ConfigMaps | K8s-native | Restart for changes |
| Config service | Dynamic | Operational overhead |
| Feature flags | Powerful | Tech debt accumulation |
| Static config | Predictable | Slow to change |
| Dynamic config | Fast change | Risk |

## 30.17 Scaling Challenges

- Feature flag count grows; cleanup discipline required.
- Config service availability becomes a dependency.
- Multi-region config consistency.
- Different configs per tenant at scale.

## 30.18 Security

- Audit who can change what configs.
- Sensitive configs encrypted.
- Config services authenticate clients.
- Flags can change behavior dramatically — guard with permissions.

## 30.19 Deployment Guide

Standing up modern config management:
1. Strict separation of config from code.
2. Pick a config service (start with ConfigMaps + Vault).
3. Add a feature flag service.
4. CI validation of config changes.
5. Gradual rollout standard.
6. Quarterly flag cleanup.

## 30.20 Monitoring Strategy

- Config service health.
- Flag evaluation rate.
- Config change rate.
- Drift detection results.

## 30.21 Cost Optimization

- Feature flag services charge per MAU; right-size plan.
- Self-host (Unleash) for scale beyond hosted plans.
- Clean up unused flags.

## 30.22 Interview Questions

- *Why separate config from code?*
- *Feature flag lifecycle?*
- *Compare LaunchDarkly and Unleash.*
- *How do you validate config changes?*
- *What is drift?*

## 30.23 Hands-on Exercises

1. Audit your service's configuration sources. List each.
2. Identify three flags older than 6 months. Plan their removal.
3. Add schema validation to one config file.

## 30.24 Common Mistakes

- Config in code (recompile to change).
- No feature flag cleanup.
- Global config rollout (no canary).
- Manual changes drift production from source.
- Stale flags hide dead code.

## 30.25 Enterprise Best Practices

Strict 12-factor separation. Config in Git. Feature flag service with audit. Quarterly flag cleanup. Gradual config rollout default. Validation in CI. Drift detection. Per-tenant configs where needed.
