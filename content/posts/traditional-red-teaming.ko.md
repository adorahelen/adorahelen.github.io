---
title: "전통적(비-AI) 레드티밍 — 방법론·표준·블루팀 연결"
date: 2026-07-10T12:00:00+09:00
tags:
  - security
  - red-teaming
  - siem
  - korean
summary: "AI 레드티밍을 이해하려면 전통 레드티밍이 무엇이었는지가 먼저다. 방법론·표준·블루팀 연결을 정리했다. 레드팀의 성공 기준은 취약점 발견이 아니라, 조직의 사람·프로세스·기술로 이뤄진 탐지·대응 체계가 실제로 작동하는지다."
---

> 🇺🇸 **[English version of this post →](https://adorahelen.github.io/posts/traditional-red-teaming/)**

---

> **맥락**: AI 대상 레드티밍 노트([레드티밍 가이드](/ko/posts/kisa-ai-redteaming-guide/) / [위협 대응 매뉴얼](/ko/posts/kisa-ai-threat-response-manual/))가 "AI가 표적"인 반면, 이 글은 그 뿌리인 **전통적 공격보안(offensive security) 규율**을 정리한다. 세 축 중 첫 번째: **AI가 표적 / AI가 도구 / 전통 대상**.
> **성격**: 개념·방법론 수준 정리(실전 익스플로잇 코드·공격 플레이북 아님). 방어자·SIEM 운영 관점의 지식 정리.
> **핵심 한 줄**: 레드팀의 성공 기준은 "취약점 발견"이 아니라 "조직의 사람·프로세스·기술로 이뤄진 탐지·대응 체계가 실제로 작동하는가"다.

## 1. 용어 지형 — 무엇이 무엇과 다른가

| 활동 | 초점 | 익스플로잇 | 범위 |
|---|---|---|---|
| 취약점 평가(VA) | 취약점 **나열** | ✗ | 광범위 스캔 |
| 침투 테스트(Pentest) | 취약점 **악용 입증** | ○ | 넓은 커버리지, 규칙 명시 |
| 적대행위 모방(Adversary Emulation) | 특정 위협행위자 TTP **재현** | ○ | 깊이(tradecraft) 중심 |
| 레드팀(Red Teaming) | **목표 지향 end-to-end 작전** | ○ | 시나리오 기반, 탐지·대응 검증 |

- **레드팀 / 블루팀 / 퍼플팀**: 레드=공격(TTP 모방), 블루=방어(탐지·IR·취약점 관리), **퍼플=별도 팀이 아니라 공격을 실행하며 탐지 로직을 즉시 검증·개선하는 협업 기능**.

## 2. 공통 언어 — ATT&CK / Kill Chain

- **MITRE ATT&CK**: 실제 위협 인텔에서 도출한 공격자 TTP 지식베이스. **Tactic(왜/목표)** × **Technique(어떻게)**. Enterprise는 14전술·216기술·475서브기술(Win·macOS·Linux·SaaS·IaaS·컨테이너·ESXi), Mobile·ICS 매트릭스 별도. (v18~v19 기준, Defense Evasion이 Stealth/Defense Impairment로 분리되는 등 지속 개정)
- **MITRE Caldera**: ATT&CK 기술 ID로 **adversary profile**을 정의하고 agent를 배치해 자동 적대 모방. Ability(기술 실행 단위)를 환경 학습에 따라 동적 선택 → 공격자 적응 행동 모사.
- **Cyber Kill Chain**(Lockheed 2011, 7단계): Recon→Weaponization→Delivery→Exploitation→Installation→C2→Actions on Objectives. 한 단계 차단으로 전체 실패 유도. 한계: 경계 침투 이후(내부 이동)를 약하게 다룸.
- **Unified Kill Chain**(2017, Pols): Lockheed + ATT&CK 통합 **18단계**. post-exploitation(측면이동·자격증명탈취) 포함, APT 복합공격 표현에 적합.

## 3. 침투 테스트 표준

| 표준 | 성격 |
|---|---|
| **PTES** | 가장 완결적 engagement 7단계 (사전협의→정보수집→위협모델링→취약점분석→익스플로잇→포스트→보고) |
| **OSSTMM** | 과학·지표 기반, 네트워크·물리·무선·인간 채널 측정, **RAV** 정량점수 |
| **OWASP WSTG** | 웹/API 상세 체크리스트 |
| **NIST SP 800-115** | 기술 시험·평가 가이드, 규제·연방 환경 |
| **PCI-DSS Req.11** | 카드데이터환경 연 1회+ 내·외부 펜테스트, 세그멘테이션 검증 |

> 실무는 단일표준이 아니라 **PTES(전체 프로세스) + OWASP WSTG(웹 기술 체크리스트)** 식으로 조합.

## 4. Engagement 유형

- **접근 수준**: 블랙박스(사전정보 無, 최현실적) / 그레이박스(일부 정보, 정기·컴플라이언스에 유리) / 화이트박스(소스·문서 완전 지식). → AI 레드티밍의 블랙/그레이/화이트박스가 여기서 온 개념.
- **Assumed-breach(가정된 침해)**: 발판 확보 상태에서 시작 → **초기 침투가 아니라 탐지·대응(내부 확산 차단)** 검증에 집중.
- **공격 벡터**: 사회공학(피싱·전화·SNS), 물리 침입(RFID 복제·출입통제 우회), 네트워크 측면이동.

## 5. C2(Command & Control) 프레임워크 — 개념 수준

침해 시스템과의 통신 유지·post-exploitation 지원 플랫폼. 구조: **implant/beacon**(주기 콜백) + **team server**(제어) + **transport**(HTTP/DNS) + **malleable profile**(트래픽 위장).
- **Cobalt Strike**(상용, 사실상 표준) / **Metasploit**(오픈소스 익스플로잇 프레임워크) / **Sliver·Mythic·Havoc·Brute Ratel**(모듈러 대안, 2025~26 부상).
- 방어 측에는 이들의 **통신·행동 시그니처가 탐지 엔지니어링의 주요 대상**.

## 6. 현대적 진화

- **CTEM**(Gartner 2022): Scoping→Discovery→Prioritization→Validation→Mobilization 5단계. CVE 주기스캔을 넘어 오설정·아이덴티티·과도권한·자격증명유출까지 지속 관리.
- **BAS(Breach and Attack Simulation)**: 적대행위 자동 모방으로 탐지 통제 검증 → CTEM의 Validation 담당.
- **TIBER-EU / CBEST**: 금융권 인텔리전스 기반 레드팀. 실운영 시스템 대상, 소수 **White Team**만 인지해 실제 탐지·대응 평가. EU **DORA의 TLPT(Threat-Led Penetration Testing)** 요건과 연계.

## 7. 레드팀 → 블루팀 / SOC / SIEM (핵심 연결)

전통 레드팀은 통제 검증의 **입력**, SOC/탐지 엔지니어링은 개선의 **출력**으로 맞물린다.
- **ATT&CK 기반 탐지 엔지니어링**: 탐지룰을 기술 ID로 태깅 → 커버리지·공백 가시화. 레드팀은 기술 체인(Initial Access→Execution→Credential Access→Lateral Movement→Exfiltration)으로 시나리오 설계.
- **피드백 루프**: debrief에서 탐지 성공/누락을 ATT&CK에 매핑 → **탐지 공백 백로그** → 블루팀 룰 보강. 퍼플팀이 이 루프를 실시간 단축.
- **SIEM 구현**: Splunk·QRadar·Microsoft Sentinel·Elastic이 상관룰에 ATT&CK ID 태깅 지원. MITRE **CAR**(Cyber Analytics Repository) 같은 정렬 분석 라이브러리로 가속.

> **SIEM 운영자 관점**: 레드팀/BAS는 "우리 SIEM이 무엇을 탐지해야 하는가"를 ATT&CK 기술 단위로 정의해주는 요구사항 공급처. C2 프레임워크의 beacon 주기·malleable 프로파일·측면이동 텔레메트리가 곧 탐지룰 대상.

---

## 종합 — AI 레드티밍과의 관계

전통 레드팀을 규율로 만드는 것은 취약점 나열이나 컴플라이언스 체크가 아니라, **실제 위협행위자 TTP를 목표 지향으로 재현하는 end-to-end 작전**이고 그 성공 기준이 "탐지·대응 체계가 작동하는가"라는 점이다. 이 분야는 ATT&CK/Kill Chain(공통 언어), PTES·OSSTMM·NIST 800-115·TIBER-EU(표준·규제), 레드→블루→퍼플(검증-개선 루프)로 구조화돼 있다.

**AI 대상 레드티밍(jailbreak·프롬프트 인젝션)은 이 규율의 정신(적대적 사고·목표 기반 시나리오·통제 검증)을 새 공격 표면(모델·프롬프트·데이터 경계)으로 확장한 하위 분야**다. 방법론 계보는 이어지지만(블랙/그레이/화이트박스, 페르소나=위협행위자 모델, 교전규칙=engagement rules), 대상이 실행파일·네트워크·자격증명이 아니라 모델 행동·언어 입력이라 도구·전술 계층은 상당 부분 별개로 발전한다. 요컨대 **AI 레드팀 = 전통 레드팀의 연장이자 특수화**이지 단절된 별개 실천이 아니며, 둘 다 "방어 검증을 위해 공격자처럼 사고한다"는 동일 원리를 공유한다.

---
