---
layout: post
title: "Pretraining Large Language Models with NVFP4 - 4비트 정밀도 LLM 사전학습 방법론 정리"
author: 'Juho'
date: 2026-10-20 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, Benchmark, AI]
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
2. [NVFP4 포맷](#nvfp4-포맷)
   - [MX 포맷과 MXFP4](#mx-포맷과-mxfp4)
   - [NVFP4의 개선점](#nvfp4의-개선점)
   - [Blackwell Tensor Core 지원](#blackwell-tensor-core-지원)
   - [양자화 절차와 수식](#양자화-절차와-수식)
3. [학습 방법론](#학습-방법론)
   - [혼합 정밀도와 선택적 고정밀 레이어](#혼합-정밀도와-선택적-고정밀-레이어)
   - [Random Hadamard Transform](#random-hadamard-transform)
   - [2D 블록 스케일링](#2d-블록-스케일링)
   - [Stochastic Rounding](#stochastic-rounding)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [12B 모델 10T 토큰 사전학습](#12b-모델-10t-토큰-사전학습)
   - [NVFP4와 MXFP4 비교](#nvfp4와-mxfp4-비교)
   - [고정밀 전환으로 손실 격차 줄이기](#고정밀-전환으로-손실-격차-줄이기)
6. [Ablation 연구](#ablation-연구)
7. [NVIDIA 블로그의 8B 실험과 처리량](#nvidia-블로그의-8b-실험과-처리량)
   - [실험 구성](#실험-구성)
   - [다운스트림 정확도](#다운스트림-정확도)
   - [처리량과 메모리](#처리량과-메모리)
   - [NeMo Megatron Bridge 레시피](#nemo-megatron-bridge-레시피)
8. [한계와 주의사항](#한계와-주의사항)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

NVIDIA는 기술 보고서 "Pretraining Large Language Models with NVFP4"(arXiv 2509.25149)에서 4비트 부동소수점 포맷인 NVFP4로 대규모 LLM을 안정적으로 사전학습하는 방법론을 제시했습니다.
프런티어 모델 학습에는 수십에서 수백 yottaflops 규모의 연산이 필요하므로 사전학습 효율 개선은 차세대 LLM 개발의 핵심 과제입니다.
FP8 학습은 이미 널리 쓰이고 있으며, FP4로 더 좁은 정밀도로 넘어가면 연산 속도와 자원 활용을 추가로 개선할 수 있습니다.
하지만 이 수준의 양자화는 특히 긴 토큰 구간에서 학습되는 대형 모델의 학습 안정성, 수렴, 구현 측면에서 어려움이 있습니다.

논문이 제안하는 방법은 네 가지 요소로 구성됩니다.
블록 단위 outlier를 제한하는 Random Hadamard Transform(RHT), forward와 backward에서 일관된 표현을 유지하는 2차원(2D) 양자화, 편향 없는 그래디언트 추정을 위한 stochastic rounding, 그리고 선택적 고정밀 레이어입니다.
이 방법으로 12B 파라미터 모델을 10조(10T) 토큰으로 학습했으며, 논문은 이를 공개적으로 문서화된 4비트 정밀도 학습 중 가장 긴 학습이라고 설명합니다.
NVFP4로 학습한 모델은 FP8 베이스라인과 비슷한 학습 손실과 다운스트림 정확도를 보였고, 예를 들어 MMLU-Pro 정확도는 NVFP4가 62.58%, FP8이 62.62%였습니다.

이 포스트는 논문 본문과 함께, NVIDIA Technical Blog의 [Using NVFP4 Low-Precision Model Training for Higher Throughput Without Losing Accuracy](https://developer.nvidia.com/blog/using-nvfp4-low-precision-model-training-for-higher-throughput-without-losing-accuracy/){:target="_blank"}(2026년 2월 23일)에 실린 8B 모델 실험과 처리량 측정 결과를 함께 정리합니다.

## NVFP4 포맷

### MX 포맷과 MXFP4

좁은 부동소수점 포맷의 제한된 범위를 보완하기 위해 microscaling(MX) 포맷이 도입되었습니다.
MX 포맷은 여러 데이터 원소가 하나의 공통 스케일 팩터를 공유하는 블록 단위 표현을 사용하며, MXFP8, MXFP6, MXFP4가 이에 해당합니다.
MXFP4의 각 원소는 E2M1, 즉 부호 1비트, 지수 2비트, 가수 1비트로 표현됩니다.
따라서 표현 가능한 값은 ±0, ±0.5, ±1, ±1.5, ±2, ±3, ±4, ±6입니다.

FP32나 BF16 값은 대개 FP4 범위를 벗어나므로 양자화 시 표현 가능한 범위로 스케일링해야 합니다.
스케일 팩터는 보통 블록 내 절댓값 최대값(amax)이 FP4 최대 표현값에 매핑되도록 정해집니다.
MX 포맷은 연속된 32개 원소마다 하나의 8비트 스케일 팩터를 공유하며, 이 스케일은 2의 거듭제곱만 표현하는 unsigned E8M0(UE8M0) 포맷으로 저장됩니다.

### NVFP4의 개선점

NVFP4는 MXFP4 대비 세 가지 개선점을 갖습니다.

| 항목 | MXFP4 | NVFP4 |
|------|-------|-------|
| 원소 포맷 | E2M1 | E2M1 |
| 블록 크기 | 32 | 16 |
| 블록 스케일 포맷 | UE8M0 (2의 거듭제곱) | E4M3 (가수 비트 포함) |
| 텐서 단위 스케일 | 없음 | FP32 per-tensor 스케일 |

첫째, 블록 크기를 32에서 16으로 줄여 블록 내부의 동적 범위를 좁히고 값이 FP4 범위에 더 잘 들어가도록 합니다.
둘째, 블록 스케일을 UE8M0 대신 E4M3로 저장해 지수 범위 일부를 가수 비트와 맞바꿉니다.
셋째, 텐서 단위의 FP32 스케일을 추가로 적용해 블록 스케일의 범위를 보존합니다.

이 두 단계 스케일링은 다음과 같이 동작합니다.
먼저 per-tensor FP32 스케일이 텐서 전체 값을 블록이 표현 가능한 범위(FP4 원소와 FP8 블록 스케일이 함께 표현하는 범위)로 재매핑하고, 이어서 per-block E4M3 스케일이 블록 내 값을 FP4 표현 범위로 옮깁니다.
그 결과 NVFP4는 블록 내 최소 6.25%의 값(16개 원소 블록마다의 amax 값)을 FP8에 가까운 정밀도로 인코딩하고 나머지 값은 FP4로 저장합니다.
반면 MXFP4는 모든 값을 FP4로 저장하며, 2의 거듭제곱 스케일 반올림 때문에 최대 1 binade의 동적 범위와 네 개의 샘플(±4, ±6)을 잃을 수 있습니다.

논문 부록 B.4는 이 손실을 구체적인 예로 설명합니다.
블록 amax가 3 + δ(δ는 작은 증분)인 경우 decode 스케일은 amax/6 = 0.5 + δ/6이 되고, 포화를 피하기 위해 다음 2의 거듭제곱인 1로 올림됩니다.
스케일링 후 amax는 3 + δ가 되어 FP4에서 3으로 양자화되므로, 최악의 경우 ±4와 ±6을 표현하지 못합니다.
이때 FP4의 최소 양수 크기 0.5를 기준으로 전체 log2(6/0.5) = 3.58 binade 대신 log2(3/0.5) = 2.58 binade만 활용됩니다.
NVFP4는 더 정밀한 E4M3 블록 스케일로 블록 amax를 FP4 최대 표현값에 훨씬 가깝게 매핑해 이 한계를 극복합니다.

### Blackwell Tensor Core 지원

NVIDIA Blackwell GPU는 MXFP8, MXFP6, MXFP4, NVFP4 등 다양한 microscaling 포맷의 GEMM을 Tensor Core에서 네이티브로 지원합니다.

| 포맷 | 원소 | 스케일 | 블록 | GB200 속도 (BF16 대비) | GB300 속도 (BF16 대비) |
|------|------|--------|------|------|------|
| MXFP8 | E5M2/E4M3 | UE8M0 | 32 | 2x | 2x |
| MXFP6 | E3M2/E2M3 | UE8M0 | 32 | 2x | 2x |
| MXFP4 | E2M1 | UE8M0 | 32 | 4x | 6x |
| NVFP4 | E2M1 | E4M3 | 16 | 4x | 6x |

Tensor Core는 좁은 정밀도 입력과 16개 또는 32개 원소 블록마다의 8비트 스케일 팩터를 읽어 블록 단위 부분 내적을 계산합니다.
각 부분 내적에 해당 스케일 팩터를 곱해 양자화 시 적용한 스케일을 되돌리고, 부분 결과를 고정밀로 누적해 최종 내적을 FP32로 생성합니다.
또한 Blackwell GPU는 FP4 변환 명령에 대해 round-to-nearest-even과 stochastic rounding을 포함한 여러 반올림 모드를 네이티브로 지원합니다.
FP4 연산 처리량은 FP8 대비 GB200에서 2배, GB300에서 3배이며, FP4 피연산자를 쓰면 메모리 사용량도 FP8 대비 약 절반으로 줄어듭니다.

### 양자화 절차와 수식

논문 부록 B는 고정밀 텐서 x를 NVFP4로 변환하는 절차를 다음 수식으로 정리합니다.
여기서 6과 448은 각각 E2M1과 E4M3의 최대 표현 크기입니다.

```text
(1) 전역 encode 스케일:   s_enc = (6 × 448) / amax_x        (amax_x = 텐서 전체의 max|x|)
    전역 decode 스케일:   s_dec = 1 / s_enc                  (FP32로 저장)

(2) 로컬 decode 스케일:   s_dec,b = amax_b / 6               (amax_b = 블록 b의 max|x_i|)

(3) FP8 저장용 스케일:    s_dec,b,e4m3 = e4m3(s_dec,b × s_enc)
    로컬 encode 스케일:   s_enc,b = 1 / (fp32(s_dec,b,e4m3) × s_dec)

(4) 원소 변환:            x̂_i = q(x_i × s_enc,b)            (q는 FP4 양자화 함수)

```

수식 (5)에서 Tensor Core는 두 입력 피연산자의 로컬 decode 스케일을 읽어 블록 단위 부분 내적에 곱합니다.
GEMM 연산이 끝나면 전역 decode 스케일을 최종 출력에 같은 방식으로 적용합니다.

전역 스케일은 텐서 전체에서 동적으로 계산되므로 전역 amax 계산과 FP4 변환 전 스케일링을 위해 디바이스 메모리를 한 번 더 통과해야 합니다.
논문은 전역 스케일을 행이나 블록처럼 더 작은 단위로 적용하면 이 추가 왕복을 피할 수 있다고 언급합니다.
수식 (3)의 decode 스케일 계산에는 round-to-nearest-even이 사용됩니다.

## 학습 방법론

논문이 권장하는 NVFP4 학습 레시피는 다음 네 가지입니다.

1. 수치적으로 민감한 일부 linear 레이어를 고정밀로 유지합니다(네트워크의 15%, 고정밀 레이어 대부분은 네트워크 끝부분).
2. weight gradient GEMM(Wgrad)의 입력에 16×16 크기의 Random Hadamard Transform을 적용합니다.
3. weight에는 16×16 블록의 2D 스케일링을, activation과 gradient에는 1×16 블록의 1D 스케일링을 사용합니다.
4. gradient에는 stochastic rounding을, weight와 activation에는 round-to-nearest-even을 사용합니다.

작은 모델이나 짧은 토큰 구간에서는 이 기법이 모두 필요하지 않을 수 있지만, 12B 모델을 10T 토큰 동안 학습할 때는 각 요소가 수렴과 안정성에 필수적이었다고 논문은 보고합니다.

### 혼합 정밀도와 선택적 고정밀 레이어

연산의 대부분, 특히 linear 레이어의 GEMM은 FP4로 수행됩니다.
각 linear 레이어에는 forward의 Fprop, backward에서 activation gradient를 계산하는 Dgrad, weight gradient를 계산하는 Wgrad의 세 GEMM이 있습니다.
GEMM은 FP4 텐서를 입력으로 받아 BF16 또는 FP32로 출력합니다.

모든 linear 레이어를 FP4로 양자화하면 학습이 발산합니다.
ablation 결과 마지막 몇 개의 linear 레이어가 FP4보다 넓은 동적 범위와 가수 정밀도를 요구해 발산의 원인이 되었습니다.
따라서 논문은 마지막 레이어의 작은 비율(예: 15% 미만)을 BF16 또는 MXFP8로 유지할 것을 권장합니다.

12B 모델에서는 보수적인 구성으로 처음 2개 블록과 마지막 8개 블록(FFN 또는 Mamba-2, 각 블록은 linear 레이어 2개)을 BF16으로 유지했으며, 이는 전체 linear 레이어의 16%에 해당합니다.
다만 ablation에서는 마지막 4개 블록만 고정밀로 유지해도 수렴이 안정적이어서, 더 많은 부분을 FP4로 학습할 수 있었을 것으로 보입니다.

수치 안정성을 위해 embedding, output projection head, normalization 레이어, 비선형 함수, softmax와 query-key 및 attention score-value batched GEMM을 포함한 attention 구성 요소는 원래 정밀도(BF16 또는 FP32)를 유지합니다.
옵티마이저가 저장하는 master weight, weight gradient(마이크로배치 및 data-parallel 복제본 간 누적용), 옵티마이저 상태도 FP32로 유지합니다.
Tensor parallel reduction은 BF16으로 수행합니다.

### Random Hadamard Transform

Microscaling은 텐서 값 표현에 필요한 동적 범위를 줄이지만, outlier는 여전히 FP4 포맷에 큰 영향을 미쳐 정확도를 떨어뜨릴 수 있습니다.
Random Hadamard Transform은 outlier를 근사적인 가우시안 분포로 재분배해 좁은 포맷에서 표현하기 쉽게 만듭니다.

Hadamard 변환은 직교 회전이므로 GEMM의 양쪽 입력에 적용하면 내적 과정에서 서로 상쇄됩니다.
논문 부록 C에 따르면 Hadamard 행렬은 원소가 ±1로 제한된 정규화 행렬이며, 직교성 덕분에 행렬 곱의 두 피연산자에 적용해도 각 피연산자의 변환이 내적 안에서 상쇄됩니다.
랜덤성은 대각 원소가 {-1, 1}에서 무작위로 선택된 대각 행렬 S를 곱해 행 단위 부호를 뒤집는 방식으로 도입됩니다.
변환은 d×d Hadamard 행렬을 텐서의 각 d×d 타일에 곱하는 타일 방식으로 수행되며, d가 텐서 차원보다 훨씬 작으면 비용이 작고 batched 행렬 곱으로 구현해 다른 레이어와 fusion할 수 있습니다.

| 설계 항목 | 선택 | 근거 |
|------|------|------|
| 적용 GEMM | Wgrad 입력에만 적용 | 12B 모델에서 Wgrad 변환이 손실 개선, 소규모에서 Fprop·Dgrad는 이득 없음 |
| 행렬 크기 | d = 16 | d = 4보다 수렴이 좋고 d = 128과 유사 |
| 랜덤 부호 벡터 | 학습 전체에서 모든 linear 레이어가 공유하는 단일 벡터 | 벡터 수를 늘려도 측정 가능한 효과 없음 |

행렬 크기에는 정확도와 성능의 트레이드오프가 있습니다.
큰 행렬은 outlier를 더 많은 값으로 분산시키지만 연산과 메모리 비용이 증가하고, 너무 작은 행렬은 가우시안 분포를 재현할 가능성이 낮습니다.
논문은 이를 대형 모델일수록 outlier가 많기 때문으로 해석합니다.
랜덤 부호 벡터는 Hadamard 기저에 정렬된 "구조적" outlier가 변환 후에도 남을 가능성을 줄이며, 소규모에서는 효과가 없지만 긴 토큰 구간의 대형 모델에서 이득이 있었습니다.

### 2D 블록 스케일링

학습 중 변환과 스케일링은 내적 차원을 따라 적용되므로, forward(행 방향)와 backward(열 방향)에서 텐서가 다르게 변환·스케일링됩니다.
backward 패스가 텐서를 전치하면서 내적 차원이 바뀌기 때문입니다.
그 결과 같은 텐서가 두 가지 양자화 표현을 갖게 되고, backpropagation이 forward와 다른 함수를 미분하게 되어 chain rule이 깨집니다.
논문은 weight에서의 chain rule 위반이 정확도 저하에 기여한다고 가정합니다.

이를 완화하기 위해 weight는 16×16 블록(입력 채널 16 × 출력 채널 16) 단위로 그룹화해 스케일링합니다.
2D 블록 스케일은 Tensor Core에 전달될 때 각 1×16 블록에 복제되며, FP32 per-tensor 스케일도 계속 사용합니다.
activation과 gradient는 더 세밀한 스케일링이 양자화 정확도를 높이므로 표준 NVFP4 스케일링(1×16 블록)을 사용합니다.
activation도 chain rule 문제가 있지만 학습은 weight보다 activation의 불일치에 덜 민감하고, weight는 FP4 값에 적응할 수 있어 스케일 단위에도 더 관대합니다.

같은 이유로 RHT도 weight 텐서에는 적용하지 않습니다.
그러면 weight 관련 GEMM의 activation과 gradient 변환을 weight 변환으로 상쇄할 수 없어 Fprop과 Dgrad는 변환의 이점을 얻지 못하므로, RHT는 Wgrad 텐서로 한정됩니다.

### Stochastic Rounding

FP4로 양자화할 때 round-to-nearest-even 같은 결정적 반올림은 특정 방향으로 반올림되기 쉬운 가수 분포, 0으로의 underflow, 최대 표현값으로의 포화 때문에 체계적 편향을 만들 수 있습니다.
이 편향은 gradient 텐서에서 특히 두드러져 수렴에 영향을 줍니다.
Stochastic rounding은 값을 가장 가까운 두 표현값 중 하나로 거리에 반비례하는 확률로 반올림해 편향을 줄입니다.

12B 모델에서는 gradient 텐서에 stochastic rounding을 적용하는 것이 수렴에 필수적이었습니다.
backward의 다른 텐서에는 효과가 없었고, forward 텐서에 적용하면 nearest rounding 대비 양자화 오차가 커져 오히려 해로웠습니다.

## 실험 셋업

논문은 세 가지 모델을 사용합니다.
12B 하이브리드 Mamba-Transformer는 NVFP4 학습 방법 검증용, 8B 하이브리드 모델은 NVFP4와 MXFP4 비교용, 1.2B Transformer는 ablation용입니다.

| 항목 | 12B 하이브리드 | 8B 하이브리드 | 1.2B Transformer |
|------|------|------|------|
| 블록 수 | 62 | 52 | 20 |
| 블록 구성 | Self-Attention 6, FFN 28, Mamba-2 28 | Self-Attention 4, FFN 24, Mamba-2 24 | Self-Attention + FFN |
| 모델 차원 | 5120 | 4096 | 2048 |
| FFN 차원 | 20480 | 21504 | 6144 |
| Q heads / KV heads | 40 / 8 | 32 / 4 | 16 / 8 |
| 학습 토큰 | 10T | 1T | 1T |
| 시퀀스 길이 | 8192 | 8192 | 8192 |
| 배치 크기 | 736 | 768 | 768 |
| 학습률 (WSD) | 4.5e-4 → 4.5e-6 (마지막 20% 감쇠) | 8.0e-4 → 8.0e-6 (마지막 15% 감쇠) | 1.2e-3 → 1.2e-5 (마지막 15% 감쇠) |
| 베이스라인 | FP8 | BF16 | BF16 |

12B 모델은 Nemotron-H 계열의 Nemotron-Nano-12B-v2-Base와 같은 아키텍처를 사용합니다.
Mamba-2 블록은 그룹 8, state 차원 128, head 차원 64, 확장 계수 2, 합성곱 윈도우 4를 사용하며, FFN에는 Squared ReLU, 정규화에는 RMSNorm을 씁니다.
position embedding, dropout, linear 레이어 bias는 없고 embedding과 output 레이어 weight는 분리되어 있습니다.
Adam 파라미터는 β1 = 0.9, β2 = 0.95, weight decay는 0.1입니다.

데이터는 웹 크롤, wikipedia, 수학, 코드, 학술, crawl++, 다국어, 합성 SFT 스타일 데이터로 구성된 10T 토큰 코퍼스입니다.
12B 학습은 3단계 data-blending을 사용해 첫 단계가 학습의 70%(다양성 중심), 두 번째와 세 번째 단계가 각각 마지막 20%와 10%(고품질 데이터 중심)를 차지합니다.
8B와 1.2B 모델은 2단계 blending을 사용하며, 8B는 처음 60%와 마지막 40%로 나뉩니다.

FP8 베이스라인은 첫 블록과 마지막 두 블록을 제외한 모든 linear 레이어를 E4M3로 계산합니다.
스케일 팩터는 weight에 128×128 블록, activation과 gradient에 1×128 블록 단위로 온라인 계산되어 FP32로 저장됩니다.

## 주요 결과

### 12B 모델 10T 토큰 사전학습

NVFP4의 검증 손실은 학습 전 구간에서 FP8을 가깝게 따라갔습니다.
stable 구간에서 NVFP4의 상대 손실 오차는 1% 미만을 유지했고, 학습 말미 학습률이 감쇠할 때 1.5%를 약간 넘는 수준으로 벌어졌습니다.
8T 토큰 지점의 기울기 변화는 학습률 감쇠 때문이며, 9T 토큰 지점의 작은 손실 상승은 데이터 블렌드 변경 때문입니다.

손실에 작은 격차가 있었지만 다운스트림 정확도는 거의 영향을 받지 않았습니다.
지식 집약 추론, 수학, 코딩, 상식 추론 전반에서 NVFP4는 학습 내내 FP8과 비슷한 정확도를 보였습니다.
아래 표는 최종 12B 모델의 평가 결과이며, 평가는 BF16으로 수행되었습니다.

| 분야 | 태스크 | FP8 | NVFP4 |
|------|------|------|------|
| General | 평균 | 68.99 | 69.82 |
| General | MMLU | 77.36 | 76.57 |
| General | MMLU-Pro 5-shot | 62.62 | 62.58 |
| General | AGIEval English CoT | 67.01 | 70.31 |
| Math | 평균 | 86.20 | 86.88 |
| Math | GSM8k CoT | 89.08 | 92.27 |
| Math | MATH | 83.32 | 81.48 |
| Multilingual | 평균 | 77.93 | 80.24 |
| Multilingual | Global MMLU | 74.00 | 74.94 |
| Multilingual | MGSM | 81.87 | 85.53 |
| Code | 평균 | 59.52 | 56.67 |
| Code | HumanEval+ | 59.93 | 57.43 |
| Code | MBPP+ | 59.11 | 55.91 |
| Commonsense Understanding | 평균 | 77.29 | 76.75 |
| Commonsense Understanding | ARC Challenge | 91.81 | 91.81 |
| Commonsense Understanding | HellaSwag | 83.83 | 83.09 |
| Commonsense Understanding | OpenBookQA | 47.60 | 47.40 |
| Commonsense Understanding | PIQA | 82.64 | 82.70 |
| Commonsense Understanding | Winogrande | 80.58 | 78.77 |

대부분의 벤치마크에서 NVFP4는 FP8과 비슷한 정확도를 보였고, 코딩 태스크에서만 약간 뒤처졌습니다.
논문은 이 차이가 평가 노이즈 때문일 수 있다고 추정합니다.
MBPP+ 정확도가 최종 체크포인트에서 하락했으며, 다른 체크포인트를 선택하면 더 나은 결과가 나올 수 있다는 설명입니다.

### NVFP4와 MXFP4 비교

두 FP4 포맷을 비교하기 위해 8B 하이브리드 Mamba-Transformer를 1T 토큰으로 학습했고, 참조 모델은 BF16으로 학습했습니다.
두 포맷 모두 4장의 학습 방법론을 따랐으며, MXFP4는 블록 크기에 맞춰 Wgrad 입력에 d = 32 크기의 RHT를 적용했습니다.
두 설정 모두 마지막 8개 블록(FFN 또는 Mamba-2)을 BF16으로 유지했으며, 이는 모델의 약 15%입니다.

| 항목 | NVFP4 | MXFP4 |
|------|------|------|
| BF16 대비 학습 손실 상대 오차 (1T 토큰) | 약 1.5% | 약 2.5% |
| NVFP4 손실에 도달하는 데 필요한 토큰 | 1T | 1.36T (36% 추가) |

NVFP4는 MXFP4보다 더 낮은 손실로 수렴했습니다.
MXFP4의 학습 토큰을 1T에서 1.5T 사이로 늘려 격차를 좁히는 실험에서, MXFP4는 36% 더 많은 토큰(1T 대신 1.36T)을 사용했을 때 NVFP4의 손실과 같아졌습니다.
이는 MXFP4의 학습 시간이 상당히 늘어난다는 의미이며, 논문은 다양한 파라미터 규모와 토큰 구간에서 두 포맷의 scaling law를 평가하는 후속 연구가 필요하다고 덧붙입니다.

### 고정밀 전환으로 손실 격차 줄이기

손실 최소화가 중요한 경우, 학습 후반에 더 높은 정밀도로 전환하면 격차를 줄일 수 있습니다(부록 D).
12B 모델에서 8.2T 토큰 이후(학습의 18%) 정밀도를 전환하면 손실이 FP8 베이스라인과 같아졌고, 10T 토큰 이후(학습의 1% 미만) 전환하면 약간 못 미치는 수준이었습니다.
늦게 전환하면 완전히 회복되지 않는데, 논문은 학습률이 너무 낮아 weight 업데이트가 충분하지 않기 때문으로 추정합니다.

손실 격차의 대부분은 forward 패스의 텐서 양자화에서 발생했습니다.
8.2T 토큰부터 forward만 고정밀로 전환해도 상대 오차가 1.5%에서 0.5%로 줄었습니다.
backward 전환으로 손실이 회복된다고 보고한 선행 연구(Chmiel et al., 2025)와 달리 이 모델에서는 그런 개선이 관찰되지 않았습니다.
forward만 전환하면 전체 연산의 약 6%(마지막 18% 구간의 약 3분의 1)만 고정밀로 수행되므로 오버헤드가 작습니다.
논문은 완전한 손실 회복을 위해서는 학습률 감쇠 시작 직전에, 런타임 영향을 최소화하면서 손실을 개선하려면 학습 맨 끝에 전환할 것을 권장합니다.

## Ablation 연구

12B 모델 ablation은 처음 2개와 마지막 8개 블록을 제외하고 NVFP4로 3.43T 토큰까지 학습한 모델에서 출발해, 구성 요소를 하나씩 제거하며 진행했습니다.
stochastic rounding, RHT, 2D weight 스케일링 중 하나를 제거하거나 BF16 블록 수를 줄이면 모두 수렴이 나빠졌습니다.
마지막 4개 블록만 BF16으로 유지한 경우에도 수렴은 안정적이었습니다.

1.2B 모델(1T 토큰)과 12B 모델에서 수행한 부록 E의 ablation 결과는 다음과 같습니다.

| 실험 | 대상 모델 | 결과 |
|------|------|------|
| 기법 결합 (E.1) | 1.2B | 모든 레이어 NVFP4, 1×16 스케일, RTN만 쓰는 base 방법은 초기에 발산, 기법을 결합할수록 손실 격차 감소 |
| 레이어 민감도 (E.2) | 1.2B | 마지막 4개 블록 BF16이면 수렴, 처음 4개 블록 BF16은 불안정, 처음 2개와 마지막 2개 BF16은 안정 |
| SR 적용 텐서 (E.3) | 1.2B | gradient에 SR 적용 시 안정, activation이나 weight에 적용 시 발산 |
| SR 적용 GEMM (E.3) | 12B | Dgrad와 Wgrad로 들어가는 gradient 모두에 SR 필요 |
| RHT 적용 GEMM (E.4.1) | 1.2B | Wgrad 입력 변환은 손실 개선, Fprop·Dgrad 변환은 품질 저하 |
| Hadamard 크기 (E.4.2) | 1.2B | 2×2, 4×4, 16×16, 128×128 간 차이 거의 없음 |
| Hadamard 크기 (E.4.2) | 12B | 3.4T 토큰 이후 4×4로 바꾸면 손실 증가, 128×128은 소폭 개선 |
| 랜덤화 (E.4.3) | 12B | 랜덤 부호 벡터가 없으면 품질 저하, 변환마다 새 seed를 써도 개선 없음 |
| 랜덤화 (E.4.3) | 1.2B | 랜덤화 전략 간 차이 없음 |
| 표현 일관성 (E.5) | 1.2B | 불일치 weight는 손실 악화, 16×16 2D 스케일이 다른 차원 1×16보다 개선, activation은 학습률 감쇠 구간에서만 영향 |

레이어 민감도 분석에서는 마지막 레이어들이 weight gradient(FP4 입력으로 계산한 Wgrad 출력)에서 더 큰 양자화 오차를 보였습니다.
논문은 양자화 오차 지표가 어떤 linear 레이어를 고정밀로 남길지 결정하는 기준이 될 수 있다고 제안합니다.
RHT가 Fprop과 Dgrad에서 품질을 떨어뜨린 이유로는 추가 양자화 오차가 outlier 제거 이점을 상쇄한다는 가설을 제시합니다.

표현 일관성 실험에서 같은 차원 1×16 스케일은 forward와 backward의 표현이 같지만, Tensor Core가 내적 차원을 따라 스케일을 요구하므로 실제로는 구현할 수 없습니다.
실제 구현 가능한 선택지는 다른 차원 1×16 스케일과 16×16 2D 스케일이며, 2D 스케일은 블록 단위가 더 크지만 손실이 더 좋았습니다.
논문은 불일치 weight의 오차가 activation gradient로 나타나 backpropagation 동안 레이어를 따라 전파되기 때문에 weight가 activation보다 더 큰 영향을 받는다고 가정합니다.
또한 1.2B와 12B의 결과 차이는 소규모에서 얻은 결론이 대형 모델과 긴 토큰 구간에서 항상 유지되지 않음을 보여줍니다.

## NVIDIA 블로그의 8B 실험과 처리량

### 실험 구성

NVIDIA Technical Blog 포스트는 BF16 베이스라인과 세 가지 저정밀 포맷을 multi-hundred-billion-token 규모 사전학습과 다운스트림 벤치마크로 비교했습니다.

| 포맷 | 설명 |
|------|------|
| FP8-CS | per-tensor current scaling을 사용하는 FP8 |
| MXFP8 | 32개 원소 블록 스케일링을 사용하는 FP8, Blackwell에 최적화 |
| NVFP4 | 두 단계 계층적 스케일링을 사용하는 4비트 포맷 |

모델은 Llama 3 8B와, Llama 3 8B와 유사한 dense grouped query attention을 쓰는 NVIDIA 내부 연구 모델 Research-8B입니다.
학습은 NVIDIA B200 GPU에서 NeMo Megatron Bridge로 1T 토큰 동안 진행했으며, 데이터셋은 Lingua DCLM과 NVIDIA 내부 연구 데이터셋입니다.
FP8-CS와 MXFP8의 손실 곡선은 BF16을 가깝게 따라갔고, NVFP4는 손실이 약간 높지만 비슷하게 안정적인 하락 추세를 보였습니다.

### 다운스트림 정확도

다운스트림 평가는 모두 BF16으로 수행되었습니다(단위 %).

| 모델 | 데이터셋 | 정밀도 | MMLU | HellaSwag | WinoGrande | ARC-C |
|------|------|------|------|------|------|------|
| Llama 3 8B | DCLM | BF16 | 45.98 | 76.44 | 70.17 | 51.28 |
| Llama 3 8B | DCLM | FP8-CS | 46 | 75.25 | 70.24 | 49.91 |
| Llama 3 8B | DCLM | MXFP8 | 46.56 | 75.46 | 71.27 | 51.11 |
| Llama 3 8B | DCLM | NVFP4 | 45.64 | 75.59 | 69.38 | 51.28 |
| Llama 3 8B | Internal | BF16 | 52.73 | 75.71 | 67.88 | 51.37 |
| Llama 3 8B | Internal | FP8-CS | 52.46 | 75.65 | 70.17 | 54.52 |
| Llama 3 8B | Internal | MXFP8 | 53.7 | 75.54 | 69.69 | 51.62 |
| Llama 3 8B | Internal | NVFP4 | 52.83 | 75.04 | 71.98 | 53.58 |
| Research-8B | Internal | BF16 | 53 | 76.98 | 70.4 | 55.89 |
| Research-8B | Internal | FP8-CS | 52.62 | 75.81 | 70.8 | 54.44 |
| Research-8B | Internal | MXFP8 | 52.38 | 76.55 | 69.77 | 53.58 |
| Research-8B | Internal | NVFP4 | 52.21 | 76.19 | 70.32 | 54.95 |

모든 저정밀 포맷이 모델과 데이터셋 전반에서 BF16과 비슷한 정확도를 보였으며, 개별 점수는 양방향으로 몇 포인트씩 차이가 났습니다.
블로그는 MXFP8이 더 세밀한 스케일링 덕분에 표준 FP8보다 약간 나은 성능을 보였다고 정리합니다.
NVFP4는 AdamW ε=1e-8, 학습률 6e-4에서 6e-6으로 감쇠, global batch size 768의 레시피에서 동작했습니다.
ablation에서 전부 NVFP4로 학습한 모델은 발산했고, 마지막 4개 transformer 레이어를 BF16으로 유지하는 것으로 안정적인 학습이 가능했습니다.

### 처리량과 메모리

처리량은 NVIDIA GB200 NVL72에서 Llama 3 8B를 GBS=128, 시퀀스 길이 8192로 학습하며 측정했습니다.
표기 FxLy는 처음 x개와 마지막 y개 transformer 레이어를 BF16으로 유지한다는 의미입니다.

| 정밀도 | 마이크로배치 | TFLOP/s/GPU | BF16 대비 속도 |
|------|------|------|------|
| BF16 | 2 | 1165 | 1.00x |
| FP8-CS (F1L1) | 2 | 1547 | 1.33x |
| MXFP8 | 2 | 1540 | 1.32x |
| NVFP4 (F0L4) | 4 | 1850 | 1.59x |

NVFP4는 BF16 대비 최대 1.59배의 처리량을 보였습니다.
또한 NVFP4는 메모리 사용량을 줄여 사전학습 중 마이크로배치 크기를 2에서 4로 두 배 늘릴 수 있었습니다.
블로그의 메모리 표에 따르면 스케일 팩터는 FP8 텐서 스케일링이 weight 텐서당 하나, MXFP8이 32개 원소당 하나, NVFP4가 1×16 블록마다 복제되는 16×16 2D 블록 스케일입니다.

### NeMo Megatron Bridge 레시피

[NeMo Megatron Bridge](https://github.com/NVIDIA-NeMo/Megatron-Bridge){:target="_blank"}는 저정밀 학습 레시피를 제공하며, 포맷 전환은 `mixed_precision_recipe` 설정 하나만 바꾸면 됩니다.
설정값에는 `bf16_with_mxfp8_mixed`, `bf16_with_fp8_current_scaling_mixed`, `bf16_with_nvfp4_mixed` 등이 있습니다.
블로그는 Llama 3 8B 예제와 함께 [저정밀 학습 튜토리얼 노트북](https://github.com/NVIDIA-NeMo/Megatron-Bridge/blob/main/tutorials/training/reduced_precision_training.ipynb){:target="_blank"}을 안내합니다.
논문 역시 Blackwell에서의 NVFP4 학습이 Transformer Engine 업데이트로 완전히 지원된다고 밝힙니다.

## 한계와 주의사항

논문은 대규모 FP4 학습의 가능성을 입증하는 데 초점을 두며, 런타임 효율이나 시스템 수준 최적화는 주요 관심사가 아니라고 명시합니다.
처리량 수치는 논문이 아니라 NVIDIA 블로그의 Llama 3 8B 측정 결과입니다.

여전히 linear 레이어 일부(12B 모델 기준 16%)를 BF16으로 유지해야 하며, 모든 linear 레이어를 FP4로 양자화하면 학습이 발산합니다.
embedding, output head, normalization, attention 구성 요소, 통신 경로도 고정밀로 남아 있습니다.
12B 모델에서 NVFP4의 손실은 학습률 감쇠 구간에서 FP8 대비 1.5%를 약간 넘는 상대 오차를 보였고, 코딩 벤치마크에서는 FP8보다 낮은 점수를 기록했습니다.

ablation 결과의 일부는 규모에 따라 달라졌습니다.
Hadamard 행렬 크기와 랜덤화는 1.2B 모델에서는 차이가 없었지만 12B 모델에서는 영향을 주었으므로, 소규모 실험의 결론을 대형 모델에 그대로 적용하기 어렵습니다.
MXFP4 비교도 8B 모델과 1T 토큰이라는 단일 설정에서 수행되었으며, 논문은 다른 파라미터 규모와 토큰 구간에서의 scaling law 평가를 후속 과제로 남깁니다.

## 결론

이 논문은 NVFP4로 대규모 사전학습을 안정적이고 정확하게 수행할 수 있음을 보였습니다.
2D weight 스케일링, Wgrad에 대한 Random Hadamard Transform, gradient의 stochastic rounding, 선택적 고정밀 레이어를 결합해 12B 하이브리드 Mamba-Transformer를 10T 토큰으로 학습했고, 손실과 다운스트림 정확도가 FP8 베이스라인을 가깝게 따라갔습니다.
논문은 이를 multi-trillion-token 규모에서 4비트 사전학습이 지속 가능함을 보인 첫 공개 사례로 제시합니다.

NVFP4는 MXFP4보다 적은 토큰으로 비슷한 손실에 도달했으며, MXFP4는 같은 손실을 위해 36% 더 많은 토큰이 필요했습니다.
NVIDIA 블로그의 8B 실험에서도 NVFP4는 BF16과 비슷한 다운스트림 정확도를 유지하면서 GB200 NVL72에서 1.59배의 처리량을 보였습니다.

향후 과제로 논문은 수렴 저하 없이 모든 linear 레이어를 양자화하도록 방법론을 개선하고, 남은 고정밀 레이어를 줄이며, NVFP4를 attention과 통신 경로로 확장하는 것을 제시합니다.
또한 post-training 시나리오, 더 큰 모델, 더 긴 토큰 구간, mixture-of-experts 같은 아키텍처로의 적용도 계획하고 있습니다.

## Reference

- [Pretraining Large Language Models with NVFP4 (arXiv 2509.25149)](https://arxiv.org/abs/2509.25149)
- [Using NVFP4 Low-Precision Model Training for Higher Throughput Without Losing Accuracy - NVIDIA Technical Blog](https://developer.nvidia.com/blog/using-nvfp4-low-precision-model-training-for-higher-throughput-without-losing-accuracy/)
- [NVIDIA NeMo Megatron Bridge](https://github.com/NVIDIA-NeMo/Megatron-Bridge)
- [Megatron Bridge Reduced Precision Training Tutorial](https://github.com/NVIDIA-NeMo/Megatron-Bridge/blob/main/tutorials/training/reduced_precision_training.ipynb)
