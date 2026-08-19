---
title: "AI 레드티밍 실무 — 도구·조직 방법론·규제·커뮤니티"
date: 2026-07-10T15:00:00+09:00
tags:
  - ai-security
  - red-teaming
  - governance
  - korean
summary: "앞선 글들이 무엇을 공격하는가였다면 이 글은 실제로 어떻게 운영하는가다. 도구·조직 방법론·규제·커뮤니티를 정리했다. AI 레드티밍은 인간 전문가의 창의성, 자동화의 커버리지, 표준·규제의 요구사항이 맞물린 자리에 있다."
---

---

> **맥락**: 앞선 글들이 "무엇을 공격하는가(위협·연구 계보)"였다면, 이 글은 **"AI 레드티밍을 실제로 어떻게 수행하는가"** 를 정리한다. 자동화 도구 생태계 → 프론티어 랩의 조직 방법론 → 규제상 의무 → 공개 커뮤니티(GRT) 순. 짝: [멀티모달·에이전트 확장 표면](/ko/posts/multimodal-agent-red-teaming/).
> **성격**: 도구·방법론·거버넌스 수준 정리(익스플로잇 코드·플레이북 아님). 방어자·SIEM 운영 관점.
> **핵심 한 줄**: AI 레드티밍은 "인간 전문가의 창의성 + 자동화의 커버리지 + 표준·규제의 요구사항"이 맞물린 하나의 **실무 규율**로 제도화됐다 — 도구는 이미 성숙했고, 병목은 도구가 아니라 무엇을 위협으로 볼지 정의하는 위협모델링이다.

## 1. 오픈소스 도구 생태계 — 자동 레드티밍 프레임워크

| 도구 | 제작 | 유형 | 핵심 구조 | 주요 커버리지 |
|---|---|---|---|---|
| **PyRIT** | Microsoft (`Azure/PyRIT`) | 프레임워크 | Orchestrator·Target·Converter·Scorer·Memory·Dataset 6모듈 | 멀티턴 jailbreak, 유해콘텐츠, 프롬프트 변형공격 |
| **garak** | NVIDIA (`NVIDIA/garak`) | 스캐너 | Probe·Detector·Generator·Evaluator·Harness (전부 플러그인) | injection, jailbreak, 데이터유출, toxicity, hallucination |
| **promptfoo** | promptfoo → **OpenAI 인수(2026-03)** | eval + red team(DAST) | 프롬프트/모델 비교 + red team 플러그인 | injection, jailbreak, data leak, tool misuse |
| **Giskard** | Giskard-AI | 스캐너(SDK+Hub) | LLM detectors(heuristic + LLM-assisted) | hallucination, injection, bias, 민감정보 |
| **deepteam** | Confident AI (DeepEval 계열) | 프레임워크 | vulnerabilities × attacks(Linear/Sequential/Tree Jailbreak) | 40+ 취약점(PII·bias·injection 등) |

- **PyRIT의 자동화 원리**: orchestrator가 `converter(프롬프트 변형) → target(대상 AI) → scorer(응답 판정)` 루프를 돌려 성공할 때까지 후속 프롬프트를 적응시킴. Microsoft AI Red Team이 2022년 내부 one-off 스크립트로 시작 → **2024-02-22 오픈 프레임워크로 공개**. (arXiv 2410.02828)
- **garak**은 "모든 것이 플러그인" 구조로 static(알려진 익스플로잇 재현)·dynamic(창발적 약점)·adaptive(실패 반복 정교화) 3방식 지원, JSONL/HTML 리포팅. (arXiv 2406.11036)
- **학술 jailbreak 알고리즘**(PAIR·TAP·GCG 등)이 이 도구들의 probe로 채택됨 — GCG는 [가이드 노트](/ko/posts/kisa-ai-redteaming-guide/)의 자동 jailbreak 계보와 직결.

> ⚠️ **promptfoo OpenAI 인수는 사실**(2026-03-09 OpenAI 공식 발표, Frontier 플랫폼에 통합 예정, 오픈소스는 유지). 개발자 수·밸류에이션 등은 벤더/보도 주장.

