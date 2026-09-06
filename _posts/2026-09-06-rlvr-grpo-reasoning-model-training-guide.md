---
title: "정답만 확인하면 학습이 된다: RLVR과 GRPO로 추론 모델을 훈련하는 원리와 실전"
date: "2026-09-06"
keywords: ["RLVR", "GRPO", "강화학습", "추론 모델", "DeepSeek-R1"]
lang: "ko"
description: "RLHF의 보상 모델 대신 코드로 정답을 검증하는 RLVR의 작동 원리, GRPO의 그룹 상대 어드밴티지 수식, TRL·verl로 직접 훈련하는 방법, 그리고 pass@k 논쟁이 밝힌 한계까지 정리한다."
---

# 정답만 확인하면 학습이 된다: RLVR과 GRPO로 추론 모델을 훈련하는 원리와 실전

2025년 초 DeepSeek-R1이 공개되었을 때 사람들이 가장 놀란 지점은 모델 성능 자체보다 훈련 방법이었다. 사람이 좋다/나쁘다 평가한 데이터로 보상 모델을 학습하는 RLHF가 아니라, 수학 문제의 최종 답이 맞는지만 기계적으로 확인해 보상을 주는 방식으로, 모델이 스스로 답을 검토하고 되돌아가는 추론 행동을 획득했다는 사실 때문이었다. 이 레시피는 이후 오픈소스 진영 전체로 퍼졌고, 지금 추론 모델 포스트트레이닝의 사실상 표준이 되었다.

이 글에서는 이 방식 — RLVR(Reinforcement Learning with Verifiable Rewards) — 이 정확히 무엇인지, 함께 쓰이는 GRPO 알고리즘이 어떻게 동작하는지, 직접 돌려볼 수 있는 프레임워크는 무엇인지, 그리고 "정말 새로운 추론 능력을 가르치는가"라는 현재 진행 중인 논쟁까지 정리한다.

## RLVR은 무엇이고, RLHF와 뭐가 다른가

RLVR의 정의는 한 문장으로 끝난다. **강화학습으로 모델을 fine-tuning하되, 보상을 학습된 보상 모델이나 사람 평가자가 아니라 자동 검증기(verifier)에서 받는다.** 수학 문제라면 최종 답을 파싱해 정답과 비교하고, 코딩 문제라면 테스트 케이스를 돌려본다. 맞으면 1, 틀리면 0. 그게 전부다.

이 차이가 왜 중요한지는 RLHF의 구조를 떠올리면 명확해진다.

- **RLHF**: 사람이 응답 쌍에 선호 라벨을 붙인다 → 그 라벨로 보상 모델을 학습한다 → 정책 모델이 보상 모델을 최대화하도록 학습한다. 보상 모델은 "사람의 취향에 대한 근사"이므로, 길이가 길수록, 자신감 있는 톤일수록 점수를 높게 주는 식으로 게임당할 수 있다.
- **RLVR**: 검증기는 정답과 일치하는지만 본다. 아무리 그럴듯하게 써도 x가 5가 아니면 0점이다. 라벨링 비용도 없고, 프록시를 과최적화할 여지도 구조적으로 줄어든다.

RLVR이라는 이름은 Ai2의 Tulu 3 논문("Tulu 3: Pushing Frontiers in Open Language Model Post-Training", arXiv:2411.15124, 2024년 11월)에서 처음 공식화됐다. Tulu 3는 SFT → DPO → RLVR의 3단계 레시피를 데이터와 코드까지 전부 공개했고, Llama 3.1 베이스 모델이 GPT-4o-mini나 Claude 3.5-Haiku의 instruct 버전을 넘어서는 결과를 내며 이 접근의 유효성을 보여줬다. 다만 기술 자체의 씨앗은 더 빨리 뿌려져 있었다 — DeepSeek가 2024년 2월 DeepSeekMath 논문에서 수학 정답 여부를 규칙 기반 보상으로 쓰고 전용 알고리즘 GRPO를 도입한 것이 시작이고, 2025년 1월 DeepSeek-R1(arXiv:2501.12948, 이후 Nature 게재)이 이를 프론티어급 규모로 확장하며 대중화됐다.

## GRPO: 크리틱을 지우고 그룹 평균을 baseline으로 쓴다

RLVR은 "보상을 어디서 받느냐"의 문제이고, 실제 학습 알고리즘은 대부분 GRPO(Group Relative Policy Optimization)가 담당한다. GRPO가 풀려는 문제는 PPO의 구조적 비용이다.

