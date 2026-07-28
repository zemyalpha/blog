---
title: "PPO에서 GRPO까지: LLM 정렬 알고리즘이 평가 모델을 버린 과정과 실전 선택 기준"
date: "2026-07-28"
keywords: ["GRPO", "DPO", "RLHF", "SimPO", "RLVR", "LLM 정렬", "선호도 최적화", "DeepSeek R1"]
lang: "ko"
description: "PPO가 평가 모델에 의존하던 RLHF에서 GRPO와 SimPO가 어떻게 벗어났는지, RLVR이 추론 능력을 어떻게 끌어냈는지를 알고리즘 수준에서 분석하고 실전 선택 기준을 정리한다."
---

# PPO에서 GRPO까지: LLM 정렬 알고리즘이 평가 모델을 버린 과정과 실전 선택 기준

2023년 초, ChatGPT가 보여준 RLHF(Reinforcement Learning from Human Feedback) 파이프라인은 명확했다. 사전학습 모델 위에 SFT(Supervised Fine-Tuning)를 얹고, 인간 평가자가 만든 선호도 데이터로 보상 모델(reward model)을 학습한 뒤, PPO(Proximal Policy Optimization)로 정책 모델을 미세조정한다. 이 과정에서 최대 네 개의 모델 — 정책 모델, 참조 모델, 보상 모델, 가치 모델(critic) — 이 동시에 메모리에 올라가야 했다.

그로부터 2년이 지난 지금, 이 파이프라인은 근본적으로 바뀌었다. 2025년부터 2026년 초까지 공개된 주요 모델 — DeepSeek-R1, DAPO 기반 시스템, SimPO 적용 모델 — 은 PPO를 사용하지 않는다. 가치 모델을 없애고(GRPO), 참조 모델을 없애며(SimPO), 아예 RL 자체를 우회하기도 했다(DPO). 가장 극적인 변화는 인간 평가 데이터 없이도 추론 능력이 발현된다는 발견(RLVR)이다.

이 글에서는 PPO에서 GRPO까지 정렬 알고리즘이 어떻게 진화했는지를 수식과 코드 수준에서 추적하고, 각 방법의 트레이드오프와 실전 선택 기준을 정리한다.

## PPO의 짐: 왜 네 개의 모델이 필요했나

PPO(Schulman et al., 2017)는 RLHF에서 가장 오래 쓰인 알고리즘이다. 원래는 게임 RL에 쓰이던 기법인데, OpenAI가 이를 언어 모델 정렬에 적용하면서 RLHF의 표준이 되었다. PPO가 작동하려면 다음 네 가지 구성 요소가 필요하다:

1. **정책 모델(policy model)** $\pi_\theta$ — 실제 학습 대상. 인간의 피드백을 반영하도록 업데이트된다
2. **참조 모델(reference model)** $\pi_{\text{ref}}$ — SFT 단계의 원본 모델. 정책이 너무 멀리 벗어나지 않도록 KL 발산 제약을 건다
3. **보상 모델(reward model)** $r_\phi$ — 인간 선호도 데이터로 학습된 모델. 각 응답에 점수를 부여한다
4. **가치 모델(value/critic model)** $V_\psi$ — 각 프롬프트에 대해 "기대 보상"을 예측하는 모델

가치 모델이 필요한 이유는 어드밴티지(advantage)를 계산하기 위해서다. 어드밴티지는 "이 응답이 평균적으로 기대할 수 있는 것보다 얼마나 좋거나 나쁜가"를 측정하는 값이다. 단순히 보상값 그 자체를 쓰면 쉬운 프롬프트(보상이 높게 나오는)와 어려운 프롬프트(보상이 낮게 나오는)를 구분할 수 없다. 가치 모델은 각 프롬프트의 난이도를 학습하여, 프롬프트 맥락을 고려한 기준선(baseline)을 제공한다.

PPO의 어드밴티지 계산:

$$\hat{A}_t = r_t + \gamma V_\psi(x_{t+1}) - V_\psi(x_t)$$

