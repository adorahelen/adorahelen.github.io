---
title: "AI Red Teaming as a Practice: Mature Tools, a Consolidating Market, and Regulation That Only Bites at the Frontier"
date: 2026-07-10T15:00:00+09:00
tags:
  - ai-security
  - red-teaming
  - governance
summary: "The tooling is done — PyRIT, garak, promptfoo, Giskard, deepteam all ship. The bottleneck is threat modelling: deciding what counts as a threat. Also three corrections worth having: the EU AI Act says 'adversarial testing' not 'red teaming', Korea's law never uses the term at all, and the US mandate people still cite was revoked in January 2025."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/ai-red-teaming-practice/)**

## TL;DR

- **The tools are mature; the bottleneck is upstream.** Five capable open frameworks exist. What limits a programme is threat modelling — deciding what counts as a threat — which is why Microsoft's own white paper leads with a threat model ontology rather than a tool list.
- **Four organisations converged on the same method**: start with threat modelling, treat security and responsible-AI harms together, divide labour between humans and automation, and wire the results into deployment gates. Not a one-off test — a gate.
- **Regulation applies to the frontier tier only.** EU AI Act Article 55 and Korea's Article 32 are both compute-threshold-gated. The US reporting mandate under EO 14110 was **revoked in January 2025**, leaving voluntary NIST guidance.
- **Terminology matters legally.** The EU text says "adversarial testing", not "red teaming". Korea's statute says "safety assurance measures". What is mandated is the outcome, not the technique.

> The preceding posts covered *what* gets attacked. This one covers *how the practice actually runs*: tooling, organisational method, regulatory duty, public community. Companion: [multimodal and agentic surface](/posts/multimodal-agent-red-teaming/).

---

## 1. The open-source tooling

| Tool | From | Type | Core structure | Coverage |
|---|---|---|---|---|
| **PyRIT** | Microsoft | Framework | Six modules: orchestrator, target, converter, scorer, memory, dataset | Multi-turn jailbreak, harmful content, prompt transformation attacks |
| **garak** | NVIDIA | Scanner | Probe, detector, generator, evaluator, harness — all plugins | Injection, jailbreak, data leakage, toxicity, hallucination |
| **promptfoo** | promptfoo → **acquired by OpenAI (March 2026)** | Eval plus red team (DAST) | Prompt/model comparison with red team plugins | Injection, jailbreak, data leak, tool misuse |
| **Giskard** | Giskard-AI | Scanner (SDK and hub) | LLM detectors, heuristic and LLM-assisted | Hallucination, injection, bias, sensitive data |
| **deepteam** | Confident AI (DeepEval family) | Framework | Vulnerabilities × attacks (linear, sequential, tree jailbreak) | 40+ vulnerability classes |

**How PyRIT automates**: an orchestrator loops `converter` (transform the prompt) → `target` (the system under test) → `scorer` (judge the response), adapting follow-up prompts until it succeeds. It began as one-off internal scripts in Microsoft's AI Red Team in 2022 and was **released as an open framework in February 2024** ([arXiv 2410.02828](https://arxiv.org/abs/2410.02828)).

