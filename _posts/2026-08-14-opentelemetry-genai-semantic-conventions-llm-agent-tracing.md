---
title: "OpenTelemetry가 LLM 콜 내부를 들여다보는 법: GenAI Semantic Conventions로 에이전트 트리를 추적하는 실전 가이드"
date: "2026-08-14"
keywords: ["OpenTelemetry", "GenAI Semantic Conventions", "LLM 관측성", "에이전트 트레이싱", "TTFT", "분산 트레이싱", "OTel"]
lang: "ko"
description: "2026년 5월 CNCF 졸업과 함께 자리 잡은 OpenTelemetry GenAI Semantic Conventions의 span·metric·event 정의를 분해하고, LLM 호출과 멀티 에이전트 워크플로를 단일 트리로 추적하는 방법을 코드로 구현한다."
---

# OpenTelemetry가 LLM 콜 내부를 들여다보는 법: GenAI Semantic Conventions로 에이전트 트리를 추적하는 실전 가이드

LLM을 프로덕션에 넣으면 로그 기반 디버깅이 즉시 한계에 부딪힌다. 사용자가 "답변이 이상해요"라고 신고했을 때, 여러분이 가진 것은 API 게이트웨이의 HTTP 200 로그와, 어딘가에서 호출된 LLM의 토큰 사용량 카운터뿐이다. 어떤 프롬프트가 들어갔는지, 에이전트가 몇 개의 서브태스크로 작업을 분할했는지, 첫 번째 토큰이 지연된 것인지 전체 추론이 느린 것인지 — 이 모든 것이 블랙박스다.

2026년 5월 21일, CNCF가 OpenTelemetry를 졸업 프로젝트(Graduated Project)로 승격하면서 이 문제에 대한 업계의 답이 분명해졌다. OpenTelemetry는 단순한 분산 트레이싱 표준이 아니라, GenAI 워크로드를 위한 전용 시맨틱 컨벤션(Semantic Conventions)을 갖춘 관측성의 사실상 표준이다. 이 글에서는 그 컨벤션이 정확히 무엇을 정의하는지, LLM 호출과 에이전트 워크플로를 어떻게 단일 트리로 시각화하는지, 그리고 실제로 계측 코드를 어떻게 작성하는지를 다룬다.

## 왜 기존 HTTP/DB 계측으로는 부족한가

전통적인 OpenTelemetry 계측은 HTTP 요청, 데이터베이스 쿼리, 메시지 큐를 대상으로 설계됐다. 캡처하는 핵심 신호는 요청 지속 시간, 상태 코드, 에러율이다. 이것은 CRUD API에는 충분하지만, LLM 파이프라인에는 해당하지 않는다.

LLM 워크로드에서 진짜로 알아야 할 신호는 다르다:

- **토큰 수 (입력/출력/추론)**: 비용 귀속(cost attribution)의 기준. HTTP 바이트 수와는 전혀 다른 단위다.
- **모델 식별자와 제공자**: `gpt-4o`와 `claude-sonnet-4`는 응답 품질과 지연 패턴이 다르다. 모델 버전별로 성능을 비교하려면 이 정보가 span에 있어야 한다.
- **TTFT (Time To First Token) vs 종단 간 지연**: 둘은 다른 장애를 가리킨다. TTFT가 높으면 프리필(prefill) 단계나 큐 대기가 병목이고, 종단 지연만 높으면 디코딩 자체가 느린 것이다.
- **에이전트 간 서브태스크 위임**: 한 에이전트가 세 개의 하위 에이전트를 호출하고, 각각이 LLM을 부르면, 인과 관계가 깊이 중첩된다. 로그로는 이 트리를 복원할 수 없다.
- **도구 호출과 결과**: 에이전트가 어떤 도구를 호출했고, 그 결과가 무엇이었는지가 의사결정 경로를 설명한다.

GenAI Semantic Conventions는 이 다섯 가지 차원에 대해 일관된 속성 스키마를 정의한다. 핵심은 "벤더마다 다르게 부르던 이름을 통일했다"는 점이다. OpenAI, Anthropic, AWS Bedrock 모두 `gen_ai.request.model`이라는 같은 속성 키를 사용하므로, 백엔드를 바꿔도 계측 코드를 다시 작성할 필요가 없다.

## GenAI Semantic Conventions가 정의하는 4가지 신호

이 컨벤션은 `open-telemetry/semantic-conventions-genai` 리포에서 관리되며, 2026년 현재 Development(개발) 상태다. 4가지 신호 유형을 표준화한다.

