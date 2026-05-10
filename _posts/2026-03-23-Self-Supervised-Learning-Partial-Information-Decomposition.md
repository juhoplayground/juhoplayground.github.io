---
layout: post
title: "Partial Information Decomposition 관점에서 자기지도 학습 재고"
author: 'Juho'
date: 2026-03-23 00:00:00 +0900
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
   - [SSL과 mutual information 논쟁](#ssl과-mutual-information-논쟁)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [PID 분해 프레임](#pid-분해-프레임)
   - [Progressive self-supervision 3단계](#progressive-self-supervision-3단계)
   - [Information Bottleneck 정합](#information-bottleneck-정합)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [CIFAR-10 / CIFAR-100](#cifar-10--cifar-100)
   - [ImageNet과 Tiny ImageNet](#imagenet과-tiny-imagenet)
   - [Transfer learning](#transfer-learning)
   - [Ablation](#ablation)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"Rethinking Self-Supervised Learning Within the Framework of Partial Information Decomposition"은 SSL을 mutual information 단일 지표가 아닌 PID(Partial Information Decomposition)로 재해석한 논문이다.
저자는 Salman Mohamadi, Gianfranco Doretto, Donald A. Adjeroh(West Virginia University)이며 2024년 12월 arXiv에 공개되었다.

논문 abstract의 핵심은 다음과 같다.
SSL이 두 augmentation view 사이 MI를 최대화해야 하는지 최소화해야 하는지에 대한 기존 논쟁을 PID 분해로 재구성한다.
PID는 두 view 표현과 target 표현 사이의 joint MI를 unique·redundant·synergistic 세 성분으로 분해한다.
저자는 sample-level과 cluster-level supervision을 점진적으로 결합한 progressive self-supervision으로 모든 PID 성분을 활용해 4개 baseline(SimCLR, BYOL, W-MSE, Barlow Twins)에 일관된 향상을 보인다.

## 배경과 선행 연구

### SSL과 mutual information 논쟁

SSL 커뮤니티 안에는 두 view 표현 사이 MI를 어떻게 다뤄야 하는지에 대한 상반된 입장이 있다.
한쪽은 contrastive 흐름처럼 MI 최대화를 주장하고, 다른 쪽은 Info-Min 원리로 MI 감소(단 task-relevant 정보 보존)를 옹호한다.

저자들은 이 논쟁이 잘못 설정된 질문이라고 본다.
중요한 것은 두 view 표현과 task target T 사이의 joint MI 구조를 어떻게 분해하느냐이고, 이 분해가 PID로 가능하다는 것이다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Contrastive | SimCLR, MoCo | view 간 MI 최대화 — PID 일부만 활용 |
| Negative-free | BYOL, SimSiam | implicit collapse 방지 — PID 관점 부재 |
| Decorrelation | Barlow Twins, VICReg, W-MSE | 차원 통계 — PID 관점 부재 |
| Info-Min | Tian 2020 | MI 최소화 — PID 분해 부재 |
| PID 이론 | Williams & Beer 2010, Bertschinger 2014 | 이론 정립 — 본 연구가 SSL에 응용 |

## 방법론

### PID 분해 프레임

joint MI를 다음과 같이 분해한다.

```
I(S_1, S_2 ; T) = R(T; S_1, S_2)
                + Sy(T; S_1, S_2)
                + U(T; S_1)
                + U(T; S_2)
```

R은 두 source의 공통 정보(redundant), Sy는 두 source의 비선형 결합으로만 얻어지는 synergistic 정보, U는 각 source의 고유 정보다.
저자는 기존 contrastive SSL이 redundant와 synergistic 성분만 활용하고 unique 정보가 누락된다고 진단한다.

### Progressive self-supervision 3단계

Phase 1 — 100 epoch 동안 baseline SSL 손실 L_ssl로 초기 학습한다.

Phase 2 — 학습된 표현에 K-means++ 클러스터링을 적용해 hard pseudo-label을 생성한다(K는 데이터셋의 알려진 클래스 수).

Phase 3 — 다음 결합 손실로 900 epoch를 진행한다.

```
L = L_ssl(z_1, z_2) + α * { L_ps(z_1, ŷ) + L_ps(z_2, ŷ) }
```

α는 10^-5에서 10^-1로 점진적으로 증가하며, pseudo-label은 100 epoch마다 갱신된다.

이 결합이 PID 세 성분을 모두 활용한다.
sample 단위 invariance가 local 클러스터링을 유도하고, cluster 단위 invariance가 global 구조를 부여한다.

### Information Bottleneck 정합

저자는 PID 결합 손실이 IB 원리와 두 수준에서 정합된다고 본다.
intra-alignment에서는 SSL 손실 자체가 augmentation 효과 무화(I(X; T_θ) 감소)와 target 학습(I(T_θ; Y) 증가)의 균형을 이룬다.
inter-alignment에서는 결합 손실이 sample-level과 cluster-level 연관 사이의 bottleneck 표현을 권장한다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 데이터셋 | CIFAR-10, CIFAR-100, ImageNet, Tiny ImageNet |
| Baseline SSL | SimCLR, BYOL, W-MSE2, Barlow Twins |
| 평가 | Linear evaluation + K-NN evaluation |
| Phase 1 epoch | 100 |
| Phase 3 epoch | 900 |
| α 범위 | 10^-5 → 10^-1 |
| pseudo-label 갱신 | 100 epoch마다 |
| K (클러스터 수) | 데이터셋 클래스 수 |
| Transfer | ResNet50 ImageNet → CIFAR-10/100 |

## 주요 결과

### CIFAR-10 / CIFAR-100

CIFAR-10 linear evaluation.

| 방법 | Baseline | +PID |
|------|-----------|--------|
| BYOL | 91.81 | 93.12 (+1.31) |
| SimCLR | 91.93 | 93.27 (+1.34) |
| W-MSE2 | 90.16 | 92.03 (+1.87) |
| Barlow Twins | 92.55 | 93.97 (+1.42) |

CIFAR-10 K-NN evaluation(supervised 학습 없음).

| 방법 | Baseline | +PID |
|------|-----------|--------|
| BYOL | 89.39 | 91.77 (+2.38) |
| SimCLR | 88.63 | 91.03 (+2.40) |

CIFAR-100 linear average +1.59%, K-NN average +1.92%였다.

### ImageNet과 Tiny ImageNet

ImageNet linear average +1.55%, K-NN average +2.97%였다.
Tiny ImageNet linear average +1.38%, K-NN average +2.51%였다.

K-NN 평가에서 향상 폭이 더 크다는 점이 흥미롭다.
저자는 이는 PID 분해가 표현 공간의 의미 구조를 더 잘 형성한다는 증거로 해석한다.

### Transfer learning

ResNet50을 ImageNet에서 사전학습한 뒤 CIFAR로 transfer.

| 데이터셋 | 평균 향상 |
|----------|------------|
| CIFAR-10 | +1.18% |
| CIFAR-100 | +1.31% |

### Ablation

cluster 수 K 영향(CIFAR-100 기준).

| K | 향상 |
|----|--------|
| 50 | +1.18% |
| 100 (정답) | +1.59% |
| 150 | +0.74% |

K가 작을수록 더 compact한 표현을 형성하지만, 정답 K가 가장 좋은 성능을 보였다.

non-progressive double supervision(800 epoch 사전학습 + 200 epoch 고정 pseudo-label)은 +0.98%로 progressive 갱신(+1.59%) 대비 떨어졌다.
이는 점진적 α 증가와 주기적 pseudo-label 갱신이 핵심임을 보여준다.

epoch 확장 영향(K-NN).

| Epoch | 향상 |
|--------|--------|
| 1000 | +1.92% |
| 1200 | +2.23% |
| 1500 | +2.60% |

학습이 길수록 향상이 누적되며, 이는 PID 결합 손실이 representation collapse 없이 점점 더 정교한 구조를 형성함을 시사한다.

## 한계와 디스커션

저자가 명시한 한계는 다음과 같다.
첫째, pseudo-label 생성에 분류 task의 클래스 구조를 가정한다. 회귀나 ranking task에는 직접 적용이 어렵다.
둘째, PID 분해는 두 source 변수에 대한 것이다. 세 개 이상 변수로의 확장은 향후 과제다.
셋째, augmentation 자체가 PID 성분과 어떻게 정렬되는지에 대한 명시적 분석은 본 논문에서 수행하지 않았다.

디스커션의 핵심은 두 가지다.
첫째, MI를 늘리거나 줄이는 이분법보다 PID 세 성분(unique, redundant, synergistic)을 어떻게 효율적으로 활용할지 묻는 것이 더 생산적이다.
둘째, contrastive SSL에서 빠진 unique 정보 성분이 progressive self-supervision으로 살아나는 것이 향상의 핵심 메커니즘이다.

## 결론

PID 관점은 SSL의 mutual information 논쟁을 해소하는 새로운 정보이론적 기반을 제공한다.
SimCLR, BYOL, W-MSE2, Barlow Twins 4개 baseline에 progressive self-supervision을 추가하면 CIFAR-10 +1.31~1.87%, CIFAR-100 +1.59%, ImageNet +1.55%, Tiny ImageNet +1.38%의 일관된 향상이 발생한다.
K-NN 평가에서 향상 폭이 더 크고 epoch가 길수록 누적된다는 점은 PID 결합 손실이 표현 공간의 의미 구조를 점진적으로 정교화한다는 것을 보여준다.
SSL을 새로운 정보이론적 관점에서 재해석한 의미 있는 기여다.

## Reference

- [Rethinking Self-Supervised Learning Within the Framework of Partial Information Decomposition (arXiv:2412.02121)](https://arxiv.org/abs/2412.02121/)
