---
layout: post
title: "Qwen3.8 시리즈 정리 - 27B와 2.4T-A95B의 아키텍처부터 배포까지"
author: 'Juho'
date: 2026-08-16 00:00:00 +0900
categories: [LLM]
tags: [LLM, Benchmark, vLLM, GPU]
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
2. [라인업과 라이선스](#라인업과-라이선스)
   - [공식 오픈웨이트 리포지토리 4개](#공식-오픈웨이트-리포지토리-4개)
   - [27B와 2.4T-A95B의 대비](#27b와-24t-a95b의-대비)
   - [오픈웨이트판과 Qwen3.8-Max의 기능 격차](#오픈웨이트판과-qwen38-max의-기능-격차)
   - [모델별로 갈린 라이선스](#모델별로-갈린-라이선스)
3. [아키텍처 내부 구조](#아키텍처-내부-구조)
   - [하이브리드 레이어 구성](#하이브리드-레이어-구성)
   - [공통 하이퍼파라미터와 갈라지는 지점](#공통-하이퍼파라미터와-갈라지는-지점)
   - [thinking 제어와 reasoning effort](#thinking-제어와-reasoning-effort)
4. [컨텍스트 길이](#컨텍스트-길이)
   - [네이티브 262144와 확장 최대치](#네이티브-262144와-확장-최대치)
   - [YaRN 설정과 트레이드오프](#yarn-설정과-트레이드오프)
5. [벤치마크](#벤치마크)
   - [2.4T-A95B 공식 수치](#24t-a95b-공식-수치)
   - [27B 공식 수치](#27b-공식-수치)
   - [평가 조건과 harness 비대칭](#평가-조건과-harness-비대칭)
   - [독립 평가 결과](#독립-평가-결과)
6. [실행과 배포](#실행과-배포)
   - [하드웨어 요구량](#하드웨어-요구량)
   - [vLLM 실행 명령](#vllm-실행-명령)
   - [Transformers와 OpenAI SDK 호출](#transformers와-openai-sdk-호출)
   - [MTP speculative decoding](#mtp-speculative-decoding)
   - [양자화 선택지](#양자화-선택지)
   - [권장 샘플링 파라미터](#권장-샘플링-파라미터)
   - [커뮤니티 실측 처리량](#커뮤니티-실측-처리량)
   - [채팅 템플릿 예외로 인한 연동 실패](#채팅-템플릿-예외로-인한-연동-실패)
7. [커뮤니티 평가](#커뮤니티-평가)
   - [과도한 thinking](#과도한-thinking)
   - [Dense 27B의 구조적 페널티](#dense-27b의-구조적-페널티)
   - [벤치마크 회의론](#벤치마크-회의론)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Qwen3.8은 2026년 7월 19일 Max 프리뷰, 8월 3일 Max 정식 API 출시, 8월 12일 Max 오픈웨이트 공개, 8월 13~14일 27B 오픈웨이트 공개 순으로 풀렸다.
이름은 하나지만 실제로 손에 잡히는 산출물은 성격이 크게 다른 두 모델과 API 전용 한 모델로 갈라진다.
[Qwen3.8-27B](https://huggingface.co/Qwen/Qwen3.8-27B){:target="_blank"}는 27B Dense에 비전 인코더가 붙은 단일 GPU급 모델이고, [Qwen3.8-2.4T-A95B](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B){:target="_blank"}는 총 2.4T 파라미터 중 95B가 활성화되는 텍스트 전용 MoE다.
여기에 Qwen Cloud에서만 접근 가능한 Qwen3.8-Max가 별도로 존재한다.

이 글은 패밀리 단위 조망과 개별 모델 카드의 세부 사양을 한자리에 모으는 것을 목적으로 한다.
라인업 구성과 라이선스, 두 모델이 공유하는 아키텍처와 갈라지는 지점, 컨텍스트 확장, 공식 벤치마크와 그 각주, 실제 서빙 구성과 호출 코드, 그리고 커뮤니티 실측과 비판까지 순서대로 다룬다.
수치는 2026년 8월 16일 기준으로 확인된 1차 소스의 원 수치를 그대로 옮겼다.

## 라인업과 라이선스

### 공식 오픈웨이트 리포지토리 4개

Hugging Face의 Qwen 조직이 배포한 Qwen3.8 리포지토리는 네 개가 전부다.
Instruct와 Thinking을 나눈 별도 리포는 존재하지 않으며, thinking 제어는 요청 단위 파라미터로 처리된다.
사전학습 Base 체크포인트도 공개되지 않았고 두 카드 모두 post-trained 가중치라고 명시한다.

| 리포지토리 | 성격 | 라이선스 | 리포 총 용량 | 공개일 |
|------------|------|----------|--------------|--------|
| Qwen/Qwen3.8-27B | Dense 27B, 비전 포함 | Apache-2.0 | 55.59 GB | 2026-08-05 |
| Qwen/Qwen3.8-27B-FP8 | 27B FP8 양자화 | Apache-2.0 | 30.89 GB | 2026-08-13 |
| Qwen/Qwen3.8-2.4T-A95B | MoE, 텍스트 전용 | Qwen3.8-Max License | 4,892.39 GB | 2026-08-08 |
| Qwen/Qwen3.8-2.4T-A95B-FP8 | MoE FP8 양자화 | Qwen3.8-Max License | 2,496.15 GB | 2026-08-08 |

두 모델의 model_type은 각각 qwen3_5, qwen3_5_moe_text이고 architectures는 Qwen3_5ForConditionalGeneration, Qwen3_5MoeForCausalLM이다.
즉 Qwen3.8은 Qwen3.5 아키텍처 클래스를 그대로 재사용하며, 두 카드 모두 "Built on the architectural foundation of Qwen3.5"라고 서술한다.

채택 지표는 27B 쪽에 몰려 있다.
27B 카드 기준 월 다운로드 91,917회이며, 파생 양자화 모델 386개와 fine-tuning 모델 63개가 등록되어 있다.

### 27B와 2.4T-A95B의 대비

라인업에 중간 규모가 없다는 점이 이 시리즈의 첫 번째 특징이다.
단일 GPU에 올라가는 27B와 최소 8 GPU를 요구하는 2.4T 사이가 비어 있다.
두 번째 특징은 작은 쪽이 멀티모달이고 큰 쪽이 텍스트 전용이라는 역전 구조다.

| 항목 | Qwen3.8-27B | Qwen3.8-2.4T-A95B |
|------|-------------|-------------------|
| 구조 | Dense | MoE (총 2.4T / 활성 95B) |
| 이미지·비디오 입력 | 지원 | 미지원 |
| thinking 비활성화 | 가능 | 불가 |
| 라이선스 | Apache-2.0 | Qwen3.8-Max License |
| 레이어 수 | 64 | 92 |
| Hidden Dimension | 5120 | 8192 |
| BF16 가중치 | 55.59 GB | 약 4.45 TiB |

2.4T-A95B 카드는 "Qwen3.8-2.4T-A95B is a text-only model ... Multimodal inputs are not supported"라고 명시한다.
thinking 역시 끌 수 없어 모든 응답이 reasoning 블록으로 시작한다.
반대로 27B는 thinking을 요청마다 끌 수 있고 non-thinking 전용 샘플링 파라미터 세트가 별도로 제시된다.

### 오픈웨이트판과 Qwen3.8-Max의 기능 격차

공개된 2.4T-A95B는 Qwen3.8-Max와 같은 모델이 아니다.
모델 카드가 직접 이 점을 밝힌다.

> "Qwen3.8-Max is the official version based on Qwen3.8-2.4T-A95B with more features, such as vision input & non-thinking support, 1M context length by default, official built-in tools, etc."

즉 오픈웨이트판에서는 비전 입력, non-thinking 모드, 기본 1M 컨텍스트, 공식 빌트인 도구가 빠져 있다.
Qwen Cloud의 qwen3.8-max 사양은 다음과 같다.

| 항목 | 값 |
|------|-----|
| 컨텍스트 | 1M 토큰 |
| 입력 모달리티 | Image, Text, Video |
| 최대 입력 | 991K 토큰 (thinking 시 983K) |
| 최대 출력 | 131K 토큰 |
| 최대 reasoning | 262K 토큰 |
| 입력 가격 | 1M 토큰당 $2 |
| 출력 가격 | 1M 토큰당 $6 |
| 암묵 캐시 | $0.25 |
| 명시 캐시 생성 / 읽기 | $2.5 / $0.17 |
| 내장 도구 | code interpreter, web extractor, web search, image search |

27B의 Qwen Cloud 호스팅판은 카드에 "The service is coming soon"으로만 예고되어 있고 해당 모델 페이지는 현재 404다.
Qwen Cloud 모델 목록에서 Qwen3.8 계열은 Qwen3.8-Max 하나뿐이며 plus, flash, coder, vl 같은 파생 API 모델은 목록에 없다.

가중치 공개판이 호스팅 API보다 기능이 적다는 점은 공개 직후 지적 대상이 됐다.
비전을 제거한 것이 클라우드 엔드포인트로 유도하려는 조치라는 비판, 27B까지 포함해 두 모델을 아예 공개하지 않을 수도 있었으므로 공개 자체는 평가해야 한다는 반론이 함께 나왔다.

### 모델별로 갈린 라이선스

패밀리 전체가 Apache 2.0이라는 서술은 정확하지 않다.
27B와 27B-FP8만 Apache-2.0이고, 2.4T-A95B와 그 FP8판은 Qwen3.8-Max License라는 커스텀 라이선스를 따른다.

Apache-2.0인 27B 쪽에는 매출 임계값도 표기 의무도 없다.
Qwen3.8-Max License의 조건은 두 갈래다.

| 조건 | 발동 기준 | 요구 사항 |
|------|-----------|-----------|
| 표기 의무 | 상용 제품·서비스의 MAU 1억 초과 또는 월 매출 2천만 달러 초과 | 해당 제품 UI에 모델 이름을 눈에 띄게 표시 |
| 별도 라이선스 | Model as a Service 또는 AI Work Assistant 사업, 연속 12개월 합산 매출 5천만 달러 초과 | 상용 사용 전 Qwen으로부터 별도 라이선스 취득 |

라이선스 원문은 Model as a Service를 제3자가 입력·파라미터·학습데이터를 실질적으로 제어할 수 있는 형태로 추론 또는 파인튜닝 접근을 제공하는 것으로 정의한다.
AI Work Assistant는 AI 코딩 또는 오피스 생산성 전용 독립 제품으로 한정하고, 단일 목적 AI 툴이나 부가 기능형 어시스턴트는 제외한다.
제3자에게 모델·출력·능력을 제공하지 않는 내부 사용은 예외다.

대규모 상업 사용자에 대한 매출 셰어 비율과 정확한 기준선은 아직 확정되지 않았다는 보도가 있다.
Moonshot AI가 Kimi K3에 연 매출 2천만 달러 초과 시 별도 계약과 최대 30% 매출 셰어를 도입한 선례가 있어, 중국 대형 모델 진영이 상업 조건에서 수렴한다는 분석도 함께 제시됐다.

## 아키텍처 내부 구조

### 하이브리드 레이어 구성

두 모델 모두 Gated DeltaNet 기반 linear attention과 Gated Attention을 섞은 하이브리드다.
config.json의 full_attention_interval 값이 4로, 4레이어마다 한 번 full attention이 배치된다.

| 항목 | Qwen3.8-27B | Qwen3.8-2.4T-A95B |
|------|-------------|-------------------|
| Hidden Layout | 16 x (3 x (Gated DeltaNet 후 FFN) 후 1 x (Gated Attention 후 FFN)) | 23 x (3 x (Gated DeltaNet 후 MoE) 후 1 x (Gated Attention 후 MoE)) |
| 전체 레이어 | 64 | 92 |
| full attention 레이어 | 16 | 23 |
| linear attention 레이어 | 48 | 69 |
| Gated DeltaNet 헤드 | V 48, QK 16 | V 128, QK 16 |
| Gated DeltaNet 헤드 차원 | 128 | 128 |
| Gated Attention 헤드 | Q 24, KV 4 | Q 64, KV 4 |

linear attention 레이어는 상수 크기 recurrent state를 유지하므로 컨텍스트가 늘어도 KV 캐시가 선형으로 늘지 않는다.
반대로 full attention 레이어는 그대로 KV를 쌓는다.
이 3대 1 비율은 Kimi K3의 93층 구성과도 같아서, 근본적 분화가 아니라 업계 전반의 효율 최적화 수렴이라는 해석이 나왔다.

### 공통 하이퍼파라미터와 갈라지는 지점

배율이 다를 뿐 두 모델의 하이퍼파라미터 패턴은 동일하다.

| 파라미터 | 값 (두 모델 공통) |
|----------|-------------------|
| head_dim | 256 |
| partial_rotary_factor | 0.25 |
| rope_theta | 10000000 |
| vocab_size | 248,320 |
| max_position_embeddings | 262,144 |
| mtp_num_hidden_layers | 1 |
| mtp_use_dedicated_embeddings | false |
| output_gate_type | swish |
| linear_conv_kernel_dim | 4 |

partial_rotary_factor가 0.25이므로 head_dim 256 중 64차원에만 RoPE가 적용된다.
mtp_num_hidden_layers가 1이라는 것은 MTP draft 헤드가 체크포인트에 1레이어로 내장되어 있다는 뜻이다.
별도 draft 모델 없이 speculative decoding을 쓸 수 있는 근거가 여기 있다.
mtp_use_dedicated_embeddings가 false라 임베딩은 본체와 공유한다.

갈라지는 지점은 FFN과 MoE다.

| 항목 | Qwen3.8-27B | Qwen3.8-2.4T-A95B |
|------|-------------|-------------------|
| FFN intermediate_size | 17,408 | 해당 없음 |
| moe_intermediate_size | 해당 없음 | 2048 |
| shared_expert_intermediate_size | 해당 없음 | 2048 |
| num_experts | 해당 없음 | 512 |
| num_experts_per_tok | 해당 없음 | 10 |
| shared expert | 해당 없음 | 1 |
| router_aux_loss_coef | 해당 없음 | 0.001 |

27B에만 멀티모달 3D RoPE 항목이 있다.
rope_parameters에 mrope_interleaved가 true, mrope_section이 [11, 11, 10]으로 들어 있고, 텍스트 전용인 MoE 쪽에는 이 항목이 아예 없다.
27B의 vision_config는 depth 27, hidden_size 1152, patch_size 16, spatial_merge_size 2, out_hidden_size 5120 구성이다.

두 config의 transformers_version 값이 다르다는 점도 눈에 띈다.
27B는 5.8.0.dev0, MoE는 4.57.3으로 기록되어 있다.

### thinking 제어와 reasoning effort

thinking 관련 스위치는 세 개이고, 모델별로 지원 범위가 다르다.

| 파라미터 | 기본값 | 27B | 2.4T-A95B |
|----------|--------|-----|-----------|
| enable_thinking | True | 켜고 끄기 가능 | 끌 수 없음 |
| preserve_thinking | True | 지원 | 지원 |
| reasoning_effort | xhigh | 지원 | 지원 |

reasoning_effort는 xhigh, medium, low 세 단계이고 두 카드의 정의가 동일하다.
xhigh는 철저한 분석이 필요한 복잡한 작업, medium은 정확도와 속도의 균형, low는 속도와 비용을 위한 효율적 추론이다.

구현은 프롬프트에 삽입되는 시스템 지시문 형태다.
채팅 템플릿을 보면 xhigh와 low에만 별도 지시문이 붙고 medium은 빈 문자열이다.
즉 medium이 "지시문 없음" 상태이고 나머지 두 단계만 프롬프트를 추가한다.

preserve_thinking은 과거 메시지의 thinking 블록 유지 여부를 결정한다.
27B 카드는 기본값 유지가 대화 전체의 추론 흐름을 보존할 뿐 아니라 KV 캐시 활용도를 개선한다고 설명한다.
False로 두면 마지막 사용자 메시지 이후의 thinking만 남는다.

reasoning_effort를 낮추는 것이 항상 이득은 아니라는 경고도 카드에 실려 있다.

> "In multi-turn agentic tasks, lower reasoning effort does not always reduce overall task completion time. Although it may produce faster per-turn responses, it can also lead to insufficient analysis, more failures, and repeated retries, which may increase total latency and token consumption."

## 컨텍스트 길이

### 네이티브 262144와 확장 최대치

두 모델 모두 config.json의 max_position_embeddings가 262,144이고, 카드도 네이티브 컨텍스트를 262,144로 표기한다.
확장 최대치는 소스 간에 값이 갈린다.

| 소스 | 27B | 2.4T-A95B |
|------|-----|-----------|
| 모델 카드 | 1,000,000 | 1,010,000 |
| vLLM 레시피 예시 | 1,010,000 | 1,010,000 |

27B 카드는 "extensible up to 1,000,000 tokens"라고 적고 2.4T 카드는 1,010,000을 적는다.
vLLM 레시피는 두 모델 모두 max-model-len을 1010000으로 예시한다.
어느 한쪽이 오기인지 확인되지 않았으므로 두 값을 함께 두고 읽는 편이 안전하다.

API 전용 Qwen3.8-Max는 기본 1M 컨텍스트를 제공하며 최대 입력 991K, thinking 활성 시 983K다.

### YaRN 설정과 트레이드오프

기본 배포 config의 rope_type은 default이고 YaRN은 사용자가 명시적으로 켜야 한다.
27B 카드가 제시하는 교체 설정은 다음과 같다.

```json
{
    "mrope_interleaved": true,
    "mrope_section": [11, 11, 10],
    "rope_type": "yarn",
    "rope_theta": 10000000,
    "partial_rotary_factor": 0.25,
    "factor": 4.0,
    "original_max_position_embeddings": 262144
}
```

factor 4.0에 262,144를 곱하면 1,048,576이 된다.
서빙 프레임워크에서는 기동 시 오버라이드로 같은 값을 넘긴다.

```shell
VLLM_ALLOW_LONG_MAX_MODEL_LEN=1 vllm serve ... \
  --hf-overrides '{"text_config": {"rope_parameters": {"mrope_interleaved": true, "mrope_section": [11, 11, 10], "rope_type": "yarn", "rope_theta": 10000000, "partial_rotary_factor": 0.25, "factor": 4.0, "original_max_position_embeddings": 262144}}}' \
  --max-model-len 1000000
```

```shell
SGLANG_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN=1 python -m sglang.launch_server ... \
  --json-model-override-args '{"text_config": {"rope_parameters": {"rope_type": "yarn", "factor": 4.0, "original_max_position_embeddings": 262144}}}' \
  --context-length 1000000
```

27B 카드는 세 번째 서빙 경로로 TokenSpeed도 함께 안내한다.
TOKENSPEED_ALLOW_OVERWRITE_LONGER_CONTEXT_LEN 환경변수와 --hf-overrides로 동일한 설정을 전달하는 방식이다.

YaRN 없이 max_position_embeddings만 올리는 대안도 vLLM 레시피에 있다.
27B는 text_config 안에, MoE는 최상위에 값을 넣는다는 점이 다르다.

```bash
# 27B
vllm serve Qwen/Qwen3.8-27B --max-model-len 1010000 \
  --hf-overrides '{"text_config": {"max_position_embeddings": 1010000}}'

# 2.4T-A95B
VLLM_ALLOW_LONG_MAX_MODEL_LEN=1 vllm serve Qwen/Qwen3.8-2.4T-A95B \
  --max-model-len 1010000 --hf-overrides '{"max_position_embeddings": 1010000}'
```

트레이드오프에 대한 경고가 카드에 명시되어 있다.

> "All the notable open-source frameworks implement static YaRN, which means the scaling factor remains constant regardless of input length, potentially impacting performance on shorter texts."

즉 스케일링 계수가 입력 길이와 무관하게 고정되므로 짧은 입력에서 성능이 떨어질 수 있다.
카드는 장문 처리가 필요할 때만 rope_parameters를 수정하고, 실제 사용하는 컨텍스트 길이에 맞춰 factor를 조정하라고 권한다.
예로 일상적인 컨텍스트가 524,288 토큰이면 factor를 2.0으로 두라고 적는다.

실제 확장에서 문제가 보고되기도 했다.
RTX PRO 6000 Blackwell 96GB에서 Qwen3.8-27B-UD-Q8_K_XL을 YaRN 4배로 1M까지 늘린 뒤 약 925K 프롬프트를 넣으면 prefill 520,192 토큰 지점에서 llama-server가 assert도 에러도 없이 종료된다.
같은 환경에서 네이티브 윈도우 224K 프롬프트와 YaRN 2배 444K 프롬프트는 정상 완료되고 needle 검색도 통과했다.

컨텍스트 길이는 KV 예약량에도 직결된다.
vLLM 레시피는 262,144로 잡으면 엔진이 동시 요청 25개 분량의 KV만 확보하는 반면, 9,240으로 낮추면 같은 70 GiB로 506개를 처리한다고 기록한다.

27B에서 장시간 비디오를 다룰 때는 컨텍스트와 별개로 비디오 토큰 상한을 따로 올려야 한다.
video_preprocessor_config.json의 longest_edge를 조정해 상한을 224K 토큰까지 확장한다.

```json
{"longest_edge": 469762048, "shortest_edge": 4096}
```

## 벤치마크

### 2.4T-A95B 공식 수치

모델 카드는 오픈웨이트 2.4T-A95B의 벤치마크 열을 Qwen3.8-Max로 표기한다.
비교군은 Opus 4.8, Fable 5, GPT 5.6 Sol (max), Qwen3.7-Max다.

코딩 에이전트 계열이다.

| 벤치마크 | Opus 4.8 | Fable 5 | GPT 5.6 Sol (max) | Qwen3.7-Max | Qwen3.8-Max |
|----------|----------|---------|-------------------|-------------|-------------|
| Terminal Bench 2.1 | 84.6 | 84.6 | 88.8 | 74.5 | 86.6 |
| SWE-bench Pro | 69.2 | 80.0 | 64.6 | 60.6 | 67.7 |
| DeepSWE 1.1 | 59.0 | 70.0 | 73.0 | 21.6 | 56.6 |
| NL2Repo-Bench | 69.4 | -- | -- | 47.2 | 55.9 |
| FrontierSWE | 70.0 | 88.8 | -- | 40.7 | 73.5 |
| MLS-Bench-Lite | 42.8 | 49.9 | 46.2 | 31.7 | 41.0 |
| PaperBench | 80.3 | 88.8 | 90.5 | 64.8 | 93.0 |
| AndroidBench | 69.8 | 84.5 | 74.0 | 56.5 | 75.1 |
| QwenSWEBench | 84.0 | 86.3 | 73.5 | 63.4 | 80.7 |
| QwenQoderBench | 62.7 | 63.1 | 53.8 | 36.8 | 58.4 |
| QwenReactBench (Elo) | 1694 | 1770 | 1564 | 1538 | 1724 |
| QwenSVGBench (Elo) | 1648 | 1690 | 1758 | 1499 | 1713 |

일반 에이전트 계열이다.

| 벤치마크 | Opus 4.8 | Fable 5 | GPT 5.6 Sol (max) | Qwen3.7-Max | Qwen3.8-Max |
|----------|----------|---------|-------------------|-------------|-------------|
| CoWorkBench | 72.3 | 75.9 | 71.5 | 64.6 | 74.8 |
| WorkSpaceBench | 66.8 | 68.7 | 65.6 | 61.4 | 67.7 |
| JobBench | 48.4 | 57.4 | 45.4 | 31.3 | 53.4 |
| SkillsBench | 65.1 | 70.9 | 73.5 | 61.2 | 70.2 |
| Agents' Last Exam (Pass / Score) | 27.0 / 45.1 | -- / -- | 30.6 / 53.6 | 11.8 / 31.1 | 27.0 / 52.4 |
| Automation-Bench (Pass@1) | 27.2 | 29.1 | 29.7 | 14.2 | 27.3 |
| Toolathlon Verified (Pass@1) | 76.2 | 77.9 | 74.9 | 49.7 | 72.5 |
| WideSearch | 72.9 | 81.2 | -- | 75.2 | 81.9 |
| HLE w/ tools | 57.9 | 64.5 | 58.0 | 53.5 | 56.2 |

일반 능력 계열이다.

| 벤치마크 | Opus 4.8 | Fable 5 | GPT 5.6 Sol (max) | Qwen3.7-Max | Qwen3.8-Max |
|----------|----------|---------|-------------------|-------------|-------------|
| GPQA Diamond | 92.0 | 92.6 | 94.1 | 92.4 | 92.6 |
| HLE | 45.7 | 53.3 | 47.2 | 41.4 | 43.6 |
| IFBench | 62.2 | 63.5 | 72.7 | 79.1 | 82.8 |
| $OneMillion-Bench (expert score) | 41.8 | 55.9 | 53.8 | 44.4 | 52.5 |
| HealthBench | 52.4 | -- | 55.3 | 54.5 | 60.2 |
| PLawBench | 69.6 | 70.2 | 72.3 | 58.9 | 73.2 |
| PRBench-Legal | 52.7 | 57.6 | 57.6 | 48.5 | 57.6 |
| PRBench-Finance | 51.9 | 55.8 | 55.5 | 46.8 | 58.3 |
| MRCR v2 256K (8-needle) | 83.2 | -- | 93.8 | 86.7 | 92.9 |
| LongBench v2 | 69.1 | -- | 67.1 | 65.3 | 66.3 |

PaperBench 93.0, IFBench 82.8, WideSearch 81.9는 비교표 1위다.
반면 DeepSWE 1.1 56.6과 Terminal Bench 2.1 86.6에서는 GPT 5.6 Sol과 Fable 5에 밀린다.
세대 간 상승폭은 에이전트 계열에 몰려 있다.
Terminal Bench 2.1은 74.5에서 86.6으로, PaperBench는 64.8에서 93.0으로 올랐지만 GPQA Diamond는 92.4에서 92.6으로 사실상 정체다.

### 27B 공식 수치

27B의 비교군은 Qwen3.6-27B, Qwen3.7-Plus, Muse Glimmer-30B, Opus4.6 Max로 다르게 잡혀 있다.
비교 대상이 Opus 4.6 Max로 한정된 점은 커뮤니티에서 논쟁이 됐다.

텍스트 성능이다.

| 벤치마크 | Qwen3.8-27B | Qwen3.6-27B | Qwen3.7-Plus | Muse Glimmer-30B | Opus4.6 Max |
|----------|-------------|-------------|--------------|------------------|-------------|
| Terminal Bench 2.1 (Terminus) | 73.0 | 63.4 | 64.0 | 51.7 | 78.2 |
| SWE-bench Pro | 61.7 | 53.5 | 57.6 | 51.2 | 53.4 |
| NL2Repo-Bench | 42.3 | 36.2 | 41.1 | -- | 47.6 |
| DeepSWE 1.1 | 42.2 | 13.3 | 14.2 | -- | -- |
| QwenSWEBench | 79.0 | 49.3 | 59.2 | -- | 63.8 |
| CoWorkBench | 70.7 | 61.0 | 65.1 | -- | 68.2 |
| JobBench | 33.4 | 21.8 | 27.6 | -- | -- |
| Agents' Last Exam (Pass@1) | 20.4 | 10.6 | 13.2 | -- | -- |
| Agents' Last Exam (Score) | 42.9 | 27.3 | 33.6 | -- | -- |
| IFBench | 79.5 | 69.1 | 79.1 | 77.0 | 62.5 |
| GPQA Diamond | 89.2 | 87.8 | 90.3 | 83.5 | 91.3 |
| HLE | 30.8 | 24.0 | 34.7 | 22.0 | 40.0 |
| LiveCodeBench v6 | 90.3 | 83.9 | 89.6 | -- | 88.8 |

에이전틱 멀티모달 성능이다.

| 벤치마크 | Qwen3.8-27B | Qwen3.6-27B | Qwen3.7-Plus | Muse Glimmer-30B | Opus4.6 Max |
|----------|-------------|-------------|--------------|------------------|-------------|
| OSWorld-Verified | 84.3 | 63.9 | 73.3 | 65.9 | 72.7 |
| WebArena-Verified | 64.8 | 48.8 | 55.3 | -- | -- |
| AndroidWorld | 81.9 | 70.3 | 81.0 | -- | 62.0 |
| RecreationBench | 47.1 | 29.8 | 30.2 | -- | -- |
| ClawEval-MM (Pass@3 / Average) | 57.4 / 56.9 | 42.6 / 50.4 | 57.4 / 60.1 | -- | 52.5 / 54.7 |
| SWE-MM | 38.6 | 25.7 | 30.0 | -- | 27.1 |
| Vision2Web | 62.9 | 45.0 | 42.1 | -- | -- |

일반 멀티모달 성능이다.
CI는 Code Interpreter를 뜻한다.

| 벤치마크 | Qwen3.8-27B | Qwen3.6-27B | Qwen3.7-Plus | Muse Glimmer-30B | Opus4.6 Max |
|----------|-------------|-------------|--------------|------------------|-------------|
| MathVision (Without CI) | 90.0 | 85.1 | 90.3 | -- | 65.5 |
| MathVision (With CI) | 94.6 | -- | -- | -- | -- |
| BabyVision (Without CI) | 65.7 | 28.9 | 64.7 | -- | 12.6 |
| BabyVision (With CI) | 85.6 | -- | 70.4 | -- | -- |
| CharXiv RQ (Without CI) | 83.7 | 78.4 | 85.8 | 78.8 | 66.0 |
| CharXiv RQ (With CI) | 90.2 | -- | 85.9 | -- | -- |
| OmniDocBench 1.5 | 91.1 | 89.4 | 91.4 | 75.8 | 86.6 |
| RealWorldQA | 85.9 | 84.1 | 86.9 | -- | 73.9 |
| ERQA | 65.5 | 62.5 | 69.8 | -- | 40.8 |

27B가 상위 라인업인 Qwen3.7-Plus를 다수 항목에서 앞선다는 점이 눈에 띈다.
SWE-bench Pro 61.7 대 57.6, CoWorkBench 70.7 대 65.1, OSWorld-Verified 84.3 대 73.3이다.
반대로 GPQA Diamond 89.2 대 90.3, HLE 30.8 대 34.7, ERQA 65.5 대 69.8에서는 뒤진다.

### 평가 조건과 harness 비대칭

표 아래 각주를 읽지 않으면 수치를 잘못 해석하기 쉽다.

먼저 인하우스 벤치마크가 여럿 섞여 있다.
QwenSWEBench, QwenQoderBench, QwenReactBench, QwenSVGBench, CoWorkBench, RecreationBench는 모두 Qwen이 설계한 벤치마크다.
과제와 채점기가 성숙한 공개 스위트만큼의 외부 감사 가능성을 아직 제공하지 않는다는 지적이 있다.

harness가 모델별로 다르게 적용된 항목도 있다.
SkillsBench는 Opus 4.8과 Fable 5를 Claude Code로, GPT-5.6 Sol을 Codex로, Qwen 계열을 OpenCode로 평가했다.
WideSearch는 외부 모델에 Claude Code harness를, 자사 모델에 Qwen-Agent harness를 썼다.
Terminal Bench 2.1은 Qwen 측이 Claude Code로 avg@10을 냈고 타 모델은 harness별 공개 최고 점수를 인용했다.

심사 모델도 항목마다 다르다.
PaperBench는 Claude Opus 4.6이, $OneMillion-Bench와 PLawBench는 gemini-3.1-pro-preview가, 27B의 HLE는 GPT-4o가, Vision2Web은 gpt-5.4-2026-03-05가 심사했다.
MathVision은 Qwen3.8-27B만 고정 프롬프트로 평가하고 타 모델은 두 변형 중 높은 점수를 채택했다.
MathVision과 CharXiv RQ는 잘못된 정답 라벨을 수동 수정한 뒤 전 점수를 재계산했다고 밝힌다.
SWE-bench Pro에서는 Opus4.6 Max만 공식 발표 점수를 그대로 가져오고 나머지는 자체 평가했다.

토큰 예산도 항목마다 크다.
Terminal Bench 2.1과 MLS-Bench-Lite는 5시간 타임아웃에 max_tokens 131,072, QwenSWEBench는 8시간 타임아웃, QwenQoderBench는 6시간 타임아웃, PaperBench는 회당 최대 12시간 3회 평균이다.

### 독립 평가 결과

Artificial Analysis Intelligence Index에서 Qwen3.8-Max는 56점으로 Claude Opus 4.8 max와 동률을 기록했다.
Kimi K3가 57, Meta Muse Spark 1.2와 Grok 4.5가 54, GLM-5.2 max가 51이다.
GDPval-AA Elo에서는 Claude Opus 5 max 1852, Claude Fable 5 1743, Qwen3.8-Max 1739, GPT-5.6 Sol max 1730, Kimi K3 1685 순이다.

Arena 계열은 보드마다 순위가 다르다.
Frontend Code Arena에서 4위 1,668 Elo로 오픈웨이트 중 2위이며, Claude Opus 5 (Max) 1,705와 Kimi K3 (Max) 1,676 아래, Claude Opus 5 (High) 1,669와 동급이다.
Arena.AI 텍스트 부문은 5위 1496, 비전 부문은 2위 1305다.
Vals Index에서는 오픈웨이트 중 2위 66.1이다.

태스크당 실측 비용은 Artificial Analysis 기준 GLM-5.2 max $0.57, Kimi K3 max $0.86, Qwen3.8-Max $1.14, GPT-5.6 Sol max $1.23, Claude Opus 5 max $2.03, Claude Fable 5 $3.15 순이다.
토큰 단가는 낮지만 태스크당 토큰을 많이 쓰기 때문에 실제 비용 순위가 단가 순위와 다르다.

균형을 위해 함께 봐야 할 수치가 공식 표 안에도 있다.
HLE 43.6은 Opus 4.8 45.7, GPT 5.6 Sol 47.2, Fable 5 53.3보다 낮아 비교군 중 최하위다.
에이전트 계열의 상승과 달리 순수 지식·추론 항목에서는 격차가 남아 있다는 뜻이다.

Artificial Analysis 순위 자체가 짧은 기간에 바뀌기도 했다.
Artificial Analysis 팀은 채점 모델 업그레이드를 반영한 업데이트로 Qwen3.8 Max가 1위에서 2위로 내려갔다고 직접 밝혔고, 동시에 에이전트 능력에서는 큰 도약이라고 덧붙였다.

## 실행과 배포

### 하드웨어 요구량

2.4T-A95B는 정밀도에 따라 요구 GPU 수가 크게 달라진다.

| Variant | Weights | Sized | B300 | MI355X | H200 | GB300 tray |
|---------|---------|-------|------|--------|------|------------|
| BF16 | 4.45 TiB | 5871 GB | 24 GPUs | 24 GPUs | 48 GPUs | 6 trays |
| FP8 | 2.27 TiB | 2996 GB | 16 GPUs | 16 GPUs | 32 GPUs | 4 trays (TP16) |
| MXFP4 (AMD) | 1.45 TiB | 1917 GB | -- | 8 GPUs | 16 GPUs | -- |
| NVFP4 W4A4 (NVIDIA) | 1.32 TiB | 1737 GB | 8 GPUs | -- | 16 GPUs | 2 trays (TP8) |

주의할 제약이 하나 있다.
어텐션 헤드가 64개라 TP 값이 64를 나눠야 하므로 1, 2, 4, 8, 16, 32만 유효하다.
용량 계산상 12 GPU면 될 것 같아도 실제로는 TP16이 필요하다는 뜻이다.

27B는 상황이 훨씬 단순하다.
vLLM 레시피는 모든 정밀도에서 단일 Blackwell GPU에 올라간다고 적고, NVFP4가 24.6 GiB이며 1M 컨텍스트에서 6.6M KV 토큰을 수용한다고 기록한다.
SGLang 쿡북은 H200, RTX PRO 6000, RTX 5090, DGX Spark를 단일 GPU 타깃으로 명시한다.

파이프라인 병렬은 GB300에서 TP4 곱하기 PP3, 3 tray FP8 구성으로 검증됐다.
하이브리드 linear attention 모델이 PP 지원에서 뒤처진다는 이전 경고는 해소됐다고 레시피가 밝힌다.

### vLLM 실행 명령

프레임워크 요구 버전부터 다르다.

| 항목 | 값 |
|------|-----|
| Transformers (27B) | 5.8.0 이상 |
| Transformers (2.4T) | 5.4.0 이상 |
| vLLM (27B) | 0.17.0 이상 |
| vLLM (2.4T) | nightly 빌드 |

2.4T 설치는 nightly 인덱스를 쓴다.

```bash
uv venv && source .venv/bin/activate
uv pip install -U vllm --extra-index-url https://wheels.vllm.ai/nightly
uv pip install -U "transformers>=5.4.0"
```

27B는 단일 GPU 저지연 구성과 FP8 TP4 구성이 제시된다.

```bash
# 저지연 NVFP4, TP1
vllm serve Inferact/Qwen3.8-27B-NVFP4 \
  --tensor-parallel-size 1 \
  --max-model-len 262144 \
  --kv-cache-dtype fp8 \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice --tool-call-parser qwen3_coder

# FP8, TP4 (GB300 1 tray)
vllm serve Qwen/Qwen3.8-27B-FP8 \
  --tensor-parallel-size 4 \
  --max-model-len 262144 \
  --kv-cache-dtype fp8 \
  --reasoning-parser qwen3
```

2.4T는 TP8 NVFP4 또는 TP16 2노드 FP8이 기준 구성이다.

```bash
# 저지연 NVFP4 W4A4, TP8
vllm serve Inferact/Qwen3.8-2.4T-A95B-NVFP4 \
  --tensor-parallel-size 8 \
  --max-model-len 262144 \
  --kv-cache-dtype fp8 \
  --reasoning-parser qwen3 \
  --enable-auto-tool-choice --tool-call-parser qwen3_coder

# FP8, TP16 2노드
vllm serve Qwen/Qwen3.8-2.4T-A95B-FP8 \
  --tensor-parallel-size 16 \
  --nnodes 2 --node-rank 0 --master-addr $HEAD_ADDR \
  --max-model-len 262144 \
  --kv-cache-dtype fp8 \
  --reasoning-parser qwen3
```

고처리량 구성은 MoE 백엔드를 바꾼다.
FP8은 TP4 곱하기 DP4에 EP16을 얹고 flashinfer_trtllm을, NVFP4는 DEP16에 flashinfer_cutedsl을 쓴다.

툴 콜 포맷이 JSON이 아니라 XML 스타일 중첩 구조라는 점도 실무에 영향을 준다.
채팅 템플릿이 tools를 받으면 tool_call 태그 안에 function 태그를 중첩하고 그 안에 parameter 태그를 넣는 형식을 시스템 메시지로 주입한다.
vLLM에서 대응 파서는 qwen3_coder다.

가중치 로딩 시간도 튜닝 대상이다.
1.32 TiB 체크포인트에서 fastsafetensors와 lazy 로드 전략을 쓰면 545초에서 306초로 44% 줄었다고 레시피가 기록한다.

### Transformers와 OpenAI SDK 호출

27B는 비전 인코더를 포함하므로 Transformers에서 image-text-to-text pipeline으로 바로 쓸 수 있다.

```python
from transformers import pipeline

pipe = pipeline("image-text-to-text", model="Qwen/Qwen3.8-27B")
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "image",
                "url": "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/p-blog/candy.JPG"
            },
            {"type": "text", "text": "What animal is on the candy?"}
        ]
    },
]
pipe(text=messages)
```

processor와 model을 직접 로드하는 경로도 지원한다.

```python
from transformers import AutoProcessor, AutoModelForMultimodalLM

processor = AutoProcessor.from_pretrained("Qwen/Qwen3.8-27B")
model = AutoModelForMultimodalLM.from_pretrained(
    "Qwen/Qwen3.8-27B",
    device_map="auto"
)

inputs = processor.apply_chat_template(
    messages,
    add_generation_prompt=True,
    tokenize=True,
    return_dict=True,
    return_tensors="pt",
).to(model.device)

outputs = model.generate(**inputs, max_new_tokens=40)
print(processor.decode(outputs[0][inputs["input_ids"].shape[-1]:]))
```

Docker Model Runner로 한 줄 실행도 가능하다.

```bash
docker model run hf.co/Qwen/Qwen3.8-27B
```

서빙 프레임워크는 모두 OpenAI 호환 Chat Completions API를 제공하므로 클라이언트 코드는 공통이다.
thinking 관련 옵션은 extra_body의 chat_template_kwargs로, reasoning_effort는 최상위 인자로 전달한다.

```python
from openai import OpenAI

client = OpenAI()

completion = client.chat.completions.create(
    model="Qwen/Qwen3.8-27B",
    messages=messages,
    extra_body={
        "chat_template_kwargs": {
            "enable_thinking": True,
            "preserve_thinking": True,
        },
    },
    reasoning_effort="xhigh",
    stream=True,
    stream_options={"include_usage": True},
)
```

스트리밍 응답에서는 thinking 내용이 delta.reasoning_content 또는 delta.reasoning 필드로, 최종 답변이 delta.content로 분리되어 나온다.
비디오 입력은 vLLM에 한해 mm_processor_kwargs로 fps와 do_sample_frames를 지정해 프레임 샘플링을 제어할 수 있다.

알려진 이슈도 정리되어 있다.
NVIDIA 장비에서 MXFP4는 linear method 지원이 빠져 있어 의도대로 동작하지 않으므로 NVFP4를 쓰라고 명시한다.
CUDA Graph 캡처 크기가 recurrent state 캐시를 넘으면 assert가 발생하므로 max-cudagraph-capture-size를 줄여야 한다.
엔진 기동 타임아웃은 VLLM_ENGINE_READY_TIMEOUT_S를 3600으로 올리고, readiness 체크는 health 대신 실제 chat completions 프로브를 권한다.
SGLang 쪽은 speculative decoding 활성화 시 max-running-requests를 지정하지 않으면 자동으로 48로 재설정되고, CP와 DP-Attention 조합은 dp_size 1을 강제해 지원되지 않는다.

### MTP speculative decoding

MTP 헤드가 체크포인트에 내장되어 있으므로 플래그 하나로 켤 수 있다.

```
--speculative-config '{"method":"mtp","num_speculative_tokens":3}'
```

2.4T 기준 저지연 성능은 MTP 유무에 따라 두 배 이상 벌어진다.

| Variant | Layout | MTP 없음 | MTP-3 | 커널 최적화 적용 |
|---------|--------|----------|-------|------------------|
| FP8 | TP16 | 130 | 307 | 150 / 360 (target) |
| NVFP4 | TP8 | 133 | 304 | -- |

depth 1은 권장되지 않는다.
레시피는 MTP-1의 draft acceptance rate가 64.8%로 측정됐고, 동시성 1에서 3.4% 개선에 그치는 반면 그 위에서는 손해라고 적는다.
동시성 128에서 9% 손실, 256에서 23% 손실이다.
depth 3이 사용자당 약 2.3배 개선을 준다.

고처리량 구성의 총 처리량은 다음과 같다.

| Variant | Layout | 백엔드 | GPU당 총 TPS |
|---------|--------|--------|--------------|
| FP8 | TP4 곱하기 DP4 + EP16 | one-sided all2all, trtllm-gen MoE, activation opt | 최대 3200 |
| NVFP4 | DEP16 | one-sided all2all, cute-dsl MoE | 최대 4300 |

NVFP4 W4A4 TP8에서 8k 입력 1k 출력으로 검증한 동시성별 수치다.

| 동시성 | 사용자당 TPS | GPU당 총 TPS | TPOT |
|--------|--------------|--------------|------|
| 1 | 101 | 108 | 9.9 ms |
| 32 | 41.9 | 1338 | 23.9 ms |
| 64 | 26.9 | 1715 | 37.2 ms |
| 128 | 18.3 | 2170 | 54.7 ms |

문제는 MTP의 효과가 스택에 따라 부호까지 뒤집힌다는 점이다.
Apple Silicon M4 Pro에서 예측 가능·불가능 프롬프트를 나눠 통제 실험한 결과가 이를 잘 보여준다.

| 엔진 | MTP 헤드 | 예측 가능 | 예측 불가 | 비율 |
|------|----------|-----------|-----------|------|
| GGUF q4_K_M | 없음 | 11.91 tok/s | 11.78 tok/s | 1.01배 |
| GGUF mtp-q4_K_M | 있음 | 11.40 tok/s | 5.14 tok/s | 2.22배 |
| MLX nvfp4 | 있음 | 38.36 tok/s | 20.26 tok/s | 1.89배 |

draft가 거부되면 GGUF MTP 경로는 5.14 tok/s로 떨어져 speculation을 아예 쓰지 않는 것보다 2.3배 느려진다.
같은 하드웨어에서 MLX 경로는 반대로 큰 이득을 낸다.
같은 저자는 nvfp4 가중치 약 13.9GB와 대역폭 270GB/s로 계산한 이론 상한 19.4 tok/s에 대해 MLX 예측 가능 경로가 38.36 tok/s, 즉 상한의 198%를 냈다는 점을 근거로 다중 토큰 방출을 확인했다.

전용 GPU에서는 이득이 확인된다.
RTX 4080 SUPER 16GB에서 speculation 없음 30.73 tok/s, ngram 30.86 tok/s, draft-mtp가 산문 41.42 tok/s와 코드 재작성 46.87 tok/s로 35~52% 개선됐다.
다만 브라우저가 VRAM을 약 2GB 점유한 데스크톱에서는 같은 설정이 40% 손실로 뒤집혔다.

최적 depth에 대한 합의도 없다.
RTX Pro 6000 Blackwell Max-Q 300W에서 vLLM 0.27.1 FP8 기준 측정치는 없음 46.8, 1이 56.3, 2가 62.2, 3이 59.6 tok/s로 2가 최적이었다.
같은 카드 600W 버전에서는 3이 최적이었고, 측정자는 최적 num_speculative_tokens가 전력 예산에 의존하는 것으로 보인다고 정리했다.
같은 카드에서 튜닝 없이 num_speculative_tokens를 5로 두면 acceptance rate 26.4%에 23 tok/s로 오히려 손해가 났다.

### 양자화 선택지

공식 양자화는 FP8 두 개뿐이다.
방식은 블록 크기 128의 fine-grained fp8 양자화이며, 카드는 성능이 원본과 거의 동일하다고 적는다.
27B-FP8은 modules_to_not_convert에 비전 타워 블록 전체를 포함해 비전 인코더를 FP8로 변환하지 않는다.

GGUF, NVFP4, MXFP4, AWQ-GPTQ, MLX는 전부 서드파티다.
공식 GGUF와 NVFP4, AWQ는 존재하지 않는다.

| 리포 | 종류 | 다운로드 |
|------|------|----------|
| unsloth/Qwen3.8-27B-GGUF | GGUF | 867,963 |
| lmstudio-community/Qwen3.8-27B-GGUF | GGUF | 171,518 |
| unsloth/Qwen3.8-27B-NVFP4 | NVFP4 | 90,924 |
| ggml-org/Qwen3.8-27B-GGUF | GGUF | 34,520 |
| RadixArk/Qwen3.8-27B-NVFP4 | NVFP4 | 30,122 |
| Inferact/Qwen3.8-27B-NVFP4 | NVFP4 | 21,527 |
| unsloth/Qwen3.8-2.4T-A95B-GGUF | GGUF | 12,028 |
| amd/Qwen3.8-2.4T-A95B-Quark-MXFP4 | MXFP4 | 3,222 |

여기서 짚어야 할 점은, 공식 양자화가 FP8뿐인데 정작 vLLM과 SGLang 공식 레시피가 저지연 경로로 지목하는 체크포인트는 서드파티 NVFP4라는 것이다.
27B 저지연 예시는 Inferact/Qwen3.8-27B-NVFP4, 2.4T SGLang 쿡북은 RadixArk 계열을 가리킨다.
공식 문서가 비공식 가중치를 기본 경로로 안내하는 구조이므로 운영 시 검증 책임이 사용자에게 넘어온다.

서드파티 NVFP4에서 실제로 버그가 보고되기도 했다.
SGLang에서 unsloth NVFP4를 서빙하면 첫 토큰부터 무한 반복이 발생하는데, CompressedTensorsConfig가 ParallelLMHead를 디스패치하지 않아 lm_head.weight_scale이 유실되고 FP8 원시 가중치를 BF16으로 소비하는 것이 원인으로 규명됐다.
동일 체크포인트가 vLLM에서는 정상 동작한다.
vLLM 쪽에서는 KV 캐시 dtype을 turboquant 4bit 또는 3bit로 두면 8K 이상 입력에서 반복 붕괴가 발생하고 fp8이면 정상이라는 12회 재현 리포트가 올라와 있다.

양자화 품질에 대한 정량 자료도 있다.
wikitext-2 100청크 기준 KL 다이버전스 측정에서 unsloth UD-IQ2_XXS가 8.39 GiB에 top-1 일치 82.98%, bartowski IQ2_XXS가 76.53%, stock IQ1_M이 65.51%, stock IQ1_S가 60.49%였다.
측정자의 결론은 이 모델에서 2비트 미만은 값어치가 없다는 것이다.
Gated DeltaNet 경로를 보호하는 수동 레시피는 오히려 결과가 나빴는데, FFN이 가중치의 64%를 차지해 어텐션 보호분을 FFN에서 빼오는 것이 손해이기 때문이다.

### 권장 샘플링 파라미터

thinking 가능 여부가 갈리므로 27B는 두 세트, 2.4T는 한 세트다.

| 파라미터 | 27B Thinking | 27B non-thinking | 2.4T-A95B |
|----------|--------------|------------------|-----------|
| temperature | 1.0 | 0.7 | 1.0 |
| top_p | 0.95 | 0.80 | 0.95 |
| top_k | 20 | 20 | 20 |
| min_p | 0.0 | 0.0 | 0.0 |
| presence_penalty | 0.0 | 1.5 | 0.0 |
| repetition_penalty | 1.0 | 1.0 | 1.0 |

두 카드 공통으로 무한 반복을 줄이려면 presence_penalty를 0에서 2 사이로 조정할 수 있다고 안내하되, 값이 높으면 언어 혼합과 약간의 성능 저하가 나타날 수 있다고 덧붙인다.
generation_config.json 실측값은 두 모델 동일하게 do_sample true, temperature 1.0, top_k 20, top_p 0.95다.

출력 길이 권장치는 1M 컨텍스트 기준으로 reasoning content 최대 262,144 토큰, 최종 응답 최대 131,072 토큰이다.

### 커뮤니티 실측 처리량

27B의 하드웨어별 실측치는 편차가 크다.
같은 카드라도 양자화, 스택, MTP 설정에 따라 몇 배씩 갈린다.

| 하드웨어 | 양자화 / 스택 | 실측 |
|----------|---------------|------|
| RTX Pro 6000 96GB | NVFP4+Q5_K, 자체 엔진 | 140 tok/s (MTP 없이 69), TTFT 0.156s |
| RTX Pro 6000 Max-Q 300W | FP8, vLLM 0.27.1, MTP-2 | 62.2 tok/s |
| RTX Pro 6000 600W | FP8, vLLM 커스텀 튜닝 | 80~100 tok/s |
| RTX 5090 | UD-Q4 + MTP, llama.cpp | 101 tok/s 생성, 2,650 tok/s prefill |
| RTX 5090 + RTX 3060 | 동일 구성 | 53 tok/s 생성, 1,700 tok/s prefill |
| RTX 4090 | q4km, llama.cpp | 약 48 tok/s |
| RTX 4090 | Q5_K_S, llama.cpp | 약 33 tok/s |
| RTX 5070 Ti 16GB | UD-Q2_K_XL, 128K ctx | 50~60 tok/s |
| RTX 3090 단일 | UD-Q4 | 약 60 tok/s |
| 2x RTX 3080 20G | UD-Q6_K_XL, MTP, 212992 ctx | 평균 40~50 tok/s |
| 2x RTX A6000 | UD-Q8_K_XL, 배칭 없음 | 약 60 tok/s |
| 3x AMD Instinct MI50 16GB | Q8_0, ROCm 6.2.4, 262K ctx | 디코드 29~31 tok/s |
| MBP M4 64GB | MLX + MTP | 약 45 tok/s |
| Mac mini M4 Pro 64GB | LM Studio | 12.75 tok/s |
| MacBook M5 Max 48GB | GGUF, OpenCode | 15 tok/s 절전 / 30 tok/s 성능 |
| Ryzen AI Max+ 395 128GB | UD-Q5_K_XL, MTP-4, 64K ctx | 16.68 tok/s 생성, 33.87 tok/s prefill |
| AMD 7840U CPU 전용 | llama.cpp Vulkan | 약 4 tok/s |

Ryzen AI Max+ 395 리포트는 런타임 선택의 영향도 함께 보여준다.
llama.cpp Vulkan이 16.68 tok/s인 반면 LM Studio ROCm 런타임은 12.04 tok/s로, Vulkan이 37% 빨랐다.
공식 llama.cpp ROCm 바이너리는 gfx1151을 인식하지 못했다.
같은 장비의 OpenCode 실사용에서는 툴 스키마 41개와 프롬프트 13,264 토큰의 콜드 요청에서 프롬프트 평가만 154.5초가 걸렸고 전체 상호작용은 189초였다.
KV 재사용 시에는 200 tok/s를 넘겼다.

같은 RTX 4090이라도 보고 편차가 큰데, llama.cpp에서 70~80 tok/s가 나온다는 보고도 함께 올라와 있다.
이 보고에는 KV 캐시를 양자화하면 성능 저하가 관찰된다는 단서가 붙어 있다.
하이브리드 구조로 KV 캐시 자체가 이미 줄어든 만큼 추가 양자화의 이득이 크지 않을 수 있다는 맥락이다.

3.6 대비 속도 회귀 보고가 여럿 있었지만 상당수는 원인이 따로 밝혀졌다.
mmproj 파일을 함께 내려받은 경우 CPU 오프로딩이 발생했고, 파일을 지우자 오프로딩이 사라졌다는 확인이 있다.
인용 시 이 반박까지 함께 봐야 정확하다.

### 채팅 템플릿 예외로 인한 연동 실패

Qwen3.8-27B 공식 Jinja 템플릿에는 강제 assertion이 다섯 개 들어 있다.
그중 하나가 시스템 메시지가 맨 앞에 오지 않으면 예외를 던지는 조건이다.

```jinja
{% raw %}{{- raise_exception('System message must be at the beginning.') }}{% endraw %}
```

이 때문에 Claude Code, Codex, SillyTavern 연동이 기본 상태에서 깨진다.
최신 Claude Code가 대화 중간에 두 번째 system 메시지를 보내는데 qwen3.8 렌더러가 position 0만 허용하기 때문이다.
동일 요청을 qwen3.6:27b나 gpt-oss:20b로 재생하면 정상 동작하고 qwen3.8:27b만 거부한다는 최소 재현이 보고됐다.

3.6과 3.8의 템플릿 동작 차이는 다음과 같다.

| 동작 | 3.8 | 3.6 |
|------|-----|-----|
| user 뒤에 오는 system 메시지 | 예외 발생 | 조용히 무시 |
| assistant 뒤에 오는 developer 메시지 | 예외 발생 | 조용히 무시 |
| 500 에러 유발 | 발생 | 미발생 |
| reasoning_effort 지원 | 지원 | 미지원 |

수정 현황은 스택마다 다르다.
Ollama는 PR로 대응해 0.32.14에서 해결됐고, 검증 코멘트는 0.32.13에서 500이던 요청이 rc0에서 200으로 바뀌었다고 기록한다.
추가로 밝혀진 함정은 배열 내 위치가 아니라 최상위 system 필드가 원인이라는 점이다.
최상위 필드가 메시지 0으로 prepend되면서 배열 안의 system이 index 1 이상으로 밀린다.

llama.cpp 쪽은 아직 열려 있다.
llama-server의 오토파서가 합성 메시지로 템플릿을 프로빙하는데 일부 프로브가 system을 앞에 두지 않아 토큰 생성 전에 400이 발생한다.
Qwen3.5 계열 템플릿의 알려진 미해결 상류 이슈로 정리되어 있다.
Codex도 동일하며, 이전 Qwen3.6 채팅 템플릿으로 교체하면 해결된다는 보고가 있다.
커뮤니티 수정 템플릿 저장소도 여럿 공개되어 raise_exception을 정상 렌더링으로 교체하는 방식으로 우회한다.

에이전트 harness별 연동 성적을 정리하면 다음과 같다.

| harness | 결과 |
|--------|------|
| Claude Code | 기본 상태 실패. Ollama 0.32.14 또는 패치 템플릿 필요 |
| Codex | 3.6 템플릿으로 교체해야 동작 |
| OpenCode | 혼재. 툴콜 정상 보고와 잦은 에러·타임아웃 보고가 공존 |
| Qwen Code | 동작. 스킬 로딩이 약해 front matter 보강 필요 |
| Hermes agent | 동작. 다중 GPU 구성 공개됨 |

이 밖에 서빙 단계의 초기 버그도 여럿 보고됐다.
sm120 장비에서 FlashInfer 샘플러 JIT가 부팅을 크래시시켜 VLLM_USE_FLASHINFER_SAMPLER를 0으로 두는 우회가 필요하다.
llama.cpp에서 split-mode tensor와 draft-mtp를 함께 쓰면 CUDA 락업이 발생하고, split-mode layer로 바꾸면 나타나지 않는다.
Strix Halo ROCm 환경에서는 동시 요청 하에 생성 상태가 점진적으로 오염되어 MCQ 정확도가 예상 60%대에서 17%로 떨어졌다가 언로드 후 복구되는 현상이 보고됐다.
DGX Spark FP8 검증 리포트는 쿡북이 생성하는 mem-fraction-static 0.95 대신 실측 안정값이 0.70이었다고 정정한다.

## 커뮤니티 평가

### 과도한 thinking

가장 많이 반복된 비판은 응답에 과도한 thinking 시간을 쓴다는 것이다.
핵심 원인으로 지목된 것은 reasoning_effort 기본값이 xhigh인데 이 변경이 충분히 고지되지 않았다는 점이다.

구체적 사례가 여럿 남아 있다.
Mac mini M4 Pro 64GB에서 "svg owl" 한 줄 프롬프트에 17분 12초를 thinking하며 36.3KiB의 thinking 텍스트를 출력한 보고가 있다.
레트로 사이드스크롤 게임 캐릭터를 HTML 단일 파일로 만들라는 프롬프트에 49분 16초를 기다리다 포기했다는 스레드도 있다.
삼각수에 대한 정보 요청에 40분 넘게 thinking하며 4만 개의 thinking 토큰을 생성했다는 보고, MTP 지원 여부를 묻는 질문에 툴 콜 50회와 약 85,000 토큰을 소모했다는 보고도 있다.
PPT 작성을 8시간 동안 끝내지 못해 Qwen3.6-27B로 바꾸니 30분에 완료됐다는 사례도 올라왔다.
컨텍스트의 80%가 thinking으로 채워져 실제 코딩에 쓸 여지가 부족하다는 지적도 반복된다.

완화책은 대체로 reasoning_effort를 낮추는 방향으로 모인다.

```
--chat-template-kwargs '{"preserve_thinking":true,"reasoning_effort":"medium"}'
```

reasoning budget을 직접 자르고 마무리 문장을 강제하는 방식도 공유됐다.
다만 medium이 low보다 thinking 분량이 적더라는 역설적 보고, low에서 xhigh보다 결과가 좋았다는 보고가 함께 있어 단계별 동작이 직관과 어긋난다는 인상이 남는다.
thinking을 너무 이르게 자르면 오히려 품질이 떨어진다는 반론, 어려운 작업에서는 긴 thinking이 나쁜 것이 아니며 다른 로컬 모델도 같은 경향을 보인다는 반론도 나왔다.
LM Studio가 reasoning_effort 드롭다운을 노출하지 않아 기본 xhigh에 갇힌다는 점도 체감 악화에 기여했다.

관점을 바꾸자는 의견도 있다.
원시 thinking 토큰 수가 아니라 태스크 성공률과 태스크당 소요 시간으로 봐야 한다는 정리다.

### Dense 27B의 구조적 페널티

27B가 Dense라는 점이 로컬 실행에서 구조적 불리함으로 작용한다는 관찰은 여러 환경에서 재현됐다.
AMD 7840U CPU 전용 환경에서 27B Dense가 약 4 tok/s인 반면 Qwen3.6-35B-A3B MoE는 약 20 tok/s로 5배 빨랐다.
M1 Max에서는 Dense 27B가 9.5 tok/s, 3.6-35B-A3B가 60 tok/s대였다.
Mac mini M4 Pro에서는 27B가 12.75 tok/s에 21,769 토큰을 소모한 반면 같은 머신의 Qwen3.6-35B-A3B MLX는 1.59초 thinking에 2,398 토큰, 80.83 tok/s를 냈다.
MoE는 일부를 시스템 RAM에 두어도 치명적 손실이 없지만 Dense는 전량을 VRAM에 올리지 않으면 급격히 느려진다는 정리도 나왔다.

KV 캐시 효율도 약점으로 지목된다.
32K 컨텍스트에 2.5GB의 VRAM이 들어가 128K도 맞추기 어렵다는 보고, 32GB VRAM에서 200K가 한계라는 보고가 있다.
2x RTX A6000에서 경쟁 모델은 128K 컨텍스트로 동시 24개를 처리하는데 27B는 6개에 그친다는 비교도 나왔다.
다만 토큰당 캐시 사용량 자체는 64KB 대 80KB로 Qwen이 다소 유리하며 격차가 크지 않다는 반론, vLLM 실사용에서는 오히려 컨텍스트 오버헤드가 적었다는 반론도 함께 있다.

이 때문에 Hugging Face에는 35B-A3B 형태의 MoE 버전을 요청하는 스레드가 최소 일곱 개 개설됐다.
27B가 로컬에서 처음으로 실제로 쓸만한 모델이라는 평가와, 동급 MoE 대비 속도 페널티가 크다는 평가가 동시에 존재하는 상태다.

긍정 평가의 축은 품질과 자율성이다.
같은 엔진 설정에서 3.6 대비 60% 이상 빠르고 thinking 루프가 줄었다는 보고, 한두 문장 지시만으로 몇 시간을 스스로 진행했다는 보고, 스팸 필터링 자체 벤치마크에서 지금까지 테스트한 어떤 모델보다 좋은 점수를 냈다는 보고가 있다.
반대로 세계 지식과 의도 파악은 Opus급과 격차가 있고 50만 LOC 이상 코드베이스에서 혼란스러워진다는 지적, 잘 정의된 작업의 실행자로 범위를 한정해야 한다는 절충론도 나왔다.

### 벤치마크 회의론

벤치마크에 대한 커뮤니티 정서는 회의적인 편이다.
Qwen 모델이 벤치마크 점수 대비 실사용 품질에서 아쉽다는 인상이 반복적으로 언급되고, 벤치마크와 실사용의 상관이 과거에도 미덥지 않았다는 평가가 있다.
27B가 Opus 4.7보다 100배 작은데 정말 100배 파라미터 효율적이겠느냐는 회의도 제기됐다.
비교 대상이 Opus 4.6 Max로 잡힌 점, 모델 카드가 지능 지표는 보여주면서 소요 시간이나 토큰 사용량 지표는 제시하지 않은 점이 함께 지적됐다.
이에 대해 Opus 4.6도 최대 추론 노력 설정이므로 공정한 대결이라는 반박, 벤치마크 수치를 내려면 상당한 일반화 능력이 필요하다는 반박도 있었다.

다만 여기서 선을 그어야 한다.
수집된 논의 전체에서 Qwen3.8이 특정 벤치마크를 학습셋에 넣었다고 직접 주장하는 근거는 확인되지 않았다.
회의론은 어디까지나 정서와 간접 정황 수준이며, 오염을 사실로 서술할 근거는 없다.

독립 재현이 부족하다는 지적은 별개로 유효하다.
공개 당일 시점에 27B의 독립 벤치마크 재현이 발견되지 않았다는 서드파티 조사가 있고, QwenSWEBench와 CoWorkBench, RecreationBench는 Qwen이 설계한 것이라 외부 감사 가능성이 낮다는 정리가 있다.
학습 코퍼스, 토큰 수, 지식 컷오프, 포스트트레이닝 레시피, 안전성 평가, 지원 언어 수는 모두 미공개다.

직접 돌린 소규모 비교 결과들은 방향이 엇갈린다.
MLX 기반 비교표에서 Qwen3.8-27B는 GSM8K 93.3%와 MATHQA 46.7%로 Qwen3.6-35B-A3B 계열의 96.7%, 60.0%에 뒤졌고, LIVECODEBENCH는 43.3%로 앞섰지만 소요 시간이 1040.4초 대 283.7초로 약 4배 길었다.
CVE 탐지 벤치마크에서는 Qwen 3.8이 Opus 5와 같은 81.3%로 DeepSeek v4 Pro의 87.5%에 이어 공동 2위였다.
코딩에서 GLM 5.2가 더 낫다는 평가와, 로컬에서 GLM-5.2급이 돌아간다는 평가가 동시에 존재한다.

## 결론

Qwen3.8은 하나의 모델이 아니라 성격이 크게 다른 세 갈래로 봐야 하는 시리즈다.
27B는 Apache-2.0에 비전을 포함하고 thinking을 끌 수 있는 단일 GPU 모델이고, 2.4T-A95B는 커스텀 라이선스에 텍스트 전용이며 thinking을 끌 수 없는 클러스터급 MoE다.
Qwen3.8-Max는 이 둘 중 어느 쪽으로도 대체되지 않는 API 전용 상위판으로, 비전 입력과 non-thinking, 기본 1M 컨텍스트, 빌트인 도구를 독점한다.

아키텍처는 배율만 다른 동일 패턴이다.
full_attention_interval 4, Gated DeltaNet 3에 Gated Attention 1, head_dim 256, partial_rotary_factor 0.25, rope_theta 1e7, vocab 248,320, 네이티브 262,144, MTP 1레이어 내장이 두 모델에 공통이며 FFN과 512 expert MoE로만 갈린다.
확장 최대치는 카드와 레시피 사이에 1,000,000과 1,010,000이 엇갈리므로 그대로 병기해 두는 편이 안전하다.

벤치마크는 원 수치와 각주를 함께 읽어야 한다.
공식 표에는 인하우스 벤치마크가 섞여 있고 harness와 심사 모델이 항목마다 다르다.
독립 평가에서 Artificial Analysis Index 56점과 Frontend Code Arena 4위로 프런티어권 진입은 확인되지만, 같은 표 안의 HLE 43.6은 비교군 최하위다.

배포 실무에서 가장 먼저 확인할 세 가지는 MTP depth, 양자화 경로, 채팅 템플릿이다.
MTP는 2.4T FP8 TP16에서 130에서 307 tok/s로 이득이 크지만 depth 1은 권장되지 않고, Ollama GGUF 경로에서는 최대 2.3배 손실로 부호가 뒤집힌다.
공식 양자화는 FP8뿐인데 공식 레시피가 서드파티 NVFP4를 저지연 경로로 지목하므로 검증 책임이 사용자에게 남는다.
채팅 템플릿의 raise_exception은 Claude Code와 Codex 연동을 기본 상태에서 깨뜨리며 Ollama는 0.32.14에서 해결됐지만 llama.cpp 경로는 아직 열려 있다.

마지막으로 reasoning_effort 기본값이 xhigh라는 점은 도입 전에 반드시 확인해야 한다.
이 값 하나가 단순 질문에 수십 분과 수만 토큰을 쓰게 만드는 사례의 상당 부분을 설명하며, 커뮤니티 부정 평가의 가장 큰 단일 원인이었다.

## Reference

- [Qwen3.8-27B 모델 카드](https://huggingface.co/Qwen/Qwen3.8-27B)
- [Qwen3.8-27B-FP8 모델 카드](https://huggingface.co/Qwen/Qwen3.8-27B-FP8)
- [Qwen3.8-2.4T-A95B 모델 카드](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B)
- [Qwen3.8-2.4T-A95B-FP8 모델 카드](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B-FP8)
- [Qwen3.8 공식 컬렉션](https://huggingface.co/collections/Qwen/qwen38)
- [Qwen3.8-Max License 전문](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B/raw/main/LICENSE)
- [Qwen3.8-27B config.json](https://huggingface.co/Qwen/Qwen3.8-27B/raw/main/config.json)
- [Qwen3.8-2.4T-A95B config.json](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B/raw/main/config.json)
- [Qwen3.8-27B chat_template.jinja](https://huggingface.co/Qwen/Qwen3.8-27B/raw/main/chat_template.jinja)
- [Qwen Cloud qwen3.8-max 사양](https://www.qwencloud.com/models/qwen3.8-max)
- [vLLM Recipes - Qwen3.8-27B](https://recipes.vllm.ai/Qwen/Qwen3.8-27B)
- [vLLM Recipes - Qwen3.8-2.4T-A95B](https://recipes.vllm.ai/Qwen/Qwen3.8-2.4T-A95B)
- [SGLang Cookbook - Qwen3.8-27B](https://docs.sglang.io/cookbook/autoregressive/Qwen/Qwen3.8-27B)
- [SGLang Cookbook - Qwen3.8](https://docs.sglang.io/cookbook/autoregressive/Qwen/Qwen3.8)
- [Ollama qwen3.8 라이브러리](https://ollama.com/library/qwen3.8)
- [ollama#17774 - Claude Code system 메시지 위치 오류 원인 분석](https://github.com/ollama/ollama/issues/17774)
- [ollama#17776 - Apple Silicon MTP 통제 실험](https://github.com/ollama/ollama/issues/17776)
- [llama.cpp#27107 - llama-server 템플릿 프로빙 400 오류](https://github.com/ggml-org/llama.cpp/issues/27107)
- [llama.cpp#27139 - Codex 연동 실패와 3.6 템플릿 우회](https://github.com/ggml-org/llama.cpp/issues/27139)
- [llama.cpp#27122 - split-mode tensor와 MTP CUDA 락업](https://github.com/ggml-org/llama.cpp/issues/27122)
- [llama.cpp#27090 - YaRN 4배 확장 시 520K prefill 크래시](https://github.com/ggml-org/llama.cpp/issues/27090)
- [vllm#52475 - turboquant KV 캐시와 MTP 반복 붕괴](https://github.com/vllm-project/vllm/issues/52475)
- [sglang#34895 - NVFP4 lm_head weight_scale 유실](https://github.com/sgl-project/sglang/issues/34895)
- [sglang#34872 - DGX Spark FP8 검증 리포트](https://github.com/sgl-project/sglang/issues/34872)
- [lemonade#3160 - Strix Halo 동시 요청 생성 오염](https://github.com/lemonade-sdk/lemonade/issues/3160)
- [HF Discussion - 2.4T 오픈웨이트 기능 제거 비판](https://huggingface.co/Qwen/Qwen3.8-2.4T-A95B/discussions/13)
- [HF Discussion - 모델 카드 기본값 논쟁](https://huggingface.co/Qwen/Qwen3.8-27B/discussions/84)
- [HF Discussion - 양자화 KLD 벤치마크](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/discussions/49)
- [HF Discussion - Ryzen AI Max+ 395 종합 벤치마크](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/discussions/41)
- [HF Discussion - vLLM MTP depth 튜닝](https://huggingface.co/Qwen/Qwen3.8-27B-FP8/discussions/9)
- [Hacker News - Qwen 3.8 27B 스레드](https://news.ycombinator.com/item?id=49299605)
- [Hacker News - Qwen3.8-Max 발표 스레드](https://news.ycombinator.com/item?id=49150470)
- [Hacker News - Qwen3.8-2.4T 오픈웨이트 스레드](https://news.ycombinator.com/item?id=49273478)
- [Hacker News - Artificial Analysis 순위 논란 스레드](https://news.ycombinator.com/item?id=49200652)
- [The Decoder - Qwen3.8 오픈웨이트 공개](https://the-decoder.com/alibabas-qwen-team-releases-qwen-3-8-models-with-open-weights-under-the-apache-2-0-license/)
- [TechRepublic - Qwen3.8-Max 가격과 엔터프라이즈 함의](https://www.techrepublic.com/article/news-alibaba-qwen3-8-max-pricing-open-weights/)
- [OfficeChai - Artificial Analysis Intelligence Index 56점](https://officechai.com/ai/qwen-3-8-max-scores-56-on-artificial-analysis-intelligence-index-ahead-of-all-us-companies-except-anthropic-and-openai/)
- [Forkast - 라이선스 비판](https://forkast.news/open-weights-closed-revenue-ceiling-alibabas-qwen-3-8-license-is-a-platform-play-not-a-gift/)
- [Open Source For You - 매출 셰어 방침](https://www.opensourceforu.com/2026/08/alibaba-to-introduce-revenuesharing/)
- [explainx - 오픈웨이트 체크포인트 실물 검증](https://www.explainx.ai/blog/qwen3-8-max-open-weights-live-hugging-face-august-2026)
- [BigGo - 중국 대형 모델 아키텍처 수렴 분석](https://finance.biggo.com/news/92b1d4b2-76e9-4342-b637-ec1bc0e4f067/)
- [Latent Space - Qwen3.8 Max 24T와 27B 종합](https://www.latent.space/p/ainews-qwen-38-max24t-and-27b-new)
- [kingy.ai - Qwen3.8-27B 스펙과 로컬 하드웨어 점검](https://kingy.ai/blog/qwen3-8-27b-specs-benchmarks-local-hardware/)
