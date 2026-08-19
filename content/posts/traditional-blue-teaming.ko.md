---
title: "전통적(비-AI) 블루티밍 — SOC·탐지 엔지니어링·DFIR·헌팅·CTI"
date: 2026-07-13T10:00:00+09:00
tags:
  - security
  - blue-team
  - siem
  - detection-engineering
  - korean
summary: "SOC·탐지 엔지니어링·DFIR·헌팅·CTI를 하나로 정리했다. 블루팀의 성숙도는 알럿을 얼마나 많이 만드는가가 아니라, 공격자 TTP를 어느 추상화 수준에서 탐지하는가(Pyramid of Pain의 위쪽)로 결정된다."
---

---

> **맥락**: [전통 레드티밍](/ko/posts/traditional-red-teaming/)의 방어 대응물. 레드팀이 "공격자처럼 사고해 통제를 검증"한다면, 블루팀은 그 검증 대상인 **탐지·대응 체계 자체를 설계·운영**한다. 블루팀 3축 중 첫 번째: **전통 대상 / AI 시스템 방어 / AI를 도구로 쓰는 방어**. 짝: [AI 시스템 방어](/ko/posts/ai-system-defense/)·[AI 활용 방어](/ko/posts/ai-augmented-defense-ai-soc/).
> **성격**: 프레임워크·표준·방법론 수준 정리. SIEM 실무 운영 관점.
> **핵심 한 줄**: 블루팀의 성숙도는 "얼마나 많은 알럿을 만드는가"가 아니라 **"공격자 TTP를 어느 추상화 수준에서 탐지하고(Pyramid of Pain 위쪽), 그 탐지를 코드처럼 관리하며(Detection-as-Code), 인텔로 끊임없이 개선하는가(F3EAD 루프)"** 로 측정된다.

## 1. 방어의 공통 언어 — D3FEND / Engage

레드팀의 ATT&CK/Kill Chain에 대응하는 방어 지식체계.

- **MITRE D3FEND** — ATT&CK의 방어 짝. NSA·국방부 자금, **2025-01-16 정식 1.0 릴리스**(현재 1.4.0). 7전술: **Model·Harden·Detect·Isolate·Deceive·Evict·Restore**(Restore가 1.0 신규). 핵심은 ATT&CK처럼 공격↔방어를 1:1 표로 잇는 게 아니라, 양쪽이 공통 참조하는 **Digital Artifact Ontology를 매개로 추론적(inferential) 매핑**하는 **지식그래프**라는 점 — SPARQL 질의·도구 통합이 설계 목표. Detect가 최대 카테고리(v1.3.0 기준 90개).
- **MITRE Engage**(구 **Shield**) — 기만(deception)·적대자 관여 프레임워크. Shield(2020)를 커뮤니티 피드백 후 대체, **v1.0 2022-02-28**. 매트릭스 5열: **Prepare**(입력)·**Expose**(고신뢰 탐지로 노출)·**Affect**(적 작전 방해)·**Elicit**(적 TTP 정보 유도)·**Understand**(출력·분석). 단 ATT&CK 대비 유지보수 활동은 낮은 편(기부 프로그램 의존).

> D3FEND의 Deceive 전술과 Engage가 겹치는 지점이 능동방어(active defense)·허니팟 영역. 레드팀 [C2 프레임워크](/ko/posts/traditional-red-teaming/) 시그니처가 D3FEND의 Detect 기법으로 직결.

## 2. 탐지 엔지니어링 — 탐지를 코드처럼

