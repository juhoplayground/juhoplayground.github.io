---
layout: post
title: "Attention Is All You Need: 순환 구조를 걷어낸 Transformer 아키텍처"
author: 'Juho'
date: 2026-08-18 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Embedding]
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
2. [배경과 문제의식](#배경과-문제의식)
   - [순환 모델의 구조적 제약](#순환-모델의-구조적-제약)
   - [합성곱 기반 대안의 한계](#합성곱-기반-대안의-한계)
3. [방법론](#방법론)
   - [인코더-디코더 스택](#인코더-디코더-스택)
   - [Scaled Dot-Product Attention](#scaled-dot-product-attention)
   - [Multi-Head Attention](#multi-head-attention)
   - [모델 내 어텐션의 세 가지 사용처](#모델-내-어텐션의-세-가지-사용처)
   - [Position-wise Feed-Forward Network](#position-wise-feed-forward-network)
   - [임베딩과 Softmax](#임베딩과-softmax)
   - [Positional Encoding](#positional-encoding)
4. [Self-Attention을 선택한 이유](#self-attention을-선택한-이유)
5. [학습 설정](#학습-설정)
   - [데이터와 배치](#데이터와-배치)
   - [하드웨어와 학습 시간](#하드웨어와-학습-시간)
   - [옵티마이저와 Learning Rate 스케줄](#옵티마이저와-learning-rate-스케줄)
   - [정규화](#정규화)
6. [주요 결과](#주요-결과)
   - [WMT 2014 기계 번역](#wmt-2014-기계-번역)
   - [Model Variations 어블레이션](#model-variations-어블레이션)
   - [English Constituency Parsing](#english-constituency-parsing)
7. [한계와 주의사항](#한계와-주의사항)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

`Attention Is All You Need`는 2017년 NIPS(31st Conference on Neural Information Processing Systems)에서 발표된 Google Brain·Google Research·University of Toronto 공동 연구다.
저자는 Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N. Gomez, Lukasz Kaiser, Illia Polosukhin 8인이며 기여 순서는 무작위로 나열되어 있다.

논문의 주장은 제목 그대로다.
당시 sequence transduction 분야를 지배하던 순환 신경망과 합성곱 신경망을 전부 제거하고, 어텐션 메커니즘만으로 인코더-디코더를 구성해도 더 나은 품질을 얻을 수 있다는 것이다.
저자들은 이 아키텍처를 Transformer라고 명명했다.

핵심 성과는 세 가지다.
WMT 2014 English-to-German 번역에서 BLEU 28.4를 기록해 앙상블을 포함한 기존 최고 성능을 2 BLEU 이상 앞섰다.
WMT 2014 English-to-French에서는 8개 GPU로 3.5일 학습한 뒤 단일 모델 기준 최고 성능인 BLEU 41.8을 달성했으며, 이는 문헌상 최고 모델 학습 비용의 극히 일부에 해당한다.
그리고 English constituency parsing에 적용해 대규모·소규모 데이터 양쪽에서 잘 일반화됨을 보였다.

## 배경과 문제의식

### 순환 모델의 구조적 제약

LSTM과 gated recurrent 신경망은 언어 모델링과 기계 번역 같은 시퀀스 모델링 문제에서 확고한 최신 기법으로 자리잡고 있었다.
순환 모델은 입력·출력 시퀀스의 심볼 위치를 따라 계산을 분해한다.
위치를 계산 시간 단계에 정렬시켜, 이전 은닉 상태 ht-1과 위치 t의 입력을 함수로 받아 은닉 상태 ht의 시퀀스를 생성한다.

이 본질적인 순차성이 문제다.
학습 예제 내부의 병렬화가 원천적으로 불가능해지며, 시퀀스가 길어질수록 이 제약은 치명적이 된다.
메모리 제약 때문에 예제 간 배칭으로 이를 상쇄하기도 어렵기 때문이다.

factorization trick이나 conditional computation으로 계산 효율을 크게 개선한 연구들이 있었고 후자는 모델 성능도 함께 끌어올렸지만, 순차 계산이라는 근본 제약 자체는 그대로 남아 있었다.

어텐션 메커니즘은 이미 여러 태스크에서 설득력 있는 시퀀스 모델링의 필수 구성 요소가 되어 있었다.
입력·출력 시퀀스 내 거리와 무관하게 의존성을 모델링할 수 있기 때문이다.
그러나 극소수 사례를 제외하면 이런 어텐션은 항상 순환 신경망과 결합된 형태로 쓰였다.

### 합성곱 기반 대안의 한계

순차 계산을 줄이려는 목표는 Extended Neural GPU, ByteNet, ConvS2S의 토대이기도 하다.
이들은 합성곱 신경망을 기본 블록으로 삼아 모든 입출력 위치의 은닉 표현을 병렬로 계산한다.

문제는 임의의 두 위치 사이 신호를 연결하는 데 필요한 연산 수가 위치 간 거리에 따라 증가한다는 점이다.
ConvS2S는 선형으로, ByteNet은 로그 스케일로 증가한다.
이 때문에 멀리 떨어진 위치 간 의존성 학습이 더 어려워진다.

Transformer에서는 이 연산 수가 상수로 줄어든다.
다만 어텐션 가중치가 적용된 위치들을 평균 내는 과정에서 유효 해상도가 낮아지는 대가가 따르며, 저자들은 이를 Multi-Head Attention으로 상쇄한다.

self-attention(intra-attention)은 단일 시퀀스 내 서로 다른 위치를 연결해 그 시퀀스의 표현을 계산하는 어텐션 메커니즘이다.
독해, 추상 요약, textual entailment, 태스크 독립적 문장 표현 학습 등에서 이미 성공적으로 쓰이고 있었다.
end-to-end memory network는 시퀀스 정렬 순환 대신 순환 어텐션 메커니즘에 기반하며 단순 언어 질의응답과 언어 모델링에서 좋은 성능을 보였다.

저자들이 아는 한, Transformer는 시퀀스 정렬 RNN이나 합성곱 없이 오직 self-attention만으로 입출력 표현을 계산하는 최초의 transduction 모델이다.

## 방법론

경쟁력 있는 대부분의 신경망 시퀀스 transduction 모델은 인코더-디코더 구조를 갖는다.
인코더는 심볼 표현 입력 시퀀스 (x1, ..., xn)을 연속 표현 시퀀스 z = (z1, ..., zn)으로 매핑한다.
디코더는 z를 받아 출력 시퀀스 (y1, ..., ym)을 한 번에 한 원소씩 생성한다.
각 단계에서 모델은 auto-regressive하게 동작하며, 이전에 생성한 심볼을 다음 심볼 생성 시 추가 입력으로 소비한다.

Transformer는 이 전체 구조를 따르되, 인코더와 디코더 양쪽 모두를 stacked self-attention과 point-wise fully connected 레이어로 구성한다.

### 인코더-디코더 스택

인코더는 N = 6개의 동일한 레이어를 쌓아 만든다.
각 레이어는 두 개의 서브레이어를 갖는다.
첫 번째는 multi-head self-attention 메커니즘이고, 두 번째는 단순한 position-wise fully connected feed-forward 네트워크다.

두 서브레이어 각각에 residual connection을 적용한 뒤 layer normalization을 수행한다.
즉 각 서브레이어의 출력은 다음과 같다.

```text
LayerNorm(x + Sublayer(x))
```

여기서 Sublayer(x)는 해당 서브레이어가 구현하는 함수다.
residual connection을 가능하게 하기 위해 모델의 모든 서브레이어와 임베딩 레이어는 차원 d_model = 512의 출력을 만든다.

디코더 역시 N = 6개의 동일한 레이어로 구성된다.
인코더의 두 서브레이어에 더해, 인코더 스택 출력에 대해 multi-head attention을 수행하는 세 번째 서브레이어가 삽입된다.
인코더와 마찬가지로 각 서브레이어 주위에 residual connection과 layer normalization을 적용한다.

디코더의 self-attention 서브레이어는 각 위치가 후속 위치를 참조하지 못하도록 수정된다.
이 마스킹과 출력 임베딩을 한 위치만큼 offset하는 처리가 결합되면, 위치 i에 대한 예측은 i보다 앞선 위치의 알려진 출력에만 의존하게 된다.

### Scaled Dot-Product Attention

어텐션 함수는 query와 key-value 쌍의 집합을 출력으로 매핑하는 함수로 기술할 수 있다.
query, key, value, 출력은 모두 벡터다.
출력은 value들의 가중합으로 계산되며, 각 value에 할당되는 가중치는 query와 대응하는 key의 compatibility function으로 계산된다.

저자들이 사용하는 어텐션을 Scaled Dot-Product Attention이라 부른다.
입력은 차원 d_k의 query와 key, 차원 d_v의 value로 구성된다.
query와 모든 key의 dot product를 계산하고, 각각을 sqrt(d_k)로 나눈 뒤 softmax를 적용해 value에 대한 가중치를 얻는다.

실제 구현에서는 여러 query를 행렬 Q로 묶어 동시에 계산한다.
key와 value도 각각 행렬 K, V로 묶는다.

```text
Attention(Q, K, V) = softmax(Q * K^T / sqrt(d_k)) * V
```

가장 널리 쓰이는 어텐션 함수는 additive attention과 dot-product(multiplicative) attention 두 가지다.
dot-product attention은 스케일링 인자 1/sqrt(d_k)를 제외하면 이 알고리즘과 동일하다.
additive attention은 은닉층 하나짜리 feed-forward 네트워크로 compatibility function을 계산한다.

이론적 복잡도는 둘이 비슷하지만, 실제로는 dot-product attention이 훨씬 빠르고 공간 효율적이다.
고도로 최적화된 행렬 곱 코드로 구현할 수 있기 때문이다.

d_k가 작을 때는 두 방식의 성능이 비슷하지만, d_k가 커지면 스케일링 없는 dot-product attention보다 additive attention이 더 나은 성능을 보인다.
저자들은 d_k가 클 때 dot product의 크기가 커지면서 softmax를 기울기가 극도로 작은 영역으로 밀어넣는 것이 원인이라고 추정한다.
논문의 각주는 그 이유를 정량적으로 설명한다.
q와 k의 각 성분이 평균 0, 분산 1의 독립 확률변수라면 dot product는 평균 0, 분산 d_k를 갖는다.
이 효과를 상쇄하기 위해 dot product를 1/sqrt(d_k)로 스케일링한다.

### Multi-Head Attention

d_model 차원의 key, value, query로 단일 어텐션 함수를 수행하는 대신, 서로 다른 학습된 선형 투영으로 query, key, value를 각각 d_k, d_k, d_v 차원으로 h번 투영하는 것이 유익하다는 것을 발견했다.
투영된 각 버전에 대해 어텐션 함수를 병렬로 수행하면 d_v 차원 출력값이 나온다.
이들을 concat한 뒤 다시 한 번 투영해 최종값을 만든다.

```text
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) * W^O

where head_i = Attention(Q * W_i^Q, K * W_i^K, V * W_i^V)
```

투영 파라미터 행렬의 형태는 다음과 같다.

| 파라미터 | 형태 |
|----------|------|
| W_i^Q | d_model x d_k |
| W_i^K | d_model x d_k |
| W_i^V | d_model x d_v |
| W^O | (h * d_v) x d_model |

multi-head attention은 모델이 서로 다른 위치의 서로 다른 표현 부분공간(representation subspace) 정보를 동시에 참조할 수 있게 한다.
단일 어텐션 헤드만 쓰면 평균화 때문에 이런 능력이 억제된다.

이 논문에서는 h = 8개의 병렬 어텐션 레이어, 즉 헤드를 사용한다.
각 헤드에 대해 d_k = d_v = d_model / h = 64를 쓴다.
헤드별 차원이 줄어든 덕분에 전체 계산 비용은 전체 차원을 쓰는 단일 헤드 어텐션과 비슷한 수준으로 유지된다.

### 모델 내 어텐션의 세 가지 사용처

Transformer는 multi-head attention을 세 가지 방식으로 사용한다.

첫째, encoder-decoder attention 레이어다.
query는 이전 디코더 레이어에서 오고, memory key와 value는 인코더 스택 출력에서 온다.
이를 통해 디코더의 모든 위치가 입력 시퀀스의 모든 위치를 참조할 수 있다.
이는 전형적인 seq2seq 모델의 encoder-decoder 어텐션 메커니즘을 모방한 것이다.

둘째, 인코더의 self-attention 레이어다.
self-attention 레이어에서는 key, value, query가 모두 같은 곳, 즉 인코더 이전 레이어의 출력에서 온다.
인코더의 각 위치는 이전 레이어의 모든 위치를 참조할 수 있다.

셋째, 디코더의 self-attention 레이어다.
디코더의 각 위치가 자기 자신을 포함해 그 이전까지의 모든 디코더 위치를 참조하게 한다.
auto-regressive 속성을 보존하려면 디코더 내에서 왼쪽 방향으로의 정보 흐름을 막아야 한다.
이는 scaled dot-product attention 내부에서 softmax 입력 중 불법 연결에 해당하는 값들을 음의 무한대로 마스킹하는 방식으로 구현된다.

### Position-wise Feed-Forward Network

어텐션 서브레이어에 더해, 인코더와 디코더의 각 레이어는 fully connected feed-forward 네트워크를 포함한다.
이 네트워크는 각 위치에 개별적이고 동일하게 적용된다.
두 번의 선형 변환 사이에 ReLU 활성화가 들어간 구조다.

```text
FFN(x) = max(0, x * W1 + b1) * W2 + b2
```

선형 변환은 서로 다른 위치에 대해 동일하지만, 레이어마다 다른 파라미터를 사용한다.
이를 커널 크기 1의 합성곱 두 개로 기술할 수도 있다.
입출력 차원은 d_model = 512이고, 내부 레이어 차원은 d_ff = 2048이다.

### 임베딩과 Softmax

다른 시퀀스 transduction 모델과 마찬가지로, 학습된 임베딩으로 입력 토큰과 출력 토큰을 d_model 차원 벡터로 변환한다.
디코더 출력을 다음 토큰 확률로 변환할 때도 통상의 학습된 선형 변환과 softmax 함수를 사용한다.

이 모델에서는 두 임베딩 레이어와 pre-softmax 선형 변환 사이에 동일한 가중치 행렬을 공유한다.
임베딩 레이어에서는 이 가중치에 sqrt(d_model)을 곱한다.

### Positional Encoding

모델에 순환도 합성곱도 없기 때문에, 시퀀스의 순서 정보를 활용하려면 토큰의 상대적 또는 절대적 위치 정보를 별도로 주입해야 한다.
이를 위해 인코더와 디코더 스택의 최하단에서 입력 임베딩에 positional encoding을 더한다.
positional encoding은 임베딩과 동일한 차원 d_model을 가지므로 두 값을 그대로 더할 수 있다.

이 논문에서는 서로 다른 주파수의 sine, cosine 함수를 사용한다.

```text
PE(pos, 2i)   = sin(pos / 10000^(2i / d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i / d_model))
```

여기서 pos는 위치, i는 차원이다.
즉 positional encoding의 각 차원이 하나의 sinusoid에 대응한다.
파장은 2*pi에서 10000*2*pi까지 기하급수적으로 증가한다.

이 함수를 택한 이유는 모델이 상대적 위치로 참조하는 법을 쉽게 학습할 것이라고 가정했기 때문이다.
임의의 고정 offset k에 대해 PE(pos+k)를 PE(pos)의 선형 함수로 표현할 수 있기 때문이다.

학습된 positional embedding도 실험했고 두 방식은 거의 동일한 결과를 냈다.
sinusoidal 방식을 택한 이유는 학습 중 만난 것보다 긴 시퀀스 길이로 외삽할 수 있게 해줄 가능성 때문이다.

## Self-Attention을 선택한 이유

저자들은 가변 길이 심볼 표현 시퀀스 (x1, ..., xn)을 같은 길이의 다른 시퀀스 (z1, ..., zn)으로 매핑하는 문제에서 self-attention 레이어를 순환·합성곱 레이어와 비교한다.
비교 기준은 세 가지다.

첫째는 레이어당 총 계산 복잡도다.
둘째는 병렬화 가능한 계산량이며, 필요한 최소 순차 연산 수로 측정한다.
셋째는 네트워크 내 장거리 의존성 간 경로 길이다.

장거리 의존성 학습은 많은 시퀀스 transduction 태스크의 핵심 난제다.
이 능력에 영향을 주는 주요 요인 중 하나는 순방향·역방향 신호가 네트워크 내에서 통과해야 하는 경로의 길이다.
입력과 출력 위치의 임의 조합 사이 경로가 짧을수록 장거리 의존성 학습이 쉬워진다.

### 레이어 타입별 비교

여기서 n은 시퀀스 길이, d는 표현 차원, k는 합성곱 커널 크기, r은 restricted self-attention의 이웃 크기다.

| Layer Type | Complexity per Layer | Sequential Operations | Maximum Path Length |
|------------|----------------------|-----------------------|---------------------|
| Self-Attention | O(n^2 · d) | O(1) | O(1) |
| Recurrent | O(n · d^2) | O(n) | O(n) |
| Convolutional | O(k · n · d^2) | O(1) | O(log_k(n)) |
| Self-Attention (restricted) | O(r · n · d) | O(1) | O(n / r) |

self-attention 레이어는 상수 개의 순차 실행 연산으로 모든 위치를 연결하는 반면, 순환 레이어는 O(n)의 순차 연산이 필요하다.
계산 복잡도 측면에서 self-attention 레이어는 시퀀스 길이 n이 표현 차원 d보다 작을 때 순환 레이어보다 빠르다.
word-piece나 byte-pair 표현을 쓰는 최신 기계 번역 모델의 문장 표현에서는 대개 이 조건이 성립한다.

매우 긴 시퀀스를 다루는 태스크의 계산 성능을 개선하려면, self-attention을 각 출력 위치를 중심으로 크기 r의 이웃만 보도록 제한할 수 있다.
이 경우 최대 경로 길이는 O(n/r)로 늘어난다.
저자들은 이 접근을 향후 연구로 남겼다.

커널 폭 k가 n보다 작은 단일 합성곱 레이어는 모든 입출력 위치 쌍을 연결하지 못한다.
연속 커널의 경우 O(n/k)개, dilated convolution의 경우 O(log_k(n))개의 합성곱 레이어 스택이 필요해 경로 길이가 늘어난다.
합성곱 레이어는 일반적으로 순환 레이어보다 k배 비싸다.
separable convolution은 복잡도를 O(k · n · d + n · d^2)까지 상당히 낮추지만, k = n인 경우에도 그 복잡도는 self-attention 레이어와 point-wise feed-forward 레이어의 조합, 즉 이 논문이 택한 방식과 같다.

부수적 이점으로 self-attention은 더 해석 가능한 모델을 만들 수 있다.
어텐션 분포를 살펴보면 개별 헤드가 서로 다른 작업을 수행하도록 학습될 뿐 아니라, 상당수가 문장의 구문적·의미적 구조와 관련된 동작을 보인다.
논문 부록의 시각화에서는 6개 레이어 중 5번째 레이어의 인코더 self-attention이 동사 `making`의 원거리 의존성을 따라가 `making...more difficult` 구를 완성하는 예, 그리고 대명사 `its`에 대해 anaphora resolution에 관여하는 것으로 보이는 두 헤드의 예가 제시된다.

## 학습 설정

### 데이터와 배치

English-German은 약 450만 문장 쌍으로 구성된 표준 WMT 2014 데이터셋으로 학습했다.
문장은 byte-pair encoding으로 인코딩했으며 소스-타깃 공유 어휘는 약 37000 토큰이다.

English-French는 훨씬 큰 WMT 2014 데이터셋을 사용했다.
3600만 문장으로 구성되며 토큰은 32000 word-piece 어휘로 분할했다.

문장 쌍은 근사 시퀀스 길이 기준으로 배칭했다.
각 학습 배치는 소스 토큰 약 25000개와 타깃 토큰 약 25000개를 포함하는 문장 쌍 집합으로 구성된다.

### 하드웨어와 학습 시간

NVIDIA P100 GPU 8장이 장착된 단일 머신에서 학습했다.

| 모델 | 스텝당 시간 | 총 스텝 | 총 학습 시간 |
|------|-------------|---------|--------------|
| base | 약 0.4초 | 100,000 | 약 12시간 |
| big | 약 1.0초 | 300,000 | 약 3.5일 |

### 옵티마이저와 Learning Rate 스케줄

Adam 옵티마이저를 beta1 = 0.9, beta2 = 0.98, epsilon = 1e-9으로 사용했다.
learning rate는 학습 진행에 따라 다음 공식으로 변화시켰다.

```text
lrate = d_model^(-0.5) * min(step_num^(-0.5), step_num * warmup_steps^(-1.5))
```

이는 처음 warmup_steps 동안 learning rate를 선형으로 증가시키고, 그 이후에는 스텝 수의 역제곱근에 비례해 감소시키는 것에 해당한다.
warmup_steps는 4000을 사용했다.

### 정규화

학습 중 세 가지 정규화를 적용한다.

Residual Dropout은 각 서브레이어의 출력이 서브레이어 입력에 더해지고 정규화되기 전에 dropout을 적용한다.
추가로 인코더·디코더 스택 양쪽에서 임베딩과 positional encoding의 합에도 dropout을 적용한다.
base 모델은 P_drop = 0.1을 사용한다.

Label Smoothing은 값 ls = 0.1로 적용했다.
이는 모델이 더 불확실해지도록 학습되므로 perplexity를 악화시키지만, 정확도와 BLEU 점수는 개선한다.

## 주요 결과

### WMT 2014 기계 번역

WMT 2014 English-to-German에서 Transformer (big)은 앙상블을 포함한 기존 최고 보고 모델을 2.0 BLEU 이상 앞서며 BLEU 28.4의 새로운 최고 성능을 세웠다.
학습은 P100 GPU 8장으로 3.5일이 걸렸다.
base 모델조차 이전에 발표된 모든 모델과 앙상블을 능가했으며, 경쟁 모델 대비 학습 비용은 극히 일부에 불과했다.

English-to-French에서는 이전 최고 성능 모델 학습 비용의 1/4 미만으로 이전에 발표된 모든 단일 모델을 능가했다.
이 부분에서 논문 본문 6.1절은 BLEU 41.0으로, Abstract와 Table 2는 41.8로 기재하고 있어 수치가 일치하지 않는다.
English-to-French용 Transformer (big)은 dropout 비율로 0.3 대신 P_drop = 0.1을 사용했다.

추론 설정은 다음과 같다.
base 모델은 10분 간격으로 저장된 마지막 5개 체크포인트를 평균한 단일 모델을 사용했고, big 모델은 마지막 20개 체크포인트를 평균했다.
beam size 4, length penalty alpha = 0.6의 beam search를 사용했다.
추론 시 최대 출력 길이는 입력 길이 + 50으로 설정하되 가능한 경우 조기 종료했다.

학습 비용 FLOPs는 학습 시간, 사용 GPU 수, GPU별 지속 단정밀도 부동소수점 처리 능력 추정치를 곱해 산출했다.
GPU별로 K80은 2.8, K40은 3.7, M40은 6.0, P100은 9.5 TFLOPS 값을 사용했다.

newstest2014 기준 결과는 다음과 같다.

| Model | EN-DE BLEU | EN-FR BLEU | EN-DE 학습 비용 (FLOPs) | EN-FR 학습 비용 (FLOPs) |
|-------|------------|------------|--------------------------|--------------------------|
| ByteNet | 23.75 | - | - | - |
| Deep-Att + PosUnk | - | 39.2 | - | 1.0e20 |
| GNMT + RL | 24.6 | 39.92 | 2.3e19 | 1.4e20 |
| ConvS2S | 25.16 | 40.46 | 9.6e18 | 1.5e20 |
| MoE | 26.03 | 40.56 | 2.0e19 | 1.2e20 |
| Deep-Att + PosUnk Ensemble | - | 40.4 | - | 8.0e20 |
| GNMT + RL Ensemble | 26.30 | 41.16 | 1.8e20 | 1.1e21 |
| ConvS2S Ensemble | 26.36 | 41.29 | 7.7e19 | 1.2e21 |
| Transformer (base model) | 27.3 | 38.1 | 3.3e18 | 3.3e18 |
| Transformer (big) | 28.4 | 41.8 | 2.3e19 | 2.3e19 |

base 모델의 학습 비용 3.3e18 FLOPs는 ConvS2S의 9.6e18보다도 작으면서 BLEU는 27.3 대 25.16으로 앞선다.
big 모델의 2.3e19는 GNMT + RL Ensemble의 EN-FR 학습 비용 1.1e21의 약 2% 수준이다.

### Model Variations 어블레이션

Transformer 각 구성 요소의 중요도를 평가하기 위해 base 모델을 여러 방식으로 변형해 English-to-German 개발 세트 newstest2013에서 성능 변화를 측정했다.
앞 절과 같은 beam search를 쓰되 체크포인트 평균은 적용하지 않았다.
표에 기재된 perplexity는 byte-pair encoding 기준 word-piece 단위이므로 word 단위 perplexity와 비교해서는 안 된다.

| 변형 | 설정 | PPL (dev) | BLEU (dev) | params (10^6) |
|------|------|-----------|------------|---------------|
| base | N=6, d_model=512, d_ff=2048, h=8, d_k=d_v=64, P_drop=0.1, ls=0.1, 100K steps | 4.92 | 25.8 | 65 |
| (A) | h=1, d_k=d_v=512 | 5.29 | 24.9 | - |
| (A) | h=4, d_k=d_v=128 | 5.00 | 25.5 | - |
| (A) | h=16, d_k=d_v=32 | 4.91 | 25.8 | - |
| (A) | h=32, d_k=d_v=16 | 5.01 | 25.4 | - |
| (B) | d_k=16 | 5.16 | 25.1 | 58 |
| (B) | d_k=32 | 5.01 | 25.4 | 60 |
| (C) | N=2 | 6.11 | 23.7 | 36 |
| (C) | N=4 | 5.19 | 25.3 | 50 |
| (C) | N=8 | 4.88 | 25.5 | 80 |
| (C) | d_model=256, d_k=d_v=32 | 5.75 | 24.5 | 28 |
| (C) | d_model=1024, d_k=d_v=128 | 4.66 | 26.0 | 168 |
| (C) | d_ff=1024 | 5.12 | 25.4 | 53 |
| (C) | d_ff=4096 | 4.75 | 26.2 | 90 |
| (D) | P_drop=0.0 | 5.77 | 24.6 | - |
| (D) | P_drop=0.2 | 4.95 | 25.5 | - |
| (D) | ls=0.0 | 4.67 | 25.3 | - |
| (D) | ls=0.2 | 5.47 | 25.7 | - |
| (E) | sinusoid 대신 학습된 positional embedding | 4.92 | 25.7 | - |
| big | N=6, d_model=1024, d_ff=4096, h=16, P_drop=0.3, 300K steps | 4.33 | 26.4 | 213 |

행 (A)는 계산량을 일정하게 유지하면서 어텐션 헤드 수와 key·value 차원을 변화시킨 결과다.
단일 헤드 어텐션은 최적 설정보다 0.9 BLEU 낮고, 헤드가 너무 많아도 품질이 떨어진다.

행 (B)는 어텐션 key 크기 d_k를 줄이면 모델 품질이 나빠짐을 보여준다.
compatibility를 판정하는 일이 쉽지 않으며, dot product보다 정교한 compatibility function이 유익할 수 있음을 시사한다.

행 (C)와 (D)에서는 예상대로 큰 모델이 더 낫고, dropout이 과적합 방지에 매우 유용함을 확인했다.
행 (E)에서 sinusoidal positional encoding을 학습된 positional embedding으로 대체했더니 base 모델과 거의 동일한 결과가 나왔다.

### English Constituency Parsing

Transformer가 다른 태스크로 일반화되는지 평가하기 위해 English constituency parsing 실험을 수행했다.
이 태스크는 특유의 난점을 갖는다.
출력이 강한 구조적 제약을 받으며 입력보다 훨씬 길다.
또한 RNN seq2seq 모델은 소규모 데이터 환경에서 최고 성능에 도달하지 못했다.

Penn Treebank의 Wall Street Journal 부분, 약 4만 학습 문장으로 d_model = 1024의 4레이어 Transformer를 학습했다.
추가로 약 1700만 문장 규모의 high-confidence 및 BerkleyParser 코퍼스를 사용한 준지도 설정에서도 학습했다.
WSJ 전용 설정에는 16K 토큰 어휘를, 준지도 설정에는 32K 토큰 어휘를 사용했다.

dropout(어텐션과 residual 양쪽), learning rate, beam size는 Section 22 개발 세트에서 소수의 실험으로만 선택했고 나머지 파라미터는 English-to-German base 번역 모델에서 그대로 유지했다.
추론 시 최대 출력 길이를 입력 길이 + 300으로 늘렸다.
WSJ 전용과 준지도 설정 모두 beam size 21, alpha = 0.3을 사용했다.

WSJ Section 23 기준 결과는 다음과 같다.

| Parser | Training | WSJ 23 F1 |
|--------|----------|-----------|
| Vinyals and Kaiser et al. (2014) | WSJ only, discriminative | 88.3 |
| Petrov et al. (2006) | WSJ only, discriminative | 90.4 |
| Zhu et al. (2013) | WSJ only, discriminative | 90.4 |
| Dyer et al. (2016) | WSJ only, discriminative | 91.7 |
| Transformer (4 layers) | WSJ only, discriminative | 91.3 |
| Zhu et al. (2013) | semi-supervised | 91.3 |
| Huang and Harper (2009) | semi-supervised | 91.3 |
| McClosky et al. (2006) | semi-supervised | 92.1 |
| Vinyals and Kaiser et al. (2014) | semi-supervised | 92.1 |
| Transformer (4 layers) | semi-supervised | 92.7 |
| Luong et al. (2015) | multi-task | 93.0 |
| Dyer et al. (2016) | generative | 93.3 |

태스크 특화 튜닝이 거의 없었음에도 모델은 놀라울 만큼 잘 작동했다.
Recurrent Neural Network Grammar를 제외한 이전 보고 모델 전부보다 나은 결과를 냈다.
RNN seq2seq 모델과 달리 Transformer는 4만 문장의 WSJ 학습 세트만으로 학습해도 BerkeleyParser를 능가한다.

## 한계와 주의사항

첫째, self-attention의 레이어당 복잡도는 O(n^2 · d)로 시퀀스 길이에 대해 이차적이다.
논문은 self-attention이 순환 레이어보다 빠른 조건을 n이 d보다 작을 때로 명시한다.
즉 매우 긴 시퀀스에서는 이 우위가 성립하지 않는다.

둘째, 저자들은 긴 시퀀스 대응책으로 이웃 크기 r로 제한하는 restricted self-attention을 제시하지만, 이는 최대 경로 길이를 O(1)에서 O(n/r)로 악화시킨다.
게다가 이 접근은 향후 연구 과제로만 남겨졌고 논문 내에서 실험되지 않았다.

셋째, 어텐션 가중치가 적용된 위치들을 평균하는 과정에서 유효 해상도가 감소한다.
Multi-Head Attention이 이를 상쇄하는 장치지만 근본적으로 상수 연산 수를 얻기 위해 치른 대가다.

넷째, 행 (B) 어블레이션은 dot product가 compatibility function으로 충분히 정교하지 않을 수 있음을 시사한다.
d_k를 줄이면 품질이 떨어지며, 저자들 스스로 더 정교한 compatibility function이 유익할 수 있다고 언급한다.

다섯째, label smoothing은 perplexity를 악화시킨다.
정확도와 BLEU는 개선되지만 지표 간 트레이드오프가 존재한다.

여섯째, 평가 범위가 좁다.
실험은 WMT 2014 두 개 번역 태스크와 English constituency parsing에 한정된다.
텍스트 이외의 입출력 모달리티 확장은 논문 시점에서 계획 단계였다.

일곱째, 디코더는 여전히 auto-regressive하다.
학습 시 병렬화는 크게 개선되지만 생성 자체는 순차적으로 남아 있으며, 저자들은 생성의 비순차화를 향후 연구 목표로 명시한다.

여덟째, EN-FR BLEU 수치가 Abstract·Table 2(41.8)와 6.1절 본문(41.0) 사이에서 불일치한다는 점은 인용 시 유의할 부분이다.

## 결론

이 논문은 순환 레이어를 multi-head self-attention으로 대체해, 전적으로 어텐션에 기반한 최초의 시퀀스 transduction 모델 Transformer를 제시했다.

번역 태스크에서 Transformer는 순환·합성곱 레이어 기반 아키텍처보다 훨씬 빠르게 학습될 수 있다.
WMT 2014 English-to-German과 English-to-French 양쪽에서 새로운 최고 성능을 달성했으며, 전자에서는 이전에 보고된 모든 앙상블마저 능가했다.

저자들이 밝힌 향후 방향은 세 가지다.
텍스트 이외의 입출력 모달리티로 Transformer를 확장하는 것, 이미지·오디오·비디오처럼 큰 입출력을 효율적으로 다루기 위해 local·restricted 어텐션 메커니즘을 연구하는 것, 그리고 생성 과정을 덜 순차적으로 만드는 것이다.

학습·평가 코드는 tensorflow/tensor2tensor 저장소에 공개되어 있다.

## Reference

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
