---
title: "AI 에이전트는 크래시 후 어디서부터 다시 실행되나: Temporal 리플레이와 LangGraph 체크포인팅으로 보는 내구성 실행 설계"
date: "2026-08-16"
keywords: ["내구성 실행", "durable execution", "Temporal", "Restate", "DBOS", "LangGraph", "AI 에이전트", "이벤트 소싱"]
lang: "ko"
description: "LLM 에이전트가 프로세스 크래시 후 처음부터 다시 시작하는 문제를 내구성 실행으로 해결한다. 이벤트 히스토리 리플레이(Temporal·Restate)와 상태 체크포인팅(LangGraph·DBOS)의 동작 원리와 코드, 선택 기준을 정리했다."
---

# AI 에이전트는 크래시 후 어디서부터 다시 실행되나: Temporal 리플레이와 LangGraph 체크포인팅으로 보는 내구성 실행 설계

고객 이메일에 답장하는 에이전트를 생각해보자. 작업은 대략 이렇다: 이메일 수신 → 분류 → 주문 조회 → 환불 정책 확인 → 답장 초안 → 승인 대기 → 발송. 승인 대기는 사람이 개입하므로 몇 시간이 걸릴 수 있다. 그런데 승인을 기다리던 중 밤새 서버가 재시작했다. 다음 날 아침 에이전트를 다시 띄우면 이 실행은 어디서 이어질까?

대부분의 LLM 에이전트 프레임워크에서 답은 냉정하다. **처음부터 다시 시작한다.** 아니면 더 나쁘다 — 이미 환불 API를 호출한 뒤 죽었다면, 재시작 후 같은 호출을 반복해 이중 환불이 일어난다. 상태가 프로세스 메모리에만 있기 때문이다.

Temporal은 자사 기술 자료에서 이를 "에이전트는 위장한 분산 시스템(agents are distributed systems in disguise)"이라고 표현한다. LLM 호출을 하고, 외부 API를 기다리고, 사람의 입력을 받고, 예측 불가능하게 분기하는 — 살아있는 상태를 가진 시스템이라는 뜻이다. 분산 시스템이면 크래시 복구를 설계해야 한다. 이 글에서는 그 해법인 **내구성 실행(durable execution)**의 두 가지 구현 방식을 실제 코드와 함께 비교하고, 프로젝트 상황에 따른 선택 기준을 정리한다.

## 문제의 본질: 왜 에이전트 루프는 크래시에 취약한가

일반적인 에이전트 루프를 의사 코드로 쓰면 다음과 같다.

```python
state = {"messages": [], "step": 0}
while not done(state):
    action = llm.decide(state)        # 30초 걸리는 호출
    result = tools.execute(action)    # 외부 부작용 발생
    state["messages"].append(result)
    state["step"] += 1
```

이 코드가 프로덕션에서 부서지는 지점은 세 곳이다.

**첫째, 상태가 메모리에만 있다.** `state` 딕셔너리는 프로세스가 죽으면 사라진다. 며칠짜리 휴먼인더루프 작업은 커녕, K8s 디플로이먼트 롤아웃 한 번으로 진행 중이던 모든 실행이 증발한다.

**둘째, 부분 완료(partial completion)가 남는다.** 10단계 중 7단계에서 외부 API를 호출한 뒤 죽으면, 그 7번 호출의 부작용은 이미 세상에 나가 있다. 재시작해서 1단계부터 다시 실행하면 1~7단계의 API 호출이 전부 반복된다. 결제, 이메일 발송 같은 작업이라면 사고다.

**셋째, 실행이 조용히 멈춘다(silent stall).** 프로세스는 안 죽었는데 LLM API 응답이 영영 오지 않거나, 큐에서 메시지를 꺼낸 뒤 처리 전에 워커가 죽는 경우다. 장애 알림조차 없이 작업이 증발한다.

이 세 가지를 한 번에 해결하는 것이 내구성 실행이다. 핵심 약속은 하나다: **코드는 완료될 때까지 실행되며, 실패하면 완료된 단계를 반복하지 않고 이어서 재개된다.**

