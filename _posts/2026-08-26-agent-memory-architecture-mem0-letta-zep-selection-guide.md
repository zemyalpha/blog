---
title: "AI 에이전트 메모리 설계: Mem0·Letta·Zep(Graphiti) 아키텍처 비교와 선택 기준"
date: "2026-08-26"
keywords: ["AI 에이전트 메모리", "Mem0", "Letta", "Zep", "LongMemEval"]
lang: "ko"
description: "AI 에이전트의 장기 기억은 이제 모델 선택만큼 중요한 설계 결정이다. Mem0의 추출 기반 메모리 레이어, Letta의 자기 편집형 런타임, Zep의 시간 지식 그래프를 아키텍처와 벤치마크 관점에서 비교하고 선택 기준을 정리한다."
---

# AI 에이전트 메모리 설계: Mem0·Letta·Zep(Graphiti) 아키텍처 비교와 선택 기준

메모리가 없는 에이전트는 비싼 챗봇에 불과하다. 단일 컨텍스트 창 안에서는 그럴듯하게 추론하지만 세션이 끝나는 순간 모든 것을 잊는다. 사용자는 다시 설명해야 하고, 에이전트는 다시 배워야 하고, 아무것도 누적되지 않는다.

이 문제의 심각성은 숫자로 확인된다. ICLR 2025에 발표된 LongMemEval 벤치마크(arXiv:2410.10813)는 지속적인 상호작용 속에서 상용 챗 어시스턴트와 롱컨텍스트 LLM이 **약 30%의 정확도 하락**을 보인다는 것을 측정했다. 컨텍스트 창을 백만 토큰으로 늘려도, 며칠 전 대화에서 나온 사실을 정확히 꺼내 쓰는 것은 별개의 문제라는 뜻이다.

2026년 현재 에이전트 메모리는 하나의 독립된 인프라 카테고리가 됐다. 이 글에서는 생산 환경에서 가장 많이 언급되는 세 접근 — Mem0, Letta(구 MemGPT), Zep/Graphiti — 을 아키텍처 철학부터 벤치마크 읽는 법까지 비교하고, 실제 선택 기준을 정리한다.

## 먼저 용어부터: 메모리 3계층

프레임워크를 비교하기 전에 설계 언어를 통일하자. 인지과학의 분류를 프로덕션 관점으로 정리하면 대략 세 계층이다.

- **작업 메모리(Working memory, in-context)**: 모델이 현재 추론 중인 컨텍스트 창 그 자체. 빠르고 지연이 없지만 용량이 비싸고 세션 간 지속되지 않는다.
- **에피소드 메모리(Episodic memory)**: 무슨 일이 있었는지의 로그. 지난 대화, 완료된 작업, 과거 결정. "3주 전에 하던 워크플로우를 이어서 하는" 능력이 여기에 달렸다.
- **의미 메모리(Semantic memory)**: 사용자에 대한 사실과 개체 간 관계의 지식 베이스. "앨리스는 인프라 팀을 관리한다"는 텍스트 유사도가 아니라 관계(relationship)로 표현해야 제대로 다뤄지는 정보다.

실무에서는 네 번째 계층인 절차 메모리(학습된 행동 규칙, 프롬프트 지시)를 의미 메모리에 합치거나 시스템 프롬프트로 하드코딩하는 경우가 많다. 중요한 건, 대부분의 팀이 첫 빌드에서 세 계층을 모두 제대로 만들지 못한다는 점이다. 인-컨텍스트 메모리로 데모를 만들고, 사용자가 두 번째 세션에 돌아왔을 때 에피소드 검색의 필요를 깨닫고, 비슷한 개체를 헷갈리기 시작하는 규모에 도달해야 관계 모델링을 추가한다.

이 세 계층을 어떻게 채우느냐가 곧 세 프레임워크의 차이다.

## Mem0: 기존 스택에 끼워 넣는 플러그형 메모리 레이어

Mem0(mem0ai/mem0, Apache 2.0, GitHub 스타 약 6.4만 개(63.7k) — 2026년 8월 기준 에이전트 메모리 카테고리 최다)는 "메모리 레이어"다. LangChain이든 CrewAI이든 자체 루프든, 이미 쓰고 있는 에이전트 프레임워크에 `add()`와 `search()` API 두 개로 끼워 넣는다.

2026년 4월에 발표된 새 알고리즘에서 핵심이 되는 설계는 다음 네 가지다(공식 README 기준).

