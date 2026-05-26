# Chapter 28 — Compliance

## 28.1 Concept Explanation

Compliance is the practice of conforming to regulatory, legal, or contractual requirements. For SRE, compliance translates abstract rules into specific technical and process controls. A SOC 2 audit asks "do you log access to customer data?" and the SRE answer is the actual audit log infrastructure, retention, and review process.

Compliance is not security. The two overlap but are different:
- **Security** = actually being safe.
- **Compliance** = demonstrating you meet a defined standard.

A system can be secure but not compliant (missing documentation). A system can be compliant but not secure (checking boxes without substance). Aim for both.

## 28.2 The Major Frameworks

**SOC 2.** Service Organization Control 2. The most common B2B SaaS compliance. Audits trust principles: security, availability, processing integrity, confidentiality, privacy.

**ISO 27001.** International information security standard. Common in Europe and for global enterprise sales.

**HIPAA.** US healthcare. Patient health information must be protected with specific controls. AI vendors need BAAs.

**PCI DSS.** Payment Card Industry Data Security Standard. Required if you touch credit card data.

**GDPR.** EU data protection. Lawful basis, right to deletion, data residency, transparency.

**CCPA / CPRA.** California's privacy laws. Similar to GDPR in spirit.

**FedRAMP.** US federal cloud. Strict; long approval process; lucrative if you achieve it.

**ISO 27701, 27017, 27018.** Various privacy and cloud extensions to ISO 27001.

**HITRUST.** Healthcare-specific framework that combines HIPAA and other standards.

**EU AI Act.** Emerging European regulation on AI systems.

The frameworks overlap significantly. Achieving one helps with others.

## 28.3 SOC 2 in Depth

The most common framework. Two types:
- **Type I.** Point-in-time audit of controls.
- **Type II.** Audit over a period (typically 6-12 months) of controls actually operating.

Most customers want Type II.

Trust Service Criteria:
1. **Security.** Default. Always included.
2. **Availability.** Usually included for SaaS.
3. **Processing Integrity.** Less common.
4. **Confidentiality.** Common for B2B.
5. **Privacy.** Common for consumer-facing.

Controls covered: access management, change management, monitoring, incident response, vendor management, BCP/DR, encryption, vulnerability management, employee training, more.

The audit is a process. An auditor examines your evidence (logs, policies, screenshots) against the controls.

## 28.4 The Audit Lifecycle

A typical audit:
1. **Scoping.** What systems and processes are in scope.
2. **Gap assessment.** Where you fall short.
3. **Remediation.** Fix the gaps.
4. **Evidence collection.** Logs, screenshots, policies during the audit period.
5. **Auditor review.** They examine evidence.
6. **Report.** SOC 2 report (Type I or Type II).
7. **Annual renewal.**

For Type II, you must operate the controls for months before the audit, since the auditor examines operation, not just existence.

## 28.5 Common Controls SREs Implement

Most SOC 2 / ISO 27001 controls map to specific SRE work:

- **Audit logs** — every privileged action recorded.
- **Access reviews** — quarterly verification of who has access to what.
- **Change management** — every prod change reviewed, approved, recorded.
- **Encryption** — at rest and in transit.
- **Vulnerability scans** — regular, with remediation.
- **Incident response** — documented process, exercised.
- **BCP/DR** — plans documented, tested.
- **Backup** — tested.
- **Monitoring** — uptime, integrity, security events.
- **Vendor management** — third-party risk assessments.
- **Onboarding/offboarding** — access provisioning/deprovisioning.
- **Employee training** — security awareness.

A control's existence isn't enough; you must have evidence it operated consistently.

## 28.6 Data Residency

For GDPR and similar regulations, where data is physically stored matters. EU citizen data must (usually) stay in EU. Some regulations are stricter (data sovereignty laws).

Implementation:
- **Per-region deployments.**
- **Tenant pinning** — each customer's data in their required region.
- **Data classification** — knowing what's sensitive.
- **Egress controls** — preventing data from crossing region boundaries.

Multi-region adds complexity; data residency demands it.

## 28.7 Right to Deletion (GDPR Article 17)

When a user requests deletion, you must delete their data within a defined timeframe (typically 30 days).

Complications:
- Vector indices and ML training data.
- Backups (must eventually be deletable or aged out).
- Replicated stores.
- Logs that contain user data.
- Analytics aggregates.

Building a reliable deletion pipeline is non-trivial. Plan it during initial architecture, not after the first request.

## 28.8 Data Processing Agreements

For GDPR, every vendor that touches EU data must sign a Data Processing Agreement (DPA) defining their obligations.

