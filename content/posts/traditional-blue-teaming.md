---
title: "Blue Team Maturity Is Not Alert Volume — It Is Where on the Pyramid You Detect"
date: 2026-07-13T10:00:00+09:00
tags:
  - security
  - blue-team
  - siem
  - detection-engineering
summary: "The defensive counterpart to red teaming: D3FEND and Engage as the shared vocabulary, Sigma and detection-as-code as the engineering practice, four hunting methodologies, and the 2025 rewrite of NIST 800-61 that quietly deleted the four-phase incident response lifecycle everyone still cites."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/traditional-blue-teaming/)**

## TL;DR

- **Maturity is measured by abstraction level, not alert count.** Detect at the TTP level (the top of the Pyramid of Pain), manage detections like code, and keep improving them with intelligence. Volume of alerts measures nothing.
- **D3FEND is not a mirror of ATT&CK.** It is a knowledge graph that maps offence to defence *inferentially* through a shared digital artifact ontology — designed to be queried, not read as a lookup table.
- **NIST SP 800-61 Rev.3 (April 2025) deleted the four-phase lifecycle.** It is now a CSF 2.0 community profile. If you cite "the NIST four phases" after that date you are citing the superseded revision, and you should say so.
- **The chronic disease of SIEM is rule maintenance cost.** Sigma in Git with a documented blind-spot analysis per rule is the treatment, and it is the blue-team half of the red-team feedback loop.

> Defensive counterpart to [traditional red teaming](/posts/traditional-red-teaming/). First of three blue-team axes: conventional targets here, [defending AI systems](/posts/ai-system-defense/) and [defending with AI](/posts/ai-augmented-defense-ai-soc/) separately.

---

## 1. The shared vocabulary: D3FEND and Engage

**MITRE D3FEND** is the defensive counterpart to ATT&CK — NSA and DoD funded, formally released at 1.0 in January 2025 and now at 1.4. Seven tactics: **Model, Harden, Detect, Isolate, Deceive, Evict, Restore** (Restore was new at 1.0).

The important structural point: D3FEND is not a one-to-one table joining attack to defence. Both sides reference a shared **digital artifact ontology**, and the mapping between them is **inferential** — it is a knowledge graph designed for SPARQL queries and tool integration, not a spreadsheet. Detect is the largest category, at around 90 techniques.

**MITRE Engage** (which replaced Shield after community feedback, v1.0 in February 2022) covers deception and adversary engagement. Five columns: **Prepare** (input), **Expose** (surface the adversary with high-confidence detection), **Affect** (disrupt their operation), **Elicit** (draw out their TTPs), **Understand** (output and analysis). Worth knowing that its maintenance activity is much lower than ATT&CK's — it leans on a contribution programme.

> Where D3FEND's Deceive tactic overlaps Engage is the active defence and honeypot space. And the [C2 framework](/posts/traditional-red-teaming/) signatures from the red-team side land directly in D3FEND's Detect techniques.

## 2. Detection engineering: treat detections like code

**Sigma** is a SIEM-neutral detection signature format in YAML — "Snort or YARA, for logs" — created in 2017 by Florian Roth and Thomas Patzke. Conversion tooling moved from the older `sigmac` to **pySigma and sigma-cli**, and **spec v2.0 (August 2024)** added correlation and taxonomy support. It is the de facto exchange format for detection-as-code.

**Palantir's ADS** (alerting and detection strategy) framework forces each detection to be documented with goal, categorisation (mapped to ATT&CK), strategy abstract, technical context, **blind spots and assumptions**, false positives, validation, priority and response. The blind-spots field is the one that earns its keep: it forces the author to state what the detection *cannot* see, which is the thing every rule silently omits.

**Detection-as-code** applies version control, code review, CI/CD and unit tests to detection rules. There is no formal standard — it is industry practice that converged.

For maturity measurement, Ryan Stillions' **DML model** (2014) scores an organisation by the abstraction level it can detect at, from atomic indicators (DML-0) up to adversary goals (DML-8).

> **From a SIEM operator's seat**: keep Sigma rules in Git and document each correlation rule in ADS form — blind spots, false positive conditions, validation method — and rule maintenance cost, the chronic disease of SIEM operations, becomes tractable. This is the blue-team implementation of the [red-team feedback loop](/posts/traditional-red-teaming/).

## 3. Threat hunting: hypotheses against what you missed

| Methodology | Origin | Structure |
|---|---|---|
| **PEAK** | Splunk SURGe (with D. Bianco), 2023 | **P**repare–**E**xecute–**A**ct with **K**nowledge, plus three hunt types: hypothesis-driven, baseline (exploratory data analysis), and **M-ATH** (machine-assisted) |
| **TaHiTI** | Dutch financial FI-ISAC, 2018 | Initiate → Hunt (define/refine plus execute) → Finalise, with CTI triggering hunts and hunts producing CTI |
| **Sqrrl hunting loop** | Sqrrl, ~2015 | Hypothesis → investigate → discover new TTPs → automate the analytic, then repeat |
| **HMM** | David Bianco, 2015 | HM0 (initial) to HM4 (leading), scored on data collection, procedural repeatability and automation |

