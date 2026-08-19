---
title: "Reading Korea's AI Red Teaming Guide, Then Checking All 25 of Its References — and Correcting Three Entries"
date: 2026-07-10T10:00:00+09:00
featured: true
tags:
  - ai-security
  - red-teaming
  - llm
  - governance
  - interpretability
summary: "I worked through KISA's AI Security Red Teaming Guide (July 2026) and traced every one of its 25 references back to the primary source. Three things stood out: automated jailbreaking has been industrialised, interpretability is dual-use in the strict sense that defence and attack read from the same map, and AI vulnerabilities have no permanent patch. I also found three factual corrections to the guide's tooling appendix."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/kisa-ai-redteaming-guide/)**

## TL;DR

- **Automated attacks have been industrialised.** GCG (2023) optimised one suffix at a time; AmpleGCG learned the *distribution* of successful suffixes and emits 200 per query in four seconds; Joint-GCG carried the same gradient paradigm into RAG; Subspace Rerouting moved the attack surface inside the model. Defensive testing has to assume a **distribution** of transferable attacks, not individual prompts.
- **Interpretability is dual-use in the strict sense.** The white-box tooling that answers "why did the guardrail fail" is the same tooling that designs the next bypass. Subspace Rerouting (2025) is precisely that inversion.
- **There is no permanent patch.** A traditional vulnerability is gone once patched. A vulnerability distributed across learned behaviour comes back after a fine-tune or a prompt change — which is why regression testing and continuous monitoring are not optional extras here.

---

## The document

- **Document**: KISA and the Ministry of Science and ICT, *AI Security Red Teaming Guide* (July 2026, Korean)
- **Underlying standard**: ISO/IEC AWI TS 42119-7 (Testing of AI — Part 7: Red teaming) — **still under development, not yet in force**
- **Contributors**: SK Shieldus EQST Lab; reviewed by Jong-Hong Jeon, ETRI

It is a practical guide for Korean organisations setting up and running an AI red team, structured in four stages: **team composition → preparation → execution → reporting**. What follows is (1) a summary, (2) what I found tracing its 25 references back to arXiv papers, GitHub repositories, official blogs and policy documents, (3) some conceptual sorting, and (4) how this reads to someone who runs a SIEM.

The guide's central claim, as I read it: **red teaming is not a one-off pre-deployment check but continuous governance against attacks that automate, transfer and scale.**

---

## 1. What the guide says

### Threat taxonomy

Four layers under test (data / model / application and interface / infrastructure), with coded threats:

| Category | Codes | Items |
|---|---|---|
| Data | D01–D03 | Imbalanced or inaccurate data, insufficient de-identification |
| Model | M01–M08 | Leakage of training data, vector DB, system prompt or model; hallucination; jailbreak; improper output handling; model DoS |
| Agent | A01–A04 | Poor tool design, hijacking, DoS, memory poisoning |
| Supply chain | S01–S04 | Data and model poisoning, vulnerable inference engines and extensions |

Pulling **agent threats** out as their own category is the notable choice. A system that calls tools and holds memory has a different attack surface from a bare model.

### Methodology

- **Perspective**: adversarial / **benign** — the latter tests errors and hallucinations triggered by ordinary users with no ill intent
- **Access level**: black box / grey box / white box
- **Decision tree**: service impact → available people → automation resources → level of internal access, evaluated in that order to pick a strategy
- **Timing**: integrated across development, deployment and monitoring (continuous / periodic / challenge-based)

### Execution — two-stage verification

1. **Automated red teaming** — large-scale payload injection, an attacker LLM for mutation, LLM-as-a-Judge for classification
2. **Expert deep red teaming** — sustained role-play, multi-turn attacks, and where white-box access exists, chain-of-thought and attention analysis to establish root cause

### Reporting

The guide fixes four report elements (overview / scope including what was excluded *and why* / vulnerability summary / per-item detail) and requires stating that **results are a snapshot of one point in time**. Then: prioritise with a risk matrix → re-verify alongside regression tests → **avoid over-defence** → transition to continuous monitoring → connect to CVD/VDP.

