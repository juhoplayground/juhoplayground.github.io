---
layout: post
title: "Gemma 4 Multi-Token Prediction - 품질 손실 없이 최대 3배 추론 가속"
author: 'Juho'
date: 2026-05-10 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, GPU]
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
2. [배경](#배경)
3. [핵심 내용](#핵심-내용)
   - [MTP 동작 원리](#mtp-동작-원리)
   - [성능 측정 결과](#성능-측정-결과)
   - [지원 환경](#지원-환경)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google이 Gemma 4 패밀리를 위한 Multi-Token Prediction(MTP) 드래프터를 공개했다.
이 기술은 출력 품질이나 추론 로직의 저하 없이 최대 3배의 속도 향상을 제공하는 것이 특징이다.
MTP는 speculative decoding(추측 디코딩) 기법을 활용하여 추론을 가속하며, 특히 소비자급 하드웨어에서의 지연 시간 문제를 해결하기 위해 설계되었다.

## 배경

표준 LLM 추론에는 결정적인 병목 구간이 존재한다.
"프로세서가 단일 토큰 하나를 생성하기 위해 수십억 개의 파라미터를 VRAM에서 컴퓨트 유닛으로 옮기는 데 대부분의 시간을 쓴다"는 메모리 대역폭 병목이다.
이 구조 때문에 컴퓨트 자원은 제대로 활용되지 못하고, 사용자 경험을 좌우하는 지연 시간은 길어진다.
특히 소비자 하드웨어처럼 메모리 대역폭이 제한된 환경에서 이 문제는 더 두드러진다.

## 핵심 내용

### MTP 동작 원리

MTP는 큰 타깃 모델(예: Gemma 4 31B)과 가벼운 드래프터를 한 쌍으로 묶는 구조이다.
드래프터는 타깃 모델이 토큰 하나를 처리하는 동안 유휴 상태로 남는 컴퓨트 자원을 활용해 미래 토큰 여러 개를 동시에 예측한다.
타깃 모델은 그 다음 단계에서 드래프터가 제안한 토큰들을 병렬로 검증한다.

핵심 동작은 다음과 같이 정리된다.

| 단계 | 역할 |
|------|------|
| 드래프트 | 가벼운 드래프터가 다음 토큰들을 미리 예측 |
| 검증 | 타깃 모델이 제안된 토큰들을 단일 forward pass로 병렬 검증 |
| 수락 | 동의하는 시퀀스 전체를 한 번에 수락하고, 추가로 자체 토큰 한 개를 생성 |

타깃 모델이 드래프트에 동의하면 시퀀스 전체를 한 번의 forward pass에서 수락하며, 그 과정에서 자체적으로도 토큰 하나를 추가 생성한다.
이는 단순한 추측 디코딩보다 한 단계 더 효율적인 동작이다.

### 성능 측정 결과

Google은 여러 추론 프레임워크에서 MTP의 가속 효과를 측정했다.

| 환경 | 가속 효과 |
|------|-----------|
| LiteRT-LM, MLX, Hugging Face Transformers, vLLM | 최대 3배 속도 향상 |
| Apple Silicon (배치 4-8) | 약 2.2배 |
| Nvidia A100 | 배치 처리량 증가 환경에서 유사한 수준 |

품질 측면에서는 출력 결과가 동일하게 유지된다.
"Zero quality degradation"이 명시되어 있으며, 동일한 출력 품질을 보장한다.

### 지원 환경

MTP 드래프터는 Apache 2.0 라이선스로 공개되어 있어 상업적 활용이 가능하다.
다양한 배포 채널과 추론 프레임워크에서 사용할 수 있다.

| 카테고리 | 채널 |
|----------|------|
| 모델 배포 | Hugging Face, Kaggle |
| 추론 프레임워크 | Transformers, MLX, vLLM, SGLang, Ollama |
| 모바일 | Google AI Edge Gallery |

## 의미와 시사점

이번 발표의 핵심은 "품질 저하 0"과 "최대 3배 가속"이 동시에 달성된다는 점이다.
speculative decoding 자체는 새로운 개념이 아니지만, Google이 Gemma 4 모델 패밀리에 맞춘 전용 드래프터를 직접 학습해 배포함으로써 일반 사용자가 별도 학습 없이 곧바로 가속 효과를 누릴 수 있게 되었다.

특히 Apple Silicon에서 약 2.2배의 가속이 측정된 점은 로컬 LLM 사용 환경에서 의미가 크다.
배치가 작고 메모리 대역폭이 제한된 환경에서 추론 속도를 끌어올리는 것은 온디바이스 LLM의 실용성을 결정짓는 핵심 요소이기 때문이다.
A100 같은 고성능 환경에서도 배치 처리량이 늘어나는 시나리오에서 유사한 가속을 보여준다는 점에서, 클라우드와 로컬 양쪽 모두에 효용이 있다.

## 결론

Gemma 4의 Multi-Token Prediction 드래프터는 메모리 대역폭 병목이라는 LLM 추론의 본질적 한계를 speculative decoding 방식으로 우회한다.
타깃 모델이 한 토큰을 처리하는 동안 드래프터가 여러 토큰을 미리 예측하고, 타깃 모델이 이를 병렬 검증하는 구조이다.
LiteRT-LM, MLX, Transformers, vLLM 등 다양한 환경에서 최대 3배의 속도 향상이 측정되었으며, 품질 저하는 발생하지 않는다.
Apache 2.0 라이선스로 공개되어 있어 즉시 도입이 가능하다.

## Reference

- [Multi-token prediction for Gemma 4 (Google Blog)](https://blog.google/innovation-and-ai/technology/developers-tools/multi-token-prediction-gemma-4/)