- **Sigma** — SIEM 중립 **범용 탐지 시그니처 포맷**(YAML). 2017 Florian Roth·Thomas Patzke. "로그의 Snort/YARA". 변환은 구형 `sigmac`→**pySigma + sigma-cli**로 세대교체, **Spec v2.0(2024-08)**에서 correlation·taxonomy 추가. Detection-as-Code의 사실상 교환 포맷.
- **Palantir ADS**(Alerting & Detection Strategy) — 각 탐지를 Goal·Categorization(ATT&CK 매핑)·Strategy Abstract·Technical Context·**Blind Spots and Assumptions**·False Positives·Validation·Priority·Response로 강제 문서화하는 "알럿 품질 게이트". 특히 Blind Spots 항목이 탐지의 한계를 명시하게 강제.
- **Detection-as-Code** — 탐지룰에 Git 버전관리·코드리뷰·CI/CD 테스트·유닛테스트 적용(공식 표준 없는 업계 관행).
- **성숙도 모델**: Ryan Stillions **DML**(2014, DML-0 원자지표~DML-8 적의 목표) — "무엇을 탐지할 수 있는가"의 추상화 수준으로 조직 역량 측정.

> **SIEM 운영자 관점**: Sigma 룰을 Git으로 관리하고 ADS 양식으로 각 상관룰의 blind spot·오탐 조건·검증법을 문서화하면, 룰 유지 비용(SIEM의 만성병)이 통제 가능해진다. 이것이 [레드팀→블루팀 피드백 루프](/ko/posts/traditional-red-teaming/)의 블루팀 측 구현.

## 3. 위협 헌팅 — 가설로 미탐을 사냥

| 방법론 | 출처 | 구조 |
|---|---|---|
| **PEAK** | Splunk SURGe(D. Bianco 참여), 2023 | **P**repare–**E**xecute–**A**ct with **K**nowledge + 헌트 3유형: 가설기반 / Baseline(EDA) / **M-ATH**(ML 지원) |
| **TaHiTI** | 네덜란드 금융 FI-ISAC, 2018 | Initiate→Hunt(Define/Refine+Execute)→Finalize. CTI가 헌트를 촉발하고 헌트가 CTI를 생산하는 양방향 |
| **Sqrrl 헌팅 루프** | Sqrrl(→AWS 인수), ~2015 | 가설→조사→신규 TTP 발견→분석 자동화 순환 |
| **HMM** | David Bianco, 2015 | HM0(Initial)~HM4(Leading), 데이터 수집·절차 반복성·자동화가 축 |

- **Pyramid of Pain**(Bianco, 2013): Hash→IP→Domain→Network/Host Artifacts→Tools→**TTPs**, 위로 갈수록 공격자 교체비용(=고통)↑. **TTP 수준 탐지를 지향하라**는 논거 — 탐지 엔지니어링·CTI·헌팅의 공통 기반. 레드팀 [ATT&CK 시나리오](/ko/posts/traditional-red-teaming/)가 곧 이 피라미드 상단.

## 4. DFIR 표준 — NIST 800-61 대전환

- **NIST SP 800-61 Rev.3(2025-04-03)** 가 Rev.2(2012)를 **공식 대체**. 제목이 "Computer Security Incident Handling Guide"→**"IR Recommendations and Considerations for Cybersecurity Risk Management: A CSF 2.0 Community Profile"**로 바뀌며, **고정 4단계 라이프사이클(Preparation→Detection&Analysis→Containment/Eradication/Recovery→Post-Incident)을 폐기**하고 IR 활동을 **CSF 2.0의 6기능(Govern/Identify/Protect/Detect/Respond/Recover)에 매핑**하는 구조로 전환. 상세 how-to는 단일 정적 문서로 유지 불가라 삭제.
- **SANS PICERL(6단계)**: Preparation→Identification→Containment→Eradication→Recovery→Lessons Learned. NIST Rev.2 4단계와 실질 동일하되 봉쇄·박멸·복구를 분리·순환 명시. 현장 런북은 PICERL, 거버넌스는 NIST 이원 운용이 일반적.

> **정정 주의**: 2025-04 이후 "NIST IR 4단계"를 인용하면 **구버전(Rev.2) 기준**임을 명시해야 한다. Rev.3은 절차가 아니라 CSF 2.0 프로파일이다.

## 5. SIEM→SOAR→XDR→NDR 진화 (AI SOC 직전까지)

