---
title: "MCP 2026-07-28 스펙이 세션을 없앤 방법: 스테이트리스 코어와 Tasks 확장 마이그레이션 가이드"
date: "2026-08-19"
keywords: ["MCP", "Model Context Protocol", "스테이트리스", "MCP 2026-07-28", "Tasks 확장", "MRTR", "AI 에이전트 프로토콜"]
lang: "ko"
description: "MCP 2026-07-28 최종 스펙의 핵심 변경점: initialize 핸드셰이크와 세션 제거, Mcp-Method 라우팅 헤더, 멀티 라운드트립 요청, Tasks 확장까지 마이그레이션 체크리스트로 정리한다."
---

# MCP 2026-07-28 스펙이 세션을 없앤 방법: 스테이트리스 코어와 Tasks 확장 마이그레이션 가이드

2026년 7월 28일, Model Context Protocol(MCP)의 최종 스펙 `2026-07-28`이 확정됐다. MCP 공식 블로그는 이를 "출시 이래 가장 큰 개정(the largest revision of the protocol since launch)"이라고 표현했다. 화려한 기능 추가가 아니라 **프로토콜 코어에서 세션을 아예 제거**하는 구조적 수술이 핵심이다. 5월 21일 릴리스 후보(RC) 공개를 거쳐 7월 28일 최종 확정됐으며, 브레이킹 체인지를 포함한다.

이 글은 공식 스펙 문서와 MCP 블로그, Tasks 확장 문서를 기반으로 무엇이 어떻게 바뀌었는지, 그리고 기존 MCP 서버·클라이언트를 어떻게 마이그레이션해야 하는지 정리한다.

## 1. 가장 큰 변화: 프로토콜 레이어의 세션 제거

이전 버전(2025-11-25)에서 Streamable HTTP로 도구를 호출하려면 먼저 `initialize` 핸드셰이크를 수행해야 했다. 서버는 `Mcp-Session-Id` 헤더로 세션 식별자를 내려주고, 이후 모든 요청이 이 식별자를 실어야 했다.

```http
POST /mcp HTTP/1.1
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": { },
    "clientInfo": { "name": "my-app", "version": "1.0" }
  }
}
```

이 구조의 문제는 운영 관점에서 드러난다. 세션이 특정 서버 인스턴스에 클라이언트를 고정(pinning)하기 때문에, 수평 확장하려면 스티키 세션(sticky session)이나 공유 세션 스토어가 필요했다. 게이트웨이는 본문을 열어봐야 어떤 작업인지 알 수 있었으므로 딥 패킷 검사나 본문 파싱 없이는 라우팅·레이트리밋이 어려웠다.

2026-07-28에서는 이 핸드셰이크 자체가 사라진다.

- `initialize` / `initialized` 핸드셰이크 제거 (SEP-2575)
- `Mcp-Session-Id` 헤더와 프로토콜 레벨 세션 제거 (SEP-2567)
- 프로토콜 버전, 클라이언트 정보, 클라이언트 역량(capabilities)은 매 요청의 `_meta`에 실려 전달
- 클라이언트가 미리 서버 역량을 조회하는 새 메서드 `server/discover` 추가

결과적으로 동일한 도구 호출이 하나의 자기완결형 요청이 된다.

```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: search
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search",
    "arguments": { "q": "otters" },
    "_meta": {
      "io.modelcontextprotocol/clientInfo": {
        "name": "my-app", "version": "1.0"
      }
    }
  }
}
```

공식 블로그가 요약한 실무적 효과는 명확하다. 스티키 세션, 공유 세션 스토어, 게이트웨이의 본문 검사가 필요했던 원격 MCP 서버가 이제 **평범한 라운드 로빈 로드밸런서 뒤에서** 동작한다.

## 2. "스테이트리스 프로토콜, 스테이트풀 애플리케이션"

세션이 사라졌다고 애플리케이션의 상태가 필요 없어지는 것은 아니다. 서버가 호출 간 상태를 유지해야 한다면, HTTP API가 항상 그래왔던 방식을 쓰면 된다. 도구가 명시적 핸들(예: `basket_id`, `browser_id`)을 발급하고, 모델이 이를 이후 호출의 일반 인자로 다시 전달하는 것이다.

공식 문서는 이 패턴을 세션 상태의 "그럴듯한 대체제" 이상으로 평가한다. 전송 메타데이터에 숨어 있던 상태가 모델 눈에 보이는 형태로 드러나면서, 모델이 여러 도구에 걸쳐 핸들을 조합하고 추론하고 넘겨줄 수 있게 된다는 것. 프로토콜이 상태를 관리해주지 않지만, 관리를 막지도 않는다 — 상태의 소유권이 프로토콜에서 애플리케이션으로 이동한 셈이다.

## 3. 서버→클라이언트 요청의 재구축: 멀티 라운드트립 요청(MRTR)

세션이 없어도 서버가 도구 호출 중간에 사용자 확인(elicitation)을 요청할 수 있어야 한다. 두 개의 SEP가 이 흐름을 재구축한다.

