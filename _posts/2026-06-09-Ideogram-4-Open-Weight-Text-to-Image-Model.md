---
layout: post
title: "Ideogram 4: 첫 오픈 웨이트 텍스트-이미지 파운데이션 모델"
author: 'Juho'
date: 2026-06-09 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Benchmark]
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
3. [아키텍처: 단일 스트림 DiT](#아키텍처-단일-스트림-dit)
4. [핵심 역량](#핵심-역량)
5. [nf4 양자화](#nf4-양자화)
6. [벤치마크 성능](#벤치마크-성능)
7. [라이선스와 사용](#라이선스와-사용)
8. [의미와 시사점](#의미와-시사점)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

Ideogram 4는 Ideogram이 공개한 첫 오픈 웨이트 텍스트-이미지 모델이다.
기존 모델을 파인튜닝한 것이 아니라 처음부터 학습한 SOTA 파운데이션 모델이라는 점이 핵심이다.
93억(9.3B) 파라미터 규모의 모델로, 이미지 내 텍스트 렌더링과 구조화된 제어에서 강점을 보인다.

## 배경

텍스트-이미지 생성 모델은 그동안 폐쇄형 API 형태로 제공되는 경우가 많았다.
Ideogram 4는 이 흐름과 달리 가중치를 공개하는 방향을 택했다.
또한 단순히 기존 모델을 미세 조정한 것이 아니라 처음부터 학습한 파운데이션 모델이라는 점에서 차별화된다.

## 아키텍처: 단일 스트림 DiT

Ideogram 4는 완전 단일 스트림(single-stream) Diffusion Transformer(DiT) 구조를 채택했다.
34개 레이어로 구성되며, 텍스트 토큰과 이미지 토큰을 하나의 통합 시퀀스로 처리한다.

전통적인 텍스트 인코더를 사용하는 대신 Qwen3-VL-8B-Instruct라는 비전-언어 모델을 활용한다.
이 비전-언어 모델의 13개 중간 레이어에서 특징을 추출해 더 풍부한 시각적 이해를 확보한다.

## 핵심 역량

Ideogram 4의 주요 역량은 다음 세 가지로 요약된다.

텍스트 렌더링: 간판, 로고, 캡션 등 이미지 내부의 텍스트 생성에서 탁월한 성능을 보인다.

구조화된 JSON 제어: JSON 캡션으로 학습되어 구성, 색상 팔레트, 바운딩 박스 기반 공간 레이아웃을 명시적으로 제어할 수 있다.

유연한 해상도: 256~2048 픽셀 사이의 임의 해상도(16의 배수)를 지원하며, 종횡비는 최대 6:1까지 지원한다.

## nf4 양자화

nf4 변형은 93억(9.3B) 파라미터에 nf4 가중치 양자화를 적용한 버전이다.
CUDA 하드웨어와 diffusers 통합을 지원한다.

## 벤치마크 성능

Ideogram 4는 여러 평가에서 오픈 웨이트 모델 최상위 성능을 기록했다.

평가 결과 요약

| 평가 | 결과 |
| --- | --- |
| Design Arena 리더보드 | 오픈 웨이트 최상위 |
| LMArena 리더보드 | 오픈 웨이트 최상위 |
| ContraLabs 전문 타이포그래피 평가 | 47.9% 1위 |

9.3B 파라미터 오픈 모델 중에서는 최고의 텍스트 렌더링 효율을 보였다.
특히 ContraLabs 디자이너의 전문 타이포그래피 평가에서 47.9%로 1위를 차지했다.

## 라이선스와 사용

Ideogram 4는 "Ideogram 4 Non-Commercial" 라이선스로 배포된다.
사용을 위해서는 Hugging Face 인증과 라이선스 동의가 필요한 gated access 형태다.

## 의미와 시사점

Ideogram 4는 텍스트-이미지 분야에서 오픈 웨이트 파운데이션 모델의 새로운 기준을 제시한다.
단일 스트림 DiT와 비전-언어 모델 기반 특징 추출을 결합해 시각적 이해를 강화한 점이 특징이다.
JSON 기반 공간 제어와 강력한 텍스트 렌더링은 디자인, 로고, 타이포그래피 작업에 직접 활용될 수 있는 실용적 강점이다.
다만 Non-Commercial 라이선스와 gated access는 상업적 활용 범위에 제약을 둔다.

## 결론

Ideogram 4는 처음부터 학습한 9.3B 파라미터 규모의 오픈 웨이트 텍스트-이미지 파운데이션 모델이다.
34 레이어 단일 스트림 DiT 아키텍처와 Qwen3-VL-8B-Instruct 기반 특징 추출을 통해 텍스트 렌더링과 구조화된 제어에서 강점을 보인다.
Design Arena, LMArena, ContraLabs 평가에서 오픈 웨이트 최상위 성능을 입증했다.

## Reference

- [ideogram-4-nf4 Hugging Face Model Card](https://huggingface.co/ideogram-ai/ideogram-4-nf4/)