### 1. LLM 클라이언트 Span — 개별 모델 호출

가장 기본이 되는 단위. LLM 제공자에 대한 단일 호출을 나타낸다. 핵심 속성은 다음과 같다:

| 속성 키 | 설명 | 예시 |
|---------|------|------|
| `gen_ai.provider.name` | GenAI 제공자 식별자 | `openai`, `gcp.gen_ai`, `gcp.vertex_ai` |
| `gen_ai.request.model` | 요청된 모델 이름 | `gpt-4o`, `claude-sonnet-4` |
| `gen_ai.usage.input_tokens` | 입력 토큰 수 | `1234` |
| `gen_ai.usage.output_tokens` | 출력 토큰 수 | `512` |
| `gen_ai.response.finish_reasons` | 응답 종료 사유 | `["end_turn"]`, `["stop"]` |

`gen_ai.provider.name`은 제공자를 식별하고, `gen_ai.request.model`은 구체적 모델을 식별한다. 이 둘을 결합하면 "OpenAI 호출 중 GPT-4o의 평균 출력 토큰 수는?" 같은 쿼리가 가능해진다.

### 2. 에이전트 Span — 오케스트레이션과 위임

멀티 에이전트 시스템에서 에이전트 단위의 작업을 추적한다. 스펙은 다음 span 유형을 정의한다:

- **`create_agent`**: 에이전트 생성 (원격 에이전트 서비스 사용 시)
- **`invoke_agent`**: 에이전트 호출 (클라이언트 span 또는 내부 span)
- **`invoke_workflow`**: 워크플로 호출
- **`plan`**: 에이전트의 계획 수립 단계
- **`execute_tool`**: 도구 실행

각 에이전트는 `gen_ai.agent.id`와 `gen_ai.agent.name`으로 식별된다. 중요한 점은 이 span들이 부모-자식 관계를 형성한다는 것이다. 상위 에이전트가 하위 에이전트를 호출하면, 호출되는 쪽의 span이 호출하는 쪽의 span 아래에 자식으로 붙는다. 이것이 멀티 에이전트 시스템을 단일 트리로 시각화하는 메커니즘이다.

### 3. 이벤트 (Events) — 요청 상세 정보와 평가 결과

span 속성 외에도, 컨벤션은 구조화된 페이로드를 가진 span 이벤트를 정의한다. 2026년 현재 스펙에 공식적으로 정의된 이벤트는 두 가지다:

- `gen_ai.client.inference.operation.details` — GenAI 완성 요청의 상세 정보(채팅 히스토리, 파라미터 등)를 저장. 트레이스와 독립적으로 입력/출력을 보관할 때 사용한다.
- `gen_ai.evaluation.result` — 모델 출력의 평가 결과.

이벤트는 span 속성에 넣기에는 너무 큰 데이터(예: 전체 프롬프트 텍스트, 전체 응답, 대화 맥락)를 담는 데 적합하다. 토큰 카운트만 span 속성에 넣고, 실제 텍스트와 파라미터는 이벤트로 기록하는 패턴이 권장된다. 단, 이벤트 신호는 언어별 SDK에서 아직 구현되지 않은 경우가 있으므로(op-in 상태), 도입 전 해당 언어의 spec-compliance 매트릭스를 확인해야 한다.

### 4. 메트릭 (Metrics) — 일관된 쿼리가 가능한 정의

관측성 백엔드가 동일한 방식으로 쿼리할 수 있는 표준 메트릭 정의:

| 메트릭 | 타입 | 설명 |
|--------|------|------|
| `gen_ai.client.token.usage` | Histogram | 입력/출력 토큰 사용량 |
| `gen_ai.client.operation.duration` | Histogram | 클라이언트 측 총 호출 지속 시간 |
| `gen_ai.client.operation.time_to_first_chunk` | Histogram | 첫 번째 응답 청크까지의 시간 |
| `gen_ai.server.time_to_first_token` | Histogram | 서버 측 TTFT |
| `gen_ai.server.time_per_output_token` | Histogram | 출력 토큰당 시간 |
| `gen_ai.invoke_agent.duration` | Histogram | 에이전트 호출 지속 시간 |
| `gen_ai.invoke_agent.tool_calls` | Counter | 에이전트의 도구 호출 수 |
| `gen_ai.execute_tool.duration` | Histogram | 도구 실행 지속 시간 |

