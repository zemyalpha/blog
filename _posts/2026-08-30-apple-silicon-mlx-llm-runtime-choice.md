---
title: "맥에서 로컬 LLM 서빙할 때 MLX를 고르는 기준: M2 Ultra 벤치마크와 M5 뉴럴 액셀러레이터가 바꾼 계산식"
date: "2026-08-30"
keywords: ["MLX", "Apple Silicon", "로컬 LLM", "Ollama", "llama.cpp", "MLC-LLM", "로컬 추론"]
lang: "ko"
description: "Mac Studio M2 Ultra 192GB에서 MLX·MLC-LLM·Ollama·llama.cpp·PyTorch MPS를 비교한 벤치마크 논문과 M5 뉴럴 액셀러레이터 발표를 근거로, 워크로드별 로컬 LLM 런타임 선택 기준을 정리한다."
---

# 맥에서 로컬 LLM 서빙할 때 MLX를 고르는 기준

맥미니나 맥스튜디오로 로컬 LLM을 돌리는 개발자라면 한 번쯤 이 질문에 부딪힌다. "Ollama 쓰다가 MLX로 갈아타야 하나?" 인터넷의 답은 대부분 경험담 수준이다. "Ollama가 편하다", "MLX가 빠르다더라". 그런데 2025년 11월, Persistent Systems 연구팀이 Mac Studio M2 Ultra(192GB 통합 메모리)에서 다섯 개 런타임을 동일 조건으로 비교한 논문을 발표했고([arXiv:2511.05502](https://arxiv.org/abs/2511.05502)), 같은 달 Apple은 M5 칩의 GPU 뉴럴 액셀러레이터를 MLX가 활용한다고 공식 발표했다([Apple ML Research Blog](https://machinelearning.apple.com/research/exploring-llms-mlx-m5)). 이 두 자료를 겹쳐 보면 "어떤 맥에서 무엇을 할 때 무엇을 쓸 것인가"에 대한 답이 꽤 명확해진다.

## 벤치마크 결과: 처리량과 첫 토큰 지연은 다른 이야기다

논문의 측정 환경은 이렇다. Mac Studio M2 Ultra(24코어 CPU, 76 GPU 코어, 192GB), MLX v0.26, MLC-LLM, llama.cpp, Ollama v0.10.1, PyTorch 2.7.1(MPS 백엔드). 모델은 Qwen-2.5-Coder 3B와 Qwen-2.5-7B-Instruct, 프롬프트는 1k~100k 토큰까지 다양하게 구성했다. 각 조합 10회 반복 측정이며 스크립트와 로그를 공개해 재현 가능하다.

항목별 결과를 정리하면:

| 런타임 | 지속 디코딩 처리량 | 특징 |
|---|---|---|
| MLX | ~230 tok/s | 중간 토큰 지연 5–7ms, P99 ~12ms, GPU 사용률 90% 이상 |
| MLC-LLM | ~190 tok/s | 중간 크기 입력(≤16k)에서 TTFT 가장 낮음, paged KV 지원 |
| llama.cpp | ~150 tok/s (단문맥) | 32k 토큰 이상에서 급락, 100k에서 ~1.2 tok/s |
| Ollama | 20–40 tok/s | 접두사 재사용 LRU 캐시로 지연 ~23% 감소 |
| PyTorch MPS | ~7–9 tok/s | 4GB 텐서 한계, 대형 모델에서 OOM 잦음 |

흥미로운 지점은 순위가 아니라 **트레이드오프의 구조**다. MLX는 지속 처리량이 가장 높지만 첫 토큰까지의 시간(TTFT)은 입력 길이에 따라 선형적으로 늘어난다. 반면 MLC-LLM은 처리량이 ~17% 낮은 대신 vLLM 스타일 paged KV 캐시 덕분에 중간 크기 입력에서 첫 토큰이 더 빨리 나오고, 100k 토큰 컨텍스트에서도 프리필 ~500 tok/s, 디코딩 ~190 tok/s를 유지한다(단, 128k 실행 시 RAM 70–85GB 소모). 채팅형 코드 생성처럼 턴마다 컨텍스트가 자라나는 워크로드에서는 사용자가 첫 토큰을 기다리는 시간이 체감 성능의 대부분이므로, 최대 처리량보다 TTFT가 선택 기준이 될 수 있다는 것이다.

Ollama의 20–40 tok/s 수치는 논문의 측정 세팅(서빙 구성, 양자화 조합)에 따른 것으로 일반적인 단일 사용자 체감보다 낮게 나올 수 있음을 감안해야 하지만, 이 논문의 결론인 "인체공학적 편의성 vs 속도"의 트레이드오프 자체는 부합한다.

## 긴 컨텍스트: KV 캐시 설계가 운명을 가른다

100k 토큰 길이에서 각 프레임워크의 메모리 설계 차이가 그대로 드러난다.

- **MLX** — 설정 가능한 회전(rotating) KV 캐시(기본 4k 토큰 윈도우)로 메모리가 무한정 자라지 않고, 디스크 프롬프트 캐시로 공통 접두사 재계산을 건너뛴다. 100k 토큰에서 ~40–50GB 수준.
- **MLC-LLM** — paged KV로 64k–128k까지 효율적. 단 절대 RAM 사용량이 높다.
- **Ollama** — 접두사 재사용(LRU)만 있고 페이징이 없어 32k 이상에서 성능이 가파르게 떨어지고 100k에서 ~56GB까지 치솟는다.
- **llama.cpp** — 세션 단위 슬라이딩 윈도우뿐, 세션 간 재사용 불가. 32k 넘으면 처리량이 붕괴한다.
- **PyTorch MPS** — 4GB 텐서 상한 때문에 자주 OOM난다.

즉 "짧은 대화를 편하게"라면 Ollama, "문서 전체를 백만 번 요약"이면 MLC-LLM, "그 중간 어딘가에서 균형"이면 MLX라는 그림이 나온다.

## M5 이후 계산식이 바뀌었다: TTFT는 컴퓨트, 디코딩은 메모리 대역폭

Apple의 2025년 11월 발표는 이 논문의 결론에 새 변수를 던진다. M5 GPU에 탑재된 뉴럴 액셀러레이터는 전용 행렬곱 연산을 제공하며, MLX는 Metal 4의 TensorOps/Metal Performance Primitives 프레임워크를 통해 이를 활용한다(macOS 26.2 이상 필요).

Apple이 공개한 M5 맥북프로(24GB) vs M4 비교 수치:

| 모델 | TTFT 속도 향상 | 디코딩 속도 향상 | 메모리 |
|---|---|---|---|
| Qwen3-1.7B bf16 | 3.57x | 1.27x | 4.40GB |
| Qwen3-8B bf16 | 3.62x | 1.24x | 17.46GB |
| Qwen3-14B 4bit | 4.06x | 1.19x | 9.16GB |
| gpt-oss-20b MXFP4 | 3.33x | 1.24x | 12.08GB |
| Qwen3-30B-A3B 4bit | 3.52x | 1.25x | 17.31GB |

규칙이 보이는가. 첫 토큰 생성은 컴퓨트 바운드라 **3.3~4.1배** 빨라지는데, 이후 토큰 생성은 메모리 대역폭 바운드라 **1.2배 남짓**만 빨라진다. M4의 120GB/s 대비 M5가 153GB/s(28% 증가)라는 대역폭 차이가 그대로 디코딩 속도 상한이 되는 것이다. 24GB 통합 메모리로 14B 4bit 모델의 첫 토큰이 10초 안에, 30B MoE(활성 3B)가 3초 안에 나온다는 것은 "맨 위 프리필 지연"이라는 MLX의 약점이 하드웨어 세대 교체로 줄어들고 있음을 뜻한다.

## 실전 선택 가이드

정리하면 워크로드별 추천은 이렇다.

1. **대화형 앱, 프로토타이핑, 팀 온보딩** — Ollama. 원 커맨드 설치와 OpenAI 호환 API가 주는 운영 단순성이 미세한 속도 차를 상쇄한다.
2. **단일 스트림 최대 속도, 배치·파이프라인 처리** — MLX. pip 한 줄로 설치되고 지속 처리량과 토큰 간 지연 안정성이 가장 좋다.
3. **100k 토큰급 문서 처리, 기존 GPTQ/AWQ 자산 활용** — MLC-LLM. paged KV와 양자화 유연성이 강점(3B 모델 AWQ 4bit 시 ~1.6GB, FP16 대비 처리량도 소폭 향상 측정).
4. **최소 풋프린트, 임베디드성 스크립트** — llama.cpp. 콜드스타트 0.5초 미만(캐시된 GGUF)이라는 숫자는 다른 대안이 없다.

MLX로 시작하는 가장 빠른 경로는 두 줄이다:

```bash
pip install mlx-lm
mlx_lm.chat --model mlx-community/Qwen2.5-Coder-3B-Instruct-4bit
```

Hugging Face의 `mlx-community`에 사전 양자화된 모델이 준비되어 있어 별도 변환 없이 바로 실행된다(HF 기준 해당 모델은 1,800회 이상 다운로드된 검증된 변환본이다). 직접 양자화하려면 `mlx_lm.convert` 한 줄이면 된다:

```bash
python -m mlx_lm.convert \
  --hf-path Qwen/Qwen2.5-Coder-3B-Instruct \
  --q-bits 4 --mlx-path ./qwen-coder-3b-4bit
```

4bit 양자화는 통합 메모리 기준으로 7B 모델을 ~5.6GB에 올릴 수 있어 16GB 맥에서도 여유가 생긴다.

## 남는 한계: 데이터센터와의 격차는 여전하다

논문은 Apple 실리콘 런타임들이 "프로덕션급 온디바이스 추론으로 빠르게 성숙하고 있지만" NVIDIA A100 + vLLM 대비 절대 처리량이 여전히 ~5–10배 뒤처진다고 명시한다. 로컬 추론의 승리 조건은 속도가 아니라 프라이버시(논문의 모든 측정이 텔레메트리 없는 온디바이스에서 이뤄졌다), 데이터 이탈 없는 기밀 문서 처리, 그리고 API 비용 제로라는 점이다. 개인 서버나 소규모 팀의 사내 도구 수준이라면 이제 "클라우드 대신 맥 한 대"는 충분히 현실적인 아키텍처다.

핵심 요약:

- MLX = 최고 지속 처리량(~230 tok/s) + 가장 안정적인 토큰 간 지연. Apple 칩 우선 선택지
- MLC-LLM = 낮은 TTFT + paged KV로 초장문 컨텍스트 강세. 단 RAM 사용량 큼
- Ollama = 편의성, llama.cpp = 최소 풋프린트. 둘 다 32k 토큰 이상에서 한계
- M5 뉴럴 액셀러레이터는 TTFT를 3~4배 끌어올렸지만 디코딩은 메모리 대역폭(153GB/s)이 제한 — 다음 세대 칩의 관전 포인트는 대역폭이다
- 첫 단계 제안: 24GB 이상 맥이라면 `pip install mlx-lm` 후 4bit 모델 하나로 기존 API 호출 워크로드 하나를 로컬로 옮겨보고 응답 품질과 지연을 직접 비교해보라

출처: [arXiv:2511.05502 — Production-Grade Local LLM Inference on Apple Silicon](https://arxiv.org/abs/2511.05502), [Apple ML Research — Exploring LLMs with MLX and the Neural Accelerators in the M5 GPU](https://machinelearning.apple.com/research/exploring-llms-mlx-m5) (2025-11-19)
