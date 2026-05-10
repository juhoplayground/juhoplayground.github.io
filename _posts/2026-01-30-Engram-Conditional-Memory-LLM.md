---
layout: post
title: Engram - 조건부 메모리 검색을 통한 LLM의 새로운 희소성 축
author: 'Juho'
date: 2026-01-30 00:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
   - [언어 모델링의 두 하위 과제](#언어-모델링의-두-하위-과제)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [Hashed N-gram 검색](#hashed-n-gram-검색)
   - [Context-aware gating과 다중 분기 통합](#context-aware-gating과-다중-분기-통합)
   - [System-algorithm co-design](#system-algorithm-co-design)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [Sparsity Allocation의 U자형 법칙](#sparsity-allocation의-u자형-법칙)
   - [무한 메모리 영역의 power law](#무한-메모리-영역의-power-law)
   - [대규모 사전학습 결과](#대규모-사전학습-결과)
   - [Long context와 RULER](#long-context와-ruler)
   - [Effective depth와 ablation](#effective-depth와-ablation)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Conditional Memory via Scalable Lookup: A New Axis of Sparsity for Large Language Models"는 트랜스포머에 정적 지식 lookup 기능을 부여해 conditional computation(MoE)과 보완적인 새로운 희소성 축을 도입한 연구다.
저자는 Xin Cheng, Wangding Zeng, Damai Dai, Qinyu Chen, Bingxuan Wang, Zhenda Xie 등 베이징대학교와 DeepSeek-AI 연구진이다.

논문 abstract의 핵심은 다음과 같다.
Engram은 n-gram embedding의 O(1) 결정적 lookup으로 정적 지식을 모델링하고, MoE 전용 budget의 일부를 conditional memory에 재할당하면 동일 파라미터·동일 FLOPs 조건에서도 더 우수한 성능이 나온다.
27B 규모로 확장한 Engram은 MoE-27B baseline 대비 MMLU +3.0, BBH +5.0, HumanEval +3.0의 일관된 향상을 보였다.

## 배경과 선행 연구

### 언어 모델링의 두 하위 과제

저자들은 언어 모델링이 두 가지 본질적으로 다른 하위 과제로 구성된다고 본다.
첫째, compositional reasoning은 깊은 계산이 필요한 동적 작업이다.
둘째, knowledge retrieval은 지역적·정적·고도로 정형화된 패턴 검색이다.

현행 트랜스포머는 두 번째 과제를 위한 native primitive가 없어서, attention과 FFN의 sequential depth로 정적 lookup을 시뮬레이션하며 비용을 낭비한다.
저자들은 conditional memory를 conditional computation의 보완적 sparsity 축으로 제안한다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Embedding scaling | OverEncoding(n-gram averaging), SCONE(고빈도 패턴 보조 인코더), SuperBPE, Byte Latent Transformer | iso-FLOPs 비교 부재 — 본 연구는 Sparsity Allocation 프레임으로 정량화 |
| MoE | GShard, Switch Transformer, BASE, DeepSeek-MoE | 동적 계산 — 본 연구는 정적 lookup 보완 |
| Parametric memory | PKM, PEER, Memory+, UltraMem | sparse key-value store — 본 연구는 deterministic n-gram + system co-design |
| Non-parametric memory | REALM, RETRO, PlugLM | 외부 검색 — 본 연구는 모델 내부 정적 임베딩 |

핵심 차별점은 두 가지다.
첫째, iso-parameter·iso-FLOPs 비교 프로토콜로 MoE 대비 우위를 엄격히 입증한다.
둘째, 메모리를 깊은 layer에 배치해 통신과 계산을 overlap시키는 algorithm-system co-design을 제시한다.

## 방법론

### Hashed N-gram 검색

먼저 입력을 토크나이저 압축 함수 P:V→V'로 정규화한다.
"Apple"과 " apple" 같이 의미적으로 동등한 토큰을 같은 canonical id로 묶어 128k 어휘를 약 23% 줄인다.

각 n-gram order n에 대해 K개의 hash head를 사용한다.
각 head k는 압축된 컨텍스트를 prime size M_{n,k}의 임베딩 테이블 E_{n,k} 인덱스로 매핑한다.
최종 메모리 벡터는 다음과 같이 모든 n-gram order와 hash head를 concat한 결과다.

```
e_t = || (n=2..N) || (k=1..K) e_{t,n,k}
```

이 검색은 결정적이며 routing overhead가 없는 O(1) 연산이다.

### Context-aware gating과 다중 분기 통합

검색된 임베딩은 컨텍스트 독립 prior이지만 hash collision과 polysemy 문제가 있다.
이를 해결하기 위해 attention 스타일 gating을 도입한다.

```
k_t = W_K e_t,    v_t = W_V e_t
α_t = σ( RMSNorm(h_t)^T RMSNorm(k_t) / √d )
```

α_t는 (0,1) 범위 스칼라 게이트로, 검색된 메모리가 현재 컨텍스트와 충돌하면 0에 가까워져 노이즈를 억제한다.

비선형성 강화를 위해 kernel size 4, dilation을 최대 n-gram order로 설정한 depthwise causal convolution이 더해진다.

```
Y = SiLU(Conv1D(RMSNorm(Ṽ))) + Ṽ
```

Manifold-Constrained Hyper-Connections(M=4) 다중 분기 구조에서는 sparse embedding table과 W_V를 모든 분기가 공유하고, W_K^(m)만 분기별로 분리해 분기별 gating 행동을 가능하게 한다.

### System-algorithm co-design

결정적 검색은 파라미터 저장과 연산 자원의 분리를 자연스럽게 지원한다.
학습 시에는 임베딩 테이블을 GPU 간 All-to-All 통신으로 sharding한다.
추론 시에는 결정적 인덱스를 활용해 host memory에서 PCIe로 임베딩을 비동기 prefetch하고, 직전 layer 연산과 통신을 overlap한다.

자연어 n-gram의 Zipf 분포 특성을 활용한 다단 캐시 계층(GPU HBM → Host DRAM → NVMe SSD)을 설계할 수 있다는 점도 강조된다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 학습 토큰 | 262B (대규모 사전학습), 100B (Engram 단일 스윕) |
| Tokenizer | DeepSeek-v3 (128k vocab) |
| Backbone | 30 블록, hidden 2560, MLA 32 heads, mHC M=4 |
| Engram 위치 | layer 2와 15 |
| 최대 n-gram order | 3 |
| Hash heads K | 8 |
| Embedding 차원 | 1280 |
| 임베딩 옵티마이저 | Adam, 5× learning rate, weight decay 0 |
| Long-context 확장 | YaRN (s=10, α=1, β=32, factor 0.707), 32k context |
| Long-context 추가 학습 | 5000 step, 30B 토큰 |

비교 대상 모델 4종을 동시에 학습한다.

| 모델 | 총 파라미터 | Activated | 구성 |
|------|--------------|------------|-------|
| Dense-4B | 4.1B | 3.8B | dense baseline |
| MoE-27B | 26.7B | — | 72 routed + 2 shared, top-k=6 |
| Engram-27B | 26.7B | — | 55 routed + 5.7B Engram (ρ≈74.3%) |
| Engram-40B | 39.5B | — | 55 routed + 18.5B Engram |

## 주요 결과

### Sparsity Allocation의 U자형 법칙

저자는 sparse 파라미터 budget을 MoE와 Engram에 어떤 비율로 나눌지를 ρ ∈ [0,1]로 정의한다.

```
P_MoE^(sparse) = ρ * P_sparse,    P_Engram = (1 - ρ) * P_sparse
```

두 compute 영역에서 ρ를 스윕한 결과 모두 U자형 곡선이 나타났다.

| 영역 | 전체 파라미터 | Activated | ρ=100% loss | ρ≈80% loss | 개선 |
|------|----------------|------------|--------------|--------------|------|
| 2×10^20 FLOPs | ≈5.7B | 568M | — | — | U자형 확인 |
| 6×10^20 FLOPs | ≈9.9B | 993M | 1.7248 | 1.7109 | Δ=0.0139 |

ρ의 최적값은 두 영역 모두 75–80% 부근으로 안정적이며, MoE 일변도(ρ=100%)는 항상 sub-optimal이라는 결론이 나온다.

### 무한 메모리 영역의 power law

3B MoE backbone(activated 568M)을 고정하고 Engram 슬롯 수를 2.58×10^5에서 1.0×10^7까지 스윕한 결과(추가 파라미터 약 13B), validation loss는 log-log 공간에서 strict power law를 따라 일관되게 감소했다.
OverEncoding baseline 대비 같은 메모리 budget에서 훨씬 큰 scaling 잠재력을 보였다.

### 대규모 사전학습 결과

Pile test loss (낮을수록 좋음).

| 모델 | Pile loss | Validation loss |
|------|-----------|------------------|
| Dense-4B | 2.091 | 1.768 |
| MoE-27B | 1.960 | 1.634 |
| Engram-27B | 1.950 | 1.622 |
| Engram-40B | 1.942 | 1.610 |

Knowledge & Reasoning 벤치마크.

| 벤치마크 | MoE-27B | Engram-27B | Δ | Engram-40B |
|----------|----------|--------------|----|--------------|
| MMLU 5-shot | 57.4 | 60.4 | +3.0 | 60.6 |
| CMMLU 5-shot | 57.9 | 61.9 | +4.0 | 63.4 |
| BBH 3-shot | 50.9 | 55.9 | +5.0 | 57.5 |
| ARC-Challenge 25-shot | 70.1 | 73.8 | +3.7 | 76.4 |
| HumanEval 0-shot | 37.8 | 40.8 | +3.0 | 38.4 |
| GSM8K 8-shot | 58.4 | 60.6 | +2.2 | 62.6 |
| MATH 4-shot | 28.3 | 30.7 | +2.4 | 30.6 |

Engram-27B는 iso-parameter·iso-FLOPs MoE-27B 대비 모든 벤치마크에서 일관되게 우위에 있고, 향상은 지식 집약 태스크에 국한되지 않는다.

### Long context와 RULER

32k 컨텍스트로 확장한 결과.

| 모델 | LongPPL | RULER MQ-NIAH | RULER VT |
|------|----------|------------------|------------|
| MoE-27B (50k step) | 4.38 | 84.2% | 77.0% |
| Engram-27B (46k, iso-loss) | 4.19 | 97.0% | 87.2% |
| Engram-27B (50k) | 4.14 | 97.0% | 89.0% |

iso-loss 시점(46k step)부터 이미 RULER MQ-NIAH 정확도가 84.2%에서 97.0%로 12.8%p 상승한다.
저자는 41k step 시점에서도 사전학습 FLOPs의 82%만 사용해 LongPPL이 baseline에 거의 근접한다고 보고한다.

### Effective depth와 ablation

LogitLens로 측정한 layer-wise KL divergence는 Engram 모델이 baseline보다 일관되게 작고, 특히 초기 블록에서 격차가 가장 크다.
CKA 분석은 Engram-27B의 layer 5 표현이 MoE baseline의 약 layer 12 표현과 가장 유사함을 보여, Engram이 더 이른 깊이에서 더 깊은 표현을 형성함을 정량적으로 입증한다.

Component ablation에서 multi-branch integration, context-aware gating, tokenizer compression 제거 시 모두 큰 성능 저하가 발생했고, depthwise convolution만 영향이 미미했다.

Sensitivity 분석으로 추론 시 Engram 출력을 억제하면 factual knowledge 태스크는 큰 폭으로 무너진다(TriviaQA 29% retention, MMLU 33–44%).
반면 reading comprehension은 안정적이다(C3 93%, DROP 81%, RACE 85–90%).

System efficiency 측정에서 100B 파라미터 Engram을 host memory에 offload한 inference throughput은 1.9–2.8% overhead로 측정되어 conservative baseline에서도 거의 손실이 없음을 보였다.

## 한계와 디스커션

저자가 명시한 한계는 직접 절로 분리되지 않지만 본문에서 다음을 인정한다.
첫째, 저장 인프라 가정에 의존한다. PCIe 대역폭과 host memory 용량이 클 때만 system 이득이 그대로 실현된다.
둘째, hash collision은 불가피하며 gating으로 보정하지만 해소하지는 못한다.
셋째, 결과는 DeepSeek-v3 토크나이저와 MLA 구조에 묶여 있어 다른 토크나이저·아키텍처로의 일반화는 추가 검증이 필요하다.
넷째, Engram이 효과를 내는 핵심은 "static, local, stereotyped" 패턴이며 동적 reasoning 자체를 강화하는 것은 아니다.

디스커션의 핵심 함의는 두 가지다.
첫째, conditional memory는 MoE의 보완재이지 대체재가 아니며 Sparsity Allocation 관점이 차세대 sparse 모델 설계의 출발점이 될 수 있다.
둘째, 메모리 lookup의 algorithm-system co-design을 통해 host memory와 SSD까지 활용하는 multi-tier 모델 시대를 현실화할 수 있다.

## 결론

Engram은 MoE 일변도 sparse 모델의 한계를 정량적으로 보여주며 conditional memory를 보완 축으로 도입한다.
U자형 Sparsity Allocation 법칙(ρ≈75–80%), 무한 메모리 power law, MMLU +3.0, BBH +5.0, RULER MQ-NIAH +12.8%p 같은 결과는 이 축의 실질적 가치를 입증한다.
임베딩 테이블을 host memory로 offload하는 system co-design으로 1.9–2.8% overhead만으로 100B급 메모리를 활용할 수 있다는 점은 모델 규모 확장의 새로운 경로를 제시한다.
오픈소스 코드와 함께 공개되어 sparse 모델 설계 패러다임에 직접적 영향을 줄 수 있는 연구다.

## Reference

- [Conditional Memory via Scalable Lookup (arXiv:2601.07372)](https://arxiv.org/abs/2601.07372/)
- [Engram GitHub](https://github.com/deepseek-ai/Engram/)
