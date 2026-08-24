---
title: "Digital Twins of People: What Actually Reproduces, and Where It Breaks"
date: 2026-07-01T10:00:00+09:00
tags:
  - llm
  - evaluation
  - paper-review
summary: "The widely quoted '85% replication' is a figure normalised against a person's own two-week test-retest consistency, not an absolute accuracy. Read the benchmark and replication-failure literature instead of the announcements — 1,000-person generative agents, Twin-2K-500, the Stanford SCALE megastudy — and a consistent picture appears: voice reproduces, the person does not."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/digital-twin-persona-reproduction/)**

## TL;DR

- **Separate the layers or the debate is meaningless.** L1 (voice and phrasing) is conditionally achievable. L2 (personality and values on instruments) works in aggregate and fails per person. L3 (thinking and judging like that person in a situation neither of you has seen) is currently out of reach.
- **"85%" is a normalised figure, not an accuracy.** It is measured against the ceiling set by the person's own test-retest consistency two weeks later. Read as "85% of that person replicated", it is a substantial overstatement.
- **The decisive counter-evidence is the Stanford SCALE megastudy.** Twins built from roughly 128K characters per person correlate with individual responses at **r ≈ 0.197**, and rich personas do not significantly beat demographics-only on individual accuracy (**p = 0.37**).
- **Long conversations lose the persona structurally.** Persona drift appears within about eight rounds, driven by attention decay over the system-prompt tokens — and the model then contaminates toward its interlocutor's persona.
- **A gap in the literature worth naming:** almost everything here measures interview-built twins or in-context persona prompting. I found no primary study measuring the fidelity of a twin fine-tuned on someone's own chat logs.

