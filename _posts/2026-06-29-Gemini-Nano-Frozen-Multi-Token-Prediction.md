---
layout: post
title: "Frozen Multi-Token Prediction: Pixel에서 Gemini Nano 가속하기"
author: 'Juho'
date: 2026-06-29 00:00:00 +0900
categories: [LLM]
tags: [LLM, GPU, Benchmark]
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
   - [speculative decoding의 한계](#speculative-decoding의-한계)
   - [MTP라는 대안](#mtp라는-대안)
3. [핵심 내용](#핵심-내용)
   - [동결된 백본과 경량 head](#동결된-백본과-경량-head)
   - [Zero-copy KV 캐시 설계](#zero-copy-kv-캐시-설계)
   - [정량 결과](#정량-결과)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google Research가 Pixel 9, Pixel 10 기기에서 Gemini Nano를 가속하는 새로운 추론 기법을 공개했다.
핵심은 Frozen Multi-Token Prediction(MTP)이다.
이미 학습이 끝나 동결된 Gemini Nano v3 백본에 경량 Transformer head만 추가로 붙여, 별도의 드래프트 모델 없이 텍스트 생성을 가속한다.
온디바이스 환경의 메모리 제약을 고려한 zero-copy KV 캐시 설계가 핵심 혁신이며, 출력은 원본 모델과 bit-for-bit 동일하다.
실제 배포 결과 추론 패스당 평균 약 2개의 추가 토큰을 예측하며, 작업에 따라 50% 이상의 속도 향상을 달성했다.

## 배경

온디바이스 LLM은 서버와 달리 메모리와 배터리라는 강한 제약 아래에서 동작해야 한다.
생성 속도를 높이려는 대표적 기법이 speculative decoding이지만, 모바일 환경에서는 그 비용이 만만치 않다.

### speculative decoding의 한계

전통적인 speculative decoding은 두 개의 분리된 구성 요소를 사용한다.
작은 드래프터(drafter) 모델이 후보 토큰을 생성하고, 큰 검증자(verifier) 모델이 이를 병렬로 처리해 검증한다.
이 구조는 모바일에서 두 가지 비효율을 만든다.
첫째, 드래프터와 메인 모델이 서로 메모리를 두고 경쟁한다.
둘째, 드래프터는 메인 모델이 이미 계산한 의미 표현(semantic representation)에 접근하지 못하는 컨텍스트 단절(context blindness)을 겪는다.

### MTP라는 대안

MTP는 분리형 구조 대신 통합형 구조로 전환한다.
백본의 마지막 레이어들에 경량 Transformer head를 덧붙여, 드래프트를 담당하는 구성 요소가 메인 모델이 이미 계산해 둔 풍부한 내부 표현을 그대로 활용하게 한다.
즉 별도 드래프터가 처음부터 다시 컨텍스트를 이해할 필요가 없다.

## 핵심 내용

### 동결된 백본과 경량 head

Gemma 4처럼 사전학습 단계에서 MTP head를 함께 학습하는 방식과 달리, 이번 접근은 완전히 학습된 Gemini Nano v3 백본을 동결(frozen) 상태로 유지한다.
새로 붙인 MTP head의 파라미터만 학습한다.
이 설계 덕분에 MTP는 순수하게 효율성 최적화로만 작동하며, 모델의 능력이나 안전성(safety) 속성에 어떤 저하도 일으키지 않는다.
또한 기존 모델과의 하위 호환성(backward compatibility)도 확보된다.

### Zero-copy KV 캐시 설계

가장 중요한 혁신은 모바일 메모리 제약을 정면으로 다루는 zero-copy KV 캐시 구조다.
표준 MTP 구현에서는 드래프터가 자신만의 key-value 캐시를 따로 유지해야 하므로 메모리가 중복 소비된다.
반면 이번 설계에서는 MTP head가 메인 모델의 동결된 KV 캐시에 직접 cross-attention을 수행한다.
캐시를 복제하지 않고 그대로 참조하는 것이다.
이로부터 두 가지 이점이 나온다.

| 이점 | 설명 |
|------|------|
| 드래프터 prefill latency 제거 | 캐시가 이미 존재하므로 프롬프트 컨텍스트를 위한 추가 처리가 필요 없다 |
| 메모리 footprint 감소 | 별도 임베딩 룩업 테이블과 중복 계산 파라미터를 없애 인스턴스당 약 130MB를 절감한다 |

이 설계의 또 다른 핵심은 정확성 보장이다.
검증 과정에서 잘못된 드래프트는 폐기되므로, 최종 출력은 메인 모델과 bit-for-bit 동일하다.
원본 모델과 정확히 동일한 의미적 결과가 보존된다.

### 정량 결과

MTP는 유사한 파라미터 수의 단독 드래프터보다 더 풍부한 표현에 접근할 수 있어 예측 성능이 우수하다.
지시 수행(instruction-following) 계열 작업인 요약, 재작성에서 단독 fine-tuned 대안을 크게 앞섰다.
스마트 답장처럼 구조가 예측 가능한 작업에서는 토큰 수용률(token acceptance rate)이 최대 55%까지 향상됐다.

Pixel 9, Pixel 10 기기에서의 실제 배포 결과는 다음과 같다.

| 지표 | 결과 |
|------|------|
| 추가 토큰 예측 | 추론 패스당 평균 약 2개의 추가 토큰 |
| 속도 향상 | 작업에 따라 50% 이상 |
| 에너지 | 프로세서 wake 사이클 감소로 에너지 소비와 배터리 소모 절감 |
| 사용자 기능 | AI 알림 요약, Proofread 등에서 텍스트 생성이 눈에 띄게 빨라짐 |

## 의미와 시사점

이번 기법은 온디바이스 LLM 가속에서 두 가지 통념을 바꾼다.
첫째, 가속을 위해 반드시 별도의 드래프트 모델을 추가로 적재해야 한다는 전제를 깬다.
동결된 백본에 가벼운 head만 붙이고 KV 캐시를 공유하면, 메모리 경쟁 없이도 속도를 얻을 수 있다.
둘째, 가속과 품질 보존이 트레이드오프 관계라는 통념을 깬다.
출력이 원본과 bit-for-bit 동일하므로, 속도 향상이 모델의 능력이나 안전성을 희생하지 않는다.
모바일에서 인스턴스당 약 130MB 절감과 prefill latency 제거는 배터리와 발열까지 함께 개선하는 실질적 효과로 이어진다.

향후 연구 방향으로는 병렬 디코딩(parallel decoding) 패러다임, 불확실한 컨텍스트에서의 분기 가능성 탐색, 특정 사용 사례를 위한 검증 완화(verification leniency) 기법 등이 제시됐다.

## 결론

Frozen Multi-Token Prediction은 동결된 Gemini Nano v3 백본에 경량 Transformer head를 추가하고, 메인 모델의 KV 캐시를 zero-copy로 재사용하는 방식으로 온디바이스 추론을 가속한다.
별도 드래프터를 두지 않아 메모리 경쟁과 컨텍스트 단절을 피하고, prefill latency를 없애며 인스턴스당 약 130MB를 절감한다.
출력은 원본과 bit-for-bit 동일하게 유지되고, Pixel 9, Pixel 10에서 추론 패스당 평균 약 2개의 추가 토큰 예측과 50% 이상의 속도 향상을 달성했다.
효율성과 품질 보존을 동시에 만족시키는 실용적인 온디바이스 가속 사례다.

## Reference

- [Accelerating Gemini Nano models on Pixel with frozen multi-token prediction](https://research.google/blog/accelerating-gemini-nano-models-on-pixel-with-frozen-multi-token-prediction/)
