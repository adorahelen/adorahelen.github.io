---
title: "Portfolio"
description: "An 18-slide deck on the four systems I built and what measuring them showed."
showDate: false
showAuthor: false
showReadingTime: false
---

An 18-slide deck covering the four systems described on the [Work](/work/) page — self-built XDR, a SIEM rebuilt in Go, a user-mode EDR sensor, an on-premise LLM agent console — plus the digital forensics research and one incident-response case.

> **The deck is in Korean.** The architecture diagrams, measurements and stack labels read across languages; the prose does not. An English edition is planned.

**[Download the deck (PDF, 18 slides, 1.1 MB)](/portfolio/kangmin-kim-portfolio-ko.pdf)** · [PowerPoint source](/portfolio/kangmin-kim-portfolio-ko.pptx)

## What's in it

| Slides | Contents |
| --- | --- |
| 1–3 | Positioning, profile, project map |
| 4–5 | **Self-built XDR** — four-layer monorepo, six-stage automated response chain |
| 6–7 | **signum** — commercial SIEM reverse-engineered into eleven Go microservices |
| 8–9 | **edr-lab** — Ring 3 sensor, and the Ring 0 line I deliberately did not cross |
| 10–11 | **security-labs** — Threads app forensics, the CISC-W'25 paper and award |
| 12–13 | **agent-console** — cartridge architecture, hardware-tier install |
| 14–15 | **Infrastructure** — cloud to on-premise, and a 700× write amplification incident |
| 16 | **Incident response** — an unauthenticated endpoint I found in my own system |
| 17–18 | Stack summary, how I work |

## Two notes on what is not in it

**No employer material.** Every diagram, number and screenshot comes from personal repositories. My professional work appears once, as two lines of role description on the profile slide — no product names, no architecture, no screens.

**The deck is generated, not drawn.** Content, diagrams and figures all live in a Python script that builds the file with `python-pptx`, so a change to a project is a diff, not a redraw. That is also why the numbers in it match the repositories they came from.