> **Papers**: [Generative Agent Simulations of 1,000 People (2411.10109)](https://arxiv.org/abs/2411.10109) · [Twin-2K-500 (2505.17479)](https://arxiv.org/abs/2505.17479) · [Stanford SCALE megastudy (2509.19088)](https://arxiv.org/html/2509.19088v3) · [BehaviorChain (2502.14642, ACL 2025 Findings)](https://arxiv.org/abs/2502.14642) · [TwinVoice (2510.25536)](https://arxiv.org/html/2510.25536v1) · [Persona Drift (2402.10962, COLM 2024)](https://arxiv.org/html/2402.10962v1) · [Flatten/Essentialize (2402.01908, Nature MI)](https://arxiv.org/html/2402.01908v3) · [Second Me (2503.08102)](https://arxiv.org/html/2503.08102)

---

## 1. Three layers, three verdicts

The question "can you reproduce a person with an LLM" only becomes answerable once you split it:

| Layer | Definition | Verdict |
|---|---|---|
| **L1** | Statistical imitation of voice, phrasing, tone | 🟡 Conditional — surface lexicon yes, syntax and memory no |
| **L2** | Personality and values on instruments and surveys | 🟡 Conditional — aggregate yes, **per individual no** |
| **L3** | Thinking, judging and remembering like that person out of distribution | 🔴 Not currently possible |

Most public claims about digital twins and griefbots are L1 results presented as L3 capability. The distinction is not pedantic: L1 is what makes a demo persuasive, and L3 is what people believe they are buying.

## 2. What the studies measure

| Study | What it measures | Scale |
|---|---|---|
| 1,000 People (2411.10109) | Interview-built agents reproducing survey responses | 1,052 people, 2-hour semi-structured interviews → GSS, Big Five, economic games |
| Twin-2K-500 (2505.17479) | Twin survey reproduction, normalised | 2,058 people, normalised against test-retest ceiling |
| SCALE megastudy (2509.19088) | **Individual-level** reproduction accuracy | 19 pre-registered studies, 164 outcomes |
| BehaviorChain (2502.14642) | Continuous non-conversational behaviour | 15,846 behaviours, 1,001 personas |
| TwinVoice (2510.25536) | Persona simulation decomposed into six abilities | Social / Interpersonal / Narrative |
| Persona Drift (2402.10962) | Persona retention over long dialogue | LLaMA2-chat-70B, 200 self-chat pairs |
| Flatten (2402.01908) | Distortion when reproducing identity groups | 4 LLMs, 16 demographic groups |
| Second Me (2503.08102) | A local personal twin system | Qwen2.5-7B + PEFT, Apache 2.0 |

## 3. What "85%" means

It is normalised against a ceiling, and the ceiling is the person's own inconsistency.

- **1,000 People**: on held-out GSS items, agents reach **83% of participants' own self-consistency** from interviews, 82% from surveys, 86% combined. Demographics-only reaches 74%.
- **Twin-2K-500**: ceiling 81.72%, twin 71.72%, normalised **87.67%**.

So the honest reading of the headline is: *people answer differently two weeks later, and the twin is about 85% as consistent with the original answers as the person themselves is.* The denominator is doing enormous work, and it disappears in every retelling I have seen.

Also worth holding onto: interviews beat demographics by only 83% to 74%. Two hours of semi-structured interview per person buys nine points over knowing their age, income and region.

## 4. More data does not produce the person

The SCALE megastudy is the paper that settles this, and it is the one least cited in coverage of the field.

- Mean correlation **r ≈ 0.197**. Individual accuracy 0.748, against a human test-retest figure of 0.817 and a random baseline of **0.629** — barely above chance.
- **Full personas versus demographics-only: p = 0.37.** No significant difference, on individual accuracy or on population-mean estimation.
- The authors' own summary: *"not fully ready for prime time."*

Roughly 128K characters per person is a serious corpus, and it does not beat knowing someone's demographics. The mechanism is not mysterious: a model trained with cross-entropy is pulled toward the plausible majority, so the idiosyncratic signal — the part that makes the person *that* person rather than a member of their cohort — is exactly what gets regressed away. An individual is not an average, and averaging is what the objective rewards.

## 5. Out of distribution, and over time

**Out of distribution.** BehaviorChain's conclusion is that *"even state-of-the-art models struggle with accurately simulating continuous human behaviour."* The megastudy shows the same shape: when questions vary per participant — that is, when the twin faces something it was not fitted to — twin-human correlation falls. And the "held-out" in the 1,000-people study means held-out *survey items*, not novel real-world situations. L3 is not demonstrated by that result; it is not tested by it.

**Over time.** Persona Drift finds departure from the persona within about eight rounds, and attributes it to transformer attention decay — attention to the system-prompt tokens drops away as the conversation grows. The second finding is the more unsettling one: the model then drifts *toward the interlocutor's* persona, losing its own. For anything meant to be a long-running conversational stand-in for a person, that is a structural constraint, not a tuning problem.

**Where L1 sits.** TwinVoice puts discrimination at 71.2% for GPT-5 and 76.2% for Claude — **below the human baseline**, with syntactic style and memory the weakest components and lexical fidelity the only relatively strong one. Which matches the three-layer verdict: word choice transfers, sentence construction and recall do not.

## 6. What I am keeping

1. **Always ask "relative to what" before accepting a percentage.** Most of these are normalised against a test-retest ceiling. Reported as absolute accuracy, they are inflated by a large and unstated factor.
2. **Include an OOD set and a normalised ceiling in any persona evaluation.** In-distribution scores on the instrument you fitted to are close to meaningless on their own.
3. **Evaluate L1 and L3 separately and never let one stand in for the other.** "It sounds like them" is a real result. It is not evidence of "it is them", and the gap between those two claims is where every overstatement in this field lives.
4. **For a build, weight retrieval over fine-tuning.** The bottleneck is the data and the method, not the hardware — the individual signal is thin in any realistic corpus. Retrieval over actual logs plus a shallow style LoRA is the safer allocation than betting everything on fine-tuning; it also keeps facts inspectable rather than baked into weights.
5. **The honest ceiling for a local twin is a memory-and-context provider.** That is genuinely useful. Faithful replication is unproven, and the strongest available evidence says it is not close.

## 7. Assessment

| Axis | Judgement |
|---|---|
| Problem statement | ★★★ — empirical rebuttal accumulating against a heavily promoted claim |
| Contribution | Decomposes fidelity into normalisation, OOD and drift; quantifies individual-level failure at r ≈ 0.197 |
| Practicality | Local twins work as memory and context providers; faithful reproduction is unproven |
| Limits | No direct measurement of the log-mining + QLoRA path; most benchmarks are 2024–2025, single-shot, with thin independent replication |

> **In one line: a person's distinctive judgement is not sufficiently present in their data — you can build something close to their average, and not yet them.**

---

## Verification note — claims I dropped

Three strong claims failed adversarial checking (three independent deep-research passes, discarded on 2-of-3 refutation) and are excluded above. Recording them because the omissions are part of the result:

- ❌ *"BehaviorChain's best model scores 42.5% accuracy"* (0–3, discarded) → kept only the qualitative direction, that state-of-the-art models break down.
- ❌ *"LLMs flatten responses across all metrics"* (0–3, discarded) → weakened to stereotyping on some groups and some metrics; 2402.01908 rates as medium confidence.
- ❌ *"BehaviorChain documents persona drift and error accumulation"* (1–2, discarded) → drift is cited from 2402.10962 only.

---

## Sources

- [Generative Agent Simulations of 1,000 People — arXiv 2411.10109](https://arxiv.org/abs/2411.10109)
- [Twin-2K-500 — arXiv 2505.17479](https://arxiv.org/abs/2505.17479)
- [Stanford SCALE megastudy — arXiv 2509.19088](https://arxiv.org/html/2509.19088v3) ★ the key counter-evidence
- [BehaviorChain (ACL 2025 Findings) — arXiv 2502.14642](https://arxiv.org/abs/2502.14642) · [ACL Anthology](https://aclanthology.org/2025.findings-acl.813/)
- [TwinVoice — arXiv 2510.25536](https://arxiv.org/html/2510.25536v1)
- [Persona Drift (COLM 2024) — arXiv 2402.10962](https://arxiv.org/html/2402.10962v1)
- [Flatten/Essentialize (Nature MI) — arXiv 2402.01908](https://arxiv.org/html/2402.01908v3)
- [Second Me (AI-native Memory 2.0) — arXiv 2503.08102](https://arxiv.org/html/2503.08102)
- Ethics: [Philosophy & Technology 2024 (Cambridge LCFI)](https://link.springer.com/article/10.1007/s13347-024-00744-w) · [Cambridge on "digital haunting"](https://www.cam.ac.uk/research/news/call-for-safeguards-to-prevent-unwanted-hauntings-by-ai-chatbots-of-dead-loved-ones)
