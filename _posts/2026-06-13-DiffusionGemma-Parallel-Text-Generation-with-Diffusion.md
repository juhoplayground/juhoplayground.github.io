---
layout: post
title: "DiffusionGemma - 병렬 텍스트 생성과 Diffusion 기반 언어 모델"
author: 'Juho'
date: 2026-06-13 00:00:00 +0900
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
2. [배경: 자기회귀 모델의 한계](#배경-자기회귀-모델의-한계)
3. [Diffusion 기반 텍스트 생성](#diffusion-기반-텍스트-생성)
   - [이미지 Diffusion과의 비교](#이미지-diffusion과의-비교)
   - [Masked Diffusion vs Uniform State Diffusion](#masked-diffusion-vs-uniform-state-diffusion)
4. [DiffusionGemma 아키텍처](#diffusiongemma-아키텍처)
   - [인코더-디노이저 설계](#인코더-디노이저-설계)
   - [추론 메커니즘](#추론-메커니즘)
5. [성능 및 벤치마크](#성능-및-벤치마크)
6. [자기회귀 vs Diffusion 비교](#자기회귀-vs-diffusion-비교)
7. [가용성 및 지원 환경](#가용성-및-지원-환경)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Google은 텍스트 Diffusion 방식을 채용한 실험적 오픈 모델 [DiffusionGemma](https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/){:target="_blank"}를 발표했다.
DiffusionGemma는 Apache 2.0 라이선스로 공개된 26B MoE(Mixture of Experts) 모델로, 기존 자기회귀 방식과 달리 256개 토큰을 동시에 생성하는 병렬 처리 방식을 채택한다.
GPU 환경에서 최대 4배 빠른 텍스트 생성 속도를 달성하며, NVIDIA H100에서 1,000 토큰/초 이상의 성능을 기록했다.

## 배경: 자기회귀 모델의 한계

기존 LLM은 토큰을 순차적으로 하나씩 생성하는 자기회귀(autoregressive) 방식을 사용한다.
이 방식은 다수의 사용자에게 동시에 서비스할 때는 효율적이지만, 단일 사용자 시나리오에서는 메모리 대역폭에 병목이 발생한다.
단일 사용자 기준으로는 아무리 많은 컴퓨팅 자원을 투입해도 지연 시간(latency)이 개선되지 않는 구조적 한계가 존재한다.
DiffusionGemma는 이 문제를 해결하기 위해, 유휴 상태의 컴퓨팅 자원을 개별 사용자의 생성 속도 향상에 활용하는 Diffusion 방식을 도입했다.

## Diffusion 기반 텍스트 생성

### 이미지 Diffusion과의 비교

이미지 생성 Diffusion 모델은 완전한 노이즈에서 시작하여 프롬프트에 의해 유도되면서 점진적으로 노이즈를 제거(denoising)하는 과정을 반복한다.
Forward Diffusion 단계에서 훈련 이미지에 Gaussian 노이즈를 추가하고, Reverse Diffusion 단계에서 모델이 추가된 노이즈를 예측하고 제거하도록 학습한다.
텍스트 생성에서도 동일한 원리를 적용하여, 무작위 토큰으로 채워진 캔버스에서 시작해 반복적으로 정제해 나가는 방식을 사용한다.

### Masked Diffusion vs Uniform State Diffusion

텍스트 Diffusion에는 두 가지 접근 방식이 있다.

Masked Diffusion은 토큰을 [MASK] 토큰으로 교체하는 방식이다.
그러나 한번 확정된 예측값을 수정할 수 없어, 자기회귀 모델과 유사한 '되돌리기 불가' 문제가 발생한다.

Uniform State Diffusion은 토큰을 무작위 대안 토큰으로 교체하는 방식이다.
모델이 스스로 노이즈를 감지할 수 있으며, 이전에 선택된 토큰도 반복 과정에서 지속적으로 수정할 수 있다.
DiffusionGemma는 Uniform State Diffusion을 채택하여 디노이징 전 과정에 걸쳐 자기 수정(self-correction)이 가능하도록 설계되었다.

## DiffusionGemma 아키텍처

### 인코더-디노이저 설계

DiffusionGemma는 Gemma 4 26B A4B(MoE 모델)를 기반으로 구축되었으며, 인코더 모드와 디노이저 모드 두 가지로 동적으로 전환된다.

디노이저 모드에서는 causal attention을 양방향(bidirectional) attention으로 전환하고, 노이즈가 포함된 캔버스 전체를 동시에 처리한다.
표준 LLM이 마지막 토큰에 대한 logit만 출력하는 것과 달리, 모든 캔버스 위치에 대해 logit을 생성한다.

인코더 모드에서는 사용자 쿼리를 표준 방식으로 처리하며, Key-Value 캐시를 디노이저와 공유한다.
별도의 cross-attention 수정 없이 디노이저에게 의미적 가이드를 제공한다.

26B 전체 파라미터 중 실제 활성화되는 파라미터는 3.8B에 불과하며, 양자화 시 18GB VRAM에서 동작한다.

### 추론 메커니즘

DiffusionGemma의 추론 과정은 여러 세부 구성 요소로 이루어진다.

Self-Conditioning은 이전 단계의 예측값을 다음 단계에 전달하는 메커니즘이다.
Softmax 확률과 토큰 임베딩을 곱한 확률 가중 표현을 작은 피드포워드 네트워크를 통해 다음 단계로 전달한다.

Multi-Canvas Sampling(Block Diffusion)은 256토큰 캔버스 한계를 초과하는 긴 출력을 처리하는 방법이다.
Diffusion으로 256개 토큰을 생성한 뒤 프롬프트에 추가하고, 인코더 KV 캐시를 재사용하면서 반복한다.
이 하이브리드 접근 방식은 KV 캐시를 매번 재계산하지 않고 재사용하여 연산 효율성을 유지한다.

스케줄러 시스템은 세 가지 구성 요소로 이루어진다.
Step Count는 최대 디노이징 반복 횟수를 제어하며, 횟수가 많을수록 품질이 높아지지만 속도는 느려진다.
Logits Scheduler는 온도 기반 샤프닝을 적용하며, 초기 단계에서 탐색을 넓게 하고 후기 단계에서 수렴을 유도한다.
Adaptive Stopping은 N 스텝 동안 동일한 예측이 유지되거나(안정성), 엔트로피 기반 신뢰도 임계값(기본 0.005)을 충족하면 조기 종료한다.

Entropy Bounded Sampler는 토큰 수용 여부를 결정하는 구성 요소다.
초기화 시 무작위 균등 토큰을 선택하고, 엔트로피 기반 신뢰도 점수로 토큰 그룹을 수용하거나, 임계값 미충족 토큰을 무작위 대안으로 재노이즈화한다.

## 성능 및 벤치마크

DiffusionGemma는 기존 자기회귀 모델 대비 최대 4배 빠른 텍스트 생성 속도를 달성한다.
NVIDIA H100에서 1,000 토큰/초 이상, RTX 5090에서 700 토큰/초 이상의 성능을 기록했다.
디코딩 병목을 메모리 대역폭 제한에서 컴퓨팅 제한으로 전환함으로써 고성능 GPU의 이점을 최대한 활용할 수 있게 된다.
Gemma 4 기반의 파라미터 효율성을 바탕으로 26B 규모에서도 3.8B만 활성화하여 연산을 수행한다.

## 자기회귀 vs Diffusion 비교

두 방식의 주요 차이점을 정리하면 다음과 같다.

| 항목 | 자기회귀 | Diffusion |
|------|----------|-----------|
| 처리 방식 | 포워드 패스당 토큰 1개 | 다수 토큰 동시 처리 |
| 병목 | 메모리 대역폭 한정 | 컴퓨팅 한정 |
| 지연 시간 | 다수 동시 사용자에 유리 | 단일 사용자 지연 시간에 유리 |
| 자기 수정 | 확정된 토큰 수정 불가 | 반복 과정에서 지속 수정 가능 |
| 훈련 복잡도 | 표준 | 노이즈 식별 학습 필요 |

## 가용성 및 지원 환경

DiffusionGemma는 Hugging Face에서 모델 가중치를 Apache 2.0 라이선스로 공개했다.
MLX, vLLM, Hugging Face Transformers를 통해 사용할 수 있으며, llama.cpp 지원도 예정되어 있다.
클라우드 환경에서는 Google Cloud와 NVIDIA NIM을 통한 배포가 가능하다.

## 결론

DiffusionGemma는 기존 자기회귀 방식의 단일 사용자 지연 시간 한계를 Diffusion 기반 병렬 토큰 생성으로 극복한 실험적 모델이다.
Uniform State Diffusion, Self-Conditioning, Block Diffusion, Entropy Bounded Sampler 등 다양한 기법을 결합하여 품질과 속도를 동시에 추구한다.
Diffusion과 자기회귀가 상호 배타적이지 않으며 실용적으로 결합될 수 있음을 보여주는 사례로, 고성능 GPU 환경에서 단일 사용자 서비스의 응답 속도 향상에 새로운 방향을 제시한다.

## Reference

- [A Visual Guide to DiffusionGemma](https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-diffusiongemma/)
- [DiffusionGemma: 4x faster text generation](https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/)
