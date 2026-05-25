# Chapter 26 — Enterprise AI Engineering

## 26.1 Concept Explanation

Enterprise AI is the deployment of GenAI inside large organizations with the constraints that come with size: governance, compliance, audit, multi-tenancy, integration with existing systems, risk management. The technical patterns are not fundamentally different from startup AI. The discipline is.

A startup ships fast; an enterprise ships *safely*. The architecture must accommodate legal, security, procurement, change management, and procurement-driven vendor selection. The deployment must satisfy regulators. The operations must produce audit evidence at any moment.

## 26.2 Governance

AI governance is the system of policies, processes, and oversight that ensures AI use aligns with the organization's risk posture and regulatory obligations.

Components:
- **AI Council or Committee.** Cross-functional body (legal, security, engineering, business) that approves AI use cases.
- **Use case inventory.** Every AI feature registered, with owner, purpose, data inputs, risk class.
- **Risk classification.** Tiered (high-risk, medium-risk, low-risk) with different review depths.
- **Acceptable use policy.** What is and is not allowed.
- **Vendor evaluation.** Documented process for selecting AI vendors.

Without governance, every team ships AI independently, and compliance gaps appear at audit time.

## 26.3 Compliance

Compliance varies by industry. The most common frameworks for AI:

**SOC 2.** Service Organization Control 2. Information security controls audited annually. Most B2B SaaS requires it.

**GDPR.** EU data protection. Affects any system processing EU resident data. Lawful basis, right to deletion, data residency, transparency obligations all apply to AI.

**HIPAA.** US healthcare. Patient health information must be protected with specific controls. AI vendors must sign BAAs.

**PCI DSS.** Payment card data. AI features touching card data require segmented infrastructure.

**EU AI Act.** Tiered regulation by AI risk class. High-risk AI (employment, credit, education) faces strict obligations. Even general-purpose models face transparency obligations.

**FedRAMP.** US federal compliance. AI vendors selling to federal customers must clear high security bars.

**ISO 27001 / 27701 / 42001.** International standards for information security and AI management.

Compliance is not a checkbox. It shapes architecture: data residency forces regional deployment, retention policies force deletion pipelines, audit requirements force pervasive logging.

## 26.4 Audit Logging

Audit logs answer: who did what, when, why. For AI:
- Every model call (user, model, prompt hash, output hash, cost).
- Every tool invocation.
- Every retrieval (which docs, by whom).
- Every prompt or model change.
- Every secret access.
- Every admin action.

Logs are immutable, retained per policy (often 1-7 years for regulated industries), reviewed periodically. The discipline is logging early and reliably, not after a regulator asks.

## 26.5 Policy Enforcement

Policies become real when enforced in code:
- **Content policies** enforced by classifiers on inputs and outputs.
- **Data classification** policies enforced by detectors before logging or training.
- **Access policies** enforced by RBAC/ABAC at every layer.
- **Tool use policies** enforced by gateways that scope tool calls.
- **Model use policies** enforced by routing — frontier models gated to high-trust use cases.

A policy engine (OPA, Cedar, Cerbos) lets policies live as code, separate from application logic, versioned and reviewed.

## 26.6 Enterprise Onboarding

Bringing a new AI feature into enterprise deployment is a process:

1. Use case proposal with risk assessment.
2. Security review.
3. Legal review (data, IP, vendor terms).
4. Pilot in a controlled environment.
5. Evaluation against acceptance criteria.
6. Production deployment with phased rollout.
7. Ongoing monitoring and review.

Faster startups skip these steps and ship; enterprises follow them and ship reliably. The latter scales better in regulated environments.

## 26.7 Multi-Tenant Isolation

Enterprises often serve multiple internal organizations or external customers. Isolation strategies:

- **Logical isolation.** One database, tenant_id on every row, enforced by app.
- **Schema isolation.** One database, separate schemas per tenant.
- **Physical isolation.** Separate database per tenant.
- **Per-tenant deployments.** Entirely separate stacks.

For AI specifically:
- Per-tenant vector indices or filtered shared indices.
- Per-tenant prompts and configurations.
- Per-tenant rate limits and budgets.
- Per-tenant audit logs.

The right answer depends on the customer's contract and regulatory needs. SOC 2 logical isolation usually suffices; HIPAA and finance often demand physical.

## 26.8 Vendor Risk Management

AI introduces vendor concentration risk. If your product depends on one provider and they go down, raise prices, or change terms, you are exposed.

Mitigations:
- **Multi-provider** via LLM gateway.
- **Open model fallback** for critical paths.
- **Vendor SOC 2 / ISO 27001** review before signing.
- **Data processing agreements** specifying retention, training, and incident notification.
- **Exit strategy** documented — what would migration look like if needed.

## 26.9 Data Handling