**garak** is plugins all the way down, supporting static probing (replaying known exploits), dynamic probing (emergent weaknesses) and adaptive probing (refining after failure), with JSONL and HTML reporting ([arXiv 2406.11036](https://arxiv.org/abs/2406.11036)).

Academic jailbreak algorithms — PAIR, TAP, GCG — have been absorbed as probes in these tools, which connects directly to the automated jailbreak lineage in the [KISA guide analysis](/posts/kisa-ai-redteaming-guide/).

> ⚠️ The promptfoo acquisition is confirmed by OpenAI's own announcement (March 2026), with the open source continuing. Developer counts and valuations circulating alongside it are vendor and press claims.

## 2. The commercial market consolidated fast

| Vendor | Positioning | Status |
|---|---|---|
| **Robust Intelligence** | AI firewall plus red teaming | **Acquired by Cisco (2024)** → Cisco AI Defense |
| **Protect AI** | MLSecOps (model scanning, discovery) | **Acquired by Palo Alto**, ~$700M agreed April 2025, closed July 2025, folded into Prisma AIRS |
| **Lakera** | AI-native runtime security plus adversarial testing | **Acquired by Check Point**, $300M announced September 2025, closed November 2025 |
| **HiddenLayer** | AI detection and response | Independent |
| **Adversar AI** | Continuous red teaming, agents and MCP | Independent |

> **What this means if you run a SIEM**: three of the largest network security vendors bought AI red team startups within eighteen months. AI security testing is moving **from an independent niche into a component of the integrated security stack** — which means it will arrive in your existing platform as a bundled capability, whether or not you go looking for it.

## 3. How the frontier labs organise it

**Microsoft AI Red Team** was founded in 2018, covering both security and **responsible AI**. From *"Lessons From Red Teaming 100 Generative AI Products"* ([arXiv 2501.07238](https://arxiv.org/abs/2501.07238), January 2025), the lessons that matter most to a defender:

- **"You don't need gradients to break AI."** Simple prompt manipulation outperforms sophisticated optimisation attacks in practice. If your threat model only covers the sophisticated version, it is the wrong shape.
- **"Red teaming is not safety benchmarking."** Benchmarks measure the known; red teaming discovers unknown failures.
- **"Automation broadens coverage, but the human element is essential."**
- Their threat model ontology has five elements: system under test, actor, TTPs (**mapped to ATT&CK/ATLAS where possible**), underlying weaknesses, downstream impacts.

**Google** places red teaming as the practice behind "adapt controls / continuous testing" in its Secure AI Framework, with a six-part attack taxonomy: prompt attacks, training data extraction, backdooring, adversarial examples, data poisoning, exfiltration. It states explicitly that traditional red teaming is a good starting point but AI complexity demands specialist expertise — which matches the "extension and specialisation" argument in [traditional red teaming](/posts/traditional-red-teaming/).

**OpenAI** runs an external red teaming network, used from a 2022 image-model deployment onward and with 100-plus external red teamers for a major 2023 model launch ([arXiv 2503.16431](https://arxiv.org/abs/2503.16431)). The method: prioritise domains via threat modelling, anchor each area with hypotheses about risk, target and source, then feed results into risk assessment **and into automated evaluations**. They published automated red teaming research alongside it, which is the human-plus-automation split made concrete.

**Anthropic**'s frontier red team focuses on three domains — CBRN, cyber and autonomy — designing evaluations with domain experts. The structural point is that results are wired into **capability thresholds in a responsible scaling policy**: reaching a defined risk capability constrains deployment as a matter of policy. Their own writing names **the absence of standardisation** as the field's shared difficulty.

> **The common pattern**: all four (a) start from threat modelling, (b) treat security and safety harms together, (c) run humans and automation in parallel, (d) connect results to deployment decisions and policy. AI red teaming has been institutionalised as **a continuous process attached to a release gate**, not a test you pass once.

## 4. What regulation actually requires

| Jurisdiction | Provision | Requirement | Force |
|---|---|---|---|
| **EU** | AI Act **Art. 55(1)(a)** (threshold in **Art. 51**) | General-purpose AI with systemic risk (presumed above **10²⁵ cumulative training FLOP**, or Commission-designated) must perform and document model evaluation including **adversarial testing** to standardised protocols. Recital 114 mentions internal and independent external testing | Legal obligation, phasing in |
| **US** | EO 14110 §3(d), §4.2(a)(i)(C) | Reporting of **AI red-team results** for dual-use foundation models to Commerce under the Defense Production Act | **Revoked** — EO 14148 (20 Jan 2025) rescinded it; the duty is not in force |
| **US (standards)** | **NIST AI 600-1** GenAI Profile (Jul 2024), Appendix A.1.5 | Defines red teaming as pre-deployment risk identification, with **four types** (general public, expert, combination, human-AI) and measurement actions | **Voluntary guidance**, unaffected by the EO revocation |
| **Korea** | AI Framework Act **Art. 32** (large-scale), **Arts. 33–35** (high-impact) | Safety assurance for high-compute AI; risk management, human oversight and documentation for high-impact AI | Legal obligation, in force 22 Jan 2026 |

Three corrections worth carrying:

> **Terminology.** The EU AI Act text and recitals say **"adversarial testing", not "red teaming"**. Korea's statute contains no equivalent term either, framing the duty as "safety assurance measures" and "risk management". So what the law compels is an outcome — adversarial testing and risk assessment — and red teaming is one way to discharge it. That distinction matters when someone asks whether a specific methodology is mandatory.
>
> **Tier.** EU Article 55 and Korea's Article 32 apply **only to the frontier / systemic-risk tier**, gated on compute. Red teaming is not legally mandated for ordinary AI systems. This is the same structure as the correction in my [manual analysis](/posts/kisa-ai-threat-response-manual/) — Article 32 covers frontier, not high-impact — though the thresholds differ between jurisdictions, and the Korean figure may sit in a decree rather than the statute (unverified).
>
> **The US.** The mandatory red-team reporting under EO 14110 lapsed with the January 2025 revocation. Saying "the US has legislated red teaming" is inaccurate as of 2026; only the NIST guidance remains, and it is advisory.

## 5. Humans versus automation

**Automation** brings coverage, reproducibility, low cost and regression testing on every release — PyRIT, garak and deepteam inject prompts in bulk and in parallel.

**Humans** bring discovery of genuinely novel attacks, judgement on context- and culture-dependent harms, and the framing of *what counts as a problem in the first place*. Microsoft's fifth lesson exists to nail this down.

**The consensus is division of labour, not competition**: automation sweeps the known attack surface broadly (quantity), humans judge unknown and contextual risk (quality). Keeping a human verification loop on the final adjudication is the same principle as accounting for false positives in [offensive AI](/posts/ai-augmented-red-teaming/) — the tooling is a force multiplier on both sides and unreliable in the same way on both sides.

## 6. The public community: DEF CON's Generative Red Team

**GRT at DEF CON 31 (August 2023)** was the largest public LLM red teaming exercise held: 2,200-plus participants, eight vendor models, a hosted platform, 164,000 messages.

On the White House involvement, which is frequently overstated: OSTP provided **technical advice on challenge design, not funding** (per the office's own August 2023 statement), aligned to the Blueprint for an AI Bill of Rights and the NIST AI RMF. An independent transparency report analysed results across the eight models in 2024.

**GRT2 at DEF CON 32 (August 2024)** was the qualitative shift:

- From individual findings to **statistical flaw reports** — demonstrating bias tendencies with representative datasets — and a **modified CVE process** for reporting "flaws" against model card claims, including bias against protected groups, rather than only "vulnerabilities".
- Run against an open model with the vendor panel present, structured through UK AISI's Inspect AI framework: 495 participants, around 200 flaw reports, $7,400 in bounties.

> **Why GRT2 matters more than GRT1**: the "flaw" concept — statistical bias, not just exploitable vulnerability — shows AI red teaming outgrowing the security field's vulnerability frame. **Responsible-AI harms do not fit in a CVE**, and a reporting process that cannot represent them will systematically under-report them.

---

## Where this leaves a practitioner

1. **The tools are done.** PyRIT, garak, promptfoo, Giskard, deepteam. The bottleneck is **threat modelling** — what you decide to call a threat — which is why the best white paper in the field leads with an ontology rather than a tool.
2. **Organisational method has converged**: threat modelling → security and RAI harms together → humans and automation in parallel → wired into deployment gates and policy. Continuous, not episodic.
3. **Regulation binds the frontier tier only**, and it mandates adversarial testing as an outcome rather than red teaming as a technique. Check the tier and the wording before claiming a duty applies.
4. **The market is consolidating**, so expect AI red teaming and runtime defence to arrive bundled into the security stack you already own.
5. **Where it meets the SOC**: tag scanner output from garak or PyRIT with OWASP LLM Top 10 and MITRE ATLAS IDs, and the same coverage-and-gap workflow as [ATT&CK-tagged detection engineering](/posts/traditional-red-teaming/) applies to AI systems. The discipline transfers; only the technique IDs change.

> **The map**: ① AI as the target ([guide](/posts/kisa-ai-redteaming-guide/), [manual](/posts/kisa-ai-threat-response-manual/)) ② AI as the weapon ([offensive AI](/posts/ai-augmented-red-teaming/)) ③ conventional targets ([traditional red teaming](/posts/traditional-red-teaming/)) — and this post is the practice and governance layer that cuts across all three.