**The Pyramid of Pain** (Bianco, 2013) is the load-bearing idea underneath all of it: hash → IP → domain → network/host artifacts → tools → **TTPs**, with the attacker's cost of replacement rising as you climb. It is the argument for aiming detection at the TTP level, and the shared foundation of detection engineering, CTI and hunting alike. A red team's [ATT&CK scenario](/posts/traditional-red-teaming/) *is* the top of this pyramid.

## 4. DFIR: the NIST 800-61 turn

**NIST SP 800-61 Rev.3 (3 April 2025) formally supersedes Rev.2 (2012)**, and the change is larger than a revision. The title moved from *Computer Security Incident Handling Guide* to *Incident Response Recommendations and Considerations for Cybersecurity Risk Management: A CSF 2.0 Community Profile*. The **fixed four-phase lifecycle was removed** — preparation, detection and analysis, containment/eradication/recovery, post-incident — and incident response activity is now mapped onto **CSF 2.0's six functions**: Govern, Identify, Protect, Detect, Respond, Recover. The detailed how-to was dropped on the reasoning that it cannot be maintained as a single static document.

**SANS PICERL** (preparation, identification, containment, eradication, recovery, lessons learned) is substantively the old four phases with containment, eradication and recovery separated and the loop made explicit. The common arrangement is PICERL for field runbooks and NIST for governance.

> ⚠️ **Citation warning**: after April 2025, referencing "the NIST four phases" requires noting that you mean Rev.2. Rev.3 is not a procedure; it is a CSF 2.0 profile.

## 5. How SIEM became SOAR, XDR and NDR

| When | What |
|---|---|
| 2005 | Gartner (Williams and Nicolett) first uses **SIEM** for SIM + SEM |
| ~2013 | **EDR** enters the vocabulary (commonly credited to Chuvakin at Gartner) |
| 2015 → 2017 | Gartner redefines **SOAR** as security orchestration, automation and response |
| 2018 | **XDR** coined at Palo Alto (Nir Zuk), extending EDR across multiple telemetry layers |
| June 2020 | Gartner renames NTA to **NDR**, with response as a core function |

The driver throughout is the same chronic problem: SIEM's rule maintenance cost and its high volume of context-free alerts. That pressure produced response automation (SOAR), vertically integrated detection (XDR) and a network-layer complement for an encrypted world (NDR). This accumulated automation is the immediate predecessor of the [AI SOC](/posts/ai-augmented-defense-ai-soc/) discussion.

## 6. SOC operating models

**SOC-CMM** (Rob van Os, 2016) is the de facto open maturity standard: five domains — business, people, process, technology, services — with the last two measured for both maturity and capability.

**Tiered versus tierless** is the live argument. The conventional Tier 1 (triage) / Tier 2 (investigation) / Tier 3 (hunting) structure loses context at every handoff and burns out Tier 1. The tierless alternative has an analyst own an incident end to end.

**Alert fatigue** remains the dominant complaint — a 2025 SANS survey put 73% of respondents naming false positives as their biggest challenge. That single statistic is the background for both hunting-centric redesigns and the AI SOC pitch.

## 7. CTI: intelligence leads the defence

**Levels**: strategic (executive), operational (imminent attack), tactical (TTPs). Note that sources disagree — some use "tactical" for IOCs — so state your definition when citing.

**STIX/TAXII**: the OASIS standards, with **STIX 2.1 and TAXII 2.1 approved on 10 June 2021** (a JSON model of SDOs, SCOs and SROs).

**The Diamond Model** (2013) frames an intrusion across four vertices — adversary, capability, infrastructure, victim — with activity threads correlating them into campaigns.

**F3EAD** joins an operations cycle (find, fix, finish) to an intelligence cycle (exploit, analyse, disseminate). *Intelligence-Driven Incident Response* (Roberts and Brown) popularised it as the loop where IR and CTI reinforce each other.

---

## How this relates to red, and to AI

What makes blue teaming a discipline is not owning a SIEM and an EDR. It is a **closed loop**: detect adversary TTPs at a high abstraction level (D3FEND, Pyramid of Pain), manage those detections like code (Sigma, ADS, detection-as-code), and improve them continuously with intelligence (CTI, F3EAD, hunting). If [red teaming](/posts/traditional-red-teaming/) is the requirements supplier for that loop, blue teaming is the implementation — detection rules, playbooks, hunt hypotheses. Purple closes the gap in real time.

> **On defending AI systems**: the spirit transfers — TTP-centred detection, defence in depth, closed-loop improvement — but the target is the **model, the prompt and the data boundary** rather than executables and networks, so both what you detect and the control layers get largely redesigned. MITRE built the AI versions separately: ATT&CK became **ATLAS**, and D3FEND's role is taken by ATLAS mitigations. An AI blue team is an extension and a specialisation of this one, not a different profession.
