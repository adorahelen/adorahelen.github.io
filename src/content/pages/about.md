---
title: "About"
description: "Security and AI engineer. What I work on, and what this blog is for."
---

I'm **Kangmin Kim**, a security engineer working at the intersection of security operations and LLM systems.

At work that means the security side of an observability platform and its LLM assistant — auth on model-query endpoints, SSRF guards on generated SQL, PII masking in prompt logs, and building the evaluation gates that decide whether a model's output is trustworthy enough to ship.

Outside work I build the same categories of system from scratch on my own hardware, because that's how I find out what actually holds:

- **SIEM / SOAR** — a four-layer XDR on a single home server: collection, detection, analysis, and an automated response chain
- **Detection engineering** — a SIEM rebuilt as Go microservices after reverse-engineering a commercial product, including a detection-rule DSL and its parser
- **Endpoint telemetry** — a Windows user-mode (Ring 3) sensor in C++, written specifically to find out where Ring 3 stops being enough
- **Digital forensics** — app artifact analysis; first author on a paper at CISC-W'25 (KISTI Director's Award)
- **On-prem LLM** — a cartridge-based agent console that keeps the engine fixed and swaps domain, knowledge, and model

## What this blog is for

Write-ups where I ran the thing and measured it, rather than summarizing what other people measured.

The bias is toward **what the result costs** — the layer that had to be tuned, the metric that hid the damage, the number that looked good and was meaningless. Those are the parts that get dropped from most write-ups, and they're usually the parts that matter when you have to make a decision.

Posts are in English; some have Korean originals in the linked repositories.

## Elsewhere

- GitHub — [github.com/adorahelen](https://github.com/adorahelen)
- Email — [kmkim26@outlook.com](mailto:kmkim26@outlook.com)
