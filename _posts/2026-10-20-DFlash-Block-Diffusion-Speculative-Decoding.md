---
layout: post
title: "DFlash: Block Diffusion for Flash Speculative Decoding 논문 정리"
author: 'Juho'
date: 2026-10-20 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, Benchmark, vLLM]
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
2. [배경과 관련 연구](#배경과-관련-연구)
   - [Speculative Decoding](#speculative-decoding)
   - [Diffusion Language Model](#diffusion-language-model)
   - [Diffusion 기반 Speculative Decoding](#diffusion-기반-speculative-decoding)
3. [Preliminaries: 속도 향상의 구조](#preliminaries-속도-향상의-구조)
4. [방법론](#방법론)
   - [추론 구조](#추론-구조)
   - [KV Injection 메커니즘](#kv-injection-메커니즘)
   - [학습 방법](#학습-방법)
5. [실험 설정](#실험-설정)
6. [주요 결과](#주요-결과)
   - [Instruct 모델](#instruct-모델)
   - [Reasoning 모델](#reasoning-모델)
   - [SGLang 서빙 환경](#sglang-서빙-환경)
   - [추가 모델과 vLLM](#추가-모델과-vllm)
   - [Long Context 적응](#long-context-적응)
7. [Ablation Study](#ablation-study)
8. [한계와 주의사항](#한계와-주의사항)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

DFlash는 UC San Diego의 Jian Chen, Yesheng Liang, Zhijian Liu가 발표한 speculative decoding 프레임워크입니다.
논문은 ICML 2026(PMLR 306)에 게재되었습니다.
핵심 아이디어는 가벼운 block diffusion 모델을 draft 모델로 사용해 여러 draft 토큰을 한 번의 forward pass로 병렬 생성하는 것입니다.
여기에 타깃 모델에서 추출한 hidden feature를 draft 모델의 모든 레이어 KV cache에 주입해 draft 품질을 끌어올립니다.

autoregressive LLM은 성능이 뛰어나지만 순차적 디코딩 때문에 latency가 높고 GPU 활용률이 낮습니다.
speculative decoding은 빠른 draft 모델이 제안한 토큰을 타깃 LLM이 병렬로 검증하는 방식으로 이 병목을 완화하지만, 기존 방법은 여전히 autoregressive drafting에 의존합니다.
반면 diffusion LLM은 병렬 생성이 가능하지만 단독으로는 autoregressive 모델보다 품질이 낮습니다.
DFlash는 diffusion 모델을 drafting 단계에만 한정해 두 방식의 장점을 결합합니다.

논문이 보고한 핵심 수치는 다음과 같습니다.

| 항목 | 결과 |
|------|------|
| 최대 lossless 가속 | 6배 이상 (Qwen3-8B, 최대 6.1배) |
| EAGLE-3 대비 | 최대 약 2.5배 높은 speedup |
| Qwen3 greedy 평균 speedup | 약 4.9배 (EAGLE-3 tree 16 대비 2.4배) |
| Qwen3 sampling 평균 speedup | 약 4.1배 (EAGLE-3 대비 2.2배) |
| SGLang 서빙 최대 speedup | 5.1배 (Qwen3-8B, B200) |

## 배경과 관련 연구

LLM 추론은 각 출력이 이전 전체 컨텍스트에 의존하는 token-by-token 생성 과정이 지배합니다.
이 순차성 때문에 추론이 느리고 memory-bound 상태가 되며, 최신 GPU를 충분히 활용하지 못합니다.
긴 Chain-of-Thought를 생성하는 reasoning 모델이 등장하면서 이 병목은 더 심각해졌습니다.

### Speculative Decoding

초기 방법(Leviathan et al., 2023)은 작은 draft 모델이 토큰 시퀀스를 제안하고 큰 타깃 모델이 이를 병렬로 검증합니다.
Medusa는 외부 draft 모델 대신 base LLM에 여러 prediction head를 붙이고 tree attention으로 병렬 검증합니다.
EAGLE 시리즈는 frozen 타깃 모델의 feature-level 컨텍스트를 활용합니다.
EAGLE-1은 미래 hidden state 분포를 예측해 acceptance rate를 높였고, EAGLE-2는 adaptive drafting tree를, EAGLE-3는 학습 목표를 개선해 speedup을 확장했습니다.
그러나 이들 대부분은 autoregressive drafting에 의존하므로 본질적으로 순차적입니다.
논문은 이러한 직렬 drafting이 비효율적일 뿐 아니라 오류 누적에 취약해, 실질적인 speedup이 약 2~3배 수준에 머문다고 지적합니다.

### Diffusion Language Model

diffusion LLM(dLLM)은 masked 토큰을 병렬로 예측하는 방식으로 autoregressive 생성의 대안을 제시합니다.
LLaDA는 dLLM을 수십억 파라미터 규모로 확장해 LLaMA-3.1-8B에 견줄 만한 성능을 보였습니다.
하지만 완전 병렬 diffusion 모델은 고정 길이 생성 문제가 있고 효율적인 KV cache 지원이 부족합니다.
Block diffusion 모델은 시퀀스를 block 단위로 denoise하여 병렬성과 autoregressive 구조를 결합합니다.
Fast-dLLM v2와 SDAR는 사전학습된 autoregressive LLM을 block diffusion 변형으로 전환했습니다.
그럼에도 기존 dLLM은 최신 autoregressive 모델보다 품질이 낮고, 품질 유지를 위해 많은 denoising step이 필요해 실제 추론 속도가 제한됩니다.

### Diffusion 기반 Speculative Decoding

TiDAR는 diffusion과 autoregressive 목표를 함께 학습해 diffusion으로 병렬 "thinking", autoregressive로 순차 "talking"을 수행하지만 최종 생성 품질이 아직 lossless는 아닙니다.
Samragh et al.(2025)은 autoregressive LLM이 미래 토큰 정보를 암묵적으로 담고 있음을 관찰하고, LoRA adapter로 병렬 drafting을 가능하게 했습니다.
DiffuSpec과 SpecDiff-2는 대형 사전학습 dLLM(예: 7B 파라미터)을 drafter로 사용합니다.
acceptance length는 길지만 메모리와 drafting latency 부담이 커서 실제 speedup은 3~4배 수준에 그칩니다.
PARD는 작은 autoregressive 모델이 diffusion 방식의 병렬 생성을 흉내 내도록 학습하지만, 작은 모델의 표현력 한계로 acceptance length가 짧고 speedup 상한이 약 3배입니다.

논문은 이 상황을 바탕으로 "가벼우면서도 정확한 diffusion drafter를 만들 수 있는가"라는 질문을 던집니다.
답으로 제시하는 핵심 통찰은 "the target knows best", 즉 타깃 모델의 hidden feature를 draft 모델의 컨텍스트로 쓰는 것입니다.

## Preliminaries: 속도 향상의 구조

타깃 모델 Mt와 draft 모델 Md가 있을 때, 각 디코딩 사이클에서 draft 모델은 B개의 토큰을 제안하고 타깃 모델이 이를 병렬 검증합니다.
Sadhukhan et al.(2025)을 따르면 토큰당 평균 latency는 다음과 같습니다.
아래 수식의 기호(B, τ, λ)는 설명을 위해 이 포스트에서 붙인 표기입니다.

```text
L = (T_draft + T_verify) / τ          (1)

T_draft : draft 토큰 생성 시간
T_verify: 검증 비용
τ       : 사이클당 기대 수락 토큰 수 (1 이상 B+1 이하, 타깃 모델의 bonus 토큰 포함)
speedup η = L_target / L
```

따라서 speedup을 높이려면 acceptance length τ를 늘리거나 drafting 비용 T_draft를 줄여야 합니다.

autoregressive drafter는 토큰을 순차 생성하므로 drafting 비용이 B에 비례해 증가합니다.

```text
autoregressive: T_draft = B × t_step      (2)
diffusion     : T_draft = t_parallel      (3)
```

latency를 감당하기 위해 autoregressive drafter는 매우 얕은 구조(EAGLE-3는 transformer 레이어 1개)로 제한됩니다.
그 결과 B를 늘려도 모델 용량 한계 때문에 acceptance length가 빠르게 포화됩니다.
반면 diffusion drafter는 B개 토큰을 한 번의 forward pass로 생성하며, 비슷한 크기의 모델이라면 t_parallel은 B × t_step보다 훨씬 작습니다.
적당한 block 크기에서는 T_draft가 B에 거의 영향을 받지 않습니다.

이 특성 덕분에 diffusion drafter는 latency 손해 없이 더 깊고 표현력 있는 구조를 쓸 수 있습니다.
논문에 따르면 16개 토큰을 생성하는 5-layer DFlash draft 모델이 8개 토큰을 생성하는 EAGLE-3보다 drafting latency가 낮으면서 acceptance length는 더 높습니다.

## 방법론

### 추론 구조

DFlash의 추론 과정은 다음 순서로 진행됩니다.

1. 타깃 모델이 입력 프롬프트에 대해 일반적인 prefill을 수행해 첫 토큰을 생성합니다.
2. 이 과정에서 얕은 층부터 깊은 층까지 균일하게 샘플링한 고정된 레이어 집합의 hidden representation을 추출합니다.
3. 추출한 hidden state를 concat한 뒤 가벼운 projection 레이어로 융합해 compact한 target context feature를 만듭니다.
4. 이 feature를 draft 모델의 모든 레이어 Key/Value projection에 주입하고 KV cache에 저장해 drafting 반복 간에 재사용합니다.
5. draft 모델은 다음 block의 mask 위치를 bidirectional attention으로 한 번의 forward pass에 병렬 디코딩합니다.
6. 타깃 모델이 draft block을 병렬 검증합니다.

타깃 모델 조건 없이 학습한 5-layer block diffusion draft 모델은 수학 벤치마크에서 약 2~3배 수준의 제한적인 speedup만 보였습니다.
타깃 모델의 내부 표현에 접근하지 못하면 drafter가 미래 토큰을 처음부터 예측해야 하기 때문입니다.
반면 대형 autoregressive 타깃 모델의 hidden representation은 token-level logit보다 훨씬 많은 정보를 담고 있으며, 장거리 의존성과 task 의미, 그리고 미래 토큰 정보까지 암묵적으로 포함합니다.

EAGLE-3도 타깃 모델의 hidden feature를 활용하지만, 이를 draft 모델의 token embedding과 융합해 입력으로만 넣습니다.
이 경우 draft 모델이 깊어질수록 타깃 정보가 희석되어 레이어를 늘려도 acceptance length 향상이 줄어듭니다.
DFlash는 target context feature를 지속적인 컨텍스트 정보로 보고 모든 draft 레이어에 직접 주입하므로, draft 레이어 수에 따라 acceptance length가 효과적으로 늘어납니다.
논문은 결과적으로 draft 모델이 타깃 모델의 깊은 컨텍스트 feature를 활용하는 "diffusion adapter" 역할을 한다고 설명합니다.

### KV Injection 메커니즘

선택된 타깃 레이어(5개)의 hidden state를 concat한 뒤 draft hidden 차원으로 한 번 projection합니다.

```text
H_t = RMSNorm( W_c [ H(l1); ...; H(l5) ] )

레이어 i에서:
Q_i = W_i^Q · H_d
K_i = [ W_i^K · H_t ; W_i^K · H_d ]_seq
V_i = [ W_i^V · H_t ; W_i^V · H_d ]_seq
```

H_t는 모든 draft 레이어가 공유합니다.
query는 draft 토큰(H_d)에서만 만들어지고, key/value는 타깃 feature와 draft 토큰 양쪽에서 만들어져 시퀀스 방향으로 이어 붙습니다.
즉 타깃 feature는 masked block draft 토큰을 위한 추가 KV 항목으로만 쓰이며, draft 모델의 Q projection, output projection, self-attention 업데이트, FFN을 거치지 않습니다.

메모리 오버헤드는 작습니다.
추가 파라미터는 공유 projection W_c(D × 5D 크기)뿐입니다.

| 항목 | 값 (Qwen3.5-35B-A3B, D=2048, BF16) |
|------|------|
| W_c 파라미터 메모리 | 5 × 2048 × 2048 × 2 ≈ 42 MB |
| 타깃 모델 크기 | 약 70 GB |
| projection 입력 activation (batch 1, seq 2048) | 약 40 MB |
| projection 출력 activation (batch 1, seq 2048) | 약 8 MB |
| 디코딩 중 임시 activation (block 16) | 400 KB 미만 |

### 학습 방법

DFlash draft 모델은 frozen autoregressive 타깃 모델의 출력에 block 단위 diffusion 예측을 정렬하도록 학습됩니다.
표준 block diffusion 학습을 그대로 쓰지 않고 다음과 같은 수정을 가했습니다.

**KV injection**
프롬프트와 응답으로 이루어진 clean 시퀀스 전체를 타깃 모델에 통과시켜 모든 토큰의 hidden feature를 추출·융합한 뒤, 추론과 동일하게 draft 모델의 Key/Value projection으로 주입합니다.

**Masked block 랜덤 샘플링**
표준 block diffusion 학습은 응답을 균일하게 block으로 나누고 각 block 안의 임의 위치를 mask합니다.
DFlash는 응답에서 anchor 토큰을 랜덤 샘플링해 각 anchor를 block의 첫 위치로 두고 나머지 위치를 mask합니다.
draft 모델은 다음 (block size - 1)개 토큰을 병렬 예측하도록 학습됩니다.
이는 추론 시 draft 모델이 항상 타깃 모델이 만든 clean 토큰(이전 검증 단계의 bonus 토큰)을 조건으로 삼는 상황과 정확히 일치합니다.
anchor 위치를 무작위로 두면 더 다양한 target context feature에 노출되어 데이터 효율과 커버리지도 높아집니다.

**Sparse attention mask**
학습 시 모든 block을 하나의 시퀀스로 이어 붙이고 sparse attention mask로 함께 처리합니다.
토큰은 같은 block 안에서는 bidirectional로, 그리고 대응하는 주입된 target context feature에 attention하지만 다른 block으로의 attention은 금지됩니다.
이 구조 덕분에 여러 draft block을 Flex Attention으로 한 번의 forward/backward pass 안에서 효율적으로 학습할 수 있습니다.

**효율적인 long-context 학습**
EAGLE-3 같은 방법은 비용이 큰 training-time test 때문에 긴 컨텍스트 학습이 어렵습니다.
DFlash는 시퀀스당 masked block 수를 고정하고 매 epoch마다 anchor 위치를 새로 샘플링해, 학습 비용을 일정하게 유지하면서 data augmentation 효과를 얻습니다.

**Loss weighting**
draft block 앞쪽 위치에서 오류가 나면 뒤의 모든 토큰이 무효화되므로 앞쪽 예측이 acceptance length에 훨씬 중요합니다.
이를 반영해 block 내 위치 k의 토큰에 지수적으로 감소하는 가중치를 cross-entropy loss에 적용합니다.

```text
w_k = exp( -(k - 1) / λ )      (4)

λ: 감쇠율 (block size 16 → 7, block size 10 → 5, block size 8 → 4)
```

uniform 가중치 대비 수렴이 빠르고 더 높은 acceptance length를 얻습니다.

**Embedding과 LM head 공유**
draft 모델은 타깃 모델의 token embedding 레이어와 LM head를 공유하고 학습 중 frozen 상태로 둡니다.
draft Transformer 레이어만 업데이트되므로 학습 파라미터 수가 줄고, draft 모델이 타깃 모델의 표현 공간에 밀접하게 정렬된 경량 diffusion adapter로 동작하게 됩니다.

## 실험 설정

| 항목 | 내용 |
|------|------|
| 타깃 모델 | LLaMA-3.1 Instruct 8B, Qwen3 4B / 8B / Coder-30B-A3B-Instruct (부록에서 Qwen3.5, GPT-OSS 추가) |
| Math 벤치마크 | GSM8K, MATH(MATH-500), AIME25 |
| Code 벤치마크 | HumanEval, MBPP, LiveCodeBench |
| Chat 벤치마크 | MT-Bench, Alpaca |
| 지표 | 평균 acceptance length τ, autoregressive 대비 end-to-end speedup |
| 하드웨어 | NVIDIA H200 (별도 표기 없을 때), SGLang·vLLM 실험은 B200 |
| 학습 데이터 | NVIDIA Nemotron Post-Training Dataset V2 + CodeAlpaca 약 800K 샘플 |
| 데이터 구성 | 원본 응답 대신 타깃 모델이 생성한 응답으로 재구성 |
| draft 레이어 수 | 5 (Qwen3 Coder는 8) |
| block size | 16 (LLaMA 3.1은 10) |
| 타깃 hidden feature | 두 번째 레이어부터 끝에서 세 번째 레이어 사이에서 균일하게 고른 5개 레이어 |
| 비교 대상 | vanilla autoregressive decoding, EAGLE-3 |

다른 dLLM 기반 speculative decoding 방법(TiDAR, Samragh et al., DiffuSpec, SpecDiff-2)은 오픈소스 구현이 없어 비교에서 제외했습니다.
Qwen3의 EAGLE-3 체크포인트는 AngelSlim(Tencent)이 공개한 것을, LLaMA-3.1-Instruct는 EAGLE-3 팀의 공식 체크포인트를 사용했습니다.

학습 하이퍼파라미터는 다음과 같습니다.

| 항목 | 값 |
|------|------|
| Optimizer | AdamW |
| Epoch | 6 |
| Learning rate | 6 × 10^-4 |
| Gradient clipping | 1.0 |
| Scheduler | cosine, warmup ratio 0.04 |
| 최대 시퀀스 길이 | 3072 (Qwen3-Coder는 4096) |
| 시퀀스당 anchor 수 | 512 |

학습은 online(매 step마다 타깃 hidden feature 계산)과 offline(타깃 hidden feature를 미리 계산해 캐시) 방식 모두 가능합니다.

## 주요 결과

### Instruct 모델

Qwen3 모델에서 thinking mode를 끄고, Transformers backend로 최대 2048 토큰을 생성해 평가했습니다.
EAGLE-3는 DFlash block size 16과 drafting 예산을 맞춘 tree size 16, 그리고 EAGLE-3 논문처럼 acceptance length를 최대화하는 tree size 60 두 설정으로 비교했습니다.
두 설정 모두 draft step 7, top-k 10입니다.

Qwen3-8B, temperature 0 결과(speedup / τ)입니다.

| 벤치마크 | EAGLE-3 (16) | EAGLE-3 (60) | DFlash (16) |
|------|------|------|------|
| GSM8K | 1.94배 / 3.23 | 2.23배 / 3.71 | 5.15배 / 6.54 |
| MATH-500 | 1.81배 / 3.02 | 2.05배 / 3.49 | 6.08배 / 7.87 |
| AIME25 | 1.79배 / 3.00 | 2.05배 / 3.44 | 5.62배 / 7.08 |
| HumanEval | 1.89배 / 3.17 | 2.17배 / 3.65 | 5.14배 / 6.50 |
| MBPP | 1.69배 / 2.82 | 1.93배 / 3.25 | 4.65배 / 5.95 |
| LiveCodeBench | 1.57배 / 2.65 | 1.81배 / 3.03 | 5.51배 / 7.27 |
| MT-Bench | 1.63배 / 2.83 | 1.90배 / 3.26 | 2.75배 / 4.24 |
| 평균 | 1.76배 / 2.96 | 2.02배 / 3.40 | 4.86배 / 6.49 |

모델과 temperature별 평균(speedup / τ)은 다음과 같습니다.

| 설정 | EAGLE-3 (16) | EAGLE-3 (60) | DFlash (16) |
|------|------|------|------|
| Qwen3-4B, temp 0 | 1.81배 / 3.05 | 2.08배 / 3.48 | 4.91배 / 6.54 |
| Qwen3-8B, temp 0 | 1.76배 / 2.96 | 2.02배 / 3.40 | 4.86배 / 6.49 |
| Qwen3-4B, temp 1 | 1.72배 / 2.95 | 1.93배 / 3.36 | 4.24배 / 5.69 |
| Qwen3-8B, temp 1 | 1.68배 / 2.83 | 1.88배 / 3.26 | 4.03배 / 5.48 |

DFlash는 모든 task와 설정에서 EAGLE-3를 앞섭니다.
greedy decoding에서 평균 4.9배 speedup으로 EAGLE-3 (16) 대비 2.4배, sampling(temperature 1)에서는 4.1배 speedup으로 EAGLE-3 대비 2.2배 향상입니다.
tree size 60인 EAGLE-3보다도 acceptance length가 높으면서 verification 오버헤드는 훨씬 적습니다.
다만 MT-Bench 같은 chat task에서는 수학·코드 task보다 speedup 폭이 작습니다.

### Reasoning 모델

Qwen3의 thinking mode를 켜고 Transformers로 평가했으며, draft 모델은 reasoning trace를 포함한 타깃 모델 출력으로 학습했습니다.
세 개 벤치마크(GPQA, MATH-500, AIME25)에서 측정한 결과 범위는 다음과 같습니다.

| 모델 | Temperature | Speedup 범위 | τ 범위 |
|------|------|------|------|
| Qwen3-4B | 0 | 4.23~4.59배 | 5.23~5.74 |
| Qwen3-4B | 1 | 3.64~3.93배 | 4.55~4.89 |
| Qwen3-8B | 0 | 4.17~4.64배 | 5.17~5.82 |
| Qwen3-8B | 1 | 3.70~4.03배 | 4.65~5.06 |

논문은 대략 4.5배(greedy)와 3.9배(sampling) 수준의 speedup으로 요약하며, 생성 시간이 긴 reasoning 모델 배포에서 특히 가치가 크다고 설명합니다.

### SGLang 서빙 환경

오픈소스 추론 프레임워크 SGLang에서 단일 B200 GPU, FlashAttention-4(FA4) backend, Spec-v2 scheduling overlap을 켜고 평가했습니다.
concurrency 1~32 구간의 speedup과 평균 τ입니다.

| 모델 | Task | C=1 | C=4 | C=8 | C=16 | C=32 | 평균 τ |
|------|------|------|------|------|------|------|------|
| Qwen3-4B | Math500 | 4.8배 | 4.3배 | 4.1배 | 3.5배 | 2.9배 | 8.01 |
| Qwen3-4B | HumanEval | 4.0배 | 3.6배 | 3.2배 | 2.7배 | 2.2배 | 6.63 |
| Qwen3-8B | Math500 | 5.1배 | 4.5배 | 4.5배 | 3.9배 | 2.8배 | 8.01 |
| Qwen3-8B | HumanEval | 4.2배 | 3.6배 | 3.6배 | 3.0배 | 2.4배 | 6.50 |
| Qwen3-Coder-30B-A3B | HumanEval | 3.5배 | 3.0배 | 3.2배 | 3.2배 | 3.1배 | 8.09 |
| Qwen3-Coder-30B-A3B | LCB | 2.6배 | 2.4배 | 2.3배 | 2.4배 | 2.3배 | 6.42 |
| Qwen3-Coder-30B-A3B | MBPP | 3.2배 | 3.0배 | 3.2배 | 3.3배 | 3.1배 | 7.23 |

처리량 기준으로 보면 Qwen3-8B Math500에서 baseline이 concurrency 1에서 230 tok/s, 32에서 5694 tok/s인 반면 DFlash는 각각 1175 tok/s, 16076 tok/s입니다.
Qwen3-4B Math500은 concurrency 32에서 baseline 7136 tok/s 대비 DFlash 20417 tok/s입니다.
dense 모델은 concurrency가 높아질수록 speedup이 줄어드는 반면, Qwen3-Coder-30B-A3B는 concurrency 전 구간에서 비교적 일정한 speedup을 보입니다.

LLaMA-3.1-8B-Instruct에서는 EAGLE-3와 동일한 학습 데이터(UltraChat, ShareGPT)로 학습한 DFlash(block size 10)를 공식 EAGLE-3 체크포인트와 비교했습니다.
SGLang Spec-v2가 EAGLE-3의 tree 기반 drafting을 지원하지 않아 Spec-v1(scheduling overlap 없음), Flashinfer backend, 단일 B200에서 측정했습니다.
EAGLE-3는 draft step 7, top-k 10에 draft 토큰 10 또는 60개를 사용했습니다.

| Task | 방법 | C=1 | C=4 | C=8 | C=16 | C=32 | 평균 τ |
|------|------|------|------|------|------|------|------|
| GSM8K | Baseline (TPS) | 249 | 923 | 1739 | 3245 | 5349 | - |
| GSM8K | EAGLE-3 (10) | 1.6배 | 1.5배 | 1.4배 | 1.2배 | 1.0배 | 3.49 |
| GSM8K | EAGLE-3 (60) | 1.9배 | 1.6배 | 1.3배 | 0.9배 | 0.6배 | 4.55 |
| GSM8K | DFlash (10) | 2.4배 | 2.2배 | 2.1배 | 1.8배 | 1.6배 | 4.32 |
| HumanEval | Baseline (TPS) | 245 | 922 | 1778 | 3336 | 5854 | - |
| HumanEval | EAGLE-3 (10) | 2.0배 | 1.9배 | 1.8배 | 1.5배 | 1.2배 | 3.62 |
| HumanEval | EAGLE-3 (60) | 2.0배 | 1.7배 | 1.3배 | 0.9배 | 0.6배 | 4.65 |
| HumanEval | DFlash (10) | 2.8배 | 2.6배 | 2.5배 | 2.1배 | 1.8배 | 4.91 |
| Alpaca | Baseline (TPS) | 245 | 906 | 1745 | 3237 | 5434 | - |
| Alpaca | EAGLE-3 (10) | 1.5배 | 1.4배 | 1.4배 | 1.1배 | 0.9배 | 3.11 |
| Alpaca | EAGLE-3 (60) | 1.8배 | 1.5배 | 1.2배 | 0.8배 | 0.5배 | 4.07 |
| Alpaca | DFlash (10) | 2.2배 | 2.0배 | 1.8배 | 1.5배 | 1.4배 | 3.73 |

DFlash는 모든 task, concurrency, EAGLE-3 tree 설정에서 EAGLE-3를 앞섭니다.
EAGLE-3 (60)은 concurrency 16 이상에서 baseline보다 느려지는(1배 미만) 반면, DFlash는 concurrency 32에서도 1.4~1.8배를 유지합니다.

### 추가 모델과 vLLM

부록에서는 SGLang(B200 1장, concurrency 8)에서 더 많은 모델을 평가했습니다.
네이티브 MTP가 있는 Qwen3.5 모델과의 비교(τ / speedup)는 다음과 같습니다.

| 모델 | 방법 | Math500 | HumanEval | MT-Bench |
|------|------|------|------|------|
| Qwen3.5-4B | MTP | 6.5 / 1.5배 | 6.3 / 1.6배 | 5.3 / 1.3배 |
| Qwen3.5-4B | DFlash | 7.1 / 3.0배 | 7.3 / 2.9배 | 5.6 / 2.3배 |
| Qwen3.5-9B | MTP | 6.7 / 1.7배 | 6.6 / 1.7배 | 5.3 / 1.3배 |
| Qwen3.5-9B | DFlash | 7.3 / 3.5배 | 7.9 / 3.4배 | 5.5 / 2.5배 |
| Qwen3.5-35B-A3B | MTP | 6.9 / 1.7배 | 7.1 / 1.6배 | 5.2 / 1.2배 |
| Qwen3.5-35B-A3B | DFlash | 7.2 / 2.4배 | 7.9 / 2.3배 | 5.4 / 1.7배 |

DFlash는 MTP가 있는 경우 모두 MTP보다 높은 speedup을 보였고, Qwen3.5-27B, Qwen3-Coder-Next, GPT-OSS-20B, GPT-OSS-120B 같은 더 큰 모델로도 확장된다고 보고합니다.

vLLM에서는 Qwen3.5-9B로 DFlash 처리량(tok/s)과 autoregressive 대비 speedup을 측정했습니다.

| Task | C=1 | C=8 | C=16 | C=32 |
|------|------|------|------|------|
| Math500 | 849 (4.0배) | 5096 (3.2배) | 7669 (2.5배) | 9836 (1.9배) |
| HumanEval | 969 (4.6배) | 5434 (3.4배) | 8131 (2.7배) | 10258 (2.1배) |
| MT-Bench | 627 (3.0배) | 3536 (2.2배) | 5297 (1.7배) | 6787 (1.3배) |

낮은·중간 concurrency에서 speedup이 크고, 높은 concurrency에서도 처리량 이득을 유지합니다.

### Long Context 적응

4K 컨텍스트로 학습한 base DFlash draft 모델이 적은 fine-tuning으로 긴 컨텍스트에 적응할 수 있는지 확인했습니다.
Qwen3.5-27B draft 모델을 LongAlign-10K의 1.6K 샘플로 3 epoch fine-tuning하고 LongBench 일부 데이터셋에서 acceptance length를 측정했습니다.

| 데이터셋 | 모델 | 1K | 2K | 4K | 8K | 16K | 32K |
|------|------|------|------|------|------|------|------|
| hotpotqa | Base | 4.91 | 4.97 | 4.91 | 4.46 | 3.61 | - |
| hotpotqa | Long | 4.99 | 5.06 | 5.41 | 5.76 | 6.05 | - |
| qasper | Base | 5.27 | 5.50 | 5.17 | 4.17 | 3.57 | - |
| qasper | Long | 5.38 | 5.67 | 5.80 | 5.62 | 6.00 | - |
| gov report | Base | 4.53 | 4.35 | 3.93 | 3.32 | 2.67 | 2.09 |
| gov report | Long | 4.53 | 4.38 | 4.25 | 4.04 | 3.81 | 3.56 |

base 모델은 4K를 넘어서면 acceptance length가 떨어지지만, fine-tuning한 모델은 이를 유지하거나 오히려 높입니다.
논문은 추출한 타깃 feature가 긴 컨텍스트에서도 대표성을 유지하고, draft 모델이 가벼운 적응만으로 장거리 패턴을 학습할 수 있음을 보여준다고 해석합니다.

## Ablation Study

별도 표기가 없으면 ablation 모델은 전체 데이터에서 무작위로 뽑은 100K 샘플로 학습했고, 단일 H200 GPU에서 greedy decoding으로 평가했습니다.

### 타깃 feature 없는 diffusion drafter

타깃 context feature 없이 학습한 5-layer block diffusion draft 모델 결과(speedup / τ)입니다.

| Temperature | GSM8K | Math500 | AIME24 | AIME25 |
|------|------|------|------|------|
| 0 | 2.83배 / 3.38 | 3.73배 / 4.61 | 3.43배 / 4.12 | 3.35배 / 4.07 |
| 1 | 2.76배 / 3.29 | 3.31배 / 4.12 | 2.66배 / 3.23 | 2.65배 / 3.24 |

타깃 조건이 없으면 acceptance length와 speedup이 제한적이라는 점을 확인할 수 있습니다.

### Draft 레이어 수

block size 16, 타깃 hidden feature 5개 조건에서 draft 레이어 수를 바꾼 결과(speedup / τ)입니다.

| 설정 | Math500 | HumanEval | MT-Bench |
|------|------|------|------|
| 3-layer | 4.69배 / 5.64 | 3.90배 / 4.61 | 2.38배 / 3.18 |
| 5-layer | 4.71배 / 5.99 | 3.96배 / 4.94 | 2.35배 / 3.37 |
| 8-layer | 4.64배 / 6.33 | 3.96배 / 5.29 | 2.23배 / 3.50 |

레이어가 깊을수록 acceptance length는 늘지만 drafting latency도 커집니다.
8-layer가 가장 긴 acceptance length를 보이지만, 비용과 품질의 균형 측면에서 5-layer가 전체 speedup이 가장 좋습니다.
최적 레이어 수는 배포 환경에 따라 달라집니다.

### 타깃 hidden feature 수

3-layer draft 모델, block size 16 조건입니다.

| 설정 | Math500 | HumanEval | MT-Bench |
|------|------|------|------|
| hidden feature 3개 | 4.49배 / 5.38 | 3.80배 / 4.47 | 2.32배 / 3.07 |
| hidden feature 5개 | 4.69배 / 5.64 | 3.90배 / 4.61 | 2.38배 / 3.18 |

더 많은 타깃 레이어에서 feature를 뽑으면 의미 정보와 미래 토큰 정보가 풍부해져 draft 품질이 올라갑니다.
다만 offline 학습에서는 타깃 hidden state 캐시에 필요한 저장 공간이 추출 feature 수에 비례해 늘어납니다.

### 학습-추론 block size 불일치

8-layer draft 모델, 타깃 hidden feature 5개 조건입니다.

| 학습 BS | 추론 BS | Math500 | HumanEval | MT-Bench |
|------|------|------|------|------|
| 16 | 16 | 4.64배 / 6.33 | 3.96배 / 5.29 | 2.23배 / 3.50 |
| 16 | 8 | 3.87배 / 5.09 | 3.39배 / 4.44 | 2.12배 / 3.18 |
| 8 | 16 | 3.78배 / 5.02 | 3.24배 / 4.28 | 2.09배 / 3.09 |
| 8 | 8 | 3.97배 / 5.21 | 3.53배 / 4.61 | 2.22배 / 3.29 |

학습과 추론 block size가 일치할 때 block size 16 모델이 수학·코드 task에서 훨씬 높은 acceptance length를 보입니다.
Math500에서 block size 8 모델은 block 전체를 수락하는 비율이 35.7%로, block size 8이 자주 충분히 활용되지 못함을 보여줍니다.
큰 block size로 학습한 모델은 작은 추론 block size로 잘 일반화되지만, 그 반대는 성립하지 않는 비대칭성이 있습니다.
이 성질은 추론 중 동적 block size 스케줄링을 가능하게 합니다.
큰 batch처럼 compute-bound인 환경에서는 큰 block이 verification 비용을 높이므로, block size를 줄이는 것이 전체 speedup에 유리할 수 있습니다.

### KV Injection vs Input Fusion

Qwen3-4B, 5-layer draft 모델, block size 8 조건에서 타깃 feature를 입력층에만 융합하는 방식(EAGLE-3 스타일)과 모든 레이어에 KV로 주입하는 방식을 비교했습니다(τ / speedup).

| Drafting | 변형 | 주입 방식 | GSM8K | HumanEval | MT-Bench |
|------|------|------|------|------|------|
| Autoregressive | EAGLE-3-5L | Input | 4.2 / 2.1배 | 4.3 / 2.2배 | 3.1 / 1.4배 |
| Autoregressive | DFlash-AR | KV | 4.8 / 2.4배 | 4.6 / 2.3배 | 3.4 / 1.5배 |
| Block diffusion | DFlash | Input | 3.5 / 2.9배 | 3.5 / 2.9배 | 2.6 / 2.0배 |
| Block diffusion | DFlash | KV | 4.2 / 3.3배 | 4.0 / 3.2배 | 3.0 / 2.2배 |

autoregressive drafting과 block diffusion drafting 모두에서 KV injection이 input fusion보다 acceptance length가 높습니다.
DFlash는 EAGLE-3-5L과 비슷한 acceptance length를 보이지만 병렬 drafting 덕분에 speedup은 훨씬 높습니다.
즉 DFlash의 이득은 KV injection을 통한 강한 conditioning과 병렬 drafting 두 요소에서 함께 나옵니다.

### Loss decay와 masked block 랜덤 샘플링

block 내 위치별 지수 감쇠 가중치를 적용한 경우, uniform 가중치보다 Math500 acceptance length 기준으로 더 빠르고 더 좋게 수렴합니다.
anchor 랜덤 샘플링 효과는 3-layer draft 모델, 타깃 feature 5개, block size 16, 100K 데이터 조건에서 측정했습니다.

| 설정 | Math500 | HumanEval | MT-Bench |
|------|------|------|------|
| Standard (균일 block 분할) | 4.13배 / 4.94 | 3.29배 / 3.86 | 2.13배 / 2.80 |
| Sample (anchor 랜덤 샘플링) | 4.69배 / 5.64 | 3.90배 / 4.61 | 2.38배 / 3.18 |

## 한계와 주의사항

논문에 별도의 Limitations 섹션은 없지만, 본문에서 다음과 같은 제약과 trade-off가 확인됩니다.

- 비교 범위: 다른 dLLM 기반 speculative decoding 방법(TiDAR, DiffuSpec, SpecDiff-2 등)은 오픈소스 구현이 없어 직접 비교하지 않았습니다.
- concurrency 의존성: SGLang과 vLLM 모두 concurrency가 높아질수록 dense 모델의 speedup이 줄어듭니다(예: Qwen3-4B Math500에서 4.8배에서 2.9배).
- block size와 verification 비용: 큰 batch 같은 compute-bound 환경에서는 큰 block이 verification 비용을 높이며, 이를 위한 adaptive block size 스케줄링은 향후 과제로 남겨 두었습니다.
- block size 일반화의 비대칭: 작은 block size로 학습한 모델은 더 큰 추론 block size로 일반화되지 않습니다.
- 레이어 수 trade-off: draft 레이어를 늘리면 acceptance length는 늘지만 drafting latency가 증가해, 최적 깊이가 배포 환경에 따라 달라집니다.
- 학습 저장 비용: offline 학습 시 타깃 hidden state 캐시 용량이 추출 feature 수에 비례해 증가합니다.
- 긴 컨텍스트: 4K 컨텍스트로 학습한 base drafter는 4K를 넘어서면 acceptance length가 떨어지며, long-context fine-tuning이 필요합니다.
- task별 편차: chat task(MT-Bench, Alpaca)는 수학·코드 task보다 speedup이 작습니다.

## 결론

DFlash는 diffusion 언어 모델을 drafting 단계에만 사용해 병렬성은 살리고, 단독 사용 시의 품질 저하는 타깃 모델 검증으로 피하는 speculative decoding 프레임워크입니다.
타깃 모델의 풍부한 context feature를 모든 draft 레이어 KV에 주입해 높은 acceptance rate를 확보했고, 이를 통해 기존 speculative decoding 방법보다 추론 속도를 크게 끌어올렸습니다.
Qwen3-8B에서 최대 6.1배 lossless 가속, EAGLE-3 대비 최대 약 2.5배 높은 speedup을 보였으며, SGLang과 vLLM 서빙 환경에서도 이득이 확인되었습니다.

논문은 diffusion LLM의 새로운 개발 방향도 제안합니다.
autoregressive 모델과 end-to-end 생성 품질로 경쟁하기보다, 빠르고 정확한 block 예측에 특화된 경량 drafter로 쓰는 방식입니다.
이 관점에서는 병렬성을 극대화하기 위해 denoising step을 공격적으로 줄일 수 있고, 출력 품질은 speculative verification이 보장합니다.
저자들은 SGLang 통합을 함께 진행했으며, 코드와 모델은 프로젝트 페이지 [dflash.z-lab.ai](https://dflash.z-lab.ai){:target="_blank"}에서 안내합니다.

## Reference

- [DFlash: Block Diffusion for Flash Speculative Decoding (arXiv 2602.06036)](https://arxiv.org/abs/2602.06036)
- [DFlash Project Page](https://dflash.z-lab.ai)
