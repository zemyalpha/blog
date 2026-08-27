---
title: "영어 1위 모델이 한국어에서는 진다: 2026년 오픈소스 STT 실전 선택 가이드"
date: "2026-08-27"
keywords: ["오픈소스 STT", "음성인식", "Qwen3-ASR", "Whisper", "한국어 음성인식", "Parakeet TDT", "ASR 벤치마크"]
lang: "ko"
description: "2026년 오픈소스 STT 지형을 리더보드와 한국어 실측 데이터로 분석한다. 영어 정확도 1위 Canary-Qwen과 한국어 실전 최강 Qwen3-ASR, 용도별 모델 선택 기준과 faster-whisper, vLLM 실행 코드까지."
---

# 영어 1위 모델이 한국어에서는 진다: 2026년 오픈소스 STT 실전 선택 가이드

음성 AI가 흔해진 만큼 "STT는 그냥 Whisper 쓰면 되는 거 아니냐"는 말을 자주 듣는다. 1년 전이라면 맞는 말이었다. 당시 오픈소스 음성인식 선택지는 사실상 Whisper와 Vosk 둘로 압축됐으니까. 그런데 2026년의 지형은 완전히 다르다. NVIDIA가 분기마다 새 모델을 내놓고, Alibaba의 Qwen이 ASR 경쟁에 뛰어들었으며, 폰과 마이크로컨트롤러용 초경량 모델까지 쏟아졌다.

문제는 선택지가 많아질수록 "무엇을 골라야 하는가"가 어려워진다는 점이다. 더 큰 문제는 영어 벤치마크 순위가 여러분의 서비스 품질을 보장하지 않는다는 사실이다. 이 글에서는 2026년 8월 기준 공개 리더보드 데이터와, 한국어 실제 상담 데이터 1만 건에 대한 실측 결과를 함께 놓고 본다. 어떤 모델을 고를지 결정하는 데 필요한 정보는 전부 여기 있다.

## 2026년 오픈소스 STT 지형: 리더보드가 말해주는 것

먼저 영어 중심의 Hugging Face Open ASR Leaderboard와 각사 공개 자료 기준으로 주요 모델을 정리하면 다음과 같다.

| 모델 | WER (%) | RTFx | 파라미터 | 언어 | 라이선스 |
|------|---------|------|----------|------|----------|
| Canary-Qwen 2.5B (NVIDIA) | 5.63 | 418 | 2.5B | 영어 | CC-BY-4.0 |
| Granite Speech 3.3 8B (IBM) | 5.85 | 미공개 | 약 9B | 영어+유럽어 | Apache 2.0 |
| Qwen3-ASR-1.7B (Alibaba) | 5.76 | 148 | 1.7B | 52개 언어+방언 | Apache 2.0 |
| Whisper large-v3 (OpenAI) | 7.4 | 런타임별 상이 | 약 1.55B | 99개 | MIT |
| Whisper large-v3-turbo | 7.75 | 216 | 809M | 99개 | MIT |
| Parakeet TDT 1.1B (NVIDIA) | 약 8.0 | 2,000+ | 1.1B | 영어 | CC-BY-4.0 |

수치 출처: Northflank 벤치마크 가이드(2026년 1월) 및 Hugging Face 모델카드 공식 eval 결과. WER은 낮을수록 좋고, RTFx는 1초 연산으로 몇 초의 오디오를 처리하는지를 나타내므로 높을수록 빠르다.

여기서 눈에 띄는 세 가지:

**첫째, 정확도 1위와 속도 1위가 갈린다.** Canary-Qwen 2.5B가 영어 정확도(5.63%)를 주도하지만, 처리량은 Parakeet TDT가 압도한다. RTFx 2,000 이상은 1초의 연산으로 30분 넘는 오디오를 처리한다는 뜻이다. 배치 전사 파이프라인이라면 정확도보다 처리량이 비용을 결정한다.

**둘째, LLM 결합형 ASR이 대세가 됐다.** Canary-Qwen은 FastConformer 인코더에 Qwen3-1.7B 디코더를 결합한 SALM(Speech-Augmented Language Model) 구조다. Qwen3-ASR 역시 Qwen3-Omni의 오디오 이해 능력 위에 만들어졌다. 순수 ASR 인코더-디코더 시대에서 "음성을 입력받는 언어모델" 시대로 넘어온 것이다. 이 구조 덕분에 구두점, 대소문자, 숫자 표기 품질이 이전 세대보다 눈에 띄게 좋아졌다.

**셋째, Whisper는 여전히 만능 도구지만 최정상은 아니다.** 99개 언어 지원과 생태계의 규모는 여전히 최강이지만, 영어 정확도만 보면 상위권 모델보다 1.5~2%p 뒤처져 있다.

## 리더보드가 말해주지 않는 것: 한국어 실전 성능

여기부터가 이 글의 핵심이다. 위 표에서 한국어를 제대로 지원하는 모델은 Whisper 계열과 Qwen3-ASR 정도다. Canary와 Parakeet은 영어 전용이고, Granite Speech도 한국어는 지원하지 않는다. 그렇다면 Whisper와 Qwen3-ASR 중 한국어에서는 누가 이기는가.