이 어드밴티지로 정책을 업데이트한다:

$$\mathcal{J}_{\text{PPO}} = \mathbb{E}\left[\min\left(r_t(\theta)\hat{A}_t,\; \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon)\hat{A}_t\right)\right]$$

여기서 $r_t(\theta) = \frac{\pi_\theta(a_t|s_t)}{\pi_{\theta_{\text{old}}}(a_t|s_t)}$는 새 정책과 이전 정책의 확률 비율이다. 클리핑(clipping)은 정책이 한 번에 너무 크게 변하는 것을 막아 학습을 안정화한다.

### PPO의 비용 문제

문제는 가치 모델이 또 하나의 완전한 언어 모델이라는 점이다. 70B 정책 모델을 학습하려면 70B 가치 모델도 같이 올려야 한다. 메모리 사용량이 두 배가 되고, 가치 모델 자체의 학습 노이즈가 전체 학습 불안정으로 이어진다. 또한 가치 모델의 예측이 부정확하면 어드밴티지 계산이 틀려져 정책 업데이트 방향이 어긋난다. 이것이 GRPO가 탄생한 배경이다.

## GRPO: 가치 모델을 그룹 통계로 대체하다

GRPO(Group Relative Policy Optimization)는 DeepSeek팀이 DeepSeekMath(Shao et al., 2024, arXiv 2402.03300)에서 처음 제안하고 DeepSeek-R1에서 본격적으로 사용한 알고리즘이다. 핵심 질문은 단순하다: "가치 모델이 예측하는 기대 보상을, 직접 측정하면 안 되나?"

GRPO의 접근은 각 프롬프트마다 $K$개의 응답을 샘플링하는 것이다. 예를 들어 $K=4$라면, 같은 프롬프트에 대해 네 개의 다른 응답을 생성한다. 각각에 보상을 매기고, 그 평균과 표준편차를 계산한다. 이 그룹 통계가 가치 모델을 대체한다.

### GRPO 알고리즘 단계

각 프롬프트 $x$에 대해:

1. 현재 정책에서 $K$개 응답 샘플링: $y_1, y_2, \ldots, y_K \sim \pi_\theta(\cdot|x)$
2. 각 응답에 보상 부여: $r_1, r_2, \ldots, r_K$
3. 그룹 평균 계산: $\mu = \frac{1}{K}\sum_{i=1}^{K} r_i$
4. 그룹 표준편차 계산: $\sigma = \sqrt{\frac{1}{K}\sum_{i=1}^{K}(r_i - \mu)^2}$
5. 정규화된 어드밴티지: $\hat{A}_i = \frac{r_i - \mu}{\sigma}$

이 어드밴티지를 PPO의 클리핑 목적 함수에 그대로 대입한다. 가치 모델은 사라진다.

구체적인 예시를 보자. 프롬프트가 "$\int_0^3 x^2 \, dx$를 구하라"이고 정답이 9라고 하자. 모델이 네 개의 응답을 생성한다:

| 응답 | 내용 | 보상 |
|------|------|------|
| $y_1$ | 정확하고 간결한 풀이 | 10 |
| $y_2$ | 마지막 산술에서 실수 | 2 |
| $y_3$ | 설정은 맞으나 대수 오류 | 4 |
| $y_4$ | 정확하지만 불필요하게 장황 | 8 |

그룹 평균 $\mu = (10+2+4+8)/4 = 6$, 표준편차 $\sigma \approx 3.16$. 정규화된 어드밴티지:

- $y_1$: $(10-6)/3.16 \approx +1.27$ — 강하게 강화
- $y_2$: $(2-6)/3.16 \approx -1.27$ — 강하게 억제
- $y_3$: $(4-6)/3.16 \approx -0.63$ — 중간 억제
- $y_4$: $(8-6)/3.16 \approx +0.63$ — 중간 강화

