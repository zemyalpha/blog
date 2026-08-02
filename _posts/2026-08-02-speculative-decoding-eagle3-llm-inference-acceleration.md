---
title: "추측적 디코딩이 LLM 추론을 6.5배 빠르게 만드는 방법: EAGLE-3까지의 진화와 프로덕션 적용 가이드"
date: "2026-08-02"
keywords: ["추측적 디코딩", "speculative decoding", "EAGLE", "vLLM", "LLM 추론 가속화", "draft model"]
lang: "ko"
description: "추측적 디코딩(speculative decoding)이 어떻게 LLM 추론을 2배에서 6.5배까지 가속하는지, Leviathan의 원조 논문부터 EAGLE-3까지의 진화와 vLLM·SGLang 프로덕션 적용 방법을 정리한다."
---

# 추측적 디코딩이 LLM 추론을 6.5배 빠르게 만드는 방법

LLM 서비스에서 가장 큰 비용과 병목은 autoregressive 디코딩이다. 토큰을 하나 생성하려면 모델 전체의 가중치를 GPU의 HBM에서 불러와야 하고, 다음 토큰은 이전 토큰에 의존하므로 직렬로만 진행할 수 있다. 배치 크기가 작을 때, 즉 지연 시간(latency)에 민감한 채팅 시나리오에서는 메모리 대역폭이 충분히 활용되지 못한다. GPU는 연산을 기다리며 쉬고 있는 셈이다.

추측적 디코딩(speculative decoding)은 이 낭비를 역이용한다. GPU의 여유 연산력을 활용해 한 번에 여러 토큰을 "추측"하고, 큰 모델이 이를 병렬로 검증하는 방식이다. 2022년 말 Google의 논문으로 시작된 이 기술은 2025년 EAGLE-3에 이르러 최대 6.5배의 가속을 달성했고, vLLM, SGLang, NVIDIA TensorRT-LLM 같은 주요 서빙 프레임워크에 기본 통합되었다.

이 글에서는 추측적 디코딩이 어떻게 작동하는지, 왜 "손실 없는(lossless)" 가속이 가능한지, 그리고 2026년 프로덕션 환경에서 실제로 어떻게 적용하는지를 정리한다.

## 추측적 디코딩의 핵심 원리

### 왜 autoregressive 디코딩이 느린가

LLM 디코딩의 본질적 병목은 연산이 아니라 메모리 대역폭이다. 토큰 하나를 생성할 때마다 모델의 전체 가중치를 한 번씩 읽어야 한다. 예를 들어 70B 모델을 fp16으로 서빙하면 가중치만 약 140GB다. 이를 A100 80GB 2장에 분산해 올려도, 토큰당 가중치를 한 번씩 읽어오는 비용이 지배적이다. 이를 memory-bound 상태라고 부른다.

이때 GPU의 연산 유닛(SM)은 상대적으로 한가하다. 메모리에서 데이터가 도착하기를 기다리며 유휴 상태(idle)로 머무는 비율이 높다. 추측적 디코딩은 바로 이 유휴 연산력을 활용한다 — "GPU가 쉬고 있을 때, 한 번에 여러 토큰을 검증하자"는 아이디어다.

### Draft-Verify 구조

추측적 디코딩은 두 단계로 구성된다.

**1. Draft 단계**: 작고 빠른 draft 모델(또는 드래프트 헤드)이 다음 k개의 토큰을 빠르게 생성한다. 예를 들어 "The capital of France is" 다음에 "Paris, which is located"라는 5개 토큰을 추측한다. 이때 draft 모델은 큰 모델과 같은 어휘를 사용하지만 훨씬 작아서 토큰당 생성 비용이 수십 배 낮다.

**2. Verify 단계**: 큰 타겟 모델이 이 5개의 토큰을 **한 번의 forward pass**로 병렬 검증한다. autoregressive 디코딩에서는 5개 토큰을 생성하는 데 5번의 순차 forward pass가 필요하지만, 추측적 디코딩에서는 검증을 병렬로 수행하므로 1번의 forward pass로 끝난다. 타겟 모델은 각 위치에서 "진짜 다음 토큰"을 계산하고, draft 모델의 추측과 비교해 일치하는 만큼 채택(accept)한다.