I was glad to see over-defence written in explicitly. Pushing the refusal rate up also blocks legitimate questions — which is exactly the axis I measured in [my abliteration reproduction](/posts/reproducing-abliteration-qwen25-3b/).

---

## 2. What tracing the 25 references showed

The guide's claims root into three distinct lineages.

| Lineage | Representative sources | Where it lands in the guide |
|---|---|---|
| Standards and policy | ISO/IEC AWI TS 42119-7, national CVD/VDP roadmap | Overall structure, ch. VI |
| Operating lessons from frontier labs | Microsoft, Anthropic, Google, OpenAI, Japan AISI | ch. II, III, V |
| Attack and defence research | The GCG line, RAG poisoning, interpretability | ch. V |

### 2-1. The trajectory of automated attacks

Following the technical basis for the "automated red teaming" section, the attacks evolve in four steps.

- **GCG** (Zou et al. 2023, [arXiv 2307.15043](https://arxiv.org/abs/2307.15043)) — the origin of automated jailbreaking. Gradient plus greedy search optimises an **adversarial suffix** so the model opens with "Sure, here is…" instead of a refusal. The decisive finding was **transfer**: suffixes built on open models worked on commercial ones. Alignment alone is not a defence.
- **AmpleGCG / AmpleGCG-Plus** (Liao & Sun 2024, [arXiv 2404.07921](https://arxiv.org/abs/2404.07921); Kumar et al. 2024, [arXiv 2410.22143](https://arxiv.org/abs/2410.22143)) — instead of optimising one suffix, learn the **distribution** of successful ones with a generative model: 200 candidates per query in about four seconds. Plus improves on gibberish suffixes and gets past purpose-built defences such as circuit breakers. This is the industrialisation step.
- **Joint-GCG** (Wang et al. 2025, [arXiv 2506.06151](https://arxiv.org/abs/2506.06151)) — the same paradigm extended **into RAG**. Retriever and generator are attacked under a single objective, producing poisoned documents that are both easy to retrieve and effective at corrupting generation. This is the empirical version of the knowledge-base poisoning and indirect prompt injection the guide warns about.
- **Subspace Rerouting** (Winninger et al. 2025, [arXiv 2503.06269](https://arxiv.org/abs/2503.06269)) — moves the attack point from the prompt surface **into the model's representation space**. Interpretability locates an "acceptance subspace" that does not trigger refusal, embeddings are rerouted there, and jailbreaks that used to take hours land in seconds at 80–95% success.

**The implication**: because attacks automate, transfer and scale, defence has to be verified continuously at the level of **transferability and internal representations**, not individual prompts.

### 2-2. What white-box root-cause analysis stands on

The guide's line about "using chain-of-thought and attention analysis to establish why the guardrail was ignored" sits on top of a mechanistic interpretability stack. Four layers:

1. **Superposition** (Elhage et al. 2022, *Toy Models of Superposition*) — networks store more features than they have dimensions, overlapping them. "One neuron, one concept" does not hold, which is the theoretical reason attention alone cannot establish cause.
2. **Monosemanticity / SAEs** (Bricken et al. 2023, *Towards Monosemanticity*) — sparse autoencoders decompose overlapped activations back toward one-meaning-one-feature, which is what makes a "refusal" feature traceable at all.
3. **Circuits** (Wang et al. 2022, IOI circuit, ICLR) — reverse-engineered a 26-attention-head circuit performing a specific task in GPT-2 through causal intervention. The direct ancestor of "attention analysis".
4. **Steering vectors** (Turner et al. 2023, ActAdd, [arXiv 2308.10248](https://arxiv.org/abs/2308.10248)) — build a vector from activation differences and add or subtract it during the forward pass. Subtract the refusal direction and you have a jailbreak.

**This is the part of the guide I think matters most.** Defence ("why did it break") and attack ("how do I break it") **read from the same map of internal representations**. Push step 4 and you get abliteration; Winninger (2025) is the case of that same interpretability being inverted into an attack design. White box is therefore both the deepest and least practical method, and simultaneously the only real root-cause tool.

### 2-3. The tooling appendix — where I found three corrections

Checking the appendix against the actual repositories and official announcements turned up three factual issues.

| Tool | Maintainer | Strength | Licence | Fits |
|---|---|---|---|---|
| DeepTeam (+DeepEval) | Confident AI | 50+ vulnerabilities and 20+ attack techniques out of the box, runs locally | Apache 2.0 | Broad pre-production scanning |
| garak | NVIDIA | Static, dynamic and adaptive probing with audit reporting | Apache 2.0 | Auditing a model itself |
| promptfoo | promptfoo | Context-aware dynamic testing, CI/CD integration | MIT | Teams shipping an app frequently |
| Inspect AI | UK AISI | 200+ prebuilt evaluations, modular | MIT | Standardised safety benchmarking |
| PyRIT | Microsoft AIRT | 100+ real engagements behind it, human-in-the-loop, multi-turn | MIT | A dedicated security team's general-purpose kit |

**Three corrections**

- ⚠️ **promptfoo was acquired by OpenAI in March 2026.** The MIT-licensed open source continues, but this predates the guide's July 2026 publication, and a change of governing entity belongs in a tool-selection decision.
- ⚠️ **Inspect AI is an evaluation harness, not an attack tool.** Listing it alongside the other four invites a category error. It belongs as the skeleton of a red-team pipeline.
- ℹ️ **DeepTeam is an extension on top of DeepEval, not a standalone tool.** Adopt it and you are adopting DeepEval too.

As a selection heuristic: garak to audit a model, promptfoo to red-team an app, DeepTeam for broad scanning, Inspect AI for standardised evaluation, PyRIT when a dedicated team needs one kit.

### 2-4. What the labs actually learned

- **Microsoft, lessons from red-teaming 100 products** (2025, [arXiv 2501.07238](https://arxiv.org/abs/2501.07238)) — *"you don't need gradients to break AI"* (simple techniques still work), *"red teaming is not safety benchmarking"*, and *"AI security is never done"*. This is where the break-fix loop in the guide comes from.
- **Anthropic, challenges in red teaming** (2024) — four types (domain expert / automated / multimodal / open-ended), with the observation that **the absence of standardisation makes systems incomparable**. That is the background to the guide's emphasis on ISO alignment.
- **Google SAIF**, Fabian & Crisp (2023) — six attack types, plus the reminder that **getting traditional security controls right already reduces risk substantially**.
- **Microsoft**, Siva Kumar (2023) — ordinary well-meaning users also elicit harmful output, and the probabilistic nature of these systems means repeating across rounds. This is the source of the guide's benign red teaming and persona material.
- **OpenAI** (2024/2025) — four design decisions for external red teams (cohort composition, access level, guidance provided, degree of automated evaluation), with cohorts spanning 20 countries and 24 languages.
- **Japan AISI**, *Known Attacks* 2nd edition (2026) — a taxonomy of attack-to-impact relationships, referenced by the guide's threat table and impact analysis.
- **ISO/IEC AWI TS 42119-7** — **under development and not yet in force**, aligned with the ISO/IEC 29119 testing process.
- **National AI Strategy Committee CVD/VDP roadmap** (2026) — a legal pathway for good-faith researchers to report, get remediation, then disclose. A five-month pilot across 7 private and 8 public organisations, with **legislation targeted for 2027**.

The four lineages converge on the same four points: human and automated testing are complementary; testing belongs in the lifecycle as a break-fix loop; defence in depth needs both a security and a responsible-AI lens; and standardisation, third-party verification and eventual regulation are the direction of travel.

---

## 3. How this differs from traditional red teaming

Is it just a different domain? **Only partly.** The attacker mindset carries over; the nature of the target does not.

| Axis | Traditional cyber red teaming | AI red teaming |
|---|---|---|
| Determinism | Deterministic — a vulnerability is present or absent | Probabilistic — same input, different output |
| How vulnerabilities exist | Discrete defects in code or configuration | Distributed across learned behaviour (superposed) |
| Remediation | Patch and it is permanently gone | Recurs after fine-tuning or a prompt change; **no permanent patch** |
| Attack surface | Network, application, infrastructure | Data, weights, system prompt, learned behaviour |
| Objective | Security (confidentiality, integrity, availability) | Security *plus* safety, quality, bias |
| Assumed adversary | A malicious actor | Malicious actors **and well-meaning ordinary users** |

Three things have no counterpart in traditional penetration testing: **non-determinism**, **benign red teaming**, and the **extension of scope into safety and quality**. The overlap is still large, though — many AI attacks follow traditional patterns and existing controls remain effective. Traditional red teaming is a **subset of, and precondition for**, AI red teaming.

So the conclusion I settled on is not that this is a new domain, but that it is a **superset**: traditional red teaming plus probabilistic behaviour, safety scope and impermanent fixes.

### AI as tool versus AI as target

An easy distinction to blur:

1. **AI as the target** ← **this guide**. LLMs, RAG systems and agents are the object under test.
2. **AI as the method** — present inside the guide, but only as a means: LLM-as-attacker, LLM-as-judge.
3. **AI-augmented penetration testing of conventional systems** — outside the guide's scope.

The guide is therefore "red-team AI (target) using AI (tool)". The target is fixed; the tooling is partially adopted.

### "How many kinds of red teaming are there?"

There is no single number. Several **orthogonal axes** each classify it, and any real engagement picks one value per axis.

| Axis | Values | Count |
|---|---|---|
| ① Target domain | Traditional / cyber security / AI | 3 |
| ② Role of AI | Target / tool / AI-augmented pentest | 3 |
| ③ Perspective | Adversarial / benign | 2 |
| ④ Access level | Black / grey / white box | 3 |
| ⑤ Who performs it | In-house / external specialist | 2 |
| ⑥ Automation | Automated / expert deep-dive | 2 |
| ⑦ Cadence | Continuous / periodic / challenge | 3 |
| ⑧ Lifecycle stage | Development / deployment / monitoring | 3 |

Seven or eight axes, and the guide is built so that axes ③ through ⑧ are combined **simultaneously** to arrive at a strategy.

---

## 4. Reading this as someone who runs a SIEM

Red and blue point in opposite directions, but they are not separate fields — they are **two interlocking halves of the same loop**.

| | Blue team | Red team |
|---|---|---|
| Purpose | Detect, respond, recover | Penetrate, discover vulnerabilities |
| Typical activity | Running SIEM/SOAR, forensics, monitoring, IR | Pentesting, exploitation, red teaming |
| Output | Detection rules, playbooks, response history | Attack scenarios, vulnerability reports |

The attack scenarios, payloads and reproduction scripts a red team produces are the raw material for SIEM detection rules and SOAR playbooks. In the other direction, blue-team logs and detection blind spots are the evidence for the red team's next target. Joining the two is what purple teaming means.

And the guide itself crosses into blue-team territory. Chapter VI's "transition to continuous monitoring" is blue-team work:

- Operating guardrails = real-time input/output detection and blocking, an AI-side defensive line
- Scheduled automated checks in CI/CD = continuous monitoring
- Records management (log schema, integrity, access logs) = forensic and audit traceability

Which is where I landed: **for someone operating a SIEM, AI red teaming is not the opposing camp. It is the requirements supplier that defines what the SIEM has to detect and block.**

---

## 5. Five things I am keeping

1. **Automated attacks are already industrialised.** GCG → AmpleGCG → Joint-GCG → Subspace Rerouting. Defensive testing must assume a distribution of large-scale, transferable, representation-level attacks.
2. **Interpretability is dual-use.** The white-box root-cause tool is the attack blueprint. Defence and attack share one knowledge base.
3. **Impermanence.** AI vulnerabilities are not permanently removed by a patch. They recur through fine-tuning or prompt changes, which makes regression testing and continuous monitoring mandatory.
4. **Humans and automation are complementary.** Every lab converges here: automation for coverage, humans for domain and context judgement.
5. **The red team supplies the blue team's requirements.** Unless red-team output feeds back into detection rules and monitoring criteria, the exercise stays a one-off.

---

*This post summarises the guide and reports what I found tracing its 25 references back to arXiv abstracts, GitHub repositories, official blogs and policy documents. The three corrections to the tooling appendix came out of that process. The companion document, KISA's* AI Security Threat Response Manual, *is the subject of the next post.*
