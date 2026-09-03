---
title: "음성 AI 에이전트가 답답하게 느껴질 때: 800ms 지연 예산을 분해하는 방법"
date: "2026-09-03"
keywords: ["voice agent", "latency", "TTFA", "TTFS", "음성 AI", "실시간 음성"]
lang: "ko"
description: "사람은 200ms 안에 대화 순서를 넘긴다. 음성 에이전트의 지연시간을 TTFT·TTFS·TTFA로 쪼개고, 검증된 실측 수치로 800ms 예산을 어디에 배분할지 계산하는 실전 가이드."
---

# 음성 AI 에이전트가 답답하게 느껴질 때: 800ms 지연 예산을 분해하는 방법

전화 상담 봇이든 음성 비서든, 사용자가 "로봇 같다"고 느끼는 결정적 원인은 지능이 아니라 지연시간이다. 언어학 연구에 따르면 사람 사이의 대화에서 한 화자가 말을 마치고 상대가 답하기 시작하기까지의 갭은 평균 약 200밀리초다(Turn-taking 관련 연구, Levinson 등). 문제는 음성 에이전트의 전체 파이프라인 — STT, 엔드포인팅, LLM, TTS, 네트워크, 재생 버퍼 — 를 순서대로 실행하면 이 갭을 수 배 초과하기 십상이라는 점이다.

이 글에서는 음성 에이전트의 지연시간을 측정 가능한 단위로 쪼개고, 공개된 실측 수치를 근거로 "800ms 예산"을 어떻게 배분하고 어디를 잘라야 하는지 정리한다.

## 1. TTFT만 보면 안 되는 이유: 세 가지 지표

LLM 벤치마크에서 흔히 쓰는 TTFT(Time To First Token)는 음성 UX의 절반만 설명한다. 첫 토큰이 나와도 사용자가 듣는 것은 오디오이기 때문에, 실제로 측정해야 할 지표는 세 가지다.

| 지표 | 정의 | 측정 시점 |
|------|------|-----------|
| TTFT | LLM이 첫 토큰을 내놓는 시점 | 요청 전송 → 첫 토큰 수신 |
| TTFS | TTS가 첫 문장(절)의 오디오를 만들기 시작하는 시점 | 요청 전송 → TTS 첫 오디오 샘플 생성 |
| TTFA | 사용자 단말에서 첫 소리가 재생되는 시점 | 요청 전송 → 클라이언트 재생 시작 |

LiveKit은 TTFS를 사용자 체감 지표로 제시하고 있다. TTFT가 빨라도 토큰 생성 속도(tokens/sec)가 느리면 첫 문장이 완성되는 시점, 즉 TTFS가 늦어진다. 반대로 TTFT가 다소 느려도 처리량이 높은 모델이 TTFS에서 앞설 수 있다.

측정할 때 주의할 점이 하나 더 있다. TTS API의 스트리밍 응답은 첫 바이트가 WAV 헤더나 Ogg 메타데이터인 경우가 많다. 첫 바이트 도착 시각을 재면 실제보다 수백 ms 빠르게 측정되므로, 반드시 컨테이너 헤더를 건너뛴 "재생 가능한 오디오 샘플"의 도착 시각을 재야 한다.

## 2. 지연 예산 분해: 800ms를 어디에 쓰고 있는가

LiveKit과 Daily가 제시하는 실용적 타깃은 **음성 간(voice-to-voice) 중앙값 700~800ms, p95 1.5초 이내**다. 이 예산을 전형적인 스트리밍 파이프라인(STT → LLM → TTS)으로 구성하면 대략 다음과 같이 배분된다.

| 구성 요소 | 전형적인 지연 범위 | 비고 |
|-----------|------------------|------|
| STT + 엔드포인팅(EOT) | 100~200ms | VAD 임계값 튜닝이 체감을 좌우 |
| LLM 추론(스트리밍, warm) | 300~500ms | 모델·호스팅에 따라 편차 큼 |
| TTS 모델 추론 | 100~200ms | 벤더 수치는 네트워크 제외인 경우 많음 |
| 네트워크 RTT + 재생 버퍼 | 50~150ms + α | 웹 플레이어 버퍼만 200~500ms인 경우도 |

