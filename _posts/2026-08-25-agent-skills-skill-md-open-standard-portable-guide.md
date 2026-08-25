---
title: "SKILL.md 하나로 에이전트를 가르치는 시대: Agent Skills 오픈 표준 실전 가이드"
date: "2026-08-25"
keywords: ["Agent Skills", "SKILL.md", "Claude Code", "Codex", "오픈 표준"]
lang: "ko"
description: "Anthropic이 연 SKILL.md 오픈 표준이 Claude Code, Codex, Cursor, Gemini CLI를 어떻게 하나로 묶는지, 실제 스킬을 만들어 검증하는 과정까지 정리했다."
---

# SKILL.md 하나로 에이전트를 가르치는 시대: Agent Skills 오픈 표준 실전 가이드

같은 팀에서 Claude Code를 쓰는 개발자는 `CLAUDE.md`를, Codex CLI를 쓰는 개발자는 `AGENTS.md`를, Cursor를 쓰는 개발자는 `.cursorrules`를 관리해왔다. 내용이 거의 같은 파일이 도구마다 복사되고, 한쪽만 고치면 다른 쪽은 금방 낡는다. 컨벤션이 바뀔 때마다 세 파일을 순서대로 수정하는 일은 AI 시대에 어울리지 않는 낭비였다.

이 문제에 대한 업계의 답이 **Agent Skills 오픈 표준**이다. Anthropic이 2025년 10월 Claude Code에 Skills 기능을 처음 선보인 뒤, 같은 해 12월 18일 스펙을 agentskills.io라는 이름의 오픈 표준으로 공개했다. 핵심은 놀랄 만큼 단순하다. **폴더 하나와 `SKILL.md` 파일 하나**다. 빌드 단계도, 패키지 매니저도, 런타임도 없다.

이 글에서는 이 표준이 정확히 어떻게 동작하는지, 실제로 쓸 만한 스킬을 직접 만들어보며 확인한다.

## Skills는 정확히 무엇이고, 기존 설정 파일과 뭐가 다른가

Skill은 도메인 지식을 담은 폴더다. 최소 구조는 이렇다:

```
my-skill/
├── SKILL.md          # 필수: 메타데이터 + 지시사항
├── scripts/          # 선택: 실행 가능한 코드
├── references/       # 선택: 상세 문서
└── assets/           # 선택: 템플릿, 리소스
```

`CLAUDE.md`가 "이 프로젝트에서 항상 지켜라"라는 상시 지침이라면, Skill은 "이 작업을 할 때만 펼쳐 읽는 매뉴얼"이다. 이 차이를 가능하게 하는 설계가 **점진적 공개(progressive disclosure)**다. 3단계로 동작한다:

1. **발견(Discovery)** — 에이전트 시작 시 모든 스킬의 `name`과 `description`만 시스템 프롬프트에 올린다. 각 스킬 수싱~수백 토큰 수준이다.
2. **활성화(Activation)** — 사용자 요청이 특정 스킬의 `description`과 맞다고 판단하면, 그때 전체 `SKILL.md`를 컨텍스트로 읽어들인다.
3. **실행(Execution)** — 본문 지시에 따라 작업하며, 필요할 때만 `references/`의 개별 파일을 추가로 읽거나 `scripts/`의 코드를 실행한다.

즉, 스킬 50개를 설치해도 평상시 컨텍스트 비용은 메타데이터 50줄 수준이다. Anthropic의 엔지니어링 블로그는 이를 "목차 → 챕터 → 부록"에 비유한다. PDF 처리 스킬의 경우 양식 채우기 지침을 별도 파일(`forms.md`)로 분리해, 양식을 채울 때만 그 파일을 읽게 만드는 식이다.

프롬프트에 전부 밀어 넣는 방식과의 차이는 극적이다. 컨텍스트 창이 커져도 관련 없는 지식이 추론 품질을 희석시키는 문제는 남는데, Skills는 구조적으로 이를 피한다.

## 스펙의 최소 조건과 프론트매터 규칙

agentskills.io의 공식 스펙에서 필수 필드는 단 둘이다:

| 필드 | 필수 | 제약 |
|------|------|------|
| `name` | O | 최대 64자, 소문자·숫자·하이픈만. 디렉터리명과 일치해야 함 |
| `description` | O | 최대 1024자. 무엇을 하는 스킬인지 + 언제 쓰는지 |
| `license` | X | 라이선스명 또는 번들된 라이선스 파일 참조 |
| `compatibility` | X | 최대 500자. 환경 요구사항 |
| `metadata` | X | 임의의 키-값 쌍 |
| `allowed-tools` | X | 사전 승인된 도구 목록 (실험적) |

`name`에는 잔소리가 많다. 대문자 불가, 하이픈으로 시작/끝 불가, 연속 하이픈(`--`) 불가, 그리고 **부모 디렉터리명과 반드시 일치**해야 한다. 이 제약 덕분에 스킬 이름이 곧 설치 경로가 되고, 충돌이 없다.

`description`이 실전에서 가장 중요한 필드다. 에이전트는 이 문장만 보고 스킬을 발동할지 결정한다. "커밋 메시지를 생성할 때, 사용자가 커밋을 요청할 때 사용. 스테이지되지 않은 변경에는 사용 금지"처럼 **언제 쓸지와 언제 쓰지 않을지를 함께 적어야** 오탐이 줄어든다.

## 실전: 15분 만에 팀 전체가 쓰는 스킬 만들기

Conventional Commits 규칙을 검사하는 스킬을 만들어보자. 어느 도구에서나 똑같이 동작한다.

**1단계 — 디렉터리와 SKILL.md 작성:**

```markdown
---
name: conventional-commit
description: >
  Generates a Conventional Commits message from staged git changes.
  Use when the user asks to commit or write a commit message.
  Do NOT use for unstaged changes.
license: MIT
metadata:
  author: my-team
  version: "1.0"
---

## Instructions

1. Run `git diff --cached --stat` to see staged changes.
2. Classify the change: feat, fix, refactor, docs, test, chore, perf.
3. Write a message matching: type(scope): subject (max 72 chars).
4. Validate the message with scripts/validate.py before presenting it.
```

**2단계 — 검증 스크립트 추가** (`scripts/validate.py`):

```python
import sys, re

PATTERN = r'^(feat|fix|refactor|docs|test|chore|ci|perf|style|build)(\(.+\))?: .{1,72}$'
message = sys.stdin.readline().strip()
sys.exit(0 if re.match(PATTERN, message) else 1)
```

에이전트는 이 스크립트를 **컨텍스트에 올리지 않고 실행만 한다**. 정규식 검증 같은 결정적 작업을 토큰 생성으로 하는 것은 비싸고 불안정하므로, 코드로 맡기는 게 스킬 설계의 기본 원칙이다.

**3단계 — 리포지터리에 커밋:**

```bash
git add .agents/skills/conventional-commit/
git commit -m "chore: add conventional-commit skill"
```

이제 리포를 클론한 누구나 자기 에이전트에서 같은 스킬을 쓴다. Claude Code 사용자든 Codex 사용자든 별도 설정이 필요 없다. 이것이 이 표준의 실질적 가치다.

## 도구별 채택 현황: 어디까지 호환되나

2026년 8월 현재 agentskills.io의 클라이언트 쇼케이스에는 수십 개 제품이 등재되어 있다. 확인된 주요 도구들이다:

- **Claude Code / Claude.ai / Agent SDK** — 원천 구현. `context: fork`(서브에이전트 실행), `user-invocable`(슬래시 커맨드 노출) 등 자체 확장 필드 보유
- **OpenAI Codex CLI** — `.agents/skills/`, `$HOME/.agents/skills/` 등 네 곳을 스캔. `$skill-creator` 마법사와 `$skill-installer` 내장
- **Cursor** — 공식 문서(cursor.com/docs/skills)에서 SKILL.md 기반 스킬의 자동 발견과 슬래시 커맨드 호출을 명시. 업계 보도에 따르면 2026년 초 2.4 릴리스를 기준으로 에디터와 CLI 지원이 정식화됐다
- **Gemini CLI** — 공식 튜토리얼(geminicli.com)로 스킬 생성 가이드 제공
- **그 외** — GitHub Copilot, Kiro, Goose, OpenCode, Letta, Tabnine 등. 개인 에이전트 영역에서도 Nous Research의 **Hermes Agent**가 클라이언트 목록에 이름을 올리고 있다