SRE relevance:
- DPA list for every cloud vendor.
- BAA (HIPAA's equivalent) for healthcare vendors.
- Sub-processor management.

## 28.9 Encryption Requirements

Most frameworks require:
- **Encryption at rest** for sensitive data.
- **Encryption in transit** (TLS 1.2+).
- **Key management** documentation.
- **Approved algorithms** (no MD5, no DES).

PCI DSS adds:
- Specific requirements for cardholder data.
- Quarterly vulnerability scans.
- Network segmentation.

## 28.10 Audit Logs as Compliance Evidence

Audit logs are central to most compliance:
- **Immutability** — cannot be tampered with after recording.
- **Retention** — typically 1-7 years.
- **Coverage** — all privileged actions, data access, config changes.
- **Reviewable** — searchable, with reporting.

Tools: SIEM platforms, immutable S3 buckets, dedicated audit log services.

## 28.11 Change Management

Every production change must be:
- Reviewed.
- Approved.
- Documented.
- Reversible (or with documented rollback).

GitOps + PR review + ArgoCD provides natural change management. The PR is the change record; the CI/CD pipeline is the evidence of approval.

## 28.12 Access Reviews

Periodic verification of who has access to what:
- Quarterly cadence (more often for highest-privilege).
- List access per system.
- Manager approves continued access.
- Remove unused or unjustified access.

Audit tools and IAM platforms automate this. The discipline is sustaining it.

## 28.13 Vendor Risk Management

You inherit your vendors' risk. Practices:
- **Security questionnaires** when onboarding vendors.
- **SOC 2 / ISO 27001 reports** from vendors annually.
- **Contractual obligations** — DPAs, breach notification, SLAs.
- **Inventory** of vendors processing data.
- **Periodic review** — drop vendors that no longer meet bar.

## 28.14 Compliance as Code

Modern practice: encode compliance requirements as automated checks.

Examples:
- **CIS Benchmarks** for cloud configurations (Wiz, Prisma Cloud).
- **OPA / Gatekeeper** policies in Kubernetes.
- **Terraform Sentinel** for IaC compliance.
- **Drata, Vanta, Secureframe** — automated compliance evidence platforms.

Compliance-as-code shifts compliance from periodic audit prep to continuous verification.

## 28.15 Real-World Use Cases

- A startup pursuing first enterprise customer needed SOC 2 Type II. Spent 6 months building evidence; achieved the report; signed the customer.
- A healthcare AI company needed HIPAA + SOC 2 + HITRUST. Multi-year program.
- A consumer app pivoted to enterprise. GDPR was already in place; SOC 2 took 4 months.

## 28.16 Production Architecture (Compliance Layer)

```
   Infrastructure with compliance baseline:
     - VPC with isolation
     - Encryption at rest (KMS)
     - TLS everywhere
     - Audit logging to immutable store
     - Vulnerability scanning automated
     - Access via SSO + MFA
     - Workload identity for services
        |
   Process layer:
     - Change management via GitOps
     - Access reviews quarterly
     - Vendor risk management
     - Incident response procedures
     - DR tested
        |
   Compliance automation:
     - Drata / Vanta / Secureframe collecting evidence
     - CIS / NIST benchmarks via tooling
     - Continuous configuration scanning
        |
   Audit:
     - Type II evidence over period
     - Auditor review
     - Report
```

## 28.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Multi-framework | Broad sales reach | Maintenance overhead |
| Single framework | Focused | Limited markets |
| Compliance-as-code | Continuous | Setup investment |
| Manual compliance | Lower upfront | Painful audit prep |
| Strict access | Auditable | Engineer friction |
| Tooling-heavy | Less manual | Tool cost |

## 28.18 Scaling Challenges

- More services = more controls.
- More vendors = more risk management.
- Geographic expansion = more frameworks.
- Audit prep takes increasing effort without automation.

## 28.19 Security vs Compliance

The mature view: aim for security; achieve compliance as a byproduct.

Teams that chase compliance checkboxes without underlying security build fragile systems. Teams that build secure systems make compliance straightforward.

## 28.20 Deployment Guide

Starting a compliance program:
1. Identify target frameworks (driven by sales / market need).
2. Gap assessment.
3. Pick a compliance platform (Drata, Vanta, Secureframe).
4. Implement missing controls.
5. Operate for the required period.
6. Engage an auditor.
7. Annual renewal.

## 28.21 Monitoring Strategy

- Control failures.
- Audit log coverage.
- Access review completion rate.
- Vulnerability remediation SLAs.
- Backup test success.
- Drill completion.
- Vendor review status.

## 28.22 Cost Optimization

- Compliance platforms cost $10k-$100k+ per year. Worth it for the time saved.
- Reuse evidence across frameworks (one log set for SOC 2 and ISO 27001).
- Open-source tools where possible.
- Avoid scope creep — only what is required for the framework.

## 28.23 Interview Questions

- *Compare SOC 2 and ISO 27001.*
- *What does HIPAA require technically?*
- *Walk through implementing GDPR deletion.*
- *What is compliance-as-code?*
- *How does SRE work map to SOC 2 controls?*

## 28.24 Hands-on Exercises

1. List the SOC 2 controls that map to your current SRE work.
2. Identify three gaps that would block an audit.
3. Plan a deletion workflow for one type of personal data in your system.

## 28.25 Common Mistakes

- Compliance theater (boxes ticked, security not improved).
- Auditing for the audit, not maintaining controls year-round.
- Vendors signed without risk review.
- No documented incident response.
- Audit logs missing critical actions.
- Backup retention not aligned with deletion requirements.

## 28.26 Enterprise Best Practices

Compliance platform from early stage. Continuous evidence collection. Quarterly access reviews. Vendor risk management. Compliance-as-code in CI. Senior SRE involvement in audit prep. Cross-functional team (security, legal, engineering) for compliance.
