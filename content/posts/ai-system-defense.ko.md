---
title: "AI 시스템 방어 — 'AI가 표적'인 공격에 대한 블루팀 통제"
date: 2026-07-13T11:00:00+09:00
tags:
  - ai-security
  - blue-team
  - llm
  - korean
summary: "AI가 표적인 공격에 블루팀은 무엇으로 대응하는가. 결론부터 말하면 가드레일과 필터는 확률적 완화 계층이지 보안 경계가 아니다 — 우회가 실증으로 반복 확인되기 때문이다. 그럼 경계는 어디에 두어야 하는지를 정리했다."
---

> 🇺🇸 **[English version of this post →](https://adorahelen.github.io/posts/ai-system-defense/)**

---

> **맥락**: 블루팀 3축 중 두 번째. [가이드](/ko/posts/kisa-ai-redteaming-guide/)·[매뉴얼](/ko/posts/kisa-ai-threat-response-manual/)·[멀티모달/에이전트](/ko/posts/multimodal-agent-red-teaming/) 노트가 "AI를 어떻게 공격하는가"였다면, 이 글은 **그 공격(jailbreak·프롬프트 인젝션·데이터 오염·모델 공급망)을 어떻게 방어하는가**를 정리한다. 전통 방어 규율은 [전통적 블루티밍](/ko/posts/traditional-blue-teaming/).
> **성격**: 방어 통제·연구 수준 정리(공격 코드 아님). 방어자·SIEM 관점.
> **핵심 한 줄**: 가드레일·필터는 **확률적 완화 계층이지 보안 경계가 아니다** — 우회가 실증으로 반복 확인됐다. 유효한 방어는 단일 필터가 아니라 **아키텍처적 신뢰경계 분리 + 최소권한 + 출력 처리 + 가드레일 + 로깅의 심층방어**이며, 그 설계 원칙이 Willison의 **lethal trifecta**다.

## 1. 가드레일 — 모델 앞뒤의 검사 계층

세 계열은 구조가 다르다: NeMo는 프로그래머블 정책 파이프라인, Guardrails AI는 검증기(validator) 조합, Purple Llama는 분류기 모델 자체.

| 도구 | 제공 | 적용 지점 | 무엇을 차단 |
|---|---|---|---|
| **NeMo Guardrails** | NVIDIA(OSS) | 입력·출력·대화·검색·실행 **5종 레일** | 입력: jailbreak 탐지·인젝션 필터·민감정보 마스킹 / 출력: 정책위반·민감정보 제거 / RAG 청크·도구호출도 검사 |
| **Guardrails AI** | Guardrails AI(OSS) | 입력 Guard + 출력 Guard | Hub의 validator 60여종 조합(PII=Presidio 기반·독성·환각·구조검증). 구조화 출력 강제 강점 |
| **Llama Guard** | Meta(Purple Llama) | **입력+출력** | LLM 기반 안전 분류기(위험 taxonomy 판정). Llama Guard 3는 사이버공격 조력 응답도 탐지 |
| **Prompt Guard** | Meta | **입력 전용** | 소형 BERT계 분류기로 인젝션·jailbreak를 메인 모델 도달 전 탐지(간접 인젝션 겨냥) |
| **Code Shield** | Meta | **출력 전용** | 추론시점 생성 코드의 취약점을 정적분석으로 플래그(코드 인터프리터 남용 방지) |

## 2. LLM 방화벽 / 런타임 보호 — 시장은 대형 벤더로 통합

공통 개념: **입력 인젝션 탐지 → 출력 검사 → PII/시크릿 필터**를 API 프록시/게이트웨이로 런타임 수행. [레드티밍 실무 노트](/ko/posts/ai-red-teaming-practice/)에서 본 인수 러시의 방어 제품 측면:
- **Lakera Guard**(→Check Point): 입출력 양방향 allow/block/sanitize. 자사 게임 "Gandalf" 수집 데이터로 학습.
- **Prisma AIRS**(Palo Alto, Protect AI 흡수): Runtime+Model Security+Posture+Red Teaming+Agent Security 묶음.
- **Cisco AI Defense**(Robust Intelligence 흡수, 2025-01): shadow AI 발견 + 배포전 자동 레드팀 + 런타임 가드레일.

> ⚠️ 벤더 탐지율 수치(예: Lakera "98%+·50ms 이하")는 **독립 검증 출처 없음** — 벤더 주장으로 취급.

## 3. 프롬프트 인젝션 방어 연구 — 신뢰 경계를 복원하려는 시도

근본 문제: LLM은 "명령"과 "데이터"를 아키텍처적으로 구분하지 못한다. 아래는 그 경계를 인위적으로 세우려는 계보.

| 접근 | 개념 | 한계 |
|---|---|---|
| **CaMeL**(Google DeepMind, 2025, arXiv 2503.18813) | Dual-LLM 강화판. 특권 LLM이 신뢰 요청으로 코드(계획) 생성, 격리 LLM이 비신뢰 데이터 처리(도구권한 無), 커스텀 인터프리터가 데이터 출처를 **capability로 추적**해 제어흐름+데이터흐름 보호. 모델 수정 불필요 | AgentDojo에서 공격 **67% 방어(전부 아님)**, 토큰 ~2.7배, 정책 작성 부담 |
| **Dual-LLM 패턴**(Willison, 2023) | 도구권한 LLM은 비신뢰 콘텐츠를 직접 안 봄 | 제어흐름만 보호, 도구 인자(데이터흐름) 오염 가능 |
| **Spotlighting**(Microsoft, 2024, arXiv 2403.14720) | 비신뢰 입력 출처 표시: delimiting·**datamarking**(토큰 사이 특수문자)·**encoding**(base64) | 프롬프트 수준이라 우회 상존, 인코딩은 소형 모델 성능 저하. 저자도 확률적 완화로 규정 |
| **StruQ**(UC Berkeley, USENIX Sec 2025) | 보안 프런트엔드가 프롬프트/데이터를 특수 토큰 채널로 분리 + 데이터부 명령 무시하도록 구조화 튜닝 | 재학습 필요, 적응형(GCG류) 공격엔 불완전. 후속 **SecAlign**(2410.05451)이 선호 최적화로 개선 |

**"완전 방어는 없다"는 합의(검증)**: OWASP LLM01:2025가 생성형 AI의 확률적 본질상 **"fool-proof 예방법은 없을 가능성이 높다"** 명시. 영국 NCSC도 프롬프트 인젝션을 "영구 미해결일 수 있는 문제"로 경고. Willison의 **lethal trifecta**(2025-06) — **비공개 데이터 접근 + 비신뢰 콘텐츠 노출 + 외부 통신**이 한 에이전트에 공존하면 유출은 필연 — 이 실무 위협모델의 사실상 표준이며, 방어는 "탐지"보다 **세 요소 중 하나를 아키텍처로 제거**하는 쪽이 권장된다.

## 4. AI-SPM·ML 공급망 — 모델 파일 자체가 공격 벡터

**검증 사실**: pickle 직렬화는 역직렬화 중 임의 연산을 허용, Hugging Face 악성 모델 다수가 pickle 익스플로잇 사용. 스캐너 자체도 우회 취약(picklescan 4건) — **스캐닝은 필요조건이지 충분조건 아님**.

| 통제 | 내용 |
|---|---|
| **모델 스캐닝** | Protect AI **ModelScan**(OSS, 다중포맷 직렬화공격 탐지·CI/CD), HiddenLayer Model Scanner(상용) |
| **안전한 직렬화** | **safetensors**(가중치 값만, 코드실행 경로 無, Rust). 2023 Trail of Bits 감사에서 치명결함 無 판정→HF 기본. **pickle(.pt/.pkl) 금지 + safetensors 강제**가 표준 |
| **모델 서명** | OpenSSF/Sigstore **model-transparency**: 아티팩트 암호서명·출처증명 |
| **AI-BOM** | **CycloneDX v1.6**부터 ML-BOM 공식 지원, MITRE ATLAS **AML.M0023**로 채택 |

## 5. 표준의 방어 매핑 — ATLAS Mitigations·OWASP·NIST

**MITRE ATLAS Mitigations**(AML.M0000~M0025, 26개) — 이 주제 직결:

| 코드 | 완화책 | 대응 공격 |
|---|---|---|
| AML.M0007 / M0025 | Sanitize Training Data / Dataset Provenance | 데이터 오염 |
| AML.M0015 | Adversarial Input Detection | 인젝션·회피 입력 |
| AML.M0020 / M0021 / M0022 | GenAI Guardrails / Guidelines / Model Alignment | jailbreak·인젝션 |
| AML.M0013 / M0014 / M0023 | Code Signing / Verify ML Artifacts / AI-BOM | 모델 공급망 |
| AML.M0004 / M0005 / M0019 | 쿼리 제한 / 접근통제 | 모델 추출·자원 남용 |

- **OWASP LLM Top 10 2025 완화책**: LLM01(권한 최소화·출력검증·인간승인·신뢰경계 분리), LLM02(PII 필터), LLM03/04(스캐닝·SBOM·서명·출처검증), LLM05(**출력을 비신뢰 입력으로 취급**·인코딩), LLM06(도구 최소권한·human-in-the-loop), LLM08(RAG 접근통제), LLM10(rate limit).
- **NIST AI 100-2 E2025**: 예측/생성 AI로 공격 분류 + 각 완화책에 **그 한계까지 병기**한 것이 특징(어느 것도 완전하지 않음 명시).
- **DASF v3.0**(13컴포넌트, 에이전틱 추가, 97위험·73통제) / **Google SAIF**(Data·Infra·Model·App 4영역 + 위험 자가진단).

## 6. 가드레일의 한계 — 문헌 근거

가드레일은 **확률적 완화이지 보안 경계가 아니다**가 문헌 합의.

| 한계 | 근거 |
|---|---|
| 분류기 우회 | Meta Prompt Guard(초기판)가 **글자 사이 공백 삽입만으로 99.8% 우회**(Robust Intelligence 발견, Meta 인정·수정) |
| 체계적 우회 | 보이지 않는 유니코드·적대적 접미사로 상용·OSS 탐지기 다수 우회(arXiv 2504.11168) |
| 전이 공격 | 한 모델의 적대 쿼리가 가드 모델로 전이돼 평균 ~82% 성공. **보류(held-out) 프롬프트로 자체 평가** 권고(SoK arXiv 2506.10597) |
| 과차단(over-refusal) | **XSTest**(안전 프롬프트 250)·**OR-Bench**(8만): 안전학습 강도와 과차단률이 상관 — 안전성↔유용성 근본 긴장 |
| 가드 자체가 공격면 | 일부 가드 모델이 분류 대신 유해 콘텐츠 생성("helpful mode" jailbreak) |

---

## 종합 — 심층방어만이 답 (방어자 관점)

1. **단일 가드레일 신뢰 금지** — 우회 실증이 반복. 필터는 여러 층 중 하나일 뿐.
2. **심층방어 스택**: ① 아키텍처적 신뢰경계 분리(CaMeL류 / lethal trifecta 세 요소 중 하나 제거) ② 도구 **최소권한** ③ 출력 처리 통제(LLM 출력을 비신뢰 입력 취급) ④ 가드레일(입출력 분류) ⑤ **로깅·탐지**(ATLAS AML.M0024) 의 중첩.
3. **공급망은 별도 축**: safetensors 강제·모델 스캐닝·서명·AI-BOM. 데이터 오염은 사후탐지가 어려워 **출처 관리(provenance)가 1차 통제**.
4. **평가 규율**: 벤더 벤치 점수가 아니라 held-out 적대 프롬프트로 자체 검증([레드티밍 실무](/ko/posts/ai-red-teaming-practice/)의 자동 레드팀 도구가 여기 투입).
5. **SIEM/SOC 접점**: 가드레일 판정·거부·인젝션 탐지 이벤트를 로그로 수집해 OWASP LLM01/MITRE ATLAS ID로 태깅하면, 전통 SIEM의 [ATT&CK 태깅 탐지 엔지니어링](/ko/posts/traditional-blue-teaming/)과 동일 커버리지·공백 관리 워크플로우를 AI 시스템에 적용 가능.

> **블루팀 3축 지도**: ① 전통 대상([전통적 블루티밍](/ko/posts/traditional-blue-teaming/)) ② **AI 시스템 방어(이 글)** ③ AI를 도구로([AI를 이용한 방어 — AI SOC와 퍼플팀](/ko/posts/ai-augmented-defense-ai-soc/)). 이 글은 레드팀 ①(AI가 표적)의 정확한 방어 대응물이다.

---