핵심은 **검증이 병렬로 이루어져도 한 번의 forward pass 비용이 단일 토큰 생성과 거의 같다**는 점이다. 메모리 대역폭 관점에서, 가중치는 한 번만 읽으면 되고 검증 토큰들이 그 위에서 연산을 공유하기 때문이다.

### 왜 손실이 없는가: 수정된 거부 샘플링

가장 놀라운 점은 추측적 디코딩이 **출력 분포를 바꾸지 않는다**는 것이다. 단순히 "틀리면 버린다"가 아니라, Leviathan et al. (2022)과 Chen et al. (2023)이 독립적으로 제안한 수정된 거부 샘플링(modified rejection sampling)을 사용한다.

동작은 이렇다. 타겟 모델이 계산한 확률 분포 `q(x)`와 draft 모델의 확률 `p(x)`를 비교한다. `p(x) ≤ q(x)`이면 해당 토큰을 채택한다. `p(x) > q(x)`이면 확률 `1 - q(x)/p(x)`로 거부하고, 거부된 경우 `q(x) - p(x)`에 비례하는 확률로 타겟 분포에서 재샘플링한다. 이 과정을 수학적으로 풀면, 최종 출력의 기대 분포가 정확히 타겟 모델의 분포 `q(x)`와 일치한다. 즉 "어떤 품질 저하도 없이" 속도만 빨라지는 것이다.

> 이론적 보장은 멱급수 형태로 표현된다. k개의 토큰을 추측했을 때, 평균적으로 채택되는 토큰 수의 기댓값 E는 draft 모델과 타겟 모델의 분포 유사도에 비례한다. 두 모델의 출력이 완전히 같으면 모든 토큰이 채택되어 k배 가속이 되고, 전혀 다르면 1개만 채택되어 일반 디코딩과 같아진다. 실제로는 그 사이 어딘가에서 작동한다.

## EAGLE까지의 진화: 어떻게 draft 모델을 개선했나

### 1세대: 별도 draft 모델 (Leviathan, Chen)

초기 추측적 디코딩은 작은 별도 모델을 draft로 사용했다. 예를 들어 70B 타겟 모델에 7B draft 모델을 짝지는 식이다. Leviathan et al.은 T5-XXL에서 2-3배 가속을 보였고, Chen et al. (DeepMind)은 Chinchilla 70B에서 분산 환경에서 2-2.5배 가속을 달성했다.

문제는 두 가지였다. 첫째, 별도 모델을 학습하고 유지하는 비용이 크다. 둘째, 작은 모델은 큰 모델과 출력 분포가 달라 acceptance rate(채택률)가 낮다. 채택률이 낮으면 추측이 많이 틀려서 검증만 하고 실제로는 토큰을 얻지 못하는 셈이 된다.

### 2세대: Medusa — 드래프트 헤드 추가

Medusa(Cai et al., 2024)는 별도 모델 대신 타겟 모델 자체에 여러 개의 예측 헤드를 추가했다. 각 헤드는 현재 위치에서 1, 2, 3... 토큰 앞을 예측한다. 이 헤드들은 타겟 모델의 표현을 공유하므로 분포 정렬이 더 쉽고, tree attention으로 여러 후보를 동시에 검증한다. Medusa-1(백본 고정)은 2.2배 이상, Medusa-2(백본 공동 학습)는 2.3-3.6배 가속을 달성했다.

### 3세대: EAGLE — feature-level autoregression

EAGLE(Li et al., ICML 2024)의 핵심 통찰은 "토큰 수준보다 feature(두 번째 최상위층) 수준에서 autoregression을 하는 것이 더 쉽다"는 것이다. 드래프트 모델이 토큰이 아니라 타겟 모델의 중간 표현(feature)을 예측하면, 불확실성이 크게 줄어든다. 그리고 한 스텝 앞선 토큰 시퀀스를 입력에 포함해(feature uncertainty 해결) 예측 정확도를 높였다. 결과적으로 LLaMA2-Chat 70B에서 2.7-3.5배 가속을 달성했고, 동시에 처리량(throughput)도 두 배로 늘렸다.

### EAGLE-2: 동적 드래프트 트리

