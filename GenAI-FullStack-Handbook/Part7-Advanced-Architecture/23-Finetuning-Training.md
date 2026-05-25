# Chapter 23 — Fine-tuning and Training

## 23.1 Concept Explanation

Training is how a model becomes a model. Pretraining produces a base model with general capability; post-training (instruction tuning, RLHF, DPO) shapes its behavior into an assistant; fine-tuning adapts it to a specific domain or task. Most GenAI engineers never pretrain. Most will fine-tune at some point. All should understand the process well enough to make informed buy-vs-train decisions.

The defining principle: fine-tuning teaches *behavior*, not *knowledge*. For knowledge, use retrieval. Confusing the two leads to expensive fine-tunes that fail to improve facts and disappointing eval results.

## 23.2 Pretraining — A Quick Tour

Pretraining is unsupervised next-token prediction on trillions of tokens of text. The model learns grammar, facts, reasoning patterns, and world knowledge by being asked to predict the next token millions of times. Cost: tens of millions to billions of dollars. Time: weeks to months on thousands of GPUs.

Only a handful of organizations pretrain frontier models. The output, however, is shared with the world (Llama, Mistral, Qwen, DeepSeek as open weights; GPT, Claude, Gemini as APIs). Everyone else builds on top.

The takeaway for engineers: choose a base model that fits your needs and build from there. Pretraining is not your problem.

## 23.3 Instruction Tuning (SFT)

Supervised Fine-Tuning teaches the base model to follow instructions in conversational style. The dataset is pairs of (instruction, ideal response). The model is trained to produce the ideal response given the instruction.

This is what turns a base model (good at autocomplete) into an assistant (good at following directions). All chat models you interact with have been instruction tuned.

For application engineers, SFT becomes relevant when:
- Output style is consistently wrong despite prompt engineering.
- A specific format must be enforced more reliably than prompting achieves.
- A domain vocabulary or jargon must be mastered.

## 23.4 RLHF — Reinforcement Learning from Human Feedback

After SFT, models are further refined with RLHF. Humans rank multiple model outputs by quality; a reward model is trained to predict these rankings; the base model is then trained via reinforcement learning to maximize the reward model.

RLHF makes models more helpful, more harmless, and more aligned with human preferences. It is also responsible for many of their quirks (over-cautious refusals, sycophancy).

RLHF is expensive — both in compute and in the human labor of producing rankings. Few teams below frontier-lab scale do full RLHF.

## 23.5 DPO — Direct Preference Optimization

DPO is a simpler alternative to RLHF. Instead of training a reward model and then RL-tuning against it, DPO directly optimizes the model on preference pairs (a preferred response and a rejected response).

DPO requires less infrastructure than RLHF and often matches its quality. It is the dominant preference-tuning method in open-source today.

For most teams that want to improve a model with preference data, DPO is the right starting point.

## 23.6 PEFT — Parameter Efficient Fine-Tuning

Full fine-tuning updates every model weight. For a 70B model, that requires tens to hundreds of GPUs.

PEFT techniques update only a small fraction of parameters, dramatically reducing compute and memory needs. The fine-tuned artifact is also tiny — megabytes instead of hundreds of gigabytes.

PEFT has democratized fine-tuning. A 70B fine-tune that would have required a cluster now fits on a single high-end consumer GPU.

## 23.7 LoRA — Low-Rank Adaptation

LoRA freezes the base model and inserts trainable low-rank "adapter" matrices alongside each weight matrix. Only the adapters are trained. At inference, the adapter's effect is added to the base weights.

Why "low-rank": instead of one big matrix update, two small matrices whose product is the same shape but with much fewer parameters. The math exploits the empirical observation that fine-tuning changes are usually low-rank.

LoRA adapters are typically 1–100 MB. You can keep one base model in memory and swap LoRA adapters per request — enabling per-tenant or per-task personalization at scale.

## 23.8 QLoRA — Quantized LoRA

QLoRA combines LoRA with 4-bit quantization of the frozen base model. The base model takes minimal memory; only the small LoRA adapters are trained in higher precision.

QLoRA enables fine-tuning 70B models on a single 24-48 GB GPU. The quality is excellent for many tasks.

For most teams doing supervised fine-tuning, QLoRA is the default starting point.

## 23.9 When to Fine-tune vs Prompt

| Goal | Better via |
|---|---|
| Reliable output format | Structured outputs first, then SFT |
| Domain vocabulary | Prompt + glossary first, then SFT |
| Knowledge of new facts | RAG, not fine-tuning |
| Reduced refusals on legitimate queries | SFT or DPO |
| Match a brand voice | SFT |
| Faster inference of specialized task | Fine-tune a smaller model (distillation) |
| Higher accuracy on hard reasoning | Reasoning models or chain-of-thought prompting |

The decision flow: prompt → RAG → SFT/PEFT → DPO → distill to small model. Stop when quality is good enough.

## 23.10 Training Data Quality

The single biggest determinant of fine-tune quality is data quality, not algorithm choice.

Principles:
- Smaller, higher-quality datasets beat larger, noisier ones.
- Diverse examples cover edge cases.
- Format consistency teaches format.
- Negative examples (what NOT to do) help in DPO.
- Human review of training data catches issues before training catches them.

