---
layout: post
title: "TorchComms: PyTorch를 위한 새로운 분산 통신 API"
author: 'Juho'
date: 2026-06-16 00:00:00 +0900
categories: [Pytorch]
tags: [Pytorch, GPU, Python]
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
2. [설계 목표와 동기](#설계-목표와-동기)
   - [기존 torch.distributed와의 관계](#기존-torchdistributed와의-관계)
   - [지원 백엔드](#지원-백엔드)
3. [핵심 API와 사용 예시](#핵심-api와-사용-예시)
   - [통신자 생성과 컬렉티브 연산](#통신자-생성과-컬렉티브-연산)
   - [비동기 연산](#비동기-연산)
4. [설치](#설치)
5. [프로젝트 상태](#프로젝트-상태)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

TorchComms는 PyTorch를 위한 새로운 실험적 통신 라이브러리입니다.
분산 컴퓨팅을 위한 컬렉티브(collective) 연산을 제공합니다.
프로젝트는 스스로를 PyTorch를 위한 새로운 실험적 통신 API라고 설명합니다.
고수준 컬렉티브 API와 함께 여러 종류의 백엔드 구현을 기본 제공합니다.

공식 문서에서는 TorchComms를 PyTorch Distributed(PTD)를 위한 가볍고 실험적인 통신 API로 소개합니다.
분산 컬렉티브 연산을 위한 단순화된 객체 지향 인터페이스를 제공하는 것이 목표입니다.
이 글에서는 [TorchComms 저장소](https://github.com/meta-pytorch/torchcomms){:target="_blank"}와 공식 문서를 기반으로 라이브러리의 설계 목표, 지원 백엔드, 핵심 API를 정리합니다.

## 설계 목표와 동기

TorchComms는 분산 학습에서 발생하는 여러 문제를 다룹니다.
저수준 통신의 복잡성을 깔끔한 객체 지향 설계로 추상화하는 데 초점을 맞춥니다.

주요 설계 목표는 다음과 같습니다.

| 목표 | 설명 |
|------|------|
| 단순화된 API | 저수준 통신 복잡성을 객체 지향 설계로 추상화 |
| 백엔드 유연성 | 코드 변경 없이 통신 백엔드 간 전환 |
| 프로덕션 검증 | NCCLX 백엔드는 Meta 프로덕션 환경에서 대규모 AI 워크로드를 구동하며 검증됨 |
| 타입 안전성 | 전체 타입 힌트와 검증으로 개발 경험 개선 |
| 성능 | GPU 가속 통신을 지원하는 최적화 구현 |

통신자 모델은 백엔드를 초기화하기 위해 new_comm() 을 사용합니다.
랭크(rank)와 월드 사이즈(world size) 정보를 조회하는 메서드를 함께 제공합니다.
그룹 관리 측면에서는 서브 그룹, 계층적 패턴, 자동 리소스 정리 기능을 갖추고 있습니다.

### 기존 torch.distributed와의 관계

TorchComms는 PyTorch Distributed를 위한 가벼운 통신 API로 자리매김합니다.
기존의 암묵적인 프로세스 그룹 대신 new_comm() 으로 이름이 부여된 통신자 인스턴스를 명시적으로 생성합니다.
통신자 생성 시 디바이스를 명시적으로 지정해야 합니다.
연산은 async_op 파라미터를 통해 비동기 변형으로 실행할 수 있으며, 반환된 work 객체는 wait() 메서드를 지원합니다.

지원하는 컬렉티브 연산은 다음과 같습니다.

| 분류 | 연산 |
|------|------|
| 리덕션 계열 | AllReduce, ReduceScatter, Reduce |
| 수집/분배 계열 | AllGather, Broadcast, Scatter, Gather |
| 점대점 통신 | Send, Recv |

위 연산들은 모두 동기 변형과 비동기 변형을 함께 제공합니다.

### 지원 백엔드

TorchComms는 여러 통신 백엔드를 지원합니다.
하드웨어 유연성을 확보하기 위해 다양한 벤더의 라이브러리를 포괄합니다.

| 백엔드 | 설명 |
|--------|------|
| NCCL | NVIDIA 표준 내장 백엔드 |
| NCCLX | Meta가 확장한 NCCL |
| RCCL | AMD ROCm |
| RCCLX | Meta가 확장한 RCCL |
| XCCL | Intel OneAPI |
| Gloo | CPU 폴백 (CPU 및 GPU 지원) |

NCCLX 백엔드는 Meta 프로덕션 환경에서 대규모 AI 워크로드를 구동하며 검증되었습니다.

## 핵심 API와 사용 예시

라이브러리는 동기와 비동기 컬렉티브 연산을 모두 지원합니다.
핵심 연산에는 ReduceOp.SUM 같은 리덕션 연산을 지원하는 all_reduce() 가 포함됩니다.

### 통신자 생성과 컬렉티브 연산

문서는 torchrun 을 사용해 여러 GPU에 걸쳐 AllReduce를 수행하는 완전한 예시를 제공합니다.

```python
#!/usr/bin/env python3
import torch
from torchcomms import new_comm, ReduceOp

def main():
    device = torch.device("device")
    torchcomm = new_comm("backend", device, name="main_comm")

    rank = torchcomm.get_rank()
    world_size = torchcomm.get_size()

    num_devices = torch.cuda.device_count()
    device_id = rank % num_devices
    target_device = torch.device(f"cuda:{device_id}")

    tensor = torch.full((1024,), float(rank + 1),
                        dtype=torch.float32, device=target_device)

    torchcomm.all_reduce(tensor, ReduceOp.SUM, async_op=False)
    torch.cuda.current_stream().synchronize()
    torchcomm.finalize()
```

new_comm() 으로 통신자를 생성한 뒤 get_rank() 와 get_size() 로 랭크와 월드 사이즈를 조회합니다.
all_reduce() 호출 시 async_op=False 로 동기 실행을 지정합니다.
연산이 끝나면 현재 스트림을 동기화하고 finalize() 로 통신자를 정리합니다.

### 비동기 연산

라이브러리는 work 핸들을 반환하여 논블로킹 연산을 지원합니다.

```python
work = torchcomm.all_reduce(tensor, ReduceOp.SUM, async_op=True)
# Perform other work
work.wait()
```

async_op=True 로 호출하면 즉시 work 핸들이 반환됩니다.
다른 작업을 수행한 뒤 work.wait() 를 호출하여 연산 완료를 기다립니다.

## 설치

TorchComms는 안정(stable) 빌드와 나이틀리(nightly) 빌드 모두 pip로 제공됩니다.
여러 CUDA 버전(12.6, 12.8, 13.0, 13.2)을 지원합니다.

소스에서 빌드하려면 CMake 3.22 이상과 Ninja 1.10 이상이 필요합니다.
NCCLX, RCCL, RCCLX에 대한 백엔드별 빌드 스크립트가 제공됩니다.

## 프로젝트 상태

저장소는 372개의 스타, 152개의 포크, 2,434개의 커밋을 기록하고 있습니다.
열린 이슈는 31개입니다.
배포 라이선스는 BSD 3-Clause이며, 서드파티 백엔드는 다른 라이선스를 사용할 수 있습니다.
NCCL은 별도로 포함됩니다.

## 결론

TorchComms는 PyTorch 분산 학습을 위한 새로운 실험적 통신 API입니다.
암묵적인 프로세스 그룹 대신 명시적이고 이름이 부여된 통신자 모델을 채택하여 단순화된 객체 지향 인터페이스를 제공합니다.
NCCL, NCCLX, RCCL, RCCLX, XCCL, Gloo 등 다양한 백엔드를 코드 변경 없이 전환할 수 있고, NCCLX 백엔드는 Meta 프로덕션 환경에서 검증되었습니다.
동기와 비동기 컬렉티브 연산, 점대점 통신을 모두 지원하므로 대규모 AI 워크로드의 통신 계층을 구성하려는 개발자에게 참고할 만한 라이브러리입니다.

## Reference

- [meta-pytorch/torchcomms (GitHub)](https://github.com/meta-pytorch/torchcomms/)
- [TorchComms Documentation](https://meta-pytorch.org/torchcomms/main/index.html/)