이 방식의 장점은 프롬프트 난이도가 자동으로 정규화된다는 것이다. 쉬운 프롬프트에서는 대부분의 응답이 높은 점수를 받아 그룹 평균이 높아지고, 정작 최상위 응답만이 양의 어드밴티지를 받는다. 어려운 프롬프트에서는 대부분 낮은 점수를 받아 그룹 평균이 낮아지고, 그룹 평균을 넘는 응답이 강화된다. 가치 모델이 학습을 통해 습득하던 능력을, 단순한 산술 통계로 직접 관측하는 것이다.

### GRPO의 단점

GRPO가 잃는 것은 토큰 수준의 신용 할당(credit assignment)이다. 하나의 응답 내에서 어느 토큰이 좋았고 어느 토큰이 나빴는지를 구분하지 못한다 — 응답 전체에 동일한 어드밴티지 $\hat{A}_i$를 할당한다. 가치 모델 기반 PPO는 이론적으로 각 토큰 위치마다 별도의 어드밴티지를 추정할 수 있다.

하지만 실제로는 응답 전체를 하나의 단위로 평가하는 작업(정답 여부, 도움 됨 여부)이 대부분이므로, 이 단점은 큰 문제가 되지 않는다. 가치 모델을 유지하는 비용(메모리 두 배, 학습 불안정)에 비하면, 토큰 수준 신용 할당의 손실은 감수할 만한 거래다.

### GRPO의 수학적 정당성

GRPO가 "그냥 작동하는 임시방편"이 아니라는 것을 보이는 연구도 있다. GRPO의 정책 그래디언트가 U-통계량(U-statistic)이라는 분석이 나왔는데, 이는 GRPO가 이상적인 가치 함수에 접근할 수 있는 오라클 알고리즘과 점근적으로 동등하다는 것을 의미한다. 즉, 그룹 크기 $K$가 충분히 크면 가치 모델을 쓰는 것과 수학적으로 같아진다.

## RLVR: 검증 가능한 보상이 만든 추론 능력의 발현

GRPO가 가치 모델을 없앴다면, RLVR(Reinforcement Learning with Verifiable Rewards)은 보상 모델까지 없앴다. 그리고 그 결과가 예상 밖이었다.

RLVR의 핵심 발상은 단순하다. 수학 문제는 답이 맞는지 틀린지를 프로그램으로 검증할 수 있다. 코드는 테스트를 통과하는지 여부로 판단한다. 형식 증명(formal proof)은 증명 검사기(proof checker)가 승인하는지로 결정한다. 이런 보상은 이진 신호(binary signal)이지만, 인간 평가자보다 빠르고, 저렴하고, 일관되다.

DeepSeek-R1은 SFT 없이 순수 RLVR만으로 학습된 모델(R1-Zero)에서 자발적인 사고의 사슬(chain-of-thought) 추론이 발현되는 것을 보였다. 인간이 작성한 추론 예시 없이도 모델이 스스로 성찰(self-reflection)하고 전략을 바꾸는 능력을 개발한 것이다. 이는 LLM 학습에서 가장 주목받은 발견 중 하나다.

RLVR의 파이프라인은 다음과 같다:

```
프롬프트(수학/코딩 문제)
    ↓
정책 모델이 K개 응답 생성 (GRPO 샘플링)
    ↓
검증기(verifier)가 각 응답 평가:
  - 수학: 최종 답이 정답과 일치? → 1 또는 0
  - 코딩: 테스트 통과? → 통과한 테스트 비율
  - 증명: 증명 검사기 승인? → 1 또는 0
    ↓
GRPO 어드밴티지 계산 → 정책 업데이트
```

인간 평가자가 없다. 보상 모델도 없다. 검증기(verifier)만이 보상 신호를 제공한다.

## DAPO: 긴 사고의 사슬에서 RL을 안정화하다

RLVR로 추론 모델을 학습하다 보면 긴 사고의 사슬(chain-of-thought)이 만들어지는데, 긴 시퀀스는 RL 학습을 불안정하게 만든다. DAPO(Yu et al., 2025, arXiv 2503.14476)는 ByteDance와 Tsinghua가 공동 개발한 알고리즘으로, 이 문제를 네 가지 기법으로 해결한다:

1. **Clip-Higher**: PPO의 클리핑 상한을 높여 엔트로피 붕괴(entropy collapse)를 방지. 모델이 계속 탐색하도록 유지
2. **Dynamic Sampling**: 정보량이 적은 배치(모든 응답이 같은 보상을 받는 경우 등)를 필터링하여 그래디언트 신호의 일관성 유지
3. **Token-level Policy Gradient Loss**: 시퀀스 수준 손실이 긴 CoT에서 기울기 소실(vanishing gradient)을 일으키는 문제를 토큰 수준 손실로 해결
4. **Overlong Reward Shaping**: 길이 제한을 초과하는 응답에서 발생하는 보상 노이즈를 감소

DAPO는 Qwen2.5-32B 베이스 모델로 AIME 2024에서 50점을 달성했으며, 학습 코드와 데이터셋을 모두 공개했다. 이는 DeepSeek-R1-Zero보다 50% 적은 학습 스텝으로 동등한 성능을 냈다.

## 선호도 최적화: DPO에서 SimPO, KTO, ORPO까지

RL 기반 방법(GRPO, DAPO)이 추론 능력에 집중하는 동안, 일반 채팅과 정렬(alignment) 영역에서는 DPO(Direct Preference Optimization) 계열이 발전했다.

### DPO: RL 없이 선호도 학습하기

DPO(Rafailov et al., 2023, arXiv 2305.18290)는 RL을 아예 건너뛴다. 핵심 통찰은 "보상 함수를 명시적으로 학습하지 않아도, 선호도 데이터에서 직접 최적 정책을 도출할 수 있다"는 것이다. 수학적으로, RLHF의 최적해는 보상 함수와 정책 사이의 닫힌 형식(closed-form) 관계로 표현되며, 이를 이용하면 보상 모델 학습 + RL의 두 단계를 하나의 지도 학습으로 합친다.

DPO는 선호도 쌍(chosen, rejected)이 필요하다: 같은 프롬프트에 대해 인간(또는 강한 모델)이 더 선호하는 응답과 덜 선호하는 응답을 짝지은 데이터다. 학습 목표는 chosen 응답의 로그 확률을 높이고 rejected 응답의 로그 확률을 낮추되, 참조 모델로부터의 KL 발산으로 정규화하는 것이다.

### SimPO: 참조 모델마저 제거

SimPO(Meng et al., 2024, NeurIPS 2024, arXiv 2405.14734)는 DPO에서 참조 모델까지 없앤다. 응답의 평균 로그 확률을 암묵적 보상(implicit reward)으로 사용하여, 학습 중 메모리에 참조 모델의 복사본을 유지할 필요가 없다. 추가로 Bradley-Terry 목적에 타겟 보상 마진(target reward margin)을 도입하여, 좋은 응답과 나쁜 응답 사이의 간격을 더 크게 벌린다.

논문에 따르면 SimPO는 AlpacaEval 2에서 DPO 대비 최대 6.4점, Arena-Hard에서 최대 7.5점 향상을 달성했다. Gemma-2-9B-it 기반 SimPO 모델은 AlpacaEval 2에서 72.4% 길이 정규화 승률을 기록했으며, 10B 미만 모델 중 Chatbot Arena 1위를 차지했다.

### KTO: 이진 피드백만으로 정렬

KTO(Ethayarajh et al., 2024, ICML 2024, arXiv 2402.01306)는 선호도 "쌍"이 아니라 이진 피드백(좋아요/싫어요)만으로 학습한다. 이는 실제 서비스 환경에서 유용하다 — 사용자의 좋아요/싫어요 버튼, 재생성 신호 등은 짝이 지어진 선호도 데이터보다 훨씬 풍부하게 수집된다. KTO는 카네만-트버스키 전망 이론(prospect theory)을 적용하여 인간의 손실 회피 성향(loss aversion)을 모델링에 반영한다. 1B에서 30B 규모에서 선호도 기반 방법과 동등하거나 더 나은 성능을 보였다.