Hundreds of high-quality examples often outperform thousands of mediocre ones.

## 23.11 Distillation

Distillation trains a small "student" model to mimic a large "teacher" model. The student learns from the teacher's output distribution, not just final labels.

Use cases:
- Run a frontier model's quality at a fraction of cost on a small task.
- Self-distillation to improve a model from its own reasoning traces.
- Specialize a small model for a single workflow.

Distillation is increasingly central to production AI: a small distilled model serving the hot path, with the frontier model used only for difficult edge cases.

## 23.12 DeepSpeed and FSDP

For distributed training across many GPUs:

- **DeepSpeed (Microsoft).** ZeRO optimizer state sharding, pipeline parallelism, large-model training.
- **FSDP (PyTorch Fully Sharded Data Parallel).** Native PyTorch sharding, often the modern default.
- **Megatron-LM (NVIDIA).** Tensor and pipeline parallelism for large language models.
- **Accelerate (Hugging Face).** Higher-level abstraction over many backends.

These libraries shard model state across GPUs and nodes, enabling training of models larger than any single GPU can hold.

## 23.13 Checkpointing

Long training runs must survive interruptions. Checkpoints save the model state, optimizer state, and training progress periodically so a crashed run can resume.

For GenAI fine-tuning:
- Checkpoint every N steps or every M hours.
- Keep multiple recent checkpoints; bugs can require rollback.
- Push checkpoints to durable storage (object store).
- Validate before promoting a checkpoint to "best."

## 23.14 Optimizer States

Optimizers like Adam track moving averages per parameter — two extra floats per parameter beyond the gradient. For a 70B model in mixed precision, this is roughly 280 GB of optimizer state, on top of 140 GB of weights and gradients.

This is why training requires so much more memory than inference, and why sharding (ZeRO, FSDP) is essential at scale.

## 23.15 Training Pipeline

```
   Data collection
        |
   Cleaning & filtering
        |
   Format conversion (chat templates, schemas)
        |
   Train/validation split
        |
   Training (SFT or DPO)
        |
   Evaluation on held-out set
        |
   Safety evals
        |
   Quantization / packaging
        |
   Model registry
        |
   Staged deployment with monitoring
```

Every step has tooling. Every step has failure modes. Document each and version each.

## 23.16 Real-World Use Cases

- **Customer support assistant** fine-tuned on past resolved tickets in the company's voice.
- **Legal document drafting** fine-tuned on accepted contracts and revisions.
- **Code completion** fine-tuned on the organization's codebase for internal style.
- **Translation** fine-tuned on domain-specific bilingual data.
- **Function calling reliability** improved by SFT on the company's actual tool schemas.

## 23.17 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| Full fine-tune | Maximum quality | Massive compute |
| LoRA / QLoRA | Cheap, swappable | Slight quality cap |
| SFT | Behavior change | Data labeling effort |
| DPO | Preference shaping | Need preference data |
| Distillation | Cheap inference | Quality cap of student |

## 23.18 Scaling Challenges

- Compute: even PEFT needs serious GPUs for large models.
- Data labeling: high-quality data is the bottleneck.
- Eval: knowing whether the fine-tune actually helped is the hardest part.
- Deployment: fine-tuned variants multiply the artifact zoo.

## 23.19 Security and Privacy

- Fine-tuning data can leak into model outputs. Audit for PII before training.
- Customer-specific fine-tunes raise data residency questions.
- Backdoors can be inserted via poisoned training data.
- Watermark or detect outputs from sensitive fine-tunes.

## 23.20 Cost

A QLoRA fine-tune of a 7B model on a few thousand examples: a few hours on a single A100 or H100, under $50 of GPU time.

A QLoRA fine-tune of 70B: tens of hours on a few GPUs, hundreds of dollars.

Full RLHF on a 70B from scratch: tens of thousands of dollars and significant engineering.

Distillation runs similarly — cost is dominated by inference on the teacher to produce data, then SFT on the student.

## 23.21 Interview Questions

- When do you fine-tune instead of prompt?
- Explain LoRA in one paragraph.
- Why is RAG usually better than fine-tuning for knowledge?
- Compare RLHF and DPO.
- Design a fine-tuning pipeline for a customer support bot.

## 23.22 Hands-on Exercises

1. Decide for three example tasks: prompt, RAG, fine-tune, or distill.
2. Estimate the GPU and time cost of QLoRA on a 13B model with 10k examples.
3. Plan an evaluation harness specifically for fine-tune regressions.

## 23.23 Common Mistakes

- Fine-tuning to "teach facts" — fails; use RAG.
- Tiny noisy datasets producing brittle behaviors.
- No held-out eval — can't tell if fine-tune helps.
- Forgetting that fine-tuning a base model is different from fine-tuning an instruction-tuned model.
- Skipping safety re-evaluation after fine-tuning; RLHF guardrails can erode.

## 23.24 Enterprise Best Practices

Treat fine-tuning as a measurable engineering project, not an experiment. Maintain training data version control. Evaluate on the same harness used for prompt iterations. Document every fine-tune's purpose, dataset, scores, and decommission date. Re-evaluate safety after every fine-tune.
