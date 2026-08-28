---
title: "MCP 서버의 91.5%는 OAuth 없이 돌아간다: Astrix 스캔 5,200대가 드러낸 인증 실태와 OAuth 2.1 리소스 서버 구현"
date: "2026-08-28"
keywords: ["MCP 보안", "OAuth 2.1", "Model Context Protocol", "MCP 서버 인증", "CVE-2025-6514", "Astrix", "토큰 검증"]
lang: "ko"
description: "Astrix가 5,200개 오픈소스 MCP 서버를 스캔한 결과 OAuth를 쓰는 서버는 8.5%뿐이었다. 실제 침해 사례 3건을 해부하고 Python SDK로 JWT 토큰을 검증하는 OAuth 2.1 리소스 서버를 구현하는 방법을 다룬다."
---

# MCP 서버의 91.5%는 OAuth 없이 돌아간다: Astrix 스캔이 드러낸 인증 실태과 OAuth 2.1 구현

2026년 7월, MCP(Model Context Protocol)의 월간 SDK 다운로드가 보도에 따르면 4억 회를 넘어섰다. 올해 초 9,700만 수준에서 몇 달 만에 4배가량 성장한 수치다. 그런데 이 생태계의 지하 수도관을 열어 보면 초라한 풍경이 펼쳐진다. 보안 업체 Astrix가 2025년 10월 오픈소스 MCP 서버 5,200개를 직접 스캔한 결과, 공식 스펙이 권장하는 OAuth를 실제로 구현한 서버는 **8.5%** 에 불과했다.

문제는 이 숫자가 이론적인 경고가 아니라는 점이다. 2025년 여름 이후 MCP 생태계에서는 실제 사고가 연쇄적으로 터졌다. Asana의 테넌트 격리 붕괴, mcp-remote의 원격 코드 실행(RCE), Cursor의 승인 우회. 셋 다 국가급 해커가 필요하지 않았다. 검증 단계 하나를 건너뛴 서버 하나면 충분했다.

이 글은 ① Astrix 스캔이 보여준 인증 실태, ② 실제로 터진 사고 3건의 해부, ③ Python SDK로 OAuth 2.1 토큰 검증을 직접 구현하는 방법을 다룬다. MCP 서버를 만들고 있거나, 사내 에이전트가 외부 MCP 서버에 붙어 있다면 읽을 가치가 있다.

## 1. 실태: 88%가 자격증명을 요구하지만 절반은 죽지 않는 키

Astrix Research가 2025년 10월 15일 발표한 "State of MCP Server Security 2025" 리포트의 핵심 수치는 다음과 같다. 오픈소스 MCP 서버 구현체 5,200개 이상을 고유 기준으로 수집해 인증 방식을 분류했다.

| 항목 | 수치 |
|------|------|
| 어떤 형태든 자격증명을 요구하는 서버 | 88% |
| 만료 없는 정적 시크릿(API 키, PAT)에 의존 | 53% |
| OAuth 구현 | 8.5% |
| API 키를 쓰는 서버 중 환경변수에서 직접 읽는 비율 | 79% |

읽어야 할 지점은 "88%가 자격증명을 요구한다"가 아니라 "그중 절반이 절대 만료되지 않는 키"라는 점이다. 개인 액세스 토큰(PAT)이나 API 키는 한 번 유출되면 폐기 전까지 유효하다. 그리고 MCP의 사용 패턴은 유출에 최액으로 불리하다. MCP 서버는 Claude Desktop, Cursor, 자체 에이전트 같은 다양한 클라이언트가 사용자를 대신해 호출한다. 하나의 공유 시크릿으로 여러 호출자를 구분할 방법이 없으니, 한 클라이언트가 유출되면 전체 서버가 함께 노출된다.

