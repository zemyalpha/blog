---
title: "GraphRAG 정말 필요한가: RAGSearch 벤치마크가 밝힌 multi-hop 성능과 1M 토큰당 구축 비용"
date: "2026-09-02"
keywords: ["GraphRAG", "Agentic Search", "RAG 벤치마크", "Search-R1", "HippoRAG", "multi-hop QA"]
lang: "ko"
description: "RAGSearch 벤치마크 논문을 분석해 agentic search 시대에 GraphRAG가 언제 필요한지, 1M 토큰당 구축 비용과 성능 데이터로 정리한다."
---

# GraphRAG 정말 필요한가: RAGSearch 벤치마크가 밝힌 multi-hop 성능과 1M 토큰당 구축 비용

RAG 시스템을 설계하다 보면 항상 같은 갈림길에 서게 된다. 단순한 임베딩 기반 dense retrieval로 충분한가, 아니면 엔티티를 뽑아 그래프를 만드는 GraphRAG 파이프라인을 구축해야 하는가. GraphRAG는 문서 간 관계를 명시적으로 연결해 multi-hop 질문에 강하다고 알려져 있지만, 그 대가로 오프라인 구축 비용과 복잡도를 감수해야 한다.

2026년 4월 arXiv에 공개된 "Do We Still Need GraphRAG? Benchmarking RAG and GraphRAG for Agentic Search Systems"는 이 질문에 정량적인 답을 준다. 이 논문이 특히 가치 있는 이유는 두 가지다. 첫째, Search-R1 같은 agentic search(에이전트가 여러 라운드로 검색하고 추론하는 방식)가 보편화된 지금, "에이전트가 열심히 검색하면 그래프 없어도 되는가"라는 실전적 질문을 직접 다룬다. 둘째, LLM 백본, 검색 예산, 추론 프로토콜을 통일하고 전체 테스트셋으로 평가해 기존 논문들의 불공정한 비교 관행을 바로잡았다.

이 글에서는 이 벤치마크(RAGSearch)의 핵심 수치를 정리하고, 실제 서비스에서 retrieval 백엔드를 고를 때 적용할 수 있는 의사결정 기준을 제안한다.

## RAGSearch 벤치마크는 어떻게 설계되었나

RAGSearch는 dense RAG과 5가지 GraphRAG 변형을 "교체 가능한 retrieval 백엔드"로 취급한다. 그리고 그 위에 4가지 agentic search 시스템을 얹어 동일 조건에서 비교한다.

**Retrieval 백엔드 (6종):**
- Dense RAG (vanilla, 2018 Wikipedia 덤프 사용)
- GraphRAG (Microsoft, Edge et al. 2024)
- RAPTOR (트리 기반 계층 요약)
- HippoRAG2 (엔티티 중심 코퍼스 그래프 + Personalized PageRank)
- HyperGraphRAG (하이퍼그래프)
- LinearRAG (관계 추출 없는 Tri-Graph)

**Agentic search 시스템 (4종):**
- Training-free: Search-o1 (추론 중 온디맨드 검색), GraphSearch (쿼리 분해 + 그래프/청크 동시 조회)
- RL 기반: Search-R1 (강화학습으로 multi-turn 검색 학습), Graph-R1 (GraphRAG 설정으로 확장)

백본은 Qwen-2.5-7B, 지식 구축에는 GPT-4o-mini를 사용했다. 데이터셋은 일반 QA 3종(NQ, PopQA, TriviaQA)과 multi-hop QA 3종(HotpotQA, 2Wiki, Musique)이다.

## 결과 1: single-shot에서 GraphRAG의 우위는 multi-hop에 집중된다

한 번의 검색으로 답하는 single-shot 설정에서 차이는 극명하게 갈린다.

**일반 QA에서는 GraphRAG의 이득이 평균 +0.47에 불과했다.** NQ/PopQA/TriviaQA처럼 단일 사실을 찾으면 되는 질문에서는 dense RAG로 충분하다는 뜻이다. PopQA에서는 오히려 GraphRAG가 0.95 낮았다.

반면 **multi-hop QA에서는 평균 +27.23의 압도적인 격차**가 나타났다. HotpotQA +27.70, 2Wiki +27.03, Musique +26.96. "A의 감독이 일한 스튜디오의 모회사는 어디인가" 같은 연쇄 추론 질문에서는 그래프 구조가 문서 사이를 직접 연결해주기 때문이다.

