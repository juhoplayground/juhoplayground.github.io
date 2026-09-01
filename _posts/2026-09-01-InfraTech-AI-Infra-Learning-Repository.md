---
layout: post
title: "InfraTech: AI 인프라 학습을 위한 오픈소스 자료 저장소"
author: 'Juho'
date: 2026-09-01 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, vLLM]
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
2. [저장소 구조](#저장소-구조)
   - [디렉토리 구성](#디렉토리-구성)
   - [콘텐츠 형식](#콘텐츠-형식)
3. [주요 주제 영역](#주요-주제-영역)
   - [추론 기초와 프레임워크](#추론-기초와-프레임워크)
   - [추론 성능 최적화](#추론-성능-최적화)
   - [강화학습 프레임워크](#강화학습-프레임워크)
   - [훈련 프레임워크와 기초](#훈련-프레임워크와-기초)
   - [어텐션 메커니즘과 모델 분석](#어텐션-메커니즘과-모델-분석)
4. [보조 도구](#보조-도구)
5. [학습 경로](#학습-경로)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

[InfraTech](https://github.com/CalvinXKY/InfraTech){:target="_blank"}는 AI 인프라 분야의 지식과 코드 실습을 공유하는 오픈소스 학습 자료 저장소다.
PyTorch, vLLM, SGLang, slime, vime 등 훈련과 추론 양쪽의 프레임워크를 다루며, 성능 가속과 대규모 모델 기초를 함께 정리한다.
현재 3.8k 스타와 358개 포크를 기록하고 있으며 98개의 커밋이 누적되어 있다.

일반적인 프레임워크 문서가 API 사용법에 집중하는 것과 달리, 이 저장소는 내부 동작 원리를 노트북 형태의 실행 가능한 코드로 풀어내는 데 초점을 둔다.
KV 캐시 관리, 병렬화 전략, 어텐션 변형 구현 같은 주제를 실제 코드와 함께 확인할 수 있다.
콘텐츠는 중국어(간체)로 작성되어 있다.

## 저장소 구조

저장소는 주제별 디렉토리로 나뉘며, 각 디렉토리 안에 노트북과 설명 문서가 배치되어 있다.

### 디렉토리 구성

| 디렉토리 | 용도 |
|------|------|
| llm_infer | LLM 추론 실습 및 최적화 기법 |
| models | 주요 모델 아키텍처 소개 |
| docs | AI 인프라 공유 자료 |
| deeplearning_framework | 딥러닝 프레임워크 기초 |
| deepseek_v3 | DeepSeek V3 모델 분석 |
| multi_lora | 멀티 LoRA 기법 |
| pytorch_vista | PyTorch 계산 그래프 시각화 |
| rl | 강화학습 관련 내용 |

### 콘텐츠 형식

실습 자료는 대부분 Jupyter Notebook 형식으로 제공된다.
노트북을 그대로 실행하면서 중간 텐서 값과 연산 결과를 확인할 수 있는 구성이다.

대표적인 노트북은 다음과 같다.

| 파일 | 내용 |
|------|------|
| rope_principle.ipynb | RoPE 계산 원리 |
| nano_vllm.ipynb | 단순화된 vLLM 구현 |
| speculative_decoding.ipynb | 투기적 디코딩 구현 |
| LoRA_to_Multi_LoRA.ipynb | LoRA에서 멀티 LoRA로의 확장 과정 |

각 문서에는 난이도 표시가 붙어 있어 학습 순서를 잡는 데 참고할 수 있다.
난이도는 3단계로 구분되며, 별도로 중요도를 나타내는 표시도 함께 사용된다.

## 주요 주제 영역

### 추론 기초와 프레임워크

추론 영역은 저장소에서 가장 비중이 큰 부분이다.
25개 이상의 노트북이 추론 관련 주제를 다룬다.

| 주제 | 설명 |
|------|------|
| ChunkedPrefill과 FlashDecoding | 프리필 분할과 디코딩 가속 원리를 코드 예제와 함께 설명 |
| LLM 추론 샘플링 | 샘플링 파라미터의 동작과 실전 적용 가이드 |
| Speculative Decoding | 투기적 추론의 원리와 방안별 비교 |
| 추론 양자화 | 양자화 기법의 기초 개념 정리 |

프레임워크 단위 자료도 시리즈로 구성되어 있다.
Nano-vLLM은 vLLM의 핵심 로직만 남긴 극단적 단순화 버전이고, vLLM은 1편부터 9편까지, SGLang은 1편부터 3편까지 순차적으로 다룬다.
프레임워크 전체를 읽기 전에 Nano-vLLM으로 골격을 파악하는 방식이 권장된다.

### 추론 성능 최적화

성능 최적화 섹션은 구체적인 개선 배수와 함께 기법을 정리한다.

| 기법 | 효과 |
|------|------|
| Ulysses 병렬 최적화 | DeepSeek V3/V3.2 적용 시 3.0배 향상 |
| PD 분리(Prefill-Decode Separation) | 1.5배 향상 |
| Swap 특성 활용 | 1.3배 향상 |
| AF 분리(Attention-FFN Separation) | 어텐션과 FFN 단계 분리 배치 |
| MoE 병렬 로드 밸런싱(EPLB) | 전문가 부하 불균형 완화 |
| FlashMLA | MLA 연산 가속 |

PD 분리는 프리필과 디코드를 별도 자원 풀에 배치해 각 단계의 특성에 맞춘 스케줄링을 가능하게 하는 기법이다.
디코드 단계는 메모리 대역폭에, 프리필 단계는 연산량에 각각 병목이 걸리는 특성 차이를 활용한다.

### 강화학습 프레임워크

RL 섹션은 vime 프레임워크와 slime 통합을 중심으로 구성된다.
vime은 slime과 vLLM을 결합한 RL 프레임워크로, 공유 카드 모드를 지원한다.

주요 주제는 다음과 같다.

- MoE-RL 훈련과 추론 사이의 일관성 문제(R3 원리)
- 메모리 에이전트 통합
- vLLM 백엔드 적응
- 비공유 카드 환경에서의 가중치 동기화
- P2P 분할 최적화

RL 훈련에서 롤아웃을 담당하는 추론 엔진과 학습을 담당하는 훈련 엔진 사이의 수치 불일치는 실무에서 자주 문제가 되는 지점인데, 이 부분을 별도 주제로 다룬다.

### 훈련 프레임워크와 기초

훈련 영역은 PyTorch 내부 동작 이해에 초점을 맞춘다.

| 주제 | 설명 |
|------|------|
| PyTorch Autograd 원리 | 자동 미분이 계산 그래프를 구성하고 역전파하는 방식 |
| PyTorch 없이 MLP 훈련 구현 | 순수 Python으로 전체 훈련 루프 작성 |
| PyTorch 메모리 관리 3부작 | 할당자 소스 코드 수준 해석 |
| 분산 훈련 기초 | DDP 사용법 |

고급 주제로는 Context Parallelism 원리, FP8 연산 적용, CUDA IPC 기반 텐서 공유가 포함된다.
분산 훈련과 추론의 공통 기반인 집합 통신(collective communication)의 원리와 실습도 별도로 다룬다.

### 어텐션 메커니즘과 모델 분석

어텐션 관련 자료는 위치 인코딩과 어텐션 변형을 세부적으로 파고든다.

- RoPE 계산 원리를 1D에서 3D까지 확장하며 설명
- MLA(Multi-head Latent Attention) 계산 흐름과 흡수 행렬 비교 분석
- DeepSeek V4의 RoPE 설계 해석
- DSA(DeepSeek Sparse Attention) 분석
- 선형 어텐션의 원리와 세부 구조(AlphaDeltaGate)
- Qwen3 VL 멀티모달 구조 해석
- Kimi K2.5/VL의 비전-언어 융합 흐름

주요 모델별 아키텍처 분석도 별도 디렉토리로 정리되어 있다.

| 모델 | 핵심 아키텍처 |
|------|--------------|
| DeepSeek V3 | MLA + MoE |
| DeepSeek V3.2 | MLA + DSA |
| Kimi K2 | MLA + MoE |
| Kimi K2.5 | MLA + MoE + MoonViT |
| Kimi K3 | KDA + Gated MLA |
| GLM 5 | MLA(DSA) + MoE |
| Qwen3-VL | Dense + MoE |

## 보조 도구

저장소는 계산기와 시각화 도구를 함께 제공한다.

| 도구 | 목적 |
|------|------|
| LLM 메모리 계산 공식 | 모델 배포 시 메모리 사용량 추정 |
| MFU 계산기 | 훈련 효율(Model FLOPs Utilization) 측정 |
| DeepSeek V3 MFU 도구 | V3 아키텍처 특화 계산 |
| PyTorch 메모리 시각화 | Snapshot 기반 할당 패턴 분석 |
| pytorch_vista | 계산 그래프 구조 시각화 |

MFU 계산기는 [calvinxky.github.io/mfu_calculation](https://calvinxky.github.io/mfu_calculation){:target="_blank"}에서 사용할 수 있다.
CUDA 관련 실습은 별도 저장소인 [BasicCUDA](https://github.com/CalvinXKY/BasicCUDA){:target="_blank"}에 정리되어 있다.

## 학습 경로

저장소는 다음 순서의 학습 경로를 권장한다.

1. 대규모 모델 추론의 핵심 개념과 용어 정리
2. 추론 병렬 전략(DP/TP/PP/SP/EP) 이해
3. ChunkedPrefill에서 FlashDecoding, Speculative Decoding으로 이어지는 추론 최적화
4. Nano-vLLM에서 vLLM, SGLang으로 이어지는 프레임워크 학습
5. PD 분리, AF 분리, MoE 최적화 등 고급 주제

이 순서는 개념 정의에서 시작해 병렬화 전략, 개별 최적화 기법, 프레임워크 구현, 시스템 수준 설계로 추상화 계층을 올려가는 구조다.

## 결론

InfraTech는 AI 인프라 엔지니어링을 개별 논문이나 프레임워크 문서 단위가 아니라 체계적인 커리큘럼 형태로 정리한 저장소다.
추론 최적화 기법의 원리를 노트북 코드로 검증할 수 있고, 최신 모델의 아키텍처 변화를 MLA, DSA, MoE 같은 구성 요소 단위로 추적할 수 있다는 점이 특징이다.

다만 콘텐츠가 중국어로 작성되어 있어 언어 장벽이 있고, Zhihu 기사에 대한 링크 의존도가 높은 편이다.
코드 예제와 표는 언어와 무관하게 참고 가능하므로, 노트북 위주로 접근하는 것이 실용적이다.

## Reference

- [CalvinXKY/InfraTech](https://github.com/CalvinXKY/InfraTech)