비공식 레지스트리인 mcp.so에는 16,000개 이상의 MCP 서버가 인덱싱되어 있다. 이 시장에서 API 키 발급 방식이 사실상 표준처럼 굳어진 것이다. Astrix는 같은 리포트에서 즉시 처방으로 "MCP Secret Wrapper"라는 오픈소스를 공개했는데, 서버를 감싸서 런타임에 볼트에서 시크릿을 가져오는 방식이다. 근본 해결은 아니지만 호스트 머신에 시크릿이 상주하는 문제는 줄인다.

## 2. 사고는 이미 터졌다: 3건의 해부

### Asana — 테넌트 격리 붕괴 (2025년 6월)

Asana는 2025년 6월 4일 자사 MCP 서버에서 데이터 노출 버그를 확인했다. MCP 사용자의 권한 범위 안에 있던 데이터 — 태스크 세부 내용, 프로젝트 메타데이터, 팀 정보, 코멘트, 업로드 파일 — 가 **다른 조직으로 새어 나갈 수 있었던** 것이다. Asana 추산 영향 범위는 약 1,000개 고객 조직. 외부 해킹이 아니라 논리 오류였다. 에이전트가 "사용자 권한 범위"를 넘어서는 게 아니라, 그 권한 검증 자체가 조직 경계를 고려하지 않은 설계였다.

### CVE-2025-6514 — mcp-remote 커맨드 인젝션 (CVSS 9.6)

JFrog 보안 연구팀(Or Peles)이 2025년 7월 9일 공개한 이 취약점은 MCP의 인증 절차 자체를 공격 무기로 뒤집는다. mcp-remote는 로컬 클라이언트를 원격 MCP 서버에 연결해 주는 도구로, 연결할 때 서버가 알려주는 `authorization_endpoint` URL을 처리한다. 문제는 악성 MCP 서버가 이 URL에 `file:/c:/windows/system32/calc.exe` 같은 값을 싣으면, 클라이언트가 검증 없이 이를 셸에 전달했다는 것. 즉 **연결하려는 순간 서버 쪽에서 클라이언트의 코드 실행이 가능했다.** 영향 버전은 0.0.5부터 0.1.15까지다. 스펙이 이미 요구하는 검증을 클라이언트가 건너뛴 사례다.

### CVE-2025-54136 — Cursor 승인 우회 (CVSS 8.8)

Cursor 1.2.4 이하에서는 이미 승인된 MCP 서버 정의를 수정해도 재승인 경고가 뜨지 않았다. 공유 GitHub 저장소에 들어 있는 MCP 설정 파일을, 협업자가 승인한 뒤 악성 명령으로 조용히 바꾸면 되는 구조다. 사용자 입장에서는 무해해 보이는 MCP 서버를 한 번 수락한 것뿐인데, 그 순간부터 설정 파일 쓰기 권한을 가진 누구나 코드 실행을 이어받는다. 1.3에서 수정되었다.

세 사건의 공통점은 명확하다. 암호학이 뚫린 게 아니라 **신뢰 경계(trust boundary)의 설계 오류**였다. MCP는 호출자가 브라우저가 아니라 에이전트이고, 서버가 다시 다른 서비스의 OAuth 클라이언트가 되는 이중 구조라서, 기존 웹 앱의 "로그인 세션 하나" 모델이 그대로는 작동하지 않는다. 스펙은 이걸 알고 있고, 그래서 답이 OAuth 2.1의 특정 부분집합이다.

## 3. 스펙이 실제로 요구하는 것

공식 문서의 Authorization 섹션은 MCP 인증이 "OAuth 2.1의 선별된 부분집합"임을 명시한다. 전부 구현할 필요는 없지만, 핵심 조각은 고정되어 있다.

