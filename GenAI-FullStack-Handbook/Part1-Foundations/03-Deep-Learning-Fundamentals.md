# Chapter 3 — Deep Learning Fundamentals

## 3.1 Concept Explanation

Deep learning is the practice of building functions out of stacked layers of simple, differentiable operations whose parameters are learned from data. The "deep" refers to the number of layers stacked. The reason depth matters: each layer transforms its input into a slightly more abstract representation, and stacked layers compose those abstractions into something powerful enough to recognize a face, translate a sentence, or generate a paragraph.

A GenAI engineer does not train foundation models from scratch — that is a multi-million-dollar exercise. But you will fine-tune, you will quantize, you will run inference at scale, and you will debug models that misbehave. All of those require the mental model in this chapter.

## 3.2 Neural Networks — The Building Block

A neural network is a function. It takes an input vector, multiplies it by a learned weight matrix, adds a learned bias vector, and passes the result through a nonlinear activation function. That is one "layer." Stacking layers produces deeper functions. Stacking the right kinds of layers in the right pattern produces transformers, convolutional networks, diffusion models, and everything else.

Without nonlinear activation, stacking layers is mathematically equivalent to a single layer — you just get a bigger linear function. Nonlinearity is what gives depth its power. The activation function bends the flat line into a curve, and stacking many bent curves can approximate any function. This is the universal approximation theorem in plain English.

## 3.3 Forward Propagation

Forward propagation is the act of running the network from input to output. Input vector enters layer 1, gets transformed, exits as a new vector, enters layer 2, and so on until the final layer produces the output. In an LLM, this output is the logits vector — one number per token in the vocabulary. Forward propagation at inference time is what your production system spends 99% of its compute on.

There is no learning in forward propagation. The weights are fixed; the computation is deterministic given the input and the sampling seed. Forward pass per token is what dictates inference latency.

## 3.4 Backpropagation — How Learning Happens

Backpropagation computes the gradient of the loss with respect to every parameter in the network. It does this by applying the chain rule of calculus, working backward from the output layer to the input layer. Each layer receives the gradient flowing in from above, uses it to compute the gradient with respect to its own parameters, and passes the resulting gradient downward.

This is *exclusive to training*. Production inference does not do backpropagation. But knowing it exists explains why training is roughly 3× the memory of inference: you must store intermediate activations from the forward pass so backprop can use them. This is also why "gradient checkpointing" is a common training trick — recompute some activations rather than store them, trading compute for memory.

## 3.5 Loss Functions

The loss function is a scalar that measures how wrong the model is. Training minimizes this scalar. The choice of loss depends on the task.

**Cross-entropy loss** dominates language modeling. For each predicted token, it asks: how surprised was the model by the correct next token? If the model assigned 90% probability to the correct token, the loss is small. If it assigned 0.01%, the loss is huge.

**Mean squared error** dominates regression. **Triplet/contrastive losses** dominate embedding training. **KL divergence** dominates distillation, where a small model is trained to match the output distribution of a large one.

You do not implement these. But you should recognize them in papers and config files, because they tell you what behavior the model is being shaped toward.

## 3.6 Optimizers

The optimizer is the algorithm that converts gradients into parameter updates. The naïve version is SGD: subtract a small multiple of the gradient. Modern optimizers are far more sophisticated.

**Adam** and **AdamW** track two running averages per parameter — the mean and variance of recent gradients — and use these to adapt the per-parameter step size. This converges much faster than SGD on transformer architectures.

**Adafactor** approximates Adam with less memory, useful at scale.

**Lion** is a newer optimizer that uses only one momentum value and produces competitive results with lower memory.

Production implication: Adam doubles your optimizer state memory (two moments per parameter). For a 7B model in mixed precision, that is roughly 14 GB of weights, 14 GB of gradients, 28 GB of optimizer state, plus activations — easily 80 GB for training versus 14 GB for inference. This is why training requires bigger GPUs than inference.

## 3.7 Activation Functions

The nonlinearity injected between layers. The historical sequence:

- **Sigmoid** squashes any input to (0, 1). Used in older networks; suffers from vanishing gradients at the extremes.
- **Tanh** squashes to (−1, 1). Similar issues to sigmoid.
- **ReLU** outputs max(0, x). Solves vanishing gradients but introduces "dead neurons" (outputs stuck at 0).
- **GELU** is a smoother approximation of ReLU, standard in transformers.
- **SwiGLU** is a gated variant used in many modern LLMs (Llama, PaLM). It splits the layer into two halves, multiplies them, and provides better gradient flow.

You will not pick the activation function in production — the model architecture has already picked. But knowing which activation a model uses is part of understanding why it has the inference characteristics it does.

## 3.8 CNN — Convolutional Networks

CNNs dominated vision before transformers and are still the most efficient architecture for many image tasks. The core idea is *weight sharing*: instead of learning a separate weight for every pixel position, the network learns small filters that slide over the image. This drastically reduces parameter count and exploits the fact that visual features (edges, textures) look the same regardless of where they appear in the image.

For GenAI engineers, CNNs are relevant in two places: image embedding models (CLIP-style encoders), and the earliest stages of multimodal models that ingest images before passing tokens to a transformer.

## 3.9 RNN and LSTM — Predecessors of the Transformer

