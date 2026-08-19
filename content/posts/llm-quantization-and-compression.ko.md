---
title: "LLM 양자화 종류와 압축 기법 — 정리 노트"
date: 2026-06-30T11:00:00+09:00
tags:
  - llm
  - systems
  - quantization
  - on-device
  - korean
summary: "LLM을 작고 빠르게 만드는 압축 축 다섯(양자화 / 프루닝 / 증류 / 저랭크 / KV캐시)을 정리하고, 실무에서 가장 자주 쓰는 양자화를 어떤 기준으로 고를지에 답한다. INT4에서 부동소수 마이크로스케일링(NVFP4/MXFP4)으로 옮겨가는 추세까지."
---

> **주제**: LLM을 더 작게·빠르게 만드는 **압축 4+1 축**(양자화 / 프루닝 / 증류 / 저랭크 / KV캐시)과, 그중 실무 핵심인 **양자화의 종류**(비트 표현 · 생태계 포맷 · GGUF 접미사 · 보정 기법)를 공개 자료 수준에서 정리.
> **관점**: 온디바이스·로컬 서빙 실무자용. "어떤 양자화를 고를까"에 바로 답하는 것이 목적.
> **연결**: [LLM 안전정렬은 어디에 있고 어떻게 무뎌지나](/ko/posts/llm-safety-alignment-bypass/) §8의 "Q4 추론 vs bf16 수술" 메모리 차이의 **원리**가 여기 §2~3에 있다.

---

## 1. 큰 그림 — 압축은 두 방향뿐

모델을 줄이는 길은 본질적으로 둘이다: **(a) 가중치 하나를 더 적은 비트로**(정밀도↓) 또는 **(b) 가중치 개수 자체를 줄임**(파라미터↓). 5개 기법이 이 둘에 매핑된다.

| 축 | 줄이는 것 | 방식 | 재학습 | 비고 |
|---|---|---|---|---|
| **① 양자화 (quantization)** | 비트/가중치 | 16비트 → 8/4/2비트 | 보통 불필요(PTQ) | **압축 대비 품질 최고** — 1순위 |
| **② 프루닝 (pruning)** | 가중치 개수 | 덜 중요한 weight·뉴런·레이어 제거 | 보통 필요(회복) | 구조적/비구조적, perplexity 상승 대가 |
| **③ 지식 증류 (distillation)** | 파라미터(통째) | 큰 teacher → 작은 student 모방 학습 | 필요(학생 학습) | Gemma·Qwen 소형판이 이 산물 |
| **④ 저랭크 분해 (low-rank)** | 파라미터(행렬) | W ≈ A·B 로 분해 | 경우에 따라 | LoRA(튜닝)·SVD(압축) 양용 |
| **⑤ KV 캐시 압축** | 추론 메모리 | 런타임 KV 텐서 압축(가중치 아님) | 불필요 | 긴 문맥에서 진짜 병목 → §6 |

> **한 줄**: 실무 압축의 90%는 **①양자화**다. ②③④는 양자화로 부족할 때 겹쳐 쓰는 보조, ⑤는 "모델"이 아니라 "추론 중 메모리"를 줄이는 별개 축. 하이브리드(프루닝+양자화+저랭크)가 최신 연구 방향이지만, 단일 기법으로는 양자화의 압축/품질 비율을 못 이긴다.

---

## 2. 양자화의 두 갈래 — 언제 자르나(PTQ/QAT), 무엇을 자르나(W-only/W+A)

### 2-1. 시점: PTQ vs QAT

| | **PTQ** (Post-Training Quantization) | **QAT** (Quantization-Aware Training) |
|---|---|---|
| 시점 | 학습 **끝난 뒤** 양자화 | 학습 **중에** 양자화 시뮬레이션 |
| 비용 | 싸다(보정 데이터 수백 샘플) | 비쌈(재학습 파이프라인 필요) |
| 품질 | 4비트까지는 충분 | **2비트 등 극저비트에서 우위** |
| 실무 | **로컬 LLM은 거의 다 PTQ** (GPTQ/AWQ/GGUF) | 프런티어 랩이 극저비트 낼 때 |

→ **개인·로컬 환경에서 받는 모델은 사실상 전부 PTQ.** QAT는 원학습 파이프라인 접근이 필요해 재배포 모델엔 드물다.

### 2-2. 대상: weight-only vs weight+activation

