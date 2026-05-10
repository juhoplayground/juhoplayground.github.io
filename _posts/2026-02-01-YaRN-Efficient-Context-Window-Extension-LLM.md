---
layout: post
title: YaRN - LLM 컨텍스트 윈도우를 효율적으로 확장하는 방법
author: 'Juho'
date: 2026-02-01 02:00:00 +0900
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
2. [배경과 선행 연구](#배경과-선행-연구)
   - [RoPE의 동작](#rope의-동작)
   - [Position Interpolation의 한계](#position-interpolation의-한계)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [NTK-aware interpolation](#ntk-aware-interpolation)
   - [NTK-by-parts와 ramp function](#ntk-by-parts와-ramp-function)
   - [Dynamic NTK](#dynamic-ntk)
   - [YaRN 최종 정의와 attention temperature](#yarn-최종-정의와-attention-temperature)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [Proof-pile 슬라이딩 윈도우 perplexity](#proof-pile-슬라이딩-윈도우-perplexity)
   - [GovReport long document](#govreport-long-document)
   - [Passkey Retrieval 128k](#passkey-retrieval-128k)
   - [표준 벤치마크 회귀](#표준-벤치마크-회귀)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"YaRN: Efficient Context Window Extension of Large Language Models"는 RoPE 기반 트랜스포머의 컨텍스트 윈도우를 적은 학습 비용으로 크게 확장하는 기법을 제시한 논문이다.
저자는 Bowen Peng(Nous Research), Jeffrey Quesnelle, Honglu Fan(EleutherAI / 제네바 대학교), Enrico Shippole이며 2023년 8월에 공개되었다.

논문 abstract의 핵심은 다음과 같다.
YaRN은 기존 컨텍스트 확장 기법보다 10배 적은 토큰과 2.5배 적은 학습 스텝으로 LLaMA 모델을 128k 컨텍스트까지 확장한다.
NTK-by-parts interpolation에 attention temperature 보정을 결합한 단순한 변환만으로 학습 길이를 넘어선 외삽까지 안정적으로 수행한다.

## 배경과 선행 연구

### RoPE의 동작

RoPE는 query와 key를 복소 임베딩 공간에서 위치에 따라 회전시킨다.

```
f_q(x_m, m) = e^{i m θ} W_q x_m
f_k(x_n, n) = e^{i n θ} W_k x_n
```

θ는 차원별 주파수 벡터이며 `θ_d = b^{-2d/|D|}` (b=10000)로 정의된다.
attention score는 `Re( x_m^* W_q^* W_k x_n e^{i θ (m-n)} )` 형태가 되어 상대 위치 (m−n)에만 의존한다.
각 차원의 wavelength는 `λ_d = 2π / θ_d = 2π b^{2d/|D|}`로 표현된다.

### Position Interpolation의 한계

Position Interpolation(PI)은 위치 인덱스를 단순 스케일링으로 압축한다.

```
f'_W(x_m, m, θ_d) = f_W(x_m, m * L/L', θ_d)
s = L'/L
```

PI는 모든 차원을 균일하게 압축하므로 high-frequency 차원의 정보까지 함께 잃는다.
저자들은 이 균일 압축이 토큰 간 거리 식별에 핵심적인 고주파 신호를 훼손한다고 진단한다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| 절대 위치 | sinusoidal, learnable absolute | 외삽 불가 |
| 상대 위치 | T5 Relative Bias, RoPE, XPos, ALiBi | RoPE는 외삽 약함 |
| Context 확장 | Position Interpolation(PI) | 균일 압축 — high-freq 손상 |
| NTK 계열 | NTK-aware (Reddit), Code Llama RoPE 변형 | 외삽 시 fine-tuning 충돌 — YaRN은 by-parts로 해결 |

본 논문은 PI와 NTK-aware의 약점을 모두 분석하고 NTK-by-parts와 attention temperature 보정을 결합한 통합 해법을 제시한다.

## 방법론

### NTK-aware interpolation

NTK-aware는 PI의 균일 압축 대신 base b를 변경해 저주파를 더 많이, 고주파를 덜 압축한다.

```
g(m) = m
h(θ_d) = b'^{-2d/|D|}
b' = b * s^{|D|/(|D|-2)}
```

이 방식은 fine-tuning 없이도 어느 정도 외삽이 가능하지만, 일부 외삽이 발생해 fine-tuning 시 학습이 불안정해지는 단점이 있다.

### NTK-by-parts와 ramp function

각 차원의 wavelength λ가 컨텍스트 L과 어떤 관계인지로 처리를 분기한다.
λ > L인 차원은 절대 위치 정보가 지배적이고, λ < L인 차원은 상대 위치 정보가 지배적이다.

비율 함수와 ramp.

```
r(d) = L / λ_d = L / (2π * b^{2d/|D|})

γ(r) = 0           if r < α
γ(r) = 1           if r > β
γ(r) = (r - α)/(β - α) otherwise
```

NTK-by-parts 정의.

```
g(m) = m
h(θ_d) = (1 - γ(r(d))) * (θ_d / s) + γ(r(d)) * θ_d
```

LLaMA 권장 하이퍼파라미터는 α=1, β=32다.
고주파 차원은 PI를 적용하지 않고, 저주파 차원은 PI를 완전히 적용해 선택적 보간을 수행한다.

### Dynamic NTK

추론 시점에 시퀀스 길이에 맞춰 s를 동적으로 갱신한다.

```
s = max(1, l' / L)
```

학습한 컨텍스트를 넘어서면서 발생하는 급격한 성능 저하를 막는다.
KV 캐시 사용 시에는 s가 변하므로 RoPE 적용 전 임베딩을 캐싱해야 한다는 주의사항이 따른다.

### YaRN 최종 정의와 attention temperature

YaRN은 NTK-by-parts에 attention temperature 보정을 추가한다.

```
softmax( q_m^T k_n / (t * √|D|) )
```

저자들은 LLaMA 7B/13B/33B/65B에 대해 fine-tuning 없이 여러 s에서 perplexity를 측정해 다음 식을 경험적으로 도출했다.

```
√(1/t) = 0.1 * ln(s) + 1
```

핵심 트릭은 복소 RoPE 임베딩을 `1/√t`로 스케일링하면 attention 코드 변경 없이 동등한 효과를 얻는다는 점이다.
temperature 스케일링은 데이터 샘플과 토큰 위치에 무관하게 perplexity에 균일한 영향을 준다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Base 모델 | Llama 2 7B, 13B (Mistral 7B v0.1 부록) |
| 학습 데이터 | PG19, 64k segment chunked |
| Scale factor | s=16, s=32 |
| 학습 step | 400 step (s=16) → 추가 200 step (s=32) |
| 학습 토큰 | base 사전학습의 0.1% 수준 |
| Optimizer | AdamW (β1=0.9, β2=0.95), LR 2e-5 |
| 평가 perplexity | sliding window S=256 |
| Passkey | 5자리 숫자 검색 |

YaRN s=32 모델은 64k에서 학습되고 128k까지 외삽되는 점이 핵심이다.
저자는 s=16에서 s=32로의 transfer learning이 매우 효율적이라 점진적 확장이 가능하다고 강조한다.

## 주요 결과

### Proof-pile 슬라이딩 윈도우 perplexity

| 모델 | 컨텍스트 | 8k | 32k | 65k | 98k | 128k |
|------|-----------|-----|-----|-----|-----|------|
| YaRN 7B (s=16) | 64k | 3.51 | 2.65 | 2.42 | 큰 폭발 | 큰 폭발 |
| YaRN 7B (s=32) | 128k | 3.56 | 2.70 | 2.45 | 2.36 | 2.37 |
| YaRN 13B (s=32) | 128k | 3.29 | 2.53 | 2.31 | 2.23 | 2.24 |

s=16 모델은 64k까지만 안정적이고 그 이상에서는 perplexity가 10 이상으로 폭증하는 반면, s=32 모델은 128k에서도 안정적인 perplexity를 유지한다.
Code Llama(NTK-aware, 100k 학습) 7B는 98k에서 2.54, 128k에서 2.71을 기록한 반면 YaRN 7B(s=32)는 같은 128k에서 2.37로 더 우수하다.

### GovReport long document

50개 문서 32k 윈도우 평가.

| 모델 | GovReport perplexity |
|------|------------------------|
| YaRN 7B (s=16) | 3.59 |
| YaRN 7B (s=32) | 3.64 |
| YaRN 13B (s=32) | 3.39 |

13B 변형이 가장 낮은 perplexity를 보였다.

### Passkey Retrieval 128k

긴 컨텍스트 안에 숨겨진 5자리 숫자를 검색하는 태스크.

| 모델 | 컨텍스트 | 정확도 |
|------|-----------|---------|
| YaRN 7B (s=32) | 128k | 99.4% |
| YaRN 13B (s=32) | 128k | 99.4% |

8k에서 128k까지 다양한 컨텍스트에서 99% 이상의 retrieval 정확도를 유지한다.
저자는 perplexity가 비슷해도 passkey 정확도는 학습량에 더 민감하다는 점을 지적하며, perplexity 단독으로는 충분한 지표가 아님을 강조한다.

### 표준 벤치마크 회귀

Hugging Face Open LLM Leaderboard 기준.

| 모델 | ARC-c | HellaSwag | MMLU | TruthfulQA |
|------|--------|--------------|--------|--------------|
| Llama 2 7B 원본 | 53.1 | 77.8 | 43.8 | 39.0 |
| YaRN 7B (s=32) | 52.1 | 78.4 | 41.7 | 37.3 |
| Llama 2 13B 원본 | 59.4 | 82.1 | 55.8 | 37.4 |
| YaRN 13B (s=32) | 58.0 | 82.2 | 51.9 | 37.3 |

대부분 항목은 거의 그대로 유지되며 MMLU에서만 약 −2~−4점 회귀가 관찰된다.
s=16에서 s=32로 옮겨갈 때 평균 0.49% 하락에 그쳐 점진적 확장의 안정성도 보였다.

## 한계와 디스커션

저자가 명시한 한계와 주의사항은 다음과 같다.
첫째, attention temperature 공식 `√(1/t) = 0.1 ln(s) + 1`은 LLaMA 계열에서 경험적으로 도출되었으며 다른 아키텍처로의 일반화는 보장되지 않는다.
둘째, NTK-by-parts의 α=1, β=32는 LLaMA 권장값이며 모델별로 별도 튜닝이 필요할 수 있다.
셋째, perplexity가 비슷해도 passkey 같은 retrieval 정확도는 학습량에 더 민감하므로 perplexity 단독 평가는 위험하다.
넷째, 비교 baseline(PI, NTK-aware Code Llama)은 더 많은 데이터로 학습되었으므로 절대 비교가 아닌 효율 비교 관점에서 해석해야 한다.

디스커션의 핵심은 두 가지다.
첫째, NTK-by-parts와 attention temperature 보정이라는 두 단순 변환만으로 fine-tuning과 외삽 양쪽에서 안정성을 확보할 수 있다.
둘째, Dynamic-YaRN을 통해 fine-tuning 없이도 즉시 사용 가능한 추론 시점 적용도 가능하다.

## 결론

YaRN은 LLaMA 2 7B와 13B를 64k 학습으로 128k까지 안정적으로 확장했고, Proof-pile 128k에서 7B 2.37 / 13B 2.24의 perplexity를 달성했다.
Passkey Retrieval 128k에서 99.4% 정확도를 보여 long-context 활용이 실제로 가능함을 입증했다.
표준 벤치마크 회귀가 미미한 점, base 사전학습의 0.1% 수준의 데이터로 학습 가능한 점, Flash Attention 2 호환과 zero inference overhead를 갖춘 점이 실용적 도입의 큰 강점이다.
이후 다수의 오픈소스 모델이 long-context 확장의 사실상 표준으로 채택한 기법이다.

## Reference

- [YaRN: Efficient Context Window Extension of Large Language Models (arXiv:2309.00071)](https://arxiv.org/abs/2309.00071/)
- [YaRN GitHub](https://github.com/jquesnelle/yarn/)
