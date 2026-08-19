---
title: "Gemma 3 Technical Report — 논문 분석 및 프로젝트 응용"
date: 2026-06-27T14:00:00+09:00
tags:
  - llm
  - systems
  - paper-review
  - korean
summary: "Gemma 3 테크니컬 리포트를 읽고 아키텍처 개선 세 가지와 그 대가를 정리했다. 27B Dense 모델이 Chatbot Arena Elo 1338로 671B MoE·405B 모델을 앞선 결과를 어떻게 만들었는지, 로컬 서빙 관점에서 무엇이 의미 있는지."
---

> **논문**: Gemma 3 Technical Report
> **출처**: arXiv:2503.19786v1 (2025년 3월)
> **저자**: Gemma Team, Google DeepMind (Aishwarya Kamath, Johan Ferret 외 다수)
> **코드/모델**: Google에서 오픈 릴리스 (1B / 4B / 12B / 27B)

---

## 1. 핵심 주장

Gemma 3는 Gemma 2의 후속으로, 1B~27B 규모의 **경량 오픈 멀티모달 모델** 패밀리다. 이전 세대 대비 세 가지 핵심 개선을 도입한다: **(1) 비전 이해 능력** (SigLIP 인코더 통합), **(2) 128K 토큰 롱 컨텍스트**, **(3) 다국어 커버리지 확대**. 아키텍처 차원에서는 **local:global attention 비율을 5:1로 조정**하고 local attention의 sliding window를 1024로 제한하여 KV 캐시 메모리 폭발 문제를 해결했다.

> 결과: Gemma3-27B-IT는 Chatbot Arena Elo **1338로 Top 10** 진입 — DeepSeek-V3 (671B/37B MoE), LLaMA 3.1 405B보다 높은 점수를 **27B Dense 모델**로 달성. Gemma3-4B-IT는 Gemma2-27B-IT와 경쟁하고, Gemma3-27B-IT는 Gemini-1.5-Pro와 비견.

---

## 2. 모델 아키텍처

### 모델 규모

| 모델 | Vision Encoder | Embedding Params | Non-embedding Params | 컨텍스트 |
|------|---------------|-----------------|---------------------|---------|
| 1B | 없음 | 302M | 698M | 32K |
| 4B | 417M (SigLIP) | 675M | 3,209M | 128K |
| 12B | 417M (SigLIP) | 1,012M | 10,759M | 128K |
| 27B | 417M (SigLIP) | 1,416M | 25,600M | 128K |

### 핵심 아키텍처 변경점

| 요소 | Gemma 2 | Gemma 3 | 효과 |
|------|---------|---------|------|
| Local:Global 비율 | 1:1 | **5:1** | KV 캐시 메모리 60% → <15% (32K 기준) |
| Sliding window | 4096 | **1024** | perplexity 영향 미미, 메모리 대폭 절감 |
| Attention capping | Soft-capping | **QK-norm** | 안정성 개선 |
| RoPE base freq (global) | 10K | **1M** | 128K 컨텍스트 지원 |
| Tokenizer | Gemma 전용 (256K) | **Gemini 2.0 공용 (262K)** | 비영어 언어 균형 개선 |

**5:1 local:global interleaving이 핵심 기여다.** 5개의 local sliding window attention 레이어 사이에 1개의 global attention 레이어를 배치한다. Local 레이어는 1024 토큰 범위만 보고, global 레이어만 전체 컨텍스트에 attend한다. 이를 통해 perplexity 손실 없이 KV 캐시 메모리를 극적으로 줄인다.

```
레이어 구성: [Local, Local, Local, Local, Local, Global, Local, Local, ...]

KV 캐시 메모리 (2B 모델, 128K 컨텍스트):
  Global only:     ~6000 MB
  L:G=5:1, sw=1024: ~1000 MB  ← 약 6배 절감
```

---

## 3. 학습 방법

### 3.1 사전학습

| 모델 | 학습 토큰 | 인프라 | 칩 수 |
|------|---------|--------|------|
| 1B | 2T | TPUv5e | 512 |
| 4B | 4T | TPUv5e | 2,048 |
| 12B | 12T | TPUv4 | 6,144 |
| 27B | 14T | TPUv5p | 6,144 |

