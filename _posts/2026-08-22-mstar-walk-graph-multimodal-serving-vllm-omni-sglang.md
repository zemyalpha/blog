---
title: "추론은 더 이상 하나의 루프가 아니다: Stanford M*의 Walk Graph가 멀티모달 서빙을 재설계한 방식"
date: "2026-08-22"
keywords: ["멀티모달 서빙", "M*", "vLLM-Omni", "SGLang-Omni", "Walk Graph", "Qwen3-Omni", "BAGEL", "LLM 추론"]
lang: "ko"
description: "vLLM과 SGLang은 '프리필 후 토큰 하나씩 디코딩'이라는 단일 루프를 가정한 엔진이다. BAGEL·Qwen3-Omni·V-JEPA 2 같은 복합 멀티모달 모델은 이 가정을 깨는데, Stanford M*는 모델을 그래프로, 요청을 순회(Walk)로 추상화해 최대 12.5배 빠른 서빙을 보였다."
---

# 추론은 더 이상 하나의 루프가 아니다: Stanford M*의 Walk Graph가 멀티모달 서빙을 재설계한 방식

2026년의 서빙 엔진 논쟁은 보통 이렇다. "vLLM이냐, SGLang이냐, TensorRT-LLM이냐." 하지만 이 질문에는 숨은 전제가 하나 있다. **추론이 하나의 자기회귀 루프**라는 것. 프롬프트를 프리필하고, 토큰을 하나씩 디코딩하고, 종료 신호에서 멈춘다. vLLM과 SGLang의 PagedAttention, RadixAttention, 연속 배칭 같은 핵심 최적화는 전부 이 루프를 기준으로 설계됐다.

문제는 최신 모델들이 이 틀에 들어오지 않는다는 점이다. 이미지를 이해하고 생성하는 통합 모델(BAGEL), 듣고 말하는 옴니 모델(Qwen3-Omni), 로봇 플래닝을 위한 월드 모델(Meta의 V-JEPA 2)은 구조적으로 이질적인 컴포넌트 — 비전 인코더, 트랜스포머 백본, 디퓨전·플로우 헤드, 오디오 코덱, 액션 예측기 — 를 입력에 따라 다르게 조합해서 쓴다. 디퓨전 루프는 고정 횟수를 도는 비-자기회귀 루프고, 옴니 모델의 Thinker–Talker는 서로를 기다리지 않고 파이프라인으로 겹치며, 하나의 모델 안에서 "이미지 생성"과 "이미지 이해"가 서로 다른 컴포넌트 경로를 지난다.

Stanford와 University of Washington이 2026년 6월에 공개한 **M\***은 이 어긋남을 정면으로 다룬다. 기존 시스템들의 추상화를 일반화한 **Walk Graph**를 도입해, 하나의 런타임으로 통합 멀티모달 모델·옴니 모델·음성 언어 모델·VLA·월드 모델을 전부 서빙하겠다는 접근이다. 이 글에서는 기존 스택이 왜 이 모델들을 못 실었는지, Walk Graph가 무엇인지, 숫자로 검증된 성능이 무엇인지, 그리고 지금 도입할 가치가 있는지를 정리한다.

## 왜 기존 서빙 스택이 무너지는가: 모달리티 락인과 스테이지 DAG

M* 논문(arXiv 2606.12688)은 기존 시스템의 한계를 두 층으로 진단한다.

**1층: vLLM·SGLang — 모달리티 락인.** 두 엔진은 텍스트 자기회귀에는 훌륭하지만, 구조적으로 "출력이 항상 텍스트인 단일 디코드 루프"다. 이미지 입력조차 프리필 시점의 인코더 애드온으로만 들어온다. 이질적 컴포넌트를 루프나 병렬 브랜치로 조합하는 1급 방법이 없고, 컴포넌트 간 스트리밍도 없다.

**2층: vLLM-Omni·SGLang-Omni — 평면 스테이지 DAG.** 옴니 모델용 확장들은 한 발 더 나아가 요청을 여러 스테이지의 파이프라인(플랫 DAG)으로 모델링한다. vLLM-Omni(arXiv 2602.02204)은 스테이지 추상화 프론트엔드와 분리 실행 백엔드로 구성되고, Qwen3-Omni를 Thinker → Talker → Code2Wav의 3단계로 서빙한다. SGLang-Omni도 마찬가지로 전처리·인코더·AR 엔진·토크·보코더·어그리게이터 스테이지로 나눈다. Thinker–Talker–코덱 체인 정도는 표현할 수 있게 된 것이다.