## 두 가지 구현 방식: 이벤트 히스토리 리플레이 vs 상태 체크포인팅

내구성 실행 엔진은 실패 복구 방식에 따라 크게 둘로 나뉜다.

### 방식 A: 이벤트 히스토리 리플레이 — Temporal, Restate

Temporal의 공식 문서는 동작 원리를 명확히 설명한다. 워크플로우 실행 중 발생하는 모든 일은 **이벤트 히스토리(Event History)**라는 정렬된 로그에 기록된다. "타이머 5분 시작", "액티비티 X 스케줄", "액티비티 X 완료(결과 Y)", "시그널 Z 수신" 식이다.

크래시 후 재개할 때 Temporal은 메모리 스냅샷을 복원하지 않는다. 대신 **워크플로우 코드를 처음부터 다시 실행하되, 외부 효과가 필요한 지점에서 기록된 이벤트의 결과를 재사용한다.** 액티비티(외부 호출)는 재실행되지 않고 히스토리에 기록된 결과가 돌아온다. 코드가 히스토리를 따라 정확히 같은 분기를 타고 가서 중단 지점에 도달하면, 거기서부터 진짜 실행이 이어진다.

이 방식의 조건은 **결정성(determinism)**이다. 같은 이벤트 히스토리가 주어지면 코드는 항상 같은 결정을 내려야 한다. 그래서 Temporal 워크플로우 함수 안에서 직접 `Date.now()`나 랜덤, 네트워크 호출을 하면 안 되고, 리플레이 안전 API나 액티비티를 써야 한다. 공식 문서는 LLM 호출을 액티비티가 담당하는 대표 사례로 명시한다.

Restate도 같은 계열이지만 접근이 다르다. Rust로 작성된 오픈소스 런타임으로, GitHub 리포에서 확인되는 코어 프리미티브는 네 가지다: 실행 완료를 보장하는 Reliable Execution, 정확히 한 번(exactly-once) 전달을 지원하는 Reliable Communication, 장애와 재시작을 넘어 살아남는 Durable Promises와 타이머, 그리고 엔티티별 K/V 상태를 실행 진행과 함께 영속화하는 Consistent State. Temporal이 이벤트 히스토리 중심의 풀플랫폼이라면, Restate는 HTTP 서비스와 함수형 배포(FaaS)에 붙이기 쉬운 가벼운 런타임 지향이다.

### 방식 B: 상태 체크포인팅 — LangGraph, DBOS

이 방식은 코드를 재실행하지 않는다. 대신 **상태 스냅샷을 저장소에 주기적으로 기록**하고, 재시작 시 마지막 체크포인트에서 그래프를 다시 구동한다.

LangGraph의 공식 문서는 여기서 두 개념을 분리한다. **체크포인터(checkpointer)**는 하나의 스레드(대화·실행 단위)의 그래프 상태를 스냅샷으로 저장하는 단기 메모리고, **스토어(store)**는 스레드를 넘는 장기 메모리다. 프로덕션에서는 `InMemorySaver` 대신 `PostgresSaver`나 `SqliteSaver`를 써야 하며, 문서가 명시한 실전 함정도 있다 — 체크포인트가 무한히 쌓여 저장 비용과 지연이 커지므로 보존 정책(retention)을 걸어야 하고, PostgresSaver의 `thread_id`는 255자를 넘으면 안 된다.

DBOS Transact는 이 계열에서 가장 극단적으로 간단한 접근이다. 오케스트레이터 서버가 아예 없고, **오픈소스 라이브러리 하나와 Postgres 하나**로 내구성을 만든다. 공식 리포의 설명 그대로 "데코레이터로 함수에 주석만 달면" 워크플로우가 되고, 실패 시 마지막 완료 단계부터 재개된다.

## 실전 코드: 같은 에이전트를 세 엔진으로

주문 처리 에이전트의 핵심 부분을 각 엔진의 공식 API로 옮겨보자.