- **Knowledge distillation**: 토큰당 256개 logit을 teacher 확률로 가중 샘플링하여 cross-entropy로 학습. 비샘플 logit은 확률 0으로 설정 후 재정규화.
- **데이터**: 텍스트 + 이미지 혼합. 다국어 데이터 증량 (단일어 + 병렬 데이터). Unimax 전략으로 언어 불균형 처리.
- **비전 인코더**: SigLIP 400M (ViT, 896×896 입력). 4B/12B/27B에서 공유, **학습 중 frozen** — 임베딩을 미리 계산하여 LM 학습 비용에 추가 부담 없음.

### 3.2 Post-training (Instruction Tuning)

| 기법 | 설명 |
|------|------|
| Knowledge Distillation | 대형 IT teacher에서 증류 |
| BOND | Best-of-N distillation으로 RL alignment |
| WARM | Weight averaged reward models — 리워드 모델 앙상블 |
| WARP | Weight averaged rewarded policies |
| 리워드 함수 | Human feedback, 코드 실행 피드백, 수학 ground-truth 리워드 |

### 3.3 Quantization Aware Training (QAT)

원본 체크포인트에서 약 5,000 스텝 QAT 파인튜닝으로 양자화 버전 제공.

| 모델 | bf16 (GB) | Int4 (GB) | Int4 block=32 (GB) | SFP8 (GB) |
|------|-----------|-----------|-------------------|-----------|
| 1B | 2.0 | 0.5 | 0.7 | 1.0 |
| 4B | 8.0 | 2.6 | 2.9 | 4.4 |
| 12B | 24.0 | 6.6 | 7.1 | 12.4 |
| 27B | 54.0 | 14.1 | 15.3 | 27.4 |
| 27B +KV(32K) | 72.7 | 32.8 | 34.0 | 46.1 |

---

## 4. 비전 모달리티 — Pan & Scan

SigLIP 인코더는 **고정 해상도 896×896**으로 동작하므로, 비정사각형 이미지나 고해상도 이미지에서 텍스트가 읽히지 않거나 작은 객체가 사라지는 문제가 발생한다. **Pan & Scan (P&S)**은 이를 해결하는 추론 시 최적화 기법이다:

- 이미지를 동일 크기의 비겹침 크롭으로 분할
- 각 크롭을 896×896으로 리사이즈하여 인코더에 입력
- 추론 시에만 적용, 비활성화 가능

| 모델 | DocVQA | InfoVQA | TextVQA |
|------|--------|---------|---------|
| 4B (P&S 없음) | 72.8 | 44.1 | 58.9 |
| 4B (P&S 적용) | **81.0** (+8.2) | **57.0** (+12.9) | **60.8** (+1.9) |
| 27B (P&S 없음) | 85.6 | 59.4 | 68.6 |
| 27B (P&S 적용) | **90.4** (+4.8) | **76.4** (+17.0) | **70.2** (+1.6) |

**문서/인포그래픽 QA에서 P&S 효과가 극적이다.** InfoVQA에서 최대 +17.0% 향상.

---

## 5. 평가 결과

### Chatbot Arena (27B-IT)

| 순위 | 모델 | Elo | Open | Type | 파라미터 |
|------|------|-----|------|------|---------|
| 1 | Grok-3-Preview | 1412 | - | - | - |
| 1 | GPT-4.5-Preview | 1411 | - | - | - |
| 6 | DeepSeek-R1 | 1363 | yes | MoE | 671B/37B |
| **9** | **Gemma-3-27B-IT** | **1338** | **yes** | **Dense** | **27B** |
| 13 | DeepSeek-V3 | 1318 | yes | MoE | 671B/37B |
| 14 | Claude 3.7 Sonnet | 1309 | - | - | - |
| 28 | LLaMA-3.1-405B | 1269 | yes | Dense | 405B |
| 59 | Gemma-2-27B-IT | 1220 | yes | Dense | 27B |

**27B Dense 모델로 Elo 1338은 놀라운 결과다.** 671B MoE인 DeepSeek-V3보다 높고, 405B Dense인 LLaMA-3.1보다 69점 높다. Gemma 2 대비 +118 Elo 상승.

### IT 모델 벤치마크 비교 (zero-shot)