그러나 DAG는 이름 그대로 **비순환**이다. 루프는 반드시 스테이지 내부에 갇히고, 스테이지 간 병렬 조합은 불가능하다. 그 결과 디퓨전 루프나 classifier-free guidance(CFG) 팬아웃 같은 패턴은 모델별 글루 코드로 처리된다. 실제로 vLLM-Omni에서 BAGEL의 3-way CFG는 `torch.distributed` 기반의 맞춤형 플러그인으로 구현돼 있다고 M* 블로그는 지적한다. 최적화가 모델 코드와 시스템 코드에 얽혀버리는 구조다.

정리하면: HuggingFace Transformers는 유연하지만 느리고, vLLM·SGLang은 빠르지만 텍스트에 락인되며, vLLM-Omni·SGLang-Omni는 그 사이의 절충안에 머문다. M* 논문의 표현을 빌리면, 이전 추상화들은 전부 Walk Graph의 "제한된 부분집합"이다.

| | vLLM-Omni | SGLang-Omni | M* |
|---|---|---|---|
| 그래프 노드 | 엔진 인스턴스 스테이지 | 워커 풀 스테이지 | 모델 컴포넌트 |
| 조합 | 평면 DAG | 평면 DAG | 순차 / 병렬 / 루프 / 스트림 |
| 루프 | 스테이지 내부만 | 스테이지 내부만 | 임의 서브그래프에 걸쳐 |
| 배치(placement) | 스테이지 단위 | 스테이지 단위 | 컴포넌트 단위, Walk별 오버라이드 가능 |

## Walk Graph: 모델은 그래프, 요청은 순회

M*의 핵심 발상은 문장 하나로 요약된다. **모델은 데이터플로우 그래프이고, 요청은 그 그래프 위의 순회(walk)다.**

모델 저자는 텐서 엣지로 연결된 컴포넌트 노드들의 그래프를 선언하고, 각각의 동작 단계에 해당하는 명명된 **Walk**들의 집합을 정의한다. 요청이 오면 저자가 작성한 작은 상태 머신이 Walk를 하나씩 고른다. 물리적인 것 — 배치, 스케줄링, KV 캐싱, CUDA 그래프, 텐서 전송, 스트리밍 — 은 전부 런타임의 몫이다.

BAGEL(ByteDance, 활성 7B/총 14B, MoT 아키텍처)을 예로 보면 그래프는 컴포넌트 4개로 이뤄진다: `vit_encoder`, `vae_encoder`, `LLM`, `vae_decoder`. 그리고 요청 유형마다 Walk가 달라진다.

- 이미지 생성(텍스트→이미지): `prefill_text` → `image_gen`
- 이미지 이해(이미지→텍스트): `prefill_text` → `prefill_vit` → `decode`
- 이미지 편집(이미지→이미지): `prefill_text` → `prefill_vae` → `prefill_vit` → `image_gen`

Walk로 요청을 정의한다는 것은 **런타임이 그 요청에 필요한 컴포넌트만 실행한다**는 뜻이다. 이미지 이해 요청은 디퓨전 루프와 `vae_decoder`를 아예 건드리지 않는다. 요청을 상태 머신으로 고르는 코드는 이렇게 단순하다(블로그 게시 코드의 단순화 버전).

```python
# 현재 단계에 따라 다음 Walk를 고르는 상태 머신
def next_walk(self, state):
    if state.prefill_steps:
        # 입력을 아직 소비 중이면 프리필 단계부터
        return state.prefill_steps.pop(0)  # prefill_text / prefill_vae / prefill_vit
    if state.target == "image":
        return "image_gen"  # CFG가 설정됐으면 image_gen_cfg
    return "decode"         # 아니면 자기회귀 텍스트
```

### 네 개의 조합 프리미티브

Walk Graph의 표현력은 4개 프리미티브에서 나온다. `Sequential`(순차), `Parallel`(병렬), `Loop`(반복), 그리고 스트리밍 엣지. BAGEL의 이미지 생성은 플로우 매칭 스텝을 도는 루프인데, M*에서는 이렇게 표현된다.

```python
from mminf.graph.base import Sequential, Loop, GraphNode, GraphEdge

image_gen = Sequential([
    Loop(
        section=GraphNode(
            name="LLM",
            input_names=["latents", "time_index"],
            outputs=[
                GraphEdge(next_node="LLM", name="latents"),
                GraphEdge(next_node="LLM", name="time_index"),
            ],
        ),
        max_iters=49,  # 디노이징 타임스텝 수 - 1
        outputs=[GraphEdge(next_node="vae_decoder", name="latents")],
    ),
    vae_decoder,
])
```

