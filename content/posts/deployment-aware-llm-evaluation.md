---
title: "The Leaderboard Winner Is Not the Deployment Winner: Evaluating Open Reasoning Models on Three Axes"
date: 2026-06-27T13:00:00+09:00
tags:
  - llm
  - evaluation
  - paper-review
  - on-prem
summary: "A fully crossed evaluation — 7 models × 4 benchmarks × 3 prompting strategies, the same 238 examples everywhere — scores accuracy alongside latency and VRAM. The top-ranked configuration wins by 4.2% while costing 3.2× the memory and 2× the latency. If you deploy on your own hardware, that trade is almost never worth taking, and the paper has the Pareto frontier to prove it."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/deployment-aware-llm-evaluation/)**

## TL;DR

- **Rank one is not deployment one.** Gemma-4-26B-A4B takes the top weighted score at 0.794. Gemma-4-E4B scores 0.761 using **3.2× less VRAM** and running **2× faster**. You are buying 4.2% of score with 33 GB of memory.
- **Prompting reorders the field; it does not lift it uniformly.** Zero-shot versus few-shot CoT rank correlation is only ρ = 0.679. Qwen3-8B moves three places depending on the prompt — it is competitive under few-shot CoT and mid-pack everywhere else. Compare models with the prompting strategy pinned, or you are comparing noise.
- **Interface robustness is a first-class deployment property.** Phi-4-Reasoning scores 1.000 on TruthfulQA and 0.008 on GSM8K — not because it cannot do arithmetic, but because its `<think>` trace format defeats the harness's answer extractor on 98% of items. Capability you cannot parse is capability you do not have.
- **Task routing has real headroom.** No model dominates all four benchmarks. An oracle router reaches 0.825 against 0.794 for the best single configuration — +3.9% available to anyone who can classify the task.