PPO에서 어드밴티지(이 행동이 평균보다 얼마나 좋은가)를 계산하려면 상태 가치를 추정하는 별도의 신경망, 즉 크리틱(value network)이 필요하다. 이 크리틱은 정책 모델과 비슷한 크기라 GPU 메모리를 크게 잡아먹고, 학습도 불안정해지기 쉬운 요소다.

GRPO의 발상은 단순하다. 어차피 프롬프트마다 여러 응답을 샘플링한다면, **그 그룹 자체의 평균 보상이 baseline이 되면 된다**는 것. 각 응답의 어드밴티지는 그룹 통계로 정규화한 값이 된다:

```
A_i = (r_i − mean(r_1..r_G)) / std(r_1..r_G)
```

프롬프트 하나에 8개 답을 뽑았는데 3개가 정답이면, 정답인 3개는 플러스 어드밴티지를 받아 확률이 올라가고, 틀린 5개는 마이너스 어드밴티지를 받아 내려간다. 크리틱이 사라졌으니 메모리와 학습 파이프라인이 단순해진다. 크리틱 제거로 메모리를 상당 폭 절약할 수 있다는 점은 DeepSeekMath 논문 자체가 명시한 동기이며, 일부 분석에서는 약 40% 수준의 절감으로 추정하기도 한다.

binary 보상(0 또는 1)과의 궁합도 좋다. 보상 모델처럼 연속 점수를 정교하게 매길 필요 없이, 그룹 내 상대 비교만으로 학습 신호가 만들어진다. 이론적으로도 GRPO의 손실 함수가 그룹 통계로 보상을 보정하는 가중 대비 손실(weighted contrastive loss)로 해석될 수 있고, 반복 학습 시 성공 확률이 참조 정책을 넘는 고정점으로 수렴한다는 분석이 있다(Mroueh, "Reinforcement Learning with Verifiable Rewards: GRPO's Effective Loss, Dynamics, and Success Amplification", arXiv:2503.06639).

2025년 이후 GRPO의 알려진 편향을 고친 변형들도 등장했다. Dr. GRPO와 DAPO는 길이·난이도 편향을 진단하고 보정했으며(DAPO는 AIME 2024에서 50점을 달성한 오픈 레시피로 보고됐다), Qwen 팀의 GSPO는 중요도 비율을 토큰 단위에서 시퀀스 단위로 옮겨 MoE 모델의 RL을 안정화했다. 최근 프로덕션 훈련에서는 KL 정규화 항을 0으로 두는 설정도 흔하다.

## 실전: TRL GRPOTrainer로 수학 검증기 돌려보기

직접 실험해보는 가장 낮은 진입장벽은 Hugging Face TRL의 `GRPOTrainer`다. TRL 공식 문서는 GRPO를 DeepSeekMath 논문 기반으로 설명하며, 커스텀 보상 함수를 파이썬으로 정의해 넘기면 된다. 검증기가 파이썬 함수일 뿐이라는 점이 RLVR의 실용성을 잘 보여준다.

```python
from datasets import load_dataset
from trl import GRPOConfig, GRPOTrainer

# (질문, 정답) 쌍으로 된 데이터셋. 사람이 응답 전체를 라벨링할 필요가 없다.
dataset = load_dataset("microsoft/orca-math-word-problems-200k", split="train")

def verify_math_reward(completions, answer, **kwargs):
    """RLVR의 전부: 최종 답을 추출해 정답과 비교해 0/1 보상."""
    import re
    rewards = []
    for completion, gold in zip(completions, answer):
        # \boxed{} 또는 "정답은 X" 형태에서 최종 답 추출
        match = re.search(r"\\boxed\{([^}]+)\}", completion)
        final = match.group(1).strip() if match else None
        rewards.append(1.0 if final == gold.strip() else 0.0)
    return rewards

trainer = GRPOTrainer(
    model="Qwen/Qwen2.5-1.5B-Instruct",
    reward_funcs=verify_math_reward,
    args=GRPOConfig(
        num_generations=8,      # 프롬프트당 그룹 크기 G
        max_completion_length=1024,
        beta=0.0,               # KL 정규화 강도 (0으로 두는 최근 추세)
        per_device_train_batch_size=8,
    ),
    train_dataset=dataset.map(lambda x: {"prompt": x["question"], "answer": x["answer"]}),
)
trainer.train()
```

