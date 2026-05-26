# Chapter 38 — SRE Career Path

## 38.1 Concept Explanation

The SRE career path is wide. It spans from individual contributors solving immediate operational problems to staff engineers driving company-wide architecture to engineering directors building platforms. The skills compound over time.

Unlike some specializations, SRE offers genuine technical depth at every level. You can be a staff engineer SRE without becoming a manager. Some of the highest-impact engineers at any large company are senior SREs.

This chapter describes the path, what to focus on at each stage, and how to grow.

## 38.2 The Levels

A typical leveling:

**SRE / Junior SRE (L3-L4).** Owns runbooks, executes on-call, contributes to projects. Learning the systems and the practices.

**Senior SRE (L5).** Owns one or more services. Drives reliability for them. Mentors juniors. Participates in design.

**Staff SRE (L6).** Cross-team impact. Owns architecture for major systems or platforms. Influences company-wide direction.

**Principal SRE (L7).** Strategic technical direction. Solves the hardest problems. Multiplier for senior engineers.

**Distinguished / Fellow.** Industry-leading impact. Rare.

Manager track diverges at L5-L6 — some choose people management, others stay IC. Both paths reach equivalent senior levels.

## 38.3 What to Focus On — Years 0-2

- Master the daily tools: kubectl, terraform, your cloud, your observability stack.
- Read existing runbooks. Update one.
- Shadow on-call before going solo.
- Own a small project end-to-end.
- Write a postmortem.
- Build the habit of asking "why" until you understand.

The goal: become useful. Become an engineer your team trusts to be on-call alone.

## 38.4 Years 2-5

- Drive an incident from detection to postmortem to action item closure.
- Lead a small reliability project (improve an SLO, automate a tool).
- Mentor a new hire.
- Get fluent in distributed systems (Designing Data-Intensive Applications).
- Build internal tools that others use.
- Speak at a meetup or internal tech talk.
- Start specializing in something deep (Kubernetes internals, observability, GPU infra).

The goal: become a senior who scales beyond yourself. You should be able to handle any incident in your domain and produce platform-level wins.

## 38.5 Years 5-10

- Lead a multi-quarter initiative.
- Architect a major system.
- Mentor multiple engineers.
- Establish new practices in your org (eval pipelines, chaos engineering, FinOps).
- Publish technical writing (blog posts, talks).
- Develop point of view on the industry.

The goal: staff engineer. You should be the engineer others look to when the company faces a tough technical decision.

## 38.6 Years 10+

- Strategic technical direction.
- Cross-org influence.
- Industry presence (conferences, open-source contributions, books).
- Coach other staff engineers.

The career has many shapes here: principal IC, manager-of-managers, founder, consultant, instructor. SRE skills transfer to all.

## 38.7 The Skill Atlas

Skills to develop, roughly in order:

**Foundational (years 0-3)**
- Linux deeply.
- Networking fundamentals.
- One cloud (AWS, GCP, Azure) at the SAA level.
- Kubernetes.
- One scripting language (Python or Go).
- One observability stack.

**Intermediate (years 3-7)**
- Distributed systems theory.
- Database internals.
- Performance engineering.
- IaC mastery (Terraform).
- CI/CD design.
- Incident command.
- System design.

**Advanced (years 7+)**
- Multi-region architecture.
- Cost engineering at scale.
- Platform thinking.
- AI/ML systems.
- Security architecture.
- Compliance and governance.
- Technical leadership.

Each layer compounds on the prior.

## 38.8 IC vs Manager

At L5-L6, the path forks.

**IC path.** Continue technical depth. Architect bigger systems. Become irreplaceable for hard problems. Compensation scales with impact.

**Manager path.** Develop people. Build teams. Drive org strategy. Compensation scales with team scope.

Many engineers underestimate the IC path. At a large company, principal and distinguished engineers earn equivalent total comp to directors and VPs. Some engineers prefer continuing technical work; that path is fully supported.

The reverse is also true: some great engineers become great managers. Try it; switch back if it doesn't fit.

## 38.9 Specializations

SRE has sub-specialties:

- **Observability SRE.** Owns the metrics, logs, traces stack.
- **Platform SRE.** Builds the internal developer platform.
- **Network SRE.** Owns load balancing, DNS, ingress, egress.
- **Storage SRE.** Databases, caches, object stores.
- **Security SRE.** Intersection with infosec.
- **AI Infrastructure SRE.** GPUs, inference, training.
- **Edge SRE.** CDN, edge compute, mobile.

