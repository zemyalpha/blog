---
title: "LLM JSON 실패를 0%로: Constrained Decoding 엔진 비교와 실전 적용 가이드"
date: "2026-08-08"
keywords: ["constrained decoding", "structured output", "XGrammar", "Outlines", "llguidance", "JSON Schema"]
lang: "ko"
description: "LLM이 JSON을 생성할 때 발생하는 파싱 실패를 토큰 수준에서 차단하는 constrained decoding 기법을 XGrammar, Outlines, llguidance 세 엔진의 비교와 실전 코드로 정리한다."
---

# LLM JSON 실패를 0%로: Constrained Decoding 엔진 비교와 실전 적용 가이드

LLM을 프로덕션에 넣으면 누구나 같은 벽에 부딪힌다. 모델에게 "JSON으로 답해"라고 했는데 어느 순간 trailing comma가 들어가거나, 마크다운 코드 펜스가 감싸지거나, 필수 필드가 누락된다. 처음엔 2~5% 정도라 감내할 수 있어 보이지만, 이 실패율은 파이프라인 전체로 퍼진다. LLM 호출이 5단계로 직렬 연결된 워크플로우에서 각 단계의 파싱 성공률이 97%라면, 종단 간 성공률은 86%까지 떨어진다. 에이전트가 수십 개의 도구를 순차 호출하는 환경에서는 이야기가 더 심해진다.

이 문제에 대한 근본적 해결책이 **constrained decoding(제약 디코딩)**이다. 출력이 완성된 뒤에 검사하는 것이 아니라, 토큰이 생성되는 매 순간 스키마에 위배되는 토큰을 아예 선택하지 못하게 만드는 기법이다. 이 글에서는 세 가지 주요 오픈소스 엔진—XGrammar, Outlines, llguidance—의 차이를 정리하고, vLLM과 SGLang에서 실제로 어떻게 쓰는지 코드로 보여준다.

## 구조화된 출력이 실패하는 진짜 이유

가장 흔한 대응은 프롬프트에 "반드시 유효한 JSON만 반환하라"고 적는 것이다. 이 방식은 95~98% 정도는 동작한다. 남은 2~5%를 처리하기 위해 개발자들은 단계별 우회책을 쌓아 올린다.

1. **정규식 추출** — 응답에서 JSON 같은 패턴을 찾아 마크다운 펜스를 벗기고 파싱을 시도
2. **수리 휴리스틱** — trailing comma 제거, 누락된 괄호 추가, 홑따옴표를 쌍따옴표로 변환
3. **재시도 루프** — 파싱이 실패하면 모델을 다시 호출. 프롰티어 모델 기준 출력 백만 토큰당 약 $15 비용이 재시도율에 비례해 곱해진다

각 우회책은 하나의 실패 모드를 막아주지만 새로운 엣지 케이스를 만든다. 더 심각한 건 이 "파싱 취약성 세금(parsing fragility tax)"이 모델의 신뢰성에 보이지 않는 천장을 만든다는 점이다. 에이전트가 10개의 순차 작업을 수행해야 하는데 각 단계가 97% 성공이라면, 10단계를 모두 통과할 확률은 74%에 불과하다.

## Constrained Decoding: 토큰 수준의 보장

Constrained decoding은 문제를 발생 후 처리가 아니라 **생성 시점**에 해결한다. 핵심 메커니즘은 단순하다.

1. **제약을 형식 문법으로 정의** — JSON Schema를 문맥 자유 문법(Context-Free Grammar)이나 정규표현식으로 변환
2. **각 디코딩 스텝에서 유효한 토큰 계산** — 현재까지의 출력과 문법 상태를 기준으로, 어휘집(vocabulary) 중 어떤 토큰이 유효한 연속인지 판별
3. **무효 토큰 마스킹** — 샘플링이나 argmax 이전에 무효 토큰의 확률을 0으로 설정
4. **선택된 토큰으로 문법 상태 전이** — 위 과정을 반복

결과는 "거의 항상 유효한" 출력이 아니라, **수학적으로 보장된** 유효한 출력이다. 스키마를 위반하는 시퀀스는 애초에 생성될 수 없다.

성능 이야기가 흥미로운 부분이다. 단순한 구현은 각 스텝마다 어휘집 전체(현대 모델 기준 128K+ 토큰)를 문법과 대조하므로 2~5배의 지연이 발생한다. 하지만 현대 엔진들은 이를 극적으로 개선했고, 심지어 **제약이 없는 생성보다 빠를 수도 있다**. JSON에서는 많은 토큰이 결정론적이기 때문이다. `{"name": "` 이후에는 닫는 따옴표와 콜론이 고정되어 있어, 엔진이 샘플링을 건너뛰고 바로 다음 토큰을 확정한다.

## 세 엔진의 아키텍처 차이