### Temporal — 워크플로우와 액티비티의 분리

```python
from temporalio import activity, workflow
from dataclasses import dataclass
from datetime import timedelta

@dataclass
class RefundOrderInput:
    order_id: str
    reason: str

@activity.defn
async def lookup_order(order_id: str) -> dict:
    # 외부 API 호출: 리플레이 시 재실행되지 않고 히스토리 결과 재사용
    return await orders_api.get(order_id)

@activity.defn
async def call_llm_for_draft(order: dict, reason: str) -> str:
    # LLM 호출도 반드시 액티비티 안에서
    return await llm.generate(f"환불 사유 '{reason}'에 대한 답장 초안: {order}")

@workflow.defn
class RefundAgent:
    @workflow.run
    async def run(self, order_id: str) -> str:
        order = await workflow.execute_activity(
            lookup_order, order_id,
            start_to_close_timeout=timedelta(seconds=30),
        )
        if order["amount"] > 100_000:
            # 휴먼인더루프: 시그널로 승인 대기 (타이머·시그널도 이벤트로 기록됨)
            approval = await workflow.wait_condition(
                lambda: self.approved, timeout=timedelta(hours=24)
            )
        draft = await workflow.execute_activity(
            call_llm_for_draft, args=(order, "refund"),
            start_to_close_timeout=timedelta(seconds=60),
        )
        return draft
```

핵심은 `RefundAgent.run` 안에 부작용 코드가 없다는 것이다. 크래시가 나면 `run`은 처음부터 재실행되지만 `execute_activity`의 결과는 히스토리에서 돌아오므로 주문 조회도 LLM 호출도 다시 일어나지 않는다.

### DBOS — 데코레이터 두 개면 끝

```python
from dbos import DBOS

@DBOS.step()
def lookup_order(order_id: str) -> dict:
    return orders_api.get_sync(order_id)   # 이 함수는 재실행되지 않는다

@DBOS.step()
def call_llm_for_draft(order: dict) -> str:
    return llm.generate_sync(f"환불 답장 초안: {order}")

@DBOS.workflow()
def refund_agent(order_id: str) -> str:
    order = lookup_order(order_id)         # 각 단계가 Postgres에 체크포인트됨
    draft = call_llm_for_draft(order)
    return draft
```

`@DBOS.step()`이 붙은 함수는 호출 시 결과가 Postgres에 기록되고, 워크플로우 재개 시 완료된 스텝은 기록된 결과를 반환한다. 별도 서버 없이 앱 프로세스 안에서 동작한다.

### LangGraph — 체크포인터 지정 후 thread_id로 재개

```python
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.graph import StateGraph

checkpointer = PostgresSaver.from_conn_string("postgresql://localhost/agents")
checkpointer.setup()  # 테이블·인덱스 생성

graph = builder.compile(checkpointer=checkpointer)

# 최초 실행
result = graph.invoke(
    {"messages": [{"role": "user", "content": "주문 123 환불해줘"}]},
    config={"configurable": {"thread_id": "refund-123"}},  # 255자 이내
)

# 크래시 후 재개: 같은 thread_id를 주면 마지막 체크포인트에서 이어서 실행
resumed = graph.invoke(None, config={"configurable": {"thread_id": "refund-123"}})
```

같은 `thread_id`로 재진입하면 마지막 체크포인트부터 그래프가 이어서 돈다. 에이전트의 그래프 구조를 그대로 유지하면서 내구성만 붙이는 셈이라, 이미 LangGraph로 짜둔 시스템에 가장 낮은 비용으로 적용할 수 있다.

한 가지 더: 서두의 환불 승인처럼 사람의 입력을 기다리는 지점에는 `interrupt()` 함수를 쓴다. 노드 안에서 `interrupt("승인하시겠습니까?")`를 호출하면 그래프가 그 지점에서 정지하고 상태가 체크포인트되며, 승인이 오면 `Command(resume=...)`로 재개한다. 공식 문서의 표현을 빌리면 `thread_id`는 "영속 커서(persistent cursor)"다 — 같은 값을 재사용하면 같은 체크포인트에서 이어지고, 새 값을 쓰면 빈 상태의 새 스레드가 시작된다.

