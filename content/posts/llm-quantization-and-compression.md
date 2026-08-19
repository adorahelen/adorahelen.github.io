---
title: "LLM Quantisation, Sorted Out: Which Four-Bit, and Why the Suffix Matters"
date: 2026-06-30T11:00:00+09:00
tags:
  - llm
  - systems
  - quantization
  - on-device
summary: "There are only two ways to shrink a model — fewer bits per weight, or fewer weights — and quantisation wins on compression-per-unit-quality, so it is 90% of practical work. This is the map: PTQ versus QAT, weight-only versus weight-plus-activation, the number formats, the ecosystem formats that make the same '4-bit' mean different things, how to read a GGUF suffix, and why the KV cache is a separate problem from all of it."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/llm-quantization-and-compression/)**

## TL;DR

- **Two directions only**: fewer bits per weight, or fewer weights. Five techniques map onto those two, and quantisation has the best compression-to-quality ratio, so it is where you start and usually where you stop.
- **Activation outliers are the axis that separates the methods.** A handful of channels with extreme values wreck a 4-bit grid. AWQ protects the important 1% of channels, SmoothQuant migrates the difficulty into the weights, NVFP4 uses 16-element blocks with two-level scaling — all three are answers to the same problem.
- **"4-bit" is not one thing.** GGUF, GPTQ, AWQ and bitsandbytes are built for different runtimes with different algorithms. Picking the wrong one for your serving stack is the most common practical mistake.
- **The KV cache is a separate axis.** Quantising weights does not touch it, and at long context it can exceed the weights. Architecture (interleaved local attention) and runtime compression (2-bit KV) are complementary, not alternatives.

---

## 1. The big picture: compression has two directions

You can make a model smaller by using **fewer bits per weight** (lower precision) or by having **fewer weights** (fewer parameters). Five techniques map onto those two.

| Axis | What it reduces | How | Retraining | Note |
|---|---|---|---|---|
| **① Quantisation** | Bits per weight | 16-bit → 8/4/2-bit | Usually none (PTQ) | **Best quality per unit of compression** — start here |
| **② Pruning** | Number of weights | Remove less important weights, neurons, layers | Usually needed to recover | Structured or unstructured; costs perplexity |
| **③ Distillation** | Parameters wholesale | Small student imitates large teacher | Yes, train the student | The small variants of most open families are this |
| **④ Low-rank factorisation** | Parameters (matrices) | W ≈ A·B | Sometimes | LoRA for tuning, SVD for compression |
| **⑤ KV cache compression** | Inference memory | Compress the runtime KV tensors, not the weights | None | The real bottleneck at long context — §6 |

> **In one line**: 90% of practical compression is quantisation. Pruning, distillation and factorisation are what you layer on when quantisation is not enough, and KV cache compression is a different axis entirely — it shrinks *inference memory*, not the model. Hybrids are where the research is, but no single technique beats quantisation's compression-to-quality ratio.

---

## 2. Two ways to split quantisation

### When you cut: PTQ versus QAT

| | **PTQ** (post-training) | **QAT** (quantisation-aware training) |
|---|---|---|
| Timing | Quantise **after** training | Simulate quantisation **during** training |
| Cost | Cheap — a few hundred calibration samples | Expensive — needs the training pipeline |
| Quality | Good enough down to 4-bit | **Wins at extreme low bit**, e.g. 2-bit |
| In practice | **Almost every local model is PTQ** (GPTQ / AWQ / GGUF) | What frontier labs use to ship very low bit |

If you downloaded it, it is almost certainly PTQ. QAT requires access to the original training pipeline, which is why it is rare in redistributed models — and why a vendor shipping QAT quantisations (as Gemma 3 does) is worth noting.

### What you cut: weight-only versus weight-plus-activation

- **Weight-only** — weights at 4-bit, activations left at FP16. GPTQ, AWQ and GGUF live here. The goal is **memory**, and accuracy loss is small.
- **Weight-plus-activation** — activations quantised too (W8A8). Compute runs on INT8/FP8 tensor cores, so you get **speed** as well. SmoothQuant and FP8 live here, and the difficulty is activation outliers.

> **Activation outliers are the axis that separates the methods.** Extreme values in a few channels destroy a 4-bit grid. AWQ protects the important ~1% of channels; SmoothQuant shifts the difficulty into the weights; NVFP4 uses small blocks with two-level scaling. Same problem, three answers.

---

## 3. Number formats