2026년 현재 주요 오픈소스 constrained decoding 엔진은 셋이다. 세 엔진 모두 "JSON Schema를 만족하는 출력만 생성한다"는 목표는 같지만, **언제 토큰 마스크를 계산하느냐**에서 근본적으로 다른 접근을 취한다.

### XGrammar: 사전 계산된 압축 FSM

MLC-AI 팀이 개발했고 MLSys 2025 논문으로 발표되었다. 현재 vLLM과 SGLang의 기본 엔진이다. 핵심 아이디어는 어휘집을 두 부류로 나누는 것이다.

- **문맥 독립 토큰** — 문법 상태와 무관하게 항상 유효하거나 항상 무효인 토큰. 한 번만 계산하면 된다
- **문맥 의존 토큰** — 현재 문법 상태에 따라 유효성이 달라지는 토큰. 스텝마다 검사 필요

대부분의 생성 스텝에서는 극히 일부 토큰만 런타임 검증이 필요하다. 이 분할 전략 덕분에 초기 문법 기반 접근 대비 최대 100배 속도 향상을 달성했으며, 토큰당 오버헤드가 사실상 제로에 가깝다(약 1µs/token 수준).

### Outlines: 즉석 FSM 전이

dottxt-ai 팀이 개발했다. 정규표현식이나 JSON Schema를 유한 상태 기계(FSM)로 컴파일한 뒤, 디코딩 시점에 상태를 전이시키며 마스크를 계산한다. 사전 계산 단계가 짧고 유연성이 높은 것이 장점이지만, 토큰당 약 10~50µs의 오버헤드가 발생한다. 스키마가 자주 바뀌는 실험적 환경에서 쓰기 좋다.

### llguidance: Rust 기반 파서

guidance-ai 프로젝트로, Microsoft 연구진(Eric Horvitz, Harsha Nori, Michał Moskal 등)이 주도하여 개발했다. Rust로 구현되었으며 토큰당 약 50µs의 CPU 시간을 소요한다고 발표되었다. 128K 토큰 어휘집 기준에서도 GPU 추론 지연에 비하면 무시할 수 있는 수준이다. JSON Schema뿐 아니라 Lark 문법, 정규표현식 등 더 넓은 범위의 제약을 지원한다.

> 참고: 위 성능 수치(1µs, 10~50µs, 50µs)는 엔진 개발팀의 발표와 비교 리뷰에서 인용된 값이며, 실제 환경에서는 스키마 복잡도와 모델에 따라 달라질 수 있다. 핵심은 세 엔진 모두 프로덕션에서 실사용 가능한 수준의 오버헤드라는 점이다.

## vLLM에서 구조화된 출력 사용하기

vLLM은 XGrammar를 기본 엔진으로 통합하여, 서버 실행 시 별도 설정 없이도 `guided_json` 파라미터로 구조화된 출력을 지원한다.

```python
from vllm import LLM, SamplingParams
from pydantic import BaseModel

# 출력 스키마 정의
class ProductReview(BaseModel):
    product: str
    rating: int  # 1~5
    summary: str
    pros: list[str]
    cons: list[str]

llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")

# guided_json으로 스키마 주입
sampling = SamplingParams(temperature=0.3, max_tokens=512)
output = llm.chat(
    messages=[{"role": "user", "content": "아이폰 16 Pro를 리뷰해줘."}],
    sampling_params=sampling,
    chat_template_kwargs={"add_generation_prompt": True},
)

# vLLM OpenAI 호환 서버 모드에서는 이렇게:
# POST /v1/chat/completions
# body에 "guided_json": {"type": "object", "properties": {...}}
```

vLLM 서버를 OpenAI 호환 API로 띄운 경우, 요청 본문에 `guided_json` 필드를 추가하면 된다. 응답은 항상 스키마를 만족하는 JSON이다. temperature가 0이 아니더라도 구조적 유효성은 보장된다.

## Outlines로 로컬 모델에 제약 걸기

vLLM 대신 개별 모델을 직접 로드하는 환경에서는 Outlines가 유용하다. Hugging Face 모델을 래핑하여 constrained decoding을 적용한다.

```python
import outlines

# Pydantic 모델로 스키마 정의
@outlines.generate.json(model=your_model)
class UserIntent(BaseModel):
    intent: Literal["question", "complaint", "request", "feedback"]
    confidence: float
    extracted_entities: list[str]

# 정규표현식 제약도 가능
@outlines.generate.regex(model=your_model, regex=r"\d{3}-\d{4}-\d{4}")
def generate_phone(prompt):
    pass

# 또는 선택지 제약
@outlines.generate.choice(model=your_model, choices=["긍정", "부정", "중립"])
def classify_sentiment(prompt):
    pass
```

Outlines는 정규표현식, JSON Schema, 선택지(choice), Lark 문법을 모두 지원하므로, 단순 JSON을 넘어 도메인 특화 언어(DSL) 생성에도 쓸 수 있다.