국내 한 엔지니어가 실제 콜센터 상담 녹취 약 1만 건으로 베이스(파인튜닝 없는) 모델끼리 문자 오류율(CER)을 비교한 결과가 올해 2월 공개됐다. 요약하면 이렇다.

| 모델 | 파라미터 | CER (베이스) | CER (파인튜닝 후) |
|------|----------|--------------|---------------------|
| Qwen3-ASR-1.7B | 1.7B | 22.72% | 7.41% (풀 파인튜닝) |
| Qwen3-ASR-0.6B | 0.6B | 26.49% | - |
| faster-whisper-large-v3-turbo | 809M | 27.70% | 11.53% (LoRA) |

출처: MZ.\_.GPT 블로그 실측 리뷰(2026년 2월, 링크는 글 말미 참조). 전화망 특유의 노이즈와 끼어들기(overlap)가 포함된 실데이터 기반이라 실서비스 환경에 가깝다는 점이 값지다.

결론부터 말하면 **한국어 실전에서는 Qwen3-ASR이 Whisper 계열을 이겼다.** 흥미로운 점이 두 가지 있다.

첫째, 파라미터 0.6B짜리 Qwen3-ASR조차 large-v3-turbo보다 낮은 CER을 기록했다. 모델 크기가 아니라 학습 데이터의 언어 구성과 아키텍처가 한국어 성능을 좌우한다는 방증이다. Qwen3-ASR은 공식적으로 30개 언어에 한국어를 포함하고 있고, 노래·배경음 포함 음성까지 학습했다.

둘째, Whisper의 CER 상승 원인 상당 부분이 **할루시네이션**이었다. 무음 구간이나 잡음이 싹한 구간에서 "감사합니다", "Thank you" 같은 음성 내용과 무관한 단어를 반복 출력하는 현상이 다수 관찰됐다는 것. 이건 특정 실측에서만 나타나는 게 아니라 영어 리뷰에서도 지적되는 Whisper 계열의 구조적 약점이다. 무음 오디오에서 없는 자막을 지어내는 문제는 여전히 해결되지 않았다.

이 실측에서 파인튜닝 효과도 확인된다. Qwen3-ASR-1.7B를 도메인 데이터로 풀 파인튜닝하자 CER이 22.72%에서 7.41%로, 3배 이상 개선됐다. Whisper 계열(LoRA)도 27.70%에서 11.53%로 좋아졌지만 최종 성능은 Qwen3-ASR이 우위였다. 도메인 특화 음성(상담, 의료, 법률 등)을 다룬다면 "어떤 베이스 모델을 고르느냐"와 "얼마나 잘 파인튜닝하느냐"가 함께 성능을 결정한다.

## 용도별 선택 가이드와 실행 코드

벤치마크 숫자보다 실용적인 건 "내 상황에서 뭘 쓰나"다. 2026년 8월 기준으로 정리하면 다음과 같다.

- **다국어 배치 전사 (한국어 포함)**: faster-whisper + large-v3 또는 turbo. 생태계가 가장 크고 전처리·후처리 도구가 풍부하다.
- **한국어 정확도가 중요한 서비스**: Qwen3-ASR-1.7B. 도메인 데이터가 있다면 파인튜닝까지.
- **대량 아카이브 처리 (영어)**: Parakeet TDT. RTFx 2,000+는 배치 비용 구조를 바꿔놓는다.
- **실시간 스트리밍 (영어)**: Parakeet TDT 또는 Moonshine. Whisper는 기본 스트리밍을 지원하지 않는다.
- **엣지·모바일 온디바이스**: Moonshine. 라즈베리파이급에서 돌아가도록 설계됐다.

### faster-whisper로 배치 전사하기

Whisper를 실서비스에 쓴다면 대부분 CTranslate2로 재구현한 faster-whisper를 쓴다. 공식 Whisper 대비 최대 4배 빠르고 메모리도 적게 쓴다.

```python
from faster_whisper import WhisperModel

# GPU: float16, CPU: int8 권장
model = WhisperModel("large-v3-turbo", device="cuda", compute_type="float16")

segments, info = model.transcribe(
    "meeting.wav",
    language="ko",
    vad_filter=True,        # 무음 구간 잘라내기 — 할루시네이션 억제의 첫걸음
    beam_size=5,
)

for seg in segments:
    print(f"[{seg.start:.1f}s -> {seg.end:.1f}s] {seg.text}")
```

`vad_filter=True` 하나로 무음 구간을 사전에 제거하면 무음 할루시네이션을 상당히 줄일 수 있다. 한국어라면 `language="ko"`를 명시하는 편이 언어 자동 감지 오류를 막는다.

### Qwen3-ASR을 vLLM으로 서빙하기

