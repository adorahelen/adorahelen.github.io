---
title: "Korea's AI Threat Response Manual — the Companion Volume, and Three Corrections to It"
date: 2026-07-10T11:00:00+09:00
tags:
  - ai-security
  - red-teaming
  - llm
  - governance
summary: "KISA's AI Security Threat Response Manual is the other half of the red teaming guide: the guide says how to test, the manual says what the threats are and how to diagnose and mitigate them. I traced its ~100 references into five clusters and found three things worth correcting — including a statutory misattribution that, once separated, explains the document's structure better than the original does."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/kisa-ai-threat-response-manual/)**

## TL;DR

- **These two documents are a matched set.** The [red teaming guide](/posts/kisa-ai-redteaming-guide/) covers how to run a red team; this manual is the threat catalogue with diagnosis and mitigation per threat code. The guide explicitly defers to the manual for classification criteria.
- **The manual adds an H axis the guide does not have** — threats from *frontier* models: cyber-attack uplift (H01) and loss of control through autonomy (H02). This is not something an operator controls; it is the ceiling of attacker capability you have to assume.
- **A statutory correction turned out to explain the document better than the document does.** The manual's references pair Article 2 (high-impact AI, ten domains) with Article 32 (safety obligations) as if they applied to the same targets. They do not — Article 32 applies to frontier models above 10²⁶ FLOP. Separate them and you get a clean explanation of *why the H axis exists as a separate category*.
- **Controlled deployment of dual-use cyber capability is the emerging 2026 norm.** Restricting the strongest offensive-capable models to verified defenders is now something several labs actually do, and the manual cites it as the mitigation for H01.

---

## 0. Why the two documents belong together

The red teaming guide's threat table contains a line pointing elsewhere for detail: *"refer to the AI Security Threat Response Manual for specific classification criteria."* This is that manual. They are a matched pair from the same KISA AI security red team.

| | Red teaming guide (2026-02) | Threat response manual (2026-01) |
|---|---|---|
| Role | Red team **operating process** (compose → prepare → execute → report) | Threat **catalogue, diagnosis, mitigation** |
| Reader | Red team practitioners and PMs | Security teams, CISOs, SOC, developers, domain owners |
| Threat codes | D01–03 / M01–08 / A01–04 / S01–04 | **The same, plus H01/H02** |
| Deliverables | Scenarios and report templates | 32 industry scenarios, diagnostic code, mitigation tables |

The guide lists threat codes as things to test. The manual attaches, per code, a definition, root cause, impact, assessment criteria (sound versus vulnerable), **diagnostic method with actual code**, mitigation, and real incidents — 59 pages of it in Annex 2. For practical diagnosis, the manual is the primary source and the guide is the process wrapper.

---

## 1. Framing the guide did not provide

### Safety versus security

Sourced to the MIT AI Risk Repository:

| | AI safety | AI security |
|---|---|---|
| Cause | Defects in the AI itself, or how humans use it | Malicious actors |
| Protects | What is **outside** the system — users, society, environment | What is **inside** — model, data, infrastructure |
| Intent | Unintentional | Mostly intentional |

### Intrinsic versus extrinsic threats

- **Intrinsic**: arise from the model with no external attacker — training data bias, hallucination, **alignment failure**, **probabilistic non-determinism**
- **Extrinsic**: the supply chain, tools, infrastructure, external data, the user boundary — plus actors using an LLM as a weapon: jailbreaks, model extraction, supply-chain attacks, frontier-model threats

This aligns with the three-way split in the International AI Safety Report 2026 (Bengio et al.) — malicious use, malfunction, systemic risk. Intrinsic maps roughly to malfunction; extrinsic to malicious use plus systemic.

### Six components of an LLM system

The unit of analysis is not a model but a **system**: ① model supply chain ② agent tooling (MCP, plugins) ③ operating infrastructure (inference engines, GPUs) ④ external data (vector DB, RAG) ⑤ user and operator boundary (identity) ⑥ frontier models (outside the system). The background is the agentisation of late 2025 and the rapid expansion of the MCP ecosystem in early 2026.