## 실전에서 주의할 점

### 스키마가 크면 품질이 떨어진다

가장 직관에 반하는 발견이다. constrained decoding은 출력이 "구조적으로 유효"하도록 보장하지만, **의미적으로 좋은** 출력을 보장하지는 않는다. 필드가 많아지는 복잡한 JSON Schema에서는 모델이 제약을 만족하느라 실제 작업(의미 있는 값 생성)에 할당할 확률 질량이 부족해진다. 결과적으로 필드 값이 비어 있거나, 서로 모순되거나, 의미 없는 반복이 나올 수 있다. 일부 실무 리뷰에서는 스키마 필드가 수십 개를 넘어갈 때 품질 저하가 관찰된다고 보고하나, 정확한 임계값은 모델과 작업에 따라 달라진다.

실용적인 권장사항은 다음과 같다.

- **스키마를 작게 유지** — 한 번의 호출에 30개 미만의 필드만
- **큰 구조는 분할** — 50개 필드짜리 객체를 5개의 10필드 객체로 나눠 순차 호출
- **필수 필드만 제약** — `additionalProperties: false`는 보장하되, 선택 필드는 느슨하게

### 빈 문자열과 최소 길이

`"summary": ""`처럼 빈 문자열은 JSON Schema를 만족하지만 쓸모가 없다. `minLength` 제약을 걸면 되지만, 일부 엔진은 이를 완벽히 지원하지 않는다. 회피책으로 프롬프트에서 "최소 20자 이상으로 작성"을 명시하고, 후처리로 빈 값이나 지나치게 짧은 값을 재생성하는 방식을 쓸 수 있다.

### 재시도를 완전히 없앨 수는 없다

구조적 유효성은 100% 보장되지만, **의미적** 실패(잘못된 분류, 환각된 엔티티 등)는 여전히 발생한다. 따라서 구조 검증용 try/catch는 제거해도, 비즈니스 로직 검증(예: rating이 1~5 범위인지, product가 실제 존재하는 제품인지)은 별도로 둬야 한다.

## OpenAI, Claude와 비교

주요 API 제공자도 각자의 방식으로 구조화된 출력을 지원한다.

- **OpenAI** — `response_format: { type: "json_schema", json_schema: {...} }`로 strict mode를 제공. 내부적으로 constrained decoding을 적용하며, 지원하는 JSON Schema 키워드에 제한이 있다(`allOf`, `$ref` 등 일부 미지원)
- **Anthropic** — Claude의 tool use 기능을 활용해 구조화된 출력을 얻을 수 있다. 도구의 `input_schema`를 정의하면 모델이 그에 맞춰 `tool_use` 블록을 반환
- **Google Gemini** — `response_schema` 파라미터로 JSON Schema를 전달

셋 모두 "출력이 스키마를 만족한다"는 보장을 제공하지만, 호환되는 JSON Schema 키워드의 범위와 성능 특성이 다르다. 자체 호스팅(vLLM/SGLang + XGrammar)은 가장 완전한 JSON Schema 지원과 제어를 제공하지만, 인프라 관리 부담이 따른다.

## 결론

Constrained decoding은 LLM 시스템에서 "가끔 JSON이 깨지는 문제"를 단순히 줄이는 것이 아니라 **제거**하는 기술이다. 핵심을 정리하면:

- **JSON 파싱 실패는 constrained decoding으로 토큰 수준에서 원천 차단 가능** — 구조적 유효성이 수학적으로 보장된다
- **XGrammar, Outlines, llguidance 세 엔진** — 사전 계산(XGrammar), 즉석 FSM(Outlines), Rust 기반 파서(llguidance)로 각기 다른 성능·유연성 트레이드오프를 제공한다. vLLM/SGLang은 기본적으로 XGrammar를 내장
- **스키마가 커지면 품질 저하 주의** — 정확한 임계값은 없지만, 필드가 수십 개를 넘어가면 모델의 의미적 출력 품질이 떨어질 수 있다. 큰 구조는 여러 호출로 분할하는 것이 안전하다
- **의미적 검증은 별도로 필요** — 구조가 맞아도 내용이 틀릴 수 있으므로 비즈니스 로직 검증은 유지
- **OpenAI/Claude/Gemini도 지원하지만 JSON Schema 호환성에 차이** — 복잡한 스키마가 필요하면 자체 호스팅 vLLM이 가장 유연

당장 시도할 수 있는 첫 단계는 기존 LLM 호출에서 가장 파싱 실패율이 높은 엔드포인트 하나를 골라, vLLM의 `guided_json` 또는 OpenAI의 strict mode로 전환해 보는 것이다. 재시도 루프와 정규식 수리 코드를 지우는 것만으로도 코드베이스가 눈에 띄게 단순해질 것이다.
