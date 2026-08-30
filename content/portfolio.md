---
title: "Portfolio"
description: "An 18-slide deck on the four systems I built and what measuring them showed."
showDate: false
showAuthor: false
showReadingTime: false
showTableOfContents: false
---

An 18-slide deck covering the four systems described on the [Work](/work/) page — self-built XDR, a SIEM rebuilt in Go, a user-mode EDR sensor, an on-premise LLM agent console — plus the digital forensics research and one incident-response case. **The whole deck is below. There is nothing to download.**

> **The deck is in Korean.** The architecture diagrams, measurements and stack labels read across languages; the prose does not. An English edition is planned.

<div class="deck-actions">
<a class="primary" href="/portfolio/kangmin-kim-portfolio-ko.pdf">Download PDF (18 slides · 186KB)</a>
<a class="secondary" href="/portfolio/kangmin-kim-portfolio-ko.pptx">Source PPTX</a>
</div>

Pretendard is embedded in the PDF, so it renders as built without installing anything. Use the PPTX only if you need to edit it — that one needs [Pretendard](https://github.com/orioncactus/pretendard) (free, OFL) installed or the line breaks shift.

## The deck

<style>
/* 덱만 본문 읽기 폭(max-w-prose)을 벗어나 넓게 쓴다 — 16:9 슬라이드는 좁으면 못 읽는다.
   목차를 끈 페이지라 오른쪽 열이 비어 있어 그쪽으로 확장한다. */
/* 목차를 끄면 본문 열이 컨테이너 전체로 늘어나 문단이 지나치게 길어진다.
   글은 읽기 폭으로 되돌리고, 덱만 넓게 쓴다. */
article p, article ul, article ol, article table,
article blockquote, article h2, article h3{max-width:68ch}
.deck, .deck *{max-width:none}
.deck{margin:1.5rem 0;width:100%}
@media (min-width:1024px){.deck{width:min(1024px,calc(100vw - 20rem))}}
.deck figure{margin:0 0 1.6rem}
.deck img{width:100%;height:auto;display:block;border:1px solid rgba(128,128,128,.28);border-radius:6px}
.deck a{display:block;line-height:0}
.deck figcaption{font-size:.82rem;opacity:.62;margin-top:.4rem;font-variant-numeric:tabular-nums}
.deck-actions{display:flex;flex-wrap:wrap;gap:.6rem;align-items:center;margin:1.2rem 0 .4rem}
.deck-actions a.primary{display:inline-block;padding:.55rem 1.1rem;border-radius:6px;
  background:#2563eb;color:#fff!important;font-weight:600;text-decoration:none}
.deck-actions a.primary:hover{background:#1e40af}
.deck-actions a.secondary{font-size:.88rem;opacity:.75}
</style>
<div class="deck">
<figure><a href="/portfolio/slides/s01.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s01.webp" alt="Cover — Security × AI Engineer" loading="eager" decoding="async" width="1600" height="900"></a><figcaption>01 / 18 · Cover — Security × AI Engineer</figcaption></figure>
<figure><a href="/portfolio/slides/s02.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s02.webp" alt="Profile — building at home what I build at work" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>02 / 18 · Profile — building at home what I build at work</figcaption></figure>
<figure><a href="/portfolio/slides/s03.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s03.webp" alt="Project map — six personal repositories" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>03 / 18 · Project map — six personal repositories</figcaption></figure>
<figure><a href="/portfolio/slides/s04.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s04.webp" alt="SIEM-Trinity — self-built XDR on one home server" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>04 / 18 · SIEM-Trinity — self-built XDR on one home server</figcaption></figure>
<figure><a href="/portfolio/slides/s05.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s05.webp" alt="SIEM-Trinity — architecture" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>05 / 18 · SIEM-Trinity — architecture</figcaption></figure>
<figure><a href="/portfolio/slides/s06.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s06.webp" alt="LLM alignment — reproducing abliteration" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>06 / 18 · LLM alignment — reproducing abliteration</figcaption></figure>
<figure><a href="/portfolio/slides/s07.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s07.webp" alt="abliteration — what was measured" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>07 / 18 · abliteration — what was measured</figcaption></figure>
<figure><a href="/portfolio/slides/s08.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s08.webp" alt="edr-lab — how far you see without a kernel driver" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>08 / 18 · edr-lab — how far you see without a kernel driver</figcaption></figure>
<figure><a href="/portfolio/slides/s09.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s09.webp" alt="edr-lab — architecture" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>09 / 18 · edr-lab — architecture</figcaption></figure>
<figure><a href="/portfolio/slides/s10.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s10.webp" alt="security-labs — digital forensics and crypto implementation" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>10 / 18 · security-labs — digital forensics and crypto implementation</figcaption></figure>
<figure><a href="/portfolio/slides/s11.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s11.webp" alt="Threads app forensics — architecture" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>11 / 18 · Threads app forensics — architecture</figcaption></figure>
<figure><a href="/portfolio/slides/s12.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s12.webp" alt="agent-console — cartridge-based on-premise AI console" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>12 / 18 · agent-console — cartridge-based on-premise AI console</figcaption></figure>
<figure><a href="/portfolio/slides/s13.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s13.webp" alt="agent-console — architecture" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>13 / 18 · agent-console — architecture</figcaption></figure>
<figure><a href="/portfolio/slides/s14.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s14.webp" alt="Infrastructure — from cloud down to hardware I run" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>14 / 18 · Infrastructure — from cloud down to hardware I run</figcaption></figure>
<figure><a href="/portfolio/slides/s15.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s15.webp" alt="ClickHouse 700× write amplification — diagnosis to fix" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>15 / 18 · ClickHouse 700× write amplification — diagnosis to fix</figcaption></figure>
<figure><a href="/portfolio/slides/s16.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s16.webp" alt="Incident response — a flaw I found in my own system" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>16 / 18 · Incident response — a flaw I found in my own system</figcaption></figure>
<figure><a href="/portfolio/slides/s17.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s17.webp" alt="Stack summary and how I work" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>17 / 18 · Stack summary and how I work</figcaption></figure>
<figure><a href="/portfolio/slides/s18.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s18.webp" alt="Closing" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>18 / 18 · Closing</figcaption></figure>
</div>

## What's in it

| Slides | Contents |
| --- | --- |
| 1–3 | Positioning, profile, project map |
| 4–5 | **Self-built XDR** — four-layer monorepo, six-stage automated response chain |
| 6–7 | **LLM alignment** — reproducing the "refusal is one direction" result and measuring what it costs |
| 8–9 | **edr-lab** — Ring 3 sensor, and the Ring 0 line I deliberately did not cross |
| 10–11 | **security-labs** — Threads app forensics, the CISC-W'25 paper and award |
| 12–13 | **agent-console** — cartridge architecture, hardware-tier install |
| 14–15 | **Infrastructure** — cloud to on-premise, and a 700× write amplification incident |
| 16 | **Incident response** — an unauthenticated endpoint I found in my own system |
| 17–18 | Stack summary, how I work |

## Two notes on what is not in it

**No employer material.** Every diagram, number and screenshot comes from personal repositories. My professional work appears once, as two lines of role description on the profile slide — no product names, no architecture, no screens.

**The deck is generated, not drawn.** Content, diagrams and figures all live in a Python script that builds the file with `python-pptx`, so a change to a project is a diff, not a redraw. That is also why the numbers in it match the repositories they came from.

The PDF follows the same principle. Rather than passing the file through a general-purpose converter, a separate renderer draws the shape coordinates directly — converters apply Korean-Latin spacing rules and insert gaps that are not in the file.
