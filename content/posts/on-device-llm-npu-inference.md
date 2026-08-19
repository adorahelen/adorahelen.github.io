---
title: "On-device LLM Inference Is Prefill-Bound, Not Decode-Bound"
date: 2026-06-27T15:00:00+09:00
tags:
  - llm
  - systems
  - on-device
  - paper-review
summary: "The workloads people actually run on a phone have long inputs and short outputs, so prompt processing accounts for 88–99% of latency on CPU. Years of decode optimisation aimed at the wrong half. llm.npu (ASPLOS '25) restructures the prompt, tensors and blocks to make mobile NPU offloading work — 22.4× faster prefill, 30.7× less energy, and the first 1,000 tokens/sec prefill at the billion-parameter scale."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/on-device-llm-npu-inference/)**

## TL;DR

- **Measure prefill separately or you are optimising the wrong thing.** For long-input, short-output workloads, prompt processing is 88.3–98.8% of latency on CPU and 54.2–91.7% on GPU. Speculative decoding and friends address the remaining slice.
- **Mobile NPUs are fast at INT8 and hostile to LLMs.** Static shapes, an incompatible quantisation granularity, and terrible floating-point throughput (up to 159× slower than its INT8 path). Naive offloading makes things *2.5× worse*.
- **The fix is restructuring at three levels**: fixed-size prompt chunks with shared subgraphs, per-tensor INT8 on the NPU with the outlier channels computed as float on the CPU in parallel, and out-of-order subgraph execution to kill the resulting bubbles.
- **Outliers are sparse and biased.** 0.1–0.3% of channels cause the accuracy damage, which is why handling a tiny minority specially gets you both speed and accuracy.

> **Paper**: Fast On-device LLM Inference with NPUs · ASPLOS '25 · Daliang Xu et al. (Peking University, BUPT) · [arXiv:2407.05858](https://arxiv.org/abs/2407.05858) · [code](https://github.com/UbiquitousLearning/mllm)

---

## 1. The claim

The real bottleneck in on-device LLM inference is not token **generation** but prompt **processing**. Mobile CPUs and GPUs are weak at the large parallel matrix work prefill demands. A mobile **NPU** — an integer accelerator — is fast at exactly that, but it is fundamentally mismatched with how LLMs are built. `llm.npu` restructures the prompt and the model at **three levels — prompt, tensor, block** — and is the first system to make NPU offloading practical.

Results: prefill **22.4× faster on average with 30.7× less energy**, the first **1,000 tokens/sec prefill** at the billion-parameter scale, end-to-end response 1.4–32.8× faster, and under 1% accuracy loss.

---

## 2. Why prefill is the bottleneck

On-device LLM tasks — UI automation, email drafting, chat summarisation — have **long inputs and short outputs**. Personalisation and context awareness make prompts longer, not shorter.

| Task | Input length | Output length |
|---|---|---|
| UI automation (DroidTask) | 505–827 tokens | 3.5 tokens average |
| Email auto-reply (LongBench) | 1,168–1,835 tokens | 7.9 tokens average |
| Chat summarisation (Persona-Chat) | 488–584 tokens | 44 tokens average |

Prefill therefore accounts for **88.3–98.8% of total latency on CPU** and 54.2–91.7% on GPU. A large share of published on-device work optimises decode, which is the small half of the problem for these workloads.

**The opportunity** is the NPU. A current mobile NPU offers around 73 TOPS of INT8. Its INT8 matrix multiply is 4.5–5.8× faster than CPU INT8 and 1.8–3.5× faster than GPU FP16, at better energy efficiency. What did not exist was a system that ran an LLM on a commodity mobile NPU at all.

---

## 3. Three fundamental mismatches

| # | Problem | Concrete cost |
|---|---|---|
| **C1** | Variable-length prompts | NPUs support static shapes only, so every prompt length rebuilds the graph. **11.54 seconds of graph optimisation alone** for a 2B model |
| **C2** | Quantisation granularity mismatch | The per-group quantisation you want for accuracy is not natively supported, forcing sub-tensor splitting and float accumulation — **up to 10.7× overhead** |
| **C3** | Floating point cannot be eliminated | LayerNorm and attention need FP, and the NPU is bad at it — up to **159× slower than its INT8 path** |

C1 is the one that kills the naive approach: a static-shape accelerator meeting an inherently variable-length workload.

---

## 4. The three techniques

**Chunk-sharing graphs** (solves C1). Split the variable-length prompt into **fixed-size chunks** — legitimate because a decoder-only model processes causally. Then split operators in two: **static operators** (Linear, LayerNorm — depend only on chunk length, so the subgraph is **shared**) and **dynamic operators** (attention — depends on chunk position, so it stays per-chunk). On a 1.8B model, **120 of 144 subgraphs are shared, cutting memory by 75% (7.2 GB)**. Chunk length 256.