같은 `Loop` 프리미티브가 세 가지 패턴을 덮는다. 고정 횟수 디노이징(위 예), 종료 토큰에서 멈추는 자기회귀 텍스트 디코딩, 지평선(horizon)에서 멈추는 월드 모델 롤아웃. 루프가 1급 시민이므로 **연속 배칭과 CUDA 그래프 재생이 플로우 스텝에도 토큰 디코딩과 똑같이 적용된다.** 이것이 "루프를 스테이지 안에 가두는" DAG 설계와의 결정적 차이다.

CFG 팬아웃은 `Parallel`로 표현한다. 디노이징 스텝마다 무조건부 패스 1개와 조건부 패스 2개를 돌리고 결과를 합치는데, 세 브랜치를 각각 다른 GPU에 놓을 수도 있다. vLLM-Omni에서 모델별 플러그인으로 처리하던 패턴이, M*에서는 런타임이 일반적으로 처리하는 선언이 된다.

## 배치·스트리밍: 논리 그래프와 물리 토폴로지의 분리

Walk Graph의 또 다른 축은 논리 구조와 물리 배치의 분리다. 배치는 작은 YAML 파일이고, 모델 코드는 전혀 바꾸지 않는다.

```yaml
# 단일 GPU: 모든 컴포넌트를 한 랭크에
model: "bagel"
node_groups:
  - { node_names: [vit_encoder, vae_encoder, vae_decoder, LLM], ranks: [0] }
```

같은 BAGEL 그래프를 3-GPU CFG 구성으로 바꾸려면 같은 파일만 고친다.

```yaml
# 3-GPU: CFG 브랜치를 각각 전용 랭크에 (image_gen_cfg Walk에서만 활성)
model: "bagel"
node_groups:
  - { node_names: [vit_encoder, vae_encoder, vae_decoder], ranks: [0] }
  - { node_names: [LLM, combine_cfg], ranks: [0] }
  - { node_names: [LLM_cfg_text], ranks: [1], graph_walks: [image_gen_cfg] }
  - { node_names: [LLM_cfg_img],  ranks: [2], graph_walks: [image_gen_cfg] }
```

`graph_walks` 키가 포인트다. **같은 논리 노드를 Walk별로 다른 GPU에 놓을 수 있다.** 프리필은 GPU 0에서, 디코드는 GPU 1에서 실행하는 식의 프리필/디코드 분리도 이 같은 API로 표현된다. 인코더·디코더는 작은 GPU 여러 장에, LLM 백본은 큰 GPU 소수에 배치하는 컴포넌트별 독립 스케일링도 가능하다. 심지어 BAGEL의 `LLM` 노드처럼 모든 Walk에 등장하는 노드를 같은 랭크에 지정하면, 서로 다른 Walk의 요청들이 같은 물리 복제본을 자동으로 멀티플렉싱한다 — 이미지 생성 트래픽과 이미지 이해 트래픽이 하나의 LLM 복제본을 공유하는 셈이다.

### 스트리밍: Qwen3-Omni의 Thinker–Talker–Code2Wav

시간적으로 겹쳐야 하는 컴포넌트들은 스트리밍 엣지로 잇는다. Qwen3-Omni는 세 컴포넌트의 파이프라인으로 말한다. Thinker(멀티모달 추론 LLM, 약 30B MoE)가 히든 스테이트를 하나씩 Talker(약 3B MoE 코덱 예측기)에 흘려보내고, Talker는 코덱 프레임을 Code2Wav에 흘려보낸다. 전체 응답이 끝나기 전에 오디오 재생이 시작되는 구조다.

M* 논문의 흥미로운 디테일은 이 스트리밍 연결마저 정책(policy)으로 일반화했다는 점이다. Thinker→Talker 연결은 청크 크기 1의 `FixedChunkPolicy`(Thinker 출력이 도착하는 대로 Talker가 소비), Talker→Code2Wav 연결은 인과적 스무딩을 위한 `LeftContextChunkPolicy`를 쓴다. 런타임은 정책의 종류를 모른 채 같은 인프라로 처리한다.

