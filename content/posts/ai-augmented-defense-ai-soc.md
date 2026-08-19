---
title: "The AI SOC in 2026, and the Asymmetry That Favours Defenders"
date: 2026-07-13T12:00:00+09:00
tags:
  - ai-security
  - blue-team
  - siem
  - soar
summary: "Triage automation shipped as product in 2025, not as demo. The interesting finding is not the vendor accuracy claims but a benchmark result: current coding agents score far higher at patching than at exploiting — 90% against 32.5% on one measure. Where offensive AI is asymmetric, some of that asymmetry runs in the defender's favour."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/ai-augmented-defense-ai-soc/)**

## TL;DR

- **This shipped.** Alert and phishing triage agents, exposure prioritisation, malware analysis, hunting and **correlation rule generation** went from demo to product across the major security vendors during 2025.
- **The asymmetry can favour defence.** On BountyBench, current coding agents scored **90% on patching against 32.5% on exploiting**. And the same research programme that found a SQLite zero-day used it to close the hole before anyone exploited it.
- **The limits are identical to offensive AI's** — confident false positives, no business context, automation bias. Which is why the consensus lands in the same place: **keep the human in the loop**.
- **Every impressive number in this space is a vendor claim** measured in a controlled environment. I have marked them as such throughout, because the difference between "98% accurate" and "98% accurate on our evaluation set" is the whole question.

> Third of three blue-team axes, and the exact counterpart to [offensive AI](/posts/ai-augmented-red-teaming/) — using AI as a defensive tool rather than a weapon. Conventional defence is [traditional blue teaming](/posts/traditional-blue-teaming/); defending the AI system itself is [AI system defence](/posts/ai-system-defense/).

---

## 1. What actually shipped

**Microsoft Security Copilot agents** (announced March 2025 — six first-party, five partner):

| Agent | Product | What it automates |
|---|---|---|
| Phishing triage → extended to **alert triage** | Defender | Triage of reported phishing and alerts, automatic false positive dismissal |
| Alert triage | Purview | DLP and insider risk alert classification |
| Conditional access optimisation | Entra | Finding gaps in conditional access policy |
| Vulnerability remediation | Intune | Vulnerability prioritisation and patching |
| Threat intelligence briefing | Security Copilot | Organisation-specific intelligence curation |

> Vendor claim: phishing triage clears 95%-plus of reported false positives automatically, measured in a controlled evaluation.

**CrowdStrike Charlotte AI** shipped Detection Triage (February 2025) with what it calls autonomy inside customer-defined boundaries, then an **agentic security workforce of seven** in September 2025: exposure prioritisation, malware analysis, hunting, data transformation, search analysis, **correlation rule generation** and workflow generation — with custom agents authored in natural language. RBAC, audit logs and explainability are positioned as the human-in-the-loop guardrails.

> Vendor claim: 98%-plus triage accuracy, 40-plus hours saved per week.

**Google** released Sec-Gemini v1 (April 2025, experimental), combining a frontier model with threat intelligence, open vulnerability data and incident response expertise for root cause and threat analysis, and integrated natural-language investigation into its SecOps platform. Note that **Big Sleep**, which appears in my [offensive AI notes](/posts/ai-augmented-red-teaming/) as a capability demonstration, is **fundamentally defensive**: it found vulnerabilities before attackers did, including a SQLite CVE where exploitation was reportedly imminent, plus around twenty new findings in open source.

> ⚠️ Every accuracy and time-saving figure above is a **vendor claim** from a controlled setting. Real performance depends on your environment, and specifically on how noisy your existing detections are.

## 2. The startup wave, and what created it

Autonomous SOC analyst products emerged directly out of **alert fatigue and the Tier 1 triage bottleneck** — the problem in [traditional blue teaming §6](/posts/traditional-blue-teaming/). They claim to investigate alerts autonomously through recursive reasoning rather than fixed playbooks, producing an evidence chain and a decision report.

The industry statistics quoted in that pitch — roughly 90% of SOCs overwhelmed by backlog and false positives, an average of 960 alerts a day — are mostly **vendor or survey claims** whose methodology is worth checking before repeating. The underlying problem is real; the specific numbers are marketing.

## 3. Measuring autonomous defensive capability

