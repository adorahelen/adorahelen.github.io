---
title: "Where LLM Safety Alignment Actually Lives, and the Three Layers at Which It Comes Off"
date: 2026-06-30T10:00:00+09:00
tags:
  - ai-security
  - llm
  - alignment
  - red-teaming
  - interpretability
summary: "Refusal behaviour in an open-weight model is not a rules file — it is behaviour trained into the weights, and it comes off at three distinct layers: the prompt (jailbreaks), the activation space (abliteration), and retraining (fine-tuning degradation). I worked through the mechanism of each from the published literature and pulled out what it means for defence. The short version: the more safety concentrates in a single direction and in the first few tokens, the cheaper it is to remove."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/llm-safety-alignment-bypass/)**

## TL;DR

- **Safety is behaviour in the weights, not a file.** A deployed system has three layers — alignment training baked into the weights, a system prompt at runtime, and an external classifier. Only the first is in the model itself, and it turns out to be shallower than its cost to train suggests.
- **It comes off at three layers, at very different prices.** Prompt-level jailbreaks (cheapest, temporary), activation-space removal or *abliteration* (cheap, permanent, and evidence that refusal concentrates in roughly one direction), and fine-tuning (moderate, permanent, and it happens even with benign data).
- **Refusal rate 0% is not unlocked knowledge.** Abliteration strips a behavioural lock. The genuinely dangerous material was largely filtered out of training data in the first place, so what actually changes is that over-refusal on legitimate security questions goes away. Conflating the two is the most common error in this area.
- **For defenders**: alignment needs to be deeper (across the whole response, not the first few tokens) and more distributed (not one direction), with independent layers around it. A single behavioural lock is a single point of failure.

---

## 1. The rules are trained into the weights

Refusal is not a `rules.txt` the model consults. It is behaviour learned during training and stored in the weights. A production deployment typically stacks three layers:

| Layer | Location | Nature | Cost to change |
|---|---|---|---|
| ① **Alignment training** | **In the weights** | Instruction tuning plus RLHF/DPO teach "refuse harmful requests" as a behaviour | Hard — needs retraining. **But shallower than that implies, as below** |
| ② **System prompt** | External, runtime | "You answer safely" style instructions | Trivial — swap it out |
| ③ **External filter** | External, separate model | Input/output classification. For Gemma, Google ships **ShieldGemma** as a separate safety classifier | Operated separately |

So the answer to "is safety built in?" is that ① is in the weights, and ②③ are added at deployment. Download an open-weight model and you get ① only.

---

## 2. Three layers at which the guardrail comes off

### Layer A — the prompt (model unchanged): jailbreaks

Leave the weights alone and manipulate the input. Role-play ("a character with no restrictions"), hypothetical framings, encoding tricks, prefix injection.

- **Character**: temporary and model-specific. Patch the model and the specific technique stops working.
- **Research tooling**: `garak`, NVIDIA's LLM vulnerability scanner, automates probes in this space — for defenders, it is how you measure your own model's jailbreak surface rather than guess at it.

### Layer B — the activation space: abliteration

