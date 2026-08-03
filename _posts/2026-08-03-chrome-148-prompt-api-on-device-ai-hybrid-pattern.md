---
title: "Chrome 148 Prompt API가 안정화됐다: 온디바이스 AI의 하이브리드 패턴과 표준 논쟁"
date: "2026-08-03"
keywords: ["Chrome Built-in AI", "Prompt API", "Gemini Nano", "온디바이스 AI", "프로그레시브 인헨스먼트", "하이브리드 AI"]
lang: "ko"
description: "Chrome 148에서 stable로 출시된 Prompt API와 Gemini Nano를 활용한 온디바이스 AI 패턴, 하이브리드 폴백 설계, 그리고 Mozilla·Apple·W3C의 반대가 부각시킨 웹 표준의 딜레마를 정리한다."
---

# Chrome 148 Prompt API가 안정화됐다: 온디바이스 AI의 하이브리드 패턴과 표준 논쟁

웹 페이지에서 LLM을 쓰려면, 지금까지는 두 가지 길밖에 없었다. 첫째는 자체 모델을 WebGPU로 브라우저에 올리는 것이고, 둘째는 모든 요청을 OpenAI나 Anthropic 같은 클라우드 API로 보내는 것이다. 전자는 모델 다운로드가 무겁고 후자는 토큰 비용과 프라이버시가 문제였다.

2026년 5월 5일, Chrome 148이 세 번째 길을 열었다. 브라우저 자체에 내장된 Gemini Nano 모델을 JavaScript로 직접 호출하는 **Prompt API**가 stable 채널에 출시된 것이다. API 키도, 토큰 과금도, 서버 왕복도 없다. 모델은 사용자 기기에서 실행되고 추론 결과만 돌려받는다.

하지만 이 출시는 화려한 기능 소개로만 끝나지 않았다. Mozilla, Apple의 WebKit 팀, W3C 기술 아키텍처 그룹(TAG), 그리고 Microsoft까지 정식으로 반대 의견을 냈음에도 Google이 출시를 강행했기 때문이다. 이 글에서는 Prompt API가 무엇을 바꾸는지, 프로덕션에서 어떤 패턴으로 써야 하는지, 그리고 왜 표준 커뮤니티가 반대했는지를 정리한다.

## Chrome Built-in AI: API 제품군 전체 구조

Prompt API는 단일 인터페이스가 아니라 **Chrome Built-in AI**라는 제품군의 일부다. Gemini Nano(파운데이션 모델)와 Gemma 197M(경량 전문가 모델)을 기반으로, 브라우저가 모델을 다운로드·관리하며 개발자는 API만 호출한다.

현재 상태를 정리하면 다음과 같다:

| API | 역할 | 상태 (Chrome 148 기준) |
|-----|------|----------------------|
| **Prompt API** | 범용 자연어 추론, 멀티모달 입력 | Stable (웹 페이지 + 확장 프로그램) |
| **Summarizer API** | 텍스트 요약 (키 포인트, 단락, 불릿 등) | Stable |
| **Translator API** | 언어 간 번역 (36개 언어) | Stable |
| **Language Detector API** | 입력 텍스트 언어 식별 + 신뢰도 점수 | Stable |
| **Writer API** | 지정된 작문 과제로 새 텍스트 생성 | Origin Trial |
| **Rewriter API** | 기존 텍스트의 길이·어조 수정 | Origin Trial |
| **Proofreader API** | 문법·맞춤법 교정, 오류 유형 라벨 | Origin Trial |

Google이 단일 Prompt API만 제공하지 않고 작업 특화 API를 분리한 이유는 명확하다. 범용 프롬프트에 맡기면 팀마다 시스템 프롬프트를 다르게 써서 결과가 들쭉날쭉해진다. 반면 `Summarizer.create({ type: 'key-points', length: 'short', format: 'markdown' })`처럼 타입된 파라미터를 쓰면 사이트 간 결과 품질이 균일해진다.

