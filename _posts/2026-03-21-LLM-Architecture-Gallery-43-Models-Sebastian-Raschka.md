---
layout: post
title: "LLM Architecture Gallery - 43개 LLM 아키텍처를 한눈에 비교하는 갤러리"
author: 'Juho'
date: 2026-03-21 00:00:00 +0900
categories: [LLM]
tags: [LLM, AI]
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
   - [디코더 유형 분류](#디코더-유형-분류)
   - [어텐션 메커니즘](#어텐션-메커니즘)
   - [주요 모델](#주요-모델)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Sebastian Raschka 박사가 공개한 LLM Architecture Gallery는 43개의 LLM 아키텍처를 한눈에 비교할 수 있는 큐레이션 컬렉션이다.
비교 연구 논문에서 추출한 아키텍처 다이어그램, 사양, 팩트시트를 제공하며 원본 소스 링크도 포함되어 있다.
3B부터 1조 파라미터까지, 2019년 11월부터 2026년 3월까지의 모델을 다룬다.

## 배경

LLM 아키텍처는 빠르게 진화하고 있으며, 각 모델마다 다른 설계 결정을 내리고 있다.
Dense Transformer에서 시작하여 Mixture-of-Experts, 하이브리드 아키텍처, 선형 어텐션까지 다양한 접근법이 등장했다.
이러한 아키텍처들을 체계적으로 비교하고 이해하는 것은 연구자와 엔지니어 모두에게 중요하다.

## 핵심 내용

### 디코더 유형 분류

갤러리는 LLM 아키텍처를 세 가지 주요 디코더 유형으로 분류한다.

| 유형 | 대표 모델 |
|------|-----------|
| Dense Transformer | GPT-2 XL, Llama 3, OLMo 시리즈 |
| Sparse Mixture-of-Experts | DeepSeek V3, Qwen3, Grok 2.5 |
| 하이브리드 아키텍처 | Qwen3 Next, Nemotron 3 |

Dense Transformer는 모든 파라미터를 활성화하는 전통적인 방식이다.
Sparse MoE는 입력에 따라 일부 전문가만 활성화하여 효율성을 높인다.
하이브리드 아키텍처는 서로 다른 어텐션 변형을 결합한 최신 접근법이다.

### 어텐션 메커니즘

갤러리에서 다루는 주요 어텐션 메커니즘은 네 가지다.

| 메커니즘 | 약어 | 특징 |
|----------|------|------|
| Multi-Head Attention | MHA | 전통적인 다중 헤드 어텐션 |
| Grouped Query Attention | GQA | 쿼리 그룹화로 메모리 효율 개선 |
| Multi-Head Latent Attention | MLA | 잠재 공간에서의 어텐션 연산 |
| Sliding-window 및 하이브리드 | - | 긴 컨텍스트 처리를 위한 혼합 방식 |

각 모델이 어떤 어텐션 메커니즘을 채택했는지를 팩트시트에서 확인할 수 있다.

### 주요 모델

갤러리에서 특히 주목할 만한 최신 대형 모델들이 있다.

| 모델 | 파라미터 | 특징 |
|------|----------|------|
| DeepSeek V3 | 671B | Sparse MoE 아키텍처 |
| GLM-5 | 744B | 2026년 출시 대형 모델 |
| Qwen3.5 | 397B | 하이브리드 아키텍처 |
| Ling 2.5 | 1T | 선형 어텐션 모델 |

1조 파라미터를 달성한 Ling 2.5는 선형 어텐션을 채택하여 기존 Transformer 대비 계산 효율을 크게 개선했다.

## 의미와 시사점

이 갤러리는 LLM 아키텍처의 진화 흐름을 한눈에 파악할 수 있게 해준다.
Dense에서 Sparse MoE로, 단일 어텐션에서 하이브리드 어텐션으로 향하는 추세가 명확하다.
4편의 주요 연구 논문과 기술 보고서를 기반으로 하며, GitHub 참조를 통해 재현 가능성도 확보하고 있다.

연구자에게는 아키텍처 설계 결정을 비교하는 참고 자료로, 엔지니어에게는 모델 선택 시 아키텍처적 특성을 이해하는 가이드로 활용할 수 있다.

## 결론

LLM Architecture Gallery는 43개 모델의 아키텍처를 체계적으로 비교할 수 있는 귀중한 리소스다.
3B에서 1T까지, 2019년부터 2026년까지의 아키텍처 진화를 팩트시트와 다이어그램으로 제공한다.
LLM 아키텍처를 깊이 이해하고자 하는 연구자와 엔지니어에게 필수적인 참고 자료가 될 것이다.

## Reference

- [LLM Architecture Gallery](https://sebastianraschka.com/llm-architecture-gallery/)
