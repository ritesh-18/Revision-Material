# Chapter 17 — MLOps for GenAI

## 17.1 Concept Explanation

MLOps is the discipline of operating machine learning systems reliably. For GenAI, "the model" is often a managed API or a frozen open-source artifact — but everything around it (prompts, retrieval indices, embeddings, fine-tunes, evaluations, deployments) still benefits from MLOps practice. The center of gravity shifts from "training to production" toward "prompt and pipeline to production."

Done well, MLOps lets a small team operate dozens of AI features safely. Done poorly, teams burn weeks debugging silent regressions and waiting on artifact handoffs.

## 17.2 The MLOps Layers

```
   Code (application + prompts + tool defs)
        |
   Data (training data, RAG corpora, eval sets)
        |
   Pipelines (ingest, embed, fine-tune, eval)
        |
   Experiments (tracked runs with metrics)
        |
   Model Registry (versioned artifacts and metadata)
        |
   Deployment (containers, autoscaling, traffic shifting)
        |
   Monitoring (quality, drift, cost, latency)
        |
   Feedback loop (user signals back into data)
```

Every layer has tools. The good news: you do not need all of them on day one. The bad news: skipping the foundational layers (versioning, evaluation, monitoring) costs more than building them later.

## 17.3 ML Pipelines

A pipeline is a directed graph of steps that transform data and produce artifacts. For GenAI, common pipelines include:

- **Ingestion.** Pull source documents, parse, chunk, enrich, embed, index.
- **Fine-tuning.** Collect data, prepare, train, evaluate, register, deploy.
- **Evaluation.** Run a suite of prompts and inputs against a model version, score, report.
- **Drift detection.** Sample production data, compare distributions to baseline, alert.
- **Re-embedding.** When the embedding model changes, re-embed the corpus incrementally.

Pipelines are scheduled (nightly evals), event-driven (new document arrives, embed it), or manual (one-time backfill).

## 17.4 Experiment Tracking

Experiment tracking records every training, evaluation, or major prompt-change run with its inputs, outputs, metrics, and artifacts. Without tracking, you cannot reproduce yesterday's improvements; with it, you can.

For GenAI, an "experiment" might be: prompt version + model + retrieval config + eval set + scores. Each run is captured so you can compare prompt v3 against v2 a month later.

Tools: MLflow, Weights & Biases, Comet, Neptune.ai, ClearML. Many AI-specific tools (LangSmith, Phoenix, Braintrust, Langfuse) overlap with this space for prompt and LLM workflows.

## 17.5 MLflow

MLflow is the most widely adopted open-source experiment tracking system. It logs runs, parameters, metrics, and artifacts; provides a UI for comparison; and includes a model registry for versioned artifacts.

Strengths: open source, framework-agnostic, broad adoption. Weaknesses: prompt/LLM-specific features less mature than dedicated tools.

## 17.6 Kubeflow

Kubeflow is Kubernetes-native ML platform. It bundles experiment tracking (Pipelines), model serving (KServe), notebooks (Notebooks), and hyperparameter tuning (Katib). Heavy but powerful; common in large enterprises.

For GenAI, Kubeflow Pipelines is sometimes used for fine-tuning and bulk inference jobs. KServe serves traditional ML models; for LLMs, vLLM or TGI are typically preferred.

## 17.7 Airflow

Apache Airflow is the dominant workflow orchestrator in data engineering. Pipelines are DAGs of tasks written in Python. Great for batch workloads — nightly evals, ingestion, embedding refresh.

Strengths: mature, broad community, massive ecosystem of operators. Weaknesses: scheduler is a bottleneck at very high scale, real-time workflows are awkward.

## 17.8 Prefect

Prefect is a newer workflow orchestrator with a cleaner Python API and better support for dynamic, parametrized flows. Strong for GenAI pipelines that branch based on data (e.g., re-embed only changed documents).

## 17.9 Dagster

Dagster is another modern orchestrator with strong asset-based mental model — pipelines are functions that produce data assets, with dependencies inferred from the graph. Great for data + ML teams who want lineage and observability built in.

## 17.10 Comparison of Orchestrators

| Tool | Best for | Weakness |
|---|---|---|
| Airflow | Mature batch ETL, broad ecosystem | Real-time, dynamic flows |
| Prefect | Modern Python flows, dynamic | Smaller ecosystem |
| Dagster | Asset-based ML/data | Steeper learning curve |
| Kubeflow Pipelines | Kubernetes-native ML | Operational complexity |
| Temporal | Durable workflows, agents | Less data-centric |

Pick one and stick to it. Mixing orchestrators across teams becomes its own problem.

## 17.11 Feature Stores

Feature stores manage features for ML models — point-in-time correct training data and low-latency online lookups. Tools: Feast, Tecton, Hopsworks, AWS SageMaker Feature Store.

For GenAI, "features" are less central, but feature-store-like concepts apply to:
- **User profiles** for personalization.
- **Cached embeddings** for low-latency retrieval.
- **Recent activity** to ground agent context.

You may not need a full feature store. You will likely build a "context store" — a system to fetch the right context per user per request.

## 17.12 Model Registry