The key paper is **Arditi et al. (2024), "Refusal in Language Models Is Mediated by a Single Direction"** (NeurIPS 2024, [arXiv 2406.11717](https://arxiv.org/abs/2406.11717)).

```
Observation: models represent concepts as directions (vectors) in activation space.
Finding:     refusal turns out to be mediated largely by ONE direction.

Extracting the refusal direction:
  r = mean(activations on harmful prompts) − mean(activations on harmless prompts)
  → the difference of the two means is the refusal direction

The intervention:
  orthogonalise r out of the activations or the weights
  → the model's capacity to *express* refusal is weakened, with no retraining
```

- **Character**: lightweight. You touch inference-time activations, not a training loop, so it runs on modest hardware.
- **Why it matters to a defender**: it is direct evidence that if safety concentrates in a **single direction**, it is cheap to remove.

### Layer C — fine-tuning: overwriting alignment

Train on compliant data — question-and-answer pairs where the assistant does not refuse — and alignment is overwritten.

- The uncomfortable result is how little it takes. Qi et al. (ICLR 2024), *"Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!"* ([arXiv 2310.03693](https://arxiv.org/abs/2310.03693)) shows degradation **even when fine-tuning on benign data**.
- The follow-up diagnosis is **shallow alignment**: safety behaviour concentrates in the **first few tokens** of a response, so disturbing that region is enough to collapse the rest. Qi et al. (2024), *"Safety Alignment Should Be Made More Than Just a Few Tokens Deep"* ([arXiv 2406.05946](https://arxiv.org/abs/2406.05946)).

---

## 3. The three side by side

| Layer | What it touches | Cost | Persistence | Lesson for defenders |
|---|---|---|---|---|
| **A — jailbreak** | Input | Lowest | Temporary, patchable | Exposes the **limits** of input filtering and detection |
| **B — abliteration** | Activation/weight direction | Low | Permanent (weights modified) | Safety concentrated in **one direction** — alignment needs to be distributed |
| **C — fine-tuning** | Weights, retrained | Moderate | Permanent | An open-weight **supply-chain threat** — safety can be absent from a redistributed model |

---

## 4. What a defender should actually take from this

1. **From A**: input-stage filtering and detection alone will not hold. Runtime defence is a supporting layer, not the boundary.
2. **From B**: if safety rides on a small number of directions or features, one orthogonalisation removes it. Alignment has to be **distributed** across multiple directions and layers to be robust.
3. **From C**: an open-weight model carries a structural supply-chain risk — anyone can fine-tune and redistribute it with the safety gone. This is precisely why an **independent external safety layer** (ShieldGemma and equivalents) exists separately from the model: even with the in-weight alignment broken, input and output get filtered once more.

> **Design principle**: make alignment (a) deeper, across the full response length, (b) more distributed, so it does not depend on one direction, and (c) layered, in-weight alignment plus an external classifier — so that one layer failing does not take the whole thing with it.

---

## 5. Two kinds of dangerous content are handled in two different ways

This distinction explains most of what looks inconsistent about model behaviour.

| Content type | How it is handled | Why |
|---|---|---|
| Purely harmful and separable (CSAM and similar) | **Data filtering** — never learned | Possession is illegal and it does not entangle with legitimate knowledge, so it can be cleanly excluded. Abliteration does not surface much of it |
| Dual-use knowledge | **Learned, with a behavioural lock** (alignment) | It cannot be separated from legitimate knowledge, so a refusal behaviour is layered on top |

**Dual-use entanglement is the crux.** A large share of risky knowledge is inseparable from ordinary knowledge:

- Mechanisms of toxicity are in medical textbooks and on Wikipedia; exploitation principles are in security papers.
- Strip chemistry, biology, pharmacology, security and history and the model becomes incompetent at those fields entirely.
- So dual-use areas end up as "the model knows, and the lock is behavioural". That is exactly why abliteration and jailbreaks can surface some of it — and why separable worst-case content stays unavailable, because it was never learned.

**Four layers of defence in depth**, which is what you build when you refuse to bet everything on the behavioural lock:

| # | Layer | Role | Weakness |
|---|---|---|---|
| ① | Data filtering | Never learn the worst of it (including "safety pretraining" that lowers CBRN density) | Cannot remove dual-use content |
| ② | Alignment (RLHF/DPO) | Know it, refuse it | **Shallow — comes off via abliteration or fine-tuning** |
| ③ | External filter | Input/output classifiers such as ShieldGemma or moderation endpoints | Separate to operate, and evadable |
| ④ | Deployment control | Rate limits, KYC, monitoring, reporting | Not applicable to open weights |

② is the layer that fails, so ①③④ are stacked around it.

## 6. Where the industry is moving: from behavioural locks to deep alignment

"Learn everything, then add a behavioural lock" is the older posture. As abliteration and jailbreak research demonstrated that **the lock comes off easily**, frontier labs moved:

| Direction | Content |
|---|---|
| **Dangerous capability evaluation** | Measure before release whether a model materially uplifts, say, a biological attacker; hold release or add mitigation above a threshold |
| **Unlearning** | Remove specific risky knowledge after training |
| **Tamper-resistant safeguards** | Alignment that survives abliteration and fine-tuning |

> In one line: dual-use knowledge is learned with a behavioural lock, which is why some of it can be surfaced; purely harmful content is never learned, which is why it usually cannot; and the industry has concluded the lock alone is insufficient, moving toward data filtering plus deeper alignment plus external filters.

## 7. If the goal is removing over-refusal, abliteration is the wrong tool

> **The distinction that matters most: a 0% refusal rate is not unlocked knowledge.** Abliteration removes the behavioural lock only. Genuinely harmful capability is still thin because of the data filtering in §5. What you actually observe is that an over-refusal set — legitimate security and forensics questions — starts getting answered. That is the real effect, and it is what an experiment here can measure.

If what you want is a model that stops refusing legitimate penetration-testing, CTF or security-education questions, abliteration is not the best route. It is **not a precision tool for over-refusal; it is a blunt instrument that removes refusal as such**, so legitimate and malicious refusals disappear together. There is no selective unlock knob. In order of fitness:

| Option | Approach | Fit | Assessment |
|---|---|---|---|
| **System prompt** ⭐ try first | Model unchanged. State the authorised pentest/CTF/security-education context | Practical, immediate | No surgery, freely reversible, unambiguously legitimate. It **exploits** the very sensitivity to system prompts that made some models look robust |
| **A security-tuned open model** | A model fine-tuned for offensive and defensive security work | Practical, well-aimed | **The balanced option**: answers security questions while its licence still forbids clear abuse. Cleaner than a blunt instrument, and 7B-class runs on a laptop |
| **A pre-abliterated model** | Download an uncensored variant | Reproduction skipped | Inherits the blunt-instrument problem — all refusal removed. The publishers themselves note these are not safety-optimised |
| **Abliterate it yourself** | The full pipeline in §8 | **Research and reproduction** | Good for learning the mechanism and measuring how shallow alignment is; inefficient as a practical unlock |

**Recommendation: for practical use, start with the system prompt and escalate to a security-tuned model. For research and measurement, do the reproduction.** These are different goals and they want different tools.

**Scope line, fixed**: help with security questions in authorised penetration-testing, CTF, security-education and defensive-research contexts — yes. Attacks or malware that work against real targets — out of scope regardless of how the model is configured, and models are poor at that anyway.

## 8. Reproducing abliteration: pipeline, memory, and where it bites

The serving stack is not the surgery target. Quantisation comes last.

```
1. Fetch the original HF safetensors   (fp16/bf16 — a format you can operate on)
2. Abliteration in PyTorch            (activation hooks → refusal direction → orthogonalise)
3. Save modified safetensors
4. Convert and quantise to GGUF       (llama.cpp convert_hf_to_gguf.py)
5. Register with the serving runtime   ← back to serving
```

### Inference and surgery have completely different hardware requirements

> **The key point: running an already-modified model is light. Only doing the surgery yourself is heavy.**

**Running a Q4 quantisation** — memory-optimised C++ runtimes:

| Model | Disk (Q4) | RAM to run | 16 GB laptop |
|---|---|---|---|
| 4B abliterated | ~2.5 GB | ~5 GB | ✅ comfortable |
| 7B / 8B | ~4.7–5 GB | ~7–9 GB | ✅ comfortable (~10 GB even at Q8) |
| 27B | ~16–17 GB | ~19–20 GB | ❌ server territory |

**Loading a bf16 original plus activations** — PyTorch. **Memory ≈ parameters × 2 GB**, roughly twice inference:

| Model | bf16 memory | 16 GB laptop | Server |
|---|---|---|---|
| 2B | ~5 GB | ✅ | ✅ |
| 3B / 4B | ~7–12 GB | ⚠️ tight | ✅ |
| 7–8B | ~16–18 GB | ❌ effectively no (swapping) | ✅ |
| 31B | ~62 GB | ❌ | ✅ (32 GB is tight; 128 GB is comfortable) |

**Why surgery costs more than double — editing a raw file versus viewing a compressed one:**

1. **Precision per weight.** Inference runs Q4 (~4 bits, 0.5 B per weight → 3.5 GB for 7B); surgery needs bf16 (16 bits, 2 B per weight → 14 GB for 7B). **Four times the memory for the same model.** Q4 is a lossy approximation, and extracting a refusal direction or orthogonalising on top of it breaks — so you operate at bf16 and re-compress afterwards.
2. **Activations.** Extracting the direction requires `output_hidden_states=True`, holding per-layer intermediates in memory.
3. **Engine.** Inference uses aggressively memory-optimised runtimes; surgery uses PyTorch, which loads at full precision and is less frugal.

For 7B: **running** is 3.5 GB of weights plus KV and overhead, about 7–8 GB ✅. **Surgery** is 14 GB of weights plus activations, about 16–18 GB ❌.

**"Copy the code and it just works" is overstated.** Four things have to be handled before the numbers mean anything:

1. **Model dependence.** Some model families give way easily and others resist. The same code produces wildly different before/after drops — and **that comparison is itself the result**, because it falsifies "it just works".
2. **Layer selection.** A single `L = 0.6 · depth` guess is crude. The real method is to **sweep layers and token positions** and pick the best direction against a held-out set. Skip it and you conclude "it does not work" — because you did not tune it, not because the method failed.
3. **Refusal judgement.** Keyword matching catches explicit refusals and **misses soft deflection** ("consult a professional…"), so the numbers are wrong. You need a classifier to judge.
4. **Technical traps.** Unsupported operations on some accelerator backends, hook return signatures (tuple versus tensor), logit soft-capping in certain model families — expect to debug the first run.

> **What the reproduction actually shows you**: weakly aligned models drop sharply once tuned; strongly aligned ones may not drop at all — and **that is the data on how shallow alignment is**.

📊 **Measured reproduction**: [Reproducing Abliteration — refusal is one direction, but ripping it out damages the model](/posts/reproducing-abliteration-qwen25-3b/). Refusal rate **1.00 → 0.00** on Qwen2.5-3B-Instruct via inference-time ablation at layer 21 — while layers 10, 14 and 18 gave exactly zero effect (the sweep is not optional), and over-refusal was 0.00 from the baseline (removing refusal is not unlocking knowledge). Fourteen minutes end to end.

## 9. Scope of legitimate research

- **In scope**: reproducing *why* abliteration and jailbreaks work, in order to practise **detecting and defending** against them.
- **Out of scope**: using any of it to produce actual harmful content.

Defensive follow-ups worth doing:

- Run `garak` against your own model to measure its jailbreak surface and map which probes land.
- Invert the Arditi refusal direction into a **detection signal** — monitor activations for projection onto it to catch bypass attempts at runtime.
- Quantify degradation by comparing safety benchmarks before and after fine-tuning, and build that into a verification pipeline for redistributed models.

---

## Appendix: standards, law and papers

> Verify exact URLs and current versions before citing — some policies have changed status.

### CSAM filtering

| Category | Items |
|---|---|
| Technology and bodies | PhotoDNA (Microsoft; hash matching, effectively the standard), NCMEC CyberTipline, IWF (UK hash list), Thorn "Safer", Google CSAI Match |
| US law | 18 U.S.C. §2258A (mandatory NCMEC reporting), §2252/§2252A, PROTECT Act 2003 |
| EU/UK law | Directive 2011/93/EU, the proposed CSA Regulation, UK Online Safety Act 2023 |
| Papers | Thiel, Stanford Internet Observatory (Dec 2023), "Identifying and Eliminating CSAM in Generative ML Training Data and Models" — the LAION-5B findings that are the direct reason model cards emphasise CSAM filtering; Thorn & All Tech Is Human (2024), "Safety by Design for Generative AI" |

### CBRN

| Category | Items |
|---|---|
| Policy and governance | ~~US EO 14110 (Oct 2023; reporting duties for models above 10²⁶ FLOP)~~ **revoked Jan 2025 and replaced by successor policy**; EU AI Act (Aug 2024; risk assessment duties for general-purpose models with systemic risk at the 10²⁵ FLOP threshold); lab frameworks: Anthropic's RSP, OpenAI's Preparedness Framework, Google DeepMind's Frontier Safety Framework — all with explicit CBRN thresholds |
| Reports | Mouton et al., RAND (2024), "The Operational Risks of AI in Large-Scale Biological Attacks"; OpenAI (2024), "Building an early warning system for LLM-aided biological threat creation"; Anthropic's published biosecurity red-team evaluations |

### RLHF / DPO safety tuning

| Technique | Papers |
|---|---|
| RLHF | Christiano et al. (2017), "Deep RL from Human Preferences"; Ouyang et al. (2022), InstructGPT; Bai et al. (2022), "Training a Helpful and Harmless Assistant with RLHF"; Bai et al. (2022), "Constitutional AI" |
| DPO | Rafailov et al. (2023), "Direct Preference Optimization: Your Language Model Is Secretly a Reward Model" (NeurIPS 2023) |
| Alignment is shallow | Arditi et al. (2024), single direction; Qi et al. (2024), shallow alignment; Qi et al. (2023), fine-tuning degradation |

### Umbrella standards

| Body | Document | Content |
|---|---|---|
| ISO | ISO/IEC 42001:2023 | AI management system (AIMS) — the "ISO 9001 for AI" |
| ISO | ISO/IEC 23894:2023 | AI risk management |
| ISO | ISO/IEC 22989:2022 | AI concepts and terminology |
| NIST | NIST AI RMF 1.0 (Jan 2023) | AI risk management framework |
| NIST | NIST AI 600-1 (Jul 2024) | Generative AI profile, including CBRN and CSAM risks |
| EU | EU AI Act (Aug 2024) | The first comprehensive AI law; risk-based regulation |

> ⚠️ There is no single ISO standard that addresses CSAM or CBRN directly. CSAM is handled in law, CBRN in safety frameworks and governance, and ISO/NIST sit above both as management and risk-process standards. The starting point for AI safety standards is **ISO 42001 and the NIST AI RMF**.

---

## Sources

- Arditi et al., **"Refusal in Language Models Is Mediated by a Single Direction"**, NeurIPS 2024 ([arXiv 2406.11717](https://arxiv.org/abs/2406.11717))
- Qi et al., **"Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!"**, ICLR 2024 ([arXiv 2310.03693](https://arxiv.org/abs/2310.03693))
- Qi et al., **"Safety Alignment Should Be Made More Than Just a Few Tokens Deep"**, 2024 ([arXiv 2406.05946](https://arxiv.org/abs/2406.05946))
- Google, **"ShieldGemma: Generative AI Content Moderation Based on Gemma"** ([arXiv 2407.21772](https://arxiv.org/abs/2407.21772))
- `garak` — LLM vulnerability scanner (NVIDIA; Derczynski et al.)
