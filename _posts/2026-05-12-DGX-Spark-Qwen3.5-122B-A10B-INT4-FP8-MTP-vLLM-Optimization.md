---
layout: post
title: "DGX Spark에서 Qwen3.5-122B-A10B 추론 80% 가속: INT4+FP8 하이브리드와 MTP-2 투기적 디코딩"
author: 'Juho'
date: 2026-05-12 00:00:00 +0900
categories: [LLM]
tags: [LLM, vLLM, GPU, Benchmark]
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
2. [최적화 결과](#최적화-결과)
   - [성능 요약](#성능-요약)
   - [256K 컨텍스트 지원](#256k-컨텍스트-지원)
3. [구현 단계](#구현-단계)
   - [Step 0-2 호스트 준비](#step-0-2-호스트-준비)
   - [Step 3 SM121용 vLLM 빌드](#step-3-sm121용-vllm-빌드)
   - [Step 4-5 v2 이미지와 실행](#step-4-5-v2-이미지와-실행)
4. [핵심 패치 분석](#핵심-패치-분석)
   - [INT4+FP8 하이브리드 체크포인트](#int4fp8-하이브리드-체크포인트)
   - [MTP-2 투기적 디코딩](#mtp-2-투기적-디코딩)
   - [INT8 LM Head Triton 커널](#int8-lm-head-triton-커널)
5. [주의사항](#주의사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

NVIDIA DGX Spark 단일 노드에서 Qwen3.5-122B-A10B 추론 속도를 28.3 tok/s에서 51 tok/s로 80% 끌어올린 최적화 묶음이 공개되었다.
같은 패치는 동일한 아키텍처의 더 작은 모델인 Qwen3.5-35B-A3B에서도 112 tok/s를 기록했다.
품질 저하 없이 256K 컨텍스트까지 단일 사용자 기준으로 지원한다.

이 프로젝트는 vLLM 0.19.1을 SM121(Blackwell) 아키텍처용으로 다시 빌드하고, INT4+FP8 하이브리드 가중치와 MTP-2(Multi-Token Prediction) 투기적 디코딩, INT8 LM Head Triton 커널을 결합한 결과다.
모든 최적화는 독립적으로 적용 가능하다.

## 최적화 결과

### 성능 요약

각 최적화 단계별 누적 효과는 다음과 같다.

| Configuration | tok/s | Improvement | Build |
|---|---|---|---|
| Baseline (vLLM 0.19 + AutoRound INT4 + FlashInfer) | 28.3 | -- | -- |
| + Hybrid INT4+FP8 Dense Layers | 30.8 | +8.8% | step 1 |
| + MTP-2 Speculative Decoding | 38.4 | +35.7% | step 2 |
| v2 (+ INT8 LM Head v2) | 51 | +80% | Dockerfile.v2 |
| v2-tq (+ TurboQuant KV Cache) | 39 | +38% | Dockerfile.v2-tq |

베이스라인 대비 80% 향상의 핵심은 INT8 LM Head v2다.
TurboQuant 변형은 단일 사용자 처리량은 떨어지지만 KV 캐시 용량을 4배로 늘려 동시 사용자 수를 확장하는 용도다.

### 256K 컨텍스트 지원

v2는 별도의 KV 압축 없이도 256K 컨텍스트를 지원한다.
TurboQuant 변형을 사용하면 동일한 컨텍스트 길이에서 동시 사용자를 5명까지 늘릴 수 있다.

| Config | KV Cache | Concurrent Users at 256K |
|---|---|---|
| v2 (standard) | 355K tokens | 1 |
| v2-tq (TurboQuant) | 1.4M tokens | 5 |

## 구현 단계

전체 워크플로우는 0부터 5단계로 나뉘며, 자동화 스크립트인 `install.sh`로 0-4단계를 한 번에 실행할 수도 있다.
스크립트는 idempotent해서 이미 끝난 단계는 건너뛴다.

### Step 0-2 호스트 준비

호스트에서 모델을 다운로드하고 가중치를 변환하는 단계다.

```bash
# Step 0: Intel AutoRound INT4 모델 다운로드
hf download Intel/Qwen3.5-122B-A10B-int4-AutoRound
INTEL_DIR=$(find ~/.cache/huggingface/hub/models--Intel--Qwen3.5-122B-A10B-int4-AutoRound/snapshots -maxdepth 1 -mindepth 1 -type d)

# Step 1: 하이브리드 체크포인트 빌드 (선택, +9%)
python patches/01-hybrid-int4-fp8/build-hybrid-checkpoint.py \
    --gptq-dir "$INTEL_DIR" \
    --fp8-repo Qwen/Qwen3.5-122B-A10B-FP8 \
    --output ~/models/qwen35-122b-hybrid-int4fp8 \
    --force
```

Step 2에서는 Intel AutoRound 패키지에 물리적으로는 포함되어 있지만 인덱스에 등록되지 않은 MTP 가중치 785개 텐서를 `model.safetensors.index.json`에 다시 매핑한다.
이 작업이 없으면 vLLM이 인덱스만 읽기 때문에 4.8 GB MTP 헤드를 영영 보지 못한다.

### Step 3 SM121용 vLLM 빌드

DGX Spark는 SM121(Blackwell) 아키텍처를 요구한다.
PyPI에서 받는 사전 빌드 wheel로는 동작하지 않으므로 [eugr/spark-vllm-docker](https://github.com/eugr/spark-vllm-docker){:target="_blank"} 저장소의 특정 커밋을 기준으로 직접 컴파일한다.

```bash
git clone https://github.com/eugr/spark-vllm-docker.git
cd spark-vllm-docker
git checkout 49d6d9fefd7cd05e63af8b28e4b514e9d30d249f

# 두 개의 TEMPORARY PATCH RUN 블록 제거
sed -i '/# TEMPORARY PATCH for broken FP8 kernels/,/&& rm pr35568.diff/d' Dockerfile
sed -i '/# TEMPORARY PATCH for broken compilation/,/&& rm pr38919.diff/d' Dockerfile

./build-and-copy.sh -t vllm-sm121 --vllm-ref v0.19.0 --tf5
```

이 단계가 가장 까다롭다.
업스트림 Dockerfile은 두 단계(builder, runner)에서 PyTorch nightly를 따로 설치하는데, 핀이 없으면 두 호출이 같은 날의 다른 nightly로 해결되어 ABI 불일치가 이미지에 박힌다.
증상은 시작 시 `ImportError: undefined symbol: _ZN2at4cuda24getCurrentCUDABlasHandleEv` 형태로 나타나며, PyTorch가 `at::cuda::getCurrentCUDABlasHandle` 시그니처를 nightly 사이에 변경했기 때문이다.

검증된 조합은 vLLM 0.19.1(`2a69949bd` 커밋), PyTorch `2.12.0.dev20260408+cu130`, 동일 날짜의 torchvision/torchaudio다.

### Step 4-5 v2 이미지와 실행

마지막은 v2 이미지를 빌드하고 컨테이너를 띄우는 단계다.

```bash
docker build -t vllm-qwen35-v2 -f docker/Dockerfile.v2 .

docker run -d --name vllm-qwen35 \
  --gpus all --net=host --ipc=host \
  -v ~/models:/models \
  vllm-qwen35-v2 \
  serve /models/qwen35-122b-hybrid-int4fp8 \
  --served-model-name qwen \
  --port 8000 \
  --max-model-len 262144 \
  --gpu-memory-utilization 0.90 \
  --reasoning-parser qwen3 \
  --attention-backend FLASHINFER \
  --speculative-config '{"method":"mtp","num_speculative_tokens":2}'
```

서버가 모델을 로딩하고 워밍업하는 데 약 10분이 걸린다.
이후 `/health` 엔드포인트와 OpenAI 호환 `chat/completions`로 검증한다.

## 핵심 패치 분석

### INT4+FP8 하이브리드 체크포인트

Intel AutoRound INT4 체크포인트의 BF16 공유 전문가(shared expert) 가중치를 공식 Qwen FP8 체크포인트의 값으로 교체한다.
변환에는 약 20분이 걸리고 결과는 71 GB다.
이 단계만으로 약 9% 속도 향상이 발생한다.

Docker 이미지는 하이브리드와 비하이브리드 체크포인트 모두 지원한다.
FP8 가중치가 없으면 FP8 디스패치가 활성화되지 않을 뿐이다.

### MTP-2 투기적 디코딩

vLLM의 `--speculative-config '{"method":"mtp","num_speculative_tokens":2}'` 옵션을 켜면 MTP 헤드가 2개 토큰을 투기적으로 생성하고 본 모델이 검증한다.
약 80%의 수락률을 보이며 단독으로 약 35.7% 속도 향상을 가져온다.

핵심은 Step 2에서 추가한 MTP 가중치 매핑이다.
이 매핑이 없으면 MTP 옵션을 켜도 가중치가 로드되지 않아 효과가 없다.

### INT8 LM Head Triton 커널

v2 이미지의 핵심은 INT8 LM Head Triton 커널이다.
이 단독 패치만으로 38.4 tok/s에서 51 tok/s로 점프한다.
하이브리드 단계를 건너뛰어도 INT8 LM Head + MTP 조합으로 약 44 tok/s를 얻을 수 있다.

## 주의사항

| Symptom | Fix |
|---|---|
| `health` returns nothing | 약 10분 로딩 대기 |
| Garbage output | 패치된 이미지인지 확인, vanilla vLLM 사용 금지 |
| OOM at startup | `--gpu-memory-utilization`을 0.85로 낮춤 |
| `content: null` | thinking 모델의 정상 동작, 응답은 `reasoning` 필드에 위치 |
| 약 38 tok/s에서 멈춤 | `--speculative-config`의 `num_speculative_tokens:2` 확인 |

`--enable-prefix-caching`은 의도적으로 제외했다.
Qwen3.5의 DeltaNet 하이브리드 어텐션과 충돌하여 크래시한다.

vLLM 0.19에는 알려진 이슈가 있다.
`--reasoning-parser qwen3`와 `--tool-call-parser`가 동시에 활성화되면 `<think>` 블록 내부에서 발생한 도구 호출이 비스트리밍 모드에서 조용히 사라질 수 있다(vllm#39056).

도구 호출 파서는 모델별로 다르다.

| Parser | Format | Models |
|---|---|---|
| qwen3_xml | tool_call 태그 안 JSON | Qwen3.5, Qwen3-Instruct |
| qwen3_coder | 커스텀 XML 형식 | Qwen3-Coder 시리즈 |

Qwen3.5-122B에는 `qwen3_xml`을 사용해야 한다.

## 결론

이 프로젝트는 단일 DGX Spark에서 122B MoE 모델을 51 tok/s로 서빙하는 것이 가능하다는 점을 입증한다.
80% 향상의 본질은 INT8 LM Head 커널이고, MTP-2가 그다음 비중을 차지한다.
하이브리드 가중치는 약 9%로 가장 작지만 가장 손이 많이 가는 단계다.

패치는 vLLM 0.19.1과 특정 PyTorch nightly에 강하게 결합되어 있다.
다른 vLLM 버전으로 옮기려면 재테스트가 필수다.
프로덕션 운영을 고려한다면 GitHub의 `run-qwen.sh` 예시를 출발점으로 삼아 `--load-format fastsafetensors`, `--enable-chunked-prefill` 같은 플래그를 더해 나가면 된다.

## Reference

- [albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4](https://github.com/albond/DGX_Spark_Qwen3.5-122B-A10B-AR-INT4/)