| 벤치마크 | Gemma 2 27B | Gemma 3 4B | Gemma 3 27B | Gemini 1.5 Pro | Gemini 2.0 Pro |
|---------|------------|------------|------------|---------------|---------------|
| MMLU-Pro | 56.9 | 43.6 | **67.5** | 75.8 | 79.1 |
| MATH | 55.6 | 75.6 | **89.0** | 86.5 | 91.8 |
| HiddenMath | 14.8 | 43.0 | **60.3** | 52.0 | 65.2 |
| LiveCodeBench | 20.4 | 12.6 | **29.7** | 34.2 | 36.0 |
| GPQA Diamond | 34.3 | 30.8 | **42.4** | 59.1 | 64.7 |
| Global MMLU-Lite | 68.6 | 54.5 | **75.1** | 80.8 | 86.5 |
| MMMU (val) | - | 48.8 | **64.9** | 65.9 | 72.7 |
| FACTS Grounding | 62.4 | 70.1 | **74.9** | 80.0 | 82.8 |

**주목할 점:**
- **Gemma3-27B MATH 89.0** — Gemini 1.5 Pro (86.5)를 능가
- **Gemma3-4B MATH 75.6** — Gemma2-27B (55.6)를 대폭 상회, post-training 레시피의 위력
- **HiddenMath에서 Gemma3-27B (60.3)** — Gemini 1.5 Pro (52.0)보다 +8.3

### 확장 IT 벤치마크 (Gemma 2 vs Gemma 3)

| 벤치마크 | Gemma 2 27B | Gemma 3 27B | 변화 |
|---------|------------|------------|------|
| HumanEval | 51.8 | **87.8** | +36.0 |
| MBPP | 67.4 | **74.4** | +7.0 |
| GSM8K | 91.1 | **95.9** | +4.8 |
| BBH | 74.9 | **87.6** | +12.7 |

---

## 6. 핵심 Ablation 결과

### Local:Global 비율의 영향

| L:G 비율 | Perplexity 변화 | 비고 |
|---------|---------------|------|
| 1:1 (Gemma 2) | 기준선 | KV 캐시 메모리 최대 |
| 3:1 | ~0.0 | 거의 변화 없음 |
| 5:1 (Gemma 3) | ~0.0 | **채택** |
| 7:1 | ~+0.05 (2B) | 미미한 열화 |

**결론: 7:1까지도 perplexity 영향이 미미하다.** 메모리 절감 대비 품질 손실이 거의 없는 "공짜 점심"에 가까운 기법.

### Sliding Window 크기

512~4096 범위에서 perplexity 변화가 ±0.01 수준. **1024를 채택해도 품질 손실 없이 메모리를 크게 절감.**

### 교사 모델 크기 (Distillation)

- **짧은 학습 (수십B 토큰)**: 작은 teacher가 유리 (정규화 효과)
- **긴 학습 (수백B 토큰 이상)**: 큰 teacher가 유리
- **결론**: "작은 교사가 항상 낫다"는 기존 통념은 학습 기간이 짧은 실험에서 도출된 것. 충분히 길게 학습하면 **큰 교사가 더 좋다.**

### Vision Encoder 해상도

| 해상도 | DocVQA | InfoVQA | TextVQA |
|------|--------|---------|---------|
| 256 | 31.9 | 23.1 | 44.1 |
| 448 | 45.4 | 31.6 | 53.5 |
| 896 | **59.8** | **33.7** | **58.0** |

**높은 해상도 = 더 나은 성능.** 특히 문서/텍스트 이해에서 차이가 극적.

---

## 7. Memorization & Safety

- Gemma 3 모델의 **memorization rate가 이전 모든 Gemma/Gemini 모델보다 현저히 낮음** (로그 스케일에서 1~2 오더 차이)
- 1B/4B/12B/27B 간 memorization 차이는 미미 (1B가 가장 낮음)
- Approximate memorization이 exact memorization 대비 **약 24배** 더 많음
- Google SDP 서비스로 개인정보 탐지 결과: **모든 Gemma 3 모델에서 memorization으로 분류된 출력에 개인정보 0건**

---

## 8. 핵심 교훈

1. **KV 캐시는 "공짜로" 줄일 수 있다** — Local:Global 5:1 + sliding window 1024로 perplexity 손실 없이 KV 캐시를 ~6배 줄인다. 롱 컨텍스트 추론에서 메모리가 병목이면 이 기법을 최우선 검토하라.
2. **Post-training이 모델 크기를 뛰어넘는다** — Gemma3-4B가 Gemma2-27B를 수학·코딩에서 압도. Distillation + RLHF + BOND/WARM/WARP 조합이 파라미터 수 증가보다 효과적일 수 있다.
3. **큰 teacher, 긴 학습이 정답** — "작은 teacher가 낫다"는 통념은 짧은 학습에서만 성립. 충분한 토큰 예산이 있으면 가장 큰 teacher를 써라.
4. **비전 통합은 텍스트 성능을 해치지 않는다** — Gemma 3는 비전을 추가하면서도 텍스트 벤치마크에서 Gemma 2를 전반적으로 능가. Frozen encoder + 임베딩 사전 계산이 핵심.
5. **P&S는 VLM의 필수 기법** — 고정 해상도 인코더의 한계를 추론 시 크롭으로 극복. 문서/인포그래픽 이해에서 +8~17% 향상.

