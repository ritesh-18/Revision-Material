# Chapter 26 — Secrets Management

## 26.1 Concept Explanation

A secret is any piece of data that gives access if known: API keys, database passwords, signing keys, OAuth tokens, encryption keys, TLS private keys. Secrets management is the discipline of generating, storing, distributing, rotating, and auditing secrets safely.

The principle: secrets should never exist in plain text outside their intended use. This is the difference between a system that survives a credential leak and one that does not.

## 26.2 Where Secrets Live

In a well-run system:
- **Stored** — in a dedicated secret store with encryption, access control, audit.
- **Distributed** — to consumers at runtime via short-lived credentials or workload identity.
- **Used** — in process memory only; never written to disk.
- **Rotated** — automatically on a schedule.
- **Revoked** — fast when compromised.
- **Audited** — every access logged.

In a poorly run system, secrets live in code, environment variables exported in shells, Kubernetes Secrets stored as base64, slacked between engineers, written on whiteboards.

## 26.3 The Anti-Patterns

The common bad patterns:

**Secrets in code.** Even private repos get leaked. GitHub secret scanning catches some; old commits forever expose what was once committed.

**Plain text in env files.** `.env` files in dev are fine; in prod, they end up in container images, in shell history, in screenshots.

**Long-lived API keys.** A key from 5 years ago that still works is a 5-year-old vulnerability.

**Shared secrets.** Multiple services or people sharing one credential. Cannot audit individual access. Cannot rotate without coordination.

**Secrets in URLs.** `https://...?api_key=...` ends up in server logs, browser history, referrer headers.

Every one of these has caused major breaches. Industry standard practice is now built around avoiding them.

## 26.4 Secret Stores

Dedicated systems for secrets:

**HashiCorp Vault.** The gold standard. Open-source core, enterprise features. Self-hosted or HCP Vault (managed). Supports dynamic secrets, secret leasing, broad integrations.

**AWS Secrets Manager.** Native AWS, integrates with IAM, automatic rotation for some databases.

**AWS Parameter Store (SSM).** Cheaper than Secrets Manager, less feature-rich, suitable for config.

**GCP Secret Manager.** Native GCP, IAM-integrated.

**Azure Key Vault.** Native Azure.

**1Password / Bitwarden / Doppler.** Developer-focused, simpler than Vault.

**Kubernetes Secrets.** Native to K8s but base64-encoded, not encrypted by default. Improved by:
- Enabling encryption at rest in etcd.
- External Secrets Operator to sync from real secret stores.
- Sealed Secrets (Bitnami) for git-encryption.

Most teams use a primary secret store (Vault or cloud-native) plus External Secrets Operator to integrate with Kubernetes.

## 26.5 Workload Identity

The modern pattern: services do not have credentials; they have **identities**, and they get short-lived credentials at runtime.

How it works:
- Pod has an identity (Kubernetes ServiceAccount).
- Cloud trusts that identity (IAM Roles for Service Accounts on AWS; Workload Identity on GCP; Managed Identity on Azure).
- Pod requests a token; the cloud verifies the identity, issues a short-lived credential.
- Application uses the credential; it expires quickly; renews as needed.

Result: no stored credentials anywhere. No keys to rotate. No leak risk.

Implementing workload identity is one of the highest-leverage security investments. Do it.

## 26.6 Dynamic Secrets

Some secret stores can generate credentials on demand:
- Vault creates a database user with limited TTL.
- App receives the temporary username and password.
- After TTL, Vault revokes the user.

This means there are no long-lived database credentials at all. Each service or session gets its own, short-lived.

Setup is more complex; the security gain is large.

## 26.7 Secret Rotation

Static credentials should be rotated regularly. Practices:

- **Automated rotation** for supported services (Secrets Manager + RDS).
- **Quarterly rotation** for static keys.
- **Immediate rotation** on suspected compromise.
- **Rotation without downtime** via dual credentials (old and new both work during transition).

Manual rotation is brittle. Automate.

## 26.8 Encryption Keys

Some secrets *are* encryption keys. Manage these specially:

**Customer-managed keys (CMK).** You hold the key; cloud provider stores it for you. AWS KMS, GCP Cloud KMS, Azure Key Vault.

**Hardware Security Modules (HSM).** Keys live in tamper-resistant hardware.

**Envelope encryption.** Data encrypted with a data key; data key encrypted with a master key. Reduces master key usage.

**Key rotation policies.** New master key on a schedule; old keys retained for decryption.

For regulated data, key management is a board-level concern.

## 26.9 Certificate Management

TLS certificates are a special kind of secret with their own lifecycle.

Tools:
- **cert-manager** (Kubernetes) — automates ACME / Let's Encrypt.
- **HashiCorp Vault PKI** — internal CA for service certificates.
- **ACME / Let's Encrypt** — public certificates, free, automated.
- **Commercial CAs** for higher-assurance certificates.