정리하면 single-shot 시대의 결론은 단순했다: 일반 QA는 dense RAG, multi-hop QA는 GraphRAG.

## 결과 2: agentic search는 갭을 줄이지만 완전히 없애지는 못한다

논문의 핵심 발견은 여기서 나온다. 에이전트가 쿼리를 분해하고 여러 라운드로 검색하면 dense RAG도 크게 좋아진다.

Training-free GraphSearch 워크플로우에서 dense RAG와 GraphRAG의 multi-hop 성능 격차는 single-shot의 +27.23에서 +26.59로 좁혀졌고, 두 번째로 좋은 GraphRAG 변형 대비로는 32.3% 감소했다. RL로 학습된 Search-R1/Graph-R1 설정에서는 격차가 더욱 좁혀진다.

그러나 주의할 점이 두 가지 있다.

첫째, **에이전트 설계가 나쁘면 오히려 성능이 떨어진다.** Search-o1 아래에서 dense RAG는 일부 벤치마크에서 single-shot보다 성능이 하락했다. "에이전트만 얹으면 알아서 좋아진다"는 통념은 성립하지 않으며, 쿼리 분해와 반복 검색이 제대로 설계된 워크플로우여야 효과가 있다.

둘째, **복잡한 multi-hop에서는 GraphRAG가 여전히 최강이고 더 안정적이다.** RL 기반 설정의 Graph-R1-7B는 Musique에서 dense 대비 +19.10, HotpotQA에서 +5.12를 기록했다(단 NQ에서는 -3.05로 일반 QA에선 손해). 논문의 표현을 빌리면, agentic search는 구조를 대체하는 게 아니라 **구조가 만들어지는 위치를 재분배**할 뿐이다. 오프라인 그래프 구축이 담당하던 구조의 일부가 추론 시점의 상호작용으로 이동했을 뿐이라는 것이다.

## 결과 3: 비용 — 1M 토큰당 $0부터 $13.19까지

GraphRAG 선택에서 성능만큼 중요한 것이 구축 비용이다. 논문의 Appendix E에 NQ 데이터셋 기준 1M 토큰당 지식 구축 비용이 정리되어 있다.

| 방법 | 구축 시간/1M토큰 | 비용/1M토큰 | 평균 검색시간 | 컨텍스트 길이 |
|------|------|------|------|------|
| LinearRAG | 0.68h | $0 | 1.18s | 4,600 |
| HippoRAG2 | 1.19h | $2.85 | 1.00s | 3,229 |
| HyperGraphRAG | 1.37h | $3.93 | 0.77s | 1,680 |
| RAPTOR | 1.70h | $6.38 | 8.4s | 814 |
| GraphRAG | 1.72h | $13.19 | 1.16s | 22,160 |

두 가지를 눈여겨볼 만하다. Microsoft GraphRAG는 1M 토큰당 $13.19로 가장 비싸고, 컨텍스트도 평균 22,160 토큰으로 가장 길어 추론 비용까지 높인다. 반면 LinearRAG는 관계 추출을 생략한 설계 덕분에 구축 비용이 $0이면서 multi-hop에서 상당한 성능을 낸다. 코퍼스가 커질수록 이 격차는 그대로 배가된다. 100M 토큰 규모라면 GraphRAG 구축에만 $1,319 vs LinearRAG $0다.

## 실전 의사결정: 이 흐름대로 고르면 된다

벤치마크 수치를 실무 기준으로 바꾸면 다음과 같은 의사결정 흐름이 나온다.

```python
def choose_retrieval(queries, corpus_tokens, budget, latency_slo):
    # 1. 질문 유형 분석 (샘플 쿼리로 multi-hop 비율 추정)
    if multi_hop_ratio(queries) < 0.2:
        # 일반 QA 중심: agentic search + dense RAG로 충분
        # (GraphRAG 이득 평균 +0.47, 구축 비용만 추가)
        return "dense_rag + agentic_search"

    # 2. multi-hop이 많더라도 먼저 agentic search를 시도
    if budget.allow_rl_training:
        return "dense_rag + search_r1_style_agent"  # 갭 32%+ 감소

    # 3. 남는 갭이 제품에 치명적일 때만 그래프 구축
    if corpus_tokens > 10_000_000 and budget < 5 * corpus_tokens / 1e6:
        return "linear_rag"       # $0/1M토큰, multi-hop 강세
    if latency_slo < 1.0 and accuracy_is_critical:
        return "hipporag2"        # $2.85/1M토큰, 검색 1.0s
    return "graphrag"             # 최고 정확도, $13.19/1M토큰 + 장컨텍스트
```