- **weight-only** — 가중치만 4비트, 활성값(activation)은 FP16 유지. GPTQ·AWQ·GGUF가 여기. **메모리 절감이 목적**, 정확도 손실 적음.
- **weight+activation** — 활성값까지 8비트(예: W8A8). 연산을 INT8/FP8 텐서코어로 → **속도까지** 절감. SmoothQuant·FP8이 여기. 단, **활성 outlier**가 품질을 깨는 게 난점.

> **활성 outlier 문제**가 양자화 기법들을 가르는 핵심 축이다. 소수 채널의 극단값이 4비트 격자를 망가뜨림 → AWQ(중요 1% 채널 보호)·SmoothQuant(난이도를 weight로 이전)·NVFP4(블록 16 + 2단 스케일)가 전부 이 문제를 다르게 푸는 것.

---

## 3. 비트 표현 — INT4·FP8·FP4(MX/NV)·NF4

양자화 "타입"의 1차 구분은 **숫자를 어떤 자료형으로 담느냐**다.

| 표현 | 비트 | 특징 | 쓰이는 곳 |
|---|---|---|---|
| **INT8** | 8 | 정수 선형. 안전한 기본 | 서버 W8A8, 레거시 |
| **INT4** | 4 | 정수 4비트. 압축 큼, 보정 필요 | GPTQ·AWQ의 기반 |
| **NF4** (NormalFloat4) | 4 | 정규분포 가중치에 **정보이론적 최적** 4비트 격자 | bitsandbytes(QLoRA) |
| **FP8** (E4M3/E5M2) | 8 | 부동소수 8비트. 동적범위 넓음 | Hopper/Blackwell 네이티브, vLLM 서빙 |
| **MXFP4** | 4 | microscaling FP4. 블록 32개에 공유 스케일 | Blackwell, 학습까지 |
| **NVFP4** | 4 | 블록 **16** + per-block E4M3 스케일 + per-tensor FP32 2단 스케일 → MXFP4보다 정확 | Blackwell, FP8 대비 2~3× 처리량·~1.8× 메모리 |

> **추세(2025~26)**: 정수(INT4) → **부동소수 마이크로스케일링(NVFP4/MXFP4)**으로. 핵심 아이디어는 "작은 블록마다 별도 스케일"이라 outlier에 강함. NVFP4가 블록을 16까지 줄이고 2단 스케일을 얹어 FP4급 압축에 INT8급 정확도를 노린다. 단 **Blackwell급 하드웨어 필요** — M3 Air·일반 GPU엔 아직 GGUF/AWQ가 현실.

---

## 4. 생태계별 포맷 — GGUF · GPTQ · AWQ · bitsandbytes

같은 "4비트"라도 **누가/어떤 런타임용으로** 만들었냐로 갈린다. 이게 실무에서 가장 헷갈리는 지점.

| 포맷 | 런타임 | 방식 | 강점 | 적합 |
|---|---|---|---|---|
| **GGUF** | llama.cpp / ollama | k-quant/i-quant (블록별 스케일) | **CPU+GPU 하이브리드**, Mac/노트북, 비트단계 다양 | 로컬·온디바이스 ⭐ |
| **GPTQ** | HF/vLLM/exllama | Hessian(2차 도함수)로 weight별 민감도 추정 | 동일 비트 **weight 정확도 최고**, 준비 비쌈 | GPU 서빙 |
| **AWQ** | vLLM/SGLang/TensorRT | activation 크기로 **중요 1% 채널 보호** | 4비트 품질 우수(흔히 GPTQ>·Q5_K_M급), 빠름 | 프로덕션 서빙 ⭐ |
| **bitsandbytes** | HF Transformers | 로드시 NF4 즉석 양자화(`load_in_4bit`) | 변환 단계 0, **QLoRA 파인튜닝** | 튜닝·간편 실행 |

> **품질 벤치(Llama-3.1-8B, 2025-10 한 측정)**: AWQ ~95% / GGUF ~92% / GPTQ ~90% 보존. 절대수치는 측정마다 다르지만 순서·체감은 대체로 일관.

**선택 한 줄:**
- 로컬·Mac·노트북 → **GGUF Q4_K_M** (ollama 한 줄)
- vLLM/TGI 프로덕션 → **AWQ** (GPU FP8 지원되면 FP8)
- QLoRA 파인튜닝 → **bitsandbytes NF4**
- HF 파이썬 서비스에서 품질 우선 → AWQ, 없으면 GPTQ

---

## 5. GGUF 접미사 해독 — `Q4_K_M`이 뭔가

로컬에서 제일 자주 보는 게 GGUF 파일명. 규칙은 **`Q{비트}_{계열}_{크기}`**.