## 2. 상용 벤더 — AI 보안 시장의 급속한 통폐합

| 벤더 | 포지셔닝 | 인수/현황 |
|---|---|---|
| **Robust Intelligence** | AI Firewall + red teaming | **Cisco 인수(2024)** → Cisco AI Defense (공식 금액 미공개) |
| **Protect AI** | MLSecOps(ModelScan·Recon) | **Palo Alto 인수**, 2025-04 $700M 합의 → 2025-07 완료, Prisma AIRS 통합 |
| **Lakera** | AI-native 런타임 보안 + 적대 테스트 | **Check Point 인수**, 2025-09 $300M 발표 → 2025-11 완료 |
| **HiddenLayer** | AI Detection & Response(AIDR) | 독립 유지 (2023 Series A $50M) |
| **Adversar AI** | 연속형 red teaming, 에이전트·MCP | 독립 유지 |

> **SIEM 운영자 함의**: 2024~26년 대형 보안벤더(Cisco·Palo Alto·Check Point)가 AI 레드팀 스타트업을 잇따라 인수 → AI 보안 테스트가 **독립 틈새에서 통합 보안 스택의 기본 구성요소로 편입**되는 중. 곧 SIEM/SASE 제품에 AI 레드팀·런타임 방어가 번들될 신호.

## 3. 프론티어 랩·빅테크의 조직 방법론

**Microsoft AI Red Team (AIRT)** — 2018 설립(업계 최초 급), security와 **Responsible AI(RAI)** 양쪽 포괄. 백서 **"Lessons From Red Teaming 100 Generative AI Products"**(2025-01, arXiv 2501.07238)의 8교훈 중 방어자에게 특히 중요한 것:
- **"gradient 계산 없이도 AI를 깰 수 있다"** — 정교한 최적화 공격보다 단순 프롬프트 조작이 실전에서 더 효과적.
- **"레드티밍 ≠ safety benchmarking"** — 벤치마크는 알려진 것을 측정, 레드티밍은 미지의 실패를 발견.
- **"자동화가 커버리지를 넓히되, 인간 요소가 핵심"** — 4·5번 교훈이 §5의 인간 vs 자동화 논쟁으로 직결.
- 위협모델 온톨로지 5요소: system under test · actor · TTPs(가능하면 **MITRE ATT&CK/ATLAS 매핑**) · underlying weaknesses · downstream impacts.

**Google** — Secure AI Framework(**SAIF**, 2023-06) 6요소 중 "Adapt controls / continuous testing"의 실천수단이 red teaming. AI Red Team 리포트(2023-07)의 공격 taxonomy 6종: prompt attacks · training data extraction · backdooring · adversarial examples · data poisoning · exfiltration. "전통 레드팀은 좋은 출발점이나 AI는 빠르게 복잡해져 전문지식이 필요"라 명시 — [전통 레드티밍 노트](/ko/posts/traditional-red-teaming/)의 "연장·특수화" 논지와 일치.

**OpenAI** — **External Red Teaming Network**(DALL-E 2 배포 2022부터, GPT-4 사전점검에 외부 레드티머 100+명) + 백서 "OpenAI's Approach to External Red Teaming"(arXiv 2503.16431). 방법론: 위협모델링으로 도메인 우선순위 → 각 영역을 "리스크·표적·출처" 가설로 앵커 → 결과를 risk assessment와 **자동 평가의 입력**으로 환류. 자동 레드티밍 연구 논문을 동시 발표(인간+자동 병행).

**Anthropic** — **Frontier Red Team**이 **CBRN·사이버·자율성** 3도메인 집중, 각 도메인 전문가와 평가법을 공동설계. 핵심은 레드티밍 결과를 **Responsible Scaling Policy(RSP)의 capability threshold(ASL)**와 연동 — 특정 위험역량 도달 시 배포·스케일링을 정책적으로 제약. 블로그 "Challenges in Red Teaming AI Systems"에서 **표준화 부재**를 업계 공통 난제로 지적.

