---
layout: post
title: "GOAT - 학습 가능한 사전 분포로 어텐션 메커니즘을 재설계하다"
author: 'Juho'
date: 2026-05-09 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Benchmark]
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
   - [표준 어텐션의 암묵적 균등 사전 가정](#표준-어텐션의-암묵적-균등-사전-가정)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [Entropic Optimal Transport 프레임워크](#entropic-optimal-transport-프레임워크)
   - [GOAT 파라미터화 세 구성 요소](#goat-파라미터화-세-구성-요소)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [언어 모델 perplexity와 길이 외삽](#언어-모델-perplexity와-길이-외삽)
   - [Passkey와 Needle in a Haystack](#passkey와-needle-in-a-haystack)
   - [DNA 모델링과 비전](#dna-모델링과-비전)
   - [어텐션 싱크 이론적 분석](#어텐션-싱크-이론적-분석)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"You Need Better Attention Priors"는 트랜스포머 어텐션 메커니즘을 Entropic Optimal Transport(EOT, 엔트로픽 최적 수송) 이론의 관점에서 재해석한 논문이다.
저자는 Elon Litman과 Gabe Guo이며, 2026년 1월 21일 arXiv에 제출되었다.

논문 abstract는 다음과 같이 핵심을 요약한다.
표준 어텐션은 균등 사전 분포(uniform prior)로 정규화된 수송 문제에 해당하며, 이 사전 분포를 학습 가능한 분포로 일반화하면 어텐션의 안정성과 표현력을 동시에 끌어올릴 수 있다.
GOAT(Generalized Optimal transport Attention with Trainable priors)는 이 관점을 구체적인 파라미터화로 구현하면서, FlashAttention 같은 가속 커널과 호환되는 SDPA(Scaled Dot-Product Attention) 형태를 유지한다.

## 배경과 선행 연구

### 표준 어텐션의 암묵적 균등 사전 가정

저자들은 표준 scaled dot-product attention이 원리적 토대 없이 휴리스틱(softmax를 부드러운 argmax 근사로 사용)에 의존한다고 진단한다.
효율성, 표현력, 행동 안정성 세 축에서 개선 시도가 이어져 왔지만, 그 본질에 대한 통일된 설명은 부족했다.

핵심 통찰은 Shannon 엔트로피 정규화 항이 사실상 출력 분포 p와 균등 분포 사이의 음의 KL 발산과 동일하다는 것이다.
즉 표준 어텐션은 모든 토큰에 동일한 prior weight를 부여하는 균등 사전 가정 위에서 작동하고 있으며, 이 사전 분포를 학습 가능한 분포로 바꾸면 안정성과 표현력을 동시에 얻을 수 있다.

### Related Work 정리

| 카테고리 | 주요 선행 연구 | 본 연구와의 차이 |
|----------|----------------|------------------|
| 효율성 | Vaswani 2017 (Transformer), Katharopoulos 2020 (linear attention), Dao 2023 (FlashAttention-2) | 효율성 유지하되 사전 분포 자체를 학습 |
| 표현력 | Shaw 2018 (relative PE), Su 2021 (RoPE), Press 2022 (ALiBi), Transformer-XL | 위치 인코딩이 아닌 사전 분포로 통합 표현 |
| 안정성 | Xiao 2024 (attention sinks 분석) | 싱크 현상을 EOT 관점에서 이론적으로 해결 |
| 최적 수송 | Cuturi 2013 (Sinkhorn distances), Litman 2025 (attention as EOT) | EOT 관점을 학습 가능한 사전 분포로 확장 |

## 방법론

### Entropic Optimal Transport 프레임워크

논문은 어텐션을 두 분포 사이의 엔트로픽 최적 수송 문제로 재정의하고, Proposition 3.1에서 사전 분포가 들어간 어텐션 식을 도출한다.

```
p*_j = softmax(s_j / τ + log π_j)
```

여기서 π는 균등 가정을 대체하는 임의의 사전 분포다.
σ가 균등이면 표준 softmax 어텐션이 되고, π가 학습 가능하면 GOAT가 된다.

이 식의 이론적 의의는 두 가지다.
첫째, 표준 어텐션은 GOAT의 특수 사례로 환원된다.
둘째, log π_j 항이 score s_j와 동일한 형태로 더해지므로, SDPA의 행렬 곱 구조를 깨지 않고 사전 분포를 주입할 수 있다.

### GOAT 파라미터화 세 구성 요소

GOAT는 사전 분포 π를 세 가지 구성 요소로 분해해 학습한다.

첫째, Spectral Relative Prior로, 상대 위치 i−j에 대한 K^rel을 Fourier feature 합으로 표현한다.

```
K^rel_ij = Σ_r [α_r cos(ω_r (i−j)) + β_r sin(ω_r (i−j))]
```

α_r, β_r, ω_r은 모두 학습 파라미터로, 데이터 분포에 맞는 위치 사전을 표현한다.

둘째, Sink Component로, query에 무관하게 특정 key 위치에 bias를 부여하는 항이다.

```
⟨q_sink,i, k_sink,j⟩ = u(j)  (모든 i에 대해)
```

u(j)는 학습 가능한 decay 함수와 MLP의 합으로 파라미터화된다.

셋째, Unified Implementation으로, 위 두 항을 합쳐 합성된 query/key 벡터를 만들어 표준 SDPA의 한 번의 호출 안에서 모두 처리한다.

```
⟨q'_i, k'_j⟩ / √d_h = s_ij + K_ij
```

이때 content score는 1/√d_c로 스케일링되지만, 사전 분포 항 K_ij는 스케일링 없이 유효 temperature 1로 들어간다(식 20–24). 이 트릭이 FlashAttention 호환성과 사전 분포의 명시적 영향력을 동시에 보장한다.

## 실험 셋업

| 영역 | 데이터셋 | 비교 베이스라인 |
|------|----------|------------------|
| 언어 | C4 (4B 토큰) | RoPE, ALiBi, sinusoidal absolute PE, position interpolation |
| 비전 | ImageNet-1k | RoPE, 위치 인코딩 없음 |
| DNA | Human Reference Genome | RoPE |
| 합성 | Passkey Retrieval, Needle-in-a-Haystack, Copy-mixture | 위치 인코딩 변형 |

언어 모델은 125M 파라미터 규모로 학습 컨텍스트 길이 L_train=2048에서 학습되었고, 비전 모델은 224×224 해상도로 학습된 ViT 구조다.

## 주요 결과

### 언어 모델 perplexity와 길이 외삽

C4에서 in-distribution perplexity 기준 GOAT는 ALiBi 대비 1.55 포인트 낮은 perplexity를 보였다.
길이 외삽 실험에서는 학습 길이의 16배까지 안정적으로 동작했으며, RoPE는 학습 길이를 넘어서면 catastrophic하게 성능이 무너졌다.

### Passkey와 Needle in a Haystack

Passkey Retrieval에서 컨텍스트가 학습 길이를 한참 넘어서도 GOAT는 RoPE 대비 상당히 높은 정확도를 유지했다.
Needle-in-a-Haystack에서는 다양한 깊이와 길이 조합 전반에서 학습된 사전 분포가 거의 완벽한 retrieval을 유지했다.

### DNA 모델링과 비전

DNA 시퀀스 모델링에서 GOAT는 RoPE보다 낮은 validation NLL을 기록했고, peak CUDA 메모리는 2.86GB에서 1.83GB로 약 36% 감소했다.
GC 비율 추적 Pearson 상관계수는 0.466으로 RoPE의 0.320을 상회했다.
비전 ViT에서는 학습 시 보지 못한 224×224 이상의 해상도에 대해서도 정확도를 유의미하게 유지하는 zero-shot 해상도 외삽 능력을 보였다.

### 어텐션 싱크 이론적 분석

논문은 두 가지 정리로 어텐션 싱크 현상을 설명한다.

Theorem 5.1 (Collapse to Prior): content signal ω_i가 0에 가까울 때 posterior는 prior에 점별로 수렴한다.

Theorem 5.4 (Context Sensitivity Bounds): 컨텍스트 길이 L에 대한 sensitivity Ψ는 균등 prior에서 1로 수렴(Ψ_uni → 1)하는 반면, peaked prior(GOAT)에서는 (L−1)/(exp(δ)+L−1)로 상한이 잡히며 prior margin δ에 지수적으로 노이즈가 억제된다.

Figure 2c에 따르면 신호 ω가 커질수록 GOAT의 sink mass는 prior 지배(약 1)에서 content 지배(약 0)로 부드럽게 이동한다.

## 한계와 디스커션

저자는 명시적 한계 절을 길게 두지는 않지만, GOAT의 prior 파라미터화는 SDPA 호환, 평행이동 등변(equivariance), 안정성이라는 제약 아래 최적임을 강조한다.
즉 이 제약을 깨면 더 나은 표현이 가능할 수 있으나, 본 논문은 호환성을 유지하는 범위 내 최선의 설계를 제시한 것이다.

디스커션의 핵심은 표준 self-attention의 취약성이 균등 사전 가정에서 비롯된다는 점, 그리고 GOAT가 표현력과 안정성 사이의 trade-off를 해소한다는 점이다.
실제 구현 시 학습 가능한 사전 분포가 FlashAttention 커널의 메모리 접근 패턴 가정과 어떻게 결합되는지는 논문이 제시한 unified parameterization을 따라야 보장되며, 그렇지 않으면 효율성 이득이 사라질 수 있다는 실용적 주의사항이 따른다.

## 결론

GOAT는 어텐션을 Entropic Optimal Transport 문제로 재해석하고, 표준 어텐션의 암묵적 균등 사전 분포를 학습 가능한 spectral 사전과 sink 항으로 대체한 기여를 제시한다.
C4 perplexity 1.55 포인트 개선, 16배 길이 외삽, DNA 모델링에서 36% 메모리 절감, 비전 zero-shot 해상도 외삽 등 다양한 도메인에서 일관된 향상을 보였다.
"더 나은 사전 분포"라는 단순한 메시지가 어텐션 싱크 해결, 길이 일반화, 효율 커널 호환을 동시에 달성할 수 있다는 점을 이론과 실험으로 함께 입증한 것이 이 논문의 의의다.

## Reference

- [You Need Better Attention Priors (arXiv:2601.15380)](https://arxiv.org/abs/2601.15380/)
