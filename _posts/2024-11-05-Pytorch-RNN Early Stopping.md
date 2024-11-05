---
layout: post
title: Pytorch로 RNN timeseries 예측(7) - Early Stopping
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
1. [Early Stopping](#early-stopping)
2. [Scaler 종류와 특징](#scaler-종류와-특징)
 - 1) [MinMaxScaler](#1-minmaxscaler)
 - 2) [MaxAbsScaler](#2-maxabsscaler)
 - 3) [StandardScaler](#3-standardscaler)
 - 4) [RobustScaler](#4-robustscaler)
3. [Scaler가 모델에 미치는 영향](#scaler가-모델에-미치는-영향)
4. [Scaler 선택 가이드](#scaler-선택-가이드)

## Early Stopping
학습 과정에서 과적합을 방지하기 위해 학습을 조기에 중단하는 기법을 의미<br/>
모델이 일정 횟수 이상 학습을 진행하면서도 검증 성능이 개선되지 않는 경우, <br/>
학습을 중담함으로써 불필요한 학습을 방지하고 일반화 성능을 최적화할 수 있음<br/>

학습 초기에는 train_loss와 valid loss가 함께 감소하는데 <br/>
학습이 충분히 진행되면 train_loss는 계속 감소하지만, valid_loss는 감소하지 않고 증가하는 시점이 올 수 있음 <br/>
이 시점에서 모델이 훈련 데이터에 과적합되고 있음을 알 수 있음<br/>
Early Stopping은 이러한 시점에서 학습을 중단하여 과적합을 방지하고, 검증 데이터에 대한 성능이 가장 좋았던 모델을 저장하는 것 <br/>

또한 학습을 조기에 중단하면 불필요하게 더 많은 epoch를 학습하지 않게 되어 학습 시간을 절약할 수 있음<br/>
리소스가 제한된 환경에서는 매우 유용<br/>



## Early Stopping 구현
```python
class EarlyStopping:
    def __init__(self, patience=20, verbose=False, delta=0, path='model.pth'):
        self.patience = patience
        self.verbose = verbose
        self.counter = 0
        self.best_loss = None
        self.early_stop = False
        self.delta = delta
        self.path = path
    
    def __call__(self, epoch, num_epochs, val_loss, model):
        if self.best_loss is None:
            self.best_loss = val_loss
            self.save_checkpoint(epoch, num_epochs, val_loss, model)
        elif val_loss > self.best_loss - self.delta:
            self.counter += 1
            if self.verbose:
                print(f"EarlyStopping counter: {self.counter} out of {self.patience}")
            if self.counter >= self.patience:
                self.early_stop = True
        else:
            self.best_loss = val_loss
            self.save_checkpoint(epoch, num_epochs, val_loss, model)
            self.counter = 0
    
    def save_checkpoint(self, epoch, num_epochs, val_loss, model):
        if self.verbose:
            print(f"Epoch [{epoch+1}/{num_epochs}] Validation loss decreased ({self.best_loss:.7f} --> {val_loss:.7f}).  Saving model ...")
        torch.save(model.state_dict(), self.path)


early_stopping = EarlyStopping(patience=20, verbose=True, delta=1e-4, path='model.pth')

best_valid_loss = float('inf')
for epoch in range(num_epochs):
        model.train()
        train_losses = 0.0
        for batch_X, batch_y in train_loader:
            batch_X, batch_y = batch_X.to(device), batch_y.to(device)
            output = model(batch_X, batch_y, teacher_forcing_ratio=0.5, output_seq_len=output_seq_len)
            loss = criterion(output, batch_y)
            train_losses += loss.item()
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            
        train_losses /= len(train_loader)
        
        model.eval()
        valid_losses = 0.0
        with torch.no_grad():
            for batch_X, batch_y in test_loader:
                batch_X, batch_y = batch_X.to(device), batch_y.to(device)
                output = model(batch_X, batch_y, teacher_forcing_ratio=0, output_seq_len=output_seq_len)
                loss = criterion(output, batch_y)
                valid_losses += loss.item()
        
        valid_losses /= len(test_loader)
        early_stopping(epoch, num_epochs, valid_losses, model)
        if early_stopping.early_stop:
            print("Early stopping")
            break
```
`patience` : 검증 손실이 개선되지 않을 때 학습을 멈추기까지 기다리는 epoch 수<br/>
`delta`: 개선 여부를 판단할 최소 변화량으로, 설정된 값보다 개선이 작으면 개선되지 않은 것으로 간주<br/>

