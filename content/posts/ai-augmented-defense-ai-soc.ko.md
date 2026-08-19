---
title: "AI를 이용한 방어 — AI SOC와 퍼플팀"
date: 2026-07-13T12:00:00+09:00
tags:
  - ai-security
  - blue-team
  - siem
  - soar
  - korean
summary: "AI를 이용한 방어 — 2025년 AI SOC는 벤더 데모를 넘어 트리아지·헌팅·룰 생성을 실제로 자동화하는 제품으로 나왔다. offensive AI의 정확한 방어 대응물로서 무엇이 되고 무엇이 아직 안 되는지, 퍼플팀 관점까지."
---

---

> **맥락**: 블루팀 3축 중 세 번째. [AI를 이용한 레드티밍(offensive AI)](/ko/posts/ai-augmented-red-teaming/)의 정확한 방어 대응물 — **AI를 공격 도구가 아니라 방어 도구로 써서 SOC·탐지·대응을 자동화**하는 축. 전통 방어는 [전통적 블루티밍](/ko/posts/traditional-blue-teaming/), AI 시스템 자체 방어는 [AI 시스템 방어](/ko/posts/ai-system-defense/).
> **성격**: 제품·연구·거버넌스 수준 정리. 방어자·SIEM 관점.
> **핵심 한 줄**: 2025년 AI SOC는 벤더 데모를 넘어 **트리아지·헌팅·룰 생성의 실제 자동화 제품**으로 출시됐다 — 다만 강점(24/7·대량·속도)만큼 한계(자동화 편향·컨텍스트 결여)가 뚜렷해, 합의는 여전히 **human-in-the-loop 유지**다.

## 1. AI SOC 제품 — 2025년 트리아지 자동화의 해

**Microsoft Security Copilot 에이전트**(2025-03-24 발표, MS 자체 6종 + 파트너 5종):

| 에이전트 | 제품 | 자동화 대상 |
|---|---|---|
| Phishing Triage → **Alert Triage로 확장** | Defender | 신고 피싱/알림 트리아지, 오탐 자동 제거 |
| Alert Triage | Purview | DLP·내부자 위험 알림 분류 |
| Conditional Access Optimization | Entra | 조건부 접근정책 공백 탐지 |
| Vulnerability Remediation | Intune | 취약점 우선순위·패치 |
| Threat Intelligence Briefing | Security Copilot | 조직 맞춤 위협 인텔 큐레이션 |

> 벤더 주장: Phishing Triage가 신고 오탐의 95%+를 자동 정리(통제 평가 기준).

**CrowdStrike Charlotte AI** — **Detection Triage**(2025-02-12 발표) "고객 정의 경계 자율성" 하 탐지 자율 분석. **Agentic Security Workforce 7종**(2025-09-16): Exposure Prioritization·Malware Analysis·Hunt·Data Transformation·Search Analysis·**Correlation Rule Generation**·Workflow Generation. 자연어로 커스텀 에이전트 생성(AgentWorks). RBAC·감사로그·설명가능성 가드레일로 human-in-the-loop 표방.
> 벤더 주장: Detection Triage 정확도 98%+, 주당 40시간+ 절감.

**Google** — **Sec-Gemini v1**(2025-04, 실험) = Gemini + Google Threat Intelligence + OSV + Mandiant 결합해 근본원인분석·위협분석 지원. **Google SecOps**(구 Chronicle)에 Gemini 통합(자연어 조사). **Big Sleep**(Project Zero+DeepMind)은 [offensive AI 노트](/ko/posts/ai-augmented-red-teaming/)에 등장했으나 **본질은 방어** — 공격자보다 먼저 취약점 발견·차단(SQLite CVE-2025-6965 악용 임박 상태 사전 차단, OSS 신규 20건).

> ⚠️ 정확도·시간절감 수치는 전부 **벤더 주장**(통제 환경). 실제 성능은 환경 의존.

## 2. AI SOC 스타트업 — alert fatigue가 낳은 시장

자율 SOC 애널리스트(Dropzone AI, Prophet Security 등)는 **알림 피로 + Tier 1 트리아지 병목**을 배경으로 등장. playbook 없이 재귀 추론으로 알림을 자율 조사하고 증거체인·결정 리포트를 생성한다고 표방([전통 블루팀 §6 alert fatigue](/ko/posts/traditional-blue-teaming/)의 직접 대응). 업계 인용: SOC ~90%가 백로그·오탐에 압도, 평균 일 960건 알림 — 단 대부분 **벤더/설문 주장**으로 방법론 확인 필요.

## 3. 자율 방어 역량 — 측정의 초기 신호