EAGLE-2(EMNLP 2024)는 정적 드래프트 트리의 한계를 지적했다. EAGLE-1은 고정된 트리 구조를 사용했는데, 채택률은 문맥에 따라 달라진다는 점을 발견했다. 예를 들어 쉬운 문장("The sky is")에서는 draft 모델의 신뢰도가 높고 채택률도 높지만, 어려운 문장("The quantum Hamiltonian of")에서는 낮아진다. EAGLE-2는 draft 모델의 confidence score가 실제 채택률을 잘 근사한다는 사실을 이용해, 매 스텝마다 트리 구조를 동적으로 조정한다. 결과는 3.05-4.26배 가속, EAGLE-1 대비 20-40% 향상이었다.

### EAGLE-3: training-time test로 6.5배 달성

EAGLE-3(NeurIPS 2025 수락, 2025년 3월 발표)은 두 가지 근본적 변경을 가했다. 첫째, feature 예측을 버리고 직접 토큰 예측으로 전환했다. 둘째, 타겟 모델의 최상위 feature에만 의존하던 것을 저/중/고위 feature의 융합으로 바꿨다. 핵심 기술은 "training-time test"로, 학습 시점에 추론 시점의 조건을 모방(simulate)해 draft 모델이 더 많은 학습 데이터의 혜택을 받을 수 있게 했다.

결과는 인상적이다. Vicuna-13B 기준 vanilla 디코딩 대비 5.6배, EAGLE-1 대비 1.8배 가속. 더 큰 모델과 reasoning 모델에서는 최대 6.5배에 달한다. SGLang 프레임워크에서 batch size 64 환경에서는 처리량(throughput) 기준 1.38배 향상을 보였다 — 추측적 디코딩이 단일 요청 지연 시간뿐 아니라 배치 환경의 처리량에도 도움이 됨을 보여준다.

## 추측적 디코딩 방법론 비교

| 방법 | 연도/학회 | 핵심 아이디어 | 가속 배율(vanilla 대비) | 별도 모델 필요 |
|------|-----------|--------------|------------------------|---------------|
| Speculative Decoding (Leviathan) | 2022, Google | 작은 별도 draft 모델 + 병렬 검증 | 2-3배 | 예 |
| Speculative Sampling (Chen) | 2023, DeepMind | Chinchilla 70B, 분산 환경 | 2-2.5배 | 예 |
| Medusa-1/2 | 2024 | 타겟 모델에 다중 예측 헤드 추가 | 2.2-3.6배 | 아니오(헤드만) |
| EAGLE-1 | ICML 2024 | feature-level autoregression | 2.7-3.5배 | 작은 드래프트 |
| EAGLE-2 | EMNLP 2024 | 동적 드래프트 트리 | 3.05-4.26배 | 작은 드래프트 |
| EAGLE-3 | NeurIPS 2025 | training-time test + 다층 feature 융합 | 최대 6.5배 | 작은 드래프트 |

주의할 점: 표의 가속 배율은 각 논문이 보고한 특정 모델·하드웨어 조건下的 수치다. EAGLE 시리즈의 13B 수치(5.6배 등)는 Vicuna-13B, RTX 3090 2장, fp16 기준이며, 실제 프로덕션 환경(더 큰 모델, 더 큰 GPU, 더 큰 배치)에서는 다를 수 있다. 배치 크기가 커지면 GPU가 이미 포화 상태이므로 가속 효과가 줄어든다.

## 프로덕션 적용: vLLM과 SGLang에서 실행하기

### vLLM에서 추측적 디코딩 활성화

vLLM은 EAGLE, MTP(Multi-Token Prediction), 별도 draft 모델, n-gram, suffix 디코딩 등 여러 추측 방법을 지원한다. 2026년 현재 vLLM의 추측적 디코딩 설정은 `--speculative-config` 플래그로 JSON 객체를 전달하는 방식으로 통합되었다. EAGLE-3를 사용하는 예시는 다음과 같다.

```bash
# EAGLE-3 드래프트 모델과 함께 vLLM 서버 실행
vllm serve meta-llama/Llama-3.1-8B-Instruct \
    --speculative-config '{
        "method": "eagle3",
        "model": "yuhuili/EAGLE3-LLaMA3.1-Instruct-8B",
        "num_speculative_tokens": 5
    }'
```

