---
layout: post
title: "DeepSeek DeepInfer - 프로덕션 추론 프레임워크 오픈소스 공개"
author: 'Juho'
date: 2026-10-20 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, vLLM, Benchmark, Evaluation]
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
2. [배경](#배경)
3. [핵심 기능](#핵심-기능)
   - [커널과 병렬화](#커널과-병렬화)
   - [DSpark 투기적 디코딩](#dspark-투기적-디코딩)
   - [Decoder SWA Bounded Replay](#decoder-swa-bounded-replay)
   - [비동기 스케줄링과 재현성](#비동기-스케줄링과-재현성)
   - [Structured Outputs](#structured-outputs)
4. [설치와 실행](#설치와-실행)
5. [벤치마크와 평가](#벤치마크와-평가)
   - [성능 벤치마크](#성능-벤치마크)
   - [정확도 평가](#정확도-평가)
6. [이번 릴리스에 포함되지 않은 것](#이번-릴리스에-포함되지-않은-것)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

DeepSeek이 자사의 대규모 프로덕션 서비스를 구동하는 고성능 추론 프레임워크 [DeepInfer](https://github.com/deepseek-ai/DeepInfer){:target="_blank"}를 Apache License 2.0으로 공개했습니다.
이번 릴리스는 오프라인 추론(offline inference)을 대상으로 합니다.
지원 모델은 DeepSeek-V4.1-Flash이며, NVIDIA SM100 GPU와 Huawei Ascend 950 NPU에서 동작합니다.

| 항목 | 내용 |
|------|------|
| 성격 | DeepSeek 프로덕션 서비스용 고성능 추론 프레임워크 |
| 릴리스 범위 | 오프라인 추론 |
| 지원 모델 | DeepSeek-V4.1-Flash |
| 지원 하드웨어 | NVIDIA SM100 GPU, Huawei Ascend 950 NPU |
| 라이선스 | Apache License 2.0 |
| 기반 | 주로 vLLM 기반, SGLang과 TensorRT-LLM에서 아이디어 차용 |

## 배경

DeepInfer는 초기 구현을 주로 vLLM 위에 구축했고, SGLang과 TensorRT-LLM으로부터 영감을 받았다고 밝히고 있습니다.
여기에 DeepSeek이 그동안 공개해 온 커널 프로젝트인 FlashMLA, DeepGEMM, DeepEP, TileKernels, DeepSelect와 자체 TileLang 커널을 결합했습니다.
즉, 개별적으로 공개되던 커널 라이브러리들이 실제 서비스용 엔진 안에서 어떻게 조합되는지를 확인할 수 있는 저장소입니다.

저장소의 주요 디렉토리 구성은 다음과 같습니다.

| 디렉토리 | 역할 |
|----------|------|
| csrc | C/C++ 소스 |
| deep_cluster | 클러스터 관련 코드 |
| deepinfer | 프레임워크 본체 (TileLang 커스텀 커널 포함) |
| doc | 문서와 벤치마크 이미지 |
| eval | 정확도 평가 스크립트 |
| examples | 오프라인 추론, 벤치마크, 스윕 예제 |
| scripts, tests, vendor | 보조 스크립트, 테스트, 서드파티 의존성 |

## 핵심 기능

### 커널과 병렬화

모델, 샘플러 런타임과 함께 최적화된 커널이 포함되어 있습니다.
커널은 FlashMLA, DeepGEMM, DeepEP, TileKernels, DeepSelect, 그리고 `deepinfer/model_executor/layers/tilelang_ops`에 위치한 커스텀 TileLang 커널을 기반으로 합니다.

병렬화는 다음 두 축을 지원합니다.

| 영역 | 지원 방식 |
|------|-----------|
| Attention | Attention DP, balanced CP |
| MoE | Expert Parallelism(EP) + EPLB |

### DSpark 투기적 디코딩

DSpark와 confidence 기반 검증 스케줄링(confidence-scheduled verification)으로 draft 토큰을 활용한 투기적 디코딩(speculative decoding)을 수행합니다.
`DEEPINFER_ENABLE_MTP_VARLEN=1`로 활성화합니다.
`DEEPINFER_USE_NEXTN_DECODE`는 최대 draft 길이를 지정하며 기본값은 5입니다.

### Decoder SWA Bounded Replay

길이가 제한된 접미부(suffix)만 다시 재생(replay)하여 prefill 작업량을 줄이는 기능입니다.
V4.1 기본 설정은 다음과 같습니다.

```bash
DEEPINFER_YOCO_PREFILL_TRUNCATION_MODE=compact
DEEPINFER_YOCO_PREFILL_KEEP_TOKENS=140
```

유지되는 접미부는 SWA 윈도우와 최대 백트래킹 길이를 합한 범위를 모두 포함해야 합니다.
mask 방식과 compact 방식의 동등성 검증은 `examples/yoco/README.md`에 정리되어 있습니다.

### 비동기 스케줄링과 재현성

비동기 스케줄링은 CPU 측 배치 스케줄링과 디바이스 실행을 겹쳐서 수행하며, 기본으로 켜져 있습니다.

재현성 관련 기능은 NVIDIA에서만 지원됩니다.

| 기능 | 설명 | 활성화 조건 |
|------|------|-------------|
| Batch-invariant inference | 배치 구성과 무관하게 동일한 결과 | DEEPINFER_BATCH_INVARIANT=1 (NVIDIA 전용) |
| Deterministic sampling | 같은 입력 토큰과 샘플링 파라미터에 같은 출력 | DEEPINFER_BATCH_INVARIANT=1, DEEPINFER_ENABLE_MTP_VARLEN=0, SamplingParams에 고정 seq_seed (NVIDIA 전용) |

결정적 샘플링을 쓰려면 투기적 디코딩(MTP varlen)을 꺼야 한다는 점에 유의해야 합니다.

### Structured Outputs

`SamplingParams`의 `so_args`를 통해 정규식(regex), JSON schema, EBNF 문법으로 생성을 제약할 수 있습니다.
예제는 `examples/_structured_output.py`에 있습니다.
`examples/offline_inference.py`에 `--check_structured_output`을 주면 regex, JSON, EBNF 검사와 정상적인 EOS 종료 여부를 확인합니다.

## 설치와 실행

사전 요구사항은 다음과 같습니다.

| 항목 | 요구사항 |
|------|----------|
| OS | Linux |
| 컴파일러 | GCC 13.0 이상 |
| Python | 3.12 |
| NVIDIA | CUDA 13 |
| Ascend | CANN 9.2 |

설치 절차는 비어 있는 Python 3.12 환경을 만든 뒤, 가속기를 지정하고 빌드 스크립트를 실행하는 방식입니다.
`DEEPINFER_ACCELERATOR`는 이후 평가 스크립트에서도 쓰이므로 export된 상태로 유지해야 합니다.

```bash
# nvidia 또는 ascend
export DEEPINFER_ACCELERATOR=nvidia
./build.sh

python examples/offline_inference.py
```

기본 체크포인트인 `deepseek-ai/DeepSeek-V4.1-Flash-1017`은 첫 실행 시 Hugging Face 캐시로 다운로드됩니다.
로컬 사본을 쓰려면 `DEEPINFER_HF_MODEL_DIR`을 지정합니다.
`offline_inference.py`는 `--ep`, `--dp`, `--cp` 같은 병렬화 플래그를 받으며, 전체 인자는 `--help`로 확인할 수 있습니다.

Ascend NPU 디버깅 시에는 `ASCEND_LAUNCH_BLOCKING=1`로 연산자를 동기 실행해 읽기 쉬운 스택 트레이스를 얻고, `ASCEND_SLOG_PRINT_TO_STDOUT=1`로 CANN 로그를 표준 출력에 찍을 수 있습니다.

## 벤치마크와 평가

### 성능 벤치마크

README에는 prefill과 decode의 Pareto 차트가 실려 있습니다.
prefill은 캐시되지 않은 입력 처리량 대비 지연 시간, decode는 출력 처리량 대비 사용자당 속도를 보여줍니다.
측정 환경은 prefill이 SM100 GPU 4장, decode가 SM100 GPU 64장이며, 각각 하나의 scale-up 도메인 안에서 측정되었습니다.
재현 스크립트는 `examples/sweep/run_sweep.sh`입니다.

Prefill 차트는 DeepSeek V4.1 Flash를 EP4DP4, 384+32 experts, 컨텍스트 131,072, Decoder Bounded Replay(140 tokens) 설정으로 측정한 결과입니다.
각 점은 DP rank당 prefill_batch_size이며, 수치는 차트에서 읽은 근사치입니다.

| prefill_batch_size | 지연 (ms/prefill step) | 비캐시 입력 처리량 (tokens/s/GPU) |
|------|------|------|
| 4,096 | 약 70 | 약 60,000 |
| 8,192 | 약 115 | 약 71,500 |
| 16,384 | 약 200 | 약 81,500 |
| 32,768 | 약 375 | 약 87,000 |
| 65,536 | 약 730 | 약 89,500 |

배치를 키울수록 GPU당 처리량은 약 9만 tokens/s 근처에서 포화되고, 지연 시간은 배치 크기에 비례해 늘어납니다.

Decode 차트는 EP64DP64, 384+64 experts, 컨텍스트 131,072, nextn=3 고정, 토큰당 수락률 75%(τ=3.25) 조건입니다.
각 점은 DP rank당 decode_num_requests입니다.

| decode_num_requests | 사용자당 속도 (tokens/s/user) | 출력 처리량 (tokens/s/GPU) |
|------|------|------|
| 8 | 약 325 | 약 2,600 |
| 16 | 약 315 | 약 5,000 |
| 32 | 약 297 | 약 9,500 |
| 64 | 약 246 | 약 15,700 |
| 128 | 약 179 | 약 22,800 |
| 256 | 약 103 | 약 26,300 |

동시 요청을 늘리면 GPU당 출력 처리량은 약 26,000 tokens/s까지 올라가지만, 사용자당 속도는 약 325에서 약 100 tokens/s로 떨어지는 전형적인 처리량-상호작용성 트레이드오프를 보여줍니다.

`examples/benchmark.py`는 prefill과 decode를 따로 설정하며 기본값은 다음과 같습니다.

| 파라미터 | 기본값 |
|----------|--------|
| decode_num_requests | 64 |
| context_length | 32768 |
| non_cached_tokens | 4096 |
| prefill_batch_size | 32768 |
| nextn | 5 |
| 중복 expert 수 (DEEPINFER_OVERRIDE_NUM_EXTRA_EXPERTS) | 32 (ep_size로 나누어떨어지도록 올림) |

보고되는 지표는 prefill 처리량(tokens/s/GPU), decode avg_SPS(steps/s), decode 처리량(tokens/s/GPU, 전체 수락 기준 계산)입니다.
병렬 규모는 환경에서 결정되며 `ep_size = WORLD_SIZE × len(CUDA_VISIBLE_DEVICES)`이고, `--cp`는 `ep_size`의 약수여야 합니다.

```bash
python examples/benchmark.py --cp 8 --prefill_batch_size 65536
python examples/benchmark.py --decode_num_requests 1
python examples/benchmark.py --prompt_mode image
```

`run_sweep.sh`는 decode, prefill, encoder 모드를 지원하며 결과를 `sweeps/<tag>/results.jsonl`에 기록합니다.
`sweep/plot_pareto.py`로 처리량-지연 곡선을 그리고, `sweep/ve_sweep.py`로 비전 인코더만 따로 시간을 잴 수 있습니다.

### 정확도 평가

`eval/` 디렉토리는 배포한 엔진의 정확도를 확인하기 위한 세 가지 벤치마크를 제공합니다.

| 벤치마크 | 문항 수 | 문항당 샘플 | Pass@1 산출 방식 |
|----------|---------|-------------|------------------|
| AIME 2026 | 30 | 32 | 32개 샘플 평균 정확도 |
| HLE | 2,500 (텍스트 2,158, 이미지 342) | 1 | 단일 샘플 정확도 |
| GPQA Diamond | 198 | 8 | 8개 샘플 평균 정확도 |

생성과 채점은 분리되어 있어, 한 번 생성한 출력을 다시 생성하지 않고 재채점할 수 있습니다.
채점 모드는 judge 모델이 정답 일치를 판단하는 llm(기본)과 결정적 정확 일치를 쓰는 rule 두 가지입니다.
rule 모드는 AIME에서 `\boxed{}` 안의 정수를, GPQA에서 `ANSWER:` 뒤의 문자를 추출합니다.
HLE는 자유 서술형 답이 많아 LLM judge만 지원합니다.
AIME 2026에서는 두 모드가 960개 rollout 모두에서 일치했다고 합니다.

```bash
export EVAL_JUDGE_BASE_URL=https://api.deepseek.com
export EVAL_JUDGE_API_KEY=<key>
export EVAL_JUDGE_MODEL=deepseek-v4-flash
bash eval/aime/run.sh

# judge 없이 rule 기반 재채점
python eval/score.py --out-dir eval/outputs/aime/<TAG> --judge rule
```

V4.1 기본 설정 기준 참고 결과는 다음과 같습니다.

| 벤치마크 | Pass@1 |
|----------|--------|
| AIME 2026 | 99.9% |
| GPQA Diamond | 93.4% |
| HLE-Full | 42.6% |

생성이 확률적이므로 반복 실행 시 수치가 약간 달라질 수 있으며, 정확한 값이라기보다 배포 상태 점검용 기준으로 보라고 안내합니다.
`summary.json`의 `coverage_complete`가 `true`일 때만 Pass@1 결과가 유효합니다.

## 이번 릴리스에 포함되지 않은 것

다음 구성 요소는 이번 릴리스에 포함되지 않았습니다.

| 미포함 항목 | 비고 |
|-------------|------|
| KV cache 시스템 | 별도 구축 필요 |
| VE/Prefill/Decode 분리(disaggregation) | 필요한 엔진 기능 대부분은 이미 구현됨 |
| prefill/decode 데이터 병렬 복제본 간 로드 밸런싱 | 별도 구축 필요 |

따라서 완전한 온라인 서비스를 구성하려면 이 부분들을 직접 구현해야 합니다.

## 결론

DeepInfer는 DeepSeek이 개별 공개해 온 FlashMLA, DeepGEMM, DeepEP 등의 커널을 vLLM 기반 엔진에 통합한 실제 프로덕션 추론 프레임워크입니다.
DSpark 투기적 디코딩, SWA bounded replay, 비동기 스케줄링, 배치 불변 추론과 결정적 샘플링, structured outputs까지 서비스 운영에 필요한 기능을 갖추고 있습니다.
다만 이번 릴리스는 오프라인 추론에 한정되며, 지원 모델은 DeepSeek-V4.1-Flash, 하드웨어는 NVIDIA SM100과 Ascend 950으로 좁습니다.
KV cache 시스템, PD 분리, 복제본 간 로드 밸런싱이 빠져 있으므로 온라인 서빙에 바로 쓰기보다는 대규모 MoE 추론 엔진의 구조를 참고하고 성능과 정확도를 재현해 보는 용도로 접근하는 것이 적절합니다.

## Reference

- [deepseek-ai/DeepInfer - GitHub](https://github.com/deepseek-ai/DeepInfer)
- [DeepInfer eval README](https://github.com/deepseek-ai/DeepInfer/blob/main/eval/README.md)
- [DeepInfer examples README](https://github.com/deepseek-ai/DeepInfer/blob/main/examples/README.md)
