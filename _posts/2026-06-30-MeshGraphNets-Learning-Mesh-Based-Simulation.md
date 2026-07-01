---
layout: post
title: "MeshGraphNets: 그래프 신경망으로 메시 기반 물리 시뮬레이션 학습하기"
author: 'Juho'
date: 2026-06-30 00:00:00 +0900
categories: [AI]
tags: [AI, Pytorch, Benchmark]
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
2. [방법론](#방법론)
   - [Encode-Process-Decode 아키텍처](#encode-process-decode-아키텍처)
   - [적응형 Remeshing](#적응형-remeshing)
   - [학습 설정](#학습-설정)
3. [주요 결과](#주요-결과)
   - [실험 도메인](#실험-도메인)
   - [성능 비교](#성능-비교)
   - [기준선 비교와 일반화](#기준선-비교와-일반화)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

MeshGraphNets는 그래프 신경망을 사용해 메시 기반 물리 시뮬레이션을 학습하는 프레임워크입니다.
DeepMind의 Tobias Pfaff, Meire Fortunato, Alvaro Sanchez-Gonzalez, Peter W. Battaglia가 2020년 arXiv에 발표한 연구입니다.

메시 기반 유한요소 시뮬레이션은 구조역학, 공기역학, 전자기학, 지구물리학, 음향학 등에서 광범위하게 사용됩니다.
MeshGraphNets는 메시 그래프에서 메시지 전달을 수행하며, 순방향 시뮬레이션 중에 메시 이산화를 적응적으로 조정할 수 있습니다.
공기역학, 구조역학, 천(cloth) 시뮬레이션 등 다양한 물리 시스템의 동역학을 정확히 예측하면서도 기존 시뮬레이터보다 1~2자리수 빠릅니다.

이 연구의 핵심 혁신은 세 가지입니다.
첫째, 메시 공간과 유클리드 세계 공간에서 별도의 계산을 수행합니다.
둘째, 불규칙 메시를 통해 해상도에 독립적인 동역학을 학습합니다.
셋째, 시뮬레이션 도중 이산화를 자동으로 조정하는 적응형 메싱을 지원합니다.

관련 연구로는 규칙 격자를 다루는 CNN 기반 방법, 유체와 입자 동역학을 다루는 입자 기반 GNN(GNS 등), 그래프 합성곱과 Graph Element Networks 같은 메시 기반 선행 연구가 있습니다.
MeshGraphNets는 이들과 달리 3D 시스템과 수천 개 노드 규모까지 확장된다는 차별점을 가집니다.

## 방법론

### Encode-Process-Decode 아키텍처

시스템 상태는 메시로 표현합니다.
각 노드는 메시 공간의 참조 좌표, 동적 수량, 세계 공간 좌표(Lagrangian)를 가집니다.

Encoder는 멀티그래프를 생성합니다.
이 그래프는 메시 엣지와 세계 엣지 두 종류를 포함합니다.
메시 엣지는 내부 동역학을 캡처하고, 세계 엣지는 충돌과 접촉 같은 비국소 상호작용을 캡처합니다.
공간 등변성을 확보하기 위해 상대 엣지 특성을 사용합니다.
구체적으로 메시 변위와 그 크기, 세계 변위와 그 크기를 특성으로 삼고, 이를 크기 128의 잠재 벡터로 인코딩합니다.

Processor는 동일한 구조의 메시지 전달 블록 L개로 구성됩니다.
각 블록은 잔차 연결을 포함한 MLP로 엣지 특성과 노드 특성을 갱신합니다.

Decoder와 State Updater는 MLP로 잠재 노드 특성을 출력값으로 변환합니다.
1차 시스템에서는 예측값을 현재 값에 더해 다음 상태를 구합니다.
2차 시스템에서는 예측값에 현재 값의 2배를 더하고 이전 값을 빼서 적분합니다.

### 적응형 Remeshing

MeshGraphNets는 시뮬레이션 도중 메시를 세밀하게 조정하는 적응형 remeshing을 지원합니다.
sizing field가 각 위치에서 허용되는 엣지 길이를 정의합니다.

remeshing은 세 가지 로컬 연산으로 이루어집니다.

| 연산 | 목적 |
|------|------|
| 엣지 분할 | 해상도를 높이는 refinement |
| 엣지 축약 | 해상도를 낮추는 coarsening |
| 엣지 뒤집기 | aspect ratio 최적화 |

sizing field 자체도 동일한 아키텍처로 학습합니다.
테스트 시점에는 학습한 sizing field에 범용 remesher를 적용합니다.

### 학습 설정

MLP는 각 층 128 크기의 2층 ReLU 구조이며, LayerNorm과 입출력 정규화를 사용합니다.
최적화는 Adam을 사용하며, 총 10M 스텝 동안 학습합니다.
학습률은 1e-4에서 1e-6까지 지수적으로 감소시킵니다.

수백 단계에 이르는 롤아웃의 견고성을 확보하기 위해 Training Noise 기법을 사용합니다.
최근 값에 평균 0이고 고정 분산을 가진 정규 노이즈를 추가하는 방식입니다.

아키텍처 ablation 결과, 메시지 전달 블록은 15개가 최적이었습니다.
히스토리는 최소로 두는 것이 최적이며, 추가할 경우 과적합이 발생했습니다.

## 주요 결과

### 실험 도메인

여섯 개의 물리 도메인에서 실험했습니다.
각 데이터셋은 1000개의 훈련 궤적, 100개의 검증 궤적, 100개의 테스트 궤적으로 구성됩니다.

| 데이터셋 | 시스템 | 솔버 | 메시 | 스텝 | Δt |
|----------|--------|------|------|------|----|
| FlagSimple | Cloth | ArcSim | Triangle 3D Regular | 400 | 0.02 |
| FlagDynamic | Cloth | ArcSim | Triangle 3D Dynamic | 250 | 0.02 |
| SphereDynamic | Cloth | ArcSim | Triangle 3D Dynamic | 500 | 0.01 |
| DeformingPlate | Hyper-elasticity | COMSOL | Tetrahedral 3D Irregular | 400 | - |
| CylinderFlow | Incompressible NS | COMSOL | Triangle 2D Irregular | 600 | 0.01 |
| Airfoil | Compressible NS | SU2 | Triangle 2D Irregular | 600 | 0.008 |

### 성능 비교

아래 표는 각 데이터셋의 노드 수, 실행 시간, RMSE를 정리한 것입니다.
t_model은 신경망 추론만 측정한 시간입니다.
t_full은 remeshing과 그래프 재계산을 포함한 시간입니다.
t_GT는 기준 시뮬레이터의 실행 시간입니다.
RMSE는 1-step, rollout-50, rollout-all 세 가지로 측정했습니다.

| 데이터셋 | 노드수 | t_model(ms) | t_full(ms) | t_GT(ms) | RMSE 1-step | RMSE rollout-50 | RMSE rollout-all |
|----------|--------|-------------|------------|----------|-------------|-----------------|------------------|
| FlagSimple | 1579 | 19 | 19 | 4166 | 1.08 | 92.6 | 139.0 |
| FlagDynamic | 2767 | 43 | 837 | 26199 | 1.57 | 72.4 | 151.1 |
| SphereDynamic | 1373 | 32 | 140 | 1610 | 0.292 | 11.5 | 28.3 |
| DeformingPlate | 1271 | 24 | 33 | 2893 | 0.25 | 1.8 | 15.1 |
| CylinderFlow | 1885 | 21 | 23 | 820 | 2.34 | 6.3 | 40.88 |
| Airfoil | 5233 | 37 | 38 | 11015 | 314 | 582 | 11529 |

기존 솔버 대비 일관되게 1~2자리수 빠르며, 구체적으로 11배에서 289배까지의 속도 향상을 보입니다.

### 기준선 비교와 일반화

여러 기준선과 비교한 결과 MeshGraphNets의 우위가 확인되었습니다.

입자 기반 GNS는 불규칙 메시에서 실패했고 롤아웃이 불안정했습니다.
GCN은 정상상태 예측만 성공했으며 동적 시뮬레이션에서 불안정했습니다.
상대 인코딩을 제거하면 RMSE가 기본 11.5에서 26.5로 악화되었습니다.
U-Net(CNN)은 작은 규모의 언더샘플링 때문에 후류 지역에서 4배의 셀을 사용하고도 MeshGraphNets에 미치지 못했습니다.

일반화 성능도 우수했습니다.
Airfoil 파라미터 외삽 실험에서, 훈련은 각도 -25~25°와 Mach 0.2~0.7 범위에서 진행하고 테스트는 각도 -35~35°와 Mach 0.7~0.9 범위로 넓혔습니다.
이때 RMSE는 11.5에서 13.1로 소폭 증가하는 데 그쳤습니다.
물고기 모양 깃발이나 노드 20k 규모의 windsock(훈련의 10배) 같은 미경험 기하와 규모에서도 성공적으로 동작했습니다.
이는 해상도와 규모에 독립적인 학습이 이루어졌음을 입증합니다.

## 한계와 주의사항

Cloth 도메인은 카오스적 성질 때문에 50단계 이후 오류가 누적되는 decoherence 현상이 나타납니다.
tetrahedral 메시나 quad 메시는 삼각형 메시와 다른 remesher가 필요합니다.

## 결론

MeshGraphNets는 다양한 물리 시스템을 정확하고 효율적으로 모델링하는 범용 메시 기반 방법입니다.
해상도 적응성 덕분에 기존 시뮬레이터 대비 뚜렷한 우위를 갖습니다.
향후 과제로는 physics-informed loss와 energy-conserving 적분 등이 제시되었습니다.
코드와 데이터셋은 deepmind-research 저장소에 공개되어 있습니다.

## Reference

- [Learning Mesh-Based Simulation with Graph Networks](https://arxiv.org/abs/2010.03409)
