---
layout: post
title: Pytorch로 RNN timeseries 예측
author: 'Juho'
date: 2024-09-10 09:00:00 +0900
categories: [Python]
tags: [Python, Pytorch, RNN]
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
1. [RNN timeseries](#rnn-timeseries)
2. [모델링 과정](#모델링-과정)

## RNN timeseries
n개의 column을 갖는 timeseries 형태의 데이터를 예측하는 RNN 모델을 만드는 과정<br/>
직전 window size개의 정보를 이용해서 이후 t개의 시점을 예측하는 many-to-many 방식<br/>

## 모델링 과정
0) 필요한 라이브러리 import
```python
import pandas as pd
import numpy as np
import random

from sklearn.preprocessing import MinMaxScaler

from torch.utils.data import Dataset, DataLoader
import torch.optim as optim
import torch.nn as nn
import torch

import matplotlib.pyplot as plt
```
아래의 내용을 하기 위해 필요한 라이브러리 import<br/>


1) 시드 값 고정
```python
def set_seed(seed):
    torch.manual_seed(seed)
    np.random.seed(seed)
    random.seed(seed)
    if torch.cuda.is_available():
        torch.cuda.manual_seed(seed)
        torch.cuda.manual_seed_all(seed)

set_seed(1111)
```
결과의 재현성을 보장하기 위해 시드 값을 고정하여 같은 데이터와 같은 모델 설정을 사용했을 때 항상 동일한 결과를 얻을 수 있도록함<br/>

2) 데이터 정규화
```python
def data_normalization(path):
    data = pd.read_csv(path)
    data = data.to_numpy()
    scaler = MinMaxScaler()
    scaled_data = scaler.fit_transform(data)
    return scaled_data, scaler

scaled_data, scaler = data_normalization(path='../data/time_series_data.csv')
```
데이터의 스케일을 조정하여 n개의 피처간에 공정한 비교와 모델의 수렴 속도 향상, Gradient Vanishing / EXploding 문제 완화하기 위해서 데이터 정규화를 진행<br/>


3) 슬라이딩 윈도우 생성
```python
def create_windows(data, window_size):
    X, y = [], []
    for i in range(len(data) - window_size):
        X.append(data[i:i + window_size])
        y.append(data[i + window_size])
    return np.array(X), np.array(y)

window_size = 7 
X, y = create_windows(scaled_data, window_size)
```
거의 window size의 데이터를 이용하여 다음에 올 값을 예측하기 위해서 <br/>
데이터셋을 윈도우 크기만큼 슬라이딩 하여 여러개의 입출력 쌍으로 변환 <br/>
학습 데이터의 크기를 늘려 모델이 다양한 패턴을 학습하도록 함<br/>


4) Train, Valid, Test 데이터 분리
```python
def split_data(X, y):
    train_size = int(len(X) * 0.7)
    valid_size = int(len(X) * 0.15)
    X_train, y_train = X[:train_size], y[:train_size]
    X_valid, y_valid = X[train_size:train_size + valid_size], y[train_size:train_size + valid_size]
    X_test, y_test = X[train_size + valid_size:], y[train_size + valid_size:]
    return X_train, y_train, X_valid, y_valid, X_test, y_test

X_train, y_train, X_valid, y_valid, X_test, y_test = split_data(X, y)
```
모델의 성능을 평가하고 과적합을 방지하기 위해서 데이터를 0.7, 0.15, 0.15로 분리<br/>


5) Dataset, Dataloader 생성
```python
class TimeSeriesDataset(Dataset):
    def __init__(self, X, y):
        self.X = torch.tensor(X, dtype=torch.float32)
        self.y = torch.tensor(y, dtype=torch.float32)

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

train_dataset = TimeSeriesDataset(X_train, y_train)
valid_dataset = TimeSeriesDataset(X_valid, y_valid)
test_dataset = TimeSeriesDataset(X_test, y_test)
    
batch_size = 32
train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=False)
valid_loader = DataLoader(valid_dataset, batch_size=batch_size, shuffle=False)
test_loader = DataLoader(test_dataset, batch_size=batch_size, shuffle=False)
```
pyTorch 모델을 훈련할 때 데이터를 효율적으로 관리하고 배치 사이즈 단위로 모델에 공급하기 위해 사용<br/>
TimeSeriesDataset은  사용자 정의 데이터셋을 생성, Dataset은 PyTorch에서 데이터를 다루는 기본 단위로, 데이터와 그에 대응하는 레이블을 관리<br/>
`__init__`은 데이터셋의 초기화 과정에서 입력 데이터(X)와 레이블(y)을 PyTorch 텐서로 변환하여 저장, 이로 인해 데이터를 모델에 바로 입력할 수 있음 <br/>
`__len__`은 데이터셋의 길이를 반환, 이는 데이터셋의 크기를 DataLoader 등에서 참조할 때 사용<br/>
`__getitem__`은  주어진 인덱스에 해당하는 입력 데이터와 레이블을 반환, 이 메서드는 모델이 학습할 때 특정 인덱스의 데이터를 불러오는 데 사용<br/>
<br/>
DataLoader은 Dataset 객체를 받아서 미니 배치 단위로 데이터를 처리하고, 모델 학습에 효율적으로 데이터를 공급 및 데이터의 배치 처리와 관련<br/>
배치 처리 : 이터를 지정된 batch_size로 나눠서 모델에 공급. 이는 모델이 한 번에 처리할 데이터의 양을 조절하여 학습을 안정적이고 효율적으로 수행<br/>
Shuffling : 데이터의 순서를 무작위로 섞어서 모델이 특정 순서에 의존하지 않고 학습할 수 있도록 함, 하지만 시계열 데이터의 경우 순서가 중요한 경우가 많기 때문에 False로 설정<br/>
병렬 데이터 로딩 : 여러 스레드를 사용하여 데이터를 병렬로 로드할 수 있습니다. 큰 데이터셋을 사용할 때 I/O 병목 현상을 줄여줌<br/>