```
Q4_K_M
│ │ └ 크기: S(small)/M(medium)/L(large) — 같은 비트 내 품질-용량 트레이드
│ └── 계열: K = k-quant(슈퍼블록 2단 스케일)  ·  없으면 legacy(_0/_1)
└──── 비트: 가중치당 평균 비트수 (Q2~Q8)
```

**세 계열:**

| 계열 | 예 | 구조 | 언제 |
|---|---|---|---|
| **legacy** | Q4_0, Q4_1, Q8_0 | 블록당 스케일 1개(_0 대칭)·+offset(_1 비대칭). 단순 affine | Q8_0(거의 무손실)만 실용, 나머진 구식 |
| **k-quant** ⭐ | Q3_K_M, Q4_K_M, Q5_K_M, Q6_K | 작은 블록 스케일 + 슈퍼블록 스케일(2단) → piecewise-affine | **기본 권장**, ≤5비트에서 legacy보다 우수 |
| **i-quant** | IQ2_XXS … IQ4_XS, IQ4_NL | 코드북 기반, 더 공격적 저비트 | 더 작게(큰 모델·긴 문맥) 필요할 때, imatrix 의존 큼 |

**실무 권장(용량 vs 품질):**

| 양자화 | 7B 용량 | 체감 품질 | 메모 |
|---|---|---|---|
| Q8_0 | ~7.5 GB | 원본 거의 동일 | 안전빵 |
| **Q5_K_M** | ~5 GB | 열화 거의 못 느낌 | 품질 우선 기본 |
| **Q4_K_M** ⭐ | ~4.5 GB | 대부분 작업서 원본급 | **가장 많이 받는 기본** |
| Q3_K_M | ~3.5 GB | 눈에 띄는 열화 시작 | 메모리 빠듯할 때 |
| IQ2/Q2_K | ~2.5 GB | 품질 급락 | 최후의 수단 |

> **imatrix(중요도 행렬, importance matrix)**: 보정 데이터에서 **자주 활성화되는 가중치 위치**의 평균제곱 활성을 기록한 표. 양자화 스케일을 그 가중치 쪽으로 편향시켜 같은 비트에서 perplexity를 낮춤. **효과는 2~4비트 공격적 양자화에서 가장 큼** → Q4_K_M 미만은 imatrix 적용본을 받는 게 거의 필수. (`Q4_K_M`의 K가 곧 이 계열 기법을 쓴다는 뜻.)

---

## 6. KV 캐시 압축 — 긴 문맥의 진짜 메모리 병목

가중치를 4비트로 줄여도, **긴 문맥에서는 KV 캐시가 가중치보다 메모리를 더 먹는다**. 이건 모델 압축과 별개 축.

- **KIVI** (ICML 2024) — 파인튜닝 없는 2비트 KV 양자화. **key는 per-channel, value는 per-token**으로 비대칭 양자화(key에 채널 outlier가 크다는 관찰). 가중치+KV 합쳐 peak 메모리 **2.6× 절감**, 배치 4×, 처리량 2.35~3.47×.
- 계열 기법: KVTuner(레이어별 혼합정밀), ChunkKV/CompressKV(의미 기반 토큰 가지치기), MiniKV(2비트 레이어 판별).
- KV 압축 ≠ 가중치 양자화: **KIVI는 정밀도↓, CompressKV류는 토큰 개수↓** — 직교적이라 같이 쓴다.

> 본 레포 [Gemma 3 분석](/ko/posts/gemma3-technical-report/)의 "5:1 local:global attention으로 KV 6배 절감"은 **아키텍처 차원의 KV 절감** — 여기 §6의 양자화/가지치기와 보완 관계(설계 vs 사후압축).

---

## 7. 런타임 실행 — 압축된 모델은 "어디에" 올라가나

§1~6이 **"파일이 얼마나 작아지나"**라면, 여기는 **"그 파일이 실행 시 어디 올라가고 얼마나 빠른가"**다. 양자화로 용량을 줄여도 **빠른 메모리에 통째로 올라가야** 제 속도가 난다.

### 7-1. VRAM 전용 vs 오프로딩

| 방식 | 동작 | 속도 | 비고 |
|---|---|---|---|
| **VRAM 전용** | 모델을 GPU 메모리에 통째 적재 | **가장 빠름** | 권장. RTX 3060(12GB)/4070 Ti S(16GB)/5090(32GB) |
| **오프로딩(offloading)** | VRAM 부족분을 시스템 RAM에 분산 적재 | **눈에 띄게 느려짐** | GPU↔RAM 전송 대역폭이 병목. 다 못 올릴 때의 타협책 — 가능하면 회피 |

