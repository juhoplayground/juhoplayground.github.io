---
layout: post
title: "PyTorch FSDP 이해하기: 파라미터를 조각내는 분산 학습"
author: 'Juho'
date: 2026-07-08 00:00:00 +0900
categories: [Pytorch]
tags: [GPU, Python, Parallel Python]
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
2. [DDP와 FSDP 비교](#ddp와-fsdp-비교)
3. [작동 원리](#작동-원리)
   - [메모리를 결정하는 요소](#메모리를-결정하는-요소)
   - [사용 코드](#사용-코드)
4. [결론](#결론)
5. [Reference](#reference)

## 개요

FSDP(Fully Sharded Data Parallel)는 PyTorch의 고급 분산 학습 기법이다.
핵심 아이디어는 모델의 파라미터를 여러 GPU에 shard(조각) 단위로 나누어 저장하는 것이다.
전체 모델을 모든 GPU에 복제하지 않고 조각 단위로 분산하므로, 초대규모 모델을 학습할 때 메모리 효율을 극대화할 수 있다.

## DDP와 FSDP 비교

기존 DDP(Distributed Data Parallel)는 각 GPU가 전체 모델을 복제해서 들고 있다.
반면 FSDP는 파라미터 자체를 샤드 단위로 쪼개어 나눠 가진다.

| 항목 | DDP | FSDP |
|------|-----|------|
| 파라미터 관리 | 각 GPU가 전체 모델 복제 | 파라미터를 샤드 단위로 분산 |
| 메모리 사용 | 높음 | 낮음 |
| GPU 통신 | Gradient all-reduce만 수행 | 파라미터 all-gather / reduce-scatter 추가 |

DDP는 gradient에 대한 all-reduce만 수행하면 되지만, FSDP는 파라미터를 나눠 갖기 때문에 연산 시점에 all-gather로 모으고, 이후 reduce-scatter로 다시 흩뿌리는 추가 통신이 필요하다.

## 작동 원리

FSDP는 레이어 단위로 동작한다.
연산이 필요한 순간에만 해당 레이어의 파라미터를 all-gather로 모아서 연산을 수행하고, 끝나면 reduce-scatter로 다시 조각 단위로 흩어 놓는다.

```
파라미터 all-gather → 연산 수행 → reduce-scatter
```

모델을 여러 레이어 또는 파라미터 bucket 단위로 쪼개서 관리하므로, 전체 모델을 동시에 메모리에 올릴 필요가 없다.
이 덕분에 모델 병렬의 메모리 절약 장점과 데이터 병렬의 빠른 병렬처리 장점을 결합할 수 있다.

### 메모리를 결정하는 요소

학습 시 메모리에 영향을 주는 요소는 크게 세 가지다.

| 요소 | 메모리 영향 |
|------|------|
| Weights | 상대적으로 작음 |
| Activations | 가장 큼 (배치 크기에 따라 급증) |
| Gradients | Weights와 비슷함 |

Activations가 배치 크기에 따라 가장 크게 증가하는 부분이므로, 파라미터를 나누는 FSDP와 함께 배치 구성을 고려하는 것이 중요하다.

### 사용 코드

FSDP 적용 자체는 간단하다.
프로세스 그룹을 초기화한 뒤 모델을 FSDP로 감싸면 된다.

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
import torch.distributed as dist

dist.init_process_group(backend='nccl')
fsdp_model = FSDP(model)
```

멀티 머신 환경에서는 torchrun으로 노드와 프로세스 수를 지정해 실행한다.

```bash
torchrun --nnodes=2 --nproc_per_node=4 train.py
```

## 결론

FSDP는 파라미터를 GPU끼리 조각내어 나눠 갖고, 연산이 필요할 때만 all-gather로 모으는 방식으로 메모리를 절약한다.
모델 병렬의 메모리 이점과 데이터 병렬의 병렬처리 이점을 결합하기 때문에, 단일 GPU 메모리에 올라가지 않는 초대규모 모델 학습에 효과적인 선택지가 된다.

## Reference

- [PyTorch FSDP 완벽 이해하기](https://mvje.tistory.com/285/)