핵심 차이는 **모델 관리 주체**다. Transformers.js나 WebGPU 기반 접근이 개발자가 직접 모델을 호스팅한다면, Built-in AI는 브라우저가 모델을 제공한다. Gemini Nano는 디스크상 약 4GB(출처에 따라 2.7~4.27GB로 보고)를 차지하며, Chrome이 사용자 기기에서 자동으로 다운로드·업데이트·삭제를 관리한다.

## 하드웨어 요구사항과 가용성 감지

Built-in AI는 모든 기기에서 동작하지 않는다. Chrome 공식 문서가 명시한 최소 요구사항은 다음과 같다:

- **운영체제**: Windows 10/11, macOS 13(Ventura)+, Linux, ChromeOS (Chromebook Plus)
- **저장공간**: Chrome 프로필이 있는 볼륨에 22GB 이상 여유
- **GPU**: VRAM 4GB 초과, 또는
- **CPU**: RAM 16GB 이상 + 코어 4개 이상
- **오디오 입력**: GPU 필수
- **모바일**: Android·iOS 미지원

이 조건을 만족하지 못하면 API가 존재하지 않거나 `unavailable`을 반환한다. 따라서 **가용성 감지 없이 API를 호출하는 코드는 프로덕션에서 절대 안 된다.**

```javascript
// 1단계: API 존재 여부 확인 (Firefox, Safari, 구버전 Chrome)
if (!('LanguageModel' in self)) {
  return serverSideFallback();
}

// 2단계: 모델 가용성 확인
const availability = await LanguageModel.availability({
  expectedInputs: [{ type: 'text', languages: ['en'] }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});

// 반환값: 'available' | 'downloadable' | 'downloading' | 'unavailable'
if (availability === 'unavailable') {
  return serverSideFallback();
}

// 3단계: 모델이 아직 다운로드되지 않았으면 사용자 제스처 필요
if (availability === 'downloadable' && !navigator.userActivation.isActive) {
  // 페이지 로드 직후 자동 다운로드는 실패한다
  // 사용자가 클릭한 뒤에 create()를 호출해야 한다
  showEnableButton();
  return;
}
```

`availability()`가 반환하는 네 가지 상태는 UX 설계에 직접 영향을 미친다. `available`은 즉시 세션 생성이 가능하고, `downloadable`은 수 GB 다운로드가 필요하므로 사용자에게 진행 상황을 보여줘야 한다. 핵심은 `downloadable` 상태에서 사용자 활성화(user activation) 없이 `create()`를 호출하면 `NotAllowedError`가 발생한다는 점이다.

## 세션 수명주기와 하이브리드 폴백 패턴

Prompt API의 핵심 인터페이스는 `LanguageModel`이며, 세션 기반으로 동작한다. 세션은 컨텍스트 윈도우를 유지하고 이전 대화를 기억한다.

```javascript
// 세션 생성 (시스템 프롬프트 + 다운로드 진행률 모니터링)
const session = await LanguageModel.create({
  initialPrompts: [
    { role: 'system', content: '사용자 입력에서 여행 의도를 추출한다. 제공된 JSON 스키마에 맞춰 응답한다.' }
  ],
  monitor(m) {
    m.addEventListener('downloadprogress', (e) => {
      updateProgressBar(Math.floor(e.loaded * 100));
    });
  },
});

// 구조화된 출력: JSON Schema로 응답 형식 강제
const schema = {
  type: 'object',
  properties: {
    destination: { type: 'string' },
    budget: { type: 'number' },
    nights: { type: 'number' },
  },
  required: ['destination'],
};

const raw = await session.prompt(
  '프라하에서 로맨틱한 주말, 60만 원 이하로 찾아줘',
  { responseConstraint: schema }
);
const intent = JSON.parse(raw);
// → { destination: "Prague", budget: 600000, nights: 2 }

session.destroy(); // GPU 메모리 해제
```