6) RNN 모델 구성 및 초기화
```python
input_size = X_train.shape[2] 
hidden_size = 10
output_size = y_train.shape[1] 
num_layers = 1
learning_rate = 0.001
num_epochs = 500

model = RNN(input_size, hidden_size, output_size, num_layers).to(device)
criterion = nn.MSELoss()
optimizer = optim.Adam(model.parameters(), lr=learning_rate)
```

`input_size = X_train.shape[2]`은 RNN 모델의 입력 차원을 정의<br/>
X_train은 (batch_size, sequence_length, num_features) 형태의 3차원 텐서<br/>
X_train.shape[2]는 각 타임스텝에서 입력 데이터의 피처(feature) 수를 나타냄<br/>

`hidden_size = 10`은  RNN의 은닉층의 노드 개수를 정의<br/>
이 값은 RNN의 은닉 상태의 크기이며, 더 큰 값을 사용하면 모델이 더 많은 패턴을 학습할 수 있지만, 동시에 과적합의 위험도 증가할 수 있음<br/>

`output_size = y_train.shape[1]`은 RNN 모델의 출력 차원을 정의<br/>
y_train은 (batch_size, output_size) 형태의 2차원 텐서<br/>
output_size는 모델이 예측해야 하는 값의 개수 또는 출력 벡터의 크기를 나타냄<br/>

`num_layers = 1`은 RNN의 레이어 수(깊이)를 정의<br/>
RNN의 은닉층이 몇 개의 레이어로 쌓여 있는지를 결정<br/>
num_layers = 1로 설정하면 단일 레이어 RNN을 사용하게 됨<br/>
더 많은 레이어를 사용하면 더 복잡한 모델을 구성할 수 있지만, 계산 비용이 증가하고 학습이 어려워질 수 있음<br/>

`learning_rate = 0.001`은 학습률을 설정<br/>
학습률은 모델이 각 단계에서 가중치를 얼마나 빠르게 업데이트할지를 결정<br/>
너무 크면 최적화 과정에서 진동하거나 발산할 수 있고, 너무 작으면 학습이 느려질 수 있음<br/>

`num_epochs = 500`은 모델 학습 시 전체 데이터셋에 대해 몇 번 반복 수행할지 정의<br/>
에포크가 너무 적으면 모델이 충분히 학습하지 못하고, 너무 많으면 과적합될 수 있음<br/>

`model = RNN(input_size, hidden_size, output_size, num_layers).to(device)`은  RNN 모델을 초기화하고, 학습에 사용할 장치(GPU)를 설정<br/>

`criterion = nn.MSELoss()`은 손실 함수를 정의<br/>
평균 제곱 오차(MSE)를 계산하는 손실 함수<br/>
이 함수는 회귀 문제에서 자주 사용되며, 예측된 값과 실제 값의 차이를 제곱하여 평균한 값을 반환<br/>
모델이 예측하는 값과 실제 값의 차이를 최소화하기 위해 이 손실 함수를 사용<br/>

`optimizer = optim.Adam(model.parameters(), lr=learning_rate)`은 모델의 가중치를 업데이트하기 위한 옵티마이저를 정의<br/>
Adam 옵티마이저는 학습률을 조정하면서 가중치를 효율적으로 업데이트하는 알고리즘<br/>
Adam은 확률적 경사 하강법(SGD)의 변형으로, 학습률을 각 매개변수에 대해 개별적으로 조정하므로 빠르고 안정적인 학습을 가능하게 함<br/>


7) 모델 학습, 검증, 테스트 과정
```python
for epoch in range(num_epochs):
    model.train()
    train_losses = []
        
    for inputs, targets in train_loader:
        inputs, targets = inputs.to(device), targets.to(device)
            
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        train_losses.append(loss.item())
            
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    model.eval()
    valid_losses = []
    with torch.no_grad():
        for inputs, targets in valid_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
            loss = criterion(outputs, targets)
            valid_losses.append(loss.item())
        
    print(f'Epoch [{epoch+1}/{num_epochs}] Train Loss: {np.mean(train_losses):.4f}, Valid Loss: {np.mean(valid_losses):.4f}')
    
model.eval()
test_predictions = []
test_targets = []
test_losses = []

with torch.no_grad():
    for inputs, targets in test_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        test_losses.append(loss.item())
        test_predictions.append(outputs.cpu().numpy())
        test_targets.append(targets.cpu().numpy())

print(f'Test Loss: {np.mean(test_losses):.4f}')
```

`model.train()`을 통해서 학습 모드로 전환<br/>
이는 Dropout이나 Batch Normalization 등 학습 중에만 활성화되는 동작이 올바르게 수행되도록 함<br/>
train_loader에서 데이터를 배치 단위로 가져와 모델에 입력하고 손실을 계산하여 역잔파를 통해 모델의 파라미터를 업데이트 함<br/>
`model.eval()`을 통해서 검증 모드로 전환<br/>
검증 모드에서는 드롭아웃이 비활성화 됨<br/>
valid_loader에 대해 모델의 성능을 평가, 손실만 계산하며 모델의 파라미터는 업데이트 하지 않음<br/>
`torch.no_grad()`이 블록 안에서는 그라디언트를 계산하지 않음, 이는 메모리 사용을 줄이고 계산 속도를 높임<br/>



---

작성중 ...<br/>

<br/>