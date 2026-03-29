---
layout: post
title: "Google TurboQuant - 극한 압축으로 AI 효율성을 재정의하는 양자화 알고리듬"
author: 'Juho'
date: 2026-03-29 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, GPU]
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
2. [방법론](#방법론)
   - [PolarQuant 압축](#polarquant-압축)
   - [QJL 오류 보정](#qjl-오류-보정)
3. [주요 결과](#주요-결과)
   - [KV 캐시 압축 성능](#kv-캐시-압축-성능)
   - [벡터 검색 성능](#벡터-검색-성능)
4. [한계와 주의사항](#한계와-주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google Research가 ICLR 2026에서 TurboQuant를 발표했다.
TurboQuant는 이론적 기반을 갖춘 양자화 알고리듬으로, 대규모 언어 모델(LLM)과 벡터 검색 엔진의 극한 압축을 가능하게 한다.

AI 연산에 필수적인 고차원 벡터는 상당한 메모리를 소비한다.
기존 벡터 양자화 방식은 양자화 상수를 전체 정밀도(full-precision)로 저장해야 하는 "메모리 오버헤드" 문제를 안고 있었다.
TurboQuant는 이 근본적인 문제를 2단계 접근법으로 해결한다.

## 방법론

TurboQuant는 PolarQuant 압축과 QJL 오류 보정이라는 두 단계로 구성된다.
각 단계가 서로 보완적으로 작동하여 극한 압축 비율에서도 정확도를 유지한다.

### PolarQuant 압축

첫 번째 단계인 PolarQuant는 데이터 벡터를 랜덤하게 회전시켜 기하학적 구조를 단순화한다.
이후 표준 양자화를 적용하되, 기존의 카테시안(Cartesian) 좌표 대신 극좌표(polar coordinates)를 사용한다.
극좌표는 반지름(radius)과 각도(angle)로 구성되며, 이를 통해 양자화 상수의 메모리 오버헤드를 크게 줄일 수 있다.

### QJL 오류 보정

두 번째 단계는 Quantized Johnson-Lindenstrauss(QJL) 알고리듬을 활용한 1비트 잔차(residual) 보정이다.
PolarQuant 이후 남아 있는 오류를 제거하기 위해 설계되었다.
Johnson-Lindenstrauss 변환은 고차원 데이터를 저차원으로 투영하면서 거리 관계를 보존하는 고전적 기법이다.
QJL은 이 변환을 1비트 수준으로 양자화하여 추가적인 메모리 부담 없이 잔차 오류를 효과적으로 보정한다.

## 주요 결과

### KV 캐시 압축 성능

TurboQuant는 LLM의 KV 캐시에서 정확도 손실 없이 6배 압축을 달성했다.
학습이나 파인튜닝 없이 3비트 양자화가 가능하다.
H100 GPU 기준으로 32비트 비양자화 대비 최대 8배 성능 향상을 보였다.

다양한 벤치마크에서 검증이 이루어졌다.

| 벤치마크 | 평가 영역 |
|------|------|
| LongBench | 장문 이해 |
| Needle In A Haystack | 정보 검색 정확도 |
| ZeroSCROLLS | 제로샷 장문 이해 |
| RULER | 장문 컨텍스트 처리 |
| L-Eval | 장문 평가 종합 |

검증에 사용된 모델은 Gemma와 Mistral이다.

### 벡터 검색 성능

벡터 검색 영역에서도 TurboQuant는 우수한 성능을 입증했다.
GloVe 데이터셋 기준으로 기존 PQ(Product Quantization) 및 RabbiQ 대비 더 높은 재현율(recall)을 달성했다.

## 한계와 주의사항

커뮤니티에서 일부 주의사항이 제기되었다.
Johnson-Lindenstrauss 변환 자체는 오래전부터 알려진 고전적 기법이라는 점이 지적되었다.
또한 NeurIPS 2021에서 발표된 DRIVE 논문에 대한 인용이 누락되었다는 비판도 있었다.

한편, 이미 llama.cpp에 TurboQuant 구현이 진행 중이라는 점은 실용적 관심이 높다는 것을 보여준다.

## 결론

TurboQuant는 PolarQuant와 QJL이라는 두 단계 접근법으로 LLM과 벡터 검색의 메모리 효율성을 크게 개선했다.
학습 없이 3비트 양자화로 6배 압축과 8배 성능 향상을 달성한 것은 실용적으로 큰 의미가 있다.
ICLR 2026에서 발표되었으며, 이미 오픈소스 생태계(llama.cpp)에서 구현이 시작되어 빠른 확산이 기대된다.

## Reference

- [TurboQuant: Redefining AI Efficiency with Extreme Compression](https://research.google/blog/turboquant-redefining-ai-efficiency-with-extreme-compression/)
