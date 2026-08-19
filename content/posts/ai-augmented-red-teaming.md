---
title: "Offensive AI in 2026: Real Capability, Asymmetric Skills, and the Bottleneck That Did Not Move"
date: 2026-07-10T13:00:00+09:00
tags:
  - ai-security
  - red-teaming
  - llm
summary: "Autonomous offensive AI passed proof-of-concept: 18 real zero-days from the DARPA AIxCC winner, a SQLite zero-day from Big Sleep, an autonomous system topping a bug bounty leaderboard, and an agent beating 9 of 10 human testers on a university network. But the capability is lopsided — high false positive rates, weak on GUI work — and the bottleneck is still discovering unknown vulnerabilities, not writing exploits."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/ai-augmented-red-teaming/)**

## TL;DR

- **The capability is real and measured.** Not vendor claims: a DARPA competition winner found 18 real zero-days and auto-patched 68% of synthetic ones; a research agent found a genuine SQLite zero-day; an autonomous system reached the top of a bug bounty leaderboard; an agent placed second overall against ten human testers on an ~8,000-host network.
- **It is lopsided.** Systematic enumeration, parallelism and cost are overwhelming — $18/hour against $60 for an expert. False positive rates are high, GUI tasks are weak, and one agent missed an RCE that 80% of humans found.
- **The bottleneck did not move.** Given a CVE description, one 2024 study got 87% exploitation of real one-day vulnerabilities. Remove the description and it collapses. **Finding the unknown vulnerability is still the hard part**, and exploit writing is the part that got automated.
- **Controlled deployment is the labs' answer**, and the fact that they bother is itself evidence the capability is real.

> Second of three axes. Not AI as the target ([guide](/posts/kisa-ai-redteaming-guide/), [manual](/posts/kisa-ai-threat-response-manual/)) but **AI as the weapon against conventional targets**. The conventional discipline is in [traditional red teaming](/posts/traditional-red-teaming/). Research and governance level — no exploit code.

---

## 1. The research lineage: autonomous exploitation

Three papers from Kang's group at UIUC are the academic reference points.