> **Source**: [arXiv:2604.07035v2](https://arxiv.org/abs/2604.07035) · Md Motaleb Hossen Manik, Ge Wang · 84 conditions, 19,992 examples

---

## 1. The claim

Open reasoning models get compared badly. Different papers evaluate on different sample sizes, with unstandardised prompts, and rank on accuracy alone. The result is a leaderboard that answers a question nobody deploying a model actually asks.

This paper runs the fully crossed design instead: **7 models × 4 benchmarks × 3 prompting strategies = 84 conditions**, the same 238 examples in every cell, and records latency and VRAM alongside accuracy. The framing that comes out of it is that model selection is multi-objective optimisation, and a single scalar score cannot express the answer.

## 2. The grid

**Models.** Seven, split between mixture-of-experts and dense:

| Model | Total params | Active params | Type |
|---|---|---|---|
| Gemma-4-26B-A4B | 26.0B | 3.8B | MoE |
| Gemma-4-E4B | 8.0B | 4.0B | MoE |
| Gemma-4-E2B | 5.0B | 2.0B | MoE |
| Qwen3-30B-A3B | 30.0B | 3.0B | MoE |
| Qwen3-8B | 8.0B | 8.0B | Dense |
| Phi-4-Reasoning | 14.0B | 14.0B | Dense |
| Phi-4-Mini-Reasoning | 3.8B | 3.8B | Dense |

**Benchmarks**, with the weights used for the composite score:

| Benchmark | Domain | Format | Weight |
|---|---|---|---|
| GSM8K | Grade-school maths | Free response | 0.40 |
| MATH L1–L3 | Competition maths | Free response | 0.30 |
| ARC-Challenge | Science QA | Multiple choice | 0.20 |
| TruthfulQA MC1 | Truthfulness | Multiple choice | 0.10 |

**Prompting**: zero-shot (`{question}\n\nAnswer:`), CoT (`Let's think step by step.`), and few-shot CoT with three per-dataset exemplars.

The weights are the paper's choice and they are arbitrary — a point I will come back to. Their virtue is that they are *stated*, so you can re-derive the ranking under your own weighting.

## 3. Rank one versus the Pareto frontier

The top configurations:

| Rank | Model | Strategy | Weighted | Latency (s) | VRAM (GB) |
|---|---|---|---|---|---|
| 1 | Gemma-4-26B-A4B | zero-shot | **0.794** | 7.28 | 48.07 |
| 2 | Gemma-4-E4B | few-shot CoT | 0.761 | 3.68 | 14.90 |
| 3 | Gemma-4-E4B | CoT | 0.759 | 4.72 | 14.90 |
| 4 | Gemma-4-E4B | zero-shot | 0.758 | 4.37 | 14.90 |
| 5 | Gemma-4-26B-A4B | CoT | 0.756 | 8.72 | 48.07 |
| 6 | Qwen3-8B | few-shot CoT | 0.722 | 7.03 | 15.26 |

Gemma 4 takes all five top slots. More usefully, Gemma-4-E4B lands between 0.758 and 0.761 across all three prompting strategies — a one-model spread of 0.003, which is the sort of stability that makes a model predictable in production.

Now put a memory budget on it:

| Budget | Best configuration | Weighted | VRAM | Latency |
|---|---|---|---|---|
| ≤ 16 GB | Gemma-4-E4B (few-shot CoT) | 0.761 | 14.9 GB | 3.68 s |
| Unbounded | Gemma-4-26B-A4B (zero-shot) | 0.794 | 48.1 GB | 7.28 s |

**+4.2% weighted score for 3.2× VRAM and 2× latency.** On a single consumer GPU that trade is not available at any price — 48 GB is a different class of machine. Even where it is available, it is a poor buy for most workloads.

Ranking by efficiency instead — weighted accuracy divided by (latency × VRAM) — puts Gemma-4-E2B first, Gemma-4-E4B second as the practical optimum, and the 26B flagship third, reserved for accuracy-at-any-cost.

## 4. Prompting reorders the field

The interesting negative result is that prompting is not a uniform lift applied to every model.

| Comparison | Spearman ρ | Kendall τ |
|---|---|---|
| CoT vs zero-shot | 0.964 | 0.905 |
| CoT vs few-shot CoT | 0.750 | 0.619 |
| Few-shot CoT vs zero-shot | 0.679 | 0.524 |

At ρ = 0.679 the ordering is substantially different. Per-model:

| Model | CoT | Few-shot CoT | Zero-shot | Range |
|---|---|---|---|---|
| Gemma-4-E4B | 1 | 1 | 2 | 1 |
| Qwen3-30B-A3B | 4 | 4 | 4 | 0 |
| **Qwen3-8B** | 5 | **2** | 5 | **3** |

Qwen3-8B is a mid-pack model that becomes a contender when, and only when, you give it exemplars. Two consequences: a comparison that varies the prompt across models is not measuring the models, and a deployment that changes its prompt template has changed its model's effective rank.

## 5. Nobody wins everything

| Benchmark | Best model | Strategy | Accuracy |
|---|---|---|---|
| ARC-Challenge | Gemma-4-26B-A4B | zero-shot | 0.945 |
| GSM8K | Qwen3-8B | few-shot CoT | 0.819 |
| MATH L1–L3 | Gemma-4-E4B | few-shot CoT | 0.693 |
| TruthfulQA MC1 | Phi-4-Reasoning | few-shot CoT | 1.000 |

Four benchmarks, four different winners. That is what creates the routing headroom: an oracle that picks the right model per task reaches **0.825** against 0.794 for the single best configuration. The oracle is unattainable — it knows the answer in advance — but it bounds what a real classifier could recover, and +3.9% is more than the gap between the top two models.

## 6. The Phi-4 failure is not a capability failure

This is the part of the paper I would keep even if the rest evaporated.

Phi-4-Reasoning scores 1.000 on TruthfulQA MC1 and 0.008–0.042 on GSM8K. Not weak — broken:

| Strategy | GSM8K accuracy | Missing predictions | Think-tag rate |
|---|---|---|---|
| CoT | 0.008 | **98.3%** | 100% |
| Few-shot CoT | 0.021 | **97.5%** | 99.2% |
| Zero-shot | 0.042 | **95.8%** | 97.5% |

The model emits long internal `<think>` traces and formats its final answer in a way the unified harness's extractor does not recognise. Nearly every response is scored as *missing*, not as *wrong*.

Two things follow. First, a benchmark table can report near-zero for a model that is functioning — read the missing-prediction column before concluding anything about capability. Second, and this is the deployment lesson: **output-format compatibility is a selection criterion with the same standing as accuracy**. A model your pipeline cannot parse is unusable in that pipeline regardless of what it knows. Test extraction against real generations before you commit to a model, not after.

## 7. Latency and memory profile

| Model | VRAM (GB) | Mean latency (s) | Class |
|---|---|---|---|
| Phi-4-Mini-Reasoning | 7.1 | 4.6–6.7 | Ultra-light |
| Gemma-4-E2B | 9.7 | 4.4–6.0 | Light |
| **Gemma-4-E4B** | **14.9** | **3.7–4.7** | **Practical optimum** |
| Qwen3-8B | 15.3 | 7.0–7.7 | Mid |
| Gemma-4-26B-A4B | 48.1 | 7.3–10.6 | Heavy |
| Qwen3-30B-A3B | 57.6 | 14.7–15.3 | Heaviest |

Gemma-4-E4B is the fastest model in the set at 3.7 s while fitting in 14.9 GB — one GPU, no sharding. Set against Qwen3-8B at 15.3 GB and 7.0 s: same memory class, roughly half the speed, lower score. Eight billion MoE parameters with four billion active beats eight billion dense on all three axes at once, which is the clearest efficiency argument for MoE in the paper.

## 8. What I am keeping

1. **Record all three numbers or the comparison is not a comparison.** Accuracy, latency, VRAM — logged together, every run. A Pareto plot answers "which model" far better than a sorted score column.
2. **Pin the prompting strategy before comparing models.** Prompting changes rank order, not just level. Varying both at once measures neither.
3. **Test output-format compatibility before integration.** The Phi-4 result is what happens when you skip this: a working model scoring 0.008.
4. **Treat routing as available headroom, not exotica.** Different tasks have different winners; a cheap classifier in front of two models can beat one better model.
5. **Gemma-4-E4B is the default worth beating for single-GPU work.** 14.9 GB, 3.7 s, stable across prompting strategies. Anything you pick instead should have to justify itself against those three numbers.

## 9. Assessment

| Aspect | Judgement |
|---|---|
| Source | arXiv preprint, two independent researchers — not yet peer reviewed |
| Problem statement | Clear and practical; "leaderboard ≠ deployment optimum" is supported rather than asserted |
| Contribution | Fully crossed 84-condition design, Pareto analysis, prompt-sensitivity quantification, compatibility diagnostics |
| Practicality | High — the selection framework transfers directly to on-premise work |
| Limits | Four benchmarks, 238 examples per condition, arbitrary composite weights, single H100 environment, model set skewed to Gemma 4 (3 of 7), and the Phi-4 result leaves model-versus-harness genuinely unresolved |

The sample size is the real constraint. 238 examples per cell keeps the full cross feasible but leaves score differences of a few points inside the noise — which, to be fair, strengthens rather than weakens the central argument: if 0.794 and 0.761 are not reliably distinguishable, paying 3.2× the memory for the gap looks worse, not better.

> **In one line: the 0.794 model needs 48 GB and the 0.761 model needs 15 GB and runs twice as fast — so for most deployments the second-place model is the correct choice.** Evaluation should produce an operating point, not a ranking.

---

## Sources

- [Paper (arXiv)](https://arxiv.org/abs/2604.07035)
- [Full text (HTML)](https://arxiv.org/html/2604.07035v2)
