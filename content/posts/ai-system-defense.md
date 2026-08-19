---
title: "Guardrails Are a Probabilistic Mitigation, Not a Security Boundary"
date: 2026-07-13T11:00:00+09:00
tags:
  - ai-security
  - blue-team
  - llm
summary: "One classifier was bypassed 99.8% of the time by inserting spaces between letters. Adversarial queries transfer between models at roughly 82%. OWASP states outright that a fool-proof prevention for prompt injection likely does not exist. So the working defence is not a better filter — it is architectural trust separation, least privilege on tools, output handling, guardrails and logging, stacked."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/ai-system-defense/)**

## TL;DR

- **Guardrails get bypassed, repeatedly and cheaply.** One production classifier fell to inserting spaces between characters — 99.8% bypass. Invisible Unicode and adversarial suffixes defeat many commercial and open detectors. Adversarial queries transfer across models at about 82%.
- **The standards bodies say so themselves.** OWASP LLM01:2025 states that given the probabilistic nature of these systems, a fool-proof prevention **likely does not exist**. The UK NCSC calls prompt injection potentially permanently unsolved.
- **The load-bearing idea is the lethal trifecta**: access to private data + exposure to untrusted content + the ability to communicate externally. If all three sit in one agent, exfiltration is a matter of time — and the recommended defence is to **remove one of the three architecturally**, not to detect harder.
- **Supply chain is a separate axis.** Pickle deserialisation executes arbitrary code, and the scanners themselves have bypasses. Forcing safetensors and managing provenance beats scanning.

> Second of three blue-team axes. The attack side is in the [KISA guide](/posts/kisa-ai-redteaming-guide/), the [manual](/posts/kisa-ai-threat-response-manual/) and [multimodal agents](/posts/multimodal-agent-red-teaming/); the conventional discipline is [traditional blue teaming](/posts/traditional-blue-teaming/).

---

## 1. Guardrails: the inspection layers around the model

Three families with genuinely different structures — a programmable policy pipeline, a validator composition, and classifier models.

| Tool | From | Applied at | What it blocks |
|---|---|---|---|
| **NeMo Guardrails** | NVIDIA (open source) | **Five rail types**: input, output, dialogue, retrieval, execution | Input: jailbreak detection, injection filtering, sensitive data masking. Output: policy violations, sensitive data. Also inspects RAG chunks and tool calls |
| **Guardrails AI** | Guardrails AI (open source) | Input guard plus output guard | Composition of ~60 hub validators (PII via Presidio, toxicity, hallucination, structural checks). Strong at enforcing structured output |
| **Llama Guard** | Meta (Purple Llama) | **Input and output** | An LLM-based safety classifier judging against a risk taxonomy; version 3 also detects responses that assist cyber attacks |
| **Prompt Guard** | Meta | **Input only** | A small BERT-class classifier catching injection and jailbreaks before the main model sees them, aimed at indirect injection |
| **Code Shield** | Meta | **Output only** | Static analysis flagging vulnerabilities in generated code at inference time, to limit code-interpreter abuse |

The distinction that matters operationally: retrieval and execution rails inspect **RAG chunks and tool calls**, not just the user's message. Most deployments only guard the user's message, which leaves the indirect injection path — the one that actually gets used — unguarded.

## 2. LLM firewalls and runtime protection

The common design is an API proxy or gateway doing injection detection on input, inspection on output, and PII/secret filtering. The market consolidated into the large vendors during 2025 — a guard product acquired by a network security vendor, a runtime-plus-posture-plus-red-teaming bundle at another, and a third combining shadow-AI discovery with pre-deployment automated red teaming and runtime guardrails.

> ⚠️ Vendor detection figures — "98%-plus at under 50 ms" and similar — have **no independent verification**. Treat them as vendor claims.

## 3. Prompt injection research: trying to restore a trust boundary

The root problem is that an LLM cannot architecturally distinguish **instructions** from **data**. The research lineage is a series of attempts to impose that boundary from outside.