Qwen3-ASR의 실전 강점은 vLLM 기반 서빙이다. 공식 리포지토리는 배치 추론, 비동기 서빙, 스트리밍, 타임스탬프 예측을 지원하는 추론 프레임워크를 함께 제공한다. 공개 자료에 따르면 0.6B 모델은 동시성 128 조건에서 약 2,000배의 처리량을 낸다.

```python
# pip install qwen-asr (공식 패키지)
import torch
from qwen_asr import Qwen3ASRModel

model = Qwen3ASRModel.from_pretrained(
    "Qwen/Qwen3-ASR-1.7B",
    dtype=torch.bfloat16,
    device_map="cuda:0",
    max_new_tokens=256,  # 긴 오디오면 더 크게
)

results = model.transcribe(
    audio="consultation.wav",
    language="Korean",  # None이면 언어 자동 감지
)
print(results[0].language, results[0].text)
```

2026년 6월 말부터는 네이티브 Transformers 지원(`Qwen3-ASR-1.7B-hf`)도 추가돼서 torch.compile 가속 추론이 가능해졌다. Hugging Face 모델카드 기준으로 다운로드는 누적 1,300만 회를 넘겼고, GitHub 스타가 3,400개를 돌파한 만큼 커뮤니티 검증도 충분히 쌓인 상태다.

덤으로 Qwen3-ForcedAligner-0.6B라는 비자기회귀(forced-alignment) 모델도 같이 공개됐다. 음성-텍스트 쌍을 최대 5분 길이까지 문자 단위 정렬해주는데, 자막 싱크나 오디오북 제작에서 쓸모가 크다.

## 알아두면 좋은 함정 세 가지

**단종 모델을 새 프로젝트에 쓰지 마라.** 구형 추천 글에 아직 Mozilla DeepSpeech와 Coqui STT가 남아 있다. 둘 다 유지보수가 중단됐다. 돌아는 가지만 2026년에 새로 쌓는 의존성으로는 실수다. SpeechRecognition 파이썬 라이브러리도 주의하자. 이건 모델이 아니라 클라우드 API를 감싸는 래퍼라서, 프로토타입용이지 운영 의존성이 아니다.

**리더보드 WER은 깨끗한 오디오 기준이 많다.** LibriSpeech 같은 깨끗한 음성 기반 수치는 실전 콜센터·회의록 환경과 괴리가 크다. AMI(회의)나 Earnings22(어낼리스트 콜)처럼 잡음 있는 데이터의 WER은 리더보드 평균보다 훨씬 높게 나온다. Qwen3-ASR-1.7B도 깨끗한 LibriSpeech Clean에서는 1.63%지만 AMI에서는 10.56%다. 도입 전에는 반드시 자기 데이터로 소규모 실측을 돌려보자.

**오픈소스의 진짜 비용은 라이선스가 아니라 운영이다.** GPU 프로비저닝, 오토스케일링, 지연시간 관리, 모델 업데이트를 직접 책임져야 한다. 하루 몇 시간짜리 오디오라면 셀프호스팅이 저렴하지만, 트래픽이 불규칙하게 몰리는 실시간 서비스라면 관리형 API와 비용을 비교해볼 가치가 있다.

## 결론: 상황별 정답 요약

- 영어 리더보드 1위는 **Canary-Qwen 2.5B**(WER 5.63%)지만 영어 전용이다. 다국자·한국어 서비스에 무조건 적용하면 실패한다.
- **한국어 실전에서는 Qwen3-ASR이 유력하다.** 실측 데이터에서 베이스부터 Whisper를 앞섰고, 파인튜닝 시 CER 7.41%까지 도달했다. vLLM 서빙 지원도 실무적 메리트다.
- **Whisper는 여전히 기본값**이다. 특히 다국어 혼재 오디오와 풍부한 생태계가 필요할 때. 단 무음 할루시네이션에는 VAD 필터로 대응하라.
- **영어 대량 배치는 Parakeet TDT**, **엣지는 Moonshine**이 각각 최선이다.
- 어떤 모델이든 도입 전 **자기 도메인 데이터 100건으로 CER/WER 실측**을 먼저 돌려라. 그 100건이 리더보드 100위보다 정확한 답을 준다.

---

**참고 자료**
- Northflank, "Best open source speech-to-text (STT) model in 2026 (with benchmarks)" — https://northflank.com/blog/best-open-source-speech-to-text-stt-model-in-2026-benchmarks
- AssemblyAI, "Top 8 open source STT options for voice applications in 2026" — https://www.assemblyai.com/blog/top-open-source-stt-options-for-voice-applications
- QwenLM/Qwen3-ASR 공식 리포지토리 — https://github.com/QwenLM/Qwen3-ASR
- Qwen/Qwen3-ASR-1.7B 모델카드 — https://huggingface.co/Qwen/Qwen3-ASR-1.7B
- 한국어 상담 데이터 실측 리뷰 (MZ.\_.GPT) — https://mz-moonzoo.tistory.com/133
- Hugging Face Open ASR Leaderboard — https://huggingface.co/spaces/hf-audio/open_asr_leaderboard
