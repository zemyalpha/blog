---
title: "학습 없이 LLM 능력을 합치는 모델 병합: SLERP·TIES·DARE 실전 설계"
date: "2026-08-01"
keywords: ["모델 병합", "model merging", "mergekit", "SLERP", "TIES-Merging", "DARE", "task arithmetic"]
lang: "ko"
description: "파인튜닝 없이 여러 LLM의 가중치를 합쳐 새로운 능력을 만드는 모델 병합 기법의 원리와 mergekit 실전 가이드. 왜 작동하는지부터 SLERP, TIES, DARE 설정까지."
---

# 학습 없이 LLM 능력을 합치는 모델 병합: SLERP·TIES·DARE 실전 설계

코딩에 특화된 파인튜닝 모델, 수학에 강한 모델, 한국어 대화에 능숙한 모델 — 이 셋을 하나로 합칠 수 있다면? 추가 학습 한 번 없이, GPU 몇 분의 사용으로. 모델 병합(model merging)은 이 질문에 대한 답이다.

Hugging Face Open LLM Leaderboard 상위권에는 병합으로 만들어진 모델이 꾸준히 등장해 왔다. 2024년 mlabonne이 만든 Marcoro14-7B-slerp는 SLERP 병합만으로 당시 Open LLM Leaderboard 최상위권을 기록했고, 커뮤니티는 passthrough 방식으로 70B 가중치 재료를 쌓아 120B 이상 모델을 만들어냈다. 파인튜닝이나 증류와는 완전히 다른 카테고리의 기술이다.

## 왜 가중치를 더하면 모델이 된다

모델 병합의 전제는 단순하다. 같은 베이스 모델에서 파인튜닝된 두 모델의 가중치는 고차원 매개변수 공간에서 서로 가까운 위치에 존재한다. 두 점 사이를 직선으로 보간해도, 중간 지점은 여전히 손실이 낮은 "연결된 분지(connected loss basin)" 안에 있다.

이 현상은 **모드 연결성(mode connectivity)** 연구로 알려져 있다. Frankle 등의 **로터리 티켓 가설(Lottery Ticket Hypothesis)** 은 베이스 모델이 가진 "승리하는 부분망(winning subnetwork)"이 파인튜닝을 거쳐도 보존된다고 설명한다. 병합된 모델이 두 부모 모델의 능력을 동시에 유지하는 이유다.

단, 세 가지 제약이 있다:

- 병합하려는 모델은 **같은 베이스에서 파인튜닝**되어야 한다 (Llama 4 파인튜닝과 Qwen 3 파인튜닝은 병합 불가)
- 가중치 텐서의 **형태(shape)가 정확히 일치**해야 한다
- 부모 모델 어디에도 없는 **새로운 지식을 주입할 수는 없다**

병합은 "기존 체크포인트의 능력을 재조합"하는 기술이지, 새로운 학습이 아니다.

## 다섯 가지 병합 방법

### 1. 선형 보간 (Linear / Weighted Average)

가장 단순한 방법이다. 각 매개변수에 대해 가중 평균을 계산한다. 이 접근은 "Model Soups"(Wortsman et al., ICML 2022) 연구에서 비전 모델로 처음 제안되어 평균이 단일 모델보다 더 나은 일반화 성능을 달성함을 보였다.

```
W_merged = alpha * W_A + (1 - alpha) * W_B
```

같은 베이스에서 비슷한 작업으로 파인튜닝된 두 모델에 잘 작동한다. 하지만 alpha가 극단으로 치우치거나 두 파인튜닝 방향이 충돌하면 **치명적 간섭(catastrophic interference)** 이 발생한다 — 어느 쪽보다도 못한 모델이 나온다.

### 2. SLERP (구면 선형 보간)

SLERP는 가중치 텐서를 단위 구 위의 점으로 취급하고, 두 점 사이의 **대권(geodesic)** 을 따라 보간한다. 선형 보간이 중간 지점에서 가중치 크기가 줄어드는 문제를 해결한다 — SLERP는 가중치의 노름(norm)을 보존한다.