- **MCP 서버는 OAuth 2.1 리소스 서버** 역할만 하면 된다. 토큰을 발급하는 인가 서버 구현은 스펙 범위 밖이며, Auth0·Okta·WorkOS 등 기존 IdP를 붙이는 것이 의도된 사용법이다.
- **RFC 9728 (Protected Resource Metadata)** — MCP 서버는 반드시 구현. 클라이언트가 인가 서버 위치를 발견하는 통로다.
- **RFC 8707 (Resource Indicators)** — 토큰을 특정 MCP 서버에 묶어 다른 곳에서 재사용 못 하게 한다. 6514번 취약점이 노린 발견(discovery) 절차의 안전장치와 세트다.
- **OAuth 2.1 코어** — PKCE 필수, implicit/password grant 제거.
- **2025년 11월 개정**부터는 Client ID Metadata Documents(사전 등록 없이 HTTPS JSON 문서로 클라이언트 식별)가 권장되고, 동적 클라이언트 등록(RFC 7591)은 역호환용으로 남았다. 같은 개정에서 스코프 부족 시 `403` + `WWW-Authenticate: error="insufficient_scope"`로 응답하는 step-up 흐름도 정식화됐다.

stdio 전송은 예외다. 스펙상 환경변수에서 자격증명을 가져오도록 되어 있어 HTTP 방식 인증을 따르지 않는다. 문제는 HTTP로 노출된 서버 중 상당수가 이 stdio 관행을 그대로 들고 온다는 것이고, 그 결과가 79%의 환경변수 API 키다.

## 4. 실전: JWT 검증 리소스 서버 30분 만에 붙이기

결론부터: 인가 서버를 직접 만들지 마라. IdP가 발급한 JWT 액세스 토큰을 MCP 서버가 로컬에서 검증하는 구조가 정답이다. 공식 Python SDK(`mcp`)의 `TokenVerifier` 인터페이스를 구현하면 된다.

**Step 1 — 의존성 설치**

```bash
uv add "mcp[cli]" pyjwt
# 또는 pip install "mcp[cli]" pyjwt
```

**Step 2 — 토큰 검증자 작성**

IdP가 JWT를 발급한다면(Auth0·Okta·WorkOS 모두 기본 JWT), 요청마다 네트워크를 호출할 필요 없이 발급자의 JWKS 공개키로 로컬 검증한다.

```python
import jwt
from jwt import PyJWKClient
from mcp.server.auth.provider import AccessToken, TokenVerifier

class JWTTokenVerifier(TokenVerifier):
    """발급자의 JWKS로 JWT 액세스 토큰을 로컬 검증한다."""

    def __init__(self, issuer: str, audience: str, jwks_url: str):
        self.issuer = issuer
        self.audience = audience
        self.jwks_client = PyJWKClient(jwks_url, cache_jwk_set=True)

    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            signing_key = self.jwks_client.get_signing_key_from_jwt(token)
            claims = jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],
                audience=self.audience,
                issuer=self.issuer,
            )
        except jwt.PyJWTError:
            return None  # 검증 실패 = 익명 요청으로 처리 → 거부
        return AccessToken(
            token=token,
            client_id=claims.get("azp", claims.get("client_id", "unknown")),
            scopes=claims.get("scope", "").split(),
            expires_at=claims.get("exp"),
            resource=self.audience,
            subject=claims.get("sub"),
            claims=claims,
        )
```

핵심은 `verify_token`이 실패 시 `None`을 반환하도록 하는 것이다. SDK는 `None`을 받으면 요청 자체를 인가 오류로 처리한다. `jwt.decode`의 `audience=self.audience` 인자가 RFC 8707의 오디언스 검증을 대신 수행하는데, 이것이 스펙 문서가 "가장 자주 건너뛰는 단계"로 꼽는 검증이다. 토큰의 `aud` 클레임이 내 서버 URI와 정확히 일치하지 않으면 PyJWT가 `InvalidAudienceError`를 던진다.

**Step 3 — 서버에 연결하고 스코프별로 문 걸기**