Certificate expiry is a recurring outage cause. Automate renewal; alert long before expiry.

## 26.10 Secret Detection

Find leaked secrets:
- **Pre-commit hooks** — git-secrets, detect-secrets.
- **CI scanning** — Trufflehog, GitGuardian.
- **GitHub Secret Scanning** — built-in for public and private repos with the feature enabled.
- **Cloud scanning** — AWS Macie, GCP DLP for objects.

Even with prevention, scans catch leaks that slip through.

## 26.11 Response to Compromise

When a secret is leaked or suspected leaked:
1. **Revoke immediately.** Do not wait for confirmation.
2. **Issue replacement.**
3. **Update all consumers.**
4. **Audit usage of the old secret.** Look for unauthorized access.
5. **Notify** legal / security / customers if required.
6. **Postmortem.**
7. **Harden** the source of the leak.

The discipline: revoke first, investigate later. Letting a leaked secret live shortly is often worse than the impact of rotation.

## 26.12 External Secrets Operator (ESO)

A Kubernetes operator that syncs secrets from external stores (Vault, AWS Secrets Manager, etc.) into Kubernetes Secrets, which pods then consume normally.

Why: pods don't have to learn about Vault; Kubernetes Secret semantics work; secrets stay in the real secret store.

Pattern:
- ExternalSecret custom resource references a secret in the external store.
- ESO syncs the value into a regular K8s Secret.
- Pods consume the Secret as usual.
- ESO refreshes on a schedule.

## 26.13 Real-World Use Cases

- Uber's 2016 breach started with credentials in a private GitHub repo.
- Code Spaces, a cloud hosting provider, was destroyed in 2014 when attackers got AWS root credentials.
- Twilio's 2022 breach used stolen employee credentials.

In each, secret management would have made the difference.

## 26.14 Production Architecture (Secrets)

```
   Vault (central secret store)
        |
        +-- Static secrets (rotated periodically)
        +-- Dynamic secrets (generated on demand)
        +-- PKI (internal CA)
        |
   External Secrets Operator syncs to K8s
        |
   Pods consume via Secret volumes or env
        |
   For cloud access: Workload Identity (no secrets)
        |
   Audit logs from Vault to SIEM
        |
   Alerts on unusual access
```

## 26.15 Tradeoffs

| Approach | Win | Cost |
|---|---|---|
| Cloud Secret Manager | Native, easy | Per-secret cost at scale |
| Vault self-hosted | Powerful, flexible | Operational burden |
| Vault managed (HCP) | Managed | Vendor cost |
| Workload identity | No secrets | Setup complexity |
| Dynamic secrets | Best security | Most complex |
| Static rotated | Simple | Rotation discipline needed |

## 26.16 Scaling Challenges

- Vault performance at very large scale.
- Secret count grows with services.
- Multi-region replication.
- Cross-cloud secret sync.

## 26.17 Security

- Secret store itself is the highest-value target. Harden it.
- Audit logs immutable.
- Break-glass procedures for store recovery.
- Backups of the store, encrypted, separately.

## 26.18 Deployment Guide

A pragmatic adoption:
1. Pick a secret store (Vault for power; cloud-native for simplicity).
2. Enable encryption at rest for K8s Secrets in etcd.
3. Install External Secrets Operator.
4. Migrate the highest-risk secrets first (production DB credentials).
5. Set up workload identity for cloud access.
6. Roll out dynamic secrets for databases.
7. Enable secret scanning in CI.
8. Quarterly review of secrets and access.

## 26.19 Monitoring Strategy

- Secret access rate.
- Failed access attempts.
- Time since last rotation.
- Number of static credentials.
- Audit log volume.
- Secret store health.

## 26.20 Cost Optimization

- Cloud Secret Manager fees scale with secret count and access. Audit.
- Vault is free open-source; operational cost is the real expense.
- Workload identity is free and eliminates ongoing cost of secret management for cloud access.

## 26.21 Interview Questions

- *Why is putting secrets in env vars considered bad?*
- *Explain workload identity.*
- *Compare Vault and AWS Secrets Manager.*
- *Walk through a secret leak response.*
- *What are dynamic secrets?*

## 26.22 Hands-on Exercises

1. Audit one of your repos for secrets. Use a tool like Trufflehog.
2. Migrate one production secret from env to a real secret store.
3. Plan workload identity adoption for one service.

## 26.23 Common Mistakes

- Secrets in code.
- Long-lived access keys for services.
- Sharing one credential across services or engineers.
- No rotation.
- No audit on access.
- Kubernetes Secrets in plain text (unencrypted etcd).

## 26.24 Enterprise Best Practices

Centralized secret store mandatory. Workload identity for all cloud access. Dynamic secrets for databases where possible. Quarterly rotation audits. Pre-commit secret scanning. Incident response playbook for leaked secrets. Annual review of secret store access patterns.
