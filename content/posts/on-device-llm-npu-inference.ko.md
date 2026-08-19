---
title: "Fast On-device LLM Inference with NPUs (llm.npu) — 논문 분석 및 프로젝트 응용"
date: 2026-06-27T15:00:00+09:00
tags:
  - llm
  - systems
  - on-device
  - paper-review
  - korean
summary: "온디바이스 LLM의 병목은 토큰 생성(decode)이 아니라 프롬프트 처리(prefill)다. NPU로 prefill을 평균 22.4배 가속하고 에너지를 30.7배 줄인 ASPLOS '25 논문(llm.npu)을 읽고, 온디바이스 서빙 설계에 무엇을 시사하는지 정리했다."
---

> 🇺🇸 **[English version of this post →](https://adorahelen.github.io/posts/on-device-llm-npu-inference/)**

> **논문**: Fast On-device LLM Inference with NPUs
> **학회**: ASPLOS '25 (컴퓨터 구조/시스템 최고 학회)
> **저자**: Daliang Xu 외 6인 (Peking University, BUPT)
> **DOI**: https://doi.org/10.1145/3669940.3707239 · arXiv:2407.05858
> **코드**: https://github.com/UbiquitousLearning/mllm

---

## 1. 핵심 주장

온디바이스 LLM의 진짜 병목은 토큰 **생성(decode)**이 아니라 프롬프트 **처리(prefill)**다. 모바일 CPU/GPU는 prefill의 대량 병렬 연산에 약하다. **모바일 NPU(정수 연산기)** 로 prefill을 오프로딩하면 빨라지지만, NPU는 LLM과 근본적으로 안 맞는다. `llm.npu`는 프롬프트와 모델을 **3개 레벨(프롬프트·텐서·블록)** 로 재구성해 NPU 오프로딩을 처음으로 실용화했다.

> 결과: prefill **평균 22.4× 가속 + 30.7× 에너지 절감**, 10억 파라미터급에서 **최초로 1,000 tokens/sec prefill** 달성, 종단 응답 1.4–32.8× 단축, 정확도 손실 <1%.

---

## 2. 배경 — 왜 prefill이 병목인가

온디바이스 LLM 작업(UI 자동화, 이메일 자동응답, 채팅 요약)은 **입력이 길고 출력이 짧다**. 개인화·문맥 인식 때문에 프롬프트가 길어진다.

| 작업 | 입력 길이 | 출력 길이 |
|------|----------|----------|
| UI 자동화 (DroidTask) | 505–827 토큰 | 평균 3.5 토큰 |
| 이메일 자동응답 (LongBench) | 1,168–1,835 토큰 | 평균 7.9 토큰 |
| 채팅 요약 (Persona-Chat) | 488–584 토큰 | 평균 44 토큰 |

→ **prefill이 전체 지연의 88.3%–98.8%(CPU), 54.2%–91.7%(GPU)** 를 차지. 기존 연구가 집중한 decode 최적화(speculative decoding 등)는 헛다리.

**기회 — 모바일 NPU**: Qualcomm Hexagon 73 TOPS(INT8). INT8 MatMul이 CPU INT8 대비 4.5–5.8×, GPU FP16 대비 1.8–3.5× 빠르고 에너지 효율도 높음. 그런데 **COTS 모바일 NPU로 LLM을 돌리는 시스템이 없었다.**

---

## 3. 3가지 근본 난제 (NPU ↔ LLM 불일치)

| # | 난제 | 구체적 손해 |
|---|------|-----------|
| **C1** | 가변 길이 프롬프트 | NPU는 정적 shape만 지원 → 프롬프트마다 그래프 재빌드. **Gemma-2B 그래프 최적화에만 11.54초** |
| **C2** | 양자화 알고리즘 불일치 | 정확도용 per-group 양자화를 NPU가 직접 못 함 → sub-tensor 분할 + float 합산 → **최대 10.7× 오버헤드** |
| **C3** | 부동소수점(FP) 제거 불가 | LayerNorm·Attention은 FP 필수인데 NPU는 FP에 약함(INT8 대비 최대 159× 느림) |

---

## 4. 핵심 기법 3가지

### 기법 1: Chunk-sharing graph (§3.2) — C1 해결
- 가변 길이 프롬프트를 **고정 크기 "청크"** 로 분할(디코더-only라 인과적 처리 가능).
- 연산자를 둘로 구분: **정적 연산자**(Linear·LayerNorm, 청크 길이에만 의존 → **공유**) vs **동적 연산자**(Attention, 청크 순서 의존 → 개별).
- Qwen1.5-1.8B에서 **144개 중 120개 서브그래프 공유 → 메모리 75%(7.2GB) 절감**. 청크 길이 256 채택.

### 기법 2: Shadow outlier execution (§3.3) — C2 해결
- NPU엔 **per-tensor INT8** MatMul(빠름)만 돌리고, 정확도를 망치는 **이상치(outlier) 채널만 추출해 CPU에서 float로 병렬 계산** 후 합산.
- 이상치는 극히 희소(**전체 채널의 0.1–0.3%**)라 CPU 연산이 NPU 시간에 가려짐(overlap).
- 추가 최적화: 이상치가 몰리는 **"hot channel" 가중치만 메모리 상주**, 나머지는 디스크에서 on-demand(메모리 -34.3%). 게다가 **상위 85% 비중요 레이어의 이상치는 통째로 prune** → CPU-NPU 동기화 제거.

### 기법 3: Out-of-order subgraph execution (§3.4) — 버블 제거
- CPU/GPU(FP)와 NPU(INT8)를 협업시키면 **실행 버블 37%** 발생.
- 청크 순서를 무시하고 **입력 준비된 서브그래프를 먼저 실행**(C3-Graph1을 C2 버블 동안 실행 등).
- 최적 순서는 NP-hard(TSP) → **마이크로초급 온라인 휴리스틱**: "NPU stall을 가장 많이 줄이는 서브그래프 우선". **버블 37% → 0.7%**.

---

## 5. 평가 결과

**구현**: MLLM + QNN 위에 C/C++·어셈블리 10K LOC. Qualcomm Hexagon(개방형 ISA). 테스트 모델 Qwen1.5-1.8B / Gemma-2B / Phi2-2.7B / LLaMA2-7B / Mistral-7B. 기기 Redmi K70 Pro(8gen3, 24GB)·K60 Pro(8gen2, 16GB).

| 지표 | 결과 (프롬프트 1024 기준) |
|------|------|
| Prefill 속도 | llama.cpp-CPU 대비 **18.2–38.4×**, MLC-GPU 대비 32.5–43.6×, PowerInfer-V2-NPU 대비 **3.28–5.32×** |
| 에너지 | llama.cpp-CPU 대비 **35.6–59.5×** 절감 |
| 종단 지연(prefill+decode) | 베이스라인 대비 **1.4–32.8×** 단축 |
| 정확도 | FP16 대비 **평균 1% 손실** (SmoothQuant 대비 +32.9%, K-Quant 대비 +70.9% 우위) |
| 메모리 | llama.cpp 대비 1.32× (shadow outlier는 0.6–1%만 차지) |

**Ablation**: naive NPU 오프로딩은 오히려 2.55–2.68× 느림 → chunk-sharing 1.46–5.09× → shadow outlier 3.91–8.68× → OOE 추가 18–44% 개선. **세 기법이 모두 기여.**

---

## 6. 핵심 교훈

1. **온디바이스 LLM의 병목은 prefill이다** — "입력 길고 출력 짧은" 워크로드에서 decode 최적화는 의미가 적다. prefill을 따로 측정·최적화하라.
2. **양자화 방식은 하드웨어와 함께 골라야 한다** — per-group(정확)이 NPU에선 10.7× 손해. per-tensor + 이상치 별도 처리가 하드웨어 친화적.
3. **이종 프로세서(NPU+CPU/GPU) 협업 + out-of-order 스케줄링**으로 버블을 없애는 게 핵심.
4. **이상치는 희소하고 편향적** — 0.3% 채널이 80%+ 이상치를 만든다 → 소수만 특별 처리하면 정확도·속도 양립.

---

## 7. 프로젝트별 응용 방안

> 이 논문은 **모바일(Android/Qualcomm)** 기준이지만, 핵심 통찰은 **로컬 LLM 도입(맥미니/서버) 계획**과 SIEM 분석에 직접 닿는다.

### 응용 1: SIEM 로그 분석 — "이 워크로드는 prefill 바운드다"
보안 로그 분석은 **입력(로그·컨텍스트)이 매우 길고 출력(판정·요약)이 짧다** → 논문이 말한 **prefill-dominated 영역에 정확히 해당**.
- **교훈**: 모델/하드웨어를 고를 때 **decode tok/s가 아니라 prefill(prompt processing) 속도**를 벤치마크하라. 같은 모델이라도 긴 로그를 먹는 시간이 체감 지연을 좌우한다.
- **실천**: ① 반복되는 시스템 프롬프트/룰셋은 **KV 캐시 재사용·prompt caching**으로 prefill 절감 ② 긴 로그는 **chunked prefill** 지원 엔진(llama.cpp/vLLM) 활용 ③ 로그를 통째로 넣기보다 **사전 필터링 후 핵심만** 프롬프트에 → prefill 토큰 수 자체를 줄이는 게 가장 큰 이득.

### 응용 2: 맥미니 LLM 도입 결정과의 연결
- 맥(Apple Silicon)에는 **ANE(Apple Neural Engine, 16코어 NPU)** 가 있다. 현재 Ollama/llama.cpp는 GPU(Metal)만 쓰고 ANE는 거의 안 쓴다 — **이 논문의 "NPU 오프로딩"이 맥에서 아직 안 익은 영역**임을 시사.
- **판단 보강**: 단기적으론 맥의 **통합메모리 대역폭(M4 120 / M4 Pro 273 GB/s)** 이 속도를 좌우(특히 decode). 다만 prefill 바운드 워크로드라면 **GPU prefill 성능 + 충분한 RAM**이 우선. M4 Pro 48GB급이 적합.
- MLX 생태계가 ANE/prefill 최적화를 흡수하면 미래에 같은 하드웨어로 더 빨라질 여지.

### 응용 3: 엣지/모바일로 확장 시
- 경량 분석을 **모바일/엣지 기기**로 내릴 계획이 생기면, 이 논문의 mllm + QNN 스택이 직접적인 레퍼런스. Snapdragon 기기에서 1B급 모델 prefill 1000 tok/s가 현실.

### 모든 프로젝트 공통 — 즉시 적용

| 액션 | 효과 |
|------|------|
| LLM 벤치마크 시 **prefill / decode 분리 측정** | 진짜 병목 식별 |
| **prompt/KV 캐시** 도입 | 반복 시스템 프롬프트의 prefill 비용 제거 |
| 입력 **사전 필터링·요약** | prefill 토큰 수 ↓ = 가장 직접적인 지연 감소 |
| 양자화는 **per-tensor 계열 우선 검토** | 가속기 친화 + 메모리 절감 (단 정확도 확인) |

---

## 8. 종합 평가

### 논문 자체 평가
| 항목 | 판단 |
|------|------|
| 학회 수준 | ASPLOS '25 — 시스템/구조 최상위, 매우 높음 |
| 문제 제기 | 탁월. "prefill이 병목"을 수치로 못박고 NPU 3대 난제를 명확히 규명 |
| 기여 | COTS 모바일 NPU LLM 추론 **최초 시스템** + 일반화 가능한 3기법 |
| 실용성 | 오픈소스(mllm), Qualcomm 실기 검증, <1% 정확도 손실 |
| 한계 | Qualcomm Hexagon 한정(개방 ISA), decode는 CPU 백엔드(미최적), 1–7B 모델 범위, FP16 NPU 추세에 대한 장기 적합성은 논쟁 여지 |

### 핵심 한 줄 요약
> **"온디바이스 LLM의 진짜 적은 출력 속도가 아니라 '긴 입력을 읽는 시간(prefill)'이다."** — 입력이 길고 출력이 짧은 SIEM 로그 분석에 그대로 적용되는 통찰이며, 모델/하드웨어 선택의 기준을 decode가 아닌 prefill로 옮기게 한다.
