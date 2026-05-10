---
layout: post
title: "GLF - 자기지도 대조 학습을 위한 일반화된 학습 프레임워크"
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
   - [SimCLR 손실의 두 항 분해](#simclr-손실의-두-항-분해)
   - [Related Work 정리](#related-work-정리)
3. [방법론](#방법론)
   - [GLF 일반 목적함수](#glf-일반-목적함수)
   - [Distribution Calibration Module (DCM)](#distribution-calibration-module-dcm)
   - [Local Preserving Module (LPM)](#local-preserving-module-lpm)
   - [전체 ADC 손실](#전체-adc-손실)
4. [실험 셋업](#실험-셋업)
5. [주요 결과](#주요-결과)
   - [Linear evaluation](#linear-evaluation)
   - [Semi-supervised와 Transfer](#semi-supervised와-transfer)
   - [Object detection과 segmentation](#object-detection과-segmentation)
   - [Ablation](#ablation)
6. [한계와 디스커션](#한계와-디스커션)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

"A Generalized Learning Framework for Self-Supervised Contrastive Learning"은 SSCL(Self-Supervised Contrastive Learning) 기법을 align 항과 constraining 항으로 분해해 통합 프레임 GLF로 정리하고, plug-and-play 방식으로 도입 가능한 Adaptive Distribution Calibration(ADC)을 제안한 논문이다.
저자는 Lingyu Si, Jingyao Wang, Wenwen Qiang(교신저자)이다.

논문 abstract의 핵심은 다음과 같다.
GLF는 SSCL의 표현 학습 목적을 정렬과 제약의 두 부분으로 분해해 BYOL, Barlow Twins, SwAV 등을 통합 관점에서 설명한다.
ADC는 두 모듈 DCM과 LPM의 결합으로 클래스 내 응집과 클래스 간 분리를 동시에 강화하며, 기존 SSCL 기법에 추가만 해도 일관된 성능 향상을 보인다.

## 배경과 선행 연구

### SimCLR 손실의 두 항 분해

SimCLR의 NCE 손실은 limit 형태로 두 항으로 분해된다.

```
L_NCE = -(1/τ) * E[ sim(z_i, z_j) ] + E[ log E exp( sim(z_i, z_j) / τ ) ]
```

첫 항은 같은 입력의 augmentation 쌍을 가까이 두는 alignment 항, 둘째 항은 표현이 균등 분포에 가까워지도록 하는 uniformity 항이다.
저자들은 이 분해를 일반화해 GLF의 출발점으로 삼는다.

### Related Work 정리

| 카테고리 | 주요 흐름 | 본 연구와의 차이 |
|----------|-----------|------------------|
| Contrastive | SimCLR, MoCo | uniformity 항 — GLF는 명시적 클래스 구조 모델링 |
| Negative-free | BYOL, SimSiam | implicit 정칙화 — GLF로 통합 |
| Decorrelation | Barlow Twins, VICReg | 차원별 통계 — GLF로 통합 |
| Cluster-based | SwAV | prototype 클러스터링 — GLF로 통합 |

저자들이 강조하는 차별점은 두 가지다.
첫째, 다양한 SSCL 기법을 GLF의 align/constrain 분해 안에서 일관되게 설명한다.
둘째, 기존 기법이 클래스 내 응집과 클래스 간 분리를 명시적으로 모델링하지 않는다는 점을 ADC로 보완한다.

저자는 Theorem 1에서 다음 일반화 한계를 제시한다.

```
L_CE^μ(f*) ≤ L_NCE(f) + √Var(f(x)|y) + M(n)
```

이 한계는 클래스 내 분산 Var(f(x)|y) 감소가 downstream 성능 향상의 충분조건임을 보여준다.

## 방법론

### GLF 일반 목적함수

GLF는 SSCL을 다음과 같이 일반화한다.

```
min_{f, f_p}  L_align(X_aug, f, f_p) + L_constrain(X_aug, f, f_p)
```

L_align은 같은 입력의 두 augmentation 표현을 가까이 두고, L_constrain은 표현 공간에 추가 구조 제약을 부여한다.
ADC는 L_constrain 부분을 DCM과 LPM으로 구성한다.

### Distribution Calibration Module (DCM)

DCM은 anchor z_i 중심에 두 분포를 정의하고 그 차이를 최소화한다.

```
Calibration: N(z; z_i, Σ)               (Gaussian, 평균 근처 밀도 큼)
Data:        F(z; z_i, Σ'), Σ'=ρΣ/(ρ-2)  (Student-t, 꼬리 무거움)
```

목적은 KL divergence 최소화다.

```
L_DCM = Σ_i KL( Sg( N(z; z_i, Σ) ) || F(z; z_i, Σ') )
```

이산 구현은 다음과 같다.

```
L_DCM = Σ_{i,j} -Sg( p_cal^i(z_j) ) * log[ Sg(p_cal^i(z_j)) / p_dat^i(z_j) ]
```

Gaussian의 head 영역이 응집을 유도하고 Student-t의 heavy tail이 멀리 있는 샘플을 더 멀리 밀어낸다.

### Local Preserving Module (LPM)

LPM은 사전학습 feature extractor(예: CLIP)에서 prior 분포를 정의하고 현재 feature가 이를 보존하도록 Dirichlet으로 제약한다.

```
L_LPM = Σ_i Dir( P_i^data || P_i^pre )
```

이는 입력 공간의 상대 거리를 feature 공간으로 보존하며, anchor가 outlier일 경우 entropy 가중으로 영향을 줄인다.

```
L_DCM (entropy-weighted) = (1/H(P_i^pre)) * Σ -Sg(p_cal^i(z_j)) * log[ ... ]
```

### 전체 ADC 손실

ADC는 기존 contrastive 손실에 두 모듈을 결합한다.

```
L(f, f_p) = L_ctr(f, f_p) + ν * L_DCM(f, f_p) - υ * L_LPM(f, f_p)
```

ν, υ는 양수 하이퍼파라미터다.

## 실험 셋업

| 항목 | 값 |
|------|-----|
| 주된 backbone | ResNet-18 (linear eval), ResNet-50 (transfer) |
| 데이터셋 | CIFAR-10, CIFAR-100, STL-10, Tiny ImageNet, ImageNet-100, ImageNet, PASCAL VOC, COCO |
| Baseline SSCL | SimCLR, MoCo, BYOL, Barlow Twins, W-MSE, VICRegL, SimSiam, MEC |
| Pretrained encoder for LPM | identity, SimCLR, CLIP, EVA-02 |
| 반복 평균 | 5 runs, V100 GPU |

## 주요 결과

### Linear evaluation

CIFAR-10 ResNet-18.

| 방법 | Baseline | +ADC |
|------|-----------|--------|
| W-MSE | 91.99 | 93.81 |
| SimSiam | 91.71 | 93.41 |
| SimCLR | 91.80 | 93.46 |
| BYOL | 91.73 | 93.70 |

CIFAR-100 ResNet-18.

| 방법 | Baseline | +ADC |
|------|-----------|--------|
| W-MSE | 67.64 | 69.57 |
| SimCLR | 66.83 | 68.93 |

STL-10 ResNet-18.

| 방법 | Baseline | +ADC |
|------|-----------|--------|
| BYOL | 91.99 | 94.01 |

전반적으로 +ADC가 1.5~2.0%p 가까운 일관된 향상을 보였다.

### Semi-supervised와 Transfer

ImageNet 1%, 10% annotation 영역에서 +ADC가 baseline 대비 3.5%p 이상 향상되었다(논문 Table 8).

### Object detection과 segmentation

PASCAL VOC AP_50 (object detection).

| 방법 | Baseline | +ADC |
|------|-----------|--------|
| SimCLR | 75.9 | 77.5 |
| MoCo | 77.1 | 79.0 |
| SimSiam | 77.3 | 79.0 |

COCO Mask R-CNN AP^mask.

| 방법 | +ADC |
|------|--------|
| SimCLR | 35.5 |
| MoCo | 36.0 |
| MEC | 36.1 |

탐지와 분할 모두에서 +ADC가 baseline을 일관되게 능가한다.

### Ablation

하이퍼파라미터 ν, υ는 10^-1 ~ 10^1 범위에서 안정적이며 광범위한 영역에서 향상이 유지된다.
DCM과 LPM 각각이 단독으로도 baseline을 향상시키지만 결합 시 더 큰 향상이 발생해 두 모듈이 보완 관계임을 입증한다.
LPM의 사전학습 encoder를 identity, SimCLR, CLIP, EVA-02로 바꾼 실험에서 결과 변동이 미미해 encoder 선택에 강인함을 확인했다.

## 한계와 디스커션

저자가 본문에서 인정하는 한계는 다음과 같다.
첫째, ADC는 plug-and-play이지만 두 추가 손실(L_DCM, L_LPM)의 계산 비용이 baseline보다 늘어난다.
둘째, LPM은 별도 사전학습 encoder를 요구해 완전한 self-supervised 설정에서 외부 모델 의존성이 생긴다.
셋째, 평가는 주로 영상 도메인에 집중되어 있어 자연어·음성 등으로의 확장성은 추가 검증이 필요하다.

디스커션의 핵심은 두 가지다.
첫째, GLF의 align/constrain 분해는 다양한 SSCL 기법을 한 프레임에서 비교·결합할 수 있는 도구가 된다.
둘째, ADC는 클래스 내 응집과 클래스 간 분리를 명시적으로 모델링해 SSCL이 가지고 있던 implicit 한계를 보완한다.

## 결론

GLF는 SSCL을 align과 constrain의 두 항으로 정리하는 통합 프레임을 제시했고, ADC는 그 안에서 클래스 구조를 명시적으로 모델링하는 plug-and-play 모듈로 동작한다.
CIFAR-10/100, STL-10에서 1.5~2.0%p의 linear eval 향상, ImageNet 1%/10% 3.5%p 향상, VOC와 COCO 탐지·분할에서의 일관된 우위가 그 효과를 정량적으로 입증한다.
다양한 SSCL baseline에 추가만 해도 향상이 발생하고, encoder 선택에 강인한 점은 실용 도입의 큰 강점이다.

## Reference

- [A Generalized Learning Framework for Self-Supervised Contrastive Learning (arXiv:2508.13596)](https://arxiv.org/abs/2508.13596/)
