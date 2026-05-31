---
layout: post
title: "TurboQuant 완전 정리 - 이론 최적에 근접한 KV 캐시·벡터 검색 양자화와 vLLM 실측"
author: 'Juho'
date: 2026-05-31 00:00:00 +0900
categories: [LLM]
tags: [LLM, Benchmark, GPU, vLLM, Caching]
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
  .highlight pre,
  .highlight code {
    white-space: pre-wrap;
    word-break: break-word;
    overflow-wrap: break-word;
  }
</style>

## 목차
1. [개요](#개요)
2. [배경](#배경)
   - [KV 캐시가 메모리 병목인 이유](#kv-캐시가-메모리-병목인-이유)
   - [데이터 비의존 양자화의 어려움](#데이터-비의존-양자화의-어려움)
   - [기존 KV 캐시 양자화 기법](#기존-kv-캐시-양자화-기법)
3. [개발 주체와 발표 타임라인](#개발-주체와-발표-타임라인)
4. [알고리즘 원리](#알고리즘-원리)
   - [랜덤 회전과 Beta 분포](#랜덤-회전과-beta-분포)
   - [Lloyd-Max 스칼라 양자화 (Q_mse)](#lloyd-max-스칼라-양자화-q_mse)
   - [직교성을 이용한 내적 보존](#직교성을-이용한-내적-보존)
   - [편향 없는 내적 - QJL 2단계 (Q_prod)](#편향-없는-내적---qjl-2단계-q_prod)
   - [이론적 보장](#이론적-보장)
5. [구현과 통합](#구현과-통합)
   - [vLLM 업스트림](#vllm-업스트림)
   - [llama.cpp 및 커뮤니티 구현](#llamacpp-및-커뮤니티-구현)
6. [벤치마크 결과](#벤치마크-결과)
   - [원논문 KV 캐시 결과](#원논문-kv-캐시-결과)
   - [벡터 검색 결과](#벡터-검색-결과)
   - [vLLM 팀의 독립 평가](#vllm-팀의-독립-평가)
7. [다른 양자화 기법과의 비교](#다른-양자화-기법과의-비교)
8. [한계와 비판](#한계와-비판)
9. [결론과 권장 사용 시나리오](#결론과-권장-사용-시나리오)
10. [Reference](#reference)

## 개요

TurboQuant는 Google Research가 주도하고 NYU, Google DeepMind가 공동 저자로 참여한 LLM KV 캐시·벡터 검색용 온라인 벡터 양자화(online vector quantization) 알고리즘이다.
캘리브레이션 데이터와 파인튜닝 없이 매 토큰마다 들어오는 KV 벡터를 즉시 1~4비트로 압축하며, 정보이론적 distortion 하한의 약 2.7배 이내라는 이론적 보장을 제공한다.
2026년 3월 Google Research 공식 블로그가 "6배 메모리 압축, H100에서 최대 8배 attention 가속"이라는 메시지를 내보내면서 업계의 주목을 받았다.
[arXiv:2504.19874](https://arxiv.org/abs/2504.19874){:target="_blank"} 논문은 ICLR 2026에 채택되었고, vLLM 업스트림에 PR #38479로 머지되어 실제 production 추론 엔진에서도 사용할 수 있게 되었다.
한편 vLLM 팀의 후속 종합 평가에서는 "FP8가 여전히 기본 권장값"이라는 보수적 결론이 나왔는데, 이는 학계 발표와 실제 production 도입 사이의 간극을 보여주는 흥미로운 사례다.
이 글에서는 TurboQuant이 무엇이고, 어떤 원리로 동작하며, 다른 기법과 어떻게 다른지, 그리고 vLLM 실측에서 어떤 결론이 나왔는지를 하나로 정리한다.

## 배경

### KV 캐시가 메모리 병목인 이유

Transformer 기반 LLM은 디코딩 단계에서 이전 토큰의 Key, Value 텐서를 캐시(KV 캐시)해두고 매 토큰마다 attention에서 재사용한다.
key와 value는 각각 시퀀스 길이 × head 차원 형태이고, 여기에 attention 레이어 수가 곱해진다.
컨텍스트가 길어질수록 KV 캐시 크기는 토큰 수에 선형으로 증가하여, 70B급 모델에서 100K 토큰 컨텍스트는 수십 GB의 GPU 메모리를 점유한다.
실제로 Gemma 4 26B는 A100 80GB 환경에서 KV 캐시만 52 GB를 소비하고, GLM-5.1 754B MoE는 BF16 기준 1,508 GB의 KV 메모리를 요구한다.
KV 캐시 양자화(quantization)는 이러한 메모리 압박을 풀어 더 긴 컨텍스트와 더 큰 배치를 가능하게 하는 핵심 기법이다.

### 데이터 비의존 양자화의 어려움

양자화는 값의 범위를 여러 구간으로 나누는 작업이다.
값들이 균일(uniform)하거나 Gaussian, Beta처럼 잘 정의된 분포를 따르면, 그 분포에 맞춰 최적 간격을 미리 계산할 수 있다(Lloyd-Max 양자화).
문제는 attention의 key가 학습된 weight를 통과한 결과라는 점이다.
특정 채널에서만 값이 크게 튀는 outlier가 존재하고, 분포가 균일하지도 잘 정의되지도 않으며, outlier가 어느 차원에서 등장할지도 예측하기 어렵다.
또한 KV 캐시처럼 값이 실시간(streaming)으로 도착하는 환경에서는, 데이터를 미리 모아 코드북을 학습하는 data-aware 방식(k-means 등)을 쓸 수 없다.
어떤 입력이 와도 성립하는 data-oblivious 코드북이 필요한데, 기존 data-free 방식은 distortion이 이론 최적치에 한참 못 미쳤다.
게다가 기존 벡터 양자화 기법은 양자화 상수(scale 등)를 full precision으로 저장해야 해서, 숫자당 1~2비트의 추가 오버헤드가 발생하고 압축 이득이 줄어드는 한계가 있었다.

### 기존 KV 캐시 양자화 기법

2026년 5월 기준 주요 KV 캐시 양자화 트렌드는 다음과 같다.

| 기법 | 비트 | 특징 |
|------|------|------|
| FP8 KV cache | 8 | vLLM production 기본값, 2배 압축, 거의 무손실 |
| KIVI | 2~4 | tuning-free asymmetric, K per-channel V per-token |
| KVQuant | 2~4 | per-channel + outlier 처리, non-uniform |
| AWQ | 4 | activation-aware, 주로 weight 대상 |
| GPTQ | 3~4 | Hessian 기반, weight-only |
| TurboQuant | 1~4 | data-oblivious, random rotation + Lloyd-Max |

여기서 weight 양자화(AWQ, GPTQ)는 모델 파라미터를 압축하고, KV 양자화(FP8, KIVI, KVQuant, TurboQuant)는 attention의 K/V 텐서를 압축한다.
세 계층(weight / activation / KV cache)은 직교적이므로 AWQ + FP8 + TurboQuant 조합도 원칙적으로 가능하다.

## 개발 주체와 발표 타임라인

논문 제목은 "TurboQuant: Online Vector Quantization with Near-optimal Distortion Rate"이며, 저자는 Amir Zandieh(Google Research), Majid Daliri(NYU), Majid Hadian(Google DeepMind), Vahab Mirrokni(Google Research)이다.
[arXiv:2504.19874](https://arxiv.org/abs/2504.19874){:target="_blank"}로 2025년 4월 28일 v1이 공개되었고, [ICLR 2026 OpenReview](https://openreview.net/forum?id=tO3ASKZlok){:target="_blank"}에서 채택 결과를 확인할 수 있다.
TurboQuant은 두 편의 동반 논문 위에 서 있다.
하나는 AAAI 2025의 QJL(Quantized Johnson-Lindenstrauss), 다른 하나는 AISTATS 2026에 채택된 PolarQuant이다.
TurboQuant은 이 두 알고리즘을 결합한 2단계 구조로 설계되었다.

주요 발표 시점은 다음과 같다.

| 일자 | 이벤트 |
|------|--------|
| 2025-04-28 | arXiv v1 게재 |
| 2026-03-24 | Google Research 공식 블로그 발표 |
| 2026-03-26 | vLLM Feature Request Issue #38171 오픈 |
| 2026-04-15 | vLLM PR #38479 머지 (GQA/MHA 모델용) |
| 2026-05-11 | vLLM 팀 종합 평가 블로그 게재 |
| 2026 | ICLR 정식 채택 |

[Google Research 공식 블로그](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/){:target="_blank"}는 "Redefining AI efficiency with extreme compression"이라는 메시지로 6배 메모리 압축과 H100 8배 가속을 셀링 포인트로 강조했다.

## 알고리즘 원리

TurboQuant의 핵심 아이디어는 두 가지다.
첫째, 입력 벡터에 random orthogonal rotation을 가하면 좌표 분포가 표준화되어 좌표마다 동일한 1차원 스칼라 양자화기를 쓸 수 있다.
둘째, MSE에 최적화된 양자화기는 내적(inner product) 추정에서 편향(bias)을 가지므로, 잔차(residual)에 1-bit QJL projection을 추가해 unbiased 내적 추정기를 만든다.
이 두 단계가 각각 Q_mse와 Q_prod 알고리즘을 이룬다.

### 랜덤 회전과 Beta 분포

TurboQuant는 입력 벡터를 단위 초구(hypersphere) 위로 정규화한 뒤, 랜덤 회전 행렬 Π를 곱한다.
공식 블로그는 이 과정을 PolarQuant로 설명하는데, 벡터를 반지름(크기)과 각도(방향)로 나누어 표현하면 각도 성분의 패턴이 매우 집중적이고 예측 가능해져 별도의 정규화 단계를 줄일 수 있다.
랜덤 회전을 거친 단위 벡터의 각 좌표는 다음 확률밀도함수를 따른다.

```text
f_X(x) = Γ(d/2) / ( sqrt(π) · Γ((d-1)/2) ) · (1 - x^2)^((d-3)/2)
```

이는 좌표 제곱이 Beta(1/2, (d-1)/2)를 따른다는 것과 같다.
차원 d가 클수록 이 분포는 N(0, 1/d) Gaussian으로 수렴하며(measure concentration), 서로 다른 좌표들이 거의 독립(near-independence)이 된다.
이 독립성 덕분에 좌표별 스칼라 양자화만으로도 전역적으로 near-optimal한 성능이 나오고, 입력 분포를 사전 관측할 필요가 없어 data-oblivious(online)하다.

실용적으로는 일반적인 랜덤 회전 대신 Randomized Hadamard Transform(RHT)을 사용한다.
RHT는 랜덤 부호 대각 행렬과 Hadamard 행렬의 곱으로, 특정 차원에 몰려 있던 outlier를 전체 차원에 고르게 분산시키며 O(d log d)로 빠르게 계산된다.

### Lloyd-Max 스칼라 양자화 (Q_mse)

좌표 분포가 Beta로 고정되므로, 그 분포에 최적인 Lloyd-Max 코드북을 데이터 없이 미리 계산할 수 있다.
이는 다음 연속형 k-means 문제를 푸는 것에 해당한다.

```text
C(f_X, b) = min over centroids c_1..c_{2^b} of
            Σ_i ∫ |x - c_i|^2 · f_X(x) dx   (각 셀 구간에서 적분)
```

Algorithm 1(Q_mse)의 절차는 다음과 같다.

1. Random rotation matrix 생성 - 가우시안 랜덤 행렬의 QR 분해(또는 RHT)로 orthogonal matrix Π를 얻는다.
2. Rotation 적용 - 입력 KV 벡터 x에 대해 y ← Π·x를 계산한다.
3. 좌표별 양자화 - 각 좌표 y_i에 미리 계산된 Lloyd-Max(continuous k-means) 최적 스칼라 코드북을 적용하여 인덱스를 저장한다.
4. Dequantize - 코드북 lookup으로 복원하고 Π의 transpose로 역회전한다.

b=1, 2, 3, 4 bit에 대한 centroid를 사전 계산해 저장해두고 그대로 사용하므로, calibration 데이터가 전혀 필요 없다.

### 직교성을 이용한 내적 보존

KV 캐시에서 중요한 것은 복원된 key 자체가 아니라 query와의 내적(attention score)이다.
TurboQuant가 양자화하는 대상은 원본 key가 아니라 정규화·회전된 key인데, RHT가 직교(orthogonal) 행렬이라는 성질 덕분에 내적이 그대로 보존된다.

```text
M^T = M^(-1),    <M x, M y> = <x, y>
```

즉 역행렬을 따로 계산할 필요 없이, query에 RHT의 transpose만 적용하면 양자화된 key와의 내적을 바로 계산할 수 있다.
정규화 과정에서 나눈 L2 norm 값만 별도로 저장해두면 scale도 쉽게 복원된다.

### 편향 없는 내적 - QJL 2단계 (Q_prod)

MSE에 최적인 양자화는 내적에서 곱셈적 편향(multiplicative bias)을 만든다.
1-bit의 경우 복원 내적의 기댓값이 실제 내적의 2/π 배로 축소된다.

```text
E[ <y, Q_mse^{-1}(Q_mse(x))> ] = (2/π) · <y, x>   (b=1)
```

Algorithm 2(Q_prod)는 이를 2단계로 해결한다.
먼저 (b-1) bit로 MSE 최적 양자화를 수행한 뒤, 남은 잔차 r에 1-bit QJL 변환을 적용하고 잔차의 L2 norm을 함께 저장한다.

```text
Q_prod(x) = [ Q_mse(x),  Q_qjl(r),  ||r||_2 ]
```

QJL은 고차원 데이터를 압축하면서 점들 사이 거리를 보존하는 변환으로, 각 숫자를 부호 1-bit로 줄이면서도 메모리 오버헤드가 사실상 0이다.
그 결과 내적 추정이 편향 없이 복원된다.

```text
E[ <y, x_tilde> ] = <y, x>   (unbiased)
```

### 이론적 보장

TurboQuant의 가장 큰 기여는 data-oblivious 설정에서 near-optimal distortion rate를 증명했다는 점이다.
distortion 상한은 다음과 같다.

```text
D_mse  ≤ (sqrt(3)·π / 2) · (1 / 4^b)
D_prod ≤ (sqrt(3)·π^2 · ||y||^2 / d) · (1 / 4^b)
```

Shannon lower bound와 Yao minimax 원리로 유도한 정보이론적 하한은 다음과 같다.

```text
D_mse(Q)  ≥ 1 / 4^b
D_prod(Q) ≥ (1 / d) · (1 / 4^b)
```

즉 TurboQuant는 하한과 약 2.7배 이내(작은 비트 폭에서는 b=1일 때 약 1.45배)로 붙는다.
좌표당 b bit에서 실제 distortion 수치는 다음과 같다.

| 비트 폭 b | MSE distortion | 이론 하한 (1/4^b) | Inner-product distortion |
|-----------|----------------|-------------------|--------------------------|
| 1 bit | 약 0.36 | 0.25 | 1.57/d |
| 2 bit | 약 0.117 | 0.0625 | 0.56/d |
| 3 bit | 약 0.03 | 0.0156 | 0.18/d |
| 4 bit | 약 0.009 | 0.0039 | 0.047/d |

기존 data-free 방식이 이론 최적치에 크게 못 미쳤던 것과 달리, TurboQuant는 격차 대부분을 메운다.
attention 연산 시에는 quantized 표현을 BF16/FP16으로 dequantize한 뒤 표준 attention을 수행한다(vLLM 구현 기준).

## 구현과 통합

### vLLM 업스트림

vLLM에는 PR #38479가 머지되어 GQA/MHA 계열 모델에서 TurboQuant KV 캐시 압축을 CLI 플래그로 활성화할 수 있다.

```bash
# 4-bit norm-corrected
vllm serve Qwen/Qwen3-30B-A3B-Instruct-2507 \
  --kv-cache-dtype turboquant_4bit_nc \
  --max-model-len 131072

# 3-bit (메모리 극단 제약 시)
vllm serve meta-llama/Llama-3.3-70B-Instruct \
  --kv-cache-dtype turboquant_3bit_nc
```

지원되는 변형은 turboquant_k8v4, turboquant_4bit_nc, turboquant_k3v4_nc, turboquant_3bit_nc 등이다.
-nc 접미사는 norm correction을 의미하며 공격적 양자화 시 정확도를 보존하기 위한 옵션이다.
지원 모델군은 Qwen, Llama, Mistral, Gemma 등 GQA/MHA 계열이다.
구현 상세는 [vLLM API 레퍼런스](https://docs.vllm.ai/en/latest/api/vllm/model_executor/layers/quantization/turboquant/){:target="_blank"}에서 확인할 수 있다.

### llama.cpp 및 커뮤니티 구현

llama.cpp 커뮤니티는 [Discussion #20969](https://github.com/ggml-org/llama.cpp/discussions/20969){:target="_blank"}와 후속 디스커션에서 Turbo3 / Turbo4 변형을 활발히 개발 중이다.

| 변형 | 비트/value | 압축률(FP16 대비) |
|------|-----------|-------------------|
| Turbo3 | 3.25 | 약 4.9배 |
| Turbo4 | 4.25 | 약 3.8배 |

흥미로운 점은 다수의 독립 구현(Metal, CUDA, Vulkan)이 Algorithm 2(QJL residual)를 생략하고 Lloyd-Max만으로 거의 동일한 perplexity를 얻는다고 보고한다는 사실이다.
참조 구현으로는 [0xSero/turboquant](https://github.com/0xSero/turboquant){:target="_blank"}(Triton 커널 + vLLM 어댑터), [varjoranta/turboquant-vllm](https://github.com/varjoranta/turboquant-vllm){:target="_blank"}(3.8배 압축 fork), vivekvar-dl/turboquant(pip install turbokv) 등이 있다.
SGLang은 [Issue #21618](https://github.com/sgl-project/sglang/issues/21618){:target="_blank"}에서 통합을 검토 중이다.

핵심 흐름을 의사코드로 표현하면 다음과 같다.

```python
from turboquant import codebook, rotation, quantizer

# 1) 코드북 미리 로드 (dim 128/256, bits 2/3/4)
cb = codebook.load(dim=128, bits=3)

# 2) Random rotation matrix (모델 init 시 1회)
Pi = rotation.random_orthogonal(d=128, seed=42)

# 3) KV 벡터 양자화
def quantize_kv(x):           # x: [d]
    y = Pi @ x                # rotate
    idx = cb.nearest(y)       # 각 좌표별 nearest centroid index
    return idx                # 3-bit indices (bit-packed)

# 4) Attention 시점에 dequantize -> BF16
def dequantize_kv(idx):
    y_hat = cb.lookup(idx)
    return (Pi.T @ y_hat).to(torch.bfloat16)
```

0xSero/turboquant 참조 구현의 권장 설정은 key 3-bit, value 2-bit(품질 민감 시 value 4-bit)이며, vLLM 0.18.0 / PyTorch 2.10 / CUDA 12.8 / Python 3.12 환경에서 35개 검증 테스트를 통과한다.

## 벤치마크 결과

### 원논문 KV 캐시 결과

Llama-3.1-8B-Instruct와 Ministral-7B-Instruct를 대상으로 한 원논문 실험 결과는 다음과 같다.

| 벤치마크 | 압축률 | TurboQuant | FP 베이스라인 |
|----------|--------|------------|----------------|
| Needle-in-a-Haystack (4k~104k) | 4배 | 0.997 | 1.000 |
| LongBench-E (avg) | 4배 (2.5~3.5비트) | 50.06 | 50.06 |
| H100 attention | - | 최대 8배 speedup | - |

LongBench 평균 점수를 비트 폭별로 보면 다음과 같다.

| 모델 | 설정 | LongBench 평균 | full cache |
|------|------|----------------|------------|
| Llama-3.1-8B | 3.5-bit | 50.06 | 50.06 |
| Llama-3.1-8B | 2.5-bit | 49.44 | 50.06 |
| Ministral-7B | 2.5-bit | 49.62 | 49.89 |

LongBench-E의 3비트 기준에서 KIVI는 48.50, TurboQuant은 50.06으로 보고되었고, Needle-in-a-Haystack은 104k 토큰까지 100% recall을 유지했다.
저자 측 핵심 메시지는 3 bits/channel에서 정확도 손실 없이 6배 압축, 3.5 bits/channel에서 절대적 품질 중립, 2.5 bits/channel에서 marginal degradation이다.
Google 블로그 기준으로는 긴 context에서 KV 메모리를 6배 줄이고, 학습이나 파인튜닝 없이 3비트로 양자화하며, H100에서 4-bit key를 사용할 때 attention logit 계산이 32-bit 대비 8배 빨라진다고 보고한다.

### 벡터 검색 결과

TurboQuant는 KV 캐시뿐 아니라 벡터 데이터베이스 검색에도 그대로 적용된다.
GloVe(d=200), OpenAI-3 임베딩(d=1536, d=3072)에서 PQ·RabitQ보다 일관되게 높은 recall을 보이면서, 인덱싱(양자화) 시간이 사실상 무시할 수준이다.

| 기법 | 100k 벡터 양자화 시간 |
|------|----------------------|
| TurboQuant | 0.0007~0.0021초 |
| Product Quantization | 37~494초 |
| RabitQ | 597~3957초 |

calibration이 없어 전처리가 수 시간에서 거의 0으로 줄어드는 것이 핵심 장점이며, 약 4.5배 압축에서도 우수한 recall을 유지한다.

### vLLM 팀의 독립 평가

[vLLM 공식 블로그(2026-05-11)](https://vllm.ai/blog/2026-05-11-turboquant){:target="_blank"}의 Red Hat AI 소속 Eldar Kurtić, Michael Goin, Alexandre Marques는 4개 모델(Llama-3.3-70B-Instruct, Qwen3-30B-A3B-Instruct-2507, Qwen3-30B-A3B-Thinking-2507, MiniMax-M2.7)과 5개 벤치마크(OpenAI/MRCR long-context, AIME25, GPQA:Diamond, MATH500, LiveCodeBench-v6)로 종합 평가를 수행했다.
FP8와 TurboQuant의 결정적 차이는 연산 정밀도에 있다.

| 항목 | FP8 | TurboQuant |
|------|-----|------------|
| 저장 정밀도 | FP8 | 3~4비트 |
| 연산 정밀도 | FP8 (하드웨어 가속) | BF16 (dequant 후) |
| 메모리 절감 | 2배 | 4배 이상 가능 |
| 하드웨어 가속 | 있음 | 없음 |

Qwen3-30B의 MRCR@256k 정확도(AUC) 결과는 다음과 같다.

| 설정 | AUC |
|------|-----|
| BF16 | 45.8% |
| FP8 | 43.1% |
| TQ k8v4 | 43.0% |
| TQ 4bit-nc | 42.3% |
| TQ k3v4-nc | 33.5% |
| TQ 3bit-nc | 31.2% |

Throughput(BF16=100%)은 FP8가 약 100%로 동등한 반면, TQ k8v4는 75~80%, TQ 3bit-nc는 66~73%로 떨어졌다.
KV capacity 증가는 FP8가 2.0배, TQ k8v4가 2.4배, TQ 4bit-nc가 2.3~3.4배, TQ 3bit-nc가 최대 3.7배였다.
한편 burst load 상황의 Llama-70B TTFT는 BF16 약 17초, TurboQuant 변형 3.5초 미만, FP8 약 1.3초로 측정되었다.
vLLM 팀의 결론은 명확하다.

> "FP8 via --kv-cache-dtype fp8 remains the best default for KV-cache quantization."

이유는 FP8가 KV 저장과 attention 연산 모두를 hardware-native FP8 Tensor Core로 처리하는 반면, TurboQuant는 KV만 3~4비트로 압축하고 attention 시 BF16으로 dequantize하는 오버헤드가 있기 때문이다.
즉 하드웨어 가속이 가능한 정밀도(FP8)는 단순 비트 수보다 실제 성능에 더 큰 영향을 주며, 긴 컨텍스트와 reasoning 벤치마크(AIME25, LiveCodeBench-v6)에서는 작은 정확도 하락도 크게 누적된다.

## 다른 양자화 기법과의 비교

| 기법 | 비트 | 캘리브레이션 | Attention 계산 | 핵심 아이디어 |
|------|------|--------------|----------------|---------------|
| FP8 (vLLM 기본) | 8 | 불필요 | FP8 Tensor Core 네이티브 | E4M3/E5M2, dequant 불필요 |
| AWQ | 4 | 필요 | dequant 후 FP16 | weight-only, salient channel 보존 |
| GPTQ | 3~4 | 필요 | dequant 후 FP16 | weight-only, error compensation |
| KIVI | 2~4 | 필요 | dequant | K per-channel, V per-token 비대칭 |
| KVQuant | 2~4 | 필요 | dequant | Non-uniform + per-channel scaling |
| PolarQuant | 1~4 | 불필요 | dequant | Polar 좌표 변환, Gaussian 가중 |
| TurboQuant | 1~4 | 불필요 | dequant 후 BF16 | Random rotation + Lloyd-Max |

TurboQuant의 차별점은 다음과 같이 요약된다.

- Training-free + Calibration-free - KIVI, KVQuant, AWQ, GPTQ와 달리 데이터 통계 수집이 필요 없다.
- Online 적용 - 들어오는 KV 벡터에 즉시 양자화 가능하며 prefill/decode 양쪽에서 동작한다.
- 이론적 최적성 - 정보이론 distortion 하한의 약 2.7배 이내라는 명시적 보장.
- vs PolarQuant - 동일 Google 라인이지만 Beta-distribution 분석 기반의 multi-bit 일반화.
- vs FP8 - 더 낮은 비트(3~4) 달성 가능하나 attention dequant 오버헤드 존재.

또한 weight 양자화(GPTQ/AWQ/GGUF)와 직교적이므로 AWQ(weight) + FP8(activation) + TurboQuant(KV) 같은 조합도 원칙적으로 가능하다.

## 한계와 비판

TurboQuant이 이상적이지 않은 점은 vLLM 평가와 커뮤니티 검증을 통해 점점 더 분명해지고 있다.

- vLLM 결론 - FP8가 throughput/지연/정확도 모든 축에서 동등 이상이며, TurboQuant 3-bit 변형은 long-context(128K~256K)에서 정확도가 20점 이상 급락하는 사례가 있다.
- 처리량 패널티 - Qwen3-30B에서 10~60%, Llama-70B에서 10~68% 지연 오버헤드가 발생한다.
- 헤드라인 수치 정정 - "6배 메모리 × 8배 속도"는 서로 다른 실험 세팅의 값이므로 곱해서는 안 된다는 점을 공저자가 직접 정정했다.
- 선행 연구 인용 누락 - HN 토론([id=47917577](https://news.ycombinator.com/item?id=47917577){:target="_blank"})에서 EDEN(NeurIPS 2021, ICML 2022) 저자 amitport는 TurboQuant이 EDEN의 제한된 버전이며 optimal scale 유도가 없어 덜 정확하다고 주장했다. DRIVE(NeurIPS 2021) "geometric rotation prior to extreme quantization + bias correction" 역시 인용되지 않았다는 비판이 별도로 제기되었다.
- 아키텍처 의존성 - Beta 분포 근사는 차원 d가 클 때 가장 정확하며, 좌표별 스칼라 양자화라 좌표 간 상관(cross-coordinate correlation)을 적극 활용하지는 않는다. Phi-4, DeepSeek-R1, GLM-4.7-Flash(head_dim=576) 등은 자동 감지에 실패해 FP16으로 fallback한다.
- value 양자화 병목 - head_dim=256 기준 3-bit key는 cosine similarity 1.000으로 거의 무손실이지만, 2-bit value는 0.940으로 떨어져 병목이 된다(4-bit value는 0.997). 민감한 태스크에는 4-bit value가 권장된다.
- 하이브리드 모델 한계 - full-attention 레이어에만 적용되어 linear-attention/Mamba 레이어는 압축되지 않으며, MoE·하이브리드 모델은 이득이 줄어든다(dense 2.0배 vs MoE 1.45배). 하이브리드 decode 경로에서는 매 스텝 전체 히스토리를 float32로 dequantize해야 하는 오버헤드가 남는다.

반대로 긍정적 측면도 분명하다.

- Consumer GPU(RTX 5090 32GB)에서 70B 모델 + 700K 컨텍스트 실행이 가능하다는 실측 사례.
- llama.cpp의 35B 모델 CPU 테스트에서 f16/q8_0/tq3_0 사이 prompt throughput이 사실상 동일하고 perplexity도 6.20 vs 6.19로 거의 일치.
- RTX 5090 + Qwen3.5-27B-AWQ(dense)에서 prefill 처리량 +5.7%, 토큰 용량 2.0배(457k→914k), KV 30GB 확보.
- 8x RTX 3090 + Qwen3.5-35B-A3B(MoE)에서 full-attention 레이어 KV 30.9% 절감, context 1.45배 확장.
- Burst load 상황에서 BF16 TTFT 17초가 TurboQuant 3.5초 미만으로 안정화.

## 결론과 권장 사용 시나리오

TurboQuant은 "calibration-free + online + 이론적 최적성"이라는 세 가지를 동시에 만족한 첫 KV 캐시·벡터 검색 양자화 알고리즘으로서, 학술적으로는 분명한 기여가 있다.
분포를 알 수 없는 KV 캐시 key를 랜덤 회전으로 잘 정의된 Beta 분포로 바꾼 뒤, 그 분포에 최적화된 Lloyd-Max 코드북으로 압축하고, 직교성 덕분에 dequantize 없이 attention score를 계산하며, QJL 잔차 처리로 내적까지 편향 없이 복원한다.
그러나 실제 production 추론 환경, 특히 vLLM 같은 high-throughput serving 엔진에서는 hardware-native FP8 Tensor Core를 가진 FP8가 throughput과 정확도에서 우위에 있다.
따라서 현재 권장 사용 시나리오는 다음과 같이 정리된다.

- 사용을 고려해야 할 때 - KV 캐시 메모리가 실제 병목인 경우, consumer GPU에서 long-context 추론을 시도하는 경우, burst load 상황의 TTFT 안정화가 중요한 경우, 벡터 DB 인덱싱 전처리 비용을 0에 가깝게 줄여야 하는 경우.
- 피해야 할 때 - 메모리 여유가 충분한 production serving, FP8 Tensor Core를 지원하는 H100 등 최신 GPU에서의 일반적 추론, 256K급 long-context에서 reasoning 정확도가 중요한 경우(3-bit 변형).
- 기본 권장 - 대부분의 경우 FP8(--kv-cache-dtype fp8)이 default. 메모리 극단 제약 시에만 TurboQuant 4bit-nc를 검토하고, 3bit-nc/k3v4-nc는 철저한 자체 검증 후에만 도입.

KV 캐시 양자화 자체는 LLM 인프라의 핵심 축으로 자리 잡았고, TurboQuant은 이 흐름에서 "데이터 비의존 online 양자화"라는 의미 있는 옵션을 제공한다.
다만 헤드라인 수치와 실측 사이의 갭은 항상 자체 워크로드로 검증해야 한다는 점을 분명히 보여주는 사례이기도 하다.

## Reference

- [TurboQuant: Online Vector Quantization with Near-optimal Distortion Rate (arXiv:2504.19874)](https://arxiv.org/abs/2504.19874){:target="_blank"}
- [TurboQuant 논문 HTML 전문](https://arxiv.org/html/2504.19874v1){:target="_blank"}
- [ICLR 2026 OpenReview](https://openreview.net/forum?id=tO3ASKZlok){:target="_blank"}
- [HuggingFace Papers 페이지](https://huggingface.co/papers/2504.19874){:target="_blank"}
- [Google Research Blog - TurboQuant: Redefining AI efficiency with extreme compression](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/){:target="_blank"}
- [vLLM Blog - A First Comprehensive Study of TurboQuant](https://vllm.ai/blog/2026-05-11-turboquant){:target="_blank"}
- [vLLM API Reference - turboquant](https://docs.vllm.ai/en/latest/api/vllm/model_executor/layers/quantization/turboquant/){:target="_blank"}
- [vLLM Feature Request Issue #38171](https://github.com/vllm-project/vllm/issues/38171){:target="_blank"}
- [SGLang Integration Issue #21618](https://github.com/sgl-project/sglang/issues/21618){:target="_blank"}
- [llama.cpp Discussion #20969 - Turbo3/Turbo4 구현](https://github.com/ggml-org/llama.cpp/discussions/20969){:target="_blank"}
- [llama.cpp Discussion #21155 - 5.2x KV 압축 및 다중 모델 호환성 테스트](https://github.com/ggml-org/llama.cpp/discussions/21155){:target="_blank"}
- [0xSero/turboquant - Triton + vLLM 참조 구현](https://github.com/0xSero/turboquant){:target="_blank"}
- [varjoranta/turboquant-vllm - 3.8x 압축 fork](https://github.com/varjoranta/turboquant-vllm){:target="_blank"}
- [Haystack 튜토리얼 - TurboQuant Quantization with HuggingFace](https://haystack.deepset.ai/tutorials/49_turboquant_quantization_with_huggingface){:target="_blank"}
- [PolarQuant 논문 (관련 선행, arXiv:2502.02617)](https://arxiv.org/abs/2502.02617){:target="_blank"}
- [QJL 논문 (관련 선행, arXiv:2406.03482)](https://arxiv.org/abs/2406.03482){:target="_blank"}
- [KIVI 논문 (비교 baseline, arXiv:2402.02750)](https://arxiv.org/pdf/2402.02750){:target="_blank"}
- [Hacker News - TurboQuant Google 발표 토론](https://news.ycombinator.com/item?id=47513475){:target="_blank"}
- [Hacker News - TurboQuant is a restricted version of EDEN](https://news.ycombinator.com/item?id=47917577){:target="_blank"}