**Shadow outlier execution** (solves C2). Run only fast **per-tensor INT8** matrix multiply on the NPU, extract the **outlier channels that wreck accuracy**, and compute those as float on the CPU in parallel, then sum. Outliers are extremely sparse — **0.1–0.3% of channels** — so the CPU work hides inside NPU execution time. Two further optimisations: keep only the weights of the "hot channels" where outliers concentrate resident in memory and load the rest on demand (−34.3% memory), and **prune the outliers of the least important 85% of layers entirely**, which removes the CPU-NPU synchronisation for them.

**Out-of-order subgraph execution** (kills the bubbles). Cooperating FP units with an INT8 NPU produces **37% execution bubbles**. So ignore chunk order and **run whichever subgraph has its inputs ready**. Optimal ordering is NP-hard, so they use a **microsecond-scale online heuristic**: prefer the subgraph that reduces NPU stall the most. Bubbles go from **37% to 0.7%**.

---

## 5. Results

Implementation is about 10K lines of C/C++ and assembly on top of MLLM and QNN, on an open-ISA mobile NPU. Models: 1.8B through 7B. Devices: two recent flagship Android phones.

| Metric | Result (1024-token prompt) |
|---|---|
| Prefill speed | **18.2–38.4×** over llama.cpp on CPU; 32.5–43.6× over MLC on GPU; **3.28–5.32×** over the prior NPU system |
| Energy | **35.6–59.5×** less than llama.cpp on CPU |
| End-to-end latency | **1.4–32.8×** faster than baselines |
| Accuracy | **~1% average loss** against FP16 — +32.9% over SmoothQuant and +70.9% over K-Quant |
| Memory | 1.32× llama.cpp (the shadow outlier path is only 0.6–1% of it) |

The ablation is the part that justifies the paper's structure: **naive NPU offloading is 2.55–2.68× *slower*** than CPU. Chunk sharing brings 1.46–5.09×, shadow outliers 3.91–8.68×, and out-of-order execution a further 18–44%. All three are load-bearing; any one alone does not get there.

---

## 6. What I am keeping

1. **On-device LLM latency is prefill.** For long-input, short-output work, decode optimisation is close to irrelevant. Measure and optimise prefill separately.
2. **Pick the quantisation scheme together with the hardware.** Per-group is more accurate and costs 10.7× on this accelerator. Per-tensor plus separate outlier handling is what the hardware wants.
3. **Heterogeneous cooperation plus out-of-order scheduling** is what turns a fast accelerator into a fast system. The bubbles are where the wins hide.
4. **Outliers are sparse and biased** — 0.3% of channels produce over 80% of them. Special-casing a tiny minority buys you accuracy and speed at once.

---

## 7. What this changes for my own work

The paper is about phones, but the insight lands directly on server-side and desktop deployment.

### Security log analysis is a prefill-bound workload

Log analysis has **very long input** (logs plus context) and **short output** (a verdict or summary). That is precisely the regime the paper describes.

- **The lesson**: when choosing a model or a machine, benchmark **prompt processing speed, not decode tokens/sec**. For the same model, the time to ingest a long log dominates perceived latency.
- **What to do about it**: (1) reuse the KV cache for repeated system prompts and rule sets, which removes their prefill cost entirely; (2) use an engine with chunked prefill; (3) filter logs before the prompt rather than pasting them whole — **reducing the prefill token count is the single largest win available**, and it is architectural rather than a tuning knob.

### On unified-memory desktops

Apple Silicon has a neural engine, but the common local runtimes use the GPU and barely touch it — which says NPU offloading is still immature outside the mobile stack described here. In the short term, **unified memory bandwidth** (roughly 120 GB/s to 273 GB/s across current parts) governs decode speed. But for a prefill-bound workload, GPU prefill throughput plus enough RAM matters more than peak bandwidth. If the toolchain absorbs neural-engine and prefill optimisation later, the same hardware gets faster without a purchase.

### If work moves to the edge

For lightweight analysis pushed onto mobile or edge devices, this paper's stack is the direct reference. 1,000 tokens/sec prefill for a billion-parameter model on a phone is a real number, not a projection.

### Immediately applicable, regardless of project

| Action | Effect |
|---|---|
| Benchmark **prefill and decode separately** | Identifies the actual bottleneck |
| Adopt **prompt / KV caching** | Removes prefill cost for repeated system prompts |
| **Filter or summarise input** before the prompt | Fewer prefill tokens — the most direct latency reduction available |
| Consider **per-tensor** quantisation families | Accelerator-friendly and memory-efficient, accuracy permitting |

---

## 8. Assessment

| Aspect | Judgement |
|---|---|
| Venue | ASPLOS '25 — top tier for architecture and systems |
| Problem statement | Excellent. Reframes on-device LLM latency from decode to prefill with measurements, not assertion |
| Contribution | Three coordinated techniques, each independently validated by ablation |
| Practicality | Code released; the insights transfer to server-side prefill work |
| Limits | Tied to one vendor's NPU and ISA; models up to 7B; the outlier pruning of "unimportant" layers deserves more scrutiny at longer contexts |

> **In one line**: the accelerator was always fast enough. What was missing was a way to feed a variable-length, floating-point-dependent workload to a static-shape integer machine — and the answer was to reshape the workload, not the hardware.