## 선택 기준과 주의사항

**1. 이미 LangGraph로 만든 에이전트인가?** 그러면 체크포인터를 `PostgresSaver`로 바꾸는 것만으로도 크래시 복구가 확보된다. 이벤트 소싱 리플레이로 갈아타는 것은 대부분 과도한 이전이다. 단, 체크포인트 정리 정책을 반드시 세울 것.

**2. 실행이 며칠·몇 주 단위로 길어지고 사람이 개입하나?** 장기 실행 + 휴먼인더루프 + 타임아웃 분기가 본격적이면 Temporal 계열이 강하다. 이벤트 히스토리가 곧 완전한 감사 로그가 되고, 시그널·타이머·사가(saga) 패턴이 플랫폼 차원에서 제공된다. Temporal은 OpenAI가 프로덕션에서 사용한다고 자사 페이지가 밝히고, 고객지원 에이전트 업체 Gorgias가 15,000개 브랜드 규모의 에이전트를 Temporal로 운영한다고 소개한다(모두 벤더 발표 기준). NVIDIA 역시 장기 실행 GPU 워크플로 오케스트레이션에 쓴다고 한다.

**3. 인프라를 더 늘릴 수 없는 팀인가?** DBOS는 Postgres 하나가 이미 있다면 추가 서버가 제로다. 소규모 팀이 결제 파이프라인 하나의 신뢰성을 확보하는 용도로 가장 빠른 경로다.

**4. 가장 흔한 실수 — 워크플로우 함수 안에 LLM 호출 직접 넣기.** Temporal 방식에서 LLM 호출을 액티비티 밖에 두면 리플레이마다 재호출돼 비용이 배로 늘고, 논텐스턴트한 응답 때문에 결정성이 깨져 워크플로우가 재수행 실패(non-determinism error)로 망가진다. LLM 호출·API 호출·파일 I/O는 무조건 액티비티(또는 스텝)로 분리하라.

**5. 결정성 제약이 감당하기 어렵다면 체크포인팅 계열로.** 리플레이 방식은 코드 작성 규율(시간·랜덤·I/O 금지)이 따른다. 팀이 이 규율을 지키기 어렵다면 상태 스냅샷 방식(LangGraph, DBOS도 내부적으로 진행 로그 기반이지만 코드에는 규율이 덜 보인다)이 실수 여지가 적다.

**6. 무엇을 고르든 멱등성 키는 별도로.** 내구성 실행은 "완료된 단계의 반복"을 막아줄 뿐, 액티비티 자체가 실행 도중에 죽은 경우(호출은 나갔는데 결과 기록 전)의 정확히 한 번 실행은 액티비티 구현이 보증해야 한다. 결제 같은 API에는 멱등성 키를 심어 두는 것이 여전히 필수다.

## 결론

- 에이전트의 상태가 프로세스 메모리에만 있으면 크래시·롤아웃·silent stall 모두가 데이터 유실 사고가 된다.
- 내구성 실행의 회복 방식은 둘 중 하나다: 이벤트 히스토리를 재생하는 **리플레이**(Temporal, Restate) 또는 마지막 상태에서 재기동하는 **체크포인팅**(LangGraph, DBOS).
- Temporal 워크플로우에서 LLM·API 호출은 반드시 액티비티로 분리해야 리플레이가 안전하다.
- 이미 LangGraph라면 `PostgresSaver` 체크포인터가 최소 비용 해법이고, 인프라 없이 시작하려면 DBOS + Postgres가 가장 가볍다.
- 어떤 엔진을 골라도 외부 부작용 API의 멱등성 키는 별도로 설계해야 완전하다.

당장 해볼 일: 진행 중인 에이전트 코드에서 `state` 변수가 어디에 저장되는지 추적해보라. 답이 "메모리"라면 그 에이전트는 아직 프로덕션용이 아니다.