```
W_merged = [sin((1-t)*θ) / sin(θ)] * W_A + [sin(t*θ) / sin(θ)] * W_B
```

여기서 θ는 정규화된 두 가중치 벡터 사이의 각도다. SLERP는 **두 모델 병합의 기본값**으로 추천된다. 단, 한 번에 정확히 두 모델만 합칠 수 있어, 세 모델 이상은 SLERP를 계층적으로 반복해야 한다.

### 3. Task Arithmetic

Task Arithmetic(Ilharco et al., 2023)는 파인튜닝이 베이스에 더하는 "델타(delta)"에 주목한다:

```python
# 각 파인튜닝 모델의 태스크 벡터 = 파인튜닝 가중치 - 베이스 가중치
task_vector_A = weights_finetuned_A - weights_base
task_vector_B = weights_finetuned_B - weights_base

# 베이스에 스케일링하여 더하기
merged = weights_base + 0.7 * task_vector_A + 0.5 * task_vector_B
```

태스크 벡터는 **더하기·빼기·스케일링**이 가능하다. 덧셈으로 능력을 추가하고, 뺄셈으로 행동을 제거할 수 있다:

```python
# 코딩 능력 추가
merged = weights_base + 0.7 * coding_vector

# 유해성 감소 (벡터를 빼기)
safe_model = weights_base - 0.5 * toxic_vector
```

핵심 하이퍼파라미터는 스케일링 계수 λ다. 너무 높으면 한 태스크가 지배하고, 너무 낮으면 능력이 거의 전이되지 않는다.

### 4. TIES-Merging (Trim, Elect, Merge)

태스크 벡터에 노이즈가 많으면 단순 덧셈은 실패한다. TIES(Yadav et al., NeurIPS 2023)는 세 단계로 간섭을 제어한다:

1. **Trim**: 각 태스크 벡터에서 크기가 작은(의미 없는) 매개변수를 0으로 만든다. 상위 k%만 유지
2. **Elect**: 각 매개변수 위치에서 여러 태스크의 부호(+/-)를 투표해 **지배적인 부호(sign)** 를 결정한다
3. **Merge**: 지배적 부호에 동의하는 매개변수만 평균낸다

TIES는 **세 개 이상 모델을 병합**할 때 특히 효과적이다. 단순 평균이 서로 다른 부호의 델타를 상쇄시키는 문제를 부호 투표로 해결한다.

### 5. DARE (Drop And Rescale)

DARE(Yu et al., 2023, "Language Models are Super Mario")는 병합 델타에 **드롭아웃**을 적용한다. 각 모델의 델타 매개변수 중 일부(보통 90%)를 무작위로 버리고, 남은 것을 재스케일링한다:

```python
def dare_merge(base_weights, finetuned_list, drop_rate=0.9, lambda_coef=1.0):
    import torch
    merged_delta = {}
    for name in base_weights:
        deltas = []
        for ft in finetuned_list:
            delta = ft[name] - base_weights[name]
            mask = (torch.rand_like(delta) > drop_rate).float()
            kept = delta * mask
            # 드롭한 만큼 재스케일링하여 기대값 보존
            kept = kept / (1 - drop_rate)
            deltas.append(kept)
        merged_delta[name] = lambda_coef * sum(deltas) / len(deltas)
    return {name: base_weights[name] + merged_delta[name] for name in base_weights}
```

DARE는 많은 수의 모델을 동시에 병합할 때 품질 저하를 막아준다. "병합할 공간"을 확보하는 효과가 있기 때문이다. 실제로는 DARE + TIES를 결합한 **DARE-TIES**가 널리 쓰인다.

## mergekit으로 실제 병합하기

mergekit(arcee-ai/mergekit)은 오픈소스 모델 병합 도구다. 7,200개 이상의 GitHub 스타를 받았고, Charles Goddard가 개발했다. CPU만으로도 실행 가능하며, 최소 8GB VRAM으로 GPU 가속할 수 있다 — 공식 README에 명시된 사양이다.

설치는 간단하다:

```bash
pip install mergekit
```