- **DARPA AIxCC** was a competition for automated vulnerability discovery **and automatic patching**. The winning system auto-patched 68% of synthetic vulnerabilities — see the [offensive AI notes](/posts/ai-augmented-red-teaming/) for the full result. The patching objective is what distinguishes it from a pure attack tool.
- **BountyBench** (Stanford, [2505.15216](https://arxiv.org/abs/2505.15216)) structures its evaluation as detect / exploit / **patch**. The result worth carrying around: **current coding agents score substantially higher on patching than on exploiting** — one agent at 90% patch against 32.5% exploit.

> **The implication**: my [offensive AI notes](/posts/ai-augmented-red-teaming/) established that attack capability is real. This says the same technology may work **better on defence than on offence**. That is one of the few places where the asymmetry runs in the defender's favour, and it is worth planning around rather than treating as a curiosity.

## 4. Purple teaming: the loop that connects them

- **VECTR** (Security Risk Advisors) tracks purple team campaigns mapped onto MITRE ATT&CK, recording when each test case ran and whether it was detected, then quantifying detection and prevention capability as **threat resilience metrics**. It is not an attack execution tool — it is the **measurement layer** over the results.
- **Breach and attack simulation** validates defences through automated adversary emulation, filling the validation stage of CTEM from [traditional red teaming](/posts/traditional-red-teaming/). VECTR and BAS are complementary: one executes, the other quantifies.
- Purple teaming **shortens the detection engineering feedback loop** from [traditional blue teaming](/posts/traditional-blue-teaming/) — attack, log what was detected, improve the rule — to the same session. It is a way of working, not a headcount.

## 5. Defender certification and practice

| Certification / platform | Character |
|---|---|
| **BTLO** (Blue Team Labs Online) | Gamified forensics, IR and SOC labs |
| **BTL1 / BTL2** (Security Blue Team) | Hands-on defender certification — BTL1 is a 24-hour lab with 20 questions, BTL2 a 72-hour advanced scenario |
| **GIAC cyber defence family** (GCIA, GCDA and others) | The established defensive certification group |

The defensive counterpart to the offensive certification ecosystem, and distinctly cyber-range-centred rather than exam-centred — which matches the discipline.

## 6. Human versus AI defender

| AI SOC strengths | Limits |
|---|---|
| Runs continuously; high-volume, high-speed triage; false positive filtering | **False positives that look confident**; no business context (cannot tell production from a test environment) |
| Consistency and scale; replaces repetitive Tier 1 work | **Automation bias** — one study found 47% of analysts affected; explainability and trust calibration problems; **over-reliance leading to skill atrophy** |

The literature consensus is that human oversight remains indispensable. That is the same principle as "keep human verification given AI's false positive tendency" from the [offensive AI notes](/posts/ai-augmented-red-teaming/) and "guardrails are not a boundary" from [AI system defence](/posts/ai-system-defense/) — **AI is an amplifier, not a replacement**.

Skill atrophy is the limit I would watch most closely, because it is the one that compounds. An analyst who has never worked a triage queue manually has no baseline against which to notice the agent being wrong.

---

## Where this leaves a defender

1. **Symmetric response became real in 2025.** Against automated reconnaissance and volume from offensive AI, you now have automated triage and hunting.
2. **But part of the asymmetry favours defence** — agents patch better than they exploit, and vulnerability discovery used defensively closes holes before they are found.
3. **The limits are the same as the attacker's**: false positives, missing context, automation bias. So the conclusion is the same: human in the loop.
4. **Purple teaming joins the three**: red builds the attack scenario, BAS executes it, VECTR measures detection against ATT&CK, and blue fills the gaps with rules.
5. **Where it meets the SIEM**: these products are the automation lineage of [SIEM and SOAR](/posts/traditional-blue-teaming/) continued — LLM reasoning layered on playbooks. Adopt them with audit logs, explainability and a human approval gate intact, because the failure mode is a confident wrong answer at machine speed.

> **The six axes, laid out:**
>
> | | Red (offence) | Blue (defence) |
> |---|---|---|
> | AI as target | [guide](/posts/kisa-ai-redteaming-guide/), [manual](/posts/kisa-ai-threat-response-manual/), [multimodal agents](/posts/multimodal-agent-red-teaming/) | [AI system defence](/posts/ai-system-defense/) |
> | AI as tool | [offensive AI](/posts/ai-augmented-red-teaming/) | **AI SOC (this post)** |
> | Conventional target | [traditional red teaming](/posts/traditional-red-teaming/) | [traditional blue teaming](/posts/traditional-blue-teaming/) |
>
> The practice and governance layer, [AI red teaming in practice](/posts/ai-red-teaming-practice/), cuts across both columns.
