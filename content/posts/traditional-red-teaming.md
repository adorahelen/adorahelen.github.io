---
title: "Traditional Red Teaming, and Why AI Red Teaming Is Its Extension Rather Than Its Replacement"
date: 2026-07-10T12:00:00+09:00
tags:
  - security
  - red-teaming
  - siem
summary: "A red team does not succeed by finding vulnerabilities. It succeeds by establishing whether an organisation's people, processes and technology actually detect and respond. I sorted out the terminology, the shared vocabulary (ATT&CK, kill chains), the engagement standards, and the feedback loop into the SOC — then asked what of it carries over to red teaming AI systems."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/traditional-red-teaming/)**

## TL;DR

- **The success criterion is not the finding.** A vulnerability assessment lists, a penetration test proves exploitability, a red team runs a goal-oriented operation to test whether detection and response work. Conflating them is the most common misuse of the term.
- **ATT&CK is the shared language between offence and defence.** Tag detection rules with technique IDs and your coverage gaps become visible; a red team designs its scenario along the same technique chain.
- **Purple is a function, not a team.** It is running the attack while validating and improving the detection logic in the same loop.
- **AI red teaming inherits the discipline and rebuilds the tooling.** Black/grey/white box, threat-actor personas, rules of engagement — all carry over. Executables, networks and credentials do not; model behaviour and language input do.

> This is the first of three axes: **AI as the target** ([the KISA guide](/posts/kisa-ai-redteaming-guide/) and [manual](/posts/kisa-ai-threat-response-manual/)), **AI as the tool**, and the conventional target covered here. Concept and methodology level — no exploit code or attack playbooks.

---

## 1. The terminology, sorted

| Activity | Focus | Exploitation | Scope |
|---|---|---|---|
| Vulnerability assessment | **Enumerating** weaknesses | ✗ | Broad scanning |
| Penetration testing | **Proving** exploitability | ○ | Wide coverage, explicit rules |
| Adversary emulation | **Reproducing** a specific threat actor's TTPs | ○ | Depth of tradecraft |
| Red teaming | **Goal-oriented end-to-end operation** | ○ | Scenario-based, tests detection and response |

Red is offence reproducing TTPs; blue is defence — detection, incident response, vulnerability management. **Purple is not a third team.** It is the collaborative function of executing an attack while immediately validating and refining the detection logic that should have caught it.

## 2. The shared vocabulary

**MITRE ATT&CK** is a knowledge base of adversary TTPs derived from real threat intelligence, organised as **tactic** (the why) × **technique** (the how). The Enterprise matrix runs to 14 tactics, 216 techniques and 475 sub-techniques across Windows, macOS, Linux, SaaS, IaaS, containers and hypervisors, with separate matrices for mobile and ICS. It is revised continuously — recent versions split Defense Evasion into stealth and defence-impairment groupings.

**MITRE Caldera** turns that into automation: define an adversary profile by technique ID, deploy agents, and let it select abilities dynamically as it learns the environment — which is how you emulate adaptive behaviour rather than a fixed script.

**The Cyber Kill Chain** (Lockheed Martin, 2011) is seven stages: reconnaissance, weaponisation, delivery, exploitation, installation, command and control, actions on objectives. Break one stage and the whole chain fails. Its weakness is everything after the perimeter — it barely models lateral movement.

**The Unified Kill Chain** (Pols, 2017) merges that with ATT&CK into 18 stages including post-exploitation, which is what you need to describe a multi-stage intrusion honestly.

## 3. Engagement standards

| Standard | Character |
|---|---|
| **PTES** | The most complete engagement lifecycle — pre-engagement, intelligence gathering, threat modelling, vulnerability analysis, exploitation, post-exploitation, reporting |
| **OSSTMM** | Scientific and metric-driven; measures network, physical, wireless and human channels; produces a quantified RAV score |
| **OWASP WSTG** | Detailed web and API checklist |
| **NIST SP 800-115** | Technical testing and assessment guide; regulated and federal environments |
| **PCI-DSS Req. 11** | Annual internal and external testing of the cardholder data environment, plus segmentation validation |