A model registry is the canonical store for model artifacts: weights, configs, metadata, evaluation scores, ownership, lineage.

For GenAI:
- **Fine-tuned model artifacts** live in the registry.
- **Prompt versions** can live alongside or in a separate prompt registry.
- **LoRA adapters** are small artifacts with their own versioning.

MLflow Model Registry, SageMaker Model Registry, Vertex AI Model Registry, and Weights & Biases all play in this space.

## 17.13 Model Versioning

A model version is a triple of (architecture, weights, config). For GenAI, "the model" also implicitly includes the tokenizer.

Versioning practices:
- Every artifact has an immutable version ID.
- Promotions through environments (dev → staging → prod) record the version and the approver.
- Production references a version, not a tag like "latest."
- Rollback is a config change to a prior version, not a redeploy of code.

## 17.14 Prompt Versioning and Registries

Prompts are first-class artifacts in GenAI MLOps. A prompt registry stores prompt versions, their evaluation scores, owners, intended models, and deployment targets.

Tools: LangSmith, Langfuse, PromptLayer, Braintrust, plus custom solutions built on Git. Many teams start with prompts in code (versioned via Git) and graduate to a dedicated registry when the prompt count grows.

## 17.15 Eval as CI

The single highest-leverage MLOps practice in GenAI: every PR that touches a prompt, retrieval config, or model version runs the evaluation harness automatically. PRs that regress scores fail to merge unless explicitly approved with the regression noted.

Without this, prompt and model changes degrade silently. With it, regressions are caught at PR time.

## 17.16 Continuous Evaluation

Beyond CI evals, production should run continuous evaluations:
- Sample real production traffic.
- Score with LLM-as-judge or heuristics.
- Track scores by prompt version, model version, retrieval config.
- Alert on score drops.

This catches degradations from data drift, prompt drift, model provider changes, and tool changes.

## 17.17 Data Drift Detection

Data drift is when production input distributions diverge from training or evaluation distributions. For LLMs, drift manifests as:
- New user intents not represented in evals.
- New languages or styles.
- New document types in retrieval corpora.
- New tool inputs an agent has not seen.

Detect drift by comparing distributions (embeddings, length, language, intent classification) across time windows. Tools: Evidently, Arize, Fiddler, WhyLabs.

## 17.18 Feedback Loops

User signals are the most valuable training data you have.

- **Thumbs up/down** ratings.
- **Edit traces** (the user accepted an answer but edited a part).
- **Regenerations** (the user disliked the first response).
- **Session length and return rate.**
- **Explicit feedback comments.**

Feed these back into:
- Evaluation set updates.
- Fine-tuning datasets (RLHF, DPO).
- Prompt iteration priorities.
- Retrieval improvements (queries that produced poor answers).

## 17.19 Real-World Use Cases

A typical GenAI MLOps stack:
- Git for code and prompts.
- MLflow or LangSmith for experiment tracking.
- Airflow or Prefect for batch pipelines.
- Vector DB ingestion pipeline running nightly.
- Eval pipeline running on every PR and nightly on production.
- ArgoCD for deployment.
- Datadog/Grafana for observability.

## 17.20 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Full ML platform | Integration, ergonomics | Lock-in, complexity |
| Best-of-breed tools | Flexibility | Integration burden |
| Build internally | Tailored to needs | Engineering time |
| LLM-specific tools (LangSmith, Phoenix) | Native GenAI features | Newer, less mature |

## 17.21 Scaling Challenges

- Eval cost grows with eval set size and frequency. Budget it.
- Embedding refresh at large scale takes days; plan incremental.
- Model artifact storage grows; tier old artifacts.
- Pipeline orchestrator load grows; shard or scale up the orchestrator.

## 17.22 Security Concerns

- Eval sets contain real user queries — treat them as PII.
- Model artifacts in registries should be access-controlled.
- Prompts can contain secrets if poorly written.
- Pipeline credentials should use workload identity, not stored keys.

## 17.23 Cost Optimization

- Run heavy evals nightly, lightweight evals per-PR.
- Sample production traffic for continuous evaluation rather than scoring everything.
- Use cheaper judge models for first-pass eval; escalate to frontier models only on contested results.

## 17.24 Interview Questions

- What does MLOps look like when the model is an API?
- How do you version a prompt?
- Walk through a fine-tuning pipeline from data to production.
- How do you detect drift in an LLM application?
- Compare Airflow, Prefect, and Dagster.

## 17.25 Hands-on Exercises

1. Sketch the pipeline for nightly RAG eval on a 1,000-question set.
2. Design the prompt registry schema for an organization with 50 prompts across 10 services.
3. Plan a fine-tuning workflow: data collection, prep, train, eval, register, deploy.

## 17.26 Common Mistakes

- No eval set; flying blind.
- Manual rollouts of prompt or model changes.
- Skipping experiment tracking; cannot reproduce prior wins.
- Versioning code but not prompts.
- Ignoring drift until users complain.

## 17.27 Enterprise Best Practices

Eval-as-CI from day one. Standardize on one orchestrator and one experiment tracker. Maintain golden eval sets per product area. Make prompt and model versions immutable. Tie every production version to a documented owner. Quarterly drift reviews per critical feature.