`responseConstraint`로 JSON Schema를 전달하면 모델 출력이 스키마에 맞게 보장된다. 샘플링 단계에서 제약이 적용되므로 `JSON.parse`가 실패할 확률이 극적으로 낮아진다. 이는 "JSON 형식으로 답해"라는 프롬프트에 의존하는 클라우드 API와 비교했을 때 실질적인 신뢰성 차이를 만든다.

**프로덕션 패턴은 하이브리드여야 한다.** 온디바이스 모델이 항상 사용 가능하다고 가정하면 안 된다. Trip.com이 Google I/O 2026에서 발표한 접근 방식이 정석이다: `LanguageModel.availability()`가 `available`을 반환하면 로컬에서 처리하고, 그렇지 않으면 기존 서버 API로 폴백한다. 모든 추론 비용을 사용자 기기로 옮기면서도 호환성을 유지하는 구조다.

| 항목 | Gemini Nano (온디바이스) | 클라우드 API |
|------|--------------------------|-------------|
| 요청당 비용 | 0원 | 토큰 과금 |
| 첫 토큰 지연 | 200ms 이내 | 600~2000ms (네트워크 포함) |
| 데이터 프라이버시 | 기기 내 처리 | 서버 전송 |
| 출력 품질 | 어휘 작업에 적합 | 복잡한 추론까지 가능 |
| 기기 커버리지 | Chrome 148+ 데스크톱 한정 | 모든 브라우저 |
| 오프라인 동작 | 첫 다운로드 후 가능 | 불가능 |

## 왜 Mozilla, Apple, W3C가 반대했는가

Prompt API의 기술적 장점에도 불구하고, 이 기능은 웹 표준 커뮤니티에서 전례 없는 반대에 직면했다. Mozilla 개발자 관계 총괄 Jake Archibald는 2026년 5월 6일 "Mozilla: 반대. WebKit: 반대. Microsoft: 여러 우려. W3C TAG: 여러 우려. 개발자: 대체로 부정적. Chrome: 그럼에도 출시. 웹 표준의 슬픈 시대"라고 요약했다.

네 가지 핵심 반대 논리가 있다.

**첫째, 상호운용성 문제.** 전통적 웹 API는 정밀한 명세 덕분에 Chrome, Firefox, Safari에서 동일하게 동작한다. 그러나 AI 모델 출력은 확률적이고 모델 종속적이다. Gemini Nano에 최적화된 프롬프트는 다른 브라우저의 다른 모델에서 엉뚱한 결과를 낼 수 있다. Mozilla는 Prompt API 저장소가 "언어 모델 품질, 안정성, 브라우저 간 상호운용성을 보장하지 않는다"고 명시한 점을 비판했다. 2000년대 "Internet Explorer에 최적화" 사이트가再现될 위험이라는 지적이다.

**둘째, 사용자 동의 없는 자원 소비.** Apple WebKit 팀이 지적한 구조적 결함이다. Gemini Nano가 설치되면 최상위 웹 페이지가 사용자 배터리, CPU, GPU 자원을 소비하면서 AI 추론을 실행할 수 있다. 사이트는 이점을 얻고 비용은 사용자가 부담하는 비대칭 구조다. 권한 대화상자도 없다.

**셋째, 프롬프트 인젝션 취약성.** 시스템 명령어와 렌더링된 페이지 콘텐츠 사이에 권한 경계가 없다. 사용자 생성 콘텐츠(댓글, 포럼 글, 지원 티켓)를 요약하는 사이트는, 악의적 텍스트를 게시하는 사용자가 요약 결과를 조작할 수 있다. 2026년 5월 DEV Community에 게시된 보안 분석에서 Gemini Nano가 프롬프트 인젝션에 매우 취약하다고 보고되었다.

**넷째, 콘텐츠 정책 결합.** Chrome 문서는 Prompt API 접근 전 Google의 생성형 AI 사용 금지 정책(Gen AI Prohibited Uses Policy) 준수를 요구한다. 이 정책은 단순히 위법 행위를 넘어 Google이 "불쾌하다"고 판단하는 콘텐츠 생성까지 금지한다. 웹 플랫폼 API가 특정 기업의 사용 정책에 종속되는 전례를 만들었다는 비판이 있다.

