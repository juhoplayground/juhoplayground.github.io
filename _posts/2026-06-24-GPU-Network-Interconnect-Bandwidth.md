---
layout: post
title: "기초부터 이해하는 GPU Network 1: GPU Interconnect Bandwidth"
author: 'Juho'
date: 2026-06-24 00:00:00 +0900
categories: [AI]
tags: [GPU, AI]
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
2. [데이터는 어떻게 GPU까지 흘러가는가](#데이터는-어떻게-gpu까지-흘러가는가)
3. [주된 병목: 메모리 대역폭](#주된-병목-메모리-대역폭)
   - [HBM과 데이터 기아](#hbm과-데이터-기아)
   - [메모리 장벽](#메모리-장벽)
4. [GPU Interconnect Bandwidth](#gpu-interconnect-bandwidth)
   - [다이와 칩을 잇는 기술](#다이와-칩을-잇는-기술)
   - [멀티 GPU 연결: PCIe, NVLink, NVSwitch](#멀티-gpu-연결-pcie-nvlink-nvswitch)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

GPU 서버의 성능은 GPU 개수만으로 결정되지 않는다.
Disk에서 DRAM, CPU, PCIe를 거쳐 GPU Memory와 GPU로 이어지는 데이터 흐름, HBM의 메모리 대역폭, 그리고 GPU 간 PCIe와 NVLink와 NVSwitch 토폴로지가 함께 성능을 결정한다.
이 글의 핵심 주장은 다음과 같다.

1. GPU 클러스터의 주된 병목은 메모리 대역폭이다.
2. GPU는 계산은 매우 빠르지만, 데이터를 HBM에서 충분히 빨리 공급받지 못하면 대기한다.
3. 멀티 GPU 서버에서는 GPU 개수보다 GPU 간 연결 구조가 더 중요하다.
4. PCIe-only, NVLink pair, 4-way NVLink domain, HGX NVSwitch fabric은 성능 특성이 완전히 다르다.
5. 대형 LLM 학습과 추론에서는 GPU 간 activation, intermediate tensor, KV cache, gradient 교환이 critical path에 들어간다.
6. NVSwitch 기반 HGX 구조는 GPU 간 통신을 가장 균일하게 만들어 대형 모델에 유리하다.

## 데이터는 어떻게 GPU까지 흘러가는가

먼저 일반적인 서버 흐름을 보자.
Disk Controller가 DMA를 사용해 Disk 데이터를 DRAM으로 옮기고, CPU는 DRAM에 올라온 데이터를 Cache로 읽어 연산한다.
DMA는 Disk Controller가 CPU를 거치지 않고 DRAM에 직접 데이터를 쓰는 역할을 한다.
대량 데이터 이동은 Controller가 맡고, CPU는 제어와 연산에 집중하게 만든다.

GPU를 사용하는 코드에서는 흐름이 한 단계 더 길어진다.

```bash
입력 준비   : Disk → DRAM → CPU
GPU로 전달  : CPU → PCIe/NVLink → GPU Memory
GPU 연산   : GPU Memory ↔ GPU
결과 회수   : GPU Memory → PCIe → DRAM → CPU
```

dataset, model weight, checkpoint 같은 데이터가 디스크에서 System DRAM으로 올라온다.
CPU가 전처리와 batch 구성, tensor 생성을 하고, 필요한 tensor를 PCIe를 통해 GPU 전용 메모리로 복사한다.
GPU는 GPU Memory의 데이터를 읽어 kernel을 실행하고, 결과를 다시 GPU Memory에 저장한다.
다음 layer나 kernel이 있으면 CPU로 돌아가지 않고 HBM 안에서 계속 이어진다.
최종 결과만 PCIe를 거쳐 DRAM으로 복사되어 CPU가 출력과 후처리를 한다.

## 주된 병목: 메모리 대역폭

### HBM과 데이터 기아

GPU의 연산 코어(Compute Core)는 충분히 빠른데, 메모리에서 데이터를 가져오는 데서 병목이 발생한다.
GPU 입장에서 데이터 기아 상태에 놓이는 것이다.
이를 완화하기 위한 것이 HBM(High Bandwidth Memory)이다.
HBM은 처리 속도가 아니라 너비(Interface Width)를 확장하는데, 이를 위해 DRAM 칩을 수직으로 쌓아올린다.
기존 방식으로는 연결이 불가능하기 때문에, 실리콘 인터포저 위에 GPU와 HBM을 연결한다.
그럼에도 GPU는 메모리가 공급하는 속도보다 훨씬 빠르게 연산할 수 있다.
그래서 텍스트 생성 과정에서 고가의 GPU가 아무 작업도 하지 않고 대기 상태에 놓이게 된다.

### 메모리 장벽

이 문제는 메모리 장벽(The Memory Wall)이라 불린다.
2026년 1월 구글의 샤오위 마와 튜링상 수상자 데이비드 패터슨이 발표한 논문이 정확한 수치를 제시했다.
LLM 학습과 Prefill은 연산 성능이 중요하지만, 토큰을 한 개씩 만드는 Decode는 GPU FLOPS보다 메모리 대역폭과 인터커넥트 지연시간에 더 크게 제한된다는 것이다.

| 기간 | 항목 | 증가폭 |
|------|------|------|
| 2012~2022 | FP64 연산 성능 | 80배 |
| 2012~2022 | 메모리 대역폭 | 17배 |

80을 17로 나누면 약 4.7이므로, 연산 성능과 메모리 공급 능력 사이의 상대적 불균형이 약 4.7배 확대되었다.
이 격차를 메모리 장벽이라 부르며, 이 현상은 나아지기는커녕 악화되고 있다.

## GPU Interconnect Bandwidth

### 다이와 칩을 잇는 기술

GPU 성능을 이야기할 때는 다이(die)부터 랙(rack)까지 여러 층위의 연결을 구분해야 한다.

| 용어 | 쉽게 말하면 |
|------|------|
| Die | 웨이퍼에서 잘라낸 실제 회로 조각 |
| Chip | 패키징되어 제품처럼 쓰이는 반도체 부품 |
| Chiplet | 하나의 큰 칩을 여러 작은 기능 die로 나눈 조각 |
| Package | die나 chiplet을 올리고 외부와 연결하는 물리 제품 |
| Interposer | 여러 die를 고밀도로 연결하는 중간 연결판 |
| C2C | chip-to-chip 또는 die-to-die 고속 연결 |

가장 안쪽의 연결부터 보면 두 가지 기술이 있다.
첫째, 다이와 다이를 잇는 NV-HBI(High-Bandwidth Interface)다.
한 패키지 안에서 GPU 다이 두 개를 직접 붙이는 기술이다.
Blackwell은 칩 하나가 사실 다이 두 개인데, 이 둘을 약 10 TB/s로 이어 운영체제에는 단일 GPU로 보이게 만든다.
둘째, NVLink-C2C(Chip-to-Chip)다.
같은 패키지 안에서 종류가 다른 칩, 즉 CPU와 GPU를 잇는다.
Grace CPU와 Blackwell GPU를 900 GB/s로 연결하고 메모리를 공유(cache-coherent)한다.
GPU가 CPU 메모리를, CPU가 GPU 메모리를 복사 없이 바로 들여다볼 수 있다는 게 핵심이다.

### 멀티 GPU 연결: PCIe, NVLink, NVSwitch

멀티 GPU 서버에서 주로 등장하는 연결 방식은 PCIe, NVLink, NVSwitch다.

| 방식 | 역할 |
|------|------|
| PCIe | 범용 연결, CPU와 GPU 그리고 GPU 간 통신 가능하지만 대역폭이 상대적으로 낮음 |
| NVLink | GPU끼리 직접 빠르게 연결하는 고속 링크, 2 또는 4 GPU domain에 유리 |
| NVSwitch | 여러 GPU를 고대역폭 fabric으로 묶는 구조, HGX 8 GPU급 서버의 핵심 |

비유하자면 NVLink가 길이라면 NVSwitch는 교차로다.
여러 GPU의 NVLink 포트를 하나의 스위치 칩에 모아, 어떤 GPU 쌍이든 같은 속도로 직접 통신하는 All-to-All 패브릭을 만든다.
PCIe는 CPU와 주변장치를 잇는 업계 표준 버스라 호환성은 좋지만, AI가 요구하는 GPU 간 데이터량을 감당하기엔 좁다.
같은 세대 기준으로 PCIe Gen5 x16의 약 128 GB/s와 NVLink 5의 1.8 TB/s는 약 14배 차이가 난다.
다만 PCIe가 사라진 것은 아니다.
CPU와 GPU를 잇는 표준 경로이자 NIC 같은 I/O 장치를 붙이는 통로로 여전히 쓰인다.
NVLink는 GPU끼리의 지름길, PCIe는 시스템 전체의 표준 도로라고 보면 된다.

구성별 대역폭을 비교하면 차이가 분명하다.

| 구성 | Interconnect | Point-to-Point 대역폭 |
|------|------|------|
| RTX PRO 6000 (PCIe Gen5) | PCIe | 128 GB/s bidirectional |
| H100 NVL (2-card bridge) | NVLink | 600 GB/s bidirectional |
| H200 NVL (4-way bridge) | NVLink | GPU당 900 GB/s, 합산 1.8 TB/s |
| HGX H100 / H200 (SXM) | NVSwitch | GPU당 900 GB/s to fabric |
| HGX B200 (SXM) | NVSwitch | GPU당 1.8 TB/s to fabric |

단일 서버 안의 멀티 GPU 연결을 성능과 균일성 관점으로 나열하면, 왼쪽의 PCIe 기반 구성은 유연하지만 통신 병목이 크다.
오른쪽의 NVSwitch 기반 HGX 구성은 GPU 간 통신이 가장 균일하다.

## 결론

GPU 서버의 성능은 GPU 개수가 아니라 데이터가 흐르는 전체 경로로 결정된다.
연산 코어는 빠른데 메모리 대역폭이 따라오지 못하는 메모리 장벽이 가장 근본적인 병목이고, 그 격차는 시간이 갈수록 벌어지고 있다.
멀티 GPU 환경에서는 GPU 간 연결 구조가 성능을 가른다.
PCIe-only, NVLink pair, 4-way NVLink domain, HGX NVSwitch fabric은 완전히 다른 성능 특성을 가지며, 대형 LLM 학습과 추론에서는 NVSwitch 기반 HGX 구조가 가장 균일한 통신으로 유리하다.

## Reference

- [기초부터 이해하는 GPU Network 1: GPU Interconnect Bandwidth](https://gasidaseo.notion.site/GPU-Network-1-GPU-Interconnect-Bandwidth-38750aec5edf80428383e33957b728cf)
