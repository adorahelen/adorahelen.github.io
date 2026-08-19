---
title: "AI를 이용한 레드티밍 — 자율 공격보안(offensive AI)"
date: 2026-07-10T13:00:00+09:00
tags:
  - ai-security
  - red-teaming
  - llm
  - korean
summary: "AI가 표적인 경우와 AI가 도구인 경우는 다르다. 이 글은 후자 — 자율 공격보안(offensive AI)이 2025~26년에 개념증명을 넘어 측정 가능한 실전 역량에 도달한 지점과, 그 능력의 한계를 함께 정리했다."
---

---

> **맥락**: 세 축 중 두 번째. **AI가 표적**(jailbreak/프롬프트 인젝션 → [가이드](/ko/posts/kisa-ai-redteaming-guide/)·[매뉴얼](/ko/posts/kisa-ai-threat-response-manual/))이 아니라, **AI를 공격 도구로 써서 전통(비-AI) 표적을 공격**하는 축. 전통 레드팀 규율은 [전통적 레드티밍](/ko/posts/traditional-red-teaming/).
> **성격**: 연구·시스템·거버넌스 수준 정리(익스플로잇 코드·플레이북 아님). 방어자·SIEM 관점.
> **핵심 한 줄**: 2025~26년 자율 공격 AI는 개념증명을 넘어 **측정 가능한 실전 역량**에 도달했다 — 다만 능력은 비대칭적이고, 병목은 "익스플로잇 작성"이 아니라 여전히 "미지 취약점 발견"에 있다.

## 1. 연구 계보 — 자율 익스플로잇 (Kang 연구실, UIUC)

Fang·Bindu·Gupta·Kang 3부작이 학술 기준점.

| 논문 | arXiv | 결과 |
|---|---|---|
| LLM Agents can Autonomously **Hack Websites** (2024a) | 2402.06664 | GPT-4가 취약점 사전 미인지 상태에서 블라인드 SQLi 등 자율 수행. 오픈소스 모델은 실패 → 툴 사용·긴 컨텍스트가 프론티어 고유 능력 |
| LLM Agents can Autonomously **Exploit One-day Vulns** (2024b) | 2404.08144 | CVE 설명 주어지면 GPT-4가 실세계 one-day 15개 중 **87% 익스플로잇**, 그 외 모든 모델 **0%**. CVE 설명 제거 시 급락 → **병목은 취약점 탐색** |
| **Teams** of LLM Agents can Exploit **Zero-Day** Vulns (2024c) | 2406.01637 | **HPTSA** — Planner+Manager가 XSS/SQLi/CSRF/SSTI 특화 서브에이전트 6종 조율. 사전지식 없이 zero-day급 웹 취약점 팀 익스플로잇, 단일 대비 향상 |

> 용어: **one-day**=공개(CVE)됐으나 미패치, **zero-day**=미공개. 방법론이 단일→팀 오케스트레이션으로 진화.

**현재 도달 수준 (검증)**
- **CAISI 평가**: 프론티어 모델이 2023 **apprentice(1년 미만)** → 2025 **expert(10년+)** 사이버 과제 수행으로 상승.
- **ARTEMIS 실측**(arXiv 2512.09882, 2025-12): ~8,000 호스트 대학망에서 AI vs 인간 10명. **종합 2위, 인간 10명 중 9명 능가**(유효취약점 9건). 단 **오탐률 높고 GUI 과제 취약**(인간 80%가 찾은 RCE 놓침). 비용 시간당 $18 vs 전문가 $60.

## 2. 자율 침투테스트 시스템

**오픈소스/연구**
- **PentestGPT**(NTU) — 초기 대화형 어시스턴트 → 자율 에이전트로 발전
- **HackingBuddyGPT**(IPA Lab) — ~50줄 미니멀 구현. 관련: "LLMs as Hackers: Autonomous Linux Privilege Escalation"(2310.11409)
- **Nebula** — 실제 명령 실행(PentestGPT는 제안만) / **PentAGI** — Docker 샌드박스 다중 서브에이전트

**상용**
- **XBOW** — 2025.4~6 **HackerOne 미국 리더보드 1위**(자율 AI 최초), ~1,060건 제출, 시리즈B $75M.
  > ⚠️ **주의**: "1위"는 **평판 점수** 기준이고 상당수가 VDP의 Triaged(미해결) 상태. "인간 해커 전면 능가"로 확대해석 금지.

**취약점 발견 (방어 지향 포함)**
- **Google Project Naptime→Big Sleep**(Project Zero+DeepMind, 2024-10): 코드브라우저·디버거·샌드박스로 **SQLite 실제 zero-day 발견**(정식 릴리스 전, 피해 無). "AI가 실세계 SW에서 미지 메모리안전 결함 찾은 최초 공개 사례"라 주장하되, 팀 스스로 "고도 실험적, 현시점엔 타깃 특화 퍼저가 최소 동등"이라 단서.
- **DARPA AIxCC**(2025-08 DEF CON 33): 우승 **Team Atlanta(Georgia Tech·삼성·KAIST·POSTECH)의 ATLANTIS**($4M). 멀티에이전트 RL+LLM+심볼릭으로 취약점 자율 탐지·익스플로잇·**패치**. 성과: 합성취약점 54개 발견·**68% 자동패치** + **실제 zero-day 18건**(C 6, Java 12). 7팀 CRS 전부 오픈소스 공개. **방어(자동패치) 지향**이라 순수 공격도구와 구분.

