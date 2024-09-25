---
layout: post
title: Pytorch로 RNN timeseries 예측(3) - Hyper Parameters 튜닝
author: 'Juho'
date: 2024-09-25 09:00:00 +0900
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
1. [RNN timeseries Hyper parameters 튜닝](#rnn-timeseries-hyper-parameters-튜닝)
2. [튜닝 방법](#튜닝-방법)

## RNN timeseries Hyper parameters 튜닝
Hyper parameters 튜닝을 위해 사용할 것은 `Optuna`(https://optuna.readthedocs.io/en/stable/){:target="_blank"}로 설치하는 방법은 `pip install optuna`이다.<br/>
Optuna의 기본 개념은 `study` objective 함수에 기반하여 optimization을 수행, `trial`은 Study 내의 optimization 단일 수행.<br/>
Hyper parameters 튜닝을 위해서 objective와 study를 만들고, n_trials만큼 수행하면 된다.<br/>
<br/>
튜닝할 hyper parameter의 탐색 범위를 suggest_categorical, suggest_float, suggest_int 함수를 통해서 정하면 그 범위내에서 sampling 하여 최적화를 진행한다.<br/>
탐색 범위를 sampling하는 알고리즘은 `create_study`의 sampler를 수정하면 된다.<br/>
sampler의 종류는 아래와 같다.<br/>

![image](https://github.com/user-attachments/assets/e8984823-f009-4d9b-887d-8885205d0b4c)<br/>
어떠한 sampler를 사용해야할지 모를때는 Optuna에서 어떠한 sampler를 사용하는게 좋을지 표시해준 것을 참고하면 된다.<br/>
![image](https://github.com/user-attachments/assets/4263e8b8-007c-4c20-ac33-e511eee1fb70)<br/>

pruner 설정도 할 수 있는데 pruner는 학습 초반에 성능이 낮아보이는 trial을 중단시키는 내용이다.<br/>
pruner의 종류는 아래와 같다.<br/>
![image](https://github.com/user-attachments/assets/6e636831-b091-474a-8158-355bc417ab6c)<br/>

또한 어떠한 sampler와 pruner를 사용하는것이 좋은지 추천해주고 있다.<br/>
![image](https://github.com/user-attachments/assets/a8afcc56-e245-4de8-954d-20142fa402d6)<br/>

direction을 설정해서 최적화 방향을 설정할 수 있다. 기본 값으로는 minimize다.<br/>
소화를 위해서는 minimize를 설정하고, 최대화를 위해서는 maximize를 설정하면 된다.<br/>

## 튜닝 방법
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

import optuna
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


3) Hyper Parameters 튜닝
```python

def objective(trial):
    seq_length = trial.suggest_int('seq_length', 1, 31)
    batch_size = trial.suggest_categorical('batch_size', [1, 2, 4, 8, 16, 32, 64, 128, 256, 512])
    hidden_size = trial.suggest_categorical('hidden_size', [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048])
    num_layers =  trial.suggest_int("num_layers", 1, 10)
    
    sequences = create_sequences(scaled_data, seq_length)
    X, y = zip(*sequences)
    X = np.array(X, dtype=np.float32)
    y = np.array(y, dtype=np.float32)
    
    X_train, y_train, _, _ = split_data(X, y)
    
    train_dataset = TimeSeriesDataset(X_train, y_train)
    train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=False)
    
    input_size = X_train.shape[2] 
    hidden_size = hidden_size
    output_size = y_train.shape[1] 
    num_layers = num_layers

    learning_rate = 0.001
    num_epochs = 10
    
    encoder = Encoder(input_size, hidden_size, num_layers).to(device)
    decoder = Decoder(hidden_size, output_size, num_layers).to(device)
    model = Seq2Seq(encoder, decoder).to(device)
    
    criterion = nn.MSELoss()
    optimizer = optim.Adam(model.parameters(), lr=learning_rate)
    
    for epoch in range(num_epochs):
        model.train()
        train_losses = 0.0
        for batch_X, batch_y in train_loader:
            batch_X, batch_y = batch_X.to(device), batch_y.to(device)
            
            output = model(batch_X, batch_y, 30, 0.5)
            loss = criterion(output, batch_y)
            train_losses += loss.item()
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        train_losses /= len(train_loader)
        
        trial.report(train_losses, epoch)
        if trial.should_prune():
            raise optuna.exceptions.TrialPruned()
    
    return train_losses

study = optuna.create_study(sampler=optuna.samplers.TPESampler(), pruner=optuna.pruners.HyperbandPruner(), direction='minimize')
study.optimize(objective, n_trials=100)

optuna.visualization.plot_optimization_history(study)
optuna.visualization.plot_param_importances(study)
print(f"Best parameters: {study.best_params}")
print(f"Best training loss: {study.best_value}")
```


3) 슬라이딩 윈도우 생성
```python
def create_sequences(input_data, seq_length):
    sequences = []
    for i in range(len(input_data) - seq_length):
        seq = input_data[i:i + seq_length]
        label = input_data[i + seq_length]
        sequences.append((seq, label))
    return sequences

    seq_length = study.best_params['seq_length']
    sequences = create_sequences(scaled_data, seq_length)
    X, y = zip(*sequences)
    X = np.array(X, dtype=np.float32)
    y = np.array(y, dtype=np.float32)
```
거의 window size의 데이터를 이용하여 다음에 올 값을 예측하기 위해서 <br/>
데이터셋을 윈도우 크기만큼 슬라이딩 하여 여러개의 입출력 쌍으로 변환 <br/>
학습 데이터의 크기를 늘려 모델이 다양한 패턴을 학습하도록 함<br/>


4) Train, Valid, Test 데이터 분리
```python
def split_data(X, y):
    train_size = int(len(X) * 0.7)
    X_train, y_train = X[:train_size], y[:train_size]
    X_test, y_test = X[train_size:], y[train_size :]
    return X_train, y_train, X_test, y_test

X_train, y_train, X_test, y_test = split_data(X, y)
```
모델의 성능을 평가하고 과적합을 방지하기 위해서 데이터를 0.7, 0.3로 분리<br/>


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
test_dataset = TimeSeriesDataset(X_test, y_test)
    
batch_size = study.best_params['batch_size']
train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=False)
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

6) Seq2Seq RNN 
```python
class Encoder(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers=1):
        super(Encoder, self).__init__()
        self.input_size = input_size
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.rnn = nn.RNN(input_size, hidden_size, num_layers, batch_first=True)

    def forward(self, x):
        output, hidden = self.rnn(x)
        return output, hidden

class Decoder(nn.Module):
    def __init__(self, hidden_size, output_size, num_layers=1):
        super(Decoder, self).__init__()
        self.output_size = output_size
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.rnn = nn.RNN(output_size, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x, hidden):
        out, hidden = self.rnn(x, hidden)
        out = self.fc(out.squeeze(1)) 
        return out, hidden

class Seq2Seq(nn.Module):
    def __init__(self, encoder, decoder):
        super(Seq2Seq, self).__init__()
        self.encoder = encoder
        self.decoder = decoder

    def forward(self, x, y, y_len, teacher_forcing_ratio=0.5):
        batch_size = x.size(0)

        y = y.unsqueeze(1)
        target_len = y.size(2)

        _, hidden = self.encoder(x)
        decoder_input = x[:, -1, :].unsqueeze(1)
        
        outputs = torch.zeros(batch_size, target_len).to(x.device)
        for t in range(target_len):
            decoder_output, hidden = self.decoder(decoder_input, hidden)
            outputs[:, t] = decoder_output[:, 0]
            
            if random.random() < teacher_forcing_ratio:
                decoder_input = y
            else:
                decoder_input = decoder_output.unsqueeze(1)
                
        return outputs
    
    def recursive_forecast(self, initial_sequence, predict_length):
        self.eval()
        feature_dim = initial_sequence.shape[2]

        outputs = torch.zeros(predict_length, feature_dim).to(device)
        _, hidden = self.encoder(initial_sequence)
        decoder_input = initial_sequence[:,-1, :].unsqueeze(1)
        for t in range(predict_length): 
            decoder_output, hidden = self.decoder(decoder_input, hidden)
            outputs[t] =  decoder_output[0]
            decoder_input = decoder_output.unsqueeze(1)
            
        outputs = outputs.detach().cpu().numpy()
        return outputs
```

`Encoder`클래스는 입력 시퀀스를 받아서 고정된 길이의 hidden state로 변환<br/>
이 hidden state는 시퀀스의 정보를 압축하여 담고 있으며, 이후 `Decoder`에서 시퀀스 생성을 위해 사용됨<br/>

`Decoder`클래스는 `Encoder`로부터 전달받은 hidden state를 바탕으로 시퀀스를 생성<br/>

`Seq2Seq`클래스는 `Encoder`와 `Decoder`를 결합하여 시퀀스 입력을 시퀀스 출력으로 변환하는 전체 모델<br/>
`teacher_forcing_ratio`는 `Decoder`입력으로 실제 목표값을 사용할지, 아니면 모델의 출력을 사용하는지 결정하는 확률로 tearch forcing 구현<br/>
`recursive_forecast` 함수는 initial_sequence를 기반으로 주어진 길이만큼 시퀀스를 예측 <br/>

7) RNN 모델 구성 및 초기화
```python
input_size = X_train.shape[2] 
hidden_size = study.best_params['hidden_size']
output_size = y_train.shape[1] 
num_layers = study.best_params['num_layers']
learning_rate = 0.001
num_epochs = 3000

encoder = Encoder(input_size, hidden_size, num_layers).to(device)
decoder = Decoder(hidden_size, output_size, num_layers).to(device)
model = Seq2Seq(encoder, decoder).to(device)
criterion = nn.MSELoss()
optimizer = optim.Adam(model.parameters(), lr=learning_rate)
```



7) 모델 학습, 검증, 테스트 과정
```python
best_valid_loss = float('inf')
for epoch in range(num_epochs):
    model.train()
    train_losses = 0.0
    for batch_X, batch_y in train_loader:
        batch_X, batch_y = batch_X.to(device), batch_y.to(device)
            
        output = model(batch_X, batch_y, 30, 0.5)
        loss = criterion(output, batch_y)
        train_losses += loss.item()
            
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
            
    train_losses /= len(train_loader)
        
    if train_losses < best_valid_loss:
        best_valid_loss = train_losses
        torch.save(model.state_dict(), 'best_seq2seq_model.pth')
        
    print(f'Epoch [{epoch+1}/{num_epochs}] Train Loss: {train_losses:.4f}')
        
    
model.load_state_dict(torch.load('best_seq2seq_model.pth', weights_only=True))
model.eval()
test_predictions = []
test_targets = []
test_losses = []
    
with torch.no_grad():
    for inputs, targets in test_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        outputs = model(inputs, targets, 30)
        loss = criterion(outputs, targets)
        test_losses.append(loss.item())
        test_predictions.append(outputs.cpu().numpy())
        test_targets.append(targets.cpu().numpy())

print(f'Test Loss: {np.mean(test_losses):.4f}')
```

---


<br/>