- **DARPA AIxCC**: 자동 취약점 발견 + **자동 패치(방어)** 대회 — [offensive AI 노트](/ko/posts/ai-augmented-red-teaming/)의 ATLANTIS가 68% 자동패치. 순수 공격도구와 구분되는 방어 지향.
- **BountyBench**(Stanford, 2505.15216): Detect/Exploit/**Patch** 3과제. 주목할 결과 — **현행 코딩 에이전트는 공격(Exploit)보다 방어(Patch)에서 상대적 고득점**(Codex CLI Patch 90% vs Exploit 32.5%). 방어 AI 역량이 공격보다 앞선다는 초기 신호.

> **함의**: [offensive AI 노트](/ko/posts/ai-augmented-red-teaming/)가 "공격 AI의 실전 역량 도달"을 다뤘다면, 같은 기술이 **방어(패치·탐지)에서 오히려 더 잘 작동**할 수 있다는 것. 비대칭이 방어에 유리하게 작용하는 영역.

## 4. 퍼플팀 — 레드·블루를 잇는 검증 루프

- **VECTR**(Security Risk Advisors): 퍼플팀 캠페인을 MITRE ATT&CK에 매핑해 추적하는 협업 플랫폼. 각 Test Case의 수행 시점·탐지 여부를 기록, **Threat Resilience Metrics**로 탐지·차단 역량을 ATT&CK 기준 정량화. 공격 실행 도구가 아니라 **결과 추적·측정 계층**.
- **BAS(Breach and Attack Simulation)**: 자동 공격 시뮬레이션으로 방어 검증([전통 레드팀 §CTEM](/ko/posts/traditional-red-teaming/)의 Validation 담당). VECTR이 그 결과를 ATT&CK로 정량 추적하는 상보 관계.
- 퍼플팀은 [전통 블루팀](/ko/posts/traditional-blue-teaming/)의 탐지 엔지니어링 피드백 루프(공격 실행→탐지 로깅→룰 개선)를 **실시간으로 단축**하는 기능 — 별도 팀이 아니라 협업 방식.

## 5. 블루팀 자격·커뮤니티

| 인증/플랫폼 | 성격 |
|---|---|
| **BTLO**(Blue Team Labs Online) | 포렌식·IR·SOC 게임화 실습 랩 |
| **BTL1 / BTL2**(Security Blue Team) | 실습 중심 방어자 인증(BTL1 24h 랩·20문항, BTL2 72h 상급 시나리오) |
| **GIAC Cyber Defense**(GCIA·GCDA 등) | 방어 인증군(세부 명칭은 부분 미확인) |

> 레드팀의 OSCP/공격 인증에 대응하는 방어자 실습 인증 생태계. 실습(cyber range) 중심이라는 점이 특징.

## 6. 인간 vs AI 방어자 — 여전히 human-in-the-loop

| AI SOC 강점 | 한계 |
|---|---|
| 24/7 가동, 대량·고속 트리아지, 오탐 필터 | 신뢰도 높아 보이는 **오탐**, 비즈니스 컨텍스트 결여(운영 vs 테스트 구분 불가) |
| 일관성·확장성, Tier 1 반복작업 대체 | **자동화 편향**(한 연구 애널리스트 47% 영향), 설명가능성·신뢰보정 문제, **과의존→기술 퇴화** 우려 |

문헌 합의는 **human oversight 불가결**. 이는 [offensive AI 노트](/ko/posts/ai-augmented-red-teaming/)의 "AI 오탐 감안 사람 검증 유지"와 [AI 시스템 방어](/ko/posts/ai-system-defense/)의 "가드레일은 경계가 아니다"와 동일한 원칙 — **AI는 증폭기이지 대체재가 아니다**.

---

## 종합 — 대칭 대응, 그러나 비대칭 (방어자 관점)

1. **공격 AI에 방어 AI로 대칭 대응**이 2025년 현실화(Security Copilot·Charlotte AI·Sec-Gemini). [offensive AI](/ko/posts/ai-augmented-red-teaming/)의 자동 정찰·대량화에 AI SOC의 자동 트리아지·헌팅으로 맞선다.
2. 단 **비대칭이 방어에 유리한 영역**도 있다 — BountyBench에서 코딩 에이전트가 공격보다 패치를 잘하고, Big Sleep은 공격자보다 먼저 취약점을 막는다.
3. **한계는 공격 AI와 동일**: 오탐·컨텍스트 결여·자동화 편향. 그래서 결론도 동일 — **human-in-the-loop 유지**.
4. **퍼플팀이 세 팀을 잇는다**: 레드([offensive AI](/ko/posts/ai-augmented-red-teaming/))가 만든 공격 시나리오를 BAS로 실행, VECTR로 탐지 여부를 ATT&CK 기준 측정, 그 공백을 블루([전통적 블루티밍](/ko/posts/traditional-blue-teaming/))가 룰로 메운다.
5. **SIEM/SOC 접점**: AI SOC 제품은 기존 SIEM/SOAR([전통 블루팀 §5](/ko/posts/traditional-blue-teaming/))의 자동화 진화 연장선 — SOAR 플레이북 위에 LLM 추론을 얹은 것. 도입 시 감사로그·설명가능성·사람 승인 게이트를 반드시 유지.

> **블루팀 3축 + 레드팀 3축 대칭 총정리**:
> | | 레드(공격) | 블루(방어) |
> |---|---|---|
> | AI가 표적 | [가이드](/ko/posts/kisa-ai-redteaming-guide/)·[매뉴얼](/ko/posts/kisa-ai-threat-response-manual/)·[멀티모달/에이전트](/ko/posts/multimodal-agent-red-teaming/) | [AI 시스템 방어](/ko/posts/ai-system-defense/) |
> | AI가 도구 | [offensive AI](/ko/posts/ai-augmented-red-teaming/) | **AI SOC(이 글)** |
> | 전통 대상 | [전통적 레드티밍](/ko/posts/traditional-red-teaming/) | [전통적 블루티밍](/ko/posts/traditional-blue-teaming/) |
> 실무·거버넌스 층은 [AI 레드티밍 실무](/ko/posts/ai-red-teaming-practice/)가 공격·방어 양측을 관통.

---
