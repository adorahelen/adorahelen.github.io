---
title: "논문·가이드 분석 노트"
description: "2026년 6~7월에 LLM 보안·평가·시스템 문헌을 집중적으로 읽고 정리한 기록."
showDate: false
showAuthor: false
showReadingTime: false
showTableOfContents: true
---

2026년 6월 말부터 7월 중순까지, LLM 보안·평가·시스템 쪽 논문과 국내 정부 가이드를 집중적으로 읽고 정리했습니다. 원래는 공개할 생각 없이 쓴 개인 노트였고, 그래서 요약보다 **"이 주장이 무엇에 근거하는지"** 를 따라가는 데 시간을 더 썼습니다. 인용 문헌을 원문으로 되짚다가 원 문서의 사실관계를 고친 것도 몇 건 있습니다.

한 편씩 독립적으로 읽어도 되지만, 서로 참조하는 묶음이 네 개 있습니다.

## ① 국내 가이드 — KISA·과기정통부 (2026.07)

같은 시기에 나온 짝 문서 두 편입니다. 국제 프레임워크(OWASP LLM Top 10 · NIST AML · MITRE ATLAS)와의 정합을 확인하고, 인용 문헌을 원문으로 교차검증했습니다.

- [AI 보안 레드티밍 가이드 분석 — 참고문헌 25건 심화](/ko/posts/kisa-ai-redteaming-guide/) · 부록 도구 목록의 사실관계 3건을 정정했습니다
- [AI 보안 위협 대응 매뉴얼 분석 — 가이드의 짝 문서](/ko/posts/kisa-ai-threat-response-manual/)

## ② 레드팀 ↔ 블루팀 — 전통과 AI, 네 축

"AI 레드티밍은 전통 레드티밍과 무엇이 다른가"에서 시작해, 공격·방어를 각각 전통/AI로 나눠 네 축으로 정리했습니다.

- [전통적 레드티밍 — 방법론·표준·블루팀 연결](/ko/posts/traditional-red-teaming/)
- [전통적 블루티밍 — SOC·탐지 엔지니어링·DFIR·헌팅·CTI](/ko/posts/traditional-blue-teaming/)
- [AI를 이용한 레드티밍 — 자율 공격보안](/ko/posts/ai-augmented-red-teaming/)
- [AI를 이용한 방어 — AI SOC와 퍼플팀](/ko/posts/ai-augmented-defense-ai-soc/)
- [AI 시스템 방어 — "AI가 표적"인 공격에 대한 블루팀 통제](/ko/posts/ai-system-defense/)
- [멀티모달·에이전트 AI 레드티밍 — 확장된 공격 표면](/ko/posts/multimodal-agent-red-teaming/)
- [AI 레드티밍 실무 — 도구·조직 방법론·규제·커뮤니티](/ko/posts/ai-red-teaming-practice/)

## ③ 정렬과 우회 — 안전은 어디에 있나

가드레일이 어디에 저장돼 있고 어떻게 무뎌지는지를 메커니즘 수준에서 따라간 묶음입니다. 마지막 글은 직접 재현하고 측정한 실험입니다.

- [LLM 안전정렬은 어디에 있고 어떻게 무뎌지나](/ko/posts/llm-safety-alignment-bypass/)
- [RAG 보안 — PoisonedRAG와 EcoSafeRAG](/ko/posts/rag-security-poisonedrag-ecosaferag/)
- [Abliteration 재현 — 거부는 단일 방향이 맞다, 다만 뜯어내면 모델이 상한다](/ko/posts/reproducing-abliteration-qwen25-3b/)

## ④ 시스템과 평가 — 무엇을 어떻게 측정하나

모델을 고르고 돌리는 쪽의 문헌입니다. 벤치마크가 실제로 무엇을 측정하는지, 그리고 온프레미스·온디바이스에서 무엇이 병목인지.

- [Gemma 3 테크니컬 리포트 분석](/ko/posts/gemma3-technical-report/)
- [LLM 양자화와 압축 기법](/ko/posts/llm-quantization-and-compression/)
- [온디바이스 LLM NPU 추론 (llm.npu, ASPLOS '25)](/ko/posts/on-device-llm-npu-inference/)
- [배포 인지 LLM 평가 — 정확도만 보면 안 되는 이유](/ko/posts/deployment-aware-llm-evaluation/)
- [ProgramBench — 처음부터 다시 짤 수 있는가](/ko/posts/programbench-rebuild-programs/)
- [파인튜닝 LLM의 자동 취약점 수정 (ICSE 2026)](/ko/posts/finetuned-llm-vulnerability-repair/)
- [디지털 트윈·페르소나 재현 — 어디까지 되고 어디서 무너지나](/ko/posts/digital-twin-persona-reproduction/)

---

영문판은 순차적으로 올리고 있습니다. 현재 **15편**이 영문으로 있습니다 —
[AI SOC](https://adorahelen.github.io/posts/ai-augmented-defense-ai-soc/) ·
[AI를 이용한 레드티밍](https://adorahelen.github.io/posts/ai-augmented-red-teaming/) ·
[레드티밍 실무](https://adorahelen.github.io/posts/ai-red-teaming-practice/) ·
[AI 시스템 방어](https://adorahelen.github.io/posts/ai-system-defense/) ·
[Gemma 3](https://adorahelen.github.io/posts/gemma3-technical-report/) ·
[KISA 레드티밍 가이드](https://adorahelen.github.io/posts/kisa-ai-redteaming-guide/) ·
[KISA 위협 대응 매뉴얼](https://adorahelen.github.io/posts/kisa-ai-threat-response-manual/) ·
[양자화·압축](https://adorahelen.github.io/posts/llm-quantization-and-compression/) ·
[안전정렬 우회](https://adorahelen.github.io/posts/llm-safety-alignment-bypass/) ·
[멀티모달·에이전트](https://adorahelen.github.io/posts/multimodal-agent-red-teaming/) ·
[온디바이스 NPU](https://adorahelen.github.io/posts/on-device-llm-npu-inference/) ·
[RAG 보안](https://adorahelen.github.io/posts/rag-security-poisonedrag-ecosaferag/) ·
[Abliteration 재현](https://adorahelen.github.io/posts/reproducing-abliteration-qwen25-3b/) ·
[전통 블루티밍](https://adorahelen.github.io/posts/traditional-blue-teaming/) ·
[전통 레드티밍](https://adorahelen.github.io/posts/traditional-red-teaming/).