아키텍처면에서는 HTTP 서버 뒤에 요청별 Walk 상태를 관리하는 Conductor가 있고, ZeroMQ로 GPU 랭크별 Worker(단일 프로세스)에 작업을 분배한다. 랭크 간 그래프 엣지는 프로세스 간 텐서 전송이고, 데이터 플레인은 공유 메모리와 Mooncake 기반 RDMA/TCP로 플러거블하다. 각 노드는 컴포넌트 유형에 따라 엔진으로 실행되는데, 트랜스포머 노드는 FlashInfer 기반 페이지드 어텐션 KV 캐시를 쓰는 `KVCacheEngine`이 맡는다.

## 숫자로 보는 성능: 전용 시스템과의 정면 비교

추상화가 예쁘다고 채택하지는 않는다. M* 팀은 5개 실제 모델을 구현해 각 모델의 최고 전용 시스템과 직접 비교했다. 수치는 모두 Stanford SAIL 블로그 및 논문(arXiv 2606.12688) 기준이다.

| 워크로드 | 비교 대상 | 결과 |
|---|---|---|
| Qwen3-Omni TTS | vLLM-Omni | 약 2.7배 높은 처리량 (B=16) |
| Qwen3-Omni TTS | SGLang-Omni | 약 4배 높은 처리량 |
| Qwen3-Omni TTS (Thinker TP-2 샤딩) | SGLang-Omni | 약 3.8배 높은 처리량 (B=16) |
| BAGEL 텍스트→이미지 | vLLM-Omni | 약 1.3배 낮은 지연 |
| BAGEL 이미지 편집 | vLLM-Omni | 최대 2.6배 낮은 지연 |
| BAGEL 이미지→텍스트 | vLLM-Omni | 첫 토큰 약 1.6배 빠름, 짧은 출력에서 최대 46% 높은 처리량 |
| Orpheus TTS | VoxServe | 벤치마크한 모든 배치 크기에서 낮은 RTF·높은 처리량 |
| V-JEPA 2 롤아웃 | Meta 네이티브 구현 | 최대 12.5배 빠름 |

몇 가지 읽을거리가 있다.

**월드 모델의 12.5배는 루프 최적화의 직접 효과다.** V-JEPA 2-AC의 롤아웃은 매 스텝 KV 캐시를 처음부터 재계산하는 대신, M*가 이를 영속 KV 캐시를 가진 `Loop`로 표현하면서 얻은 수치다. Meta가 공개한 네이티브 구현이 느렸다는 점을 감안해도, 추상화 하나가 두 자릿수 배수를 만들어내는 경우는 드물다.

**완승은 아니다.** BAGEL 이미지→텍스트 워크로드에서 M*는 첫 토큰은 1.6배 빠르지만 토큰 간 지연의 중앙값이 1~3ms 높다. 저동시·장문 출력에서는 거의 파리티로 좁혀진다. M*의 이점은 부하가 걸리고 응답이 짧을수록 커지는 구조다.

**논문의 종합 수치**는 BAGEL 텍스트→이미지에서 vLLM-Omni 대비 평균 20% 낮은 end-to-end 지연, Qwen3-Omni TTS에서 최대 2.9배 낮은 RTF(실시간 계수)다.

## 한계와 판단: 지금 쓸 팀과 기다릴 팀

M*를 실무 관점에서 평가하려면 낙관과 회의 사이의 정확한 지점을 잡아야 한다.

**확인되는 한계부터.** 첫째, 프로젝트는 초기 단계다. GitHub 저장소(mstar-project/mstar, Apache 2.0)는 2026년 2월에 만들어졌고 글 작성 시점 기준 50개 남짓의 스타, 30개가 넘는 오픈 이슈를 가지고 있다. 논문도 프리프린트다. 둘째, 벤치마크된 모델은 5개뿐이고 텐서 병렬 셰딩은 일부 모델 패밀리에만 롤아웃됐으며, 시퀀스/DiT 병렬성과 SLO 인지 배치 자동 탐색은 로드맵에 있다. 셋째, 생태계다. vLLM은 모델 추가 속도와 커뮤니티 규모에서 압도적이며, 텍스트 중심 워크로드에서 vLLM/SGLang을 대체할 이유는 어디에도 없다.

