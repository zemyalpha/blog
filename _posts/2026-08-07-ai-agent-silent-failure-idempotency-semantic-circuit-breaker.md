---
title: "HTTP 200이지만 에이전트는 망가졌다: 조용한 실패를 잡는 멱등성과 의미 기반 회로 차단기"
date: "2026-08-07"
keywords: ["AI 에이전트 신뢰성", "silent failure", "멱등성", "circuit breaker", "tool call reliability"]
lang: "ko"
description: "에이전트가 HTTP 200을 반환하면서도 부작용을 중복 실행하고 환각 응답을 정상으로 처리하는 silent failure 문제를 분석하고, 멱등성 키·의미 기반 회로 차단기·보상 트랜잭션 패턴으로 해결하는 실전 가이드"
---

# HTTP 200이지만 에이전트는 망가졌다: 조용한 실패를 잡는 멱등성과 의미 기반 회로 차단기

에이전트가 같은 결제 이메일을 세 번 보내고, 고객의 신용카드에 두 번 청구하고, 데이터베이스에 중복 행을 삽입한 뒤 — 로그에는 한 줄의 에러도 없이 `HTTP 200` 응답만 남기는 시나리오를 상상해 보자. 모니터링 대시보드는 초록색이고, 헬스체크는 통과했으며, 에이전트는 스스로 "작업 완료"라고 보고한다. 이것이 2026년 프로덕션 AI 에이전트를 죽이는 실패 패턴이다. 그리고 이 문제는 모델 성능과는 아무 상관이 없다.

## 데모는 완벽하고 프로덕션은 죽는다

에이전트가 통제된 환경에서 잘 동작할 때와 실제 환경에서 동작할 때의 차이는 모델이 아니라 인프라에서 발생한다. 외부 API의 레이트 리밋, 멀티턴 대화에서의 컨텍스트 누적, 확률적 모델의 비결정적 출력, 그리고 사용자가 의도치 않게 만들어내는 모순적 입력 — 이 모든 것이 데모에서는 재현되지 않는다.

업계 벤치마크에서도 이 격차는 확인된다. APEX-Agents 벤치마크는 최고 성능 모델조차 실제 환경 작업을 첫 시도에 완료하는 비율이 25% 미만이라고 보고한다. 8회 재시도 후에도 성공률은 약 40% 수준에 그친다. 이는 엣지 케이스가 아니라 LLM의 작동 방식과 일반적인 에이전트 아키텍처에 내재된 구조적 실패 패턴이다.

더 중요한 것은, 이 실패의 상당수가 "실패"로 감지되지 않는다는 점이다. 전통적인 분산 시스템에서는 서비스가 동작하거나 동작하지 않는다는 이진적 신호로 충분했다. 하지만 AI 에이전트는 HTTP 200과 함께 의미적으로 잘못된 결과를 반환할 수 있다. 인용을 조작한 환각 답변은 정상적인 HTTP 요청으로 보인다. 구문적으로 유효하지만 논리적으로 잘못된 코드를 생성하는 도구는 200을 반환한다. 모든 API를 호출하고 도메인 전문가가 기각할 요약을 제공하는 연구 에이전트는 표준 모니터링에서 아무런 에러 신호를 보여주지 않는다.

## 3계층 신뢰성 스택: 재시도, 폴백, 회로 차단기

프로덕션 에이전트 신뢰성은 세 가지 상호 보완적 메커니즘으로 구성된다. 이를 서로 교체 가능한 것으로 취급하는 팀은 결국 빈틈을 만든다.

**재시도 로직**은 일시적 실패를 처리한다 — 2초면 지워지는 429 레이트 리밋, 짧은 서버 재시작 후 복구되는 503. 잘 구성된 재시도는 이를 사람의 개입 없이 투명하게 해결한다. 핵심은 어떤 에러를 재시도해야 하고 어떤 에러를 재시도하면 안 되는지 분류하는 것이다:

- **재시도할 것**: HTTP 429(레이트 리밋), 500/502/503/504(일시적 서버 에러), 네트워크 타임아웃. 이 에러들은 확률적이며 적절한 지연 후 재시도하면 성공 확률이 높다.
- **절대 재시도하지 말 것**: HTTP 401/403(인증 실패), 400(잘못된 요청), 컨텍스트 윈도우 오버플로우. 이는 결정적 실패다. 401은 매 재시도마다 401일 것이다. 컨텍스트 오버플로우를 재시도하면 토큰만 낭비하고 반드시 마주해야 할 에러를 지연시킬 뿐이다.

재시도의 역학도 분류만큼 중요하다. 지터(jitter) 없는 단순 선형 재시도는 "우르르 쳐들어가는(thundering herd)" 문제를 유발한다 — 부분적 과부하에서 막 복구된 서비스가 동기화된 재시도 웨이브에 의해 즉시 다시 공격받는다. 권장 설정은 다음과 같다:

```python
import asyncio
import random

async def retry_with_backoff(func, *args, max_retries=5, base_delay=1.0, **kwargs):
    """429/5xx에 대한 지수 백오프 + 지터 재시도"""
    for attempt in range(max_retries):
        try:
            return await func(*args, **kwargs)
        except (RateLimitError, ServerError) as e:
            if attempt == max_retries - 1:
                raise
            # 지수 백오프 + 10~25% 지터로 재시도 웨이브 분산
            delay = base_delay * (2 ** attempt)
            jitter = delay * random.uniform(0.1, 0.25)
            await asyncio.sleep(delay + jitter)
        except (AuthError, BadRequestError):
            raise  # 결정적 실패 — 재시도 무의미
```

**폴백 전략**은 지속적인 단일 서비스 실패를 처리한다. 주 LLM 제공자가 20분간 성능이 저하되면, 자동으로 보조 모델로 라우팅한다. 이는 완전한 작업 실패 대신 기능 저하를 감수하는 명시적 트레이드오프다.

**회로 차단기**는 시스템적 저하를 처리한다. 하위 API가 40%의 호출에서 실패할 때, 계속 두드리는 것은 상황을 악화시킬 뿐이다. 회로가 열리고(open) 서비스가 복구될 때까지 요청을 중단한다.

## 멱등성: 가장 비싼 '까먹는 것'

재시도 로직이 가장 흔히 구현되는 신뢰성 패턴이라면, 멱등성(idempotency)은 가장 흔히 건너뛰는 — 그리고 누락 비용이 가장 비싼 — 패턴이다. 멱등성 가드가 없으면, AI 에이전트는 부작용에 대해 "최소 한 번(at-least-once)" 실행을 보장한다. 읽기 작업에는 허용 가능하다. 이메일, 결제, 데이터베이스 쓰기에는 재앙이다.

멱등성 구현 패턴은 세 가지 요소로 구성된다:

**1. 멱등성 키**: 에이전트가 각 논리적 작업에 대해 안정적인 식별자를 생성하고 이를 헤더로 전달한다. 수신 서비스는 이 키로 결과를 저장하고, 동일한 키의 후속 요청은 재실행 없이 저장된 결과를 반환한다.

```python
import hashlib
import uuid

def make_idempotency_key(business_id: str, doc_id: str, task_type: str) -> str:
    """
    LLM이 키를 생성하게 하지 마라.
    비즈니스 식별자 + 콘텐츠 해시로 결정적 키 생성.
    """
    raw = f"{business_id}:{doc_id}:{task_type}"
    content_hash = hashlib.sha256(raw.encode()).hexdigest()[:16]
    return f"ic-{content_hash}"

# 결제 API 호출 시
response = client.charge(
    amount=4900,
    customer_id="cus_abc123",
    # 같은 논리적 작업은 같은 키 → 서버가 중복 결제 방지
    idempotency_key=make_idempotency_key("cus_abc123", "invoice_2026_08", "charge"),
)
```