| Format | Bits | Character | Where it is used |
|---|---|---|---|
| **INT8** | 8 | Linear integer. The safe default | Server W8A8, legacy |
| **INT4** | 4 | Integer 4-bit. Big compression, needs calibration | The basis of GPTQ and AWQ |
| **NF4** (NormalFloat4) | 4 | Information-theoretically optimal 4-bit grid for normally distributed weights | bitsandbytes, QLoRA |
| **FP8** (E4M3/E5M2) | 8 | Floating point 8-bit, wide dynamic range | Native on recent datacentre GPUs, vLLM serving |
| **MXFP4** | 4 | Microscaling FP4 — one shared scale per 32-element block | Recent GPUs, including training |
| **NVFP4** | 4 | Blocks of **16**, per-block E4M3 scale plus per-tensor FP32 — a second scaling level, more accurate than MXFP4 | 2–3× throughput over FP8, ~1.8× memory |

> **The 2025–26 trend** is integer INT4 giving way to **floating-point microscaling** (NVFP4, MXFP4). The idea is a separate scale per small block, which is inherently robust to outliers. NVFP4 pushes the block down to 16 and adds a second scaling level, aiming for FP4 compression at INT8 accuracy. It needs current-generation hardware, though — on a laptop or an older GPU, GGUF and AWQ are still the practical answer.

---

## 4. Ecosystem formats: the same "4-bit" is not the same file

This is the part that trips people up. The same nominal bit width splits by *who built it, for which runtime*.

| Format | Runtime | Method | Strength | Fits |
|---|---|---|---|---|
| **GGUF** | llama.cpp / ollama | k-quant and i-quant, block-wise scaling | **CPU+GPU hybrid**, laptops and Macs, many bit levels | Local and on-device ⭐ |
| **GPTQ** | HF / vLLM / exllama | Hessian-based per-weight sensitivity | Best **weight** accuracy at a given bit width; expensive to prepare | GPU serving |
| **AWQ** | vLLM / SGLang / TensorRT | Protects the important ~1% of channels using activation magnitude | Strong 4-bit quality, fast | Production serving ⭐ |
| **bitsandbytes** | HF Transformers | On-the-fly NF4 at load time (`load_in_4bit`) | Zero conversion step, **QLoRA fine-tuning** | Tuning and quick runs |

One published measurement on Llama-3.1-8B put retention at roughly AWQ 95% / GGUF 92% / GPTQ 90%. Absolute numbers move between evaluations; the ordering has been fairly stable.

**Choosing, in one line each:**

- Laptop, Mac, local → **GGUF Q4_K_M**
- vLLM or TGI in production → **AWQ**, or FP8 if the GPU supports it
- QLoRA fine-tuning → **bitsandbytes NF4**
- Python service where quality leads → AWQ, else GPTQ

---

## 5. Reading a GGUF suffix

The file name you see everywhere locally follows `Q{bits}_{family}_{size}`.

```
Q4_K_M
│ │ └ size:   S / M / L — quality-versus-size within the same bit width
│ └── family: K = k-quant (two-level super-block scaling); absent = legacy (_0 / _1)
└──── bits:   average bits per weight (Q2–Q8)
```

Three families:

| Family | Examples | Structure | When |
|---|---|---|---|
| **legacy** | Q4_0, Q4_1, Q8_0 | One scale per block (`_0` symmetric), plus offset (`_1` asymmetric). Simple affine | Only Q8_0 is still practical (near-lossless); the rest are superseded |
| **k-quant** ⭐ | Q3_K_M, Q4_K_M, Q5_K_M, Q6_K | Block scale plus super-block scale — piecewise affine | **The default recommendation**; beats legacy at ≤5 bits |
| **i-quant** | IQ2_XXS … IQ4_XS, IQ4_NL | Codebook based, more aggressive at low bit | When you need it smaller; depends heavily on an importance matrix |

Practical guidance, size against quality, for a 7B model:

| Quantisation | Size | Perceived quality | Note |
|---|---|---|---|
| Q8_0 | ~7.5 GB | Essentially the original | The safe choice |
| **Q5_K_M** | ~5 GB | Degradation hard to notice | Quality-first default |
| **Q4_K_M** ⭐ | ~4.5 GB | Original-class on most tasks | **The one most people download** |
| Q3_K_M | ~3.5 GB | Degradation becomes visible | When memory is tight |
| IQ2 / Q2_K | ~2.5 GB | Quality falls off | Last resort |

> **The importance matrix (imatrix)** records the mean squared activation at each weight position over calibration data — effectively, which weights fire often. Quantisation scales are then biased toward those weights, lowering perplexity at the same bit width. **The effect is largest at aggressive 2–4 bit quantisation**, so below Q4_K_M an imatrix build is close to mandatory.

---

## 6. KV cache compression: the real long-context bottleneck

Quantise the weights to 4-bit and **at long context the KV cache still eats more memory than the weights do**. This is a separate axis from model compression.