주요 키:
- `method`: 추측 방법. `eagle3`, `draft_model`, `ngram`, `suffix`, `mtp` 등에서 선택.
- `model`: 드래프트 모델 경로 (HuggingFace 체크포인트 또는 로컬 경로). `ngram`/`suffix`에서는 생략 가능.
- `num_speculative_tokens`: 한 번에 추측할 토큰 수. 일반적으로 3-8 사이가 적당하며, 채택률이 높은 모델일수록 큰 값이 유리하다.

vLLM 공식 문서는 EAGLE을 "강력한 범용 모델 기반 방법(strong general-purpose model-based method)"으로 분류하며, 저QPS(지연 중심)뿐 아니라 고QPS(처리량 중심) 환경에서도 중간 이상의 이득을 준다고 명시한다. 다만 파이프라인 병렬화(pipeline parallelism)는 vLLM 0.15.0 기준 추측적 디코딩과 호환되지 않는다는 제약이 있다.

EAGLE 공식 저장소에 따르면, EAGLE은 vLLM 외에도 SGLang, NVIDIA TensorRT-LLM, NVIDIA NeMo Framework, AMD ROCm, Intel Extension for Transformers, PaddleNLP, MLC-LLM, SpecForge 등 주요 프레임워크에 통합되어 있다.

### SGLang에서의 적용

SGLang은 RadixAttention과 추측적 디코딩을 결합해 추가 이득을 얻는다. EAGLE-3 논문이 SGLang 환경에서 batch size 64 처리량 1.38배 향상을 보고한 것도 이 때문이다. SGLang은 서버 실행 시 추측 알고리즘과 드래프트 모델 경로, 추측 스텝 수 등을 인자로 받아 EAGLE-3를 활성화한다. 정확한 인터페이스는 SGLang 공식 문서의 speculative decoding 섹션을 참조해야 한다(문서 구조가 자주 변경되므로 최신 문서를 직접 확인할 것).

SGLang 팀과 EAGLE 팀은 2025년 7월 SpecForge라는 학습 도구를 통해 "바로 사용 가능한(out-of-the-box)" EAGLE-3 학습을 권장하고 있다.

### 커스텀 모델용 드래프트 학습

공식 체크포인트가 없는 모델(예: 자체 파인튜닝 모델)에는 직접 드래프트를 학습해야 한다. EAGLE-3 학습은 공식 저장소의 DeepSpeed 기반 스크립트를 사용한다.

```bash
cd eagle/traineagle3
deepspeed main.py --deepspeed_config ds_config.json
```

EAGLE-1 시절에는 8x RTX 3090로 1-2일이면 학습 가능했다고 공식 README에 명시되어 있다("trainable within 1-2 days on 8x RTX 3090 GPUs, so even the GPU poor can afford it"). EAGLE-3은 training-time test 기법과 더 많은 데이터가 필요하지만, 접근 가능한 수준으로 설계되었다. 구체적인 학습 설정(데이터셋 경로, 하이퍼파라미터 등)은 공식 저장소의 `traineagle3` 디렉토리와 문서를 참조할 것.

## 언제 추측적 디코딩이 빛을 발하는가 (그리고 아닐 때)

### 효과적인 시나리오

1. **낮은 배치, 지연 민감 서비스**: 채팅, 코드 자동완성, 실시간 에이전트처럼 batch size가 1에 가깝고 첫 토큰 지연(TTFT)과 토큰 간 지연(ITL)이 중요한 경우. 이때 GPU가 memory-bound 상태이므로 추측적 디코딩의 효율이 극대화된다.

2. **예측 가능한 출력**: 코드 생성, 구조화된 출력(JSON), 템플릿 기반 응답처럼 패턴이 뚜렷한 작업. draft 모델의 채택률이 높아진다.

3. **reasoning 모델**: EAGLE-3 논문은 reasoning 모델(o1 스타일)에서도 효과적임을 보였다. 사고 과정(scratchpad)이 반복적이고 예측 가능한 패턴을 포함하기 때문으로 분석된다.

### 효과가 떨어지는 시나리오