`gen_ai.client.token.usage`는 명시적 버킷 경계(1, 4, 16, 64, 256, 1024, 4096, …)를 권장한다. 토큰 수는 멱급수 분포를 따르는 경우가 많아, 균등 버킷보다 지수 버킷이 의미 있는 히스토그램을 만든다.

클라이언트 메트릭과 서버 메트릭이 구분되어 있다는 점에 주목하자. `time_to_first_chunk`는 클라이언트가 관측하는 값(네트워크 지연 포함)이고, `time_to_first_token`은 모델 서버가 보고하는 값이다. 둘의 차이가 크면 네트워크나 큐가 병목이라는 뜻이다.

## 실전: LLM 호출을 계측하는 최소 코드

가장 우선순위가 높은 것은 LLM 클라이언트 호출에 span을 입히는 것이다. 아래는 OpenAI Python SDK를 호출할 때 GenAI 컨벤션에 맞춰 span을 생성하는 예시다:

```python
from openai import OpenAI
from opentelemetry import trace

client = OpenAI()
tracer = trace.get_tracer("my-ai-service")

def call_llm(prompt: str, model: str = "gpt-4o") -> str:
    with tracer.start_as_current_span("chat gpt-4o") as span:
        # 스펙 권장 span 이름: "{gen_ai.operation.name} {gen_ai.request.model}"
        # 핵심 속성 — GenAI Semantic Conventions 준수
        span.set_attribute("gen_ai.provider.name", "openai")
        span.set_attribute("gen_ai.request.model", model)
        span.set_attribute("gen_ai.operation.name", "chat")

        response = client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            max_tokens=1024,
        )

        # 응답 메타데이터 기록
        span.set_attribute("gen_ai.usage.input_tokens",
                           response.usage.prompt_tokens)
        span.set_attribute("gen_ai.usage.output_tokens",
                           response.usage.completion_tokens)
        span.set_attribute("gen_ai.response.finish_reasons",
                           [response.choices[0].finish_reason])

        # 요청 상세 정보는 별도 이벤트로 기록 (토큰 카운트와 분리)
        # 실제 운영에서는 gen_ai.client.inference.operation.details 이벤트 사용
        span.add_event("gen_ai.client.inference.operation.details", {
            "gen_ai.request.model": model,
            "gen_ai.usage.input_tokens": response.usage.prompt_tokens,
            "content": prompt,  # 입력 텍스트
        })

        return response.choices[0].message.content
```

이 코드 한 조각으로 비용 귀속 대시보드, 모델별 지연 비교, TTFT 추적이 모두 가능해진다. 속성 키 3개(`gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.*`)만 추가하면 된다.

## 실전: 멀티 에이전트 트리를 단일 트레이스로 묶기

에이전트가 다른 에이전트를 호출하는 시스템에서 가장 중요한 것은 **트레이스 컨텍스트 전파(trace context propagation)**다. 각 에이전트가 독립적인 트레이스를 생성하면, 전체 워크플로를 하나의 트리로 볼 수 없다.

OpenTelemetry의 해결책은 간단하다. 현재 span 컨텍스트를 자식 에이전트 호출에 전파하면, 자식이 생성하는 span이 자동으로 부모의 자식이 된다. 같은 프로세스 내에서는 `start_as_current_span`이 이를 자동 처리한다:

```python
def orchestrator_agent(user_query: str) -> str:
    with tracer.start_as_current_span("invoke_agent") as span:
        span.set_attribute("gen_ai.operation.name", "invoke_agent")
        span.set_attribute("gen_ai.agent.name", "Orchestrator")

        # 1단계: 계획 수립 (plan span)
        plan = plan_subtasks(user_query)

        # 2단계: 각 서브태스크를 전문 에이전트에 위임
        results = []
        for subtask in plan:
            # context가 자동으로 전파됨 — 같은 트레이스 트리 안에 포함
            result = specialist_agent(subtask)
            results.append(result)

        return synthesize(results)


def specialist_agent(subtask: str) -> str:
    with tracer.start_as_current_span("invoke_agent") as span:
        span.set_attribute("gen_ai.operation.name", "invoke_agent")
        span.set_attribute("gen_ai.agent.name", "Specialist")
        span.set_attribute("gen_ai.agent.id", "spec-001")

        # 이 LLM 호출은 Specialist span의 자식이 됨
        return call_llm(subtask, model="gpt-4o-mini")
```

이 코드가 실행되면 다음과 같은 트레이스 트리가 생성된다:

```
invoke_agent (Orchestrator)
├── plan
├── invoke_agent (Specialist, subtask 1)
│   └── gen_ai.request (gpt-4o-mini)
├── invoke_agent (Specialist, subtask 2)
│   └── gen_ai.request (gpt-4o-mini)
└── invoke_agent (Specialist, subtask 3)
    └── gen_ai.request (gpt-4o-mini)
```

프로세스 경계를 넘는 경우(예: 에이전트가 별도 마이크로서비스)에는 W3C Trace Context 헤더를 HTTP 요청에 주입해야 한다. OpenTelemetry의 `Inject`/`Extract` API가 이를 처리하며, 대부분의 자동 계측 라이브러리(opentelemetry-instrumentation-requests 등)가 기본 지원한다.

## 어디서부터 시작해야 하는가

전체 시스템을 한 번에 계측하려 하지 말 것. 우선순위는 명확하다.

**1순위 — LLM 클라이언트 span 3개 속성 추가.** `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.*`만 넣어도 즉시 비용 대시보드가 작동한다. 코드 변경이 가장 적고 가시성 향상이 가장 크다.

**2순위 — TTFT 추적.** 모델과 환경별로 `time_to_first_chunk`를 기록하라. TTFT는 부하가 걸릴 때 출력 토큰 수만으로는 드러나지 않는 방식으로 저하된다. LLM 서비스가 요청을 큐에 쌓기 시작하면 가장 먼저 반응하는 신호다.

**3순위 — 에이전트 서브태스크로의 컨텍스트 전파.** 멀티 스텝 워크플로가 분리된 span이 아니라 단일 트레이스 트리를 생성하도록 하라. 에이전트 프레임워크가 자동으로 처리하지 않는다면, trace context를 명시적으로 전달해야 한다.

## 마이그레이션 시 주의할 점

**속성 키 이름 변경 주의.** GenAI Semantic Conventions는 아직 Development 상태다. `gen_ai.provider.name` 등의 키 이름은 향후 안정화 과정에서 변경될 수 있다. 백엔드 쿼리나 대시보드를 하드코딩할 때는 이 점을 염두에 두고, 버전업 시 마이그레이션 비용을 예산에 포함시켜라.

**토큰 카운트의 정확성.** 스트리밍 응답에서는 일부 제공자가 사용량 정보를 마지막 청크에만 포함한다. 계측 코드가 스트리밍 응답의 끝까지 기다리지 않으면 토큰 수가 누락된다. 스펙은 "사용량을 효율적으로 얻을 수 없다면 오프라인 토큰 카운팅을 허용한다"고 명시하지만, 그렇지 않은 경우 사용량 메트릭을 보고하지 말 것을 요구한다.

**벤더 지원은 있지만 깊이가 다르다.** Datadog, Honeycomb, New Relic 등 주요 백엔드가 GenAI 속성을 지원하지만, TTFT를 일급 컬럼으로 보여주는지, 에이전트 span 트리를 시각화하는지는 백엔드마다 다르다. 계측 코드는 백엔드 독립적으로 작성되지만, 시각화 경험은 백엔드 선택에 따라 달라진다.

## 핵심 요약

- OpenTelemetry는 2026년 5월 CNCF 졸업 프로젝트가 되었고, GenAI Semantic Conventions는 LLM/에이전트 워크로드를 위한 표준 속성 스키마를 제공한다.
- `gen_ai.provider.name`, `gen_ai.request.model`, `gen_ai.usage.*` 세 속성만 추가해도 비용 귀속과 모델 비교가 즉시 가능하다.
- 멀티 에이전트 시스템은 trace context 전파를 통해 단일 트리로 추적할 수 있다. 이것이 로그 기반 디버깅을 대체하는 핵심 이점이다.
- 클라이언트 메트릭(`time_to_first_chunk`)과 서버 메트릭(`time_to_first_token`)을 구분하면 병목이 네트워크에 있는지 모델에 있는지 알 수 있다.
- 컨벤션이 아직 Development 상태이므로, 키 이름 변경 가능성을 염두에 두고 의존성을 설계하라.

지금 LLM 서비스를 프로덕션에서 운영하면서 span 계측 없이 로그만 보고 있다면, 성능 문제를 진단하거나 비용을 정확히 귀속하는 데 필요한 신호가 빠져 있는 것이다. LLM 클라이언트 호출에 속성 3개를 추가하는 것부터 시작하라 — 그것이 가장 적은 노력으로 가장 큰 가시성 향상을 가져오는 첫 단계다.