In practice nobody uses one: **PTES for the process, WSTG for the web technique checklist** is the common pairing.

## 4. Engagement types

**Access level** — black box (no prior information, most realistic), grey box (partial information, practical for recurring compliance work), white box (full source and documentation). This is where AI red teaming's black/grey/white box framing comes from.

**Assumed breach** starts with a foothold already granted, which moves the test away from initial access and onto **detection and containment** — usually the more valuable question, because initial access will eventually succeed.

**Attack vectors** span social engineering (phishing, voice, social platforms), physical intrusion (badge cloning, tailgating, access control bypass) and network lateral movement.

## 5. Command and control, at concept level

A C2 framework maintains communication with compromised systems and supports post-exploitation. The structure is consistent: an **implant or beacon** calling back on an interval, a **team server** for control, a **transport** (HTTP, DNS), and a **malleable profile** to shape traffic so it looks like something else.

The commercial standard-setter has been Cobalt Strike, alongside Metasploit as the open exploitation framework, with a set of modular alternatives (Sliver, Mythic, Havoc, Brute Ratel) that gained ground through 2025–26.

For a defender the relevant fact is that **the communication and behaviour signatures of these frameworks are a primary target of detection engineering** — beacon intervals, jitter, profile artefacts, and the telemetry lateral movement leaves behind.

## 6. Where the discipline has moved

**CTEM** (continuous threat exposure management) organises this into scoping, discovery, prioritisation, validation and mobilisation. The point is to move past periodic CVE scanning to continuous management of misconfiguration, identity, excessive privilege and credential exposure.

**Breach and attack simulation** automates adversary emulation to validate detection controls — it is what fills CTEM's validation stage.

**TIBER-EU and CBEST** are intelligence-led red teaming regimes for financial institutions: run against production, with only a small white team aware, precisely so that detection and response are genuinely measured. These connect to the threat-led penetration testing requirements in EU financial regulation.

## 7. The loop into the SOC

This is the part that matters if you operate a SIEM. Red teaming is the **input** to control validation; detection engineering is the **output**.

- **ATT&CK-based detection engineering**: tag detection rules with technique IDs and coverage becomes measurable. The red team designs scenarios along a technique chain — initial access, execution, credential access, lateral movement, exfiltration — which is the same axis the rules are tagged on.
- **The feedback loop**: at debrief, map what was detected and what was missed onto ATT&CK, producing a **detection gap backlog** that the blue team works through. Purple teaming shortens that loop from weeks to the same session.
- **Implementation**: the major SIEM platforms support ATT&CK tagging on correlation rules, and analytic libraries such as MITRE's Cyber Analytics Repository give you a starting set rather than a blank page.

> **From a SIEM operator's seat**: red teaming and BAS are the requirements supplier that defines what your SIEM has to detect, expressed per ATT&CK technique. Beacon intervals, malleable profiles and lateral movement telemetry are not abstractions — they are the detection rules you have not written yet.

---

## What carries over to AI

What makes traditional red teaming a discipline is not enumeration or compliance. It is **reproducing real threat-actor TTPs in a goal-oriented operation**, with success measured as whether detection and response worked. The field is structured by a shared language (ATT&CK, kill chains), engagement standards (PTES, OSSTMM, NIST 800-115, TIBER-EU) and the red-to-blue-to-purple validation loop.

**Red teaming AI systems extends that spirit — adversarial thinking, goal-based scenarios, control validation — onto a new attack surface: the model, the prompt, the data boundary.** The methodological lineage carries over intact: access levels, personas as threat-actor models, rules of engagement. The tooling and tactics diverge, because the target is model behaviour and language input rather than executables, networks and credentials.

So AI red teaming is **an extension and a specialisation of traditional red teaming**, not a separate practice. Both rest on the same principle: think like an attacker in order to verify a defence. Which also means the reverse is true — an organisation that cannot run a conventional red team well is unlikely to get value from an AI one.