- **KIVI** (ICML 2024, [arXiv 2402.02750](https://arxiv.org/abs/2402.02750)) — tuning-free 2-bit KV quantisation, asymmetric by design: **keys per-channel, values per-token**, from the observation that channel outliers are large on the key side. Peak memory across weights plus KV drops **2.6×**, with 4× batch size and 2.35–3.47× throughput.
- Related work: KVTuner (per-layer mixed precision), ChunkKV / CompressKV (semantic token pruning), MiniKV (identifying which layers tolerate 2-bit).
- KV compression is not weight quantisation: **KIVI lowers precision, CompressKV-style methods reduce token count**. They are orthogonal, so you use both.

> The "5:1 local:global attention cuts KV sixfold" result in [my Gemma 3 notes](/posts/gemma3-technical-report/) is the **architectural** version of the same saving — complementary to the runtime compression here. Design it out, or compress it after the fact.

---

## 7. Runtime: where the compressed file actually goes

Sections 1–6 are about how small the file gets. This section is about **whether that file fits in fast memory, and how fast it then runs**. Compression only pays off if the whole thing lands somewhere fast.

### VRAM-resident versus offloading

| Approach | Behaviour | Speed | Note |
|---|---|---|---|
| **VRAM-resident** | Entire model in GPU memory | **Fastest** | What you want |
| **Offloading** | Overflow spills into system RAM | **Noticeably slower** | The GPU↔RAM link is the bottleneck. A compromise, not a plan |

Dropping one quantisation step to fit entirely in VRAM (Q5_K_M → Q4_K_M) is usually faster than running the larger quantisation with offloading. The compression you *choose* is less important than whether the result fits.

### Unified memory

Some machines share one memory pool between CPU and GPU, which removes the VRAM-versus-RAM distinction and makes offloading unnecessary — everything is GPU-accessible. The catch is bandwidth: unified memory is often slower than dedicated VRAM, which caps token generation. A 7B Q4 model loads comfortably in 16 GB of unified memory and still will not match a fast discrete GPU. Capacity yes, throughput no. (Bandwidth numbers and why decode is bound by them: [on-device NPU inference](/posts/on-device-llm-npu-inference/).)

### Mixture of experts: total parameters ≠ speed

Some models activate only a subset of experts per token.

- Example: a 26B model with roughly **4B active** (top-k of 128 experts).
- **Memory** is set by **total** parameters — all 26B has to be resident, about 15 GB at Q4.
- **Generation speed** behaves like the **active** parameter count, about 4B.
- So it is "26B of capacity at 4B of speed". Given the memory, MoE beats a dense model of equal active size comfortably.

> **In one line**: quantisation (§1–6) decides the file size; the runtime (§7) decides whether that file fits in fast memory and whether an MoE only computes its active slice. You need both to answer "is this fast on my machine".

---

## 8. Why inference is light and surgery is heavy

The memory gap in [where safety alignment lives](/posts/llm-safety-alignment-bypass/) §8 falls straight out of the above:

- **Inference (Q4)** is the k-quant compressed build from §5 — 7B is about 3.5 GB of weights, 7–8 GB to run.
- **Abliteration surgery (bf16)** needs the 16-bit original from §3. Extracting a refusal direction and orthogonalising on top of a 4-bit approximation breaks the grid, so **the surgery happens at bf16 and the result is re-quantised to Q4 afterwards**. Hence 16–18 GB for a 7B.

> Quantisation is enough for *viewing* a model at speed. It is not enough for *editing* one — which is why the same model's memory requirement doubles depending on what you are doing to it.

---

## Sources

- Arditi et al., "Refusal in Language Models Is Mediated by a Single Direction" (NeurIPS 2024) — the context for why abliteration needs bf16
- llama.cpp — [quantize README](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md) · [imatrix README](https://github.com/ggml-org/llama.cpp/blob/master/tools/imatrix/README.md)
- "Which Quantization Should I Use? A Unified Evaluation of llama.cpp Quantization on Llama-3.1-8B" — [arXiv 2601.14277](https://arxiv.org/abs/2601.14277)
- The Kaitchup, ["Choosing a GGUF Model: K-Quants, I-Quants, and Legacy Formats"](https://kaitchup.substack.com/p/choosing-a-gguf-model-k-quants-i)
- KIVI, "A Tuning-Free Asymmetric 2bit Quantization for KV Cache" (ICML 2024) — [arXiv 2402.02750](https://arxiv.org/abs/2402.02750)
- NVIDIA, "Quantization-Aware Distillation for NVFP4" (2026); MXFP4/NVFP4 microscaling documentation