### ORPO: SFT와 정렬을 하나로

ORPO(Hong et al., 2024, arXiv 2403.07691)는 SFT 단계와 선호도 최적화 단계를 하나의 학습 목적으로 합친다. 승산비(odds ratio)를 사용하여 두 과정을 동시에 수행한다. 단계가 하나로 줄어 학습 시간이 단축되고, SFT와 정렬 사이의 분포 변화(distribution shift)가 제거된다.

### 선호도 최적화 방법 비교

| 방법 | 참조 모델 | 보상 모델 | 데이터 형식 | 추가 학습 단계 |
|------|-----------|-----------|-------------|----------------|
| PPO | 필요 | 필요 | 선호도 순위 | RL 단계 별도 |
| DPO | 필요 | 불필요 | 선호도 쌍 | 불필요 |
| SimPO | 불필요 | 불필요 | 선호도 쌍 | 불필요 |
| KTO | 필요 | 불필요 | 이진(좋아요/싫어요) | 불필요 |
| ORPO | 불필요 | 불필요 | 선호도 쌍 | SFT 통합 |

## 실전에서 어떤 방법을 선택할 것인가

### 추론 모델을 학습할 때: GRPO 또는 DAPO

수학, 코딩, 논리 추론 등 정답 검증이 가능한 도메인에서는 GRPO 또는 DAPO가 선택이다. 핵심 전제는 검증기(verifier)가 존재한다는 것이다:

- **수학**: 최종 답을 정답과 비교하거나, LaTeX/Python으로 평가
- **코딩**: 단위 테스트 통과 여부로 보상
- **형식 증명**: Lean, Coq 등의 증명 검사기

```python
# GRPO 학습 루프 개념 (TRL 프레임워크 기반)
from trl import GRPOTrainer, GRPOConfig

# 검증 가능한 보상 함수
def math_reward(prompts, completions, **kwargs):
    rewards = []
    for prompt, completion in zip(prompts, completions):
        # 정답 추출 및 비교
        extracted_answer = extract_answer(completion)
        correct = check_answer(extracted_answer, prompt["answer"])
        rewards.append(1.0 if correct else 0.0)
    return rewards

config = GRPOConfig(
    num_generations=8,        # K=8: 프롬프트당 8개 응답 샘플링
    max_completion_length=1024,
    beta=0.04,                # KL 패널티 계수
    epsilon=0.2,              # PPO 클리핑 범위
)

trainer = GRPOTrainer(
    model="Qwen/Qwen2.5-7B-Math",
    reward_funcs=math_reward,
    args=config,
    train_dataset=math_dataset,
)
trainer.train()
```

`num_generations`가 GRPO의 그룹 크기 $K$다. 보통 4~64 사이를 쓴다. 크면 통계 추정이 정확해지지만 연산 비용이 선형적으로 증가한다.

### 일반 채팅 정렬: SimPO 또는 KTO

추론이 아닌 일반 대화 품질, 안전성, 유용성을 높이는 경우:

- **선호도 쌍 데이터가 있으면**: SimPO. 참조 모델이 필요 없어 메모리 효율이 가장 좋고, AlpacaEval 등에서 일관되게 DPO를 능가
- **이진 피드백만 있으면**: KTO. 사용자 좋아요/싫어요, 재생성 신호 등을 직접 활용 가능
- **학습 파이프라인을 단순화하려면**: ORPO. SFT와 정렬을 한 단계로

```python
# SimPO 학습 (TRL 프레임워크)
from trl import SimPOTrainer, SimPOConfig

config = SimPOConfig(
    beta=2.7,                    # SimPO 온도 계수
    gamma_beta_ratio=0.3,       # 타겟 보상 마진 비율
    loss_type="simpo",
    max_length=2048,
)

trainer = SimPOTrainer(
    model="meta-llama/Llama-3.1-8B-Instruct",
    args=config,
    train_dataset=preference_pairs,
    # 참조 모델 인자가 없다 — SimPO는 필요 없음
)
trainer.train()
```

