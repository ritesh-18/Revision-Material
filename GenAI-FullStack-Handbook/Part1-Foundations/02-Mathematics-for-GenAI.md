# Chapter 2 — Mathematics for GenAI

## 2.1 Why Math Matters Here

You don't need to derive backpropagation to ship GenAI products. You *do* need enough mathematical intuition to reason about why models behave as they do — why context length is quadratic in cost, why embeddings can be compared with cosine similarity, why temperature changes output diversity, why quantization works at all. This chapter builds operational intuition, not academic rigor.

If you only remember three sentences from this chapter, remember these. Models are matrices applied to vectors. Training is gradient descent shaping those matrices. Inference is the same matrices being applied to new vectors. Every system-level decision you make in GenAI — context length, batch size, memory layout, quantization — is a consequence of these three facts.

## 2.2 Linear Algebra — The Language of Models

A model is, at its core, a giant collection of matrices being multiplied by vectors. Every internal state is a vector; every learned parameter is part of a matrix. Understanding linear algebra is understanding the *shape* of a model's computation.

A **vector** is an ordered list of numbers. In GenAI, a token's embedding is a vector of typically 768, 1024, 4096, or higher dimensions. Each dimension is a learned feature — not human-interpretable, but mathematically meaningful. Two words with similar meanings end up with embeddings that point in similar directions. "Bank" the financial institution and "bank" the river edge end up in different regions of the space — contextual embeddings disambiguate them on the fly.

A **matrix** is a 2D grid of numbers. When a model "applies a linear layer," it multiplies an input vector by a weight matrix. The matrix's rows and columns determine how input dimensions are mixed and how many output dimensions emerge.

A **tensor** is the n-dimensional generalization. A batch of 32 sequences of 512 tokens with 4096-dim embeddings is a tensor of shape [32, 512, 4096]. Every bug in a deep learning pipeline can be traced — directly or indirectly — to a tensor shape mismatch.

Intuition by analogy: a vector is a point in a high-dimensional space. A matrix is a transformation that stretches, rotates, or projects that space. A neural network is a stack of such transformations interleaved with nonlinearities so that the composition can express complex functions.

## 2.3 Matrix Multiplication — The Hot Path

Matrix multiplication is *the* operation that dominates LLM cost. When a transformer processes a sequence, it performs trillions of multiply-and-add operations across many matrix multiplications. GPUs exist in their current form because matrix multiplication parallelizes beautifully — every output element is independent of every other output element.

Worked intuition: multiplying a 4096-dim vector by a 4096×4096 weight matrix produces a new 4096-dim vector. The cost is ~16.7 million multiply-add operations. For a 70-billion-parameter model, a single forward pass through one token involves doing this kind of operation dozens of times per layer across 80+ layers. Multiply by sequence length and batch size to see why we need hardware that can do thousands of multiplications in parallel per clock cycle.

This is why GPU memory bandwidth matters more than GPU clock speed for LLM inference. The math itself is simple. Moving the matrix weights from HBM (high-bandwidth memory) into the compute cores is the bottleneck.

## 2.4 Tensors and Shapes — The Debugging Currency

Engineers spend more time fighting tensor shapes than any other single thing. A familiar pattern: a model expects [batch, sequence, embedding] but the data loader produces [sequence, batch, embedding]. The error message mentions a matrix multiplication dimension mismatch ten layers deep, and the actual bug is in the data loader.

Conventions vary. PyTorch often uses [batch, sequence, hidden]. JAX and TensorFlow sometimes use [batch, hidden, sequence]. Attention internals sometimes reshape to [batch, heads, sequence, head_dim]. Knowing which convention each library uses is half the battle in production debugging.

## 2.5 Vectors, Norms, and Distance Metrics

For embeddings to be useful, we need a notion of "closeness." Two metrics dominate in practice:

**Euclidean distance** measures the straight-line distance between two points. It is sensitive to vector magnitude. If one embedding has larger numbers overall, it will appear distant from smaller ones even if they point in the same direction.

