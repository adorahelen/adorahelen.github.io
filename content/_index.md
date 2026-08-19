---
title: "Kangmin Kim"
description: "Security engineer — SIEM/SOAR, endpoint telemetry, and the security of LLM systems."
---

I work on the security side of LLM systems — authentication on model-query endpoints, SSRF guards on generated SQL, PII masking in prompt logs, and the evaluation gates that decide whether a model's output is trustworthy enough to ship.

Outside work I build the same categories of system from scratch, because that is how I find out what actually holds.

### Selected work

**Self-built XDR** — Collection, detection, analysis and response as four layers on a single host, with a six-stage automated response chain and a SOC console. Ten containers, one exposed port.

**SIEM rebuilt in Go** — Took the functional axes a commercial SIEM covers and rebuilt them from scratch as eleven Go microservices, including a detection-rule DSL and its parser.

**User-mode EDR sensor** — 2,840 lines of C++ observing processes, files, network and registry from Ring 3, written to find out exactly where Ring 3 stops being enough. Scored against Sysmon's event set, gaps published.

**On-prem LLM agent console** — Engine fixed, domain swapped: prompts, knowledge and model are cartridges. Hardware tier detection picks the model preset at install time.

### Publication

**Kim, K.**, Byun, H., Cho, M., Kim, Y., Son, K., & Lee, C. (2025). *User Behaviour Analysis of the Threads Application from a Digital Forensics Perspective.* Proceedings of the Korea Institute of Information Security and Cryptology Winter Conference (CISC-W'25), Vol. 35, No. 2.

Awarded the **KISTI Director's Award**. First author. Recovered plaintext direct messages, follow relationships and in-app search history from a rooted Android device — including from the write-ahead log that a database-only procedure would miss.