> **공통 패턴**: 4개 조직 모두 (a) 위협모델링으로 시작 (b) security와 RAI/safety harm을 함께 다룸 (c) 인간+자동 병행 (d) 결과를 배포 결정·정책과 연동. AI 레드티밍이 "일회성 테스트"가 아니라 **배포 게이트에 물린 지속 프로세스**로 제도화됐음을 보여줌.

## 4. 규제·표준상 레드티밍 의무

| 관할 | 정확 조문 | 요구 내용 | 강제성/현황 |
|---|---|---|---|
| **EU** | AI Act **제55조(1)(a)** (임계는 **제51조**) | systemic-risk GPAI(**누적학습 >10²⁵ FLOP** 추정 또는 위원회 지정)는 표준 프로토콜에 따른 model evaluation + **adversarial testing 수행·문서화** 의무. Recital 114가 내부/독립 외부 테스트 언급 | 법적 의무, 적용 단계 |
| **미국** | EO 14110 **§3(d)**(정의)·**§4.2(a)(i)(C)**(보고) | 이중용도 파운데이션 모델의 **AI red-team 결과**를 DPA(50 U.S.C. 4501) 근거로 Commerce 보고 | **철회** — EO 14148(2025-01-20)이 일괄 폐지, EO 14179가 후속 정리. 의무 실효 |
| **미국(표준)** | **NIST AI 600-1** GenAI Profile(2024-07), Appendix A.1.5 | red-teaming을 배포 전 위험식별로 규정, **4유형(General Public·Expert·Combination·Human-AI)** 제시, MS-2.5/2.6·MP-5.1 측정액션 | **자발적 가이던스**(비강제), EO 철회와 무관하게 존속 |
| **한국** | 인공지능기본법 **제32조**(대규모)·**제33~35조**대(고영향) | 고성능 AI 안전성 확보 + 고영향 AI 위험관리·human oversight·문서화 | 법적 의무, 2026-01-22 시행 |

> **용어 주의**: **EU AI Act 본문·전문은 "red teaming"이 아니라 "adversarial testing"** 용어를 쓴다(제55조·Recital 114). **한국 인공지능기본법도 조문에 "레드팀" 용어는 없고** "안전성 확보 조치·위험관리방안" 상위 개념으로 규정. 즉 법이 강제하는 것은 "레드티밍"이라는 특정 기법이 아니라 "적대적 테스트/위험평가를 하라"는 결과 의무 — 레드티밍은 그 이행 수단이다.
> **티어 주의**: EU 제55조와 한국 제32조는 **프론티어/systemic-risk 티어에만** 적용(FLOP 임계). 일반 AI에 레드티밍이 법으로 강제되는 건 아니다. [매뉴얼 노트](/ko/posts/kisa-ai-threat-response-manual/)에서 정정한 "제32조 = 고성능/프론티어(≠고영향)" 논점과 동일 구조 — 단 EU 임계는 10²⁵, 한국은 10²⁶ 수준으로 알려졌고 **한국 수치는 법문이 아니라 대통령령 소재일 가능성(미확인)**.
> **미국 정정**: EO 14110 기반 강제 레드팀 보고는 EO 14148(트럼프 취임일 2025-01-20)의 일괄 철회로 실효. "미국이 레드티밍을 법제화했다"는 2026년 시점에 부정확 — NIST 가이드(권고)만 존속.

## 5. 인간 vs 자동화 레드팀 — 방법론 논쟁

- **자동화 강점**: 커버리지·재현성·비용·회귀 테스트(배포마다 반복). PyRIT/garak/deepteam이 대량 프롬프트를 병렬 주입.
- **인간 강점**: 창의적 신규 공격 발견, 맥락·문화 의존 harm(RAI) 판단, "무엇이 문제인가"의 프레이밍. MS 백서 5번 교훈 "인간 요소가 핵심"이 이를 못박음.
- **합의**: 대립이 아니라 **분업** — 자동화가 알려진 공격면을 넓게 훑고(양), 인간이 미지·맥락 위험을 판단(질). MS 4·5번 교훈이 나란히 선 이유. AI 레드팀 결과의 최종 판정에 사람 검증 루프를 유지하는 것이 [AI-도구 노트](/ko/posts/ai-augmented-red-teaming/)의 "AI 오탐 감안" 원칙과 동일.

