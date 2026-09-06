---
layout: post
title: "Transformer의 Layer Normalization 배치 — 왜 최신 LLM은 Post-LN에서 Pre-LN으로 옮겨갔는가"
author: 'Juho'
date: 2026-09-06 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Benchmark]
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
2. [LayerNorm이 하는 일](#layernorm이-하는-일)
   - [LayerNorm의 정의](#layernorm의-정의)
   - [RMSNorm과의 차이](#rmsnorm과의-차이)
3. [Post-LN과 Pre-LN의 구조 차이](#post-ln과-pre-ln의-구조-차이)
   - [원조 Transformer는 Post-LN이었다](#원조-transformer는-post-ln이었다)
   - [두 구조의 형식적 정의](#두-구조의-형식적-정의)
4. [Xiong et al. 2020의 이론](#xiong-et-al-2020의-이론)
   - [LayerNorm Jacobian은 입력 norm에 반비례한다](#layernorm-jacobian은-입력-norm에-반비례한다)
   - [은닉 상태 norm의 깊이 의존성](#은닉-상태-norm의-깊이-의존성)
   - [Theorem 1과 chain rule](#theorem-1과-chain-rule)
   - [warm-up stage의 정체](#warm-up-stage의-정체)
5. [정량 실험 결과](#정량-실험-결과)
   - [기계번역과 warm-up 민감도](#기계번역과-warm-up-민감도)
   - [BERT 사전학습과 옵티마이저 비교](#bert-사전학습과-옵티마이저-비교)
6. [실제 모델의 채택 현황](#실제-모델의-채택-현황)
   - [Pre-Norm이 기본값이 된 계보](#pre-norm이-기본값이-된-계보)
   - [예외 — OLMo 2와 Gemma](#예외--olmo-2와-gemma)
   - [HuggingFace 구현의 네이밍 함정](#huggingface-구현의-네이밍-함정)
7. [Pre-LN의 한계와 대안](#pre-ln의-한계와-대안)
   - [분산 지수 증가와 Curse of Depth](#분산-지수-증가와-curse-of-depth)
   - [DeepNorm, Sandwich-LN, Peri-LN](#deepnorm-sandwich-ln-peri-ln)
   - [QK-Norm과 그에 대한 반론](#qk-norm과-그에-대한-반론)
8. [실무 현장에서 보고된 문제](#실무-현장에서-보고된-문제)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

Transformer 블록에서 Layer Normalization을 residual 덧셈 뒤에 둘 것인가, 아니면 sub-layer 입력 쪽으로 옮길 것인가.
이 한 줄짜리 배치 결정이 학습 안정성과 하이퍼파라미터 튜닝 비용을 좌우한다는 것이 지난 몇 년간 축적된 결론이다.

2017년 원조 Transformer는 Post-LN 구조였고, 학습에는 4000 step의 learning rate warm-up이 반드시 따라붙었다.
2019년 GPT-2는 LayerNorm을 각 sub-block의 입력으로 옮겼고, 2020년 Xiong et al.의 [On Layer Normalization in the Transformer Architecture](https://arxiv.org/abs/2002.04745){:target="_blank"}가 그 이유를 이론적으로 설명했다.
논문의 핵심 문장은 "the location of layer normalization matters"이다.

이 글은 Xiong et al.의 이론 사슬을 중심에 놓고, LayerNorm의 동작 원리에서 시작해 실제 오픈 웨이트 모델의 채택 현황, 그리고 Pre-LN 자체가 안고 있는 한계와 대안까지 정리한다.

## LayerNorm이 하는 일

### LayerNorm의 정의

LayerNorm은 [Ba, Kiros, Hinton (2016)](https://arxiv.org/abs/1607.06450){:target="_blank"}이 제안한 정규화 기법이다.
하나의 학습 샘플 안에서, 해당 레이어의 모든 은닉 유닛 H개에 대해 평균과 분산을 계산한다.
배치 축이 아니라 feature 축, 즉 dimension 축으로 표준화한다는 점이 핵심이다.

```
mu    = (1/H) * sum_{i=1..H} a_i

sigma = sqrt( (1/H) * sum_{i=1..H} (a_i - mu)^2 )

h = f[ (g / sigma) * (a - mu) + b ]
```

여기서 g는 learnable gain, b는 learnable bias다.
정규화 이후, 비선형 함수 이전에 적용된다.

BatchNorm은 미니배치 내 학습 샘플들에 대해 뉴런별로 통계를 계산한다.
반면 LayerNorm은 단일 학습 샘플에서 한 레이어의 뉴런에 들어오는 모든 summed input으로부터 평균과 분산을 계산한다.
따라서 미니배치 크기 의존성이 사라지고 online learning에도 적용할 수 있다.

LayerNorm이 갖는 불변성은 세 가지다.

| 불변성 | LayerNorm | BatchNorm |
|--------|-----------|-----------|
| weight matrix re-scaling | 있음 | 있음 |
| weight matrix re-centering | 있음 | 없음 |
| per-sample data re-scaling | 있음 | 없음 |

시퀀스 모델에 적합한 이유는 정규화 항이 현재 time-step에서 레이어로 들어오는 summed input에만 의존하기 때문이다.
가변 길이 시퀀스마다 별도의 통계를 유지할 필요가 없다.

### RMSNorm과의 차이

[Zhang & Sennrich (2019)](https://arxiv.org/abs/1910.07467){:target="_blank"}의 RMSNorm은 LayerNorm에서 평균 중심화를 완전히 제거하고 re-scaling만 남긴다.

```
a_bar_i = ( a_i / RMS(a) ) * g_i

RMS(a) = sqrt( (1/n) * sum_{i=1..n} a_i^2 )
```

논문의 가설은 명확하다.
LayerNorm이 성공한 이유는 re-centering invariance가 아니라 re-scaling invariance라는 것이다.
즉 평균을 빼는 연산은 성능에 크게 기여하지 않는다.

계산 절감 폭은 구현과 태스크에 따라 다르게 보고되었다.

| 태스크 | 속도 개선 |
|--------|-----------|
| 기계번역 (RNNSearch) | 7% ~ 64% |
| image-caption retrieval | 40% ~ 64% |
| reading comprehension | 약 15% |
| CIFAR-10 | 약 20% |

성능은 LayerNorm과 comparable하다고 보고되어, 사실상 drop-in replacement로 쓰인다.
현재 Llama, DeepSeek-V3, Qwen3 등 최신 모델은 거의 전부 RMSNorm을 사용한다.

## Post-LN과 Pre-LN의 구조 차이

### 원조 Transformer는 Post-LN이었다

[Attention Is All You Need](https://arxiv.org/abs/1706.03762){:target="_blank"} Section 3.1의 원문은 다음과 같다.

> "the output of each sub-layer is LayerNorm(x+Sublayer(x)), where Sublayer(x) is the function implemented by the sub-layer itself."

residual 덧셈을 먼저 하고 그 결과를 LayerNorm에 통과시키는 순서, 즉 Post-LN이다.
그리고 이 구조는 learning rate warm-up을 반드시 동반한다.
논문 Section 5.3의 스케줄은 다음과 같다.

```
lrate = d_model^(-0.5) * min( step_num^(-0.5),  step_num * warmup_steps^(-1.5) )
```

warmup_steps는 4000, d_model은 512였다.
BLEU는 base 모델 100K step 기준 WMT14 En-De 27.3, En-Fr 38.1이고, big 모델 300K step 기준 En-De 28.4, En-Fr 41.8이다.

한 가지 논쟁거리가 있다.
[Sebastian Raschka의 글](https://magazine.sebastianraschka.com/p/why-the-original-transformer-figure){:target="_blank"}은 원 논문의 다이어그램은 Post-LN을 그리고 있지만 공식 업데이트 코드 구현은 Pre-LN을 기본값으로 두고 있었다며 문서와 구현의 불일치를 지적한다.
반면 같은 글의 독자 댓글에서는 공식 구현도 dropout, add, normalize 순서라 논문 그림과 일치한다는 반론이 제기되어 저자와 논쟁이 이어졌다.
논문 본문 서술 자체는 위 인용처럼 Post-LN이며, "원 구현이 무엇이었는가"에 대해서는 커뮤니티 내부에서도 합의가 완전하지 않다.

Pre-LN을 대중화한 것은 [GPT-2](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf){:target="_blank"}다.
논문 Section 2.3 원문은 다음과 같다.

> "Layer normalization was moved to the input of each sub-block, similar to a pre-activation residual network and an additional layer normalization was added after the final self-attention block. A modified initialization which accounts for the accumulation on the residual path with model depth is used. We scale the weights of residual layers at initialization by a factor of 1/sqrt(N) where N is the number of residual layers."

세 가지가 한 세트로 묶여 있다는 점이 중요하다.
LN을 sub-block 입력으로 이동시키고, 마지막 self-attention block 뒤에 최종 LN을 추가하며, residual layer weight를 1/sqrt(N)으로 스케일하는 초기화를 함께 적용한다.

### 두 구조의 형식적 정의

Xiong et al. 논문 Table 1의 정의를 의사코드로 옮기면 다음과 같다.

```
# Post-LN Transformer layer l
a1 = MultiHeadAtt( x_l, [x_l(1), ..., x_l(n)] )
a2 = x_l + a1
a3 = LayerNorm( a2 )
a4 = ReLU( a3 @ W1_l + bias1_l ) @ W2_l + bias2_l
a5 = a3 + a4
x_(l+1) = LayerNorm( a5 )
```

```
# Pre-LN Transformer layer l
p1 = LayerNorm( x_l )
p2 = MultiHeadAtt( p1, [...] )
p3 = x_l + p2
p4 = LayerNorm( p3 )
p5 = ReLU( p4 @ W1_l + bias1_l ) @ W2_l + bias2_l
x_(l+1) = p3 + p5

# 스택 최상단에 최종 정규화가 추가된다
x_Final = LayerNorm( x_(L+1) )
```

구조적 차이는 두 가지로 요약된다.
Post-LN은 residual 덧셈 뒤에 LN이 놓이므로 residual 경로 위에 LN이 얹히고 identity path가 끊긴다.
Pre-LN은 residual branch 안쪽 입력에만 LN이 적용되므로 residual 경로가 순수한 identity로 끝까지 뚫린다.
대신 Pre-LN은 스택 맨 끝에 최종 LayerNorm이 하나 추가된다.

## Xiong et al. 2020의 이론

논문이 답하려는 질문은 하나다.
왜 Transformer 학습에는 learning rate warm-up이 필수인가.

Introduction에서 warm-up의 비용을 이렇게 정리한다.
warm-up은 "not only slows down the optimization process but also brings more hyperparameter tunings"이며, 최종 성능이 최대 learning rate 값과 warm-up iteration 수 양쪽에 민감하다.
BERT나 XLNet 같은 대규모 모델에서는 이 튜닝 비용이 특히 크다.

Related Work에서는 두 종류의 warm-up을 구분한다.
하나는 large-batch training에서 알려진 warm-up이고, 다른 하나는 배치 크기와 무관하게 Transformer 최적화에서 실무자 전부가 겪는 보편적 warm-up이다.
Pre-LN 구조 자체는 선행 연구가 있었지만, 논문은 "the troublesome warm-up stage still remains in their training pipelines"라며 warm-up을 제거해도 된다는 것을 이론적으로 밝힌 점을 자신의 기여로 제시한다.

### LayerNorm Jacobian은 입력 norm에 반비례한다

논증 전체의 핵심 장치는 Lemma 3이다.

```
|| J_LN(x) ||_2 = O( sqrt(d) / ||x||_2 )      where  J_LN(x) = d LN(x) / d x
```

LN의 Jacobian 크기가 입력 벡터의 norm에 반비례한다.
즉 LN을 통과해 역전파되는 gradient는 그 지점의 은닉 상태 크기로 나눠진다.
따라서 은닉 상태의 norm이 깊이에 따라 어떻게 자라는지가 곧 gradient가 얼마나 줄어드는지를 결정한다.

### 은닉 상태 norm의 깊이 의존성

Lemma 1은 준비 단계다.
X가 가우시안 벡터일 때 ReLU 통과 후의 기대 제곱 norm이 절반으로 줄어든다는 사실이다.

```
X ~ N(0, sigma^2 * I_d)  =>  E( ||ReLU(X)||^2 ) = (1/2) * sigma^2 * d
```

Lemma 2가 초기화 시점의 은닉 상태 norm을 두 구조에 대해 각각 계산한다.

```
Post-LN:  E( ||a5 at layer l||^2 ) = (3/2) * d          for all l > 0

Pre-LN:   (1 + l/2) * d  <=  E( ||x_l||^2 )  <=  (1 + 3l/2) * d     for all l > 0
```

해석은 다음과 같다.

| 구조 | 은닉 상태 norm | 원인 |
|------|----------------|------|
| Post-LN | 깊이 l과 무관한 상수 (3/2)d | 매 레이어 끝의 LN이 크기를 매번 리셋 |
| Pre-LN | 깊이 l에 선형 비례, 대략 l 곱하기 d 규모 | residual이 identity로 계속 누적 |

Lemma 3과 결합하면 결론이 나온다.
Pre-LN에서 최종 LN의 입력 norm은 대략 sqrt(L) 곱하기 sqrt(d) 규모이므로, 그 지점의 Jacobian이 sqrt(d) 나누기 sqrt(L 곱하기 d), 즉 1/sqrt(L)만큼 gradient를 줄여준다.
바로 이 1/sqrt(L)이 다음 정리의 결과를 만든다.

### Theorem 1과 chain rule

논문은 (epsilon, delta)-bounded라는 개념을 먼저 정의한다.
확률변수 Z가 (epsilon, delta)-bounded라는 것은 최소 1 빼기 delta의 확률로 Z의 상대 편차가 epsilon 이하라는 뜻이다.

Theorem 1의 결과는 다음과 같다.

```
Post-LN:  || dL / dW2_L ||_F  <=  O( d * sqrt(ln d) )

Pre-LN:   || dL / dW2_L ||_F  <=  O( d * sqrt(ln d / L) )
```

핵심 대비는 L의 유무다.
논문 표현으로 "for the Post-LN Transformer, the scale of the gradients to the last FFN layer is of order O(d sqrt(ln d)) which is independent of L"이다.
반면 Pre-LN은 1/sqrt(L)로 감소한다.
깊어질수록 출력층 gradient가 작아진다는 뜻이다.

이 결과가 나오는 경로는 chain rule에서 확인된다.
마지막 FFN 가중치의 gradient는 다음과 같이 분해된다.

```
| dL / dW2_L |  =  | dL/dx_(L+1)  *  J_LN( 마지막 LN의 입력 )  *  dx4_L / dW2_L |
```

마지막 FFN 가중치의 gradient는 반드시 마지막 LayerNorm의 Jacobian을 통과한다.
Post-LN에서 그 LN의 입력은 은닉 상태 norm이 상수 (3/2)d이므로 Jacobian이 sqrt(d) 나누기 sqrt(d), 즉 O(1) 수준이라 gradient를 줄여주지 못한다.
Pre-LN에서 그 LN의 입력은 norm이 L 곱하기 d 규모까지 자라 있으므로 Jacobian이 O(1/sqrt(L))로 gradient를 눌러준다.

정리하면 LayerNorm 입력이 Jacobian의 분모에 등장하는 구조와, Pre-LN에서 그 입력이 깊이에 따라 커지는 구조의 조합이 Pre-LN gradient 안정성의 원천이다.

실측 관측도 이와 일치한다.
Figure 3(a,b)에서 "the scale of the expected gradients grows along with the layer index for the Post-LN Transformer"이며, Pre-LN은 "the scale almost keeps the same for different layers"다.
Figure 3(c,d)에서는 레이어 수를 늘렸을 때 Post-LN의 출력층 gradient norm이 약 1.6 근처로 유지되는 반면 Pre-LN은 감소한다.

### warm-up stage의 정체

이 이론이 말하는 warm-up의 정체는 명확하다.
warm-up은 아키텍처적 문제, 즉 초기화 시점의 큰 gradient를 옵티마이저 쪽 step size로 임시 보정하던 장치다.
초기에 극히 작은 learning rate를 써서 큰 gradient가 파라미터를 망가뜨리는 것을 막는 방식이다.

Post-LN에서 gradient가 O(d sqrt(ln d))로 크고 L에 무관하게 유지되므로, 초기에 큰 learning rate를 쓰면 학습이 발산하거나 나쁜 지점에 갇힌다.
Pre-LN은 초기 gradient가 이미 작으므로 그 보정 장치 자체가 불필요해진다.
논문의 결론 문장은 "the layer normalization plays a crucial role in controlling the gradient scales"이며, LN을 residual block 안으로 옮기면 "we can safely remove the learning rate warm-up stage, and thus, the number of hyper-parameter is reduced"이다.

## 정량 실험 결과

실험은 기계번역 두 종류와 BERT 사전학습으로 구성된다.

| 실험 | 설정 |
|------|------|
| IWSLT14 De-En | 10K BPE, 6-layer enc/dec, embedding 512, FFN hidden 1024, heads 4, batch 4096 tokens |
| WMT14 En-De | 37K BPE, Transformer base, batch 8192 tokens per GPU, P40 16장 |
| BERT 사전학습 | English Wikipedia + BookCorpus 약 3.4B words, 12 layers, hidden 768, heads 12, P40 32장 |

옵티마이저는 양쪽 모두 Adam이며 beta는 (0.9, 0.98)이다.
Post-LN baseline은 warm-up 4000 step 후 inverse square root decay를 쓰고, Pre-LN은 warm-up 없이 초기 learning rate에서 시작해 특정 epoch에서 decay한다.

### 기계번역과 warm-up 민감도

IWSLT14 De-En에서 Post-LN(warm-up 사용)과 Pre-LN(warm-up 없음)은 최종 BLEU가 약 34로 동등했고 validation loss도 약 4로 같았다.
차이는 수렴 속도에서 나타났다.
논문 표현으로 "the 9-th checkpoint of the Pre-LN Transformer achieves nearly the same performance as 15-th checkpoint of the Post-LN Transformer"다.

Post-LN에서 warm-up을 줄였을 때의 민감도는 다음과 같다.

| warm-up step 수 | BLEU |
|------------------|------|
| 1 (사실상 warm-up 없음) | 약 8.45 |
| 500 | 약 31.16 |
| 4000 (baseline) | 약 34 |

warm-up을 4000에서 500으로만 줄여도 BLEU가 약 3점 떨어지고, 사실상 제거하면 학습이 붕괴한다.
WMT14 En-De에서도 Pre-LN은 warm-up 없이 경쟁력 있는 BLEU를 달성했고 최적 validation loss에 더 빨리 도달했다.

### BERT 사전학습과 옵티마이저 비교

BERT 사전학습에서 논문이 명시한 가속은 Post-LN baseline 대비 40% speed-up rate다.
downstream 평가인 GLUE의 MRPC와 RTE에서도 Pre-LN이 warm-up 없이 성능을 유지했다.

옵티마이저를 바꿔가며 본 결과는 warm-up이 무엇을 보정하고 있었는지를 잘 보여준다.

| 설정 | 결과 |
|------|------|
| Post-LN, SGD, warm-up 없음 | BLEU 0 수준, 학습 발산 |
| Post-LN, Adam, warm-up 없음 | BLEU 약 8.45 |
| Post-LN, Adam, warm-up 4000 | BLEU 약 34 |
| Pre-LN, Adam, warm-up 없음 | BLEU 약 34 |

RAdam은 Post-LN을 warm-up 없이 학습시킬 수 있게 해주지만, Pre-LN에서는 Adam 대비 차이가 거의 없다고 보고되었다.
옵티마이저 쪽 보정 장치가 Pre-LN에서는 더 이상 필요하지 않다는 뜻이다.

## 실제 모델의 채택 현황

### Pre-Norm이 기본값이 된 계보

[The Big LLM Architecture Comparison](https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison){:target="_blank"}이 정리한 2025년 오픈 웨이트 모델들의 정규화 배치를 보면 Pre-Norm이 사실상 기본값이다.
DeepSeek V3와 R1, Mistral Small 3.1, Llama 4, Qwen3, SmolLM3, Kimi K2, GPT-OSS, Grok 2.5, GLM-4.5가 모두 RMSNorm 기반 Pre-Norm을 쓴다.

HuggingFace `transformers`의 실제 구현 코드로 확인한 배치는 다음과 같다.

| 모델 | 정규화 배치 | 특징 |
|------|-------------|------|
| Transformer (2017) | Post-LN | warm-up 4000 step 필수 |
| GPT-2 (2019) | Pre-LN + 최종 LN | residual weight를 1/sqrt(N)으로 초기화 |
| Llama | Pre-LN, 레이어당 RMSNorm 2개 | 두 정규화 모두 sub-layer 입력에 적용 |
| DeepSeek-V3 | Pre-LN, 레이어당 RMSNorm 2개 | Llama와 코드 수준 동일, MLA와 MoE 결합 |
| Qwen3 | Pre-LN + QK-Norm | QKV-bias 제거, head 차원 단위 q norm과 k norm |
| Kimi K2 | Pre-Norm 계열 | 정규화 배치 대신 MuonClip으로 안정화 |
| Gemma 2 / Gemma 3 | 입력과 출력 모두 정규화, 레이어당 RMSNorm 4개 | sandwich 또는 Peri-LN 구조 |
| OLMo 2 | reordered norm(출력 정규화) + QK-Norm | 두 기법 병용 시 gradient spike score 0.108에서 0.069로 감소 |

Qwen3 테크리포트는 아키텍처를 "RMSNorm with pre-normalization"으로 기술하고, "remove QKV-bias used in Qwen2 and introduce QK-Norm to the attention mechanism to ensure stable training"이라고 밝힌다.

### 예외 — OLMo 2와 Gemma

Pre-Norm이 지배적인 흐름에서 눈에 띄는 예외가 둘 있다.

[OLMo 2](https://allenai.org/blog/olmo2){:target="_blank"}는 post-norm으로 회귀했다.
다만 원조 Post-LN과 같지는 않다.
정규화 층이 여전히 residual(skip connection) 안쪽에 있으면서 모듈 뒤에 배치되는 형태다.

```
h     := x + RMSNorm( Attention(x) )
h_out := h + RMSNorm( MLP(x) )
```

Ai2 블로그는 학습 안정성 개선을 위해 nonparametric layer norm에서 RMSNorm으로 전환하고, layer norm을 재배치하고, QK-Norm을 채택했으며, 여기에 Z-loss 정규화와 층 간 활성 및 그래디언트 스케일을 더 잘 보존하는 초기화를 더했다고 밝힌다.
ablation 결과 두 변경을 함께 적용하면 gradient의 spike score가 0.108에서 0.069로 감소했다.
커뮤니티 정리에 따르면 reordered norm과 QK-Norm은 각각 단독으로는 좋은 결과가 나오지 않았고 함께 적용했을 때 gradient L2 norm의 증가 추세와 스파이크성이 모두 개선되었다.

Gemma 2와 Gemma 3는 앞뒤를 모두 정규화한다.
Gemma 2 테크리포트 원문은 "To stabilize training, we use RMSNorm to normalize the input and output of each transformer sub-layer, the attention layer, and the feedforward layer"다.
Gemma 3 테크리포트도 "We use a Grouped-Query Attention (GQA) with post-norm and pre-norm with RMSNorm"이라고 기술하며, Gemma 2의 soft-capping을 QK-norm으로 교체했다고 밝힌다.

코드 레벨에서 Gemma 2 디코더 레이어에는 RMSNorm이 네 개 정의된다.

```python
self.input_layernorm            = Gemma2RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
self.post_attention_layernorm   = Gemma2RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
self.pre_feedforward_layernorm  = Gemma2RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
self.post_feedforward_layernorm = Gemma2RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
```

```python
residual = hidden_states
hidden_states = self.input_layernorm(hidden_states)
hidden_states, _ = self.self_attn(hidden_states=hidden_states, ...)
hidden_states = self.post_attention_layernorm(hidden_states)
hidden_states = residual + hidden_states

residual = hidden_states
hidden_states = self.pre_feedforward_layernorm(hidden_states)
hidden_states = self.mlp(hidden_states)
hidden_states = self.post_feedforward_layernorm(hidden_states)
hidden_states = residual + hidden_states
```

### HuggingFace 구현의 네이밍 함정

Llama 계열 코드에는 이름 때문에 오해하기 쉬운 지점이 있다.

```python
residual = hidden_states
hidden_states = self.input_layernorm(hidden_states)
hidden_states, _ = self.self_attn(hidden_states=hidden_states, ...)
hidden_states = residual + hidden_states

residual = hidden_states
hidden_states = self.post_attention_layernorm(hidden_states)
hidden_states = self.mlp(hidden_states)
hidden_states = residual + hidden_states
```

`post_attention_layernorm`이라는 이름 때문에 attention 출력 뒤에 붙는 Post-LN으로 읽기 쉽지만, 코드상으로는 attention 블록이 끝난 다음의 residual stream을 MLP에 넣기 전에 정규화하는 층이다.
즉 두 정규화 모두 sub-layer 입력에 적용되는 완전한 Pre-LN이다.

같은 이름이 모델 계열에 따라 다른 연산을 가리킨다는 점이 문제를 키운다.
Gemma의 `post_attention_layernorm`은 attention 출력 자체를 정규화한 뒤 residual에 더하지만, Llama의 동일 이름 모듈은 MLP로 들어가는 residual을 정규화한다.
[nnterp 이슈 #53](https://github.com/ndif-team/nnterp/issues/53){:target="_blank"}은 이 문제를 해석 도구 관점에서 정리했다.
Gemma-2, Gemma-3, GLM-4, OLMo-2는 서브레이어 출력에 post-norm이 모듈 바깥에 붙기 때문에 `self_attn`과 `mlp` 모듈의 출력이 실제로 residual stream에 더해지는 값과 다르다.
그 결과 레이어 출력이 레이어 입력과 attention 출력, MLP 출력의 합이라는 항등식이 깨지며, 측정된 상대 오차는 gemma-2에서 0.78, gemma-3에서 15, glm-4에서 1.9였다.
일반적인 pre-LN 아키텍처에서는 이 값이 0.0이다.
OLMo-2는 pre-attention norm이 아예 없다.

같은 문제군으로 NeMo에서 HuggingFace로 변환할 때 `mlp.linear_fc1.layer_norm_weight`를 `post_attention_layernorm.weight`로 잘못 매핑한 버그도 보고되었다.
정답은 `pre_feedforward_layernorm.weight`였다.
정규화 배치의 다양화는 곧 체크포인트 변환, 해석 도구, 양자화 파이프라인의 버그 표면적 확대로 이어진다.

## Pre-LN의 한계와 대안

### 분산 지수 증가와 Curse of Depth

Pre-LN이 안고 있는 문제로 반복 지적되는 것은 세 가지다.

| 문제 | 내용 |
|------|------|
| 활성 분산 폭증 | residual이 정규화되지 않은 채 누적되어 깊이에 따라 활성 크기가 지수적으로 증가하고 massive activation과 수치 불안정으로 이어짐 |
| Curse of Depth | 깊은 층의 Jacobian이 identity로 붕괴해 뒤쪽 sub-layer의 기여가 사라짐 |
| Representation collapse | LN이 residual 벡터 전체를 정규화하며 잠재 부분공간을 뒤섞어 attention 회로가 붕괴할 수 있음 |

[The Curse of Depth in Large Language Models](https://arxiv.org/abs/2502.05795){:target="_blank"}는 깊은 층이 기대보다 훨씬 덜 기여하는 현상을 Llama, Mistral, DeepSeek, Qwen 계열에서 확인하고 그 원인을 Pre-LN의 광범위한 사용으로 지목했다.
Pre-LN 출력 분산이 깊이에 따라 지수적으로 커져 깊은 블록의 미분이 항등 행렬에 가까워진다는 설명이다.
제안된 해결책 LayerNorm Scaling은 LN 출력 분산을 깊이의 제곱근에 반비례하도록 스케일링하며, 130M에서 7B 규모까지 Pre-LN 대비 성능 개선을 보고했다.
Pre-LN 모델은 특정 depth 이후 레이어를 잘라내도 정확도 변화가 거의 없다는 ablation이 반복 인용되며, 이것이 Pre-LN이 얕은 모델 앙상블처럼 동작한다는 주장의 실증적 근거로 쓰인다.

### DeepNorm, Sandwich-LN, Peri-LN

Pre-LN을 대체하거나 보완하려는 시도는 여러 갈래다.

[DeepNet](https://arxiv.org/abs/2203.00555){:target="_blank"}의 DeepNorm은 Post-LN 구조를 유지하되 residual 경로에 상수 alpha를 곱해 스케일하고 초기화를 상수 beta로 축소한다.

```
x_(l+1) = LN( alpha * x_l + G_l(x_l, theta_l) )
```

| Architecture | alpha | beta |
|--------------|-------|------|
| Encoder-only | (2N)^(1/4) | (8N)^(-1/4) |
| Decoder-only | (2M)^(1/4) | (8M)^(-1/4) |
| Encoder-decoder (encoder) | 0.81 곱하기 (N^4 M)^(1/16) | 0.87 곱하기 (N^4 M)^(-1/16) |
| Encoder-decoder (decoder) | (3M)^(1/4) | (12M)^(-1/4) |

논문의 문제 진단이 흥미롭다.
Post-LN은 초기화 시점의 exploding model update로 spurious local optima에 갇히고, Pre-LN은 안정적이지만 수렴한 Post-LN 대비 0.5에서 1.0 BLEU를 잃는다는 것이다.
즉 Pre-LN은 안정성을 얻는 대신 최종 품질을 조금 내주는 선택이라는 시각이다.
DeepNet은 Transformer를 1,000 layer까지 확장했고, WMT-17 En-De에서 100-layer 모델이 28.9 BLEU, OPUS-100에서 1,000-layer 모델이 48-layer baseline을 4.4 BLEU 상회했다.

[CogView](https://arxiv.org/abs/2105.13290){:target="_blank"}의 Sandwich-LN은 표준 pre-LN에 더해 각 residual branch 끝에 LayerNorm을 추가한다.
문제 진단은 LayerNorm의 출력이 기본적으로 hidden size의 제곱근에 비례하며, 특정 차원의 큰 값이 residual을 타고 층층이 증폭되어 FP16에서 value explosion과 NaN loss를 일으킨다는 것이다.
일반 pre-LN Transformer는 FP16에서 1,000 iteration 이내에 학습이 실패했지만, PB-Relax와 Sandwich-LN을 함께 쓰면 4B와 8.3B 모델을 FP16 연산과 FP32 optimizer state 조합으로 학습할 수 있었다.

[Peri-LN](https://arxiv.org/abs/2502.02732){:target="_blank"}은 모듈의 앞뒤를 모두 정규화하는 배치를 형식화했다.

```
Post-LN:  y_l = Norm( x_l + Module(x_l) )
Pre-LN:   y_l = x_l + Module( Norm(x_l) )
Peri-LN:  y_0 = Norm(x_0)
          y_l = x_l + Norm( Module( Norm(x_l) ) )
          y_L = Norm(x_L)
```

논문이 지적한 Pre-LN의 문제는 파라미터 업데이트가 입력을 증폭하면 은닉 상태 분산이 지수적으로 누적된다는 것, attention이나 MLP 모듈에서 발생한 큰 값이 residual identity path를 통해 그대로 유지된다는 것, 그리고 이로 인한 수치 불안정이다.
400M, 1.5B, 3.2B 파라미터에 모델당 30B 토큰, 설정당 5개 랜덤 시드로 실험한 결과는 다음과 같다.

| 항목 | 수치 |
|------|------|
| 1.5B 벤치마크 평균 | Pre-LN 53.71에서 Peri-LN 56.55로 2.84 상승 |
| 400M training loss | Pre-LN 3.43에서 Peri-LN 3.34로 감소 |
| 표준편차 | Peri-LN이 절반 이상 감소 |
| 분산 증가 패턴 | Peri-LN은 깊이에 대해 거의 선형, Pre-LN은 지수적 누적 |

논문은 Gemma2와 OLMo2가 이 output normalization을 뚜렷한 근거 설명 없이 조용히 채택하기 시작했다고 지적한다.
업계가 먼저 채택하고 학계가 나중에 이름을 붙인 구도다.
[네이버클라우드 CLOVA 기술블로그](https://clova.ai/tech-blog/%ED%9D%94%EB%93%A4%EB%A6%BC-%EC%97%86%EB%8A%94-%EC%95%88%EC%A0%95%EC%84%B1-peri-ln%EC%9C%BC%EB%A1%9C-%ED%95%99%EC%8A%B5-%EB%B0%9C%EC%82%B0%EC%9D%84-%EB%A7%89%EB%8B%A4){:target="_blank"}의 한국어 해설은 Pre-LN에서 히든 스테이트 분산이 층마다 기하급수적으로 폭증해 massive activation을 만들고, 히든 벡터 크기에 비례한 gradient 급등으로 V100 FP16 환경에서 표현 범위를 초과해 학습이 발산한다고 정리한다.

이 밖에도 [HybridNorm](https://arxiv.org/abs/2503.04598){:target="_blank"}은 블록 내부에서 attention에는 QKV normalization을, FFN에는 Post-Norm을 적용해 두 방식을 통합하고, Mix-LN은 앞쪽 층에 Post-Norm, 깊은 층에 Pre-Norm을 섞는다.
2026년 1월에는 [Post-LayerNorm Is Back: Stable, Expressive, and Deep](https://arxiv.org/abs/2601.19895){:target="_blank"}이 Post-LN의 실패 원인을 정규화 위치가 아니라 ResNet 방식 residual 경로로 재규정하고, residual을 Highway 방식 연결로 교체한 Keel 아키텍처로 1,000층을 넘는 깊이에서도 특별한 초기화 트릭 없이 안정적으로 학습된다고 주장했다.
다만 HuggingFace papers 페이지와 arXiv 어디에도 실질적인 커뮤니티 토론이 붙지 않아 검증된 반응은 아직 없는 상태다.

### QK-Norm과 그에 대한 반론

QK-Norm은 attention 모듈 내부에서 RoPE 적용 전 query와 key에 RMSNorm을 적용하는 기법이다.
기원은 2023년 ViT-22B 연구이며, LLM과 diffusion 양쪽으로 동시에 확산됐다.

```python
self.q_norm = Qwen3RMSNorm(self.head_dim, eps=config.rms_norm_eps)
self.k_norm = Qwen3RMSNorm(self.head_dim, eps=config.rms_norm_eps)

query_states = self.q_norm(self.q_proj(hidden_states).view(hidden_shape)).transpose(1, 2)
key_states   = self.k_norm(self.k_proj(hidden_states).view(hidden_shape)).transpose(1, 2)
```

Raschka는 QK-Norm을 "essentially yet another RMSNorm layer"로 요약하며 정규화 재배치와 함께 학습을 안정화한다고 긍정 평가한다.
[Kimi K2 테크리포트](https://arxiv.org/abs/2507.20534){:target="_blank"}는 다른 경로를 택했다.
Muon 옵티마이저를 대규모로 쓸 때 attention logit이 1000을 빠르게 넘어서며 수치 불안정으로 이어지는데, logit soft-capping은 capping 이전에 이미 dot product가 커져 있어 불충분하다는 진단이다.
대신 파라미터 업데이트 직후 query와 key projection weight를 head 단위로 리스케일하는 QK-Clip을 도입했다.

```
gamma = min( 1,  tau / S_max_h )
```

여기서 S_max_h는 배치 내 head h의 최대 attention logit이고, K2 사전학습에서 임계값 tau는 100이다.
Muon과 QK-Clip을 결합한 MuonClip으로 15.5조 토큰 사전학습을 loss spike 없이 완주했다.

반대편 시각도 있다.
[Kubar Labs의 MuonClip 해설](https://kubarlabs.substack.com/p/muonclip-and-the-stable-training){:target="_blank"}은 Query-Key Normalization이 단위 norm을 강제해 스케일 정보를 버리고 long-context 성능을 저하시킨다고 지적한다.
같은 글은 logit soft-capping이 attention 분포를 왜곡하고, gradient clipping은 성장의 원인이 아닌 gradient에만 작용한다고 정리한다.
2026년 Cohere Tiny Aya는 QK-Norm이 널리 쓰임에도 long context 성능과 상호작용할 수 있다는 이유로 의도적으로 제거했다.
찬반이 갈리는 살아있는 논쟁이다.

정의 자체가 프레임워크마다 다르다는 문제도 있다.
[gpt-neox PR #1367](https://github.com/EleutherAI/gpt-neox/pull/1367){:target="_blank"}은 QK norm을 어느 축에 적용할지가 파편화되어 있음을 조사표로 정리했다.
원 QK-Norm 논문, ViT-22B, Qwen3, TorchTitan, Megatron-Core는 head 차원 축으로 정규화해 헤드 간 혼합이 없지만, OLMo-core 기본값과 HuggingFace OLMo2는 전체 query 헤드를 flatten해서 정규화해 헤드 간 혼합이 발생한다.
발단 자체도 실제 버그였다.
기본값이 GQA와 충돌해 key 텐서의 shape이 맞지 않는 문제였다.
QK-Norm을 쓴다는 말만으로는 서로 다른 연산을 가리킬 수 있다는 것이 실무자들이 도달한 인식이다.

## 실무 현장에서 보고된 문제

논문이 말하는 Pre-LN의 분산 지수 증가는 현장에서 구체적인 장애로 나타난다.

[transformers 이슈 #39972](https://github.com/huggingface/transformers/issues/39972){:target="_blank"}는 학습이 아닌 추론 단계의 사례다.
Gemma3를 fp16으로 실행했을 때 5초에서 10초 만에 NaN이 발생하고 CUDA 커널이 크래시했다.
보고자는 fp32에서 hidden state가 약 264,000까지 누적되는 것을 확인했다.
fp16의 최대 표현값은 65,504이다.
원인으로 RMSNorm 층과, 경계 검사 없이 잔차를 계속 더하는 구조 두 가지가 지목되었으며, 보고자는 forward pass마다 값이 계속 커져 결국 fp64에서도 오버플로우할 것이라고 적었다.
제안된 임시 조치는 RMSNorm 직후와 attention 및 FFN 잔차 덧셈 직후에 값을 clamp하는 것이었다.

[Megatron-LM 이슈 #1799](https://github.com/NVIDIA/Megatron-LM/issues/1799){:target="_blank"}는 학습 쪽 사례다.
표준 Pre-LN과 input LayerNorm, MoE 로드밸런스 로스를 모두 적용했음에도 후반부 여러 레이어에서 hidden state 활성값의 크기가 급증했고, 평균과 분산이 층별로 누적 증가하며 gradient explosion과 NaN loss로 이어졌다.
요청자는 DeepNorm 지원을 요구했지만 메인테이너는 "DeepNorm isn't widely used in recent LLMs with most relying on RMSNorm (Llama, Qwen) and QK-Norm (DeepSeek)"이라고 답했다.

[xformers 이슈 #219](https://github.com/facebookresearch/xformers/issues/219){:target="_blank"}는 2022년 3월부터 7월까지 62개 댓글이 달린 장기 디버깅 스레드다.
ViT-B와 MAE 하이퍼파라미터를 timm과 1대1로 맞췄는데 xformers 쪽만 학습이 붕괴했다.
LayerNorm eps 차이, Triton 커널, weight init, AMP fp16을 차례로 소거해도 해결되지 않았다.
실제로 발견된 버그는 DeepNorm 초기화가 value projection을 찾도록 작성되어 있는데 self-attention은 QKV가 하나의 융합된 weight 모듈이라 스케일링이 적용되지 않는다는 것이었다.
그 버그를 고쳐도 ViT에서는 여전히 NaN이 발생했고, 8층 25M 미니 GPT로 pre, post, deepnorm을 비교한 통제 실험에서는 파국적 실패는 없었지만 DeepNorm이 나머지 둘보다 나쁜 해로 수렴했다.
최종 진단은 문제가 DeepNorm 고유가 아니라 post-norm 잔차 경로 자체가 ViT와 궁합이 나쁘다는 것이었으며, 유지보수자는 "it's not just deepnorm, the post-normalization path does not play well with ViT. GPT is fine with this normalization path"라고 정리했다.
해법은 정규화 배치 변경이 아니라 LayerNorm weight를 1이 아닌 매우 작은 값으로 초기화하는 것이었다.

RMSNorm 채택 사유가 품질이 아니라 컴파일 제약에서 출발했다는 기록도 있다.
[OLMo 이슈 #70](https://github.com/allenai/OLMo/issues/70){:target="_blank"}의 출발점은 다음과 같다.

> "torch's autocast behavior is hardcoded to upcast inputs to the function torch.layer_norm, and as a result using low precision LayerNorm requires some hacking that fails to work when using torch.compile(). Low precision LN gives a huge speedup with non-compiled models."

벤치마크 결과 autocast로 정밀도를 수동 제어하니 오히려 훨씬 느렸고, autocast를 건드리지 않으니 빨라졌다.
최종 권고는 컴파일할 때는 RMSNorm을, 컴파일하지 않을 때는 low precision LayerNorm을 쓰라는 조건부 결론이었다.
같은 스레드에서 처리량만이 아니라 다운스트림 성능 기준으로 봐야 한다는 반론과, 저정밀 LayerNorm이 충분히 안정적인지 BLOOM은 32비트 LayerNorm을 안정성 핵심 장치로 썼다는 지적이 제기되었고, 이에 대한 답변은 "That remains to be seen"이었다.
저정밀 정규화의 안정성은 미해결로 남겨두고 처리량을 택한 결정이다.

정규화 층이 많아질수록 커널 정밀도 문제가 복리로 누적된다는 사례도 있다.
[vllm 이슈 #42325](https://github.com/vllm-project/vllm/issues/42325){:target="_blank"}는 RMSNorm CUDA 커널이 weight dtype을 무시하고 항상 FP32로 곱하는 회귀를 보고했다.
단일 레이어 divergence가 0.03이었는데 20개 레이어를 지나면 124가 되었고, 최종 출력 일치율은 약 2%였다.
attention의 query와 key를 정규화하는 등 정규화 층이 많은 모델일수록 오차가 크게 증폭된다.

warm-up에 대해서도 이론과 실무 사이에 괴리가 있다.
Pre-LN은 초기화 시점 gradient가 잘 정의되어 있어 warm-up을 제거해도 된다는 것이 이론의 주장이다.
그러나 실무에서는 warm-up 설정이 최종 성능에 유의미한 영향을 주고 대규모 학습에서 이 하이퍼파라미터를 새로 튜닝하는 비용이 크기 때문에 안전한 디폴트로 남는다.
warm-up 비율을 10%에서 30%로 늘려도 Pre-LN의 gradient norm 스파이크 빈도는 줄지 않았다는 실증도 인용된다.
반대 방향의 규칙도 남아 있다.
[x-transformers](https://github.com/lucidrains/x-transformers){:target="_blank"} README는 residual attention 옵션을 설명하며 논문의 최고 성능이 post-normalization에서 나왔고 그 경우 learning warmup이 필요하다고 명시한다.
post-norm을 켜는 순간 warm-up은 선택이 아니라 요구사항이라는 실무 관행이다.

같은 README에는 정규화 변형에 대한 저자의 직접 평가가 남아 있다.

| 기법 | 평가 요지 |
|------|-----------|
| Sandwich Norm (CogView) | 학습 불안정에 직면했을 때 여러 프로젝트에서 매우 효과적이었다는 가장 강한 긍정 |
| ResiDual | pre-layernorm이나 sandwich norm보다 나쁘지 않게 동작하지만 공개 검증이 더 필요하다는 중립 평가 |
| NormFormer | 약간의 개선을 확인했으나 추가 연산 비용이 든다 |
| NormFormer의 residual scaling | 약간의 개선이 있으나 간헐적 불안정도 있어 주의해서 쓰라는 조건부 평가 |
| QK-Norm | 자기회귀 세팅에서 dot-product attention과 동등함을 검증, 22B 규모에서 안정성과 최종 성능 모두 개선된다는 분석 인용 |

부가 실무 팁으로 ResiDual은 fp16에서 prenorm 잔차가 오버플로우되기 쉬워 층마다 0.1 같은 스케일 팩터를 곱하는 것이 권장되며, LayerNorm이 입력 스케일에 불변이므로 결과는 유지된다는 점이 기록되어 있다.
임베딩 직후 LayerNorm은 BLOOM 175B와 YaLM 100B가 학습 안정화 목적으로 공통 채택한 것으로 소개된다.

문서와 구현의 불일치가 재현에 직접 영향을 준 사례도 있다.
[litgpt 이슈 #892](https://github.com/Lightning-AI/litgpt/issues/892){:target="_blank"}는 Pythia가 GPT-NeoX 기반이고 GPT-NeoX는 Megatron 코드를 사용하는데 Megatron 코드는 Post-LN 형태인 반면 GPT-NeoX 논문은 Pre-LN이라고 기술한다는 근거 사슬을 제시했다.
결론은 어느 쪽이 맞는지를 정하는 대신 Block 클래스에 pre와 post를 선택 가능한 인자로 노출하자는 제안으로 마무리되었다.
[nanoGPT 이슈 #278](https://github.com/karpathy/nanoGPT/issues/278){:target="_blank"}은 가장 흔한 입문자 혼란 유형으로, nanoGPT의 코드가 원 논문과 다른 이유를 묻는 질문이 2026년 현재까지 open 상태로 남아 있다.

마지막으로, 학습 안정성 장치로서의 정규화와 추론 시점에 실제로 필요한 정규화가 다를 수 있다는 논점이 있다.
[LessWrong 게시물](https://www.lesswrong.com/posts/THzcKKQd4oWkg4dSP/you-can-remove-gpt2-s-layernorm-by-fine-tuning-for-an-hour){:target="_blank"}은 약 1시간 파인튜닝으로 GPT-2에서 LayerNorm을 제거했고 손실 증가가 OpenWebText 기준 0.025, ThePile 기준 0.046에 그쳤다고 보고했다.
댓글 반응은 해석 가능성 측면의 환영과 일반화 손상 우려, confidence-regulation 메커니즘의 생존 여부에 대한 질문으로 갈렸다.

## 결론

이야기의 뼈대는 다음과 같이 정리된다.

2017년 원조 Transformer는 Post-LN이었고 warm-up 4000 step이 반드시 필요했지만 아무도 왜인지 정확히 설명하지 못했다.
2019년 GPT-2가 LN을 sub-block 입력으로 옮기고 최종 LN을 추가하며 residual weight를 1/sqrt(N)으로 초기화하면서 실무적으로 Pre-LN이 표준이 되어갔다.
2020년 Xiong et al.이 이론적 해명을 제시했다.
LayerNorm Jacobian이 입력 norm에 반비례하므로 LN 입력 norm이 분모에 들어가고, Post-LN은 매 층 끝의 LN이 은닉 norm을 상수로 리셋해 gradient가 L에 무관한 O(d sqrt(ln d))로 크게 남는 반면, Pre-LN은 residual 누적으로 norm이 깊이에 비례해 자라 마지막 LN이 gradient를 O(d sqrt(ln d / L))로 눌러준다.
warm-up은 Post-LN의 큰 초기 gradient를 옵티마이저로 때우던 임시방편이었던 셈이다.

여기에 RMSNorm이 re-centering을 버리고 계산을 절감하면서 Llama, DeepSeek-V3, Qwen3 모두 Pre-LN과 RMSNorm 조합으로 수렴했다.

그러나 Pre-LN도 완벽하지 않다.
DeepNet은 Pre-LN이 수렴한 Post-LN 대비 0.5에서 1.0 BLEU를 잃는다고 지적했고, CogView는 FP16 value explosion을 Sandwich-LN으로 해결했으며, Peri-LN은 은닉 상태 분산이 깊이에 따라 지수적으로 커진다는 점을 지적했다.
Gemma 2와 Gemma 3는 sub-layer 입력과 출력을 모두 정규화하고, OLMo 2는 출력 정규화로 재배치하며 QK-Norm을 함께 도입했고, Kimi K2는 QK-Clip으로 attention logit 폭주를 막는다.

논의의 중심은 이미 이동했다.
LN을 앞에 둘 것인가 뒤에 둘 것인가에서, residual stream의 분산과 attention logit의 크기를 어디서 어떻게 잡을 것인가로 옮겨갔다.
그리고 그 과정에서 정규화 배치의 다양화가 체크포인트 변환과 해석 도구, 양자화 파이프라인의 새로운 버그 표면을 만들어내고 있다는 점도 함께 고려해야 한다.

## Reference

- [On Layer Normalization in the Transformer Architecture (Xiong et al., ICML 2020)](https://arxiv.org/abs/2002.04745)
- [Layer Normalization (Ba, Kiros, Hinton, 2016)](https://arxiv.org/abs/1607.06450)
- [Attention Is All You Need (Vaswani et al., 2017)](https://arxiv.org/abs/1706.03762)
- [Root Mean Square Layer Normalization (Zhang & Sennrich, 2019)](https://arxiv.org/abs/1910.07467)
- [Language Models are Unsupervised Multitask Learners (GPT-2)](https://cdn.openai.com/better-language-models/language_models_are_unsupervised_multitask_learners.pdf)
- [DeepNet: Scaling Transformers to 1,000 Layers](https://arxiv.org/abs/2203.00555)
- [CogView: Mastering Text-to-Image Generation via Transformers (Sandwich-LN, PB-Relax)](https://arxiv.org/abs/2105.13290)
- [Peri-LN: Revisiting Normalization Layer in the Transformer Architecture](https://arxiv.org/abs/2502.02732)
- [The Curse of Depth in Large Language Models](https://arxiv.org/abs/2502.05795)
- [HybridNorm: Towards Stable and Efficient Transformer Training](https://arxiv.org/abs/2503.04598)
- [Post-LayerNorm Is Back: Stable, Expressive, and Deep](https://arxiv.org/abs/2601.19895)
- [OLMo 2 Technical Report](https://arxiv.org/abs/2501.00656)
- [Qwen3 Technical Report](https://arxiv.org/abs/2505.09388)
- [Kimi K2 Technical Report](https://arxiv.org/abs/2507.20534)
- [Gemma 2 Technical Report](https://arxiv.org/abs/2408.00118)
- [Gemma 3 Technical Report](https://arxiv.org/abs/2503.19786)
- [Ai2 - OLMo 2: The best fully open language model to date](https://allenai.org/blog/olmo2)
- [Sebastian Raschka - Why the Original Transformer Figure Is Wrong](https://magazine.sebastianraschka.com/p/why-the-original-transformer-figure)
- [Sebastian Raschka - The Big LLM Architecture Comparison](https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison)
- [Sebastian Raschka - A Dream of Spring for Open-Weight LLMs](https://magazine.sebastianraschka.com/p/a-dream-of-spring-for-open-weight)
- [Kubar Labs - MuonClip and the Stable Training of Trillion-Parameter Models](https://kubarlabs.substack.com/p/muonclip-and-the-stable-training)
- [네이버클라우드 CLOVA 기술블로그 - 흔들림 없는 안정성, Peri-LN으로 학습 발산을 막다](https://clova.ai/tech-blog/%ED%9D%94%EB%93%A4%EB%A6%BC-%EC%97%86%EB%8A%94-%EC%95%88%EC%A0%95%EC%84%B1-peri-ln%EC%9C%BC%EB%A1%9C-%ED%95%99%EC%8A%B5-%EB%B0%9C%EC%82%B0%EC%9D%84-%EB%A7%89%EB%8B%A4)
- [HuggingFace transformers - modeling_llama.py](https://github.com/huggingface/transformers/blob/main/src/transformers/models/llama/modeling_llama.py)
- [HuggingFace transformers - modeling_gemma2.py](https://github.com/huggingface/transformers/blob/main/src/transformers/models/gemma2/modeling_gemma2.py)
- [HuggingFace transformers - modeling_qwen3.py](https://github.com/huggingface/transformers/blob/main/src/transformers/models/qwen3/modeling_qwen3.py)
- [xformers issue #219 - ViT-B ImageNet MAE + Deepnorm training instability](https://github.com/facebookresearch/xformers/issues/219)
- [Megatron-LM issue #1799 - Deepnorm supporting](https://github.com/NVIDIA/Megatron-LM/issues/1799)
- [transformers issue #39972 - Gemma3 fp16 NaN](https://github.com/huggingface/transformers/issues/39972)
- [vllm issue #42325 - RMSNorm kernel precision regression](https://github.com/vllm-project/vllm/issues/42325)
- [OLMo issue #70 - Investigate RMSNorm as an alternative to LayerNorm](https://github.com/allenai/OLMo/issues/70)
- [gpt-neox PR #1367 - Default to only applying QK norm over the head dimension](https://github.com/EleutherAI/gpt-neox/pull/1367)
- [litgpt issue #892 - Pre and Post LayerNorm](https://github.com/Lightning-AI/litgpt/issues/892)
- [nanoGPT issue #278 - Why LayerNorm before Self-Attention?!](https://github.com/karpathy/nanoGPT/issues/278)
- [nnterp issue #53 - Sandwich norm architectures](https://github.com/ndif-team/nnterp/issues/53)
- [lucidrains/x-transformers](https://github.com/lucidrains/x-transformers)
- [LessWrong - You can remove GPT2's LayerNorm by fine-tuning for an hour](https://www.lesswrong.com/posts/THzcKKQd4oWkg4dSP/you-can-remove-gpt2-s-layernorm-by-fine-tuning-for-an-hour)
