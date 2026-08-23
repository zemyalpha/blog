---
title: "한 번에 256토큰을 찍어내는 텍스트 디퓨전: DiffusionGemma 로컬 실행 가이드와 AR 대비 트레이드오프"
date: "2026-08-23"
keywords: ["텍스트 디퓨전", "DiffusionGemma", "디퓨전 언어 모델", "DLM", "Gemini Diffusion", "llama.cpp", "병렬 디코딩"]
lang: "ko"
description: "토큰을 왼쪽에서 오른쪽으로 하나씩 찍어내는 autoregressive LLM과 달리, 텍스트 디퓨전 모델은 노이즈 캔버스를 여러 패스로 정제해 한 블록을 동시에 생성한다. Google DeepMind의 DiffusionGemma(26B MoE)를 로컬에서 실행하는 방법과 Gemini Diffusion 벤치마크, 그리고 디퓨전이 유리한 지점과 불리한 지점을 정리한다."
---

# 한 번에 256토큰을 찍어내는 텍스트 디퓨전: DiffusionGemma 로컬 실행 가이드와 AR 대비 트레이드오프

LLM 서빙에서 응답 지연(latency)은 여전히 가장 골치 아픈 문제다. 오늘날 대부분의 언어 모델은 autoregressive(AR) 방식, 즉 토큰을 왼쪽에서 오른쪽으로 하나씩 순서대로 생성한다. 타자기로 글을 치는 것과 같은 방식이라, 첫 토큰이 나오고 나서도 사용자는 나머지 토큰이 전부 도착할 때까지 기다려야 한다.

2026년 들어 이 구조를 정면으로 도전하는 흐름이 눈에 띄게 강해졌다. 이미지 생성을 지배한 디퓨전(diffusion) 기법을 텍스트에 적용하는 **디퓨전 언어 모델(Diffusion Language Model, DLM)**이다. Google DeepMind는 실험 모델인 Gemini Diffusion을 공개했고, 2026년 6월에는 오픈 웨이트 모델 **DiffusionGemma**를 Apache 2.0 라이선스로 풀었다. 이 글에서는 텍스트 디퓨전의 작동 원리, 공식 벤치마크 수치, 그리고 llama.cpp와 Unsloth Studio로 DiffusionGemma를 실제 로컬에서 돌리는 방법을 다룬다.

## 텍스트에 디퓨전을 적용한다는 것: 타자기에서 인쇄기로

AR 모델이 "다음 토큰 하나"를 예측하는 것과 달리, 텍스트 디퓨전 모델은 이미지 디퓨전과 비슷하게 동작한다.

1. **캔버스 생성**: 모델은 랜덤 플레이스홀더 토큰으로 채워진 캔버스에서 시작한다.
2. **반복 정제(refinement)**: 여러 패스를 거치며 확신 있는 토큰부터 확정(잠금)해 나가고, 불확실한 토큰은 다음 패스에서 다시 쓴다.
3. **블록 단위 출력**: DiffusionGemma의 경우 한 번의 forward pass에 256개 토큰을 병렬로 생성한다.

이 구조가 주는 이점은 두 가지다. 첫째, **양방향 어텐션**이다. 블록 안의 모든 토큰이 서로를 참조할 수 있어서, 스도쿠처럼 "각 토큰이 미래 토큰에 의존하는" 비선형 문제나 코드 인필링(infilling)에 구조적으로 유리하다. AR 모델이 스도쿠를 어려워하는 이유는 다음 토큰이 뒤에 올 토큰들과 제약을 공유하기 때문인데, 디퓨전은 애초에 블록 전체를 한꺼번에 보기 때문에 이 제약이 사라진다. 실제로 Unsloth는 DiffusionGemma를 파인튜닝해 스도쿠를 풀게 하는 데모를 공식 블로그에 올렸다.

둘째, **하드웨어 활용 방식의 역전**이다. AR 디코딩은 메모리 대역폭(memory bandwidth)이 병목이다. 토큰 하나를 만들 때마다 전체 가중치를 메모리에서 읽어와야 하니, GPU는 다음 "키 입력"을 기다리며 대기하는 시간이 길어진다. 디퓨전은 256토큰을 한 번에 처리하므로 병목이 연산(compute) 쪽으로 이동하고, 로컬 단일 GPU 환경에서 하드웨어를 포화 상태로 쓸 수 있다. Google의 비유를 빌리면 "순차적 타자기를 대량 인쇄기로 바꾸는 것"이다.

2025년 8월 처음 나와 2026년 6월 3판(v3)이 나온 서베이 논문 "A Survey on Diffusion Language Models"(arXiv:2508.10875)은 DLM을 "병렬 토큰 생성과 반복 디노이징으로 추론 지연을 줄이고 양방향 컨텍스트를 잡는, AR 패러다임의 유력한 대안"으로 정리하며 사전학습부터 추론 최적화(디코딩 병렬성, 캐싱)까지 체계적으로 다루고 있다. 이 분야가 연구실험 단계를 넘어 실용화 초기 단계에 진입했다는 신호다.