가장 중요한 규칙: **절대 LLM이 멱등성 키를 동적으로 생성하게 하지 마라.** 키 생성은 결정적이어야 하며, 비즈니스 식별자(고객 ID + 문서 ID + 작업 유형)와 콘텐츠 해시를 조합해 도출해야 한다. LLM이 매 재시도마다 새 UUID를 생성하면 전체 메커니즘이 무력화된다.

**2. 목표 상태 쓰기(desired-state writes)**: 에이전트 도구 호출이 이벤트를 기록하는 대신 최종 목표 상태를 계산하고 쓰도록 설계하라. 대상 상태가 이미 일치하면 쓰기는 no-op가 된다. 이는 버전 관리와 설정 관리 시스템이 멱등성을 달성하는 방식이며, 에이전트 도구 호출도 동일한 패턴을 사용할 수 있다.

```python
async def set_order_status(order_id: str, target_status: str):
    """상태를 '설정'하지, 상태 변경을 '추가'하지 마라."""
    current = await db.get_order(order_id)
    if current.status == target_status:
        return {"ok": True, "action": "noop"}  # 이미 목표 상태
    await db.update_order(order_id, status=target_status)
    return {"ok": True, "action": "updated"}
```

**3. 메시지 큐 중복 제거**: SQS, Pub/Sub, Kafka(정확히 한 번 시맨틱스 포함) 같은 대부분의 프로덕션 큐 시스템은 중복 제거 ID를 지원한다. 큐 레이어의 "최소 한 번 전달"과 소비자의 "최대 한 번 실행"을 결합하라.

## 의미 기반 회로 차단기: 200이지만 틀린 경우

전통적 회로 차단기는 이진적 실패를 가정한다: 서비스가 동작하거나 동작하지 않는다. AI 에이전트는 전통적 회로 차단기가 완전히 놓치는 실패 모드에 직면한다 — **HTTP 200을 반환하는 의미적 실패**다.

환각된 인용이 포함된 답변은 성공적인 HTTP 요청이다. 도메인 전문가가 기각할 요약을 제공하는 연구 에이전트는 표준 모니터링에서 아무 에러 신호도 보여주지 않는다. 회로는 닫혀 있고, 헬스체크는 통과하며, 쓰레기 데이터가 전속력으로 파이프라인을 통과한다.

이를 위해 프로덕션 회로 차단기에는 두 가지 적응이 필요하다:

**DEGRADED(저하) 상태**: 이진적 open/closed 대신 부분적 기능 저하를 위한 중간 상태를 추가한다. 의미적 실패율이 임계값을 초과하지만 전체 중단이 트리거되지 않은 경우, 전속력 가동이나 완전 중단 대신 보수적 폴백(더 단순한 모델, 좁은 범위, 사람 검토 큐)으로 라우팅한다.

**점진적 재활성화(graduated re-enablement)**: 타임아웃 기간 후 단일 프로브 요청 대신, 회로를 완전히 닫기 전에 윈도우에 걸쳐 여러 프로브 샘플을 사용한다. 의미적 저하 기간 후 운 좋은 단일 요청이 성공한 것은 회복의 충분한 증거가 아니다.

```python
from dataclasses import dataclass, field
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED = "closed"      # 정상 — 모든 요청 통과
    DEGRADED = "degraded"  # 부분 — 보수적 폴백 라우팅
    OPEN = "open"          # 중단 — 요청 즉시 실패

@dataclass
class SemanticCircuitBreaker:
    failure_threshold_degraded: float = 0.15  # 15% 의미 실패 → DEGRADED
    failure_threshold_open: float = 0.40      # 40% → OPEN
    recovery_window: float = 60.0             # 60초 복구 관찰
    probes_for_recovery: int = 5              # 5회 프로브 성공 필요
    state: CircuitState = CircuitState.CLOSED
    recent_results: list = field(default_factory=list)  # (timestamp, is_semantic_failure)

    def record(self, is_semantic_failure: bool):
        now = time.time()
        self.recent_results = [
            (t, f) for t, f in self.recent_results
            if now - t < 300  # 최근 5분 윈도우
        ]
        self.recent_results.append((now, is_semantic_failure))
        self._evaluate()

    def _evaluate(self):
        if len(self.recent_results) < 10:
            return
        failure_rate = sum(f for _, f in self.recent_results) / len(self.recent_results)
        if failure_rate >= self.failure_threshold_open:
            self.state = CircuitState.OPEN
        elif failure_rate >= self.failure_threshold_degraded:
            self.state = CircuitState.DEGRADED
        else:
            self.state = CircuitState.CLOSED
```