1. **Single-pass ADD-only 추출** — 기억 추출이 LLM 호출 한 번으로 끝나고, UPDATE/DELETE 단계가 없다. 사실이 덮어써지지 않고 누적되므로 이력이 온전하게 보존된다.
2. **에이전트 생성 사실의 동등 대우** — 에이전트가 수행한 행동("방금 배포를 완료했다")도 사용자 발화와 같은 무게로 저장된다.
3. **엔티티 링킹** — 개체를 추출해 임베딩하고 메모리들 사이에 연결해 검색 부스팅에 쓴다.
4. **멀티 시그널 검색** — 시맨틱, BM25 키워드, 엔티티 매칭 세 신호를 병렬로 채점해 하나의 랭킹으로 융합한다.

코드는 이렇게 생겼다(공식 README의 사용 패턴을 간략화):

```python
from openai import OpenAI
from mem0 import Memory

openai_client = OpenAI()
memory = Memory()  # 기본 LLM: gpt-5-mini, 임베딩: text-embedding-3-small

def chat_with_memories(message: str, user_id: str = "default_user") -> str:
    # 1) 관련 기억 검색
    relevant = memory.search(query=message, filters={"user_id": user_id}, top_k=3)
    memories_str = "\n".join(f"- {e['memory']}" for e in relevant["results"])

    # 2) 기억을 시스템 프롬프트에 주입해서 응답 생성
    system_prompt = (
        "You are a helpful AI. Answer based on query and memories.\n"
        f"User Memories:\n{memories_str}"
    )
    response = openai_client.chat.completions.create(
        model="gpt-5-mini",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": message},
        ],
    )
    answer = response.choices[0].message.content

    # 3) 이번 대화를 기억으로 추가
    memory.add(
        messages=[
            {"role": "user", "content": message},
            {"role": "assistant", "content": answer},
        ],
        user_id=user_id,
    )
    return answer
```

설치는 `pip install mem0ai` (또는 `npm install mem0ai`). 하이브리드 검색의 BM25·엔티티 추출까지 쓰려면 `pip install "mem0ai[nlp]"` 후 spaCy 영어 모델을 내려받는다. 셀프호스팅 서버는 `docker compose up`으로 띄우고, 관리형 플랫폼은 가입만 하면 된다.

핵심 트레이드오프는 **예측 가능성**이다. 기억 추출이 개발자 호출 시점에 결정적으로 일어나므로 같은 입력은 같은 기억을 만들고, 메모리 조작이 에이전트의 추론 토큰 예산을 잡아먹지 않는다. 대신 문맥을 이해한 "무엇이 중요한가"의 섬세한 판단은 파이프라인이 정한 규칙의 수준을 넘지 못한다.

## Letta: 에이전트가 사는 OS형 런타임

Letta(letta-ai/letta, Apache 2.0, GitHub 스타 약 2.4만 개)는 접근 자체가 다르다. 끼워 넣는 라이브러리가 아니라 **에이전트가 그 안에서 실행되는 플랫폼**이다. 전신은 2023년 10월 발표된 MemGPT 논문(arXiv:2310.08560)으로, "LLM 컨텍스트를 가상 메모리처럼 다루자"는 아이디어에서 출발했다.

MemGPT의 메타포는 운영체제다. 메모리가 세 계층으로 나뉘고:

- **Core Memory** — 컨텍스트 창 안에 상주하는 작은 블록(RAM). 에이전트가 직접 읽고 쓴다.
- **Recall Memory** — 컨텍스트 밖에 저장되는 검색 가능한 대화 이력(디스크 캐시).
- **Archival Memory** — 도구 호출로만 조회하는 장기 저장소(cold storage).

여기서 중요한 차이 하나. Mem0의 기억이 파이프라인에 의해 **수동적으로(passively) 추출**된다면, Letta의 에이전트는 **스스로 자기 메모리를 편집한다**. 추론 루프 안에서 무언가 중요하다고 판단되면 core/recall/archival에 스스로 쓰고, 필요하면 자기 메모리 계층을 검색하는 도구를 호출한다. 더 적응적이지만, 모델이 저장하지 않기로 결정하면 그 정보는 사라지고, 모든 메모리 연산이 추론 토큰을 소모한다는 비용이 따른다.

2026년 들어 Letta는 구조를 정리했다. 구형 `letta-ai/letta` 리포지토리는 이제 프로젝트 랜딩 페이지 역할을 하고, 실제 소스는 `letta-ai/letta-code`로 옮겨졌다. 설치도 Python이 아니라 npm에서 한다:

```bash
npm install -g @letta-ai/letta-code

# 대화형 터미널 UI 실행
letta

# 로컬/셀프호스티드 에이전트용 앱 서버 실행
letta server
```

