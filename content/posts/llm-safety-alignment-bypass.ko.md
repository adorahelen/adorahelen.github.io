---
title: "LLM 안전정렬은 어디에 있고 어떻게 무뎌지나 — 메커니즘·논문·방어 분석"
date: 2026-06-30T10:00:00+09:00
tags:
  - ai-security
  - llm
  - alignment
  - red-teaming
  - interpretability
  - korean
summary: "오픈웨이트 모델의 안전 가드레일은 어디에 저장돼 있고, 어떤 세 층위(탈옥 / abliteration / 파인튜닝 degrade)로 무뎌지는가. 공개 논문 수준에서 메커니즘을 정리하고 탐지·방어로 연결했다. 안전이 단일 방향·얕은 층에 몰려 있을수록 취약하다."
---

> 🇺🇸 **[English version of this post →](https://adorahelen.github.io/posts/llm-safety-alignment-bypass/)**

> **주제**: 오픈웨이트 모델(예: Gemma)의 안전 가드레일이 **어디에 저장돼 있는지**, 그리고 그것이 **무뎌지는 3개 층위**(탈옥 / abliteration / 파인튜닝 degrade)를 공개 논문 수준에서 정리.
> **관점**: 방어자·레드팀 연구용. 메커니즘이 "왜 통하는지"를 이해하고 **탐지·방어**로 연결하는 것이 목적. 유해 콘텐츠 생산 절차는 다루지 않는다.
> **핵심 한 줄**: 안전이 **단일 방향·얕은 층**에 몰려 있을수록 취약하다 — 정렬은 분산되고 깊어야 한다.

---

## 1. 안전 규칙은 "파일"이 아니라 가중치에 새겨진다

윤리/거부 규칙은 별도 `rules.txt`가 아니라 **모델 가중치 자체에 훈련으로 박힌 행동**이다. 실배포는 보통 3겹 구조:

| 층 | 위치 | 성격 | 변경 난이도 |
|---|---|---|---|
| ① **정렬 학습 (alignment)** | **가중치 내장** | instruction tuning + RLHF/DPO로 "유해 요청 거부"를 행동으로 학습 | 어려움(재학습 필요) — **하지만 후술하듯 의외로 얕음** |
| ② **시스템 프롬프트** | 외부(런타임) | "너는 안전하게 답한다" 류 지시 | 쉬움(즉시 교체) |
| ③ **외부 필터** | 외부(별도 모델) | 입출력 검열. Gemma의 경우 Google **ShieldGemma**(별도 안전 분류 모델) | 별도 운영 |

→ "내장돼 있냐"의 본질적 답은 **①이 가중치에 있다**. 본체엔 ①만 있고, 배포 시 ②③을 얹는다.

---

## 2. 가드레일이 무뎌지는 3개 층위

### 층위 A — 프롬프트 레벨 (모델 불변, "탈옥/jailbreak")

가중치는 그대로 두고 **입력만 조작**. 롤플레이("제약 없는 캐릭터"), 가상 시나리오, 인코딩 우회, prefix 주입 등.

- **특성**: 일시적·모델 의존적. 패치되면 막힘.
- **연구 도구**: `garak`(LLM 취약점 스캐너)가 이 영역의 프로브를 자동화 — 방어자가 자기 모델의 탈옥 표면을 측정하는 용도.

### 층위 B — 활성공간 조작 ("Abliteration") ★ 개념적으로 가장 흥미로움

핵심 논문: **Arditi et al. (2024), "Refusal in Language Models Is Mediated by a Single Direction"** (NeurIPS 2024).

```
관찰: 모델은 개념을 활성공간(activation space)의 '방향(벡터)'으로 표현한다.
발견: '거부(refusal)' 행동이 놀랍게도 거의 '하나의 방향'에 의해 좌우된다.

거부 방향 추출:
  r = mean(유해 프롬프트 활성값) − mean(무해 프롬프트 활성값)
  → 두 분포의 평균 차이가 '거부 방향'

조작:
  그 방향 r을 가중치/활성에서 직교 제거(orthogonalize)
  → 모델이 거부를 '표현하는 능력' 자체가 약화 → 재학습 없이 가드레일이 무뎌짐
```

- **특성**: 경량(추론 활성만 다룸, 재학습 불필요) → GPU 적게 듦. 소형 노트북급에서도 재현 가능.
- **연구 가치**: "안전이 **단일 방향**에 몰려 있으면 취약하다"는 직접적 증거.

### 층위 C — 파인튜닝 (재학습으로 정렬 덮어쓰기)

순응적 데이터(거부 안 하는 Q&A)로 LoRA/SFT를 돌리면 정렬이 덮어써진다.

- 핵심 결과: **아주 적은 양의 파인튜닝만으로도 안전 정렬이 무너진다** (few-shot으로도 degrade).
  - Qi et al. (ICLR 2024), "Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!" — **양성 데이터로 무심코 파인튜닝해도** 안전이 저하.
- 후속 연구의 진단: **"얕은 정렬(shallow alignment)"** — 안전 행동이 응답 **앞쪽 소수 토큰**에 집중돼 있어, 그 부분만 흔들려도 전체가 무너짐.
  - Qi et al. (2024), "Safety Alignment Should Be Made More Than Just a Few Tokens Deep".

---

## 3. 비교 요약

| 층위 | 건드리는 대상 | 비용 | 지속성 | 방어자에게 주는 교훈 |
|---|---|---|---|---|
| **A 탈옥** | 입력(프롬프트) | 최저 | 일시적(패치 가능) | 입력 필터링·탐지의 **한계**를 드러냄 |
| **B abliteration** | 활성/가중치 방향 | 낮음 | 영구(가중치 변형) | 안전이 **단일 방향**에 집중 → 분산 정렬 필요 |
| **C 파인튜닝** | 가중치 재학습 | 중간 | 영구 | 오픈웨이트 **공급망 위협** — 재배포 시 안전 소멸 |

---

## 4. 방어자 관점에서 진짜 배울 점

1. **A(탈옥)** → 입력 단계 필터링·탐지만으로는 못 막는다. 런타임 방어는 보조일 뿐.
2. **B(abliteration)** → 안전이 소수 방향/특징에 몰리면 직교 제거 한 번에 무너진다. **정렬을 더 분산**시키고, 다중 방향·다층에 걸치게 해야 robust.
3. **C(파인튜닝 degrade)** → 오픈웨이트 모델은 "누가 미세조정 후 재배포하면 안전이 사라진다"는 **공급망 위협**을 본질적으로 안고 있다. 그래서 모델 본체와 **별개의 외부 안전망(ShieldGemma 등)**이 따로 존재하는 것 — 본체 정렬이 깨져도 입출력단에서 한 번 더 거른다.

> **방어 설계 원칙**: 정렬을 (a) 더 깊게(응답 전체 길이에 걸쳐), (b) 더 분산되게(단일 방향 의존 탈피), (c) 다층 방어(본체 정렬 + 외부 분류기)로 — 어느 한 겹이 뚫려도 전체가 무너지지 않게.

---

## 5. 방어는 어떻게 거나 — 4겹 다층 방어 + 이중용도 엔탱글먼트

두 종류의 위험 콘텐츠는 **다른 방식**으로 막힌다:

| 콘텐츠 종류 | 막는 방식 | 이유 |
|---|---|---|
| 순수 유해·분리 가능 (CSAM 등) | **데이터 필터링** — 애초에 안 배움 | 소지 자체가 불법, 정상 지식과 안 엉킴 → 깨끗이 제거. abliteration해도 잘 안 나옴 |
| 이중용도(dual-use) 지식 | **배우되 행동 족쇄**(정렬) | 정상 지식과 엉켜 못 뺌 → 위에 "악용 요청엔 거부" 행동층을 얹음 |

**이중용도 엔탱글먼트가 핵심.** 위험 지식 상당수는 정상 지식과 뒤엉켜(entangled) 깨끗이 분리 안 됨:
- "독극물 작용 기전"은 의대 교과서·위키피디아에, "익스플로잇 원리"는 보안 논문에 있음.
- 화학·생물·약리·보안·역사를 다 빼면 모델이 그 분야 자체에 무능해짐.
- → 그래서 이중용도 영역은 **"알고는 있다 + 족쇄는 행동층"** 구조. abliteration/탈옥으로 일부 표면화가 가능한 이유가 이것. 반면 분리 가능한 최악 콘텐츠는 아예 안 배워서 그런 우회가 잘 안 통함.

**4겹 defense-in-depth** (족쇄 하나에 안 맡기는 이유):

| # | 층 | 역할 | 약점 |
|---|---|---|---|
| ① | 데이터 필터링 | 최악은 안 배우게 (CBRN 비중 낮추는 "safety pretraining" 포함) | 이중용도는 못 뺌 |
| ② | 정렬(RLHF/DPO) | 알아도 거부하게 | **얕음 — abliteration/파인튜닝으로 벗겨짐** |
| ③ | 외부 필터 | ShieldGemma, OpenAI moderation 등 입출력 분류기 | 별도 운영·우회 가능 |
| ④ | 배포 제어 | API 레이트리밋, KYC, 모니터링, 신고 | 오픈웨이트엔 적용 불가 |

②가 뚫리니까 ①③④로 겹쳐 깐다.

## 6. 업계 추세: 행동 족쇄 → 깊은 정렬

"전부 학습 + 행동 족쇄"는 옛 방식. abliteration/탈옥 연구가 **"행동 족쇄는 쉽게 벗겨진다(shallow alignment)"**를 증명하면서 프런티어 랩들은 이동 중:

| 방향 | 내용 |
|---|---|
| **위험 능력 평가** (dangerous capability eval) | "이 모델이 생물무기 공격자를 실제로 uplift하나?"를 출시 전 측정, 임계 넘으면 출시 보류/추가 완화 |
| **언러닝** (unlearning) | 학습한 특정 위험 지식을 사후 삭제 |
| **변조 저항 정렬** (tamper-resistant safeguards) | abliteration/파인튜닝으로도 안 벗겨지는 "깊은" 정렬 |

> **한 줄**: 이중용도 → "배우고 + 행동 족쇄"(그래서 일부 표면화 가능) · 순수 유해 → "애초에 안 배움" · 업계는 "족쇄만으론 부족"을 깨닫고 **데이터 필터링 + 깊은 정렬 + 외부 필터**로 이동 중.

## 7. 실용 트랙 — "과잉거부 해금"이 목표라면 (abliteration은 둔기)

> **가장 중요한 구분: 거부율 0% ≠ 지식 해금.** abliteration은 **행동 족쇄만** 벗긴다. 진짜 유해 지식은 데이터 필터링(§5) 때문에 여전히 빈약. 실제로 관측되는 효과는 — **과잉거부 평가셋(정당한 보안/포렌식 질문)이 이제 답이 나온다**는 것. 그게 이 실험으로 측정하려는 실체다.

목표가 "정당한 펜테스트/CTF/보안교육 질문에 **과잉거부 안 하는 모델**"이라면 abliteration은 최선이 아니다 — **"과잉거부만 골라 푸는 정밀 도구"가 아니라 거부 자체를 통째로 없애는 둔기**라, 정당한 질문과 악성 거부가 같이 사라진다(선택적 해금 노브 없음). 적합도 순:

| 옵션 | 방식 | 적합 | 평가 |
|---|---|---|---|
| **B 시스템 프롬프트** ⭐먼저 | 모델 불변. "인가된 펜테스트/CTF/보안교육 맥락" 지시 | 실용·즉시 | 수술 0·되돌리기 자유·합법. 거부가 시스템프롬프트에 민감한 점(Llama-2가 견고했던 이유)을 **역이용** |
| **A 보안 전용 모델 (DeepHat-V1-7B)** | 공격/방어 보안용으로 튜닝된 공개 모델 (WhiteRabbitNeo 후속/리브랜드 계열) | 실용·정조준 | **균형형**: 보안 질문엔 답하되 명백한 악용(미성년자 착취·군사·불법)은 라이선스로 금지 → 둔기보다 깔끔. 7B급 M3 Air 가능 |
| **C pre-abliterated 모델** | HF의 uncensored 변종 다운로드 (huihui-ai 등) | 재현 생략 | 둔기 단점 그대로(모든 거부 제거) — 게시자 본인도 "안전 최적화 안 거침" 명시 |
| **D 직접 abliteration** | 풀 파이프라인(§8) | **연구·재현** | 원리학습·"정렬이 얼마나 얕은가" 측정엔 좋으나, 실용 해금엔 비효율(둔기+튜닝 수고) |

→ **권고: 실용이면 B → 부족하면 A. 연구/측정이면 D.** (이 둘은 목적이 다르다.)

**구체 모델·획득 (2026-06 기준):**

| 트랙 | 모델 | 베이스 / 사양 | 획득 |
|---|---|---|---|
| **A** ⭐ | `DeepHat/DeepHat-V1-7B` | Qwen2.5-Coder-7B + 보안 파인튜닝 · BF16 safetensors · ctx 131k | HF safetensors. ollama용은 커뮤니티 GGUF 양자화본 찾거나 직접 변환(§8 4~5단계). 7B Q4 → M3 Air 16GB OK |
| C (소형) | `huihui-ai/Qwen3-4B-abliterated` | Qwen3-4B | HF / `ollama pull huihui_ai/qwen3-abliterated` |
| C (8B) | `huihui-ai/Qwen3-8B-abliterated`, `Huihui-Qwen3-8B-abliterated-v2`(개선판) | Qwen3-8B | HF. M3 Air OK |
| C (중형) | `huihui-ai/Huihui-Qwen3.5-27B-abliterated` | Qwen3.5-27B | HF. 서버급 |
| C (검증용) | Gemma abliterated (예: "Gemma 4 Heretic", Gemma-4-31B abliterated ~17GB Q4_K_M) | Gemma 4 | repo명은 Abliterated Models Guide에서 확인. **§4의 "Gemma 견고성"을 거부율로 직접 측정**하는 용도 |

> **A vs C 선택**: 정당한 보안 질문 해금이 목적이면 **A(DeepHat)** — 악용은 막히고 보안 코딩까지 잘함. **C**는 "거부 자체를 연구"하거나 A로 부족할 때만 — 둔기라 정당/악성 거부가 같이 사라진다(§7 첫 문단의 둔기 논점 그대로).

**범위 선(고정)**: ✅ 인가된 펜테스트·CTF·보안교육·방어연구 맥락의 보안 질문 해금은 돕는다. ❌ 실제 표적에 동작하는 공격·멀웨어는 모델을 어떻게 풀든 범위 밖(그리고 모델 품질도 그쪽은 어차피 낮음).

## 8. abliteration 재현 — 파이프라인·메모리·함정 (D 트랙)

**파이프라인** (ollama는 수술 대상 ❌, **서빙 끝단** ✅ — 양자화는 맨 마지막):

```
1. HF safetensors 원본 받기   (fp16/bf16 — 수술 가능 포맷)
2. PyTorch abliteration       (활성 hook → 거부방향 → 직교 제거)
3. 수정된 safetensors 저장
4. GGUF 변환·양자화           (llama.cpp convert_hf_to_gguf.py)
5. ollama create (Modelfile)  ← ollama로 복귀, 서빙
```

### 추론(실행) vs 수술(abliteration) — HW 요구가 완전히 다름

> **핵심: A/C(추론, GGUF 받아쓰기)는 가볍고, D(직접 수술)만 무겁다.** §7의 A·C는 "이미 수술된 거 실행"이라 아래 [추론] 표 적용 — 16GB로 4B·7B·8B 다 됨(27B만 ❌). [수술] 표는 D 트랙에만 해당.

**[추론] Q4 양자화본 실행** — `ollama` / `llama.cpp`(C++, 메모리 극한 최적화):

| 모델 | 디스크(Q4) | 실행 RAM | M3 Air 16GB |
|---|---|---|---|
| Qwen3-4B abliterated | ~2.5 GB | ~5 GB | ✅ 여유 |
| DeepHat-7B / Qwen3-8B | ~4.7–5 GB | ~7–9 GB | ✅ 편안 (Q8로 올려도 ~10GB) |
| Qwen3.5-27B | ~16–17 GB | ~19–20 GB | ❌ (512 SSD 저장만 OK) → 서버 |

→ 512GB SSD면 여러 개 동시 보관 가능. (BF16 **원본**은 양자화본보다 큼: 7B≈15GB, 27B≈54GB — 원본은 D 트랙에서만 필요.)

**[수술] bf16 원본 로드 + 활성값** — `PyTorch`. **메모리 ≈ 파라미터 ×2 GB**, 추론의 ~2배:

| 모델 | bf16 메모리 | M3 Air 16GB | 서버 |
|---|---|---|---|
| gemma-2-2b | ~5 GB | ✅ | ✅ |
| Qwen2.5-3B / 4B | ~7–12 GB | ⚠️ 빠듯 | ✅ |
| 7–8B 수술 | ~16–18 GB | ❌ 거의 불가(스왑) | ✅ |
| gemma-4-31B | ~62 GB | ❌ | ✅ (5090 32GB도 빠듯, 128GB급이면 여유) |

**왜 수술만 2배+ 먹나 — "압축본 보기 vs 원본 편집" (RAW/JPEG 비유):**

1. **정밀도(비트/가중치)** — 실행은 Q4(~4비트=0.5B/가중치 → 7B 3.5GB), 수술은 bf16(16비트=2B → 7B 14GB). **같은 7B인데 4배.** Q4는 4비트로 뭉갠 근사값이라 그 위에서 거부방향 추출·직교화 같은 정밀 연산을 하면 깨짐 → 진짜 숫자(bf16)로 수술하고 **끝나면 다시 Q4로 압축**(파이프라인 1→4단계).
2. **활성값(activations)** — 거부방향 뽑으려면 `output_hidden_states=True`로 레이어별 중간 결과를 다 들고 있어야 함 → 추가 메모리.
3. **엔진** — 실행은 llama.cpp(메모리 최적화 극한), 수술은 PyTorch(풀precision 통째 로드, 덜 효율적).

→ 7B 기준: **실행** 가중치 3.5GB + KV/오버헤드 ≈ **7–8GB ✅** / **수술** 가중치 14GB + 활성 ≈ **16–18GB ❌**.

**"코드 복붙 = 자동 뚫림"은 과장 — 신뢰할 숫자를 내려면 보완할 4가지:**

1. **모델 의존성** — Qwen은 잘 뚫리고 Gemma/Llama-2는 안 뚫릴 수 있음. 똑같은 코드라도 BEFORE→AFTER 하락폭이 모델별 천차만별 → **이 비교 자체가 결과**("그냥 뚫린다"가 거짓임을 증명).
2. **layer 선택** — `L = 0.6·depth` 한 방은 대충. 핵심은 **여러 layer·토큰 위치를 스윕**해 검증셋으로 best 방향 고르는 것. 안 하면 "안 뚫림"이 나옴 — 방법이 틀려서가 아니라 튜닝을 안 해서.
3. **거부 판정** — 키워드 매칭은 "I can't" 류 노골적 거부만 잡고 **부드러운 회피("consult a professional…")는 놓침** → 수치 부정확. 분류기(Llama Guard / HarmBench)로 판정해야 신뢰 가능.
4. **기술 함정** — MPS+fp16 일부 연산 미지원, hook 반환 시그니처(tuple/tensor), Gemma-2 logit soft-capping 등 첫 실행 디버깅 필요.

> **재현으로 실제 보게 되는 것**: 약한 정렬(Qwen)은 튜닝하면 거부율 급락(재현됨), 강한 정렬(Gemma?)은 안 떨어질 수도 — **그 자체가 "정렬이 얼마나 얕은가"의 데이터.**

📊 **실측 재현 결과**: [Abliteration 재현 — 거부는 단일 방향이 맞다, 다만 뜯어내면 모델이 상한다](/ko/posts/reproducing-abliteration-qwen25-3b/) — Qwen2.5-3B-Instruct에서 거부율 **1.00→0.00**(추론 ablation, L21) 재현. 단 레이어 10·14·18에선 효과 0(스윕 필수 실증), 과잉거부는 baseline부터 0(거부제거≠지식해금). 소요 14분.

## 9. 정당한 연구 범위 (윤리 경계)

- **돕는 것**: abliteration/탈옥이 **왜 통하는지** 재현 → 그것을 **탐지·방어**하는 법 실습(레드팀/블루팀 연구).
- **돕지 않는 것**: 이를 실제 유해 콘텐츠 생산에 쓰는 단계.

후속 실습 후보(방어 지향):
- 자기 모델에 `garak`을 돌려 탈옥 표면 측정 → 어떤 프로브가 통하는지 매핑.
- Arditi 방식의 "거부 방향"을 **탐지 신호**로 역이용(활성 모니터링으로 우회 시도 감지).
- 파인튜닝 전후 안전 벤치마크 비교로 **degrade 정량화** → 재배포 모델 검증 파이프라인 설계.

---

## 부록: 참고 표준·법·논문

> **정확한 URL·최신판·조문은 인용 전 웹에서 재확인 권장** (특히 효력 변경된 정책).

### CSAM 필터링
| 분류 | 항목 |
|---|---|
| 기술/기관 | PhotoDNA(MS, 해시매칭 사실상 표준), NCMEC CyberTipline, IWF(영국, 해시리스트), Thorn "Safer" / Google CSAI Match |
| 법(미) | 18 U.S.C. §2258A(NCMEC 의무신고), §2252/§2252A, PROTECT Act 2003 |
| 법(EU/영) | Directive 2011/93/EU, 제안 중 CSA Regulation(CSAR), 영국 Online Safety Act 2023 |
| 논문 | Thiel, Stanford Internet Observatory (2023.12) "Identifying and Eliminating CSAM in Generative ML Training Data and Models" — LAION-5B에서 CSAM 발견(모델카드가 CSAM 필터링을 강조하는 직접 배경); Thorn & All Tech Is Human (2024) "Safety by Design for Generative AI" |

### CBRN (화학·생물·방사능·핵)
| 분류 | 항목 |
|---|---|
| 정책/거버넌스 | ~~US EO 14110 (2023.10, 10²⁶ FLOP 초과 모델 CBRN·바이오 보고 의무)~~ **2025.1 폐지·후속 정책 대체**; EU AI Act(2024.8, 시스템적 위험 범용모델 10²⁵ FLOP 기준 위험평가 의무); 랩 자율 프레임워크: Anthropic RSP, OpenAI Preparedness Framework, Google DeepMind Frontier Safety Framework (모두 CBRN 임계선 명시) |
| 논문/보고서 | Mouton et al., RAND (2024) "The Operational Risks of AI in Large-Scale Biological Attacks"; OpenAI (2024) "Building an early warning system for LLM-aided biological threat creation"; Anthropic 바이오 레드팀 평가 공개자료 |

### RLHF / DPO 안전 튜닝
| 기법 | 논문 |
|---|---|
| RLHF | Christiano et al. (2017) "Deep RL from Human Preferences"(원형); Ouyang et al. (2022) InstructGPT(정전); Bai et al.(Anthropic 2022) "Training a Helpful and Harmless Assistant with RLHF"; Bai et al.(2022) "Constitutional AI"(RLAIF) |
| DPO | Rafailov et al. (2023) "Direct Preference Optimization: Your LM is Secretly a Reward Model" (NeurIPS 2023) |
| 정렬은 얕다 | Arditi et al.(2024) 단일 방향; Qi et al.(2024) shallow alignment; Qi et al.(2023) 파인튜닝 degrade |

### 상위 표준·프레임워크
| 분류 | 문서 | 내용 |
|---|---|---|
| ISO | ISO/IEC 42001:2023 | AI 경영시스템(AIMS), "AI용 ISO 9001" 격 |
| ISO | ISO/IEC 23894:2023 | AI 위험관리 |
| ISO | ISO/IEC 22989:2022 | AI 개념·용어 |
| NIST | NIST AI RMF 1.0 (2023.1) | AI 위험관리 프레임워크 |
| NIST | NIST AI 600-1 (2024.7) | 생성형 AI 프로파일(CBRN·CSAM 위험 포함) |
| EU | EU AI Act (2024.8) | 세계 첫 포괄 AI 법, 위험기반 규제 |

> ⚠️ CSAM·CBRN을 직접 다루는 단일 "ISO 표준"은 없다. CSAM=법률, CBRN=안전 프레임워크/거버넌스로 다뤄지고, ISO/NIST는 그 위 상위 관리·위험 프로세스 표준. AI 안전 "표준" 출발점은 **ISO 42001 / NIST AI RMF**.

---

## Sources

> 정확한 arXiv ID/링크는 인용 전 1회 재확인 권장.

- Arditi et al., **"Refusal in Language Models Is Mediated by a Single Direction"**, NeurIPS 2024 (arXiv 2406.11717).
- Qi et al., **"Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!"**, ICLR 2024 (arXiv 2310.03693).
- Qi et al., **"Safety Alignment Should Be Made More Than Just a Few Tokens Deep"** (얕은 정렬, 2024, arXiv 2406.05946).
- Google, **"ShieldGemma: Generative AI Content Moderation Based on Gemma"** (arXiv 2407.21772).
- `garak` — LLM vulnerability scanner (NVIDIA / Derczynski et al.).