## 3. 공격 하위과제별 LLM 활용 (개념)

- **정찰**: 체계적 열거·병렬 탐색 강점(ARTEMIS 실증), 오탐필터·GUI 약점.
- **피싱/멀웨어/익스플로잇 보조**(측정 도구):
  - **CyberSecEval 1~3**(Meta Purple Llama, Bhatt 2023~): 안전하지 않은 코드 생성 + 사이버공격 조력 준수율 측정. CSE2에 prompt injection·**False Refusal Rate**, CSE3에 자율공격·스피어피싱 평가 확장.
  - **Pa Pa 2023**(CSET): ChatGPT/Auto-GPT로 멀웨어 7종+도구 2종, 안전장치 하에서도 ~90분 내 ~400줄 기능성 멀웨어 생성. Auto-GPT 자동생성 프롬프트가 안전장치 우회 경향.

## 4. 공격 AI 역량 벤치마크

- **Cybench** — 프로급 CTF 40과제, 순수 공격 관점.
- **CyberSecEval 1~3** — 최장수 LLM 사이버 벤치 스위트.
- **BountyBench**(Stanford, 2505.15216) — 실세계 25개 시스템+바운티 40건. **Detect/Exploit/Patch** 3과제로 발견→수리 생애주기 측정, **공수 균형(offense-defense) 평가** + 달러 임팩트 환산.

## 5. 이중용도 거버넌스 — 통제 배포

- **OpenAI Trusted Access for Cyber(TAC)** + **GPT-5.5-Cyber**(2026-05 제한 프리뷰): 검증된 방어자에겐 classifier 거부 완화하되 자격증명 탈취·멀웨어 배포·무단 3자 익스플로잇은 계속 차단. 대상: 정부·핵심인프라·보안벤더·클라우드·금융.
- **Anthropic Project Glasswing / Claude Mythos**: 미공개 프론티어 모델로 주요 OS·브라우저 고위험 취약점 1만+ 발견 주장(AWS·Apple·Google·MS·NVIDIA·JPMorgan 등 참여). **오용 방어책 미확립으로 Mythos급 일반 미공개** = 통제 배포 대표 사례.
  > ⚠️ "1만+", "최숙련 인간 외 모두 능가"는 **Anthropic 자체 발표(레드팀/마케팅 블로그)** — 독립 검증 아님, 벤더 주장으로 취급.
- **CAISI**(NIST 산하): 40+ 평가(미공개 모델 포함), 2026-05 Google DeepMind·MS·xAI로 배포전 평가 협약 확대(기존 OpenAI·Anthropic 포함 5개 랩). 위 apprentice→expert 상승이 이 평가 결과.

## 6. 방어 대응물 — AI SOC (참고, 간략)

- **Microsoft Security Copilot 에이전트**(2025-03): Phishing/Alert Triage Agent — 경보 자율 분류 + 추론근거 제시.
- **CrowdStrike Charlotte AI / Agentic Security Workforce**(2025): Exposure Prioritization·Malware Analysis·Hunt·Correlation Rule Generation 등 7개 에이전트.
- 방향: 수동 SOC 업무 감축·트리아지 자동화, 단 AI 오탐 경향 감안한 사람 검증 루프 유지.

---

## 종합 — 2025~26 현주소 (방어자 관점)

1. 자율 공격 AI는 **측정 가능한 실전 역량**에 도달. AIxCC 실제 zero-day 18건, Big Sleep SQLite zero-day, XBOW HackerOne 1위, ARTEMIS 인간 9/10 능가 — 모두 검증됨. CAISI의 2년 apprentice→expert 상승이 뒷받침.
2. 단 능력은 **비대칭** — 체계적 열거·병렬·비용은 압도적이나 오탐 높고 GUI·창의 판단 약함, 병목은 여전히 **미지 취약점 발견**.
3. 프론티어 랩이 이를 이중용도 위험으로 보고 **통제 배포**(OpenAI TAC·Anthropic Glasswing 비공개·CAISI 배포전 평가)로 대응한다는 사실 자체가 역량의 실재성을 방증.
4. **SIEM/SOC 함의**: 정찰·초기침해의 자동화·저비용·대량화가 현실 → 대량 자동스캔·병렬 익스플로잇의 텔레메트리 특성(빠른 열거·다수 벡터 동시 시도·비인간 속도)을 탐지 로직에 반영. 동시에 Security Copilot·Charlotte AI류 **AI SOC로 대칭 대응**하되 AI 오탐 감안 사람 검증 유지.

> **세 축 총정리**: ① AI가 표적(가이드·매뉴얼) ② AI가 도구(이 글) ③ 전통 대상([전통 레드티밍](/ko/posts/traditional-red-teaming/)). 이 글의 H01(고도화된 사이버 공격 지원)이 매뉴얼의 고성능 모델 위협과 정확히 대응 — 매뉴얼은 "AI가 도구가 되는" 이 위협을 방어자 시각에서 다루고, 이 글은 그 공격 역량의 실체를 정리한다.

---
