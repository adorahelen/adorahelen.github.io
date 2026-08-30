---
title: "포트폴리오"
description: "직접 만든 네 개의 시스템과, 측정해보니 어땠는지를 담은 18장짜리 덱."
showDate: false
showAuthor: false
showReadingTime: false
showTableOfContents: false
---

[프로젝트](/ko/work/) 페이지에서 다룬 네 개의 시스템 — 자작 XDR, Go로 다시 지은 SIEM, User-mode EDR 센서, 온프레미스 LLM 에이전트 콘솔 — 에 디지털 포렌식 연구와 침해사고 대응 사례를 더한 18장짜리 덱입니다. **아래에 전문이 실려 있습니다. 내려받지 않아도 됩니다.**

<div class="deck-actions">
<a class="primary" href="/portfolio/kangmin-kim-portfolio-ko.pdf">PDF 내려받기 (18장 · 186KB)</a>
<a class="secondary" href="/portfolio/kangmin-kim-portfolio-ko.pptx">원본 PPTX</a>
</div>

PDF는 Pretendard를 파일 안에 넣어 두었습니다. 폰트를 설치하지 않아도 만든 그대로 보입니다. PPTX는 편집이 필요할 때만 쓰시고, 이쪽은 [Pretendard](https://github.com/orioncactus/pretendard)(무료·OFL)가 없으면 줄바꿈이 밀립니다.

## 덱 전문

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
<figure><a href="/portfolio/slides/s01.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s01.webp" alt="보안 × AI 엔지니어  김강민" loading="eager" decoding="async" width="1600" height="900"></a><figcaption>01 / 18 · 보안 × AI 엔지니어  김강민</figcaption></figure>
<figure><a href="/portfolio/slides/s02.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s02.webp" alt="프로필 — 실무에서 만드는 것을, 집에서 처음부터 조립한다" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>02 / 18 · 프로필 — 실무에서 만드는 것을, 집에서 처음부터 조립한다</figcaption></figure>
<figure><a href="/portfolio/slides/s03.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s03.webp" alt="프로젝트 맵 — 개인 저장소 6선 (전부 adorahelen)" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>03 / 18 · 프로젝트 맵 — 개인 저장소 6선 (전부 adorahelen)</figcaption></figure>
<figure><a href="/portfolio/slides/s04.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s04.webp" alt="SIEM-Trinity — 홈서버 1대로 굴리는 자작 XDR" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>04 / 18 · SIEM-Trinity — 홈서버 1대로 굴리는 자작 XDR</figcaption></figure>
<figure><a href="/portfolio/slides/s05.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s05.webp" alt="SIEM-Trinity — 아키텍처" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>05 / 18 · SIEM-Trinity — 아키텍처</figcaption></figure>
<figure><a href="/portfolio/slides/s06.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s06.webp" alt="LLM 정렬 취약성 실증 — abliteration 재현" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>06 / 18 · LLM 정렬 취약성 실증 — abliteration 재현</figcaption></figure>
<figure><a href="/portfolio/slides/s07.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s07.webp" alt="abliteration — 무엇을 측정했나" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>07 / 18 · abliteration — 무엇을 측정했나</figcaption></figure>
<figure><a href="/portfolio/slides/s08.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s08.webp" alt="edr-lab — 커널 없이 어디까지 관측되는가" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>08 / 18 · edr-lab — 커널 없이 어디까지 관측되는가</figcaption></figure>
<figure><a href="/portfolio/slides/s09.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s09.webp" alt="edr-lab — 아키텍처" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>09 / 18 · edr-lab — 아키텍처</figcaption></figure>
<figure><a href="/portfolio/slides/s10.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s10.webp" alt="security-labs — 디지털 포렌식 · 암호 구현" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>10 / 18 · security-labs — 디지털 포렌식 · 암호 구현</figcaption></figure>
<figure><a href="/portfolio/slides/s11.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s11.webp" alt="Threads 앱 포렌식 — 아키텍처" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>11 / 18 · Threads 앱 포렌식 — 아키텍처</figcaption></figure>
<figure><a href="/portfolio/slides/s12.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s12.webp" alt="agent-console — 카트리지형 온프레미스 AI 에이전트 콘솔" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>12 / 18 · agent-console — 카트리지형 온프레미스 AI 에이전트 콘솔</figcaption></figure>
<figure><a href="/portfolio/slides/s13.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s13.webp" alt="agent-console — 아키텍처" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>13 / 18 · agent-console — 아키텍처</figcaption></figure>
<figure><a href="/portfolio/slides/s14.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s14.webp" alt="운영 · 인프라 — 클라우드에서 내려와 직접 굴린다" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>14 / 18 · 운영 · 인프라 — 클라우드에서 내려와 직접 굴린다</figcaption></figure>
<figure><a href="/portfolio/slides/s15.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s15.webp" alt="ClickHouse 쓰기 증폭 700배 — 진단에서 조치까지 — 아키텍처" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>15 / 18 · ClickHouse 쓰기 증폭 700배 — 진단에서 조치까지 — 아키텍처</figcaption></figure>
<figure><a href="/portfolio/slides/s16.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s16.webp" alt="침해사고 대응 실사례 — 내가 만든 시스템의 결함을 내가 찾았다" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>16 / 18 · 침해사고 대응 실사례 — 내가 만든 시스템의 결함을 내가 찾았다</figcaption></figure>
<figure><a href="/portfolio/slides/s17.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s17.webp" alt="스택 총괄 · 일하는 방식" loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>17 / 18 · 스택 총괄 · 일하는 방식</figcaption></figure>
<figure><a href="/portfolio/slides/s18.webp" target="_blank" rel="noopener"><img src="/portfolio/slides/s18.webp" alt="보안 스택을 설계부터 운영까지 직접 만들어 본 엔지니어입니다." loading="lazy" decoding="async" width="1600" height="900"></a><figcaption>18 / 18 · 보안 스택을 설계부터 운영까지 직접 만들어 본 엔지니어입니다.</figcaption></figure>
</div>

## 구성

| 장 | 내용 |
| --- | --- |
| 1–3 | 포지셔닝, 프로필, 프로젝트 맵 |
| 4–5 | **자작 XDR** — 4계층 모노레포, 자동 대응 체인 6단계 |
| 6–7 | **LLM 정렬 취약성** — "거부는 단일 방향" 논문 재현과 그 대가 측정 |
| 8–9 | **edr-lab** — Ring 3 센서, 그리고 일부러 넘지 않은 Ring 0 선 |
| 10–11 | **security-labs** — Threads 앱 포렌식, CISC-W'25 논문과 원장상 |
| 12–13 | **agent-console** — 카트리지 구조, 하드웨어 티어 설치 |
| 14–15 | **운영·인프라** — 클라우드에서 온프레미스로, 그리고 쓰기 증폭 700배 사고 |
| 16 | **침해사고 대응** — 내가 만든 시스템에서 내가 찾은 무인증 엔드포인트 |
| 17–18 | 스택 총괄, 일하는 방식 |

## 덱에 들어 있지 **않은** 것 두 가지

**회사 산출물이 없습니다.** 모든 다이어그램·수치·화면이 개인 저장소에서 나왔습니다. 직장 경력은 프로필 장의 담당 서술 두 줄이 전부이고, 제품명·아키텍처·화면은 넣지 않았습니다.

**이 덱은 그린 게 아니라 생성된 것입니다.** 내용·다이어그램·수치가 전부 파이썬 스크립트 안에 있고 `python-pptx`가 파일을 만듭니다. 그래서 프로젝트가 바뀌면 다시 그리는 게 아니라 diff가 남고, 덱의 숫자가 저장소의 숫자와 어긋나지 않습니다.

위 PDF도 같은 원칙으로 만들었습니다. 변환기를 거치지 않고 도형 좌표를 직접 그리는 렌더러를 따로 두었습니다 — 범용 변환기가 한글과 영문 사이에 원본에 없는 공백을 넣기 때문입니다.