합계가 700ms~1.2초에 도달한다. 즉 어느 한 단계라도 순차 실행으로 직렬화하면 예산 초과다. 핵심은 세 단계를 겹치게(overlap) 만드는 파이프라이닝이다.

모델별 편차도 크다. LiveKit이 공개한 전체 대화 기준 TTFS 측정치를 보면 Gemma 4 31B는 354ms인 반면 GPT-5.5는 1,404ms로 4배 가까운 차이가 난다(LiveKit 측정 공개치 기준). 같은 파이프라인이라도 "어느 LLM을 대화 턴용으로 쓰느냐"가 지연의 최대 변수라는 뜻이다.

## 3. 실측 코드: 5개 시점에 타임스탬프를 박는다

최적화 전에 측정이 먼저다. 아래는 하나의 음성 턴에 대해 TTFT/TTFS/TTFA를 기록하는 최소 구조다.

```python
import time, json

class TurnInstrumenter:
    """음성 턴별 지연 시점을 기록하는 계측기"""
    def __init__(self):
        self.ts = {}

    def mark(self, event: str):
        self.ts[event] = time.perf_counter() * 1000  # ms

    def report(self) -> dict:
        t0 = self.ts.get("request_sent")
        def delta(key):
            return round(self.ts[key] - t0, 1) if key in self.ts else None
        return {
            "ttft": delta("first_llm_token"),      # 요청 → 첫 토큰
            "ttfs": delta("first_clause_tts"),     # 요청 → TTS 첫 오디오
            "ttfa": delta("playback_started"),     # 요청 → 실제 재생
            "stt_final": delta("stt_final"),
            "eot": delta("end_of_turn"),
        }

# 사용 예: 각 콜백에서 mark() 호출
inst = TurnInstrumenter()
inst.mark("request_sent")
# ... STT final 확정 시   -> inst.mark("stt_final")
# ... VAD 엔드포인트 확정  -> inst.mark("end_of_turn")
# ... LLM 첫 토큰 수신 시  -> inst.mark("first_llm_token")
# ... 첫 절 완성+TTS 시작 -> inst.mark("first_clause_tts")
# ... 클라이언트 재생 시작 -> inst.mark("playback_started")
print(json.dumps(inst.report(), ensure_ascii=False))
```

운영에서는 턴마다 이 기록을 저장해 p50/p95/p99를 따로 봐야 한다. 사용자가 기억하는 것은 중앙값이 아니라 꼬리 지연이고, 특히 네트워크 지터나 콜드 스타트가 p95를 만든다.

## 4. 검증된 절감 레버 네 가지

### (1) 컴포넌트 콜로케이션 — 가장 확실한 1순위

STT, LLM, TTS, 미디어 서버가 다른 리전에 흩어져 있으면 왕복 RTT가 예산을 잠식한다. LiveKit은 콜로케이션을 "매우 큰 영향(very high impact)"으로 명시하고 있다. 같은 클라우드 리전(가능하면 같은 AZ)에 배치하는 것만으로 중앙값과 p95가 모두 잡힌다는 게 업계 공통 관측이다.

### (2) 엔드포인팅을 STT 안으로 접기 — 최대 200~600ms

사용자가 말을 끝냈는지 판단하는 엔드포인팅(VAD)을 STT와 분리된 별도 단계로 두면 침묵 임계값 대기 시간이 그대로 더해진다. Deepgram은 엔드오브턴 판단을 인식 과정에 통합한 방식(Flux)으로 에이전트 응답 지연을 200~600ms 줄일 수 있다고 발표했다(Deepgram 발표 기준, 실제 감축폭은 워크로드에 따라 다르다). 단, 조기 EOT 판정은 정확도와 트레이드오프가 있어 오탐률을 모니터링하며 임계값을 조정해야 한다.

