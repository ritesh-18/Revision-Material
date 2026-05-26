# Chapter 25 — Security for SRE

## 25.1 Concept Explanation

Security is not a separate function. It is part of every SRE's job. An unreliable system can be made reliable; an insecure system can be made trustworthy; but a system that is reliably insecure is worse than one that is unreliably secure. Security failures destroy trust in ways outages do not.

This chapter covers what SREs must know about security. It does not replace the security team; it makes you a useful partner to them and lets you catch most production security issues before they become incidents.

## 25.2 The Security Mindset

Three foundational mental shifts:

**Assume compromise.** Some part of your system is already breached. Design for limiting damage.

**Defense in depth.** No single control should be the only barrier. Layer them.

**Least privilege.** Default deny. Grant only what is needed. Time-limit where possible.

These are not aspirations — they are operational principles applied daily.

## 25.3 The Attack Surface

For a typical SaaS:
- Public web traffic (HTTP/HTTPS).
- API endpoints.
- DNS records.
- Email and DNS verification records.
- Third-party integrations.
- Employee laptops and accounts.
- CI/CD pipelines.
- Open-source dependencies (supply chain).
- Cloud account itself.

Each is a potential entry point. Map yours.

## 25.4 OWASP Top 10

The classic list of common web vulnerabilities (current edition):
1. Broken Access Control.
2. Cryptographic Failures.
3. Injection (SQL, NoSQL, command, LDAP).
4. Insecure Design.
5. Security Misconfiguration.
6. Vulnerable and Outdated Components.
7. Identification and Authentication Failures.
8. Software and Data Integrity Failures.
9. Security Logging and Monitoring Failures.
10. Server-Side Request Forgery (SSRF).

Every SRE should be able to explain each.

## 25.5 IAM Security

Identity and Access Management is the foundation. The principles:

**Federate identity.** Use a central identity provider (Okta, Azure AD, Google Workspace) for human users. Avoid local accounts on systems.

**Use roles, not users, for services.** AWS IAM Roles + STS; GCP Workload Identity; Azure Managed Identity.

**Short-lived credentials.** Tokens that expire in hours, not years. Rotate keys.

**MFA everywhere.** Especially for human access to production and CI.

**Review access regularly.** Permissions drift. Stale grants are vulnerabilities.

**Separate accounts/projects.** Dev, staging, prod with strict boundaries.

## 25.6 Network Security

Layered network defenses:

**Public/private subnets.** Internet-facing assets in public; everything else private. NAT for egress.

**Security groups / firewall rules.** Default deny; allow specific traffic.

**Service mesh mTLS.** Internal traffic encrypted and authenticated.

**Network policies in Kubernetes.** Pod-to-pod traffic restricted.

**Zero trust.** No implicit trust based on network location.

**DDoS protection.** Cloud-native (Shield, Cloud Armor) or third-party (Cloudflare).

## 25.7 Secrets Management

Secrets (API keys, DB passwords, signing keys) deserve specific care.

Practices:
- **Never in code.** Even private repos get leaked.
- **Never in plaintext config.** Encrypt or reference.
- **Centralized storage.** Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault.
- **Workload identity for services.** Pods get credentials at runtime via identity, not files.
- **Short TTL where possible.** Database creds that auto-rotate.
- **Audit logs on access.**

Chapter 26 covers secrets management in depth.

## 25.8 Encryption

**At rest.** All data encrypted on disk. Cloud KMS or customer-managed keys.

**In transit.** TLS everywhere. mTLS internally where feasible.

**In use.** Confidential computing (Intel SGX, AMD SEV) for the most sensitive workloads. Rare.

Encryption is table stakes; the interesting work is key management.

## 25.9 Vulnerability Management

Known vulnerabilities in software you run. Process:

**Inventory.** Know what you run (SBOM — Software Bill of Materials).

**Scan.** Image scanning (Trivy, Grype, Snyk), dependency scanning (Snyk, Dependabot), infrastructure scanning (Prisma Cloud, Wiz).

**Triage.** CVE severity is a starting point; actual risk depends on exposure.

**Patch.** Critical vulnerabilities in production must be patched fast (hours to days).

**Track.** SLAs per severity. Audit compliance.

## 25.10 Supply Chain Security

Modern attacks compromise dependencies, not your code. Defenses:

- **Pin dependencies.** Lock files, hash verification.
- **Image signing.** cosign, Sigstore.
- **SBOMs** for every artifact.
- **Verify provenance.** SLSA framework.
- **Private registries** for internal artifacts.
- **Review new dependencies.** Smaller blast radius if you say no to obscure libraries.

Known incidents: SolarWinds, log4shell, xz backdoor. Supply chain is the new perimeter.

## 25.11 The Principle of Least Privilege

Apply to:
- Human accounts.
- Service accounts.
- Pods (Kubernetes RBAC, ServiceAccounts).
- API keys.
- CI/CD pipelines.
- Network access.
- Data access.

Implementation:
- Default deny.
- Grant per-purpose, per-environment.
- Time-bound grants where possible.
- Audit and revoke.

## 25.12 Privileged Access Management

For the most sensitive access (production database, root accounts):

- **Just-in-time access.** Engineer requests access for a specific task; granted for a window.
- **Approval flow.** Another engineer approves.
- **Session recording.** Audit what was done.
- **Break-glass procedure.** Emergency access with later review.

Tools: Teleport, StrongDM, AWS SSO with JIT, Cloud Identity-Aware Proxy.