**Cosine similarity** measures the angle between two vectors, ignoring magnitude. It returns 1 for vectors pointing in identical directions, 0 for perpendicular, and −1 for opposite. This is why nearly every semantic search system uses cosine similarity: we care that "dog" and "puppy" point in similar conceptual directions, not that their embeddings happen to have the same length.

A practical detail: many embedding models output already-normalized vectors (unit length). For those, cosine similarity reduces to a simple dot product, which is faster and cheaper.

## 2.6 Eigenvalues, Eigenvectors, and PCA — Why You Will Eventually Care

You will not implement PCA by hand. But you will hit a wall when your vector index becomes too large to fit in RAM, and the standard fix is *dimensionality reduction*: projecting 1536-dim embeddings down to 256 dims while preserving most of the semantic structure. The math underneath that operation is eigendecomposition. The takeaway: high-dimensional embeddings often have a much lower *intrinsic* dimension, and you can exploit that for cost.

## 2.7 Probability — The Operating System of LLMs

LLMs do not output text. They output a probability distribution over the entire vocabulary for each next token. A 100,000-token vocabulary means the model emits 100,000 numbers, each between 0 and 1, summing to 1. The text you see is the result of *sampling* from that distribution.

This is why the same prompt produces different outputs. This is why temperature, top-k, and top-p exist — they control the shape of the distribution before sampling. Temperature 0 collapses the distribution to its single most likely token (deterministic, repetitive). High temperature flattens the distribution toward uniform (creative, unpredictable, more error-prone).

Understanding this collapses a hundred mysteries: why your structured output sometimes drifts, why your evaluation scores are noisy run-to-run, why a function-calling agent occasionally invents a tool name. The model is sampling from a distribution; rare events do happen.

## 2.8 Statistics — Evaluation in Disguise

Every claim you make about a model — "it is 5% better than the baseline" — is a statistical claim that requires sample size, confidence intervals, and significance testing. The most common production mistake is comparing two prompts on 10 examples each, declaring a winner, and shipping the change. With LLM noise, 10 examples is not enough to distinguish real improvements from sampling variation.

Practical rule of thumb: any A/B comparison on subjective output quality needs hundreds of examples, ideally graded by multiple independent judges, before you trust the verdict.

## 2.9 Calculus and Gradient Descent — How Models Learn

Training is, mechanically, a loop. The model makes a prediction, the loss function measures how wrong it was, and the *gradient* — the partial derivative of the loss with respect to every parameter — tells you which direction to nudge each parameter to reduce the loss. The update step takes a small step in that direction. Repeat billions of times across trillions of tokens, and the matrices stop being random and start encoding language.

You will not write a backward pass. But you will reason about *learning rates* (step size), *gradient clipping* (preventing explosive updates), and *optimizer state* (Adam stores two moments per parameter, which is why training memory is roughly 3× model size).

## 2.10 Softmax — The Function Every LLM Output Passes Through

Softmax turns a vector of arbitrary real numbers into a probability distribution. Each output element is the exponential of the input divided by the sum of exponentials. This converts the model's raw "logits" (uncalibrated scores) into a proper probability distribution that sums to 1.

Why exponentials? Because exponentials amplify differences. A logit that is twice as large does not become twice as probable — it becomes exponentially more probable. This is what makes LLMs decisive about confident predictions and uncertain about borderline ones.

Temperature is implemented as *dividing the logits by a temperature value before softmax*. Temperature < 1 sharpens the distribution (more decisive); temperature > 1 flattens it (more uniform). Temperature = 0 is a special case that becomes "argmax" — pick the single highest logit.

## 2.11 Embeddings Mathematics

An embedding is a learned mapping from a discrete token (or sentence, or image) to a continuous vector. The training objective is constructed so that semantically similar inputs end up with similar vectors. Common objectives include masked language modeling (BERT-style), contrastive learning (pull similar pairs together, push dissimilar pairs apart), and matryoshka representations (the first N dimensions are themselves a useful embedding).

Operational implications:
- Embedding spaces are *not interchangeable*. Embeddings from one model cannot be compared meaningfully to embeddings from another.
- The dimensionality is a design choice with cost implications. 1536-dim vectors store more nuance than 256-dim but cost 6× the storage and bandwidth.
- Embeddings drift if the model is retrained. Your vector index becomes worthless until reindexed.

