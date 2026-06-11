---
layout: post
title: "NVIDIA Nemotron-3-Ultra: 550B LatentMoE 추론 모델"
author: 'Juho'
date: 2026-06-10 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI, Benchmark]
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
2. [아키텍처](#아키텍처)
3. [학습 방법](#학습-방법)
4. [벤치마크 성능](#벤치마크-성능)
5. [사용·라이선스](#사용라이선스)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

NVIDIA가 Nemotron-3-Ultra-550B-A55B 모델을 공개했다.
이 모델은 2026년 6월 4일 Hugging Face에 릴리스되었다.
총 파라미터는 550B이며, 추론 시 활성화되는 파라미터는 55B다.
복잡한 추론, 에이전트 워크플로, 장문맥 분석, 도구 사용, RAG에 적합하도록 설계되었다.
컨텍스트 길이는 최대 1M 토큰까지 지원한다.
지원 언어는 영어, 프랑스어, 스페인어, 이탈리아어, 독일어, 일본어, 힌디어, 한국어, 브라질 포르투갈어, 중국어다.
추론 모드는 chat template를 통해 구성할 수 있다.

## 아키텍처

이 모델은 Latent Mixture-of-Experts(LatentMoE) 설계를 채택했다.
interleaved Mamba-2 레이어와 MoE 레이어, 그리고 일부 Attention 레이어를 함께 사용한다.
Multi-Token Prediction(MTP)을 통해 추론을 가속한다.
사전학습에는 NVFP4 레시피가 적용되었다.
총 파라미터는 550B, 활성 파라미터는 55B로 구성된다.

## 학습 방법

학습은 4단계로 진행되었다.

첫 번째는 Pre-training 단계다.
코드, 수학, 과학, 일반 지식 데이터 약 20조(20T) 토큰으로 사전학습했다.

두 번째는 Supervised Fine-tuning 단계다.
추론, 도구 호출, 지시 따르기를 중심으로 미세조정했다.

세 번째는 Reinforcement Learning 단계다.
멀티환경 GRPO와 RLHF로 대화를 정제했다.

네 번째는 Multi-Domain On-Policy Distillation(MOPD) 단계다.
교사 모델을 활용해 코딩, 수학, 에이전트 워크플로 추론을 개선했다.

## 벤치마크 성능

주요 벤치마크 결과는 아래 표와 같다.

| 영역 | 벤치마크 | 점수 |
| --- | --- | --- |
| Agentic | SWE-Bench Verified | 71.9% |
| Reasoning | MMLU-Pro | 86.8% |
| Math | IOI 2025 | 570.0 |
| Long Context | RULER(1M) | 94.7% |
| Multilingual | MMLU-ProX 평균 | 83.0% |

## 사용·라이선스

모델 실행을 위한 최소 하드웨어 요구사항은 다음과 같다.
8x GB200/B200/GB300/B300, 또는 16x H100, 또는 8x H200 구성이 필요하다.
라이선스는 OpenMDW License Agreement v1.1로, 상업 및 비상업 사용을 허용한다.

## 결론

Nemotron-3-Ultra-550B-A55B는 LatentMoE 설계와 Mamba-2, MoE, Attention을 결합한 하이브리드 아키텍처로 구성된 추론 모델이다.
20T 토큰 사전학습부터 SFT, RL, MOPD에 이르는 4단계 학습을 거쳤다.
최대 1M 토큰 컨텍스트와 다국어 지원을 갖춰 추론, 에이전트 워크플로, 장문맥 분석, 도구 사용, RAG 등 다양한 작업에 활용할 수 있다.

## Reference

- [NVIDIA-Nemotron-3-Ultra-550B-A55B-BF16](https://huggingface.co/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B-BF16/){:target="_blank"}