## Gemini Diffusion 벤치마크: 속도가 아니라 품질이 과제

"디퓨전이 빠르다"는 것은 이제 상식이 됐지만, 정확히 얼마나 빠르고 품질은 어떤지는 공식 수치로 확인해야 한다. Google DeepMind가 Gemini Diffusion 실험 모델 페이지에 공개한 벤치마크(pass@1, Gemini 2.0 Flash-Lite 대비)는 다음과 같다.

| 벤치마크 | Gemini Diffusion | Gemini 2.0 Flash-Lite |
|---|---|---|
| LiveCodeBench (v6) | 30.9% | 28.5% |
| HumanEval | 89.6% | 90.2% |
| MBPP | 76.0% | 75.8% |
| SWE-Bench Verified* | 22.9% | 28.5% |
| GPQA Diamond | 40.4% | 56.5% |
| AIME 2025 | 23.3% | 20.0% |
| BIG-Bench Extra Hard | 15.0% | 21.0% |
| Global MMLU (Lite) | 69.1% | 79.0% |

*비에이전트 평가(단일 턴 편집), 최대 프롬프트 32K. 출처: [Google DeepMind Gemini Diffusion 페이지](https://deepmind.google/models/gemini-diffusion/)

읽어내는 요점은 세 가지다.

- **코드·수학 편집에 강하다.** LiveCodeBench에서는 오히려 Flash-Lite를 앞서고, AIME 2025에서도 23.3%로 우위다. 오류를 생성 중간에 되돌아가 고칠 수 있다는 특성이 코드와 수학 형식 검증에 유효하게 작동한다는 해석이 가능하다.
- **범용 지식·추론에서는 크게 뒤진다.** GPQA Diamond에서 16점, Global MMLU Lite에서 10점 차이다. 즉 텍스트 디퓨전은 아직 "더 똑똑한 모델"이 아니라 "다른 특성을 가진 모델"이다.
- **속도는 확실하다.** DeepMind 페이지는 샘플링 속도를 초당 1,479 토큰(오버헤드 0.84초 제외)으로 기록했다. AR 모델의 실사용 디코딩 속도가 보통 수십~100토큰/초 수준임을 고려하면 자릿수가 다르다.

## DiffusionGemma 실전 실행: llama.cpp와 Unsloth Studio

DiffusionGemma는 Gemma 4 MoE 아키텍처를 기반으로 한 26B(총 파라미터) 모델로, 추론 시 3.8B만 활성화된다. 양자화하면 18GB VRAM/RAM으로 로컬 실행이 가능하고, 공식 발표 기준 단일 NVIDIA H100에서 초당 1,000토큰 이상, 지포스 RTX 5090에서 초당 700토큰 이상을 낸다. 컨텍스트 256K, 140개 이상 언어 지원, 텍스트·이미지·비디오 입력을 받는 멀티모달 모델이라는 것이 Unsloth 문서의 설명이다.

주의할 점 하나. **AR용 추론 런타임 설정(temperature, top_p, top_k)만으로는 권장 동작이 재현되지 않는다.** 디퓨전 샘플러를 이해하는 런타임이 필요하다. 현재 가장 접근성 좋은 경로는 llama.cpp(특정 PR 브랜치)와 Unsloth Studio 두 가지다.

### 방법 1: llama.cpp로 직접 실행

llama.cpp의 디퓨전 지원 PR을 빌드해서 쓴다. Apple 실리콘에서는 Metal이 기본 켜져 있고, `-DGGML_CUDA=OFF`로 순수 CPU 빌드도 가능하다.

```bash
# 디퓨전 지원 PR 브랜치 빌드 (Apple Silicon 기준)
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=OFF && cmake --build build -j

# 모델 다운로드 (4-bit Q4_K_M, 18GB)
pip install huggingface_hub
huggingface-cli download unsloth/diffusiongemma-26B-A4B-it-GGUF

# 대화형 실행
./build/bin/llama-cli \
  -m diffusiongemma-26B-A4B-it-Q4_K_M.gguf \
  -ngl 99 -cnv -n 2048 \
  --diffusion-visual
```

여기서 눈여겨볼 플래그들이 디퓨전 특유의 튜닝 지점이다.

- `--diffusion-visual`: 캔버스가 여러 패스에 걸쳐 정제되는 과정을 라이브로 보여준다. 디퓨전이 "어떻게" 글을 쓰는지 체감하기에 최고의 학습 도구다.
- `--diffusion-eb auto`: Entropy-Bound 샘플러(기본 활성). `--diffusion-eb-max-steps`(기본 48), `--diffusion-eb-entropy-bound`(0.1), `--diffusion-eb-confidence`(0.005) 등으로 정제 패스 수와 확신 임계값을 조절한다.
- `--diffusion-kv-cache auto`: 프롬프트 접두부 KV 캐시. 단일 GPU에서 자동 활성화된다.

### 방법 2: Unsloth Studio (GUI)

더 간단한 길은 Unsloth Studio다. v0.1.463-beta 이상에서 DiffusionGemma 실행 및 파인튜닝을 지원하며, GGUF 검색·다운로드·실행을 웹 UI에서 처리하고 추론 파라미터를 자동으로 잡아준다.

```bash
# 설치 후 실행
pip install unsloth
unsloth studio
# 브라우저에서 http://127.0.0.1:8888 접속 →
# Chat 탭에서 "DiffusionGemma" 검색 → Q4_K_M 다운로드 → 실행
```

양자화 단위별 메모리 요구량(RAM+VRAM 합산, Unsloth 공식 표)은 4-bit 18GB, 5-bit 20GB, 6-bit 24GB, 8-bit 28GB, BF16/FP16 52GB다. M 시리즈 맥이나 24GB급 게이밍 GPU라면 4~6-bit 구간이 현실적인 선택이다.

## 어디에 쓰고, 어디에 쓰지 말아야 하나

텍스트 디퓨전의 속도 이점은 조건이 붙어 있다. Google이 명시한 권장 사용처와 비권장 사용처를 정리하면 다음과 같다.

**유리한 지점 —**

- **로컬·저동시성(low-concurrency) 추론**: 단일 사용자가 단일 GPU로 쓸 때 하드웨어 포화 효과가 최대가 된다.
- **인라인 편집·코드 인필링**: 블록 전체를 동시에 보므로 중간 삽입·수정에 구조적으로 강하다.
- **비선형 구조 생성**: 앞서 본 스도쿠, 아미노산 서열, 수학적 그래프처럼 토큰 간 미래 의존성이 강한 도메인.
- **빠른 프로토타이핑**: 생성 중 오류를 스스로 되돌아가 정정하므로 초안 반복 속도가 빠르다.

**불리한 지점 —**

- **고 QPS 클라우드 서빙**: AR 모델은 수천 요청을 배칭해 연산을 포화시킬 수 있어, 병렬 디코딩의 이득이 줄어들고 오히려 서빙 비용이 커질 수 있다. Google 스스로 "고품질 프로덕션 출력에는 표준 Gemma 4를 권장한다"고 못 박았다.
- **최대 품질이 필요한 작업**: DiffusionGemma의 전반적 출력 품질은 표준 Gemma 4보다 낮다. 도메인 파인튜닝으로 특정 태스크 성능은 끌어올릴 수 있다.

## 결론

텍스트 디퓨전은 AR을 "대체"하는 기술이 아니라, 지연에 민감한 영역을 책임지는 보완 패러다임으로 자리 잡아가고 있다. 오늘 시점에서 정리할 수 있는 것은 다음과 같다.

- 디퓨전 언어 모델은 노이즈 캔버스를 반복 정제해 한 번에 256토큰을 병렬 생성하며, 병목을 메모리 대역폭에서 연산으로 옮겨 로컬 GPU 활용률을 극대화한다.
- Gemini Diffusion 공식 벤치마크에서 코드·수학 편집 영역은 경쟁 AR 모델과 대등하거나 우세하지만, 범용 지식·추론(GPQA, MMLU)에서는 아직 뒤진다.
- DiffusionGemma(26B MoE / 3.8B 활성, Apache 2.0)는 4-bit 양자화 기준 18GB로 로컬 실행 가능하며, llama.cpp 디퓨전 PR 또는 Unsloth Studio로 바로 시작할 수 있다.
- 도입 판단의 기준은 "빠르냐"가 아니라 "로컬·저동시성·비선형 생성이라는 조건에 부합하냐"다. 고 QPS 클라우드 서빙이나 최대 품질 요구사항이라면 아직 AR이 정답이다.

당장 시도해볼 첫 단계는 간단하다. `unsloth studio`를 설치해 DiffusionGemma Q4_K_M을 내려받고 `--diffusion-visual` 옵션으로 캔버스가 정제되는 과정을 직접 눈으로 확인해 보는 것. AR 모델이 왜 그렇게 느리게 "타자"를 치는지, 그리고 그 대안이 실제로 어떻게 굴러가는지 감을 잡는 가장 빠른 방법이다.

**참고 자료**

- Google DeepMind, "Gemini Diffusion" 모델 페이지 — https://deepmind.google/models/gemini-diffusion/
- Google, "DiffusionGemma: 4x faster text generation" (2026-06-10) — https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/
- Unsloth, "DiffusionGemma — How to Run Locally" — https://unsloth.ai/docs/models/diffusiongemma
- Li et al., "A Survey on Diffusion Language Models" (arXiv:2508.10875, v3 2026-06-04) — https://arxiv.org/abs/2508.10875
