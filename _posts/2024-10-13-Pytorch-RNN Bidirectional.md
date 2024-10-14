---
layout: post
title: Pytorch로 RNN timeseries 예측(5) - Bidirectional
author: 'Juho'
date: 2024-10-13 09:00:00 +0900
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
1. [Bidirectional RNN](#bidirectional-rnn)
2. [Bidirectional 추가 방법](#bidirectional-추가-방법)

## Bidirectional RNN
Bidirectional RNN은 일반적인 RNN의 확장으로 입력 시퀀스를 순방향과 역방향 두가지 방향으로 처리하여 더 자세한 컨텍스트 정보를 활용하는 구조다.<br/>
순방향 RNN: 시퀀스를 첫 번째 시점부터 마지막 시점까지 순서대로 처리<br/>
역방향 RNN : 시퀀스를 마지막 시점부터 첫 번째 시점까지 역순으로 처리<br/>
양방향 RNN: 순방향 RNN과 역방향 RNN의 출력을 결합하여 최종 출력을 생성<br/>

동작원리는 간단하게 설명하면 <br/>
순방향 레이어: 입력 시퀀스를 정방향으로 처리하여 각 시점에서의 순방향 hidden state를 계산<br/>
역방향 레이어: 입력 시퀀스를 역방향으로 처리하여 각 시점에서의 역방향 hidden state를 계산<br/>
출력 결합: 각 시점에서 순방향과 역방향의 hidden state를 결합하여 최종 출력 또는 다음 레이어의 입력으로 사용<br/>

양방향 RNN의 장점 <br/>
1) 더 많은 컨텍스트 정보 활용 <br/>
2) 성능 향상<br/>

양방향 RNN의 단점<br/>
1) 실시간 예측이나 온라인 처리에는 부적합<br/>
2) 모델의 복잡도 증가<br/>
3) 과적합 위험<br/>

시계열 예측에서 양방향 RNN의 적합성 <br/>
시계열 예측은 과거의 데이터를 사용하여 미래를 예측하는 것이 목표<br/>
따라서 예측 시점에서는 미래의 데이터가 존재하지 않아 시계열 예측 문제에는 적합하지 않음<br/>

그럼에도 불구하고 굳이 추가를 해보는 과정<br/>


## Bidirectional 추가 방법
1) Encoder 추가
```python
class Encoder(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers, dropout, bidirectional):
        super(Encoder, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers
        self.dropout = dropout
        self.bidirectional = bidirectional
        self.num_directions = 2 if self.bidirectional else 1
        self.rnn = nn.RNN(input_size, hidden_size, num_layers, batch_first=True, dropout=dropout if num_layers > 1 else 0, bidirectional=bidirectional)
        
    def forward(self, x):
        h0 = torch.zeros(self.num_layers * self.num_directions, x.size(0), self.hidden_size).to(x.device)
        output, hidden = self.rnn(x, h0)
        return output, hidde
```

2) Decoder
Decoder의 경우 일반적으로 양방향을 사용하지 않음 <br/>
Decoder는 순차적으로 출력을 생성하기 때문에 미래 정보를 사용할 수 없기 때문 <br/>


3) Seq2Seq
```python
class Seq2Seq(nn.Module):
    def __init__(self, encoder, decoder, device):
        super(Seq2Seq, self).__init__()
        self.encoder = encoder
        self.decoder = decoder
        self.device = device

    def forward(self, src, trg=None, teacher_forcing_ratio=0.5, output_seq_len=1):
        batch_size = src.size(0)
        output_size = self.decoder.fc.out_features
        
        outputs = torch.zeros(batch_size, output_seq_len, output_size).to(self.device)

        _, hidden = self.encoder(src)
        hidden = hidden.view(
            self.encoder.num_layers,
            self.encoder.num_directions,
            batch_size,
            self.encoder.hidden_size
        )
        hidden = torch.sum(hidden, dim=1)
        
        decoder_input = src[:, -1, :].unsqueeze(1)
        
        for t in range(output_seq_len):
            output, hidden = self.decoder(decoder_input, hidden)
            outputs[:, t, :] = output
            
            if trg is not None and np.random.rand() < teacher_forcing_ratio:
                decoder_input = trg[:, t, :].unsqueeze(1)
            else:
                decoder_input = output.unsqueeze(1)
        return outputs
```

양방향 RNN을 사용하면 hidden state의 차원이 증가함<br/>
따라서, Seq2Seq 모델에서 Encoder의 hidden state를 Decoder로 전달하기 전에 적절히 수정 필요<br/>

<br/>