거버넌스 측면에서도 의미 있는 변화가 있었다. 업계 보도에 따르면 MCP에 이어 Agent Skills도 리눅스재단 산하 Agentic AI Foundation의 관리 체계와 연계되며 단일 벤더 종속 우려가 줄어든 상태다. 다만 표준 핵심(name/description/선택 디렉터리)은 어디서나 동일하게 동작하지만, **스킬 발견 경로와 호출 방식은 도구마다 다르다**는 점은 문서에서 명시적으로 경고하고 있으니 각 도구의 스킬 디렉터리 위치는 확인이 필요하다.

도구별 확장 필드(`agents/openai.yaml`, `when_to_use` 등)는 이해하지 못하는 도구가 무시하므로, 하나의 스킬에 공존해도 충돌하지 않는다. 다만 그런 필드에 의존하는 순간 이식성을 잃는다.

## Skills vs MCP: 경쟁이 아니라 분업

"MCP가 이미 있는데 왜 또 표준인가"라는 질문이 자연스럽다. 답은 역할이 다르기 때문이다.

- **MCP 서버**는 에이전트가 *무엇을 할 수 있는지*를 바꾼다. 외부 API, 데이터베이스, 브라우저 연결.
- **Skills**는 에이전트가 *어떻게 생각하는지*를 바꾼다. 방법론, 워크플로, 도메인 지식.

MCP로 Jira에 티켓을 만들 수 있게 하고, Skills로 "우리 팀의 티켓 작성 규칙"을 가르치는 식으로 조합한다. 실제로 Anthropic도 Skills가 MCP 서버로는 전달하기 어려운 복잡한 워크플로를 보완한다고 설명한다.

## 주의사항: 편리함의 이면

- **보안** — 스킬은 지시문과 실행 코드의 묶음이다. 악성 스킬은 데이터 유출이나 의도치 않은 명령 실행을 유도할 수 있다. Anthropic은 신뢰할 수 있는 출처의 스킬만 설치하고, 외부 스킬은 번들된 스크립트와 의존성을 전부 검토하라고 권고한다. npm 초기를 떠올리게 하는 풍경이다.
- **`allowed-tools`는 실험적** — 프론트매터로 실행 도구를 제한하는 필드가 있지만 스펙 단계에서도 실험적(experimental)으로 명시되어 있어 보안 경계로 삼아선 안 된다.
- **오탐 발동** — `description`이 모호하면 관련 없는 작업에서 스킬이 활성화된다. 실제 사용 로그를 보고 "언제 사용하지 않는지"를 계속 추가하는 것이 스킬 유지보수의 핵심이다.
- **설정 파일과의 병행** — SKILL.md가 CLAUDE.md/AGENTS.md를 완전히 대체한다기보다, 프로젝트 상시 규칙은 기존 파일에, 작업별 절차 지식은 스킬에 나누는 현실적 운영이 자리 잡고 있다.

## 결론

Agent Skills 표준의 힘은 기술적 화려함이 아니라 **적당한 최소함**에서 나온다. 마크다운 파일 하나, 필수 필드 두 개. 그 단순함이 채택 속도를 만들었고, Claude Code·Codex·Cursor·Gemini CLI가 같은 파일을 읽는 생태계가 됐다.

- 도구마다 지침 파일을 복사해 유지하던 시대는 끝나가고 있다 — `.agents/skills/` 폴더 하나로 팀 전체가 공유한다
- 점진적 공개 구조 덕분에 스킬을 많이 깔아도 컨텍스트 비용은 거의 0에 수렴한다
- 결정적 작업은 `scripts/`로, 절차 지식은 `SKILL.md`로 분리하는 것이 설계 원칙이다
- 스킬 설치 전 출처 검증은 필수다 — 이 생태계는 아직 보안 성숙 단계 초입이다

시작은 작게 하면 된다. 오늘 팀에서 반복적으로 설명하던 절차 하나 — 커밋 규칙이든, 배포 체크리스트든 — 를 골라 `SKILL.md`로 옮겨보라. 그 한 파일이 에이전트를 가르치는 가장 싸고 튼튼한 방법이 될 것이다.