### 하이브리드: SFT → SimPO → GRPO

가장 일반적인 2026년의 파이프라인은 세 단계 조합이다:

1. **SFT**: 지시문 따르기, 출력 형식 학습 (1~10M 샘플)
2. **SimPO 또는 DPO**: 일반 대화 품질과 안전성 정렬
3. **GRPO/RLVR**: 추론 능력 강화 (수학, 코딩 등 검증 가능한 도메인)

이 순서가 중요하다. SFT 없이 GRPO를 먼저 돌리면 모델이 기본적인 지시문 따르기도 안 된 상태에서 추론 학습에 들어가 효율이 떨어진다. 선호도 최적화(SimPO)가 추론 RL(GRPO)보다 먼저 와야 하는 이유는, 추론 학습 전에 모델이 기본적으로 안전하고 유용한 응답 형태를 갖추고 있어야 하기 때문이다.

## 남은 과제

### 검증기 노이즈

RLVR의 전제는 검증기가 완벽하다는 것이다. 하지만 실제 검증기는 완벽하지 않다. 수학 검사기의 경계 케이스, 불완전한 테스트 스위트, 형식 증명 검사기의 구멍 — 이런 노이즈는 모델이 거짓 양성(false positive)을 악용하게 만들 수 있다. 최근 연구는 관측된 보상에서 검증기 노이즈를 보정하는 알고리즘을 개발하고 있다.

### 길이 편향

DPO와 그 변종들은 응답 길이에 편향이 있다 — 긴 응답이 더 "정중한" 것처럼 보여 선호도에서 유리한 경향. SimPO는 길이 정규화로 이를 완화하지만, 여전히 특정 워크로드에서는 문제가 된다. DAPO의 Overlong Reward Shaping도 같은 문제를 다른 각도에서 공략한다.

### 분포 외 일반화

RLVR로 수학과 코딩에서 추론 능력을 키운 모델이, 다른 도메인(예: 의학, 법률)에서 같은 추론 능력을 발휘하는지는 미해결 문제다. 검증기가 없는 도메인에서는 여전히 인간 평가나 강한 모델을 통한 보상 모델링이 필요하다.

## 결론

- **PPO의 네 모델 구성은 끝났다** — GRPO가 가치 모델을, SimPO가 참조 모델을, RLVR이 보상 모델을 각각 제거했다. 2026년의 파이프라인은 더 적은 모델로 더 안정적으로 학습한다
- **GRPO는 그룹 통계로 가치 모델을 대체한다** — 프롬프트당 $K$개 응답을 샘플링하여 평균과 표준편차를 구하면, 학습된 가치 함수 없이도 프롬프트 맥락을 반영한 어드밴티지를 얻는다
- **RLVR은 인간 없이 추론 능력을 발현시킨다** — 수학과 코딩에서 검증 가능한 보상만으로 모델이 자발적으로 사고의 사슬을 개발한다. DeepSeek-R1이 이를 입증했다
- **DAPO는 긴 추론 시퀀스의 RL 불안정을 해결한다** — Clip-Higher, Dynamic Sampling, 토큰 수준 손실, Overlong Reward Shaping의 네 기법으로 AIME 2024에서 50점을 달성했다
- **일반 정렬은 SimPO가 현재 최선** — 참조 모델이 필요 없고, AlpacaEval 2에서 DPO 대비 6.4점 향상을 일관되게 보인다

실전에서 가장 먼저 시도할 단계는, 기존 SFT 모델에 SimPO를 적용하는 것이다. 선호도 쌍 데이터(또는 GPT-4로 생성한 합성 선호도 데이터)만 있으면 하루 만에 일반 대화 품질의 눈에 띄는 향상을 볼 수 있다. 추론 능력이 필요하면 그 위에 GRPO + 검증 가능한 보상을 얹는다.