**그럼에도 주목할 이유.** 이 글의 판단 기준은 "지금 이 추상화가 옳은가"다. 모델 쪽 트렌드는 명확히 복합 아키텍처로 향하고 있다 — 이해와 생성을 하나의 가중치 안에 넣는 통합 모델(BAGEL, 그리고 VAE 없이 단일 비전 토크나이저로 세 태스크를 처리하려는 후속 연구들), 텍스트·오디오·비디오를 아우르는 옴니 모델, 액션까지 내뱉는 VLA와 월드 모델. 각 모델마다 글루 코드를 붙이는 접근은 모델 수에 비례해 비용이 커진다. "모델은 그래프, 요청은 순회"라는 추상화는 이 다양성을 하나의 인터페이스로 접는다. 저자 명단(Horowitz, Zettlemoyer, Leskovec, Kasikci 등)과 기관(Stanford, UW, CMU)이 이 방향성에 걸고 있다는 점도 무시하기 어렵다.

실전 선택지를 정리하면 이렇다.

- **지금 살펴볼 팀**: 음성 생성·통합 멀티모달·로보틱스 월드 모델을 직접 서빙해야 하는 연구·프로토타입 팀. 특히 롤아웃이나 디퓨전 루프가 병목인 경우, Walk Graph의 루프 최적화는 즉시 체감될 수 있다.
- **지켜볼 팀**: 텍스트+이미지 입력 중심의 일반적인 VLM 서비스. vLLM/SGLang 본진과 vLLM-Omni의 성숙한 파이프라인이 당분간 더 안전한 선택이다.
- **아키텍트라면**: 이 논문의 가치는 M*라는 소프트웨어 자체보다 개념 쪽에 가깝다. "요청이 그래프의 어떤 부분경로를 밟는가"라는 질문은 어떤 서빙 시스템을 고르든 서빙 설계에 적용할 수 있는 프레임이다. 에이전트가 여러 모델 호출을 조합하는 패턴 역시 모델 호출 위의 그래프라는 점에서, M* 팀이 로드맵으로 밝힌 "모델 내 컴포넌트 그래프와 모델 간 에이전트 그래프를 하나의 런타임으로"라는 방향은 서빙과 에이전트 오케스트레이션의 경계를 흐릴 후보다.

## 결론

- vLLM과 SGLang의 근간은 "단일 자기회귀 루프" 가정이며, vLLM-Omni와 SGLang-Omni는 이를 평면 스테이지 DAG로 확장했지만 루프가 스테이지에 갇히고 병렬 조합이 불가능하다.
- M*의 Walk Graph는 모델을 컴포넌트 그래프로, 요청을 Walk(순회)로 추상화하고, 순차·병렬·루프·스트림 네 개의 프리미티브로 복합 모델 전부를 하나의 런타임에 담는다.
- 배치는 YAML만 바꾸면 되고, `graph_walks` 키로 Walk별 GPU 배치 오버라이드, 컴포넌트별 독립 스케일링, Walk 간 물리 복제본 공유까지 표현된다.
- 검증된 성능은 Qwen3-Omni TTS에서 vLLM-Omni 대비 약 2.7배, SGLang-Omni 대비 약 4배 처리량, V-JEPA 2 롤아웃에서 네이티브 대비 최대 12.5배다. 다만 첫 토큰 지연과 토큰 간 지연 사이에는 트레이드오프가 있다.
- 프로젝트는 초기 단계(2026년 2월 개설, 프리프린트)이므로 텍스트 중심 서비스는 기존 스택을 유지하고, 복합 멀티모달 워크로드를 가진 팀부터 평가 대상이 되는 게 합리적이다.

바로 해볼 수 있는 첫 단계는 [M* 문서의 퀵스타트](https://mstar.stanford.edu/mstar/quickstart.html)로 BAGEL 단일 GPU 배치를 띄워보는 것이다. 그리고 비교 기준으로 vLLM-Omni의 Qwen3-Omni 레시피(`vllm serve Qwen/Qwen3-Omni-30B-A3B-Instruct --omni`)를 같은 하드웨어에서 돌려보면, "요청은 그래프 순회다"라는 주장이 어디까지 실제인지 직접 확인할 수 있다.

**참고 자료**
- M* 블로그(Stanford SAIL): https://robotics.stanford.edu/blog/mstar/
- M* 논문: arXiv 2606.12688 — "M*: A Modular, Extensible, Serving System for Multimodal Models"
- M* 코드: https://github.com/mstar-project/mstar (Apache 2.0)
- vLLM-Omni 논문: arXiv 2602.02204 / Qwen3-Omni 서빙 최적화: vLLM 블로그(2026-07-01)
- SGLang-Omni: https://github.com/sgl-project/sglang-omni
- BAGEL(ByteDance-Seed): https://github.com/bytedance-seed/BAGEL
- V-JEPA 2(Meta): https://ai.meta.com/blog/v-jepa-2-world-model-benchmarks