## 2.12 Attention Mathematics — The Heart of the Transformer

Attention computes, for each token, a weighted combination of all other tokens' representations. The weights come from three projections of each token's embedding: query (Q), key (K), and value (V). For each token, its query is compared against every other token's key to produce attention scores; those scores are passed through softmax to become weights; those weights are applied to the value vectors and summed.

Conceptually: "for each word in my sentence, look at every other word, decide how much each matters to me right now, and mix their information into my own representation accordingly." Do this many times in parallel (multi-head attention) so different heads can capture different relationships (syntactic, semantic, coreference, positional).

Cost: comparing every token against every other token is *O(n²)* in sequence length. This is the quadratic wall mentioned in Chapter 1. Every long-context optimization is an attack on this cost.

## 2.13 Token Probability and Sampling Math

Beyond temperature, two more sampling controls dominate production:

**Top-k sampling** restricts the distribution to the k highest-probability tokens, renormalizes, and samples. This bounds how exotic the output can be while preserving diversity within the top k.

**Top-p (nucleus) sampling** restricts to the smallest set of tokens whose cumulative probability exceeds p (typically 0.9). When the model is confident, this set is tiny; when uncertain, it widens. This is adaptive in a way top-k is not.

These controls matter more than people realize. Many "hallucination" complaints disappear when temperature is lowered and top-p tightened.

## 2.14 Positional Encoding Math

Attention is permutation-invariant by default — it has no idea which token came first. Positional encoding injects the order. Two families dominate:

**Sinusoidal (absolute) encodings** add fixed sine and cosine signals at varying frequencies to the embeddings. Different positions get different patterns of waves, which the model learns to interpret.

**Rotary Position Embeddings (RoPE)** rotate the query and key vectors in a position-dependent way before the attention dot product. This is the standard in modern LLMs (Llama, Qwen, Mistral) because it generalizes better to longer contexts than the model was trained on.

The math matters operationally because *long-context inference depends on positional encoding choice*. Models with poorly chosen encodings degrade abruptly past training length; RoPE-based models degrade more gracefully and can be extended with techniques like NTK scaling and YaRN.

## 2.15 Production Architecture Implications

Every architectural decision in later chapters can be traced back to math in this chapter:
- KV cache exists because the Q-K-V projections are expensive and reusable for previously seen tokens.
- Quantization works because the matrices have enough redundancy that 4-bit precision preserves most of their information.
- Speculative decoding works because most next-token predictions are easy and a small model agrees with a large model on them.
- Embedding dimensionality reduction works because high-dim vectors have lower intrinsic dimension.

## 2.16 Common Mistakes

- Comparing embeddings from different models.
- Trusting evaluation results from tiny sample sizes.
- Treating temperature 0 as "deterministic" — it is, but the resulting outputs are often degenerate (repetitive loops).
- Confusing token count with character count when budgeting context windows.
- Assuming attention is linear in context length when planning latency budgets.

## 2.17 Interview Questions

- Why is attention quadratic, and how do production systems work around that?
- Explain temperature in one sentence using softmax.
- Why is cosine similarity preferred over Euclidean for embedding search?
- What does the optimizer state add to memory requirements during training?
- Why does RoPE generalize better to long contexts than sinusoidal encoding?

## 2.18 Hands-on Exercises

1. For a vocabulary of 100,000 tokens and a sequence of 2,048 tokens, estimate the size of the logits tensor for one forward pass.
2. If a model produces 1536-dim embeddings and you store 10 million documents, calculate raw vector storage in GB (float32) and after 8-bit quantization.
3. For a 70B model with FP16 weights, estimate model size on disk and required VRAM for inference at batch size 1.
4. Explain in one paragraph, without equations, what changes when temperature goes from 0.2 to 1.5.

## 2.19 Enterprise Best Practices

Standardize the embedding model across the organization or pay the cost of re-embedding every time a team picks a different one. Document the embedding model and version inside the vector index metadata so future engineers can detect drift. Cache attention computations (KV cache) whenever the conversation has a stable prefix. When evaluating prompt changes, use enough samples that statistical significance — not anecdote — drives the decision.
