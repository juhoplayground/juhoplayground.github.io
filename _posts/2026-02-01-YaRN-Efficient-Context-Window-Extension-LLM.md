---
layout: post
title: YaRN - LLM 컨텍스트 윈도우를 효율적으로 확장하는 방법
author: 'Juho'
date: 2026-02-01 02:00:00 +0900
categories: [AI]
tags: [LLM, Fine-tuning, Research, RoPE, Performance]
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
2. [배경 - RoPE의 한계](#배경---rope의-한계)
3. [기존 컨텍스트 확장 방법](#기존-컨텍스트-확장-방법)
4. [YaRN의 핵심 기법](#yarn의-핵심-기법)
5. [실험 결과](#실험-결과)
6. [Dynamic YaRN](#dynamic-yarn)
7. [의의와 영향](#의의와-영향)
8. [Reference](#reference)

## 개요

YaRN(Yet another RoPE extensioN method)은 Rotary Position Embedding(RoPE) 기반 LLM의 컨텍스트 윈도우를 효율적으로 확장하는 방법이다.
Nous Research, EleutherAI, 제네바 대학교의 Bowen Peng 등이 제안하였으며, ICLR 2024에 게재되었다.

트랜스포머 기반 LLM은 사전학습 시 설정된 시퀀스 길이를 넘어서면 성능이 급격히 저하되는 문제가 있다.
YaRN은 이 문제를 해결하기 위해 기존 방법 대비 10배 적은 토큰과 2.5배 적은 학습 스텝만으로 컨텍스트 윈도우를 확장한다.
원래 사전학습 데이터의 약 0.1%만으로 파인튜닝이 가능하며, LLaMA 모델 기준 최대 128K 컨텍스트까지 확장에 성공했다.

## 배경 - RoPE의 한계

### RoPE란

Rotary Position Embedding(RoPE)은 트랜스포머의 위치 인코딩 기법 중 하나이다.
어텐션 점수가 토큰의 절대적 위치가 아닌 상대적 거리에만 의존하도록 회전 행렬(rotation matrix)을 적용한다.
Query와 Key 벡터에 토큰 위치 기반의 삼각함수를 적용하여 위치 정보를 인코딩한다.
LLaMA, Mistral 등 주요 오픈소스 LLM에서 널리 사용되고 있다.

### 컨텍스트 윈도우 한계

RoPE 기반 모델은 사전학습 시 설정된 시퀀스 길이(L)를 넘어서면 일반화에 실패한다.
예를 들어 4096 토큰으로 학습된 모델에 8192 토큰의 입력을 넣으면 Perplexity가 급격히 상승하고 출력 품질이 크게 떨어진다.
이는 학습 시 보지 못한 위치 인코딩 값이 외삽(extrapolation)되면서 발생하는 문제이다.

## 기존 컨텍스트 확장 방법

YaRN 이전에 제안된 두 가지 주요 방법이 있다.

### Position Interpolation (PI)

Position Interpolation은 위치 인덱스를 s = L'/L 비율로 축소하여 새로운 시퀀스를 원래 컨텍스트 윈도우 안에 압축하는 방법이다.
완전한 재학습 없이 약 1~10B 토큰의 파인튜닝만으로 컨텍스트를 확장할 수 있다.

하지만 PI에는 한계가 있다.
모든 RoPE 차원을 동일한 비율로 축소하기 때문에 고주파(high-frequency) 정보가 손실된다.
또한 파인튜닝 후에도 짧은 시퀀스에 대한 Perplexity가 소폭 상승하는 부작용이 발생한다.

### NTK-aware Interpolation

Neural Tangent Kernel(NTK) 이론에 기반한 방법으로, 모든 차원을 균일하게 축소하는 대신 고주파는 덜 축소하고 저주파는 더 축소하는 방식이다.
RoPE의 base 파라미터를 b' = b * s^(D/(D-2))로 조정하여 보간 압력을 여러 차원에 분산시킨다.

파인튜닝 없이도 PI보다 좋은 성능을 보이지만, 파인튜닝 후에는 PI보다 성능이 떨어지는 혼재된 결과를 보인다.

### 균일 스케일링이 실패하는 이유

RoPE 임베딩을 무차별적으로 늘리면 네트워크가 유사하고 가까운 토큰들을 구분하는 데 필요한 고주파 세부 정보가 손실된다.
균일 스케일링 시 토큰 간 거리가 가까워져 인접한 토큰들의 위치 순서를 혼동하게 된다.
이로 인해 모델이 지역적(local) 관계를 이해하는 능력이 저하된다.

## YaRN의 핵심 기법

YaRN은 세 가지 핵심 기법을 결합하여 위 문제들을 해결한다.

### NTK-by-parts Interpolation

NTK-by-parts는 RoPE 차원마다 다른 보간 전략을 적용하는 하이브리드 방법이다.
각 차원의 파장(wavelength) λ와 원래 컨텍스트 길이 L의 비율 r = L/λ을 기준으로 보간 여부를 결정한다.

핵심 아이디어는 다음과 같다.
- 파장이 L보다 훨씬 짧은 고주파 차원은 보간하지 않는다(원본 유지).
- 파장이 L 이상인 저주파 차원은 보간만 수행한다(외삽 방지).
- 중간 영역의 차원은 두 전략을 혼합하여 부드러운 전환(ramp function)을 적용한다.

수식으로 표현하면 다음과 같다.

h(θ_d) = (1 - γ) * θ_d / s + γ * θ_d

여기서 γ는 파라미터 α와 β에 의해 정의되는 구간별 선형 램프 함수(piecewise linear ramp function)이다.
LLaMA 모델 계열에서 α = 1, β = 32가 실험적으로 최적값으로 확인되었다.

이 방식은 고주파 정보의 손실을 방지하면서도 파인튜닝 성능을 유지한다.

### Attention Scaling (Temperature)

어텐션 소프트맥스 계산에 온도(temperature) 파라미터 t를 도입한다.

softmax(q_m^T * k_n / (t * sqrt(D)))

56개의 16K 토큰 문서에 대한 실험으로 최적의 온도 관계식을 도출했다.

sqrt(1/t) = 0.1 * ln(s) + 1

이 수식은 모델 크기와 데이터에 관계없이 LLaMA 2의 7B, 13B, 70B 변형과 원래 LLaMA 모델에서 모두 잘 작동한다.
RoPE를 2D 행렬 집합으로 재매개변수화(reparametrization)하면 Query와 Key 벡터에 상수 인자를 곱하는 것만으로 어텐션 메커니즘을 효과적으로 변경할 수 있다.
이 방식은 추론과 학습 모두에서 오버헤드가 전혀 없다는 장점이 있다.

### YaRN의 최종 정의

YaRN은 NTK-by-parts Interpolation과 Attention Scaling의 결합이다.
튜닝해야 하는 파라미터는 다음과 같다.
- α : 램프 함수 시작점
- β : 램프 함수 끝점
- t : 온도 스케일
- L' : 목표 컨텍스트 길이

Mistral-7B를 128K 컨텍스트로 확장할 경우 온도 파라미터 수식의 계수는 a = 0.07, b = 1.0으로 설정한다.

## 실험 결과

### Perplexity 비교

LLaMA 2 모델을 대상으로 Proof-pile 데이터셋에서 슬라이딩 윈도우 Perplexity(S = 256)를 측정했다.

#### 8K 컨텍스트 확장 (LLaMA 2)

| 방법 | Perplexity | 학습 토큰 | 학습 스텝 |
|------|-----------|----------|----------|
| PI | 3.34 | 10x | 2.5x |
| NTK-aware | 3.59 | - | - |
| YaRN | 3.35 | 1x | 1x |

YaRN은 PI와 유사한 Perplexity를 달성하면서도 10배 적은 토큰과 2.5배 적은 학습 스텝만 사용했다.

#### 64K 컨텍스트 확장 (7B 모델, 65536 토큰 평가)

| 방법 | Perplexity |
|------|-----------|
| Code LLaMA (NTK-aware) | 2.55 |
| Together (PI) | 발산 (32K 이상) |
| YaRN (s=16) | 2.42 |

YaRN(s=16)이 65536 토큰에서 2.42의 Perplexity를 달성하여 Code LLaMA의 NTK-aware(2.55)를 상회했다.
Together의 PI 방식은 32K를 넘어서면 Perplexity가 발산했다.

#### Mistral 7B 128K 확장

| 방법 | Perplexity (131072 토큰) |
|------|------------------------|
| Mistral v0.1 (기본) | 발산 |
| MistralLite (NTK-aware) | 발산 |
| YaRN (s=16) | 2.19 |

Mistral 7B에서 YaRN(s=16)은 131072 토큰에서 2.19의 Perplexity를 기록했다.
기본 Mistral v0.1과 MistralLite는 해당 길이에서 Perplexity가 발산했다.

### Passkey Retrieval 테스트

8K~128K 범위의 컨텍스트에서 무작위 위치에 삽입된 패스키를 검색하는 테스트를 10회 반복 수행했다.
YaRN으로 128K 컨텍스트에서 파인튜닝된 7B와 13B 모델 모두 전체 컨텍스트 윈도우에서 99% 이상의 정확도를 달성했다.

흥미로운 점은 YaRN(s=32)이 YaRN(s=16)보다 200 스텝 더 학습했음에도 Perplexity는 유사하면서 패스키 정확도는 더 높았다는 것이다.
이는 Perplexity가 LLM의 모든 토큰에 대한 어텐션 능력을 판단하는 좋은 지표가 아닐 수 있음을 시사한다.

### 컨텍스트 외삽 능력

YaRN(s=32) 모델은 64K 토큰 데이터로 파인튜닝되었음에도 128K 토큰까지 Perplexity가 계속 감소하는 결과를 보였다.
이는 파인튜닝 데이터에 포함되지 않은 더 긴 컨텍스트 길이로의 성공적인 일반화(extrapolation)를 입증한다.

### Hugging Face Open LLM 벤치마크

짧은 컨텍스트에서의 성능 저하를 검증하기 위해 다음 벤치마크를 수행했다.
- ARC-Challenge (25-shot)
- HellaSwag (10-shot)
- MMLU (5-shot)
- TruthfulQA (0-shot)

YaRN으로 파인튜닝된 모델은 기존 RoPE 보간 방법보다 모든 벤치마크에서 개선된 결과를 보였다.
원래 모델의 짧은 컨텍스트 능력을 유지하면서도 긴 컨텍스트를 처리할 수 있다는 것이 확인되었다.

## Dynamic YaRN

Dynamic YaRN은 추론 시점에서 실제 시퀀스 길이에 따라 스케일 팩터를 동적으로 조정하는 기법이다.

s = l'/L (l'/L > 1인 경우), 그렇지 않으면 s = 1

이 방식은 원래 컨텍스트 길이 이하에서는 성능 저하를 방지하고, 이를 초과할 때만 스케일링을 적용한다.
파인튜닝 없이도 원래 컨텍스트의 2배 이상 확장이 가능하다는 것이 핵심이다.
Flash Attention 2 등 어텐션 메커니즘을 수정하는 라이브러리와도 직접 호환된다.

## 의의와 영향

### 효율성

YaRN은 원래 사전학습 데이터의 약 0.1%만으로 파인튜닝이 가능하다.
약 400 학습 스텝으로 컨텍스트 확장이 완료되며, 이는 기존 대비 10배 적은 토큰과 2.5배 적은 학습 스텝이다.
추론 시 추가 오버헤드가 거의 없어 실용적이다.

### 호환성

PI의 직접적인 대체제(drop-in replacement)로 사용할 수 있다.
기존 코드에서 최소한의 구현 변경만으로 적용 가능하다.
Flash Attention 2와 호환되어 실제 서비스 환경에서도 활용할 수 있다.

### 후속 영향

YaRN의 아이디어는 이후 LongRoPE, CodeLLaMA 등에서 활용되었다.
현재 많은 오픈소스 LLM의 컨텍스트 확장에 YaRN 또는 유사한 기법이 적용되고 있다.
LLaMA 2 기반 32K, 64K, 128K 컨텍스트 모델이 Hugging Face에 공개되어 있다.

## Reference
- [YaRN: Efficient Context Window Extension of Large Language Models - arXiv](https://arxiv.org/abs/2309.00071)
- [YaRN - ICLR 2024 Proceedings](https://proceedings.iclr.cc/paper_files/paper/2024/file/874a4d89f2d04b4bcf9a2c19545cf040-Paper-Conference.pdf)
- [YaRN - OpenReview](https://openreview.net/forum?id=wHBfxhZu1u)
- [Extending the RoPE - EleutherAI Blog](https://blog.eleuther.ai/yarn/)
- [GitHub - jquesnelle/yarn](https://github.com/jquesnelle/yarn)