Enterprise data handling for AI:
- **Classify** data sensitivity (public, internal, confidential, restricted).
- **Minimize** data sent to AI services — don't send what you don't need.
- **Mask or tokenize** PII before sending to external providers.
- **Document** data flows for each AI feature.
- **Honor** deletion requests across all systems including vector stores and backups.

## 26.10 Change Management

In enterprises, change management is a process:
- Change advisory board (CAB) for high-risk changes.
- Documented rollout plans.
- Communicated maintenance windows.
- Pre-approved rollback plans.
- Postmortems for incidents.

For AI: model and prompt changes carry quality risk. Treat them as changes subject to CAB review for high-impact features.

## 26.11 Procurement

Enterprise procurement involves:
- RFPs (Request for Proposal).
- Vendor security questionnaires.
- Master service agreements.
- Data processing addendums.
- Pilot agreements before full rollout.

Procurement adds months to vendor adoption. Plan around it.

## 26.12 Internal AI Platforms

Many enterprises build an internal AI platform — a shared service that handles model access, retrieval, observability, and governance, used by many internal teams.

Components:
- LLM gateway with multi-provider routing.
- Embedding service.
- Vector DB with multi-tenant scoping.
- Prompt registry.
- Eval harness.
- Observability stack.
- Cost dashboards.
- Policy enforcement.

Building a platform amortizes the cost of compliance and observability across many use cases.

## 26.13 The AI Center of Excellence

Many enterprises establish an AI CoE — a centralized team that:
- Builds and operates the internal platform.
- Reviews and approves AI use cases.
- Maintains evaluation harnesses and golden datasets.
- Trains other teams on AI best practices.
- Tracks risk and cost across the portfolio.

Reports to the CTO or CIO; operates as a horizontal capability.

## 26.14 Regulatory Reporting

For high-risk AI under the EU AI Act and similar regimes:
- Conformity assessment before deployment.
- Post-market monitoring.
- Reporting of incidents to regulators.
- Transparency obligations (informing users they are interacting with AI).
- Human oversight requirements.

Build the infrastructure to produce these reports as a side effect of normal operations, not as a manual scramble.

## 26.15 Real-World Use Cases

- A bank deploys an internal copilot for relationship managers. Multi-tenant per branch, audit logs to a SIEM, every prompt reviewed by compliance.
- A hospital deploys clinical decision support. HIPAA-compliant vendor, BAA signed, no patient data leaves the VPC, every output reviewed by a clinician.
- A government contractor deploys document analysis. FedRAMP-authorized model, air-gapped deployment, audit logs to a federal system.

## 26.16 Tradeoffs

| Approach | Win | Cost |
|---|---|---|
| Centralized AI platform | Consistency, compliance | Slower team velocity |
| Federated AI | Fast team velocity | Compliance gaps |
| Strict governance | Risk control | Innovation slow |
| Light governance | Innovation fast | Risk exposed |
| Single vendor | Simple, leverage | Concentration risk |
| Multi-vendor | Resilience | Complexity, cost |

## 26.17 Scaling Challenges

- More teams using AI = more diverse needs to platform.
- More use cases = more risk to govern.
- More data = more residency and retention complexity.
- More regulators = more reports.

The platform team must scale alongside.

## 26.18 Security Concerns

Covered in Chapter 20 and amplified in enterprise context:
- Insider threats; audit logs on admin access.
- Vendor compromise; defense in depth.
- Regulatory penalties for breaches.
- Supply chain risk.
- Long-running incidents that accumulate quietly.

## 26.19 Cost at Enterprise Scale

- Negotiated rates with providers.
- Reserved capacity.
- Internal showback / chargeback to teams.
- FinOps practice integrated with AI.
- Multi-year forecasts for capacity planning.

## 26.20 Interview Questions

- Walk through governance for a new AI feature.
- How do you handle GDPR deletion requests for an AI system?
- Design a multi-tenant isolation strategy for healthcare.
- Compare centralized and federated AI organizational models.
- What changes about AI security when you are FedRAMP High?

## 26.21 Hands-on Exercises

1. Draft a one-page AI governance policy for a company you know.
2. Build the use case register entry for your design from Chapter 1: owner, risk class, data, controls.
3. Plan how you would produce evidence for a SOC 2 audit covering AI.

## 26.22 Common Mistakes

- Treating governance as a blocker instead of an enabler.
- Skipping audit logging until a regulator asks.
- One-off compliance per use case instead of platform-level.
- Forgetting deletion pipelines for vector stores.
- No vendor exit strategy until forced to leave.

## 26.23 Enterprise Best Practices

Build an internal AI platform. Establish an AI CoE. Implement governance lightly but consistently. Audit log everything. Multi-provider from day one. Compliance-as-code where possible. Train teams continuously. Review portfolio risk quarterly.