구체적인 구현 팁 세 가지:

**팁 1 — agentic search는 쿼리 분해부터.** Search-o1식 단순 온디맨드 검색은 dense RAG에서 성능이 흔들렸다. 반면 쿼리 분해 → 반복 검색 → 증거 검증 구조를 갖춘 GraphSearch는 안정적으로 dense RAG를 끌어올렸다. LangGraph나 에이전트 프레임워크에서 다음처럼 분해 단계를 명시적으로 두는 것이 좋다.

```python
from pydantic import BaseModel

class DecomposedQuery(BaseModel):
    sub_questions: list[str]   # "A의 감독은?" → "그 감독의 스튜디오는?" → "스튜디오 모회사는?"
    evidence_needed: int       # 각 단계에서 몇 개 문서를 검색할지

agent.add_step("decompose", output_type=DecomposedQuery)
agent.add_step("retrieve", retries=3, top_k=5)   # 검색 예산을 명시적으로 제한
agent.add_step("verify_evidence", required=True)  # 증거 검증 없이 답 생성 금지
```

**팁 2 — 비용 산정은 1M 토큰 단위로 미리.** 위 표를 코퍼스 크기에 곱해서 구축 비용을 미리 계산하라. 재색인 주기(문서가 자주 바뀌는가?)까지 고려하면 GraphRAG의 반복 구축 비용은 의외로 빨리 누적된다. 오프라인 비용은 문서가 바뀌지 않는 한 상각(amortize)되므로, 정적 코퍼스 + 복잡 질문 조합에서는 GraphRAG의 투자가 정당화된다.

**팁 3 — 일반 QA에서 GraphRAG를 억지로 쓰지 마라.** 논문의 F-1 평가에서도 일반 QA(NQ, TriviaQA)에서는 GraphRAG가 dense RAG보다 오히려 크게 낮았다(-11.38, -11.87). 그래프 구축이 만능이 아니라, 질문 유형에 따른 손익이 명확히 갈린다는 신호다.

## 결론: "그래프 vs 에이전트"가 아니라 "구조의 위치"

RAGSearch 벤치마크가 주는 교훈을 요약하면:

- **일반 QA(단일 사실 질문)**: agentic search + dense RAG로 충분. GraphRAG 이득은 평균 +0.47에 불과하고 F-1에서는 오히려 손해.
- **multi-hop QA**: single-shot에서는 GraphRAG가 평균 +27.23 압도. agentic search가 이 갭을 상당히(변형 대비 최대 32.3%) 좁히지만, 복잡한 연쇄 추론에서 GraphRAG의 우위와 안정성은 유지됨.
- **비용**: 그래프 구축은 1M 토큰당 $0(LinearRAG)~$13.19(GraphRAG). 정적 대규모 코퍼스 + multi-hop 중심이면 상각되어 Worth it, 변동 코퍼스면 재구축 비용이 주적.
- **에이전트 설계가 절반**: 쿼리 분해와 증거 검증이 없는 무작정 multi-turn은 성능을 오히려 떨어뜨림.
- **첫 단계 제안**: 지금 쓰는 dense RAG에 쿼리 분해 + 반복 검색 + 증거 검증 워크플로우부터 붙여보고, 남은 multi-hop 실패 케이스를 샘플링한 뒤 그 비율이 제품에 치명적인지 보고 나서 그래프 구축을 결정하라.

벤치마크 코드와 평가 스크립트는 [RAGSearch GitHub 리포](https://github.com/FanDongzhe123/RAGSearch)에서, 논문 본문은 [arXiv:2604.09666](https://arxiv.org/abs/2604.09666)에서 확인할 수 있다.

*본 글의 모든 수치는 위 논문(v1, 2026-04-01)의 Table 1, 2, 7, 8 및 본문에서 직접 인용했습니다.*