Specializing helps. Generalists are valued at startups; specialists at scale.

## 38.10 Working in Different Company Sizes

**Early startup (1-50 engineers).** You do everything. Wide responsibilities, less depth. Great for learning fast.

**Growth stage (50-500).** Specialization begins. Real platform teams form. SRE practices solidify.

**Late stage (500-5000).** Established teams. Deep specialization. More process. Compensation usually highest.

**Hyperscalers (5000+).** Very deep specialization. World-class systems. Long ramp-up.

Each stage suits different career moments.

## 38.11 Side Projects and Open Source

Building public artifacts compounds career value:

- **Blog posts.** Write about what you build.
- **Conference talks.** Local meetups → regional → KubeCon.
- **Open-source contributions.** PRs to tools you use.
- **Side projects.** Personal infrastructure work, freelance.

Visibility leads to opportunities. Internal-only contribution is harder to leverage.

## 38.12 Reading List

A few books to read:

- **"Site Reliability Engineering"** (Google) — the original.
- **"Site Reliability Workbook"** (Google) — the practical companion.
- **"Designing Data-Intensive Applications"** (Kleppmann) — required.
- **"Observability Engineering"** (Majors et al.) — modern observability.
- **"The Phoenix Project"** (Kim et al.) — DevOps culture novel.
- **"Accelerate"** (Forsgren, Humble, Kim) — DORA metrics and research.
- **"Software Engineering at Google"** — engineering culture.
- **"Chaos Engineering"** (Rosenthal, Jones) — the discipline.

Skim more; read these deeply.

## 38.13 Newsletters and Communities

- **Pragmatic Engineer** (Gergely Orosz).
- **SRE Weekly.**
- **DevOps Weekly.**
- **Last Week in AWS** (Corey Quinn).
- **Honeycomb's blog.**
- **High Scalability blog.**

Communities: SRE-specific subreddits, CNCF Slack, local DevOps meetups.

## 38.14 Conferences

Worth attending if possible:

- **SREcon** — the SRE conference.
- **KubeCon** — Kubernetes and cloud-native.
- **AWS re:Invent / Google Cloud Next / Microsoft Build.**
- **Velocity / DevOpsDays.**
- **Local meetups.**

Conferences also good for networking and finding next role.

## 38.15 Burnout

A reality of SRE. Drivers:
- Heavy on-call.
- Constant interruptions.
- Difficulty saying no.
- Hero culture.

Defenses:
- Hard boundaries on work hours.
- Healthy on-call rotation.
- Mentor others to share load.
- Vacation that is real vacation.
- Therapy if needed.

Career longevity matters more than any single sprint.

## 38.16 Career Pivots

SRE skills transfer to:

- **Software engineering.** Full breadth of code work.
- **Platform engineering.** Internal developer platforms.
- **DevOps consulting.** Helping companies adopt SRE.
- **Engineering management.** People leadership.
- **Founder / CTO.** Especially at infrastructure-heavy startups.
- **Security engineering.** Heavy SRE overlap.
- **Developer advocacy.** If you enjoy speaking and writing.

The skills compound; the role can shift.

## 38.17 The Long Game

The best SRE careers are built over decades, not years. Compounding effects:
- Networks. Engineers you've worked with become hiring managers.
- Reputation. Quality work follows.
- Skills. Each year of depth makes the next year of depth easier.
- Mentorship. Engineers you've mentored become senior themselves.

Optimize for long-term position, not short-term titles.

## 38.18 Hands-on Exercises

1. Self-assess against the skill atlas. List gaps.
2. Plan a quarter of focused growth on one skill.
3. Identify a senior engineer to learn from; reach out.

## 38.19 Common Mistakes

- Chasing titles over substance.
- Hopping companies for tiny raises.
- No public artifacts.
- Burning out by year 5.
- Avoiding the harder problems.
- No mentorship (either receiving or giving).

## 38.20 Final Thoughts

SRE is a remarkable career. It rewards curiosity, calm, and craft. The systems you build matter. The teams you join shape industries. The skills compound for decades.

If you found this handbook useful, you are already on the path. Keep going.
