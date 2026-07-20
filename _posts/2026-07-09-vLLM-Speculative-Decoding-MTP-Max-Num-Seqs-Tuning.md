---
layout: post
title: "vLLM 성능 튜닝: speculative_config, MTP, --max-num-seqs"
author: 'Juho'
date: 2026-07-09 00:00:00 +0900
categories: [LLM]
tags: [vLLM, LLM, GPU, Benchmark]
pin: True
toc: True
---

<style>
  th{
    font-weight: bold;
    text-align: center;
    background-color: white;
  }
  td{
    background-color: white;
  }
</style>

## 목차
1. [개요](#개요)
2. [speculative decoding 기본 원리](#speculative-decoding-기본-원리)
   - [--speculative-config JSON 스펙](#--speculative-config-json-스펙)
   - [지원 method 정리](#지원-method-정리)
3. [MTP: Multi-Token Prediction](#mtp-multi-token-prediction)
   - [MTP가 무엇인가](#mtp가-무엇인가)
   - [설정과 권장값](#설정과-권장값)
4. [--max-num-seqs와 배치 튜닝](#--max-num-seqs와-배치-튜닝)
   - [max_num_batched_tokens와의 관계](#max_num_batched_tokens와의-관계)
   - [처리량-지연-메모리 트레이드오프](#처리량-지연-메모리-트레이드오프)
5. [실무 함정: 언제 오히려 느려지는가](#실무-함정-언제-오히려-느려지는가)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

vLLM에서 추론 속도를 끌어올리는 대표적인 노브가 speculative decoding(추측 디코딩), MTP(Multi-Token Prediction), 그리고 배치 상한을 정하는 `--max-num-seqs`입니다.
세 가지 모두 "더 빠르게"를 지향하지만, 워크로드와 부하 조건에 따라 정반대 결과가 나올 수 있습니다.
이 글은 vLLM 공식 문서와 소스코드, 그리고 GitHub 이슈·포럼의 실측 사례를 통합해 각 파라미터의 동작 원리와 성능 튜닝 관점을 정리합니다.

speculative decoding은 작은 draft(초안) 제안자가 여러 토큰을 미리 만들고 큰 타깃 모델이 한 번의 forward pass로 병렬 검증하는 방식입니다.
MTP는 모델 내부에 내장된 예측 레이어를 draft로 재사용하는 speculative decoding의 한 갈래입니다.
`--max-num-seqs`는 continuous batching에서 한 스텝에 동시에 처리할 시퀀스 개수의 상한입니다.

## speculative decoding 기본 원리

speculative decoding의 핵심 아이디어는 "생성은 싸게, 검증은 병렬로"입니다.
작은 draft 제안자가 스텝당 k개의 후보 토큰을 미리 생성합니다.
그다음 원본 verifier(검증) 모델이 이 후보들을 한 번의 forward pass로 병렬 검증합니다.
확률적으로 토큰을 수락 또는 거부(rejection sampling)하기 때문에, 원본 모델의 출력 분포를 그대로 보존합니다.
이론적으로는 hardware precision 한계 내에서 무손실입니다.

vLLM은 이 모든 draft 방식을 `speculative_config` 하나의 딕셔너리(또는 CLI의 `--speculative-config` JSON 문자열)로 통합해서 설정합니다.
vLLM V1 엔진은 EAGLE와 MTP speculative decoding을 모두 지원합니다.

### --speculative-config JSON 스펙

CLI에서는 JSON 객체로 전달하고, YAML 설정 파일에서는 escape된 JSON 문자열 대신 중첩 매핑으로 작성합니다.
공식 문서 기준 공통 파라미터는 다음과 같습니다.

| 파라미터 | 타입 | 기본값 | 설명 |
|---|---|---|---|
| method | string | None | 제안 방식 (draft_model, ngram, suffix, mtp, eagle/eagle3, medusa 등) |
| model | string | None | draft 모델 또는 보조 모델 식별자 |
| num_speculative_tokens | int | None | 스텝당 제안 토큰 수 (0보다 큰 값) |
| draft_tensor_parallel_size | int | None | draft 모델용 텐서 병렬 크기 (1 이상) |
| max_model_len | int | None | draft 모델 컨텍스트 길이 |
| rejection_sample_method | string | strict | strict / probabilistic / synthetic |

주의할 점이 하나 있습니다.
draft 모델의 텐서 병렬 크기는 `tensor_parallel_size`가 아니라 `draft_tensor_parallel_size` 키를 사용해야 합니다.
`target_model_config`, `draft_model_config` 같은 내부 필드는 vLLM이 자동으로 채우므로 사용자가 직접 설정하지 않습니다.
temperature나 top_p 같은 샘플링 파라미터도 `--speculative-config`에 넣지 않고 별도 SamplingParams로 지정합니다.

draft `model`만 지정하면 vLLM이 model_type을 보고 method를 자동 유추합니다.
예를 들어 모델명에 "eagle3"가 포함되면 eagle3, model_type이 medusa이면 medusa, MTP 계열이면 mtp로 판별합니다.

### 지원 method 정리

vLLM 소스코드의 method 자동 판별 로직상 지원되는 값은 다음과 같습니다.

| method | 특징 | draft 모델 필요 |
|---|---|---|
| ngram | 프롬프트에서 n-gram 매칭으로 제안 (prompt lookup) | 불필요 |
| draft_model | 별도의 작은 draft 모델 사용 (고전적 방식) | 필요 |
| eagle / eagle3 | hidden state 기반 경량 draft head, 현재 SOTA | 경량 speculator |
| medusa | 여러 개의 예측 head | 내장 head |
| mlp_speculator | MLPSpeculator 호환 speculator | 필요 |
| mtp | 모델 내장 Multi-Token Prediction 레이어 재사용 | 불필요 (타깃 재사용) |
| suffix | suffix tree 기반, speculation 깊이 동적 조정 | 불필요 |

N-gram은 경량이고 피크 트래픽에서 부하를 늘리지 않지만 이득은 낮은 편입니다.
`prompt_lookup_min`과 `prompt_lookup_max`(기본 5) 파라미터로 매칭 길이를 조정합니다.
Suffix decoding은 추가 모델 없이 동작하며 `suffix_decoding_max_tree_depth`(기본 24), `suffix_decoding_min_token_prob`(기본 0.1) 등으로 제어합니다.

N-gram 방식의 offline Python 설정 예시입니다.

```python
from vllm import LLM, SamplingParams

prompts = ["The future of AI is"]
sampling_params = SamplingParams(temperature=0.8, top_p=0.95)

llm = LLM(
    model="Qwen/Qwen3-8B",
    tensor_parallel_size=1,
    speculative_config={
        "method": "ngram",
        "num_speculative_tokens": 5,
        "prompt_lookup_max": 4,
    },
)
outputs = llm.generate(prompts, sampling_params)
```

EAGLE3의 V1 서버 설정 예시입니다.

```bash
VLLM_USE_V1=1 vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --seed 42 -tp 4 \
  --speculative-config '{"model": "yuhuili/EAGLE3-LLaMA3.3-Instruct-70B", "num_speculative_tokens": 3, "method":"eagle3", "draft_tensor_parallel_size":1}'
```

## MTP: Multi-Token Prediction

### MTP가 무엇인가

MTP는 모델 자체에 내장된 next-n-token 예측 레이어(MTP 모듈)를 draft 제안자로 재사용하는 speculative decoding 방식입니다.
별도의 draft 모델을 로드하지 않고, 타깃 모델과 같은 체크포인트의 MTP head를 draft로 활용한다는 것이 핵심입니다.
vLLM 소스에서는 `method=="mtp"`이면 draft 모델을 타깃과 동일하게 설정하고 quantization도 타깃 설정을 따릅니다.

대표적인 사례가 DeepSeek 계열입니다.
DeepSeek-V3는 685B 총 파라미터 중 14B가 MTP 모듈 가중치이며, 이 모듈이 다음 토큰을 예측합니다.
vLLM은 v0.7.3에서 DeepSeek MTP를 정식 지원하기 시작했고, 인자 하나만 설정하면 DeepSeek V3/R1에서 활성화됩니다.

지원 대상은 vLLM에서 MTP를 지원하는 모델 패밀리로 한정됩니다.
DeepSeek-V3 계열(model_type이 deepseek_v3/deepseek_v32이면 deepseek_mtp), Qwen3-Next(qwen3_next_mtp), LongCat(longcat_flash_mtp), Step3.5(step3p5_mtp), XiaomiMiMo/MiMo-7B, Gemma 4 assistant 계열 등이 여기 해당합니다.
설정할 때는 `method: "mtp"`로 통일하며, model_type이 deepseek_mtp 등 MTP 계열이면 내부적으로 자동 매핑됩니다.

### 설정과 권장값

MTP의 최소 설정은 다음과 같습니다.

```json
{
  "method": "mtp",
  "num_speculative_tokens": 1
}
```

vLLM 문서의 서빙 명령 예시입니다.

```bash
# MiMo-7B (온라인)
vllm serve XiaomiMiMo/MiMo-7B-Base \
    --tensor-parallel-size 1 \
    --speculative-config '{"method":"mtp","num_speculative_tokens":1}'
```

여기서 `num_speculative_tokens=1`이 권장 기본값인 데는 명확한 이유가 있습니다.
`num_speculative_tokens > 1`이면 동일 MTP 레이어에서 여러 번 forward가 반복되어 수락률(acceptance rate)이 낮아질 수 있다는 경고가 vLLM 소스에 존재합니다.
또한 draft config의 `n_predict`(= `num_nextn_predict_layers`)와의 정합성도 지켜야 합니다.
`num_speculative_tokens`를 지정할 경우 `n_predict`로 나누어떨어져야 하며, 그렇지 않으면 ValueError가 발생합니다.
DeepSeek-V3.2(deepseek_v32)는 cudagraph와 MTP를 함께 쓰지 못해 내부적으로 `enforce_eager=True`가 강제됩니다.

성능 수치를 보면, DeepSeek 공식 리포트는 두 번째 토큰 예측 수락률 85~90%, 약 1.8배 TPS를 보고합니다.
ShareGPT 데이터셋 재현에서는 수락률 81~82.3%가 관측됐고, vLLM v0.7.3의 DeepSeek MTP는 최대 69% 속도 향상이 보고됐습니다.
배치 처리량 관점에서는 2.5~3배(수천 tok/s 대역)까지 보고된 사례가 있습니다.

주의할 점은 리서치 소스에 따라 권장 토큰 수가 갈린다는 것입니다.
vLLM 공식 MTP 문서와 소스는 안정성 측면에서 `num_speculative_tokens=1`을 기본값으로 제시합니다.
반면 DeepSeek 공식 recipe와 일부 gpt-oss 벤치는 2~3 토큰을 권장합니다.
즉 모델과 버전에 따라 최적값이 다르므로, 실제 워크로드에서 수락률을 측정하며 스위핑하는 것이 정석입니다.

## --max-num-seqs와 배치 튜닝

`--max-num-seqs`의 공식 정의는 "한 번의 엔진 iteration(스텝)에서 처리할 수 있는 최대 시퀀스(요청) 수"입니다.
continuous batching에서 스케줄러가 한 스텝에 동시에 배치할 수 있는 running 시퀀스 수의 상한, 즉 배치 크기(요청 개수) 상한 역할을 합니다.

기본값은 지정하지 않으면 usage context와 GPU 메모리에 따라 자동 결정됩니다.

| 환경 | usage context | max_num_seqs | max_num_batched_tokens |
|---|---|---|---|
| 대용량 GPU (H100/MI300x 등) | 오프라인 LLM | 1024 | 16384 |
| 대용량 GPU | 온라인 서버 | 1024 | 8192 |
| 일반 GPU | 오프라인 LLM | 256 | 8192 |
| 일반 GPU | 온라인 서버 | 256 | 2048 |

`performance_mode`가 throughput이면 사용자가 명시하지 않은 경우 두 값을 각각 2배로 증가시킵니다.

### max_num_batched_tokens와의 관계

`--max-num-batched-tokens`의 정의는 "한 스텝에서 처리 가능한 최대 토큰 수"입니다.
즉 토큰 레벨의 상한입니다.
두 파라미터는 함께 continuous batching 스케줄러의 admission control(수용 제어)을 구성합니다.
`--max-num-seqs`는 시퀀스(요청 개수) 상한, `--max-num-batched-tokens`는 토큰 상한이며, 실제 배치는 두 제약을 모두 만족해야 합니다.

vLLM 소스의 defaulting 로직은 두 값을 서로 보정합니다.
`max_num_batched_tokens`를 기본화할 때는 시퀀스 수와 컨텍스트 길이의 곱을 넘지 않도록 clamp합니다.
`max_num_seqs`를 기본화할 때는 시퀀스 수가 토큰 상한을 넘지 않도록 조정합니다.
CUDA graph 캡처 크기도 `max_num_seqs`에 연동되며, speculative decoding 사용 시 decode_query_len에 `num_speculative_tokens`가 더해져 캡처 크기 산정에 반영됩니다.

서버 튜닝 명령 예시입니다.

```bash
vllm serve <model> --max-num-seqs 512 --max-num-batched-tokens 8192
```

### 처리량-지연-메모리 트레이드오프

`--max-num-seqs`는 상한선이지 실제 동시성이 아니라는 점이 중요합니다.
실제 동시성은 가용 KV 캐시에 따라 vLLM이 동적으로 결정합니다.
튜닝 방향은 다음과 같이 정리할 수 있습니다.

| 목표 | 조정 방향 | 부작용 |
|---|---|---|
| 처리량(throughput) 우선 | max_num_seqs 증가, max_num_batched_tokens 증가 | KV 캐시 압력 증가, OOM 위험 |
| 토큰당 지연(TPOT) 감소 | max_num_seqs로 배치 크기 축소 | 대기 시간 및 TTFT 증가 |
| inter-token latency 개선 | max_num_batched_tokens 축소 (예 2048) | TTFT 저하 |
| TTFT 개선 | max_num_batched_tokens 증가 (8192 초과) | 큰 배치로 TPOT 증가 |

메모리 제약이 핵심입니다.
vLLM V1에서 KV cache 요구량은 대략 max_num_seqs와 max_model_len의 곱에 비례합니다.
이 곱이 GPU KV cache 예산 안에 들어와야 하며, 넘으면 OOM 또는 preemption(선점) 위험이 생깁니다.
KV 캐시 부족으로 preemption이 발생하면 vLLM은 요청을 선점 후 재계산하거나 CPU swap space를 사용하는데, 후자는 CPU와 GPU 간 전송 지연을 유발합니다.

실무 rule of thumb는 "OOM이 나지 않는 선에서 max_num_seqs를 최대한 높게" 잡는 것입니다.
동시성과 throughput 극대화가 목표일 때 유효한 지침입니다.
반대로 OOM이나 preemption이 자주 발생하면 max_num_seqs 또는 max_num_batched_tokens를 낮추거나 gpu_memory_utilization을 높여 KV 캐시를 확보합니다.

한 가지 알려진 함정이 있습니다.
vLLM Issue #27462에 따르면 `--max-num-seqs`가 "스케줄된 시퀀스"가 아니라 "전체 running 시퀀스"를 제한하기 때문에 pipeline parallelism(PP) 환경에서 활용률이 크게 저하될 수 있습니다.
PP를 사용한다면 이 동작에 주의해야 합니다.

## 실무 함정: 언제 오히려 느려지는가

speculative decoding은 "공짜 점심"이 아닙니다.
커뮤니티의 공통 결론은 "결과가 셋업마다 극단적으로 갈린다"는 것입니다.
모델 패밀리, 트래픽 패턴, 하드웨어, 샘플링 세팅에 따라 2배가 될 수도 0.7배가 될 수도 있습니다.

가장 대표적인 실측 반례가 vLLM Discussion #13834입니다.
A100 80GB에서 Llama 3.3 70B(AWQ)에 draft로 Llama 3.2 3B(AWQ-INT4)를 붙이고 5 spec tokens, fp8 KV cache로 구동한 사례입니다.

| 지표 | spec OFF | spec ON |
|---|---|---|
| output throughput | 95.43 tok/s | 68.19 tok/s |
| total throughput | 1020.56 tok/s | 744.77 tok/s |
| inter-token latency | 334.75ms | 2298.34ms |
| acceptance rate | - | 71~74% |

거의 모든 지표가 30~50% 악화됐고 TTFT만 개선됐습니다.
작성자는 "2배 speedup을 주장한 블로그를 보고 똑같이 세팅했는데 정반대 결과가 나왔다"며 acceptance 0.72가 오버헤드를 감당하기엔 너무 낮은 것 아닌지 자문했습니다.

이런 역효과가 나는 이유는 크게 세 가지입니다.
첫째, 수락률이 낮으면 순손실입니다.
draft가 만든 토큰이 자주 거부되면 추가 검증 forward만 낭비됩니다.
커뮤니티의 rule of thumb는 acceptance 60% 이상을 목표로 하고 50% 미만이면 이득이 상쇄될 수 있다는 것입니다.
DigitalOcean 프레임워크는 더 세분화해서 0.5 미만은 순손실이라 즉시 비활성화, 0.5~0.65는 손익분기, 0.65 이상이면 유익하다고 봅니다.

둘째, 고부하와 대배치에서 compute-bound로 전환됩니다.
배치가 64개 이상으로 커지면 autoregressive 방식은 이미 weight-read 비용을 여러 시퀀스에 분산해 roofline ridge에 근접합니다.
이 상태에서 speculative decoding은 draft 생성과 k개 추가 토큰 검증 forward 오버헤드만 더합니다.
그래서 vLLM은 `--speculative-disable-by-batch-size 32` 같은 플래그로 고동시성에서 자동 폴백을 제공합니다.
EAGLE-3는 배치 56, FastEagle은 배치 32 부근에서 피크를 찍고 그 이상에서는 이득이 줄어드는 경향이 관측됩니다.

셋째, tail latency와 워크로드 동질성 문제입니다.
TPOT은 수락률에 비례해 개선되지만 P99는 동시 부하에서 오히려 악화되는 경우가 많습니다.
온도(temperature)가 높으면(중앙값 0.7 초과) 확률 분포가 평평해져 draft 거부율이 올라가므로 speculative decoding이 비권장됩니다.
혼합 온도 배치는 스케줄러 마찰을 유발해 P50이 개선되어도 P99가 악화될 수 있습니다.
성능 측정은 반드시 production 동시성 수준에서 해야 합니다.

MTP 역시 도메인에 따라 acceptance가 하락합니다.
high-entropy 분포, 즉 롱테일 토큰이 많은 창작성 프롬프트나 특이 구문의 코드 생성에서 수락률이 떨어집니다.
버전과 구현 함정도 실재합니다.
vLLM Forum 사례에서는 num_speculative_tokens=3으로 구동했는데 메인 모델은 FULL CUDA GRAPH인 반면 drafter는 PIECEWISE CUDA GRAPH로만 동작해 성능이 제한됐습니다.
mainline vLLM v0.17.1에는 drafter(DeepSeek MTP 포함) full CUDA graph 지원이 없어 관련 PR 머지를 기다려야 한다는 답변이 달렸습니다.
극단적으로는 Step-3.5-Flash MTP에서 acceptance가 2.4~4.6%까지 떨어진 버그 사례도 있습니다.
같은 조건에서 패치된 v0.15.1은 97~100%, SGLang은 약 50%였으니, MTP를 켰는데 acceptance가 한 자릿수라면 구현이나 버전 이슈를 의심해야 합니다.

균형을 위해 반대 데이터도 짚어야 합니다.
Red Hat의 gpt-oss 벤치(2026-04)는 200 동시 요청까지도 이득이 유지되어 "spec decoding은 저부하에서만 유효하다"는 통념을 일부 반박합니다.
ShareGPT +20.7%, SWE-bench +20.5%, 피크 +27.2% output throughput을 보고했고, draft tokens는 2~3개가 최적(4개는 8% throughput 하락)이라고 정리했습니다.
결국 speculative decoding과 MTP는 저~중부하와 model-based drafter(EAGLE/MTP) 조합에서 확실한 이득을 주지만, 고부하·저acceptance·단순 방식(n-gram/suffix)에서는 역효과가 날 수 있습니다.

acceptance 지표를 비교할 때도 주의가 필요합니다.
vLLM Issue #42508에 따르면 동일 조건에서도 vLLM과 SpecForge 간 acceptance 리포트가 EAGLE3에서는 vLLM이 높고 독립 draft에서는 낮게 나오는 등 방향조차 어긋납니다.
툴마다 acceptance 정의와 검증 로직이 다르므로 숫자 절대 비교는 조심해야 합니다.
vLLM 0.9.1 이상에서는 draft acceptance rate, position별 acceptance rate, mean acceptance length(verifier forward pass당 평균 토큰) 같은 메트릭을 제공하므로 이를 모니터링하는 것이 좋습니다.

## 결론

세 파라미터는 각각 다른 층위를 조절합니다.
speculative decoding과 MTP는 "생성 알고리즘" 층위에서 지연을 줄이려는 시도이고, `--max-num-seqs`는 "스케줄링" 층위에서 처리량과 메모리를 조절합니다.

핵심 지침을 정리하면 다음과 같습니다.
MTP는 지원 모델에서 `method:"mtp"`, `num_speculative_tokens=1`로 켜는 것이 안정적인 출발점이며, 모델과 버전에 따라 2~3 토큰이 나을 수도 있으니 실측이 필요합니다.
speculative decoding은 acceptance rate를 반드시 모니터링하고, 0.5 미만이면 비활성화, 고부하에서는 batch-size 기반 자동 폴백을 활용해야 합니다.
`--max-num-seqs`는 OOM이 나지 않는 선에서 최대한 높이되, `max_num_seqs × max_model_len`이 KV 캐시 예산을 넘지 않도록 관리하고 지연이 중요하면 `max_num_batched_tokens`를 낮춥니다.
어떤 경우든 만능 숫자는 없으며, production 동시성 수준에서 직접 측정하며 튜닝하는 것이 정석입니다.

## Reference

- [vLLM Speculative Decoding 공식 문서](https://docs.vllm.ai/en/latest/features/speculative_decoding/)
- [vLLM MTP(Multi-Token Prediction) 문서](https://docs.vllm.ai/en/stable/features/speculative_decoding/mtp)
- [vLLM Optimization and Tuning 문서](https://docs.vllm.ai/en/stable/configuration/optimization/)
- [vLLM EngineArgs 문서](https://docs.vllm.ai/en/latest/configuration/engine_args.html)
- [Red Hat: Fly Eagle3 fly, faster inference with vLLM speculative decoding](https://developers.redhat.com/articles/2025/07/01/fly-eagle3-fly-faster-inference-vllm-speculative-decoding)
- [Red Hat: Performance improvements for speculative decoding with vLLM (gpt-oss)](https://developers.redhat.com/articles/2026/04/16/performance-improvements-speculative-decoding-vllm-gpt-oss)
- [DigitalOcean: Speculative Decoding vLLM Configuration Guide](https://www.digitalocean.com/community/tutorials/speculative-decoding-vllm-configuration-guide)
- [Snowflake: Fast Speculative Decoding in vLLM with Arctic Inference](https://www.snowflake.com/en/engineering-blog/fast-speculative-decoding-vllm-arctic/)
- [vLLM Discussion #13834: Less token with speculative decoding on](https://github.com/vllm-project/vllm/discussions/13834)
- [vLLM Issue #8439: why speculative decoding is slower than normal](https://github.com/vllm-project/vllm/issues/8439)
- [vLLM Issue #38339: Step-3.5-Flash MTP low acceptance](https://github.com/vllm-project/vllm/issues/38339)
- [vLLM Issue #42508: acceptance metric mismatch (Qwen3-32B)](https://github.com/vllm-project/vllm/issues/42508)
- [vLLM Issue #27462: max-num-seqs limits total running sequences (PP)](https://github.com/vllm-project/vllm/issues/27462)
- [vLLM Issue #12181: Multi-Token Prediction (MTP) feature request](https://github.com/vllm-project/vllm/issues/12181)
- [vLLM Forum: DeepSeek MTP full CUDA graph support](https://discuss.vllm.ai/t/deepseek-mtp-full-cuda-graph-support/2548)
- [DeepSeek-V3 Technical Report (arXiv)](https://arxiv.org/pdf/2412.19437)
- [DeepSeek-V3 Model Card (Hugging Face)](https://huggingface.co/deepseek-ai/DeepSeek-V3)
- [Introl: Speculative Decoding LLM Inference Speedup Guide 2025](https://introl.com/blog/speculative-decoding-llm-inference-speedup-guide-2025)
- [Kaige Yang: vLLM Throughput Optimization Basics](https://medium.com/@kaige.yang0110/vllm-throughput-optimization-1-basic-of-vllm-parameters-c39ace00a519)