→ 양자화를 한 단계 더 줄여서라도(예: Q5_K_M → Q4_K_M) **VRAM 안에 다 넣는 쪽**이, 큰 양자화를 오프로딩하는 것보다 보통 빠르다.

### 7-2. 통합메모리(Unified Memory) — Apple Silicon / GB10

- **M3 Air, DGX Spark(GB10)** 등은 CPU·GPU가 **하나의 메모리 풀** 공유 → "VRAM vs RAM" 구분이 사라지고 **오프로딩 자체가 불필요**(전체가 GPU 접근 가능).
- 단, 통합메모리는 **대역폭이 외장 VRAM보다 낮은 경우가 많아** 토큰 생성(TG) 속도는 제한적. (참고: [npu 노트](/ko/posts/on-device-llm-npu-inference/) — M4 120 / M4 Pro 273 GB/s가 decode 속도를 좌우.)
- → "용량은 되는데 속도가 아쉬운" 구조. M3 Air 16GB로 7B Q4가 **올라가긴 잘 올라가도**, 대역폭 탓에 고성능 외장 GPU만큼 빠르진 않음.

### 7-3. MoE 주의 — "총 파라미터 ≠ 속도"

일부 모델은 **MoE(전문가 혼합)** 라 토큰마다 일부 전문가만 활성화된다 (본 레포 [deployment-aware 노트](/ko/posts/deployment-aware-llm-evaluation/)의 Gemma 4 계열이 이 구조).

- 예: **Gemma 4 26B-A4B** = 총 26B, **활성 ~4B**(128 experts 중 top-k).
- **메모리** = **총 파라미터** 전체를 올려야 함 (26B → ~15GB Q4).
- **속도(TG)** = **활성 파라미터**(4B)처럼 빠름.
- → **"용량은 26B, 속도는 4B".** 그래서 MoE는 "VRAM은 큰데 빠른" 트레이드 — 메모리만 받쳐주면 Dense보다 유리(동일 활성 기준 MoE가 Dense 압도).

> **한 줄**: 양자화(§1~6)는 **파일 크기**를, 런타임(§7)은 **그 파일이 빠른 메모리에 다 들어가는지 + MoE면 활성만큼만 계산되는지**를 결정한다. 둘을 같이 봐야 "내 기기에서 빠른가"가 나온다.

---

## 8. 이 레포와의 연결 — "추론은 가볍고 수술은 무겁다"의 원리

[LLM 안전정렬은 어디에 있고 어떻게 무뎌지나](/ko/posts/llm-safety-alignment-bypass/) §8의 메모리 차이가 바로 위 개념들의 귀결:

- **추론(Q4)** = §5의 k-quant 압축본. 7B가 ~3.5GB(가중치) → 실행 ~7–8GB.
- **abliteration 수술(bf16)** = §3의 16비트 원본 필요. Q4(4비트 근사값) 위에서 거부방향 직교화 같은 정밀 연산을 하면 격자가 깨짐 → **수술은 반드시 bf16, 끝나면 다시 Q4로**(GGUF 변환). 그래서 7B 수술 ~16–18GB.

> **한 줄**: 양자화는 "보기 좋게 압축"(추론)엔 충분하지만, "정밀 편집"(수술)엔 원본이 필요하다 — 같은 모델의 메모리가 2배 벌어지는 이유.

---

## Sources

- Arditi et al., "Refusal in Language Models Is Mediated by a Single Direction" (NeurIPS 2024) — abliteration이 bf16을 요구하는 맥락
- llama.cpp — [quantize README](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md) · [imatrix README](https://github.com/ggml-org/llama.cpp/blob/master/tools/imatrix/README.md)
- "Which Quantization Should I Use? A Unified Evaluation of llama.cpp Quantization on Llama-3.1-8B" — [arXiv 2601.14277](https://arxiv.org/abs/2601.14277)
- The Kaitchup, ["Choosing a GGUF Model: K-Quants, I-Quants, and Legacy Formats"](https://kaitchup.substack.com/p/choosing-a-gguf-model-k-quants-i)
- KIVI: "A Tuning-Free Asymmetric 2bit Quantization for KV Cache" (ICML 2024) — [arXiv 2402.02750](https://arxiv.org/abs/2402.02750)
- NVIDIA, "Quantization-Aware Distillation for NVFP4" (2026) · MXFP4/NVFP4 microscaling 자료
- UniComp / "A Systematic Study of Compression Ordering for LLMs" (2025–26) — 양자화·프루닝·증류 비교
- 서베이: "A Survey on Model Compression for Large Language Models" — [arXiv 2308.07633](https://arxiv.org/abs/2308.07633)
