# Chapter 29 — Infrastructure as Code

## 29.1 Concept Explanation

Infrastructure as Code (IaC) is the practice of defining infrastructure (VMs, networks, databases, IAM, Kubernetes resources) in machine-readable configuration files, versioned in source control, applied by automated tools.

The shift IaC enables: from "click around the AWS console" to "review a PR that adds a database." Infrastructure becomes software — testable, reviewable, repeatable, auditable.

Three properties define mature IaC:
1. **Declarative.** Describe desired state; the tool reconciles.
2. **Idempotent.** Running the code repeatedly produces the same result.
3. **Versioned.** All changes through source control.

Teams that practice IaC ship infrastructure changes daily. Teams that don't take days to provision a new database.

## 29.2 Why IaC

Without IaC:
- Changes are manual and untracked.
- Environments drift (dev ≠ staging ≠ prod).
- Disasters are hard to recover from (no recipe).
- Onboarding is slow.
- Reviews are oral history.

With IaC:
- Changes are PRs.
- Environments are reproducible.
- DR is "apply the IaC in another region."
- Onboarding reads the code.
- Reviews are recorded.

The investment is real. The payoff is enormous at any non-trivial scale.

## 29.3 Terraform

The dominant IaC tool. HashiCorp's, now also OpenTofu (open-source fork after license change).

What Terraform does:
- Reads `.tf` files describing resources.
- Maintains state (current actual configuration).
- Computes a plan (changes to make).
- Applies the plan (calls cloud APIs).

Key concepts:
- **Provider.** Plugin for a cloud or service (AWS, GCP, Azure, Kubernetes, etc.).
- **Resource.** A thing to manage (EC2 instance, S3 bucket, etc.).
- **Data source.** Read-only reference to existing resources.
- **Variable.** Input.
- **Output.** Computed value passed downstream.
- **Module.** Reusable bundle of resources.
- **State.** File describing what Terraform manages.

## 29.4 Terraform State

State is the heart of Terraform. The state file tracks every managed resource and its current state.

**Local state** is for tutorials only. Production needs:

**Remote state** in S3 (or equivalent) with locking (DynamoDB for AWS). Multiple engineers can apply without corrupting state.

State has implications:
- **Secrets in state** — be careful what providers store.
- **State corruption** is painful to recover.
- **Importing existing resources** — `terraform import` to bring outside-managed resources under Terraform.

## 29.5 Terraform Modules

Modules are reusable units. Patterns:

- **Root module** — your main configuration.
- **Child modules** — encapsulated chunks (one for VPC, one for EKS, etc.).
- **Module registry** — public (Terraform Registry) or private (your org's).

Good modules have:
- Clear inputs and outputs.
- Documentation.
- Versioning.
- Examples.

Bad modules: too generic (useless), too specific (single-use), inconsistent.

## 29.6 Terraform Workspaces

Workspaces let one configuration manage multiple environments (dev, staging, prod). State is separated per workspace.

Tradeoffs:
- **Workspaces:** simple, but easy to mix up.
- **Separate directories per env:** explicit, more files.
- **Terragrunt:** wrapper that adds environment management features.

Most production teams use separate directories or Terragrunt over raw workspaces.

## 29.7 Terraform Tradeoffs

| Aspect | Pro | Con |
|---|---|---|
| Declarative | Predictable | Steep learning curve |
| State | Tracks reality | Corruption risk |
| HCL language | Readable | Limited control flow |
| Modules | Reuse | Versioning discipline |
| Multi-cloud | One tool | Provider quality varies |

## 29.8 OpenTofu

The community fork of Terraform created after HashiCorp's license change in 2023. Drop-in replacement for most use cases. Maintained by the Linux Foundation.

Some teams have switched; many wait to see how the ecosystem evolves.

## 29.9 Pulumi

IaC in real programming languages (TypeScript, Python, Go, C#, Java). Same model as Terraform (declarative, state-based) but with the expressiveness of code.

Pros: loops, abstractions, real tests, IDE support.
Cons: smaller ecosystem, more ways to write bad code, state still has same risks.

For teams that want IaC without HCL limitations, Pulumi is a strong choice.

## 29.10 AWS CloudFormation

AWS's native IaC. YAML or JSON. Tightly integrated with AWS. Less ergonomic than Terraform; AWS-only.

Use when:
- All-AWS organization.
- Want AWS-native tooling (stack sets, drift detection).
- CDK (the higher-level alternative) better matches your needs.

## 29.11 AWS CDK / CDK for Terraform

Cloud Development Kit. Define infrastructure in real code (TypeScript, Python, Java); CDK synthesizes CloudFormation or Terraform.

AWS CDK targets CloudFormation; CDKTF targets Terraform. Same idea: real-language IaC.

## 29.12 Crossplane

A different model: Kubernetes-native IaC. Infrastructure is declared as Kubernetes custom resources; Crossplane manages it.

Use when:
- Kubernetes-first organization.
- Want unified control plane for everything.
- Comfortable with Kubernetes as the management surface.

## 29.13 Ansible

Configuration management adjacent to IaC. Procedural rather than declarative. Often used alongside Terraform: Terraform creates the VM, Ansible configures it.

In container-native environments, less common. In legacy / mixed environments, still valuable.

## 29.14 IaC Patterns

**Layered.** Separate IaC repos/modules for networking, platform, applications. Each layer can change independently.

**Stack per environment.** Same code, different variables per dev/staging/prod.

**Module-driven.** Heavy use of reusable modules.

**Per-team ownership.** Teams own their app's IaC; platform team owns shared infra.

**GitOps for IaC.** PR merges trigger applies via Atlantis, Spacelift, env0, Terraform Cloud.

## 29.15 IaC Testing

Yes, infrastructure can be tested:

- **`terraform validate`** — syntax check.
- **`terraform plan`** — preview changes.
- **`tflint`** — lint Terraform.
- **`checkov`, `tfsec`** — security/policy scanning.
- **`terratest`** — integration tests in Go.
- **`Open Policy Agent`** — policy validation.

CI runs these before merge. Catches issues at PR time.

## 29.16 Drift Detection

Drift is when actual infrastructure differs from declared. Causes: manual changes, partial applies, external automation.

Detection:
- `terraform plan` shows differences.
- Tools like CloudFormation Drift Detection, Spacelift, Atlantis monitor.
- Schedule periodic checks.

Fix drift by either updating IaC to match reality or reapplying to reset.

## 29.17 Real-World Use Cases

- A startup uses Terraform from day one. Six months later, replicates to a second cloud for DR in a day. Without IaC, would have taken weeks.
- A team manually changed an SG. Next Terraform apply removed their change. They learned not to bypass IaC.
- A company moved 5,000 manually-created resources into Terraform over 9 months. Painful but transformative.

## 29.18 Production Architecture

```
   Git repo (Terraform code)
        |
   PR with proposed changes
        |
   CI runs:
   +-- terraform fmt / validate
   +-- tflint
   +-- tfsec / checkov
   +-- terraform plan (against real state)
        |
   PR review
        |
   Merge
        |
   Atlantis / Spacelift / Terraform Cloud applies
        |
   State updated
        |
   Drift detection runs hourly
        |
   Alerts on drift
```

## 29.19 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Terraform | Mature, multi-cloud | HCL learning curve |
| Pulumi | Real language | Smaller ecosystem |
| CloudFormation | Native AWS | AWS-only |
| Crossplane | K8s-native | Newer, complex |
| One mega-repo | Single source of truth | Coordination |
| Per-team repos | Team autonomy | Version drift |

## 29.20 Scaling Challenges

- Large state files slow `plan` and `apply`.
- Many teams contending for state lock.
- Cross-stack dependencies.
- Module versioning across consumers.
- Migration of existing infrastructure into IaC.

## 29.21 Security

- State files contain secrets — store encrypted.
- IAM for the IaC tool itself (Terraform Cloud, Atlantis) must be tightly scoped.
- Code review every IaC change.
- Use `*_sensitive` markings.
- Plan output may leak sensitive info; gate access.

## 29.22 Deployment Guide

Adopting IaC:
1. Pick a tool (Terraform unless you have a specific reason).
2. Configure remote state.
3. Start with a small piece (one network or service).
4. Build module library as patterns emerge.
5. Add CI checks.
6. Adopt a GitOps tool (Atlantis or Terraform Cloud).
7. Gradually import existing resources.
8. Eventually, all infrastructure changes go through IaC.

## 29.23 Monitoring Strategy

- Plan/apply success rate.
- Drift detected per week.
- Time to apply changes.
- Module version distribution.

## 29.24 Cost Optimization

- Tagging policies (cost allocation).
- Right-sizing in IaC.
- Quotas as code.
- Avoiding orphaned resources via lifecycle blocks.

## 29.25 Interview Questions

- *Compare Terraform and CloudFormation.*
- *What is Terraform state and why does it matter?*
- *How do you handle secrets in IaC?*
- *Walk through a Terraform PR workflow.*
- *What is drift and how do you handle it?*

## 29.26 Hands-on Exercises

1. Write a Terraform configuration for a VPC + subnets + ALB.
2. Set up remote state with S3 + DynamoDB locking.
3. Build a reusable module for a standard service deployment.
4. Add `tfsec` to CI.

## 29.27 Common Mistakes

- Local state in production.
- No state locking (multiple engineers corrupt state).
- Hardcoded values instead of variables.
- Single mega-state for everything (large, slow, scary).
- Bypassing IaC for "quick fixes" (drift).
- Modules without versioning.

## 29.28 Enterprise Best Practices

Terraform (or chosen tool) for all infrastructure. Remote state with locking. CI pipeline with validation, security scan, plan review. GitOps-style apply. Drift detection scheduled. Module library maintained. IaC training mandatory. Quarterly review of IaC patterns and tooling.
