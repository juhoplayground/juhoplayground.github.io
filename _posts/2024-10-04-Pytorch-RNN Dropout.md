---
layout: post
title: Pytorch로 RNN timeseries 예측(4) - Dropout 추가
author: 'Juho'
date: 2024-10-04 09:00:00 +0900
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
1. [RNN timeseries Dropout 추가](#rnn-timeseries-dropout-추가)
2. [Dropout 추가 방법](#dropout-추가-방법)

## RNN timeseries Dropout 추가
Dropout은 모델이 학습하는 과정에서 과적합을 방지하기 방법이다.<br/>
과적합은 모델이 학습 데이터에 너무 잘 맞춰져서 테스트 데이터나 새로운 데이터에 대한 일반화 성능이 떨어지는 것을 이야기한다.<br/>
Dropout은 학습 과정에서 학습 단계에서 각 뉴런은 지정된 확률로 비활성화된다.<br/>
비활성화된 뉴런은 해당 학습 단계에서 입력과 출력을 계산하지 않으므로, 특정 뉴런에 의존하는 것을 줄여준다.<br/>
뉴런을 비활성화함으로써 모델이 더 많은 뉴런 조합에 대해 학습하게 되어, 더 일반화된 표현을 학습할 수 있다.<br/>
이는 새로운 데이터에 대한 모델의 성능을 향상시킨다.<br/>


## Dropout 추가 방법
1) Encoder Dropout 추가
```python
class Encoder(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers=1, dropout=0.2):
        super(Encoder, self).__init__()
        self.input_size = input_size
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.dropout = dropout
        self.rnn = nn.RNN(input_size, hidden_size, num_layers, batch_first=True, dropout=dropout if num_layers > 1 else 0)

    def forward(self, x):
        output, hidden = self.rnn(x)
        return output, hidden
```

2) Decoder Dropout 추가
```python
class Decoder(nn.Module):
    def __init__(self, hidden_size, output_size, num_layers=1, dropout=0.2):
        super(Decoder, self).__init__()
        self.output_size = output_size
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.dropout = dropout
        self.rnn = nn.RNN(output_size, hidden_size, num_layers, batch_first=True, dropout=dropout if num_layers > 1 else 0)
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x, hidden):
        out, hidden = self.rnn(x, hidden)
        out = self.fc(out.squeeze(1)) 
        return out, hidden
```

3) 하이퍼 파라미터 튜닝에 dropout 내용 추가
```python
def objective(trial):
    seq_length = trial.suggest_int('seq_length', 1, 31)
    batch_size = trial.suggest_categorical('batch_size', [16, 32, 64])
    hidden_size = trial.suggest_categorical('hidden_size', [8, 16, 32])
    num_layers =  trial.suggest_int("num_layers", 1, 10)
    dropout = batch_size = trial.suggest_categorical('dropout', [0.1, 0.2, 0.3])
```
보통 RNN 계열 모델에서는 0.2~0.3 정도의 비율을 많이 사용한다고 한다.<br/>
과적합이 심한 경우나 모델이 깊고 복잡한 경우는 0.5 정도의 비율을 사용하기도 한다고 한다.<br/>

---


<br/>