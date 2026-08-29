---
title: "구조화 출력은 100% 파싱을 보장하지만 정답을 보장하지 않는다: 스키마 유효성과 의미 품질의 품질 세금까지"
date: "2026-08-29"
keywords: ["구조화 출력", "structured outputs", "constrained decoding", "Instructor", "Outlines", "JSON Schema", "LLM 파이프라인"]
lang: "ko"
description: "LLM 구조화 출력의 3단계 신뢰성 계층과 constrained decoding의 동작 원리, 그리고 스키마는 지키지만 정답은 틀리는 '품질 세금' 문제를 실전 코드와 함께 분석한다."
---

# 구조화 출력은 100% 파싱을 보장하지만 정답을 보장하지 않는다

LLM 파이프라인을 운영해본 사람이라면 한 번쯤 겪었을 장면이 있다. 테스트에서 97% 성공률을 보였던 JSON 추출 파이프라인이 실제 트래픽에서는 마크다운 코드 펜스에 감싸인 응답, 숫자여야 할 필드의 문자열, 아예 생략된 필드 때문에 새벽 두 시에 장애 알림을 보낸다. 이런 실패는 벤치마크에 잘 잡히지 않고, 멀티스텝 파이프라인에서는 예외가 아니라 오염된 데이터로 전파되기 때문에 발견이 더 늦어진다.

2026년 현재 이 문제의 정답은 명확해 보인다. OpenAI의 strict 모드, Anthropic의 tool use 강제, vLLM의 guided decoding 같은 스키마 강제 기능을 쓰면 파싱 실패율을 사실상 0으로 만들 수 있다. 그런데 정말 중요한 문제는 그 다음에 시작된다. **문법적으로 완벽한 JSON이 반환되더라도, 그 안의 값이 모델이 자유롭게 답했을 때와 같은 값이라는 보장은 어디에도 없다.**

이 글에서는 구조화 출력의 신뢰성 계층부터 시작해, constrained decoding이 실제로 어떻게 동작하는지, 그리고 스키마 검증을 통과한 출력의 의미 품질까지 평가하는 방법까지 다룬다.

## 신뢰성의 3단계: 어느 계층에서 제약을 걸 것인가

구조화 출력 구현 방식은 제약이 걸리는 위치에 따라 세 단계로 나뉜다.

| 단계 | 방식 | 성공률 | 적합한 용도 |
|------|------|--------|------------|
| 1 | 프롬프트에 "JSON으로 답해라" | 80~95% | 프로토타입 |
| 2 | 함수 호출 / tool use | 95~99% | 일반 API 앱 |
| 3 | 네이티브 구조화 출력 (strict) | 99%+ | 데이터 파이프라인 |

1단계가 실패하는 이유는 구조적이다. "JSON으로 반환하라"는 프롬프트는 모델에게 스타일 선호 정도로 해석될 뿐 하드 제약이 아니다. 입력이 복잡하거나 컨텍스트가 길어지면 모델은 형식 준수를 우선순위에서 밀어낸다. 흔한 실패 양상은 예측 가능하다 — Python 딕셔너리 표기에 익숙한 모델이 홑따옴표를 쓰거나, JavaScript에서는 유효한 trailing comma를 남기거나, "Sure! Here's the JSON:" 같은 안내 문구를 JSON 앞에 붙이는 것이다.

토큰 단위로 보면 실패율은 곱해진다. 토큰당 99% 정확도의 모델이 200토큰짜리 JSON을 생성하면 수학적으로 유효한 결과가 나올 확률은 약 87%뿐이다. 토큰당 98%라면 70%까지 떨어진다. 오류율은 더해지지 않고 곱해진다는 것이 프롬프트 기반 JSON이 근본적으로 신뢰할 수 없는 이유다.

3단계의 결정적 차이는 **스키마 위반이 '확률적으로 낮아지는' 것이 아니라 '물리적으로 불가능해진다'**는 점이다. OpenAI strict 모드의 공식 문서는 이를 명시한다 — 응답이 반환되기만 하면 반드시 주어진 JSON Schema를 만족한다.

## Constrained Decoding: 토큰 생성 자체를 바꾸는 메커니즘

네이티브 구조화 출력의 핵심은 constrained decoding(제약 디코딩)이다. 동작 원리는 의외로 단순하다.

1. 모델이 매 스텝 전체 어휘에 확률을 배분한다
2. 시스템이 지금까지 생성된 부분 출력을 보고 스키마상 허용되는 토큰 집합을 계산한다
3. 위반될 토큰의 확률을 0으로 만든다 (로그잇 마스킹)
4. 모델은 허용된 토큰 안에서만 자기 분포대로 선택한다