> ⚠️ **Correction 1**: the manual attributes this six-component model to Databricks DASF 2.0, but **DASF 2.0 is a 12-component, 4-stage model mapping 62 risks to 64 controls**. The six-component version is KISA's own condensation and should be read as such.

---

## 2. Threat classification: identical to the guide, plus an H axis

The D/M/A/S codes are exactly the guide's. D01 imbalanced data / D02 inaccurate data / D03 insufficient de-identification / M01 training data leakage / M02 vector DB and embedding leakage / M03 system prompt leakage / M04 model leakage / M05 hallucination / M06 jailbreak / M07 improper output handling / M08 model DoS / A01 poor tool design / A02 hijacking / A03 agent DoS / A04 memory poisoning / S01 data poisoning / S02 model poisoning / S03 vulnerable inference engine / S04 vulnerable extensions.

### Frontier model threats (H) — the manual's substantive addition

- **H01, uplift for advanced cyber attacks**: frontier models abused for malware authoring and attack automation, raising the capability floor for non-expert attackers. (Grounded in MITRE ATLAS AML.T0048.)
- **H02, loss of control through autonomy**: agent autonomy escaping its intended envelope. (OWASP LLM06, Excessive Agency.)

These are not things an operator controls directly. They belong in a threat model as **the ceiling of capability available to an attacker**, which is a different kind of entry from the rest of the catalogue.

---

## 3. The reference base, cluster by cluster

The manual's threats root into three tiers: international standards → regulation and government evaluation → frontier lab deployment controls.

### International standards and risk taxonomy

| Source | What it is | What it justifies in the manual |
|---|---|---|
| **International AI Safety Report 2026** (Bengio et al., arXiv 2602.21012) | Second edition, 30+ countries and 100+ contributors; three-way split of malicious use / malfunction / systemic risk | The top-level threat taxonomy |
| **NIST AI RMF 1.0** (AI 100-1) + **GenAI Profile** (AI 600-1) | Govern / Map / Measure / Manage, plus 12 generative-specific risks | The risk governance skeleton |
| **NIST AI 100-2 e2025** (Vassilev) | Adversarial ML attack and mitigation taxonomy with NISTAML codes | Standard codes for attack types |
| **MITRE ATLAS** | 14 tactics and 80+ techniques against AI, with AML.Txxxx codes | The basis for the Annex 1 mapping |
| **EU GPAI Code of Practice** (Jul 2025) | Voluntary code for AI Act general-purpose obligations | General-purpose AI compliance |

### Regulation and government evaluation

- **NIST IR 8596 Cyber AI Profile** (preliminary draft, Dec 2025): aligned to the CSF's six functions, with three focal areas — securing AI systems, using AI for defence, and **deterring AI-enabled attacks**. Effectively the CSF-shaped version of the manual's H01.
- **CAISI** (under NIST): voluntary pre-deployment frontier evaluation agreements between government and labs, 40+ evaluations conducted. A working model for evaluating frontier systems before and after release.

### Industry security frameworks

- **Databricks DASF 2.0**: 12 components, 4 stages, 62 risks to 64 controls — the source behind the manual's six-element model.
- **Google SAIF / SAIF 2.0 Secure Agents**: decomposes agents into four components with autonomy, tool permission, memory poisoning and multi-agent risks — the international counterpart to the manual's A01–A04 and H02 control mappings.

### Frontier deployment controls — the H axis in practice

The manual cites real cases as the basis for its H01 mitigation ("staged deployment, verified users only"):

- **Restricting dual-use cyber capability** — offensive-capable models (red teaming, vulnerability validation, binary analysis) deployed only to identity- and trust-verified defenders.
- **Withholding the strongest coding and agentic models** where misuse defences are not yet established, while using them internally for defensive work on critical software — identifying thousands of zero-days across major operating systems and browsers, with several large vendors participating.
- **"MCP by Design"** (CSA / OX Security, Apr 2026): a systemic RCE exposure in the MCP SDK's STDIO design, affecting 150 million downloads and 7,000+ servers across 10+ critical CVEs. This is the evidence base for the manual's S04 (vulnerable extensions) and the agent tooling ecosystem generally.