RNNs process sequences one element at a time, maintaining a hidden state that carries information across time. LSTMs and GRUs are gated variants that mitigate the vanishing gradient problem.

These were the dominant sequence architecture until 2017. They are nearly extinct in new LLM work because they cannot be parallelized over time during training. They survive in two niches: extremely small on-device sequence models, and recent "linear attention" or "state-space" models (like Mamba) that revive RNN-like recurrence with modern tricks to scale better than transformers on very long sequences.

## 3.10 Transformers — The Architecture That Won

A transformer block is, conceptually, two halves stacked many times:

1. **Self-attention.** Each token looks at all other tokens via Q-K-V projections and attention weights.
2. **Feed-forward network.** Each token is independently transformed through a much larger hidden layer (typically 4× the embedding dimension) and projected back.

Layer normalization and residual connections wrap each half. Residual connections allow gradients to flow easily through deep stacks; without them, training a 80-layer model would be nearly impossible.

```
   Input embeddings + positional encoding
                |
        +-------v--------+
        |   Self-attn    |  <-- mixes information across tokens
        +-------+--------+
                |
            residual + norm
                |
        +-------v--------+
        | Feed-forward   |  <-- transforms each token independently
        +-------+--------+
                |
            residual + norm
                |
        (repeated N times — 32 for Llama 7B, 80 for GPT-3)
                |
        Output projection -> logits
```

Decoder-only transformers (GPT, Llama) use *causal* attention, where each token can only attend to itself and earlier tokens. This is what makes autoregressive generation possible — the model can predict the next token without having seen future tokens during training.

## 3.11 GPU Acceleration

Deep learning runs on GPUs because the computation is overwhelmingly matrix multiplication, and GPUs are matrix-multiplication machines. A modern training-grade GPU performs thousands of multiplications per clock cycle across thousands of cores. CPUs perform a handful per core across dozens of cores. The throughput gap is one to two orders of magnitude.

The catch: GPU compute is wasted unless the GPU's memory bandwidth keeps up. Modern GPUs use HBM (high-bandwidth memory) precisely because feeding the compute cores is the constraint. Chapter 14 covers the hardware in depth.

## 3.12 CUDA — The Lingua Franca of AI Hardware

CUDA is NVIDIA's parallel computing platform. Every major deep learning framework — PyTorch, JAX, TensorFlow — compiles to CUDA kernels for execution on NVIDIA GPUs. AMD has ROCm as a competing stack; Intel has oneAPI. Apple has Metal Performance Shaders. NVIDIA's lead in software ecosystem is as important as its lead in hardware.

You will rarely write CUDA. You will frequently debug installations involving CUDA, cuDNN (NVIDIA's deep learning primitives), NCCL (multi-GPU communication), and the driver version mismatches that produce mysterious crashes. Knowing this stack exists demystifies the error messages.

## 3.13 Mixed Precision Training

Modern training uses a mix of floating-point precisions: 16-bit (FP16 or BF16) for most operations to halve memory and double throughput, and 32-bit for accumulators and weight updates to preserve numerical stability.

BF16 (brain float 16) is the standard for LLM training because it has the same exponent range as FP32 — it loses precision but not magnitude. FP16 has narrower range and can underflow during loss computation, requiring "loss scaling" tricks.

For inference, even more aggressive precisions are used: FP8 (newest hardware only), INT8, and INT4 via quantization. This trades model quality for memory and speed, and is the foundation of running large models on consumer hardware.

## 3.14 Real-World Use Cases of These Fundamentals

Every production LLM system you will touch is the product of these primitives composed at scale. Embedding models are transformer encoders trained with contrastive loss. Chat models are decoder-only transformers post-trained with RLHF. Vision-language models stack image encoders onto LLMs. Diffusion image models are U-Nets — a CNN variant — trained to remove noise step by step. None of this is exotic once you see the parts.

## 3.15 Common Mistakes

- Confusing "training" and "inference" when discussing memory requirements.
- Believing that more parameters always means better — a well-trained 7B can outperform a poorly trained 70B.
- Treating quantization as free — there is always some quality loss, even if small.
- Assuming PyTorch and TensorFlow models can be swapped interchangeably; weights, layer naming, and conventions differ.
- Ignoring the dataloader as a performance bottleneck. GPUs sit idle waiting for data more often than people admit.

## 3.16 Interview Questions

- Why do we need activation functions?
- What does mixed precision training save, and what does it risk?
- Why do residual connections matter in deep networks?
- Estimate the memory required to train a 7B model with Adam.
- Explain the difference between encoder-only, decoder-only, and encoder-decoder transformers.

## 3.17 Hands-on Exercises

1. Sketch the data flow through a single transformer block. Label every tensor shape.
2. For a 13B model in BF16, calculate model weight size in GB. Now add Adam optimizer state. Now add gradients. Now estimate total training memory at batch size 4 with 4096-token sequences.
3. Explain why a GPU might be 95% memory-bound during inference and 95% compute-bound during training.

## 3.18 Enterprise Best Practices

Pin framework, CUDA, driver, and cuDNN versions together — version drift is the single biggest source of phantom production failures. Profile data loaders before profiling models; idle GPUs cost more than slow models. Standardize on BF16 for training and reserve FP16 for inference where appropriate. Build internal benchmarks that exercise the full inference path, not just the model forward pass.