## 25.13 Logging and Detection

Security depends on visibility. Log:
- Authentication events.
- Authorization decisions.
- Privileged actions.
- Access to sensitive data.
- Config changes.
- Network connections from unexpected sources.

Aggregate to a SIEM (Splunk, Sentinel, Sumo, Chronicle, Wazuh open-source).

Detection rules:
- Failed logins from unusual locations.
- Access patterns inconsistent with role.
- Privileged actions outside business hours.
- Large data exports.

## 25.14 Incident Response — Security Edition

Security incidents have legal and reputational dimensions that operational incidents do not.

Process:
1. Detect.
2. Contain (limit damage).
3. Eradicate (remove the threat).
4. Recover.
5. Investigate root cause.
6. Notify (legal, regulators, customers) where required.
7. Document.
8. Hardening based on lessons.

Legal involvement is mandatory. Do not communicate externally without their input.

## 25.15 DDoS Defense

Layered:
- **CDN / edge** absorbs most volumetric attacks (Cloudflare, AWS Shield).
- **Rate limiting** at the gateway.
- **WAF rules** for known patterns.
- **Application-level** rate limits.
- **Capacity headroom** to absorb spikes.

DDoS is more of an availability problem than a security one, but the defenses are similar.

## 25.16 Web Application Firewall (WAF)

Filters HTTP traffic for known attack patterns:
- SQL injection signatures.
- XSS patterns.
- Common exploits (log4shell-like).
- Bot detection.
- Geo-blocking.

WAFs are imperfect — false positives and negatives both. Use as a layer, not a sole defense.

## 25.17 Real-World Use Cases

- A startup leaked an AWS access key in a public commit. Within an hour, attackers had spun up $30k of crypto mining. Lesson: secret scanning in CI.
- A misconfigured S3 bucket exposed customer data. Found by a security researcher. Lesson: continuous configuration auditing.
- A vulnerable log4j version in a dependency allowed remote code execution. Patched in hours because the team knew their dependencies. Lesson: SBOMs and fast patch capability.

## 25.18 Production Architecture (Security)

```
   Edge: Cloudflare / AWS Shield / WAF
        |
   DDoS protection
        |
   Authentication (OAuth / SSO)
        |
   Authorization (RBAC / ABAC at app)
        |
   Service mesh (mTLS internally)
        |
   Network policies (default deny)
        |
   Secrets Management (Vault) for all credentials
        |
   Audit logging (SIEM)
        |
   Detection rules and alerts
        |
   Incident response with legal involvement
        |
   Regular: vulnerability scans, access reviews, pen tests
```

## 25.19 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Strict access | Secure | Engineer friction |
| Loose access | Convenient | Risk |
| Long-lived secrets | Simple | Rotation gaps |
| Short-lived | Safer | Operational complexity |
| WAF aggressive | Block more attacks | False positives |
| WAF lax | Fewer false positives | Misses attacks |

## 25.20 Scaling Challenges

- Access reviews at scale (thousands of users, hundreds of resources).
- Detection volume (large companies see millions of suspicious events daily).
- Vendor coordination (multi-cloud security).
- Cross-region encryption key management.

## 25.21 Security as Code

Modern practice: security policies expressed as code.
- **OPA / Gatekeeper** — admission control for Kubernetes.
- **Sentinel** — Terraform policy as code.
- **Cedar** — AWS open-source policy language.
- **Kyverno** — Kubernetes policy.

This makes policies testable, reviewable, versioned.

## 25.22 Deployment Guide

A baseline security setup:
1. Federate identity via SSO.
2. MFA everywhere.
3. Centralized secrets management.
4. Image scanning in CI.
5. Dependency scanning weekly.
6. WAF + DDoS protection at edge.
7. Network policies in Kubernetes.
8. mTLS via service mesh.
9. Audit logging to SIEM.
10. Detection rules.
11. Quarterly access reviews.
12. Annual pen test.

## 25.23 Monitoring Strategy

- Failed authentication rate.
- Anomalous access patterns.
- Privileged action volume.
- Configuration drift (compared to known good).
- Vulnerable component count.
- Secret access patterns.
- Network policy violations.

## 25.24 Cost Optimization

- Security tools can be expensive; consolidate where reasonable.
- Cloud-native security (Security Hub, Cloud Armor) is often cheaper than enterprise alternatives.
- Open-source options (Wazuh, Falco, OPA) for tight budgets.

## 25.25 Interview Questions

- *Explain defense in depth.*
- *What is least privilege and how do you enforce it?*
- *Walk through how you secure a service end to end.*
- *What is mTLS and why?*
- *Describe a security incident response.*

## 25.26 Hands-on Exercises

1. Audit your IAM policies. List three to tighten.
2. Find every place secrets live in your codebase or env. Migrate to Secrets Manager.
3. Plan an access review for one critical resource.

## 25.27 Common Mistakes

- Secrets in code or env.
- Long-lived AWS access keys.
- Permissive IAM ("Effect: Allow, Action: *").
- No MFA for human accounts.
- Default service account permissions in Kubernetes.
- Logs that miss security events.
- No incident response playbook.

## 25.28 Enterprise Best Practices

Centralized identity. MFA everywhere. JIT access for production. mTLS internal. Secrets management mandatory. SIEM with tuned alerts. Quarterly access reviews. Pen tests annually. Security training for all engineers. Bug bounty program for mature companies.