## 6. 공개 커뮤니티 — DEF CON GRT

**GRT(Generative Red Team) @ DEF CON 31 (2023-08)** — 사상 최대 공개 LLM 레드티밍.
- 참가 2,200+명, 8개 벤더 모델(Anthropic·Cohere·Google·HuggingFace·Meta·NVIDIA·OpenAI·Stability), **Scale AI** 플랫폼, 16.4만 메시지.
- **백악관 지원의 실제 성격**: OSTP가 **자금이 아니라 challenge 설계 기술자문** 제공(Alan Mislove 성명, 2023-08-29). Blueprint for an AI Bill of Rights + NIST AI RMF에 정합. 2023-05 백악관에서 공개 발표.
- Humane Intelligence의 Transparency Report(2024)가 8개 모델 결과 분석.

**GRT2 @ DEF CON 32 (2024-08)** — 질적 전환.
- 단일 사례 → **통계적 flaw report**(대표 데이터셋으로 편향 경향 입증) + **modified CVE 프로세스**로 "vulnerability"를 넘어 model card 기준 "flaw"(보호집단 대상 bias 포함) 보고.
- 대상은 **Ai2의 OLMo** 모델, vendor panel = DSRI+Ai2, UK AISI의 **Inspect AI** 프레임워크로 평가 구조화, 495명·약 200건 flaw report, bounty $7,400 지급.

> **의의**: 레드티밍이 프론티어 랩 내부를 넘어 **공개·참여형 거버넌스 실험**으로 확장. GRT2의 "flaw" 개념(취약점 아닌 통계적 편향까지 포함)은 AI 레드티밍이 전통 보안의 "vulnerability" 틀을 넘어섬을 보여줌 — RAI harm은 CVE로 안 잡힌다.

---

## 종합 — 실무 규율로서의 AI 레드티밍 (방어자 관점)

1. **도구는 이미 성숙**했다(PyRIT·garak·promptfoo·Giskard·deepteam). 병목은 도구가 아니라 **무엇을 위협으로 정의할지의 위협모델링** — MS 백서가 위협모델 온톨로지를 앞세운 이유.
2. **조직 방법론이 수렴**했다: 위협모델링 → security+RAI harm 병행 → 인간+자동 분업 → 배포 게이트·정책(RSP/ASL) 연동. 일회성 테스트가 아니라 지속 프로세스.
3. **규제는 프론티어 티어에만** 레드티밍/안전평가를 요구(EU 제55조·한국 제32조, FLOP 임계). 미국은 EO 14110 철회로 강제 보고 실효, NIST 가이드만 권고로 존속.
4. **시장이 통폐합** 중(Cisco·Palo Alto·Check Point의 인수 러시) → AI 레드팀·런타임 방어가 SIEM/보안 스택에 번들될 방향.
5. **SIEM/SOC 접점**: garak·PyRIT 같은 스캐너의 결과를 OWASP LLM Top 10·MITRE ATLAS ID로 태깅하면, 전통 SIEM의 ATT&CK 태깅 탐지 엔지니어링([전통 노트](/ko/posts/traditional-red-teaming/) §7)과 동일한 커버리지·공백 관리 워크플로우를 AI 시스템에도 적용 가능.

> **세 축 + 실무 층 지도**: ① AI가 표적([가이드](/ko/posts/kisa-ai-redteaming-guide/)·[매뉴얼](/ko/posts/kisa-ai-threat-response-manual/)) ② AI가 도구([offensive AI](/ko/posts/ai-augmented-red-teaming/)) ③ 전통 대상([전통 레드티밍](/ko/posts/traditional-red-teaming/)) — 이 글은 세 축 전반을 관통하는 **"어떻게 수행하는가"의 실무·거버넌스 층**이다.

---
