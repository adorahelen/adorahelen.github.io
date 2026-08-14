---
title: "Work"
description: "Systems I built, and what measuring them actually showed."
showDate: false
showAuthor: false
showReadingTime: false
showTableOfContents: true
---

Four systems I built outside of work, in the categories I work in professionally. Each one exists because I wanted to know where the thing breaks — so what follows includes the limits, not just the features.

Most repositories are private; I can grant access on request.

## Self-built XDR

**Collection → detection → analysis → response, four layers on one host, plus a SOC console.**

- Collection with Zeek, Suricata and a HIDS agent; logs and metrics into Loki and Prometheus.
- Detection as a FastAPI service with four parallel detectors (isolation forest, coefficient of variation, entropy and a rule path), 27 endpoints behind a single BFF.
- Analysis against a local LLM with 697 ATT&CK techniques in a vector store, using a fixed four-section prompt (summary, attack chain, risk, recommended action) so the output shape can't drift.
- **A six-stage automated response chain** — host visibility, automatic IP banning, active response, threat-intel matching, SOAR playbooks, case creation.

**The judgment call worth asking about**: all six stages are implemented and all four operational toggles ship **off**. An automated ban that fires on a false positive takes down my own service. The conservative default is the decision, not an unfinished feature.

Ten containers running, exactly one exposed port; everything else binds to localhost.

## SIEM rebuilt in Go

**Reverse-engineered a commercial SIEM, then rebuilt its functional axes as microservices.**

- Decompiled 128 OSGi bundles (22,844 `.java` files), triaged them into four classes, and produced a bundle-by-bundle mapping table — including why each unimplemented bundle was skipped.
- Rebuilt as **eleven Go services**: ingest gateway, parser/normalizer, storage writer, search, detection/correlation, alerting, workflow engine, integration adapter, config/identity, forwarder, console.
- **Wrote a detection-rule DSL and its parser**, with staged tests. I'm not writing rules in someone else's language; I designed the language.
- Added what the original lacked: a SOAR workflow engine, CTI over TAXII 2.1 with STIX parsing, and JWT + LDAP/AD + TOTP MFA.

103 of 128 bundle functions implemented; 22 of 22 end-to-end tests passing.

**Also in this project**: I found an unauthenticated ingest endpoint in my own test binary, and documented the whole incident-response path — reproduction, binary string analysis proving no auth existed, scope of what had been collected in plaintext, server-side block, then the code-level fix. Finding it in my own system was the point.

## User-mode EDR sensor

**2,840 lines of C++ asking one question: how far can Ring 3 go?**

- Observes process creation and termination, file changes, IPv4 TCP connections and autorun registry keys — each through a user-mode workaround for something the kernel would do with a callback.
- Every workaround costs something, and the README states each cost: file events carry no originating PID, network polling loses connections that open and close between samples.
- Rule engine with 14 rules and parent-child process correlation. Events land in SQLite; an Electron console polls the delta every second.
- Separate forensic collection mode for `$MFT`, registry hives, `.evtx` and Prefetch, with a manifest and hashes.

**Scored against Sysmon's event set: 5 covered, 3 partial, 6 not covered** — module loads, injection and `lsass` access are the blind spots, and they're published as blind spots. The repository is 298,925 lines of C/C++; **2,840 of those are mine** and the README says so before anyone asks.

## On-prem LLM agent console

**The engine is fixed; the domain is a cartridge.**

- Three slots — prompts, knowledge and model — each swappable through a path that already existed in the production system it descends from. Changing domain requires no code change.
- `install.sh` detects VRAM and RAM, picks a hardware tier, selects a model preset, and registers a service. One command, end to end.
- Retrieval is BGE-M3 dense plus ColBERT with two-way reciprocal rank fusion; the repository still contains the variants I compared to get there.
- Knowledge provenance is traced per file — for the security cartridge, 3,955 documents broken down as 1,575 public corpus, 1,930 self-authored, 301 model-generated, with license review recorded.

Largest of my codebases and the most tested: 70,424 lines of Python, 55 test files.

**What I published about it that I could have left out**: PII masking only runs on one handler path, so the API tier still sends raw text outward. It's written down as an open issue because it's a product-identity problem, not a bug.

## Research

**Kim, K.**, Byun, H., Cho, M., Kim, Y., Son, K., & Lee, C. (2025). *User Behaviour Analysis of the Threads Application from a Digital Forensics Perspective.* Proceedings of the Korea Institute of Information Security and Cryptology Winter Conference (CISC-W'25), Vol. 35, No. 2. **KISTI Director's Award.**

First author. Recovered viewed posts with timestamps, direct message bodies with sender and time, pending unsent images, and full in-app search history from a rooted Android device.

Three prior studies had concluded that mobile analysis of this app yielded little. The finding that changed that: message bodies sit in plaintext inside a BLOB column, and the write-ahead log holds the most recent messages that a database-only procedure never sees.