1. **큰 배치, 처리량 위주 서비스**: batch size가 수십~수백이면 GPU가 이미 연산 포화 상태다. 추측적 디코딩의 검증 단계가 추가 연산 부담이 되어 오히려 느려질 수 있다. EAGLE-3가 batch 64에서 1.38배 처리량 향상을 보인 것은 예외적이며, batch가 훨씬 크면 효과가 줄어든다.

2. **높은 무작위성(temperature)**: temperature가 높으면 타겟 모델과 draft 모델의 분포 차이가 커져 채택률이 급락한다. 창의적 글쓰기 같은 작업에서는 효과가 제한적이다.

3. **드래프트 모델 부재**: 공식 체크포인트가 없으면 학습 비용이 발생한다. 일회성 추론이나 프로토타입 단계에서는 단순 n-gram 추측(vLLM의 `--speculative-model "[ngram]"`)이 더 실용적일 수 있다.

## 결론: 추측적 디코딩을 도입하기 전 점검할 것

추측적 디코딩은 2026년 LLM 서빙에서 사실상 표준이 되고 있는 가속 기술이다. 핵심을 요약하면:

- **원리**: 작은 draft 모델이 여러 토큰을 추측하고, 큰 타겟 모델이 병렬로 검증한다. 수정된 거부 샘플링으로 출력 분포를 보존하므로 "손실 없는" 가속이다.
- **진화**: 별도 draft 모델(Leviathan/Chen) → Medusa(헤드 추가) → EAGLE(feature 수준 autoregression) → EAGLE-3(training-time test, 최대 6.5배). 3년 만에 가속 배율이 3배 가까이 올랐다.
- **프레임워크**: vLLM, SGLang, TensorRT-LLM 등 주요 서빙 프레임워크에 통합되어 있어, 공식 체크포인트만 있으면 플래그 하나로 켤 수 있다.
- **적용 조건**: 낮은 배치·지연 민감 시나리오에서 가장 효과적. 큰 배치 처리량 위주 환경이나 높은 무작위성 설정에서는 효과가 제한적이다.

당장 시도해볼 첫 단계는, 현재 서빙 중인 모델의 공식 EAGLE-3 체크포인트가 있는지 확인하는 것이다. 있다면 vLLM에 `--speculative-model` 플래그 하나로 A/B 테스트를 해보라. 지연 시간이 반 이하로 떨어지는 것을 직접 확인할 수 있을 것이다. 공식 체크포인트가 없다면 n-gram 추측부터 시작해 효과를 가늠한 뒤, 트래픽이 충분히 커지면 자체 드래프트 학습을 고려하면 된다.

## 참고 자료

- Leviathan, Y., Kalman, M., & Matias, Y. (2022). *Fast Inference from Transformers via Speculative Decoding*. arXiv:2211.17192 — Google의 원조 논문. T5-XXL에서 2-3배 가속.
- Chen, C., Borgeaud, S., Irving, G., et al. (2023). *Accelerating Large Language Model Decoding with Speculative Sampling*. arXiv:2302.01318 — DeepMind의 독자적 제안. Chinchilla 70B, 2-2.5배 가속.
- Cai, T., Li, Y., Geng, Z., et al. (2024). *Medusa: Simple LLM Inference Acceleration Framework with Multiple Decoding Heads*. arXiv:2401.10774 — 다중 예측 헤드, 2.2-3.6배.
- Li, Y., Wei, F., Zhang, C., & Zhang, H. (2024). *EAGLE: Speculative Sampling Requires Rethinking Feature Uncertainty*. ICML 2024. arXiv:2401.15077 — feature 수준 autoregression, 2.7-3.5배.
- Li, Y., Wei, F., Zhang, C., & Zhang, H. (2024). *EAGLE-2: Faster Inference of Language Models with Dynamic Draft Trees*. EMNLP 2024. arXiv:2406.16858 — 동적 드래프트 트리, 3.05-4.26배.
- Li, Y., Wei, F., Zhang, C., & Zhang, H. (2025). *EAGLE-3: Scaling up Inference Acceleration via Training-Time Test*. NeurIPS 2025. arXiv:2503.01840 — training-time test, 최대 6.5배.
- EAGLE 공식 저장소: github.com/SafeAILab/EAGLE — 공식 체크포인트, 학습 스크립트, 프레임워크 통합 목록 포함.