---

## 9. 프로젝트별 응용 방안

### 응용 1: 로컬 LLM 배포 — 모델 선택 기준

Gemma 3는 **소비자용 하드웨어(폰, 노트북, GPU)** 를 명시적 타겟으로 설계되었으므로, 로컬 LLM 도입 시 유력한 후보다.

| 시나리오 | 추천 모델 | 메모리 (Int4) | 근거 |
|---------|---------|-------------|------|
| 모바일/엣지 | Gemma3-1B | 0.5 GB | 텍스트 전용, 32K 컨텍스트 |
| 노트북/맥미니 | Gemma3-4B Int4 | 2.6 GB | 비전 포함, Gemma2-27B급 성능 |
| 서버/고급 GPU | Gemma3-27B Int4 | 14.1 GB | Gemini 1.5 Pro급, MATH 89.0 |

- **KV 캐시 절감**: 5:1 L:G 아키텍처 덕분에 32K 컨텍스트에서도 KV 캐시가 모델 가중치의 15% 미만. 기존 global-only 모델(LLaMA 등)은 60%+ → **같은 RAM에서 더 긴 컨텍스트 사용 가능**.
- **로그 분석 워크로드**: 입력이 긴 SIEM 로그 분석에서 128K 컨텍스트 + 낮은 KV 캐시 오버헤드가 유리.

### 응용 2: Post-training 레시피 참고

- BOND + WARM + WARP 조합은 파인튜닝 시 참고할 만한 최신 기법 스택
- 수학/코딩에서 **코드 실행 피드백** + **ground-truth 리워드**를 결합한 RL이 효과적

### 모든 프로젝트 공통 — 즉시 적용

| 액션 | 효과 |
|------|------|
| 기존 global-only 모델 대신 **Gemma 3 4B/27B 검토** | 같은 하드웨어에서 더 긴 컨텍스트, 더 낮은 메모리 |
| **KV 캐시 메모리를 모델 선택 기준에 포함** | 롱 컨텍스트 시나리오에서 실질적 제약은 KV 캐시 |
| Gemma3-4B를 **경량 비전 태스크에 활용** (문서 OCR, 차트 이해) | Int4로 2.6GB, P&S로 DocVQA 81.0 |
| LLM 벤치마크 시 **post-training 기법 차이**를 함께 기록 | 모델 크기보다 post-training이 성능에 더 큰 영향 |

---

## 10. 종합 평가

### 논문 자체 평가

| 항목 | 판단 |
|------|------|
| 출처 수준 | arXiv 테크니컬 리포트 — 학회 논문은 아니지만 Google DeepMind 공식 보고서 |
| 문제 제기 | 명확. 롱 컨텍스트의 KV 캐시 문제, 오픈 모델의 멀티모달 격차를 정면 해결 |
| 기여 | (1) 5:1 L:G attention으로 KV 캐시 혁신 (2) 27B Dense로 Top 10 Arena (3) 4B가 이전 27B 능가 |
| 실용성 | 매우 높음. 오픈 가중치 공개, QAT 양자화 버전 제공, 소비자 하드웨어 타겟 |
| 한계 | 외부 모델과 직접 비교 회피 ("평가 세팅이 다르다"), Chatbot Arena Elo가 preliminary, 롱 컨텍스트 128K 이후 급격히 열화, 비전은 SigLIP frozen이라 한계 있음 |

### 핵심 한 줄 요약

> **"KV 캐시를 5:1 local:global interleaving으로 6배 줄이면서도 품질 손실이 없다는 것은, 대부분의 attention이 로컬이면 충분하다는 뜻이다."** — 롱 컨텍스트 시대에 메모리 효율과 성능을 동시에 잡는 아키텍처 설계 원칙이며, 27B Dense가 671B MoE를 이긴다는 결과는 post-training 레시피의 중요성을 재확인시킨다.
