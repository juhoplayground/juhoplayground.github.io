---
layout: post
title: "그래프 표현 학습을 위한 Joint Embedding 예측적 자기지도 프레임워크"
author: 'Juho'
date: 2026-03-22 00:00:00 +0900
categories: [AI]
tags: [AI]
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
   - [그래프 SSL의 한계](#그래프-ssl의-한계)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [Joint Predictive Embedding 구조](#joint-predictive-embedding-구조)
   - [GMM Bayesian pseudo-label](#gmm-bayesian-pseudo-label)
   - [전체 손실](#전체-손실)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [Supervised node classification](#supervised-node-classification)
   - [SSL fine-tuning 비교](#ssl-fine-tuning-비교)
   - [Robustness ablation](#robustness-ablation)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Leveraging Joint Predictive Embedding and Bayesian Inference in Graph Self Supervised Learning"은 contrastive 손실과 negative sampling 없이 그래프 자기지도 학습을 수행하는 프레임워크를 제안한 논문이다.
저자는 Srinitish Srinivasan, Omkumar CU(Vellore Institute of Technology, India)이다.

논문 abstract의 핵심은 다음과 같다.
JEPA-스타일 non-contrastive joint embedding 예측 구조에 단일 컨텍스트당 다중 target embedding과 GMM 기반 pseudo-label scoring을 결합한다.
이 결합으로 6개 표준 그래프 벤치마크에서 contrastive 기법 또는 복잡한 디코더 없이도 일관된 성능 향상을 보인다.

## 배경과 선행 연구

### 그래프 SSL의 한계

기존 그래프 SSL은 두 가지 흐름으로 나뉜다.
첫째, contrastive 기반(GraphCL, MVGRL, GRACE 등)은 negative sampling과 mutual information 추정에 의존해 계산이 비싸고 collapse 위험이 있다.
둘째, generative 기반(GraphMAE 등)은 마스킹 후 복원으로 학습하지만 복원기가 복잡하다.

본 논문은 vision의 JEPA(Joint-Embedding Predictive Architecture)를 그래프에 적용하면서 두 한계를 모두 우회한다.

### Related Work 정리

| 카테고리 | 주요 선행 연구 | 본 연구와의 차이 |
|----------|----------------|------------------|
| Contrastive | DGI, MVGRL, GRACE, S3-CL | negative 필요 — 본 연구는 non-contrastive |
| BYOL 계열 | BGRL | momentum target — 다중 target은 본 연구 |
| Decoupled | CCA-SSG | 통계 분리 — 본 연구는 GMM Bayesian 결합 |
| Generative | GraphMAE | 디코더 무거움 — 본 연구는 latent 예측만 |
| Multi-task | ParetoGNN | 다중 obj — 본 연구는 단일 latent loss + GMM |

## 방법론

### Joint Predictive Embedding 구조

context encoder는 3-layer GCN으로 hidden 차원을 128 → 256 → 512로 늘린다(ReLU, Tanh).
target encoder는 동일 아키텍처지만 momentum 갱신을 사용한다.

```
Θ_s_target = m * Θ_{s-1}_target + (1 - m) * Θ_s_context
```

predictor는 2-layer GCN(512차원, Tanh)이다.
손실은 latent 공간에서 예측 임베딩과 target 임베딩의 L2 차이다.

```
L_J = (1/T) * Σ_k || H'_k - H^t_k ||^2
```

단일 context에 대해 다중 target을 두는 설계가 representation collapse를 막는다.

### GMM Bayesian pseudo-label

GMM 기반 pseudo-label은 EM으로 학습된다.
posterior γ(z_nk)가 cluster assignment를 표현하고, mixing coefficient π_k, mean μ_k, covariance Σ_k가 갱신된다.
GMM pseudo-label과 K-Means clustering 결과의 차이를 smooth L1 손실 L_G로 정의한다.

### 전체 손실

```
L = L_J + L_G
```

L_J가 latent 일관성을, L_G가 의미 클러스터 인식을 강제한다.
저자는 이 결합이 단순 BYOL보다 클래스 구조를 더 잘 형성한다고 주장한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| Citation 데이터셋 | Cora (2,708), Citeseer (3,312), Pubmed (19,717) |
| Product 데이터셋 | Amazon Photos, Amazon Computers |
| Collaboration | Coauthor CS |
| 평가 | linear evaluation, 단일 GCN classification layer |
| 반복 | 10 runs, mean ± std |
| Momentum m | 0.9 (best) |

## 주요 결과

### Supervised node classification

| 데이터셋 | 본 연구 | GCN baseline |
|----------|---------|----------------|
| Cora | 89.8 ± 0.9 | 81.5 ± 1.3 |
| Citeseer | 77.0 ± 0.9 | 73.1 ± 0.1 (simplified GCN) |
| Amazon Photos | 94.5 ± 0.5 | 91.4 ± 1.3 (GraphSAGE mean) |
| Coauthor CS | 93.6 ± 0.4 | 91.5 ± 0.3 (simplified GCN) |

### SSL fine-tuning 비교

10개 SSL baseline(DGI, MVGRL, GRACE, BGRL, GraphMAE 등)과의 비교.

| 데이터셋 | 본 연구 | 최고 baseline |
|----------|---------|------------------|
| Cora | 89.8 ± 0.9 | S3-CL 84.5 ± 0.4 |
| Citeseer | 77.0 ± 0.9 | S3-CL 74.6 ± 0.4 |
| Amazon Photos | 94.5 ± 0.5 | CCA-SSG 93.1 ± 0.1 |

ParetoGNN, BGRL과의 직접 비교.

| 데이터셋 | 본 연구 | ParetoGNN | BGRL |
|----------|---------|-------------|--------|
| Photos | 94.5 ± 0.5 | 93.8 ± 0.3 | — |
| Computers | 88.0 ± 0.6 | 90.7 ± 0.2 | — |
| CS | 93.6 ± 0.4 | — | 93.3 ± 0.1 |

대부분 데이터셋에서 SOTA 또는 SOTA에 근접한 성능을 보였고 Computers에서만 ParetoGNN에 뒤졌다.

### Robustness ablation

GMM constraint 유무 비교.

| 데이터셋 | constraint 없음 | constraint 있음 |
|----------|-------------------|-------------------|
| Cora | 89.0 | 89.8 |
| Citeseer | 74.1 | 77.0 |

평균 0.8~2.9%p 향상이다.

Test-time feature 변형(Amazon Computers 40% node corruption) 시 −4.7% 감소만 발생해 강한 일반화 능력을 보였다.

momentum 스윕에서 m=0.9가 최적이며, m이 1에 가까워지면 학습 속도가 떨어진다.

## 한계와 디스커션

저자가 본문에서 인정하는 한계와 주의사항은 다음과 같다.
첫째, 그래프 분류(graph classification) 결과는 본문에서 별도로 다루지 않으며 주요 평가가 노드 분류에 집중된다.
둘째, GMM은 cluster 수 K를 사전에 지정해야 하며 적절한 K 선택이 성능에 영향을 준다.
셋째, momentum m=0.9 부근에서만 최적이며 모델·데이터셋별 튜닝이 필요하다.
넷째, ParetoGNN이 일부 데이터셋(Computers)에서 더 우수한 결과를 보여 단일 best 기법이라기보다 그래프 SSL의 또 하나의 유효 접근으로 봐야 한다.

디스커션의 핵심은 두 가지다.
첫째, vision의 JEPA 아이디어가 그래프에서도 통한다는 점, 그리고 latent 예측만으로 contrastive 부담을 없앨 수 있다는 점이다.
둘째, GMM Bayesian pseudo-label은 SSL이 놓치기 쉬운 의미 클러스터 구조를 명시적으로 부여하는 단순하지만 효과적인 보강이다.

## 결론

본 연구는 그래프 SSL에 JEPA 스타일 latent 예측 구조와 GMM Bayesian pseudo-label을 결합해 contrastive 부담 없이도 6개 벤치마크 전반에서 일관된 향상을 입증했다.
Cora 89.8, Photos 94.5 등 SOTA에 근접하거나 능가하는 성능과 40% node corruption에서도 -4.7%만 떨어지는 강건성이 핵심 결과다.
간단한 구조와 적은 계산 비용으로 도입 가능한 plug-friendly 그래프 SSL 기법이라는 점이 실무적 가치다.

## Reference

- [Predict, Cluster, Refine: A Joint Embedding Predictive Self-Supervised Framework (arXiv:2502.01684)](https://arxiv.org/abs/2502.01684/)
- [JPEB-GSSL GitHub](https://github.com/Deceptrax123/JPEB-GSSL/)
