---
layout: post
title: Pytorch로 RNN timeseries 예측(6) - Scaler
author: 'Juho'
date: 2024-10-27 09:00:00 +0900
categories: [Pytorch]
tags: [Pytorch, RNN]
pin: True
toc : True
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
1. [Scaler](#scaler)
2. [Scaler 종류와 특징](#scaler-종류와-특징)
 - 1) [MinMaxScaler](#1-minmaxscaler)
 - 2) [MaxAbsScaler](#2-maxabsscaler)
 - 3) [StandardScaler](#3-standardscaler)
 - 4) [RobustScaler](#4-robustscaler)
3. [Scaler가 모델에 미치는 영향](#scaler가-모델에-미치는-영향)
4. [Scaler 선택 가이드](#scaler-선택-가이드)

## Scaler
데이터 스케일링은 모델의 학습 속도와 성능에 큰 영향을 미치는 중요한 전처리 과정<br/>
스케일링을 통해 feature간의 균형을 맞추고 모델이 각 feature를 공정하게 학습하도록 도움<br/>
특히 신경망 모델에서는 가중치의 초기화, 활성화 함수의 작동 범위, 그레디언트의 크기 등이 데이터 스케일에 민감하게 반응할 수 있음<br/>

## Scaler 종류와 특징
#### 1) MinMaxScaler <br/>
데이터를 0과 1 사이로 스케일링, 최소값과 최대값을 사용하여 모든 데이터가 동일한 범위를 갖도록 스케일링 함<br/>
데이터의 분포를 유지하면서도 모든 feature가 동일한 스케일을 가지도록 함<br/>

#### 2) MaxAbsScaler <br/>
절대 최대값으로 스케일링하여 각 feature를 해당 feature의 절대 최대값으로 나눠, 데이터는 -1과 1사이의 값으로 스케일링 함<br/>
양수와 음수 값을 모두 포함하는 데이터에 적합<br/>

#### 3) StandardScaler <br/>
평균 제거 및 단위 분산 스케일링을 통하여 각 feature의 평균을 0, 표준편차를 1로 스케일링 함<br/>
정규 분포를 따르는 데이터에 적합<br/>

#### 4) RobustScaler <br/>
중앙값과 IQR을 사용하여 이상치에 덜 민감하도록 중앙값과 사분위 값을 사용하여 스케일링 함 <br/>
이상치가 많은 데이터에 적합<br/>

## Scaler가 모델에 미치는 영향
1) 활성화 함수의 특성 : 모델에 사용하는 활성화 함수는 입력 값의 범위에 따라 출력이 달라짐<br/>
2) 그레디언트 소실 및 폭주 : 데이터의 스케일이 너무 크거나 작으면 그레디언트 소실 또는 폭주 문제가 발생할 수 있음 <br/>
3) 가중치 초기화와 학습 속도 : 데이터의 스케일이 적절하지 않으면 가중치 초기화의 효과가 감소하고 학습 속도가 저하될 수 있음<br/>

## Scaler 선택 가이드
1) 데이터의 특성 파악<br/>
- 데이터 분포 확인 : 데이터가 정규 분포를 따르는지, 이상치가 많은지 확인<br/>
- 값의 범위 : 데이터에 음수 값이 포함되어 있는지, 값의 범위가 큰지 확인<br/>
- 스케일링한 데이터의 히스토그램이나 박스 플롯을 그려 분호를 확인<br/>

2) 모델의 특성 고려<br/>
- 활성화 함수 : 모델에서 사용하는 활성화 함수의 입력 범위에 맞게 스케일링 <br/>
- sigmoid 함수 : 입력 값이 0 ~ 1 사이일 때 효과적<br/>
- tanh 함수 : 입력 값이 -1 ~ 1 사이일 때 효과적<br/>

3) 테스트를 통한 검증 <br/>
다양한 Scaler를 사용하여 모델의 성능을 비교 <br/>
각 Scaler에 대해 교차 검증을 수행하여 안정적인 성능을 확인하고<br/>
MSE,  MAE, RMSE 등 다양한 평가 지표를 통해 모델의 성능을 종합적으로 평가하여 Scaler 선택<br/>