핵심은 `verify_math_reward`가 모델 출력의 품질을 전혀 평가하지 않는다는 점이다. 문법도, 논리 전개도 보지 않고 최종 답의 일치 여부만 본다. 그럼에도 그룹 내에서 정답을 맞힌 롤아웃의 추론 경로 전체가 상향되면서, "정답에 도달하는 사고과정"이 강화된다.

더 큰 규모로 가려면 프레임워크 선택이 갈린다. TRL은 진입장벽이 낮은 대신 대규모 분산에 최적화되어 있지 않고, ByteDance가 오픈소스한 verl과 OpenRLHF가 Ray 기반 분산 롤아웃·훈련을 지원해 실무 대규모 RLVR의 주류다. 7B 이하 모델 실험은 TRL, 그 이상은 verl/OpenRLHF로 시작하는 것이 일반적인 선택 기준이다.

## pass@k 논쟁: RLVR은 새 추론을 가르치는가, 아니면 드러내는가

여기서 RLVR에 대한 가장 뜨거운 학술적 논쟁을 짚고 가야 한다. "RLVR이 base 모델에 없던 새로운 추론 능력을 만들어내는가?"이다.

Yue 등의 논문 "Does Reinforcement Learning Really Incentivize Reasoning Capacity in LLMs Beyond the Base Model?"(arXiv:2504.13837)는 이 질문에 대해 반증적인 결과를 보고했다. 여러 모델 패밀리와 RL 알고리즘, 수학/코딩/시각 추론 벤치마크에서 pass@k로 능력 경계를 측정한 결과:

- **pass@1 (k가 작을 때)**: RLVR 훈련 모델이 base 모델을 크게 앞선다. 실제 서비스 관점에서 중요한 지표다.
- **pass@k (k가 클 때)**: 상황이 역전된다. 충분히 많이 샘플링하면 base 모델이 RLVR 모델보다 오히려 높은 pass@k를 기록한다.
- 이 논문은 이를 근거로 RLVR이 "새로운 추론 패턴을 만드는" 것이 아니라 base 모델이 이미 갖고 있던 정답 경로로의 확률을 집중시키는(sharpening) 것이라고 해석했다. 반면 교사 모델로부터의 distillation은 base 모델에 없던 새 추론 패턴을 실제로 도입할 수 있다고 본다.

이 해석에 대해 다른 연구자들이 이의를 제기하며 논쟁이 계속되고 있지만(일부 후속 연구는 평가 방법론 자체를 문제 삼았다), 실무자가 취할 교훈은 명확하다. **RLVR 훈련 모델의 pass@1 향상은 사실이지만, 그것이 곧 base 모델의 능력 상한을 넘어선 것은 아니라는 점을 염두에 두어야 한다.** RLVR은 "모델이 한 번에 정답 경로를 찾도록 만드는" 기술로 이해하는 것이 정확하고, 능력 자체의 확장이 필요하다면 distillation이나 사전학습 단계의 개선이 필요하다.

## 결론: RLVR을 도입할 때 기억할 것

- RLVR은 보상의 출처를 바꾼 것이다. 학습된 보상 모델 대신 결정론적 검증기. 사람 라벨링 비용이 없고 구조적으로 게임당하기 어렵다.
- GRPO는 그 학습을 가능하게 한 알고리즘이다. 그룹 평균을 baseline으로 써서 크리틱을 지웠고, binary 보상과 궁합이 좋다. Dr. GRPO·DAPO·GSPO 같은 개량 변형이 뒤를 잇고 있다.
- 진입장벽은 생각보다 낮다. TRL의 `GRPOTrainer`에 파이썬 검증 함수 하나만 정의하면 수학·코딩 도메인에서 바로 실험할 수 있다. 대규모는 verl이나 OpenRLHF로.
- pass@k 논쟁이 알려준 한계를 인정하라. RLVR은 base 모델의 능력을 "한 번에 발휘하도록" 만드는 기술이지, 능력 상한 자체를 올리는 마법이 아니다.
- 첫 실험으로 추천하는 경로는 이것이다: 작은 instruct 모델(1~3B) + 검증 가능한 데이터셋(수학 워드프로블럼 등) + TRL GRPOTrainer + 그룹 크기 8. 하루 안에 첫 RLVR 훈련을 끝낼 수 있다.

보상이 정직하면 학습도 정직해진다. RLVR의 교훈은 결국 그 문장으로 요약된다.