> Put together, this is a **2026 frontier governance layer**: risk taxonomy → standard codes → regulation and government evaluation → controlled deployment. The lab red-team lessons I traced in the [previous post](/posts/kisa-ai-redteaming-guide/) reappear here, but promoted into law, standards and deployment policy.

---

## 3b. The Korean regulatory stack behind the 32 industry scenarios

- The **AI Framework Act** (in force 22 Jan 2026), Article 2(4) on **high-impact AI**: ten domains where AI materially affects life, body or fundamental rights — energy, drinking water, healthcare, medical devices, nuclear, criminal investigation, hiring and credit assessment, transport, public services, education. The manual's 32 industry scenarios map onto these ten.
- ⚠️ **Correction 2**: the manual's references pair *"high-impact AI (Art. 2) and safety assurance obligations (Art. 32)"* in a single line, but **Article 32 applies to frontier models — cumulative compute at or above 10²⁶ FLOP** (threshold set by decree), not to high-impact AI. Two different legal triggers.
- ✅ And separating them explains the manual better than the original phrasing does: Article 2 (ten high-impact domains) is the basis for the **industry scenarios**, while Article 32 (frontier safety obligations) is the domestic legal basis for the **H01/H02 frontier threats**. Why the H axis exists as its own category is answered by statute.
- **Ministry guidance on high-impact determination** (Jan 2026): decided by domain of use × level of impact, targeting the same ten domains.
- **Ministry AI security guide** (Dec 2025): 113 lifecycle items organised by confidentiality, integrity and availability — the parent document above this manual.

**Privacy stack** (Personal Information Protection Commission): guidance on processing publicly available personal data (Jul 2024), synthetic data (Dec 2024), an AI privacy risk management model (Dec 2024), generative AI personal data processing (Aug 2025), and ten priority areas of daily life (Mar 2025). These are the reference points for the manual's D03 and M01 mitigations.

> ⚠️ **Correction 3**: the "AI Privacy Guide (Aug 2025)" the manual cites does not resolve to a standalone document — it is most likely an alternative name for the Aug 2025 generative AI personal data guidance. And "ten priority areas of daily life" appears as a policy plan rather than an independent guide.

---

## 4. The alignment techniques the manual leans on

The manual is explicit that alignment is *not* a solution by itself but one layer among input filtering and output validation. The primary sources behind that table:

| Technique | Paper | Contribution | Limitation |
|---|---|---|---|
| RLHF | Bai 2022 ([arXiv 2204.05862](https://arxiv.org/abs/2204.05862)) | Human preference → reward model → RL for helpful and harmless behaviour | Labelling cost, unstable RL, over-refusal |
| Constitutional AI / RLAIF | Bai 2022 ([2212.08073](https://arxiv.org/abs/2212.08073)) / Lee 2023 ([2309.00267](https://arxiv.org/abs/2309.00267)) | Principle-based self-critique plus AI feedback, reducing human labels | Depends on principle quality; AI errors propagate |
| DPO | Rafailov 2023 ([2305.18290](https://arxiv.org/abs/2305.18290)) | Optimise preference pairs directly, no reward model or RL | Offline data dependence, distribution shift |
| Rule-based rewards | Mu 2024 ([2411.01111](https://arxiv.org/abs/2411.01111)) | Fine-grained rules with an LLM grader as reward; precise control of over-refusal | Rule authoring and maintenance burden |
| Deliberative alignment | Guan 2024 ([2412.16339](https://arxiv.org/abs/2412.16339)) | Train on a safety specification, then recall and reason over it via chain of thought | Higher inference cost; depends on specification quality |
| Machine unlearning | Bourtoule 2021 (SISA) / Golatkar 2020 | Selectively remove the influence of specific data | Verifying complete removal is an open problem |

No single technique guarantees safety, and their failure modes are complementary rather than overlapping — which is the technical support for the manual's defence-in-depth position, and the same conclusion the [red teaming guide analysis](/posts/kisa-ai-redteaming-guide/) reached from the attack side.

---

## 5. Annex 1: attack papers ↔ threat codes ↔ frameworks

Annex 1 maps each threat code four ways — OWASP, NISTAML, ATLAS, and academic literature. The attack research I traced in the previous post lands here precisely.

| Threat | Representative papers | Frameworks |
|---|---|---|
| M06 jailbreak | **Zou 2023 (GCG)**, Wei 2023, Chao 2025 | LLM01 / NISTAML.018 / AML.T0054 |
| M04 model leakage | Tramèr 2016, **Carlini 2024b** (partial theft of a production LM) | LLM10:2023 / NISTAML.031 / AML.T0024.002 |
| M01 training data leakage | Carlini 2021 / 2019 / 2022 | LLM02 / NISTAML.032, .033 |
| M02 vector DB and embeddings | Song 2020, Morris 2023 (embedding inversion) | LLM08 / AML.T0057 |
| S01 data poisoning | Biggio 2012, **Gu 2017 (BadNets)**, Carlini 2024a | LLM03, 04 / AML.T0020 |
| S02 model poisoning | Kurita 2020, **Hubinger 2024 (Sleeper Agents)** | AML.T0018, T0058 |
| A02 hijacking | **Greshake 2023 (indirect prompt injection)**, Zhan 2024 (InjecAgent) | LLM01, 06 / NISTAML.015 / AML.T0051.001 |
| A04 memory poisoning | **Chen 2024 (AgentPoison)**, Dong 2026 | AML.T0080.001 |
| M08 / A03 DoS | **Shumailov 2021 (Sponge)**, Gao 2024, Dong 2025 (Engorgio) | LLM10 / AML.T0029 |
| M05 hallucination | Ji 2023, Maynez 2020, Huang 2025 | LLM09 / AML.T0062 |
| H01 cyber capability | **Fang 2024** (autonomous 1-day exploitation), Pa Pa 2023, Bhatt (CyberSecEval) | AML.T0048 |
| H02 loss of control | Bengio 2024, Chan 2023, Hendrycks 2023 | LLM06 |

The GCG line and the RAG poisoning work I went through earlier are cited directly under M06, A04 and the RAG entries — so **the two KISA documents and my own reading of the literature turn out to be one connected graph**. What the manual adds on top is operational: real CVEs (a LangChain RCE, an MCP client vulnerability, zero-click exfiltration in a major productivity assistant) and working diagnostic code (logit-bias model extraction, `picklescan`, `pip-audit`).

---

## 6. What I am keeping

1. **The two documents only work as a set.** The manual is the threat catalogue and the diagnostic source (Annex 2); the guide is the operating process.
2. **The H axis is explained by statute.** Article 2 (ten high-impact domains) gives you the industry scenarios; Article 32 (frontier safety obligations above 10²⁶ FLOP) gives you H01 and H02. The manual's citation blurs two different legal triggers; separating them makes the document's structure legible.
3. **Controlled deployment is the new norm.** Restricting dual-use cyber capability to verified defenders is the mitigation the manual points at for H01, and it is already practice. The industry-scale version of that threat — autonomous vulnerability discovery to weaponisation in hours — is where all 32 scenarios converge, and it is the automated-attack research from the previous post spread across SCADA, firmware and cryptographic libraries.
4. **Alignment is one layer.** None of the six techniques above is a standalone answer — the same conclusion the attack literature forces.
5. **The reference graph closes.** The attack papers both KISA documents share (Zou, Carlini, Greshake, Shumailov) are triangulated across OWASP, NISTAML and ATLAS codes. Notably, this manual is the only Korean-language material I know of that translates Carlini 2024b — partial extraction of a production model — into a practitioner checklist via logit-bias and logprobs behaviour.

---

## Appendix: the three corrections, collected

1. **Article 32 ≠ high-impact AI.** Article 32's safety assurance obligations apply to frontier (≥10²⁶ FLOP) systems. Article 2's high-impact classification covers ten application domains. Different legal triggers; worth separating when cited together.
2. **DASF 2.0 is a 12-component model.** The manual's six-element LLM system model is KISA's adaptation, not the original.
3. **A cited privacy document does not resolve.** The "AI Privacy Guide (Aug 2025)" is most likely an alternative name for the generative AI personal data guidance of the same month, and "ten priority areas of daily life" exists as a policy plan rather than a standalone guide.