**첫째 (SEP-2260)**, 서버가 시작하는 요청은 이제 서버가 클라이언트 요청을 *처리 중일 때만* 허용된다. 이전엔 권장사항이었지만 이제 의무 사양이다. 사용자는 갑작스럽게 프롬프트를 받지 않으며, 모든 확인 요청은 사용자(또는 에이전트)가 시작한 무언가로 추적된다.

**둘째 (SEP-2322)**, 프롬프트 전달 방식이 바뀐다. SSE 스트림을 열어두는 대신 서버는 `InputRequiredResult`를 반환한다.

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "confirm": {
      "type": "elicitation",
      "message": "Delete 3 files?",
      "schema": { "type": "boolean" }
    }
  },
  "requestState": "eyJzdG...iXX0="
}
```

클라이언트는 응답을 모아 원래 호출을 `inputResponses`와 되돌려받은 `requestState`와 함께 재요청한다. 재시도에 필요한 모든 정보가 페이로드 안에 있으므로 **어떤 서버 인스턴스든** 그 재시도를 이어받을 수 있다.

## 4. 운영 가능해진 트래픽: 라우팅 헤더와 캐싱

세 가지 소소하지만 실무적 영향이 큰 변화다.

**Mcp-Method / Mcp-Name 헤더 (SEP-2243)** — Streamable HTTP 전송은 이제 모든 요청에 `Mcp-Method` 헤더를 요구하고, `tools/call`, `resources/read`, `prompts/get`에는 `Mcp-Name` 헤더까지 요구한다. 로드밸런서·게이트웨이·레이트리미터가 본문을 파싱하지 않고 작업 단위로 라우팅할 수 있다. 헤더와 본문이 불일치하는 요청은 서버가 거부한다.

**ttlMs / cacheScope (SEP-2549)** — 목록 조회와 리소스 읽기 결과에 HTTP `Cache-Control`을 모델링한 `ttlMs`와 `cacheScope`가 붙는다. 클라이언트는 `tools/list` 응답이 얼마나 신선한지, 사용자 간 공유가 안전한지 정확히 알 수 있다. 목록 변경을 알려주는 장기 SSE 스트림이 더 이상 유일한 수단이 아니다.

**W3C Trace Context (SEP-414)** — `_meta`를 통한 트레이스 컨텍스트 전파가 공식 문서화됐다. 분산 트레이싱 도구와의 연동이 표준 경로를 갖게 됐다. 한 가지 흥미로운 점은 프로토콜 레벨 Logging 기능의 권장 대체재가 OpenTelemetry라는 것 — 관측성이 프로토콜 밖으로 나가는 방향성이 스펙 전반에 일관된다.

## 5. 인가 강화: mix-up 공격 대응과 DCR 정합성

인가는 OAuth 2.0 / OpenID Connect 배포 관행에 더 가깝게 정렬됐다.

- 클라이언트는 인가 응답의 `iss` 파라미터를 RFC 9207에 따라 검증해야 한다. MCP의 전형적인 패턴인 "하나의 클라이언트가 여러 서버와 통신"에서 발생하기 쉬운 **mix-up 공격**에 대한 대응이다.
- Dynamic Client Registration에서 클라이언트가 `application_type`을 선언한다. 데스크톱·CLI 클라이언트가 "web"으로 기본 취급되어 localhost 리다이렉트 URI가 거부되던 문제가 해결된다.
- 자격 증명이 발급한 인가 서버에 묶이고, 리프레시 토큰 요청 방법과 step-up 인증 시 스코프 누적 동작이 명확해졌다.

## 6. 폐기 예정: Roots, Sampling, Logging

세 가지 핵심 기능이 공식적으로 디프리케이트됐다.

| 폐기 대상 | 기능 | 권장 대체재 |
|---|---|---|
| Roots | 클라이언트가 서버에 파일시스템 경계를 알림 | 도구 파라미터 또는 리소스 URI |
| Sampling | 서버가 클라이언트의 모델로 텍스트 생성 요청 | LLM 프로바이더 API 직접 연동 |
| Logging | 서버→클라이언트 프로토콜 레벨 로그 알림 | OpenTelemetry 기반 구조화 관측 |

디프리케이션은 제거가 아니다. 해당 메서드·타입·역량 플래그는 이번 릴리스와 **이후 1년 내에 출간되는 모든 스펙 버전에서 계속 동작한다.** 다만 제거 시계가 돌아가기 시작했으므로 새로 짓는 서버라면 대체재로 설계해야 한다.

함께 바뀐 세부 사항 두 가지도 놓치기 쉽다. 도구 입출력 스키마가 완전한 JSON Schema 2020-12를 채택해 `oneOf`/`anyOf`/`allOf` 조합이 가능해졌고, 출력 스키마는 제한이 없어져 `structuredContent`가 객체가 아닌 임의 JSON 값이 될 수 있다. 그리고 존재하지 않는 리소스의 에러 코드가 MCP 커스텀 코드 `-32002`에서 JSON-RPC 표준 `-32602`(Invalid Params)로 변경됐다. 이는 에러 코드 전반 재번호화의 일부로 `-32001`·`-32003`·`-32004`도 각각 `-32020`·`-32021`·`-32022`로 바뀌었다. **리터럴 코드 값에 매칭하는 클라이언트 코드가 있다면 전부 점검해야 한다.**

## 7. Tasks 확장: 오래 걸리는 작업의 표준 패턴

실험적 기능이었던 Tasks가 정식 확장(extension)으로 졸업했다. 모든 도구 호출이 즉시 반환되지는 않는다. CI 파이프라인, 배치 처리, 사람의 승인은 수 초에서 수 분, 그 이상 걸린다. Tasks는 차단하는 대신 **내구성 있는 핸들**을 반환한다.

동작 방식은 다음과 같다.

1. 클라이언트는 역량에 `io.modelcontextprotocol/tasks`를 포함하고, 서버는 `server/discover`로 같은 확장을 광고한다.
2. 서버는 오래 걸릴 요청에 `resultType: "task"`인 `CreateTaskResult`로 `taskId`, 초기 상태, TTL, 권장 폴링 간격을 반환한다. 태스크는 응답 전송 **이전에** 내구성 있게 생성된다.
3. 클라이언트는 `tasks/get`으로 폴링한다.
4. 도중 입력이 필요하면 상태가 `input_required`로 바뀌고 `inputRequests` 맵이 따라온다. 클라이언트는 `tasks/update`로 응답한다.
5. `completed`면 `result` 필드에 동기 호출이 반환했을 결과가 담긴다. `failed`면 JSON-RPC 에러가 담긴다.

상태 머신은 `working` → (`input_required` 반복 가능) → `completed` / `failed` / `cancelled` 다. `tasks/cancel`은 협조적(cooperative) 취소다 — 서버는 취소 의도를 확인하지만 작업 중단 의무는 없다. 태스크 ID는 연결이 끊겨도 살아남으므로, 모바일 클라이언트나 불안정한 네트워크에서 재접속 후 같은 ID로 폴링을 이어갈 수 있다.

확장 프레임워크 자체도 정식 프로세스를 갖췄다. 확장은 역방향 DNS(reverse-DNS) ID로 식별되고, 확장 맵을 통해 협상되며, 자체 리포지토리와 위임된 메인테이너를 갖고 스펙과 독립적으로 버전을 매긴다. 이번 릴리스에는 MCP Apps(샌드박스 iframe에서 렌더링되는 서버 제공 인터랙티브 UI)와 Tasks 두 개의 공식 확장이 함께 배포됐다.

## 8. 마이그레이션 체크리스트

기존 구현을 점검한다면 이 순서로 확인하자.

**MCP 클라이언트를 만든다면:**
- `initialize` 핸드셰이크 로직과 `Mcp-Session-Id` 저장·재전송 코드 제거
- 매 요청 `_meta`에 클라이언트 정보·역량 포함
- `tools/list` 응답의 `ttlMs`/`cacheScope` 존중하도록 캐시 레이어 조정
- `InputRequiredResult` 처리 루틴 추가 (SSE 대기 제거)
- `-32002` 리터럴 매칭 `-32602`로 교체, `-32001`/`-32003`/`-32004` 매칭도 재번호화된 값으로 점검
- Tasks 확장 사용 시 `input_required` 상태의 `tasks/update` 흐름 구현

**MCP 서버를 만든다면:**
- 상태가 필요하면 명시적 핸들 발급 패턴으로 재설계
- 모든 요청에 `Mcp-Method`(및 해당 시 `Mcp-Name`) 헤더 검증 추가
- 인가 서버가 RFC 9207 `iss`를 지원하는지, 데스크톱/CLI 클라이언트의 `application_type` 선언이 가능한지 확인
- Roots/Sampling/Logging 의존 부분을 도구 파라미터·직접 LLM 연동·OpenTelemetry로 이전 계획 수립
- 오래 걸리는 도구에 Tasks 확장 도입 검토

## 결론

- MCP 2026-07-28은 기능 추가가 아니라 **수평 확장을 위한 구조 제거**가 핵심이다. 세션·핸드셰이크가 사라지고 모든 요청이 자기완결형이 됐다.
- 운영 관점의 수확이 크다: 라운드 로빈 로드밸런서, 헤더 기반 라우팅, `ttlMs` 캐싱, 표준 트레이스 전파.
- 사용자 확인은 SSE 유지 대신 MRTR로, 긴 작업은 차단 대신 Tasks 확장으로 — 둘 다 "어느 인스턴스든 이어받는다"는 같은 설계 철학 위에 있다.
- Roots·Sampling·Logging은 1년의 유예 기간 후 제거 시계에 올랐다. 새 구현은 대체재로 시작해야 한다.
- 가장 먼저 할 일은 에러 코드 `-32002` 매칭 확인과 `initialize` 의존 제거다. 이 둘이 실제 브레이킹이다.

출처: MCP 공식 블로그 릴리스 노트(blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate), 공식 스펙 문서(modelcontextprotocol.io/specification/2026-07-28), Tasks 확장 문서(modelcontextprotocol.io/extensions/tasks/overview), 2026 MCP 로드맵(blog.modelcontextprotocol.io/posts/2026-mcp-roadmap)