실무에서는 JSON Schema를 유한 상태 기계(FSM)로 사전 컴파일해 스텝당 O(1) 조회로 돌리기 때문에 추론 오버헤드가 크지 않다. 오픈소스 진영에서는 dottxt-ai의 Outlines가 이 방식의 대표 구현이며, vLLM의 구조화 출력 기능에 통합되어 있어 자체 호스팅 모델에서도 같은 보장을 받을 수 있다. vLLM 공식 문서의 Structured Outputs 페이지에서 `StructuredOutputsParams`의 `json`, `regex`, `grammar` 등의 옵션으로 이 기능을 확인할 수 있다. 참고로 예전 버전의 `guided_json` 같은 파라미터는 v0.12.0에서 제거되고 `structured_outputs` 방식으로 바뀌었으니, 오래된 튜토리얼을 따라 할 때 주의해야 한다.

여기서 중요한 뉘앙스가 하나 있다. constrained decoding은 모델의 **추론이나 지식을 제약하지 않는다**. 모델은 여전히 자기가 생각하는 "옳은 답"의 방향으로 토큰을 고른다. 구조적으로 유효하지 않은 토큰만 봉쇄될 뿐이다. 그래서 출력 품질을 해치지 않으면서 구조 실패를 없앨 수 있다 — 라고 알려져 있다. 다음 섹션에서 이 통념의 구멍을 본다.

## 스키마는 통과했는데 정답이 틀리다: 품질 세금

여기가 이 글의 핵심이다. Future AGI의 2026년 평가 리포트는 흥미로운 사례를 소개한다. OpenAI strict 모드로 99.4%의 스키마 유효률을 달성한 추출 파이프라인이 있었다. 모든 JSON은 파싱되고 모든 필드 제약은 통과했다. 그런데 3주 뒤 고객 지원팀이 인입 티켓의 약 11%에서 `priority: 'urgent'`가 잘못 선택된 것을 발견했다. 정답은 'normal'이나 'high'였는데, 모델이 문법을 지키기 위해 "더 안전한" enum 기본값으로 붕괴(collapse)한 것이다. 스키마 검증은 이 오류를 잡을 방법이 없었다.

이것이 **품질 세금(quality tax)**이다. 제약 디코더가 어려운 프롬프트에서 모델이 원래 선택했을 토큰을 마스킹하면, 모델은 스키마상 합법적인 연속 중 가장 낮은 퍼플렉시티를 가진 것으로 미끄러진다. 그리고 그것이 정답이 아닌 경우가 있다. 특히:

- **약한 모델일수록 세금이 크다.** 강한 모델은 제약 안에서도 좋은 답을 찾지만, 약한 모델은 그냥 기본값으로 무너진다.
- **enum 필드가 붕괴 지점이다.** 몇 개 안 되는 선택지 중 하나로 수렴하므로 오류가 통계에 숨는다.
- **선택지가 많은 union 타입은 조용히 실패한다.** Gemini의 responseSchema 같은 일부 구현에서 보고되는 실패 양상이다.

결론적으로 평가해야 할 지표는 스키마 유효률 하나가 아니라 **schema_validity_rate × semantic_quality_on_passes**, 즉 "스키마를 통과한 것들 중 의미적으로 맞은 비율"의 곱이다. 이 곱셈을 모드별(OpenAI strict / Anthropic tool use / Gemini responseSchema / Outlines)로, 자기 데이터 위에서 측정해야 한다. 파싱률만 보고 모드를 고르는 것은 폰트 고르기와 다름없다.

## 실전 구성: 스키마 강제 + 검증 재시도의 이중 레이어

그렇다면 프로덕션에서는 어떻게 쌓아야 할까. 나는 두 겹을 권한다. 첫 번째 겹은 API/서빙 레벨의 스키마 강제, 두 번째 겕은 Pydantic 검증 + 오류 피드백 재시도다.

**첫 번째 겹 — OpenAI strict 모드:**

```python
from openai import OpenAI
from pydantic import BaseModel

class Ticket(BaseModel):
    summary: str
    priority: str  # "low" | "normal" | "high" | "urgent"

client = OpenAI()
resp = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    messages=[{"role": "user", "content": ticket_text}],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "ticket",
            "strict": True,
            "schema": Ticket.model_json_schema(),
        },
    },
)
```

**두 번째 겹 — Instructor로 검증 재시도 붙이기:**