데스크톱 앱(macOS/Windows/Linux), 브라우저(chat.letta.com), Slack/Telegram/Discord 채널, TypeScript용 Agent SDK까지 제공한다. 채택 의미를 정확히 알아야 한다 — Letta를 "메모리 솔루션"으로만 도입하려는 팀은, 실제로는 에이전트 실행 전체를 Letta 플랫폼으로 옮기는 결정을 하고 있는 것이다.

## Zep와 Graphiti: 시간이 들어간 지식 그래프

세 번째 축은 그래프다. Zep의 기반 기술인 Graphiti(getzep/graphiti)는 **시간 지식 그래프(temporal knowledge graph)**로, 2025년 11월 GitHub 스타 2만 개를 돌파했다. Zep의 접근을 정리한 논문도 있다(arXiv:2501.13956, "Zep: A Temporal Knowledge Graph Architecture for Agent Memory").

핵심 아이디어는 단순하다. "앨리스는 인프라 팀을 관리한다"는 사실에 **언제 사실이었는지**를 붙인다. 사용자가 "앨리스가 이제 플랫폼 팀을 맡아"라고 말하면 기존 엣지를 지우는 대신 유효기간을 종료하고 새 엣지를 추가한다. 그 결과 "앨리스는 지금 무슨 팀을 관리해?"와 "앨리스는 작년에 무슨 팀을 맡았어?"가 다른 쿼리가 된다. 벡터 검색은 의미적으로 비슷하지만 시간적으로 오래된 결과를 돌려주기 쉬운데, 이것이 바로 나이브한 대화 로깅이 프로덕션에서 처음 부딪히는 벽이다.

논문(arXiv:2501.13956)이 보고하는 수치도 이 방향을 뒷받침한다. MemGPT 팀이 자체 평가 지표로 쓴 Deep Memory Retrieval(DMR) 벤치마크에서 Zep는 94.8%로 MemGPT의 93.4%를 앞섰고, LongMemEval에서는 기준 구현 대비 최대 18.5%의 정확도 향상과 90%의 응답 지연 감소를 달성했다는 것이다. 시간 인식형(temporally-aware) 그래프 엔진인 Graphiti가 대화 데이터와 비즈니스 데이터를 통합하면서 이력 관계를 유지한다는 점이 차별점이다.

2026년 관점에서 눈에 띄는 건 Graphiti MCP 서버 1.0이다. Claude Desktop, Cursor 같은 MCP 클라이언트에서 코드 통합 없이 지식 그래프를 직접 읽고 쓸 수 있다. MCP 서버는 주간 수십만 사용자 규모로 쓰이고 있으며, 설정은 환경변수 대신 YAML로 옮겨갔다. 기본 데이터베이스는 FalkorDB(단일 컨테이너 설정 제공)이고 Neo4j도 지원하며, AWS 팀이 Neptune과 OpenSearch 지원을 기여했다. 9종의 사전 정의 엔티티 타입(Preference, Requirement, Event, Organization 등)이 프로덕션 배포 피드백을 반영해 추출 품질을 끌어올렸다.

## 벤치마크를 읽는 법: 자체 보고와 독립 평가의 격차

메모리 프레임워크를 비교할 때 벤치마크 숫자는 꼭 출처와 함께 읽어야 한다.

Mem0는 2026년 4월 알고리즘 개선으로 LoCoMo 71.4→**92.5**, LongMemEval 67.8→**94.4**를 보고했다. 평균 검색 토큰이 약 7,000개로, 전체 대화 이력을 컨텍스트에 넣는 방식(25,000+ 토큰) 대비 3~4배 효율이라는 주장이다. 다만 두 가지를 분명히 해둔다. 첫째, 이 수치는 **Mem0의 관리형 플랫폼 기준의 자체 보고**다. 오픈소스 SDK에는 포함되지 않은 최적화가 반영되어 있어 OSS 사용자는 "방향성은 비슷하지만 동일한 숫자는 아니다"라고 공식적으로 명시하고 있다. 둘째, 서드파티 독립 평가(vectorize.io, 2026)에서는 Mem0가 LongMemEval 49.0%로 측정된 바 있다. 어느 알고리즘 세대·어떤 배포 환경(관리형 대 OSS) 기준인지가 숫자마다 다르므로, 서로 다른 출처의 수치를 한 표에 놓고 직접 비교하면 잘못된 결론이 나온다.

Letta는 LongMemEval 결과를 공식적으로 공개하지 않았다. 에이전트가 직접 검색 도구를 호출하는 구조 특성상 기반 모델과 프롬프트에 따라 편차가 크기 때문에, 숫자 없이 설계 철학으로 비교하는 수밖에 없다.

