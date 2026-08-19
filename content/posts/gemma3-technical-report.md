---
title: "Gemma 3: Cutting the KV Cache Sixfold for Free, and What That Says About Attention"
date: 2026-06-27T14:00:00+09:00
tags:
  - llm
  - systems
  - paper-review
summary: "Gemma 3 interleaves local and global attention at 5:1 with a 1024-token sliding window and cuts KV cache memory roughly sixfold with no measurable perplexity cost. A 27B dense model lands at Elo 1338 on Chatbot Arena — above a 671B MoE and a 405B dense model. I read the report for what it tells you about local-versus-global attention, and about post-training beating parameter count."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/gemma3-technical-report/)**

## TL;DR

- **The KV cache is nearly free to shrink.** Interleaving five local attention layers per global one, with the local window capped at 1024 tokens, drops KV cache memory from about 60% of the footprint to under 15% at 32K context — and perplexity barely moves. The ablation says even 7:1 costs almost nothing.
- **Post-training beat parameter count.** Gemma 3 at 4B outscores Gemma 2 at 27B on maths (75.6 versus 55.6). The 27B model reaches Elo 1338 on Chatbot Arena, above a 671B MoE and 69 points above a 405B dense model.
- **The received wisdom on distillation teachers is wrong for long runs.** "Smaller teacher is better" holds only for short training budgets. With hundreds of billions of tokens, the bigger teacher wins.
- **For local deployment the binding constraint is the KV cache, not the weights.** That belongs in your model selection criteria and usually is not there.