| 시점 | 사건 |
|---|---|
| 2005 | Gartner(Williams·Nicolett)가 SIM+SEM=**SIEM** 용어 최초 사용 |
| ~2013 | Chuvakin(Gartner) **EDR** 용어 정립(통설) |
| 2015→2017 | Gartner **SOAR** 재정의(Security Orchestration, Automation and Response) |
| 2018 | Palo Alto Nir Zuk **XDR** 창안(EDR을 다층 텔레메트리로 확장) |
| 2020-06 | Gartner가 NTA→**NDR** 개명(response가 핵심 기능화) |

> 동력: SIEM의 만성병(룰 유지비용·컨텍스트 없는 대량 알럿)이 → 대응 자동화(SOAR)·수직 통합 탐지(XDR)·암호화 시대 네트워크 보완재(NDR)로 분화. 이 자동화 축적이 [AI SOC](/ko/posts/ai-augmented-defense-ai-soc/) 담론의 직전 단계다.

## 6. SOC 운영 모델 — 성숙도와 알럿 피로

- **SOC-CMM**(Rob van Os, 2016) — 사실상 오픈 표준. 5도메인: Business·People·Process·Technology·Services(뒤 2개는 성숙도+역량 이원 측정).
- **Tiered vs Tierless 논쟁**: 전통 Tier 1(트리아지)–2(조사)–3(헌팅) 계층은 핸드오프 컨텍스트 손실 + Tier 1 번아웃이 문제. 대안 tierless(flat) SOC는 분석가가 사건을 끝까지 소유.
- **Alert fatigue**: SANS 2025 조사에서 73%가 오탐을 최대 과제로 지목(설문 주장). 이 문제의식이 헌팅·자동화 중심 재설계와 AI SOC의 배경.

## 7. CTI — 인텔이 방어를 이끈다

- **수준**: Strategic(경영진)·Operational(임박 공격)·Tactical(TTP). 소스별로 Tactical=TTP vs =IOC로 뒤집혀 인용 시 정의 명시 필요.
- **STIX/TAXII**: OASIS 표준, **STIX 2.1·TAXII 2.1이 2021-06-10 승인**(JSON 기반 SDO/SCO/SRO 모델).
- **Diamond Model**(2013): 침입을 **Adversary–Capability–Infrastructure–Victim** 4정점으로 모델링, activity thread로 캠페인 상관.
- **F3EAD**: Find-Fix-Finish(작전) + Exploit-Analyze-Disseminate(인텔) 결합. 저서 *Intelligence-Driven Incident Response*(Roberts·Brown)가 IR↔CTI 상호강화 루프로 대중화.

---

## 종합 — 레드팀과의 관계, 그리고 AI로의 확장

전통 블루팀을 규율로 만드는 것은 도구(SIEM/EDR)의 보유가 아니라, **공격자 TTP를 높은 추상화 수준에서 탐지하고(D3FEND·Pyramid of Pain), 그 탐지를 코드처럼 관리하며(Sigma·ADS·Detection-as-Code), 인텔로 끊임없이 개선하는(CTI·F3EAD·헌팅) 폐루프**다. 레드팀([전통적 레드티밍](/ko/posts/traditional-red-teaming/))이 이 루프의 요구사항 공급처라면, 블루팀은 그 요구를 탐지룰·플레이북·헌트 가설로 구현하는 실행부다 — **퍼플팀**([ai-augmented-defense §퍼플팀](/ko/posts/ai-augmented-defense-ai-soc/))이 둘을 실시간으로 잇는다.

> **AI 대상 방어와의 관계**: 이 전통 규율의 정신(TTP 중심 탐지·심층방어·폐루프 개선)은 AI 시스템 방어로 확장되지만, 표적이 실행파일·네트워크가 아니라 **모델·프롬프트·데이터 경계**라 탐지 대상과 통제 계층은 상당 부분 새로 설계된다([AI 시스템 방어](/ko/posts/ai-system-defense/)). MITRE도 ATT&CK→**ATLAS**, D3FEND→**ATLAS Mitigations**로 AI판을 별도 구축했다. 즉 **AI 블루팀 = 전통 블루팀의 연장이자 특수화**.

---