### (3) 추론 강도(reasoning effort) 상한 걸기 — 2~3배 차이

실시간 턴에 긴 사고 트레이스를 허용하면 지연이 폭증한다. 공개 벤치마크에서 Gemini 3.1 Flash Live의 TTFA는 추론 강도 최소일 때 0.96초, 최대로 올리면 2.99초로 세 배 가까이 늘어났다. 실시간 턴에는 얕은 응답 + 낮은 max_tokens로 빠르게 답하고, 깊은 작업은 비동기 워크플로로 미루는 구조가 정답이다.

### (4) TTS는 "모델 추론 시간"이 아니라 TTFA로 고르기

Gradium가 공개한 파리 기준 실측(2026년, 스트리밍 API)에 따르면 TTFA 중앙값은 ElevenLabs Turbo v2.5가 304ms, Eleven Flash v2.5가 324ms인 반면 OpenAI TTS-1은 969ms, ElevenLabs Multilingual v2는 706ms다. 같은 벤더 안에서도 모델 선택만으로 2~3배 차이가 난다. 여기에 연결 재수립 오버헤드(턴당 약 50ms)를 없애려면 영구 웹소켓 연결에 세션 멀티플렉싱을 적용하는 것도 유효하다.

## 5. 그래서 Speech-to-Speech인가, 파이프라인인가

2026년 현재 두 접근의 선택 기준은 명확해졌다.

- **단일 speech-to-speech(Realtime) 모델**: 턴테이킹·끼어들기 구현이 단순해지고 지연 구간이 줄어든다. 대화 리듬이 제품 경쟁력인 콜센터, 예약, 실시간 안내에 적합하다. 단, 단계별 관측과 정책 적용(예: PII 마스킹 후 합성)이 어렵다.
- **STT→LLM→TTS 파이프라인**: 툴 호출, 트레이스, 컴플라이언스, 컴포넌트별 교체가 필요한 엔터프라이즈 시나리오에 적합하다. 상태 머신(중간 취소, TTS 재시작)이 복잡해지는 것이 대가다.

OpenAI의 Realtime API는 WebSocket과 WebRTC 양쪽 인터페이스를 공식 지원하며, LiveKit Agents처럼 턴 감지와 인터럽션을 프레임워크 레벨에서 다루는 도구도 성숙해졌다. 어떤 쪽이든 "측정 없이 선택하지 않는다"가 원칙이다.

## 결론

- 음성 에이전트 UX의 기준선은 인간 턴테이킹 갭 약 200ms이고, 현실적 타깃은 voice-to-voice 중앙값 700~800ms, p95 1.5초 이내다.
- TTFT가 아니라 TTFS/TTFA를 종단간으로 측정하라. 첫 바이트가 아니라 "재생 가능한 오디오"의 도착을 재라.
- 절감 우선순위는 콜로케이션 → EOT를 STT에 통합(최대 200~600ms) → 실시간 턴의 추론 강도 상한 → TTFA 기준 TTS 선택 순이다.
- 모델 선택의 영향이 가장 크다. 같은 조건에서 LLM만 바꿔도 TTFS가 354ms~1,404ms까지 갈린다.

당장 할 수 있는 첫 단계는 3장의 계측 코드를 현재 파이프라인에 붙여 턴당 리포트를 모으는 것이다. 예산을 모르면 절감도 없다.

**참고 자료**
- Levinson, "Timing in turn-taking and its implications for processing models of language" (Trends in Cognitive Sciences, 2015)
- LiveKit Agents 문서 및 TTFS 측정 공개치 (docs.livekit.io)
- Gradium, "Time to First Audio: Measuring and Reducing TTS Latency in Voice Agents" (gradium.ai/blog)
- OpenAI Realtime API 공식 문서 — WebSocket/WebRTC 가이드 (developers.openai.com)
- Deepgram 발표 자료 및 AssemblyAI 스트리밍 STT 비교 공개치