> **Source**: [arXiv:2503.19786](https://arxiv.org/abs/2503.19786) · Gemma Team, Google DeepMind · open weights at 1B / 4B / 12B / 27B

---

## 1. The claim

Gemma 3 is a family of lightweight open multimodal models from 1B to 27B. Three improvements over the previous generation: vision understanding via an integrated SigLIP encoder, 128K-token context, and wider multilingual coverage. Architecturally, the load-bearing change is **shifting the local-to-global attention ratio to 5:1** and capping the local sliding window at 1024, which is what makes long context affordable.

## 2. Architecture

| Model | Vision encoder | Embedding params | Non-embedding params | Context |
|---|---|---|---|---|
| 1B | none | 302M | 698M | 32K |
| 4B | 417M (SigLIP) | 675M | 3,209M | 128K |
| 12B | 417M (SigLIP) | 1,012M | 10,759M | 128K |
| 27B | 417M (SigLIP) | 1,416M | 25,600M | 128K |

What changed from the previous generation:

| Element | Gemma 2 | Gemma 3 | Effect |
|---|---|---|---|
| Local:global ratio | 1:1 | **5:1** | KV cache memory 60% → <15% at 32K |
| Sliding window | 4096 | **1024** | Negligible perplexity impact, large memory saving |
| Attention capping | Soft-capping | **QK-norm** | Better stability |
| RoPE base frequency (global) | 10K | **1M** | Supports 128K context |
| Tokeniser | Gemma-specific (256K) | **Shared with Gemini 2.0 (262K)** | Better balance for non-English |

**The 5:1 interleaving is the contribution.** Five local sliding-window layers, then one global layer. Local layers see 1024 tokens; only the global layers attend across the full context.

```
Layer pattern: [Local, Local, Local, Local, Local, Global, Local, Local, ...]

KV cache memory (2B model, 128K context):
  Global only:        ~6000 MB
  L:G = 5:1, sw=1024:  ~1000 MB   ← roughly 6× less
```

## 3. Training

| Model | Training tokens | Infrastructure | Chips |
|---|---|---|---|
| 1B | 2T | TPUv5e | 512 |
| 4B | 4T | TPUv5e | 2,048 |
| 12B | 12T | TPUv4 | 6,144 |
| 27B | 14T | TPUv5p | 6,144 |

- **Knowledge distillation**: 256 logits per token sampled weighted by teacher probability, trained with cross-entropy; non-sampled logits are zeroed and renormalised.
- **Vision encoder**: SigLIP 400M at 896×896, shared across 4B/12B/27B and **frozen during training** — embeddings are precomputed, so the language model training cost does not grow.
- **Post-training**: distillation from a larger instruction-tuned teacher, plus BOND (best-of-N distillation), WARM (weight-averaged reward models) and WARP (weight-averaged rewarded policies). Reward signals combine human feedback, code execution feedback and ground-truth maths rewards.

### Quantisation-aware training

About 5,000 steps of QAT fine-tuning from the original checkpoint, shipped as quantised releases:

| Model | bf16 (GB) | Int4 (GB) | Int4 block=32 (GB) | SFP8 (GB) |
|---|---|---|---|---|
| 1B | 2.0 | 0.5 | 0.7 | 1.0 |
| 4B | 8.0 | 2.6 | 2.9 | 4.4 |
| 12B | 24.0 | 6.6 | 7.1 | 12.4 |
| 27B | 54.0 | 14.1 | 15.3 | 27.4 |
| 27B + KV (32K) | 72.7 | 32.8 | 34.0 | 46.1 |

That last row is the one to read twice. At 32K context the KV cache adds 18.7 GB — and it adds the same 18.7 GB whether the weights are bf16 (54.0 → 72.7) or Int4 (14.1 → 32.8). **Quantising the weights does not quantise the cache**, so at Int4 the cache is larger than the model. That is why the 5:1 architecture work matters more than another bit of weight compression: below a certain weight precision, the cache is the whole problem.

## 4. Pan & Scan: the fix for a fixed-resolution encoder

SigLIP runs at a fixed 896×896, so non-square or high-resolution images lose small text and small objects. Pan & Scan is an inference-time workaround: split the image into equal non-overlapping crops, resize each to 896×896, feed them all. Applied at inference only, and switchable.

| Model | DocVQA | InfoVQA | TextVQA |
|---|---|---|---|
| 4B without P&S | 72.8 | 44.1 | 58.9 |
| 4B with P&S | **81.0** (+8.2) | **57.0** (+12.9) | **60.8** (+1.9) |
| 27B without P&S | 85.6 | 59.4 | 68.6 |
| 27B with P&S | **90.4** (+4.8) | **76.4** (+17.0) | **70.2** (+1.6) |

Up to +17 points on infographic QA from a crop-and-resize loop. Documents and charts are where fixed resolution hurts most.

## 5. Results

### Chatbot Arena (27B instruction-tuned)

| Rank | Model | Elo | Open | Type | Params |
|---|---|---|---|---|---|
| 1 | Grok-3-Preview | 1412 | – | – | – |
| 1 | GPT-4.5-Preview | 1411 | – | – | – |
| 6 | DeepSeek-R1 | 1363 | yes | MoE | 671B/37B |
| **9** | **Gemma-3-27B-IT** | **1338** | **yes** | **Dense** | **27B** |
| 13 | DeepSeek-V3 | 1318 | yes | MoE | 671B/37B |
| 14 | Claude 3.7 Sonnet | 1309 | – | – | – |
| 28 | LLaMA-3.1-405B | 1269 | yes | Dense | 405B |
| 59 | Gemma-2-27B-IT | 1220 | yes | Dense | 27B |

A 27B dense model at 1338 — above a 671B MoE, 69 points above a 405B dense model, and +118 Elo over its own predecessor at the same size.

### Benchmarks (zero-shot)

| Benchmark | Gemma 2 27B | Gemma 3 4B | Gemma 3 27B | Gemini 1.5 Pro |
|---|---|---|---|---|
| MMLU-Pro | 56.9 | 43.6 | **67.5** | 75.8 |
| MATH | 55.6 | 75.6 | **89.0** | 86.5 |
| HiddenMath | 14.8 | 43.0 | **60.3** | 52.0 |
| LiveCodeBench | 20.4 | 12.6 | **29.7** | 34.2 |
| GPQA Diamond | 34.3 | 30.8 | **42.4** | 59.1 |
| Global MMLU-Lite | 68.6 | 54.5 | **75.1** | 80.8 |
| MMMU (val) | – | 48.8 | **64.9** | 65.9 |
| FACTS Grounding | 62.4 | 70.1 | **74.9** | 80.0 |

Generation over generation at 27B: HumanEval 51.8 → **87.8**, BBH 74.9 → **87.6**, GSM8K 91.1 → **95.9**.

The number that reframes things is **Gemma 3 at 4B scoring 75.6 on MATH against Gemma 2 at 27B scoring 55.6**. Same family, one generation apart, a sixth of the parameters, twenty points better. That gap is post-training, not scale.

## 6. The ablations worth knowing

**Local:global ratio.** 3:1 and 5:1 are both perplexity-neutral; 7:1 costs about +0.05 at 2B. The paper adopts 5:1, but the ablation says the memory saving is close to a free lunch across the whole range.

**Sliding window size.** Perplexity moves by ±0.01 across 512 to 4096. Adopting 1024 costs nothing measurable.

**Teacher size in distillation.** Short runs (tens of billions of tokens) favour a smaller teacher, which acts as a regulariser. Long runs (hundreds of billions) favour a bigger one. The conventional "smaller teacher is better" was derived from short experiments — it does not survive a real token budget.

**Vision encoder resolution.** 256 → 448 → 896 gives DocVQA 31.9 → 45.4 → **59.8**. Resolution matters, especially for text in images.

## 7. Memorisation

- Memorisation rate is **one to two orders of magnitude lower** than every previous Gemma and Gemini model on a log scale.
- Differences between 1B, 4B, 12B and 27B are small, with 1B lowest.
- Approximate memorisation is roughly **24× more common** than exact memorisation — a reminder that exact-match measurement understates the problem.
- Running the outputs classified as memorisation through personal-data detection found **zero personal data across all Gemma 3 models**.

## 8. What I am keeping

1. **The KV cache shrinks for free.** 5:1 interleaving with a 1024 window cuts it about sixfold with no perplexity cost. If memory is your bottleneck in long-context inference, look here before anything else.
2. **Post-training can beat model size.** 4B beating the previous 27B on maths and coding is a distillation-plus-RL result, not a scaling result.
3. **Bigger teacher, longer training.** The received wisdom holds only for short runs.
4. **Adding vision did not cost text performance.** A frozen encoder with precomputed embeddings is what makes that possible.
5. **Pan & Scan is table stakes for a fixed-resolution VLM.** +8 to +17 points on document and infographic understanding from an inference-time loop.

## 9. What this changes for local deployment

Gemma 3 was explicitly designed for consumer hardware, which makes it a serious candidate for on-premise work.

| Scenario | Model | Memory (Int4) | Rationale |
|---|---|---|---|
| Mobile / edge | 1B | 0.5 GB | Text only, 32K context |
| Laptop / mini PC | 4B Int4 | 2.6 GB | Includes vision, previous-generation-27B-class quality |
| Server / larger GPU | 27B Int4 | 14.1 GB | MATH 89.0, Gemini-1.5-Pro-class on several axes |

Two things I took into my own model selection:

- **Put KV cache memory in the selection criteria.** At 32K context the cache stays under 15% of the weight footprint here, against 60%-plus for global-only architectures. Same RAM, longer context — and for log analysis, where inputs are long and outputs short, that is the entire question.
- **Record post-training differences alongside benchmark scores.** The 4B-versus-27B result says the recipe moves performance more than the parameter count does, and a benchmark table that omits the recipe hides the reason.

## 10. Assessment

| Aspect | Judgement |
|---|---|
| Source | arXiv technical report — not peer reviewed, but an official DeepMind report |
| Problem statement | Clear: the KV cache cost of long context, and the multimodal gap in open models |
| Contribution | (1) 5:1 local:global attention (2) a 27B dense model in the Arena top ten (3) 4B surpassing the previous 27B |
| Practicality | Very high — open weights, QAT quantisations shipped, consumer hardware targeted |
| Limits | Avoids direct comparison with external models on the grounds of differing evaluation setups; Arena Elo is preliminary; quality degrades sharply past 128K; the frozen SigLIP encoder caps the vision side |

> **In one line: cutting the KV cache sixfold at 5:1 interleaving with no quality loss means most attention only needs to be local.** That is an architectural principle for the long-context era — and a 27B dense model beating a 671B MoE is a second reminder that the post-training recipe is doing more work than the parameter count.