의미적 실패를 감지하려면 "이 응답이 의미적으로 옳은가?"를 평가하는 메커니즘이 필요하다. 이를 위해 검증 모델(주 모델보다 작고 빠른 모델)로 응답을 사후 검증하거나, 구조화된 출력 스키마로 응답 형식을 강제한 뒤 비즈니스 규칙 검사기를 통과시키는 방식이 실용적이다.

## Saga 패턴: 장기 실행 워크플로우의 보상 트랜잭션

외부 시스템을 건드리는 복잡한 다단계 워크플로우에서는 saga 패턴이 분산 락 없이 트랜잭션과 유사한 보장을 제공한다. saga의 각 단계에는 보상 동작(compensating action)이 있다 — 5단계가 성공하고 6단계가 실패하면, 시스템이 자동으로 1~5단계를 되돌리는 보상 동작을 실행한다. 결제 환불, 이메일 회수, 데이터베이스 롤백.

```python
@dataclass
class SagaStep:
    name: str
    execute: callable        # 정방향 실행
    compensate: callable     # 실패 시 되돌리기

async def run_saga(steps: list[SagaStep]):
    """각 단계의 보상 동작으로 장기 워크플로우 안전 실행"""
    completed = []
    try:
        for step in steps:
            await step.execute()
            completed.append(step)
        return {"status": "complete"}
    except Exception as e:
        # 역순으로 보상 실행 — 각 보상 자체도 멱등해야 함
        for step in reversed(completed):
            try:
                await step.compensate()
            except Exception as comp_err:
                # 보상 실패는 알람 대상 — 수동 개입 필요
                await alert_ops(f"보상 실패: {step.name}: {comp_err}")
        return {"status": "rolled_back", "error": str(e)}
```

보상 동작 자체도 멱등해야 하며, 자체 재시도 메커니즘 없이 실패해서는 안 된다. 보상이 실패하면 즉각 운영 알림의 대상이 된다 — 자동으로 해결할 수 없는 상태다.

## 핵심 요약

- **HTTP 200은 성공이 아니다** — 에이전트의 가장 위험한 실패는 에러 코드가 아닌 의미적으로 잘못된 결과다. 의미 기반 회로 차단기와 응답 사후 검증으로 감지하라.
- **재시도는 부작용을 중복시킨다** — 읽기가 아닌 모든 도구 호출에 멱등성 키를 적용하라. 단, 키 생성은 절대 LLM에게 맡기지 마라.
- **결정적 실패와 확률적 실패를 구분하라** — 401/403/400은 재시도해도 같은 결과다. 429/5xx만 지수 백오프 + 지터로 재시도하라.
- **3계층은 상호 보완적이다** — 재시도, 폴백, 회로 차단기를 모두 구축하라. 하나만 있는 것은 빈틈을 만든다.
- **장기 워크플로우는 saga로** — 각 단계에 보상 동작을 정의하고, 보상 자체도 멱등하게 만들어라.

에이전트를 처음 프로덕션에 올리는 팀이 가장 먼저 해볼 수 있는 것은 간단하다: 현재 에이전트의 도구 호출 중 부작용이 있는 것(write, charge, send)을 나열하고, 각각에 멱등성 키가 적용되어 있는지 확인하라. 그다음 의미적 실패율을 측정하는 메트릭을 하나 추가하라. 이 두 단계만으로 데모와 프로덕션 사이의 가장 깊은 격차를 좁힐 수 있다.