병합 설정은 YAML 파일로 작성한다. 다음은 TIES로 네 모델을 병합하는 예시다:

```yaml
# ties-merge-4models.yaml
merge_method: ties
base_model: meta-llama/Llama-3.1-8B
models:
  - model: ./models/coding-finetune
    parameters:
      density: 0.5      # 상위 50% 델타만 유지
      weight: 0.25
  - model: ./models/math-finetune
    parameters:
      density: 0.5
      weight: 0.25
  - model: ./models/korean-finetune
    parameters:
      density: 0.5
      weight: 0.25
  - model: ./models/instruct-finetune
    parameters:
      density: 0.5
      weight: 0.25
parameters:
  int8_mask: true
dtype: bfloat16
```

실행 명령:

```bash
mergekit-yaml ties-merge-4models.yaml ./merged-output \
  --copy-tokenizer \
  --cuda   # GPU 사용 시. 없으면 --cuda 제거하여 CPU 실행
```

두 모델 SLERP 병합은 더 단순하다:

```yaml
# slerp-2models.yaml
merge_method: slerp
base_model: model-a
models:
  - model: model-a
  - model: model-b
parameters:
  t:
    - filter: self_attn
      value: [0, 0.5, 0.3, 0.7, 1]
    - filter: mlp
      value: [1, 0.5, 0.7, 0.3, 0]
    - value: 0.5
dtype: bfloat16
```

주목할 점은 `t` 매개변수에 리스트를 전달해 **레이어별로 다른 보간 비율**을 적용할 수 있다는 것이다. self-attention 층과 MLP 층을 서로 다른 비율로 섞으면, 두 모델의 강점을 선택적으로 유지할 수 있다.

## 한계와 주의사항

병합은 만능이 아니다. 실전에서 자주 마주치는 한계는 다음과 같다.

**아키텍처 호환성**: 병합하려면 모든 부모 모델이 정확히 같은 아키텍처와 텐서 형태를 가져야 한다. 같은 모델 패밀리라도 양자화 여부나 텐서 병렬화 설정이 다르면 실패한다.

**평가의 어려움**: 병합 결과는 직관과 다를 수 있다. 코딩 모델과 수학 모델을 50:50으로 섞었다고 해서 각각의 절반 능력이 나오지 않는다. 반드시 병합 전후로 벤치마크를 돌려 비교해야 한다.

**배포 비용**: passthrough 방식으로 레이어를 쌓아 모델을 키우면 성능은 오르지만 추론 비용도 늘어난다. 70B 재료로 만든 120B 모델은 70B보다 느리고 비싸다. 병합이 추론 비용을 줄여주는 것은 아니다 — 비용을 줄이려면 증류(distillation)를 써야 한다.

**충돌하는 태스크**: "정중한 응답" 모델과 "직설적 응답" 모델을 병합하면 두 스타일이 충돌해 일관성 없는 출력이 나온다. 태스크 간 상호보완성이 있을 때 병합 효과가 극대화된다.

## 핵심 요약

- **병합은 학습이 아니다**: 기존 체크포인트의 가중치를 수학적으로 재조합하는 것이다. 데이터도 GPU 클러스터도 필요 없다
- **같은 베이스에서 파인튜닝된 모델만** 병합할 수 있다. 아키텍처와 텐서 형태가 정확히 일치해야 한다
- **두 모델은 SLERP, 세 개 이상은 TIES 또는 DARE**로 시작하라. 선형 보간은 가장 단순하지만 간섭에 취약하다
- **mergekit**은 YAML 설정 한 개로 CPU에서도 병합을 실행할 수 있는 사실상의 표준 도구다
- **병합 후 반드시 벤치마크**: 직감에 의존하지 말고, 병합 전후 성능을 정량으로 비교하라

가장 먼저 시도해볼 것은, 이미 파인튜닝해 둔 두 모델이 있다면 그것을 SLERP로 50:50 병합해 보는 것이다. 10분 안에 결과물이 나오고, 원본은 그대로 남아 있으니 실패해도 손해가 없다. 병합은 실험 비용이 거의 제로인 기술이다 — 그래서 가성비가 압도적이다.