```python
import instructor
from anthropic import Anthropic
from pydantic import BaseModel, Field
from typing import Literal

class Ticket(BaseModel):
    summary: str = Field(description="티켓 본문을 한 문장으로 요약")
    priority: Literal["low", "normal", "high", "urgent"] = Field(
        description="명시적 표현 기준. 사용자가 '당장 안 됨' 등 긴급 표현을 쓴 경우에만 urgent"
    )

client = instructor.from_anthropic(Anthropic())
ticket = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": ticket_text}],
    response_model=Ticket,
    max_retries=3,
)
```

Instructor는 이 패턴의 사실상 표준 라이브러리다. 2026년 8월 현재 PyPI 월간 다운로드 약 2,760만 회, GitHub 스타 1.37만 개를 기록 중이다(본인 확인 수치). Pydantic 검증이 실패하면 ValidationError 메시지를 대화 이력에 붙여 재요청하기 때문에, 맹목적 재시도와 달리 모델이 무엇이 틀렸는지 알고 자기 수정을 한다. 실전에서는 대부분의 검증 실패가 1~2회 재시도 안에 해결된다.

**자체 호스팅이라면 — vLLM structured_outputs:**

```python
from vllm import LLM, SamplingParams
from vllm.sampling_params import StructuredOutputsParams

llm = LLM(model="Qwen/Qwen2.5-32B-Instruct")
sampling_params = SamplingParams(
    structured_outputs=StructuredOutputsParams(json=Ticket.model_json_schema()),
    temperature=0.2,
)
out = llm.chat(
    messages=[{"role": "user", "content": ticket_text}],
    sampling_params=sampling_params,
)
```

## 스키마 설계 자체가 신뢰성 엔지니어링이다

제약 디코딩을 붙이기 전에 스키마부터 고쳐야 한다. 완벽한 강제 아래에서도 나쁜 스키마는 "스키마 준수하지만 의미적으로 틀린" 출력을 만들어내기 때문이다.

- **중첩은 2~3단계까지.** 그 이상에서는 실패율이 비선형적으로 오르고 디버깅도 어려워진다. 4단계 중첩이 필요해 보이면 스키마를 재구성하라는 신호다.
- **필드 description은 프롬프트다.** description 없는 `sentiment` 필드는 모델 나름의 감정 정의를 받아온다. "명시적 어조 기준, 함축적 의도 제외"처럼 판정 기준을 적어야 일관성이 생긴다.
- **reasoning 필드를 answer 앞에 배치하라.** 모델은 필드를 순서대로 생성하므로, 답을 확정하기 전에 추론을 거치게 만들면 답의 품질이 올라간다. 프롬프트 레벨 chain-of-thought를 스키마 레벨로 옮기는 셈이고, 이쪽이 모델이 추론 단계를 건너뛸 수 없어 더 확실하다.
- **required는 명시적으로.** 스키마에 필드가 존재하는 것과 required로 표시된 것은 "가끔 포함"과 "항상 포함"의 차이다.

## 결론

- 구조화 출력의 신뢰성은 3단계 계층으로 나뉘고, 파이프라인 용도라면 constrained decoding 기반의 네이티브 강제가 유일한 방어 가능한 선택이다.
- 다만 스키마 유효률과 의미 품질은 다른 축이다. strict 모드는 파싱을 보장하지만 enum 붕괴 같은 "품질 세금"을 숨긴다. 실제로는 한 사례 보고에서 스키마 유효률 99.4%인 파이프라인의 11%가 우선순위를 잘못 분류했다.
- 평가 지표는 `schema_validity_rate × semantic_quality_on_passes`의 곱으로, 자기 데이터에서 모드별로 측정해야 한다.
- 프로덕션 구성은 "API 레벨 스키마 강제 + Pydantic 검증 재시도(Instructor)"의 이중 레이어가 2026년 현재의 표준 패턴이다. 자체 호스팅이라면 vLLM structured_outputs + Outlines 계열이 동등한 보장을 제공한다.
- 당장 할 수 있는 첫 단계: 기존 파이프라인의 JSON 추출을 strict 모드로 바꾸는 것보다, 먼저 지금 출력 중 스키마를 통과한 것들의 **의미 정확도를 샘플링 측정**하는 편이 우선이다. 파싱 실패는 눈에 보이지만 품질 세금은 보이지 않는다.

**참고 자료**: OpenAI Structured Outputs 공식 문서(developers.openai.com/api/docs/guides/structured-outputs), vLLM Structured Outputs 문서(docs.vllm.ai), Instructor(github.com/instructor-ai/instructor), Outlines(github.com/dottxt-ai/outlines), Future AGI "Evaluating LLM Structured Output Modes 2026"