```python
from mcp.server.fastmcp import FastMCP
from mcp.server.auth.settings import AuthSettings
from pydantic import AnyHttpUrl

mcp = FastMCP(
    "my-secure-server",
    token_verifier=JWTTokenVerifier(
        issuer="https://your-idp.example.com",
        audience="https://mcp.example.com",  # Resource Indicator로 묶은 자기 자신
        jwks_url="https://your-idp.example.com/.well-known/jwks.json",
    ),
    # AuthSettings가 RFC 9728 메타데이터 발행 + 401 WWW-Authenticate 헤더를 자동 처리
    auth=AuthSettings(
        issuer_url=AnyHttpUrl("https://your-idp.example.com"),
        resource_server_url=AnyHttpUrl("https://mcp.example.com"),
        required_scopes=["mcp:tools-basic"],  # 와일드카드 금지 — 최소한으로
    ),
)

@mcp.tool(description="고객 데이터 조회")
def get_customer(customer_id: str) -> dict:
    # 호출자 신원은 컨텍스트 객체가 아니라 전용 헬퍼로 읽는다
    from mcp.server.auth.middleware.auth_context import get_access_token
    token = get_access_token()
    if token is None or "customers:read" not in token.scopes:
        raise PermissionError("insufficient_scope: customers:read required")
    ...
```

테스트는 간단하다. 인증 없이 curl을 날리면 `WWW-Authenticate: Bearer resource_metadata=".../.well-known/oauth-protected-resource/mcp"` 헤더와 함께 401이 돌아와야 한다. 이 헤더가 바로 RFC 9728 발견 메커니즘이고, 클라이언트는 이 URL에서 인가 서버 위치를 찾는다.

배포 시 체크리스트는 짧다. (1) `authorization_endpoint` 등 발견 절차에서 서버가 준 URL은 화이트리스트 스킴(`https:`)만 수용 — CVE-2025-6514의 교훈. (2) 정적 API 키를 쓰고 있다면 최소한 만료일을 걸고 발급자별로 분리. (3) MCP 설정 파일(`.mcp.json` 등)의 변경 이력을 코드 리뷰 대상에 포함 — CVE-2025-54136의 교훈.

## 결론

- MCP 생태계는 성장 속도(올해 들어 SDK 다운로드 4배 증가)에 비해 인증 성숙도가 크게 뒤처져 있다. OAuth 적용률 8.5%, 만료 없는 정적 시크릿 53%가 그 증거다.
- 실제 사고 3건(Asana 테넌트 노출, mcp-remote RCE, Cursor 승인 우회)은 모두 신뢰 경계 설계 오류에서 나왔다. 새로운 공격 기법이 아니라 웹 보안의 기본기를 에이전트 맥락에 다시 적용하는 문제다.
- MCP 스펙은 OAuth 2.1 부분집합 + RFC 9728/8707으로 리소스 서버 역할만 요구한다. 인가 서버는 기존 IdP에 맡기고, 서버는 토큰 검증에 집중하면 된다.
- 당장 할 일: 자신이 운영하는 MCP 서버의 인증 방식을 확인하고, HTTP로 노출된 서버라면 위의 `TokenVerifier` 패턴으로 전환한다. 30분과 파이 라이브러리 하나면 충분하다.

에이전트가 대신 호출하는 세계에서 인증은 사용자 경험이 아니라 공격면(attack surface)이다. 8.5%에 속할지 말지는 이제 선택의 문제다.

**참고 자료**
- Astrix, "State of MCP Server Security 2025" (2025-10-15) — https://astrix.security/learn/blog/state-of-mcp-server-security-2025/
- JFrog Security Research, CVE-2025-6514 (CVSS 9.6) — https://research.jfrog.com/vulnerabilities/mcp-remote-command-injection-rce-jfsa-2025-001290844/
- Oligo Security, CVE-2025-49596 (CVSS 9.4) — https://www.oligo.security/blog/critical-rce-vulnerability-in-anthropic-mcp-inspector-cve-2025-49596
- MCP 공식 스펙, Authorization — https://modelcontextprotocol.io/specification/draft/basic/authorization
- Nudge Security, "Asana MCP server data exposure incident" (2025-06-18) — https://www.nudgesecurity.com/post/asana-mcp-server-data-exposure-incident
