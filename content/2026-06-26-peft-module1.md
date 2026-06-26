Title: PEFT, LoRA, and QLoRA
Date: 2026-01-01
Category: GenAI
Tags: GenAI, LLM, fine-tuning, transformers
Slug: peft-lora-qlora-guide
Status: published

Fine-tuning a 70B model used to mean renting a server farm. Not anymore.
PEFT, LoRA, and QLoRA have quietly made it possible to adapt massive language
models on a single consumer GPU — without sacrificing much quality. Here's
how they work and when to use each.

## What Is PEFT?

**PEFT (Parameter-Efficient Fine-Tuning)** — An umbrella term for techniques
that adapt a pretrained model by training only a *tiny* fraction of its
weights, leaving the rest frozen. Instead of updating billions of parameters,
you steer the base model with a small set of additions. Memory savings are
roughly 10–20×, and quality typically lands around 90–95% of full fine-tuning.

**Why it matters** — Full fine-tuning a 7B model requires storing gradients
and optimizer states for every weight — easily 80–100 GB of VRAM. PEFT
sidesteps this entirely. In 2026, it's the primary reason serious LLM
fine-tuning can happen on a single GPU.

**Key PEFT families:**
- **Adapter layers** — Small bottleneck modules inserted between transformer
  layers; only the adapters are trained.
- **Prompt tuning / prefix tuning** — Learnable tokens prepended to the input;
  the model itself stays frozen.
- **Low-rank adaptation (LoRA)** — The most widely adopted method today
  (see below).

## LoRA — Low-Rank Adaptation

**LoRA (Low-Rank Adaptation)** — Instead of updating a full weight matrix W
directly, LoRA injects two small trainable matrices A and B (where rank r ≪
original dimension) whose product BA approximates the weight update. The base
model is frozen; only A and B are trained.

**The math in plain English** — A typical weight matrix in a 7B model might
be 4096 × 4096 (16M values). With rank r = 16, LoRA replaces that update with
two matrices: 4096×16 and 16×4096 — just ~131K values. For a whole 7B model
this works out to ~0.24% of parameters being trained.

**Modules LoRA targets inside a Transformer:**

| Module | What it does | Why LoRA is applied here |
|---|---|---|
| `q_proj` | Projects input into Query space for attention | High impact on what the model attends to |
| `k_proj` | Projects input into Key space | Shapes which tokens are considered relevant |
| `v_proj` | Projects input into Value space | Controls what information flows forward |
| `o_proj` | Projects attention output back to hidden dim | Aggregates multi-head attention results |
| `gate_proj` / `up_proj` / `down_proj` | FFN layers (MLP block) | Fine-tunes knowledge stored in feed-forward weights |

> By default, most LoRA configs target `q_proj` and `v_proj`. Targeting all
> projection layers increases expressiveness but uses more memory.

**Key hyperparameters:**

- `r` (rank) — Controls adapter size. Typical values: 8, 16, 32, 64. Higher
  rank = more capacity but more memory.
- `lora_alpha` — Scaling factor. Effective learning rate for the adapter is
  `alpha / r`. Common pattern: set `alpha = 2 × r`.
- `lora_dropout` — Regularization applied to adapter activations (e.g. 0.05).
- `target_modules` — Which layers to apply LoRA to (see table above).

**Inference advantage** — After training, LoRA weights can be *merged* back
into the base model (`W + BA`), adding zero inference latency.


## QLoRA — Quantized LoRA

**QLoRA** — Takes LoRA and adds aggressive quantization of the *frozen* base
model weights to 4-bit NormalFloat (NF4), while keeping the trainable LoRA
adapters in full bfloat16 precision. The base model is temporarily
dequantized to bfloat16 only during the forward/backward pass computation.

**Three innovations QLoRA introduces:**

- **4-bit NormalFloat (NF4)** — A data type optimized for normally-distributed
  weights (which most LLM weights are). More accurate than standard INT4.
- **Double quantization** — Quantizes the quantization constants themselves,
  saving an additional ~0.5 bits per parameter.
- **Paged optimizers** — Uses NVIDIA unified memory to page optimizer states
  to CPU RAM when GPU memory spikes, preventing OOM crashes during training.

**What this unlocks** — A 65B parameter model can be fine-tuned on a single
48 GB GPU. A 7B model fits comfortably on a 6–8 GB consumer GPU.


## Choosing the Right Method

| Scenario | Best choice |
|---|---|
| Max quality, plenty of VRAM (≥40 GB) | LoRA on full precision model |
| Limited VRAM (6–24 GB), still want good results | QLoRA |
| Fastest inference after training | LoRA (merge weights) |
| Multiple task-specific adapters from one base | LoRA (swap adapter files) |
| Smallest number of trainable params possible | VeRA or QLoRA with low rank |

> LoRA performance on standard NLP benchmarks (e.g. GLUE) often comes within
> 0.3% of full fine-tuning, while training a fraction of the parameters.

## The Full Stack (Libraries You'll Actually Use)

- **`peft`** (Hugging Face) — `LoraConfig`, `get_peft_model`,
  `PeftModel.from_pretrained()`. The main entry point for all of this.
- **`bitsandbytes`** — Provides NF4 quantization and paged optimizers for
  QLoRA. Required for `load_in_4bit=True`.
- **`transformers`** — `BitsAndBytesConfig`, `AutoModelForCausalLM`,
  `TrainingArguments`, `Trainer`.
- **`trl`** — `SFTTrainer` wraps `Trainer` with supervised fine-tuning
  helpers; pairs naturally with `peft` + `bitsandbytes`.
- **`datasets`** — Loading and preprocessing your training data.
- **`accelerate`** — Multi-GPU/distributed training support under the hood.

## Wrapping Up

PEFT changed the economics of fine-tuning. LoRA made it memory-efficient;
QLoRA pushed it all the way to consumer hardware. The quality tradeoff is
usually minimal, the adapter files are tiny (a few hundred MB vs 14 GB for a
full model copy), and you can swap them at inference time. For most
task-specific fine-tuning work today, there's little reason not to start here.