| Approach | Idea | Limit |
|---|---|---|
| **CaMeL** (Google DeepMind, 2025, [arXiv 2503.18813](https://arxiv.org/abs/2503.18813)) | A hardened dual-LLM design: a privileged model turns the trusted request into a plan as code, a quarantined model handles untrusted data with no tool access, and a custom interpreter tracks data provenance as **capabilities**, protecting both control flow and data flow. No model modification needed | Defends **67% of attacks on AgentDojo — not all**; ~2.7× tokens; you have to author the policies |
| **Dual-LLM pattern** (Willison, 2023) | The tool-privileged model never sees untrusted content directly | Protects control flow only; tool arguments (data flow) can still be poisoned |
| **Spotlighting** (Microsoft, 2024, [arXiv 2403.14720](https://arxiv.org/abs/2403.14720)) | Mark the provenance of untrusted input: delimiting, **datamarking** (special characters between tokens), **encoding** (base64) | Prompt-level, so bypasses persist; encoding degrades smaller models. The authors describe it as probabilistic mitigation themselves |
| **StruQ** (UC Berkeley, USENIX Security 2025) | A secure front end separates prompt and data into distinct token channels, with structured tuning so the model ignores instructions in the data channel | Requires retraining; incomplete against adaptive (GCG-style) attacks. **SecAlign** ([2410.05451](https://arxiv.org/abs/2410.05451)) improves it via preference optimisation |

**The consensus that no complete defence exists is explicit, not implied.** OWASP LLM01:2025 says a fool-proof prevention likely does not exist given the probabilistic nature of generative AI. The UK NCSC warns that prompt injection may be permanently unsolved.

Which is why Willison's **lethal trifecta** (June 2025) has become the de facto practitioner threat model: if **access to private data**, **exposure to untrusted content** and **the ability to communicate externally** coexist in one agent, exfiltration is inevitable. The recommended response is not better detection but **removing one of the three architecturally**. That is a design constraint, and it is testable — unlike "we have a guardrail".

## 4. AI security posture and the ML supply chain

**Established fact**: pickle serialisation permits arbitrary computation during deserialisation, and many malicious models found on model hubs used pickle exploits. The scanners themselves are bypassable — picklescan had four such bypasses — so **scanning is necessary and not sufficient**.

| Control | Detail |
|---|---|
| **Model scanning** | ModelScan (open source, detects serialisation attacks across formats, CI/CD integrable) and commercial equivalents |
| **Safe serialisation** | **safetensors** stores weight values only with no code execution path (implemented in Rust); a 2023 Trail of Bits audit found no critical flaws, and it became the hub default. **Prohibit pickle formats and mandate safetensors** is the standard control |
| **Model signing** | OpenSSF/Sigstore model-transparency — cryptographic signing and provenance attestation for artefacts |
| **AI-BOM** | CycloneDX supports ML-BOM from v1.6, adopted by MITRE ATLAS as AML.M0023 |

Data poisoning deserves separate emphasis: it is hard to detect after the fact, which makes **provenance management the primary control** rather than inspection.

## 5. How the standards map defences

**MITRE ATLAS mitigations** (AML.M0000–M0025) map directly onto this topic:

| Code | Mitigation | Attack addressed |
|---|---|---|
| AML.M0007 / M0025 | Sanitise training data / dataset provenance | Data poisoning |
| AML.M0015 | Adversarial input detection | Injection, evasion |
| AML.M0020 / M0021 / M0022 | GenAI guardrails / guidelines / model alignment | Jailbreak, injection |
| AML.M0013 / M0014 / M0023 | Code signing / verify ML artefacts / AI-BOM | Model supply chain |
| AML.M0004 / M0005 / M0019 | Query limiting, access control | Model extraction, resource abuse |

**OWASP LLM Top 10 2025 mitigations** worth internalising: LLM01 (least privilege, output validation, human approval, trust boundary separation), LLM02 (PII filtering), LLM03/04 (scanning, SBOM, signing, provenance), LLM05 (**treat model output as untrusted input** and encode it), LLM06 (least-privilege tools, human in the loop), LLM08 (RAG access control), LLM10 (rate limiting).

**NIST AI 100-2 E2025** is notable for a discipline the others lack: it states **the limits of each mitigation alongside the mitigation**, making explicit that none is complete.

## 6. The limits of guardrails, with the evidence

| Limit | Evidence |
|---|---|
| Classifier bypass | An early release of one prompt classifier was bypassed **99.8% of the time by inserting spaces between characters** (found externally, acknowledged and fixed by the vendor) |
| Systematic bypass | Invisible Unicode and adversarial suffixes defeat many commercial and open detectors ([arXiv 2504.11168](https://arxiv.org/abs/2504.11168)) |
| Transfer attacks | Adversarial queries built against one model transfer to guard models at roughly **82% success**. The recommendation is **self-evaluation on held-out prompts** ([SoK, arXiv 2506.10597](https://arxiv.org/abs/2506.10597)) |
| Over-refusal | **XSTest** (250 safe prompts) and **OR-Bench** (80,000) show safety training strength correlates with over-blocking — the safety/utility tension is structural, not a tuning error |
| The guard is itself attack surface | Some guard models can be induced to generate harmful content instead of classifying it |

That last row is the one people miss. Adding a classifier adds a component that processes untrusted input, which is a new surface, not purely a reduction.

---

## What actually works

1. **Never trust a single guardrail.** Bypasses are demonstrated repeatedly. A filter is one layer among several.
2. **Stack defence in depth**: ① architectural trust separation (CaMeL-style, or remove one leg of the lethal trifecta) ② **least privilege** on tools ③ output handling controls (treat model output as untrusted input) ④ guardrails for input and output classification ⑤ **logging and detection** (ATLAS AML.M0024).
3. **Treat the supply chain as its own axis**: mandate safetensors, scan models, sign artefacts, maintain an AI-BOM. For data poisoning, provenance is the primary control because detection after the fact is not reliable.
4. **Evaluate on held-out adversarial prompts, not vendor benchmarks.** The automated red-team tooling from [red teaming in practice](/posts/ai-red-teaming-practice/) is what you point at this.
5. **Where it meets the SOC**: collect guardrail verdicts, refusals and injection detections as logs, and tag them with OWASP LLM and MITRE ATLAS IDs. Then the same coverage-and-gap workflow you already run with [ATT&CK-tagged detection engineering](/posts/traditional-blue-teaming/) applies to the AI system — same discipline, new technique IDs.

> **The three blue-team axes**: ① conventional targets ([traditional blue teaming](/posts/traditional-blue-teaming/)) ② **defending AI systems (this post)** ③ [defending with AI](/posts/ai-augmented-defense-ai-soc/). This post is the exact defensive counterpart of red team axis ① — AI as the target.