흥미로운 점은 Chrome 엔지니어 Rick Byers가 이 API의 출시 승인자 중 한 명이면서도 "Mozilla의 표준 입장에 동의한다"고 공개적으로 인정한 것이다. 그는 그럼에도 실험의 이점이 기다림의 비용보다 크다고 주장했다. 출시 후 증거를 수집하겠다는 입장이다.

## 모델 품질의 한계

Gemini Nano는 프론티어 모델이 아니다. 어휘적 작업(분류, 파싱, 단문 요약, 번역)에는 적합하지만, 복잡한 다단계 추론이나 긴 문서 처리에는 한계가 있다.

The Register가 2026년 2월 보도한 독립 벤치마크에 따르면, Chrome의 Gemini Nano 구현은 생성 작업의 약 15.17%에서 완료에 실패했고, 분류 응답의 약 23.93%가 오답이었다고 한다. 같은 벤치마크에서 환각률은 Chrome 6%, Edge(Phi-4 mini) 17%로 Chrome이 유일하게 앞섰다. 다만 이 수치는 단일 벤치마크 결과이므로 "보도된 바에 따르면" 정도로 참고해야 한다.

실용적 교훈은 명확하다. **Gemini Nano를 범용 LLM 대체재로 쓰지 마라.** 스마트 자동완성 수준의 작업에 쓰고, 복잡한 추론은 클라우드 모델에 맡겨라. 하이브리드 아키텍처에서 온디바이스가 담당하는 영역은 가벼운 파싱·분류·요약이고, 클라우드가 담당하는 영역은 심층 추론이다. 보도에 따르면 이 조합으로 API 비용을 70~80% 줄일 수 있다고 한다.

## 결론: progressive enhancement로 접근하라

Chrome Built-in AI는 웹 개발의 비용 구조를 바꾸는 잠재력이 있다. API 키 관리, 토큰 예산, 프라이버시 컴플라이언스가 사라지는 영역이 생기기 때문이다. 하지만 현시점에서 크로스 브라우저 표준이 아니다.

실천적 권고사항을 정리한다:

- **가용성 감지를 먼저, 그리고 항상.** `'LanguageModel' in self`와 `availability()`로 체크하고, 실패 시 서버 API로 폴백하라. Firefox와 Safari 사용자는 API 자체가 없다.
- **신뢰할 수 없는 입력을 그대로 모델에 넣지 마라.** 사용자 생성 콘텐츠를 처리할 때는 프롬프트 인젝션을 전제로 출력도 신뢰할 수 없는 데이터로 취급하라.
- **세션을 꼭 destroy()하라.** 온디바이스 세션은 GPU 메모리를 점유한다. 작업이 끝나면 해제하지 않으면 사용자 기기에 부담을 준다.
- **작업 특화 API를 우선하라.** 요약이면 Summarizer, 번역이면 Translator가 범용 Prompt API보다 빠르고 일관적이다.
- **첫 다운로드 UX를 설계하라.** `downloadable` 상태의 사용자에게 진행 바를 보여주고, 데이터 제한 연결에서는 와이파이 대기를 안내하라.
- **프롬프트를 Gemini Nano에 과도하게 최적화하지 마라.** 다른 브라우저에서 다른 모델이 들어올 때 호환성 문제가 된다.

Chrome이 전 세계 브라우저 점유율의 약 65~68%를 차지하는 상황에서, Built-in AI가 사실상 웹 AI 계층의 기본이 될 가능성은 있다. 그러나 그것이 웹 표준으로 정착될지, 아니면 또 하나의 벤더 종속이 될지는 Mozilla와 Apple이 언제 대안을 내놓느냐, 그리고 개발자가 어느 정도까지 Chrome 전용 최적화를 수용하느냐에 달려 있다. 지금은 실험하고 데이터를 모으되, 폴백 없이 의존하기에는 이른 단계다.