| Paper | arXiv | Result |
|---|---|---|
| LLM Agents can Autonomously **Hack Websites** (2024a) | [2402.06664](https://arxiv.org/abs/2402.06664) | GPT-4 performed blind SQL injection and similar without prior knowledge of the vulnerability. Open models failed entirely — tool use and long context were frontier-only capabilities |
| LLM Agents can Autonomously **Exploit One-day Vulnerabilities** (2024b) | [2404.08144](https://arxiv.org/abs/2404.08144) | Given the CVE description, GPT-4 exploited **87% of 15 real one-day vulnerabilities**; every other model scored **0%**. Remove the description and performance collapses — **the bottleneck is discovery** |
| **Teams** of LLM Agents can Exploit **Zero-Day** Vulnerabilities (2024c) | [2406.01637](https://arxiv.org/abs/2406.01637) | **HPTSA** — a planner and manager orchestrating six specialised sub-agents (XSS, SQLi, CSRF, SSTI and others). Team exploitation of zero-day-class web vulnerabilities with no prior knowledge, outperforming the single-agent setup |

Terminology, since it gets muddled: **one-day** means publicly disclosed but unpatched; **zero-day** means undisclosed. The methodological arc is single agent → team orchestration.

**Where the capability stands, from independent evaluation:**

- **Government pre-deployment evaluations** put frontier models at *apprentice* level (under a year of experience) on cyber tasks in 2023 and at *expert* level (ten years plus) in 2025.
- **ARTEMIS** ([arXiv 2512.09882](https://arxiv.org/abs/2512.09882), December 2025) ran an agent against ten human testers on a university network of ~8,000 hosts. The agent placed **second overall, beating 9 of the 10 humans**, with nine valid findings. But its false positive rate was high, it was weak on GUI-driven tasks, and it **missed an RCE that 80% of the humans found**. Cost: $18 per hour against $60 for a human expert.

That last comparison is the operationally important one. The agent is not better than a good human tester. It is adequate, tireless and a third of the price, which changes the volume of attacks rather than their ceiling.

## 2. Autonomous penetration testing systems

**Open source and research**

- **PentestGPT** (NTU) — started as a conversational assistant, developed into an autonomous agent
- **HackingBuddyGPT** (IPA Lab) — a deliberately minimal implementation in roughly 50 lines; the associated paper is *LLMs as Hackers: Autonomous Linux Privilege Escalation* ([2310.11409](https://arxiv.org/abs/2310.11409))
- **Nebula** executes commands directly where PentestGPT only proposes them; **PentAGI** runs multiple sub-agents in a Docker sandbox

**Commercial**

- **XBOW** reached **first place on the US HackerOne leaderboard** between April and June 2025 — the first autonomous system to do so — with roughly 1,060 submissions, and raised a $75M Series B.
  > ⚠️ Read that carefully. "First place" is by **reputation score**, and a substantial share of the submissions sat in triaged-but-unresolved state on vulnerability disclosure programmes. It is not evidence of comprehensively surpassing human hackers.

**Vulnerability discovery, including defensively motivated work**

- **Google's Project Naptime → Big Sleep** (Project Zero with DeepMind, October 2024): equipped with a code browser, debugger and sandbox, it found a **real zero-day in SQLite** before release, with no user impact. Google described it as the first public case of AI finding an unknown memory-safety flaw in real-world software — while itself noting the work is highly experimental and that a target-specific fuzzer is at least as effective today.
- **DARPA AIxCC** (DEF CON 33, August 2025): the winning system, **ATLANTIS** from Team Atlanta (Georgia Tech with Samsung, KAIST and POSTECH), took the $4M prize. Multi-agent reinforcement learning combined with LLMs and symbolic methods to detect, exploit **and patch** autonomously. Results: 54 synthetic vulnerabilities found with **68% automatically patched**, plus **18 real zero-days** (6 in C, 12 in Java). All seven teams open-sourced their systems. Because the objective includes patching, this is defensive work — a real distinction from a pure attack tool.

## 3. Which sub-tasks the models are actually good at

**Reconnaissance** is where the strength is: systematic enumeration and parallel exploration, demonstrated by ARTEMIS. The weaknesses are false positive filtering and anything GUI-driven.

**Phishing, malware and exploit assistance** — the measurement tooling matters more than the anecdotes here:

- **CyberSecEval 1–3** (Meta Purple Llama, Bhatt et al., 2023 onward) measures both insecure code generation and compliance with requests to assist cyber attacks. Version 2 added prompt injection and a **false refusal rate**; version 3 extended to autonomous attack and spear-phishing evaluation. The false refusal metric is the one defenders should care about, because over-refusal on legitimate security work is a real cost.
- **Pa Pa et al. (2023, CSET)** produced seven malware variants and two tools using commodity assistants, generating roughly 400 lines of functional malware within about 90 minutes despite safeguards — and found that auto-generated prompts from agent frameworks tended to bypass those safeguards more readily than human-written ones.

## 4. Benchmarks for offensive capability

- **Cybench** — 40 professional-grade CTF tasks, purely offensive framing.
- **CyberSecEval 1–3** — the longest-running LLM cyber benchmark suite.
- **BountyBench** (Stanford, [2505.15216](https://arxiv.org/abs/2505.15216)) — 25 real-world systems and 40 bounties, structured as **detect / exploit / patch** to measure the full discovery-to-repair lifecycle. It scores **offence and defence in balance** and converts impact into dollars, which makes it the most decision-useful of the three.

## 5. Dual-use governance: controlled deployment

- **Restricted access programmes** pair a cyber-capable frontier model with identity verification: classifier refusals relaxed for verified defenders, while credential theft, malware distribution and unauthorised third-party exploitation stay blocked. The eligible population is government, critical infrastructure, security vendors, cloud providers and financial institutions.
- **Withheld frontier models** used internally for defensive discovery — one lab reports over ten thousand high-severity vulnerabilities found across major operating systems and browsers, with several large vendors participating, while keeping the model itself unreleased because misuse defences are not established.
  > ⚠️ Those figures and any "outperforms all but the most skilled humans" phrasing come from **the lab's own publications**. Treat them as vendor claims pending independent evaluation.
- **Government pre-deployment evaluation** now covers five major labs, including models that were never publicly released. The apprentice-to-expert movement above comes from this programme.

The governance response is itself an argument. Labs restricting their own strongest capability, and governments building pre-release evaluation, are costly signals that the capability is not hypothetical.

## 6. The defensive counterpart, briefly

Vendor SOC agents shipped in 2025 — phishing triage and alert triage agents that classify autonomously and show their reasoning, and agent suites covering exposure prioritisation, malware analysis, hunting and correlation rule generation. The direction is reducing manual SOC labour and automating triage, while keeping a human verification loop precisely because these systems are false-positive prone. Covered separately in [defending with AI](/posts/ai-augmented-defense-ai-soc/).

---

## Where this leaves a defender

1. **The capability is measured, not asserted.** AIxCC's 18 real zero-days, the SQLite finding, the bounty leaderboard placement, ARTEMIS beating 9 of 10 humans — all independently visible. The two-year apprentice-to-expert movement in government evaluation supports it.
2. **But it is asymmetric.** Enumeration, parallelism and cost are overwhelming; false positives are high; GUI and creative judgement are weak; and discovery of unknown vulnerabilities remains the bottleneck.
3. **The labs' controlled deployment is evidence.** Restricting your own product is expensive, and nobody does it for a capability that does not exist.
4. **What it changes for a SOC**: automated, cheap, high-volume reconnaissance and initial access is now the normal case. That has a telemetry signature — fast enumeration, many vectors attempted in parallel, non-human timing — and those characteristics belong in detection logic. Respond symmetrically with AI-assisted triage, while keeping human verification in the loop for the same false-positive reason that limits the attackers.

> **The three axes together**: ① AI as the target (the [guide](/posts/kisa-ai-redteaming-guide/) and [manual](/posts/kisa-ai-threat-response-manual/)), ② AI as the weapon (this post), ③ conventional targets ([traditional red teaming](/posts/traditional-red-teaming/)). The manual's H01 threat — uplift for advanced cyber attacks — is exactly this axis seen from the defender's side. The manual asks what to do about it; this post asks how real it is.