그래서 실제 선택은 벤치마크 숫자보다 아래 기준으로 하는 게 정직하다.

## 선택 기준 정리

| 상황 | 추천 | 이유 |
|------|------|------|
| 기존 에이전트 스택이 있고 빠르게 개인화를 붙이고 싶다 | **Mem0** | 프레임워크 무관 `add()`/`search()` API, 예측 가능한 추출 파이프라인, 가장 큰 커뮤니티 |
| 장기 실행 에이전트를 처음부터 새로 만든다 | **Letta** | 메모리와 실행이 통합된 런타임, 에이전트의 자기 메모리 편집, 상태 지속성 |
| 시간에 따라 변하는 사실이 핵심 도메인이다 | **Zep/Graphiti** | bi-temporal 지식 그래프, "지금 사실"과 "과거 사실"의 구분, MCP 네이티브 |
| 오프라인/데이터 주권이 최우선 | Graphiti 셀프호스팅 또는 OSS SDK | FalkorDB 단일 컨테이너, 외부 API 호출 최소화 |

여기에 라이선스와 배포 현실을 얹으면 결정이 더 명확해진다. 셋 모두 Apache 2.0 계열의 오픈소스가 있지만, 각사의 관리형 플랫폼(Mem0 Platform, Letta Cloud, Zep SaaS)에 고급 기능이 몰려 있는 구조라 "오픈소스 vs SaaS-in-disguise"를 처음부터 구분해두는 것이 좋다.

## 실전에서 자주 나오는 실수

- **메모리를 RAG로 대체하려는 것** — 대화 이력을 문서처럼 벡터스토어에 넣고 검색하는 방식은 시간 정보와 지식 갱신을 다루지 못한다. LongMemEval 논문이 지적하는 30% 하락의 상당 부분이 여기서 온다. 인덱싱-검색-리딩 3단계로 분해해서 설계하라는 것이 같은 논문의 결론이다.
- **벤치마크 숫자만 보고 선택하는 것** — 앞서 봤듯 자체 보고 수치는 관리형 플랫폼 기준이고, 독립 평가는 알고리즘 세대가 다를 수 있다. 자기 워크로드(대화 길이, 도메인, 갱신 빈도)로 소규모 재현 평가를 하는 편이 빠르다.
- **메모리 계층을 한 번에 다 만드는 것** — 작업 메모리로 시작해서, 두 번째 세션 문제가 보이면 에피소드 검색을, 개체 충돌이 생기면 관계 모델링을 추가하는 순서가 현실적이다.
- **덮어쓰기 전략의 부재** — "사용자가 이사했다"는 사실을 덮어쓰면 이력이 사라지고, 누적만 하면 오래된 사실이 새 사실과 경쟁한다. Mem0의 ADD-only + 시간 랭킹이나 Graphiti의 엣지 유효기간처럼, 갱신 전략은 반드시 설계 단계에서 정해야 한다.

## 결론

- 에이전트 메모리는 컨텍스트 창 확대로 해결되지 않는 독립 문제이며, LongMemEval이 측정한 ~30% 정확도 하락이 그 증거다.
- 세 접근의 본질은 "누가 무엇을 결정하느냐"다. Mem0는 파이프라인이 수동 추출하고, Letta는 에이전트가 스스로 편집하고, Zep/Graphiti는 시간이 붙은 그래프가 사실의 생애주기를 관리한다.
- 벤치마크는 자체 보고/독립 평가, 알고리즘 세대, 평가 환경을 구분해서 읽어야 한다. 특히 관리형 플랫폼 기준 숫자는 OSS 사용자에게 그대로 적용되지 않는다.
- 선택은 교체 비용으로 하라. 라이브러리(Mem0)는 하루면 붙였다 뗀다. 런타임(Letta)은 에이전트 실행 전체를 옮기는 결정이다. 그래프(Zep)는 도메인의 시간성이 강할수록 빛난다.

당장 해볼 첫 단계는 작다. 지금 만들고 있는 에이전트에 Mem0 OSS SDK로 `add()`/`search()`를 붙이고, 같은 사용자의 두 번째 세션에서 무엇이 기억나고 무엇이 사라지는지 관찰해보라. 메모리 설계의 필요 계층은 그 관찰에서 나온다.

**참고 자료**
- Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory — arXiv:2504.19413
- MemGPT: Towards LLMs as Operating Systems — arXiv:2310.08560
- Zep: A Temporal Knowledge Graph Architecture for Agent Memory — arXiv:2501.13956
- LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory — arXiv:2410.10813 (ICLR 2025)
- mem0ai/mem0, letta-ai/letta, getzep/graphiti GitHub 저장소 (2026-08-26 확인)
