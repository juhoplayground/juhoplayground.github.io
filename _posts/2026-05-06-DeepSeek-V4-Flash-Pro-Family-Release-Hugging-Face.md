---
layout: post
title: "DeepSeek-V4 패밀리 공개, Flash와 Pro 그리고 1.6T 베이스 모델"
author: 'Juho'
date: 2026-05-06 00:00:00 +0900
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
2. [배경](#배경)
3. [핵심 내용](#핵심-내용)
   - [Flash 라인업](#flash-라인업)
   - [Pro 라인업](#pro-라인업)
   - [Base와 Instruct의 분리](#base와-instruct의-분리)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

DeepSeek가 Hugging Face에 [DeepSeek-V4 컬렉션](https://huggingface.co/collections/deepseek-ai/deepseek-v4){:target="_blank"}을 등록하며 차세대 모델 패밀리를 공개했다.
컬렉션에는 Flash와 Pro 두 라인업이 있고, 각각 Base와 Instruct(이름상 일반 V4-Flash, V4-Pro) 버전으로 구성된다.
가장 큰 V4-Pro-Base는 파라미터 1.6T에 달하며, 가장 작은 V4-Flash도 158B로 이전 V3 시기 대비 큰 폭의 라인업 변화를 보여준다.

## 배경

DeepSeek는 V2와 V3를 거치면서 오픈 가중치 LLM 진영에서 핵심 위치를 차지해왔다.
V4 컬렉션은 컬렉션 페이지 기준 9일 전 갱신되었고, 각 모델은 6일 전에 업로드되었다.
Hugging Face 다운로드와 좋아요 수치를 보면 공개 직후부터 활발하게 사용되고 있음을 확인할 수 있다.

## 핵심 내용

### Flash 라인업

Flash 라인은 추론 비용을 우선시한 작은 모델 트랙이다.
Base는 사전학습 가중치만 제공하며, Instruct에 해당하는 V4-Flash가 텍스트 생성 태스크 용도로 함께 공개된다.

| 모델 | 파라미터 | 용도 | 다운로드 | 좋아요 |
|------|----------|------|----------|--------|
| DeepSeek-V4-Flash-Base | 292B | 사전학습 베이스 | 8.15k | 188 |
| DeepSeek-V4-Flash | 158B | Text Generation | 414k | 921 |

흥미로운 점은 Base의 파라미터(292B)가 Instruct 버전인 V4-Flash(158B)보다 크다는 것이다.
이는 컬렉션 메타데이터에 표시된 수치이며, MoE 구성이나 활성 파라미터 분리 등 실제 아키텍처 차이를 반영한 표기일 가능성이 있다.
정확한 활성 파라미터와 라우팅 구조는 모델 카드 본문 확인이 필요하다.

### Pro 라인업

Pro 라인은 최대 성능을 노리는 고용량 트랙이다.
V4-Pro-Base는 1.6T 파라미터로 컬렉션 내 최대 모델이며, V4-Pro 인스트럭트 모델은 862B 규모로 등록되어 있다.

| 모델 | 파라미터 | 용도 | 다운로드 | 좋아요 |
|------|----------|------|----------|--------|
| DeepSeek-V4-Pro-Base | 1.6T | 사전학습 베이스 | 3.16k | 253 |
| DeepSeek-V4-Pro | 862B | Text Generation | 457k | 3.44k |

다운로드와 좋아요 모두 V4-Pro가 가장 높아 사실상 컬렉션의 대표 모델이다.
좋아요 3.44k는 Hugging Face 신규 공개 모델 중에서도 매우 높은 편에 속한다.

### Base와 Instruct의 분리

V4 컬렉션의 또 다른 특징은 모든 라인업에서 Base와 Instruct를 분리해 공개한다는 점이다.
Base 모델은 추가 사후학습 없이 사전학습만 마친 가중치이며, 자체 파인튜닝이나 RL 후학습을 시도하려는 연구자에게 출발점이 된다.
Instruct(V4-Flash, V4-Pro)는 일반 사용자가 곧바로 텍스트 생성에 사용할 수 있는 형태로 제공된다.
Base의 다운로드 수가 Instruct보다 훨씬 적은 점은 일반 사용자 대다수가 Instruct를 선택한다는 시장 신호로 읽힌다.

## 의미와 시사점

V4 패밀리는 두 축의 분할로 정리된다.
하나는 Flash와 Pro라는 크기/비용 축이고, 다른 하나는 Base와 Instruct라는 학습 단계 축이다.
이 4분할 구조는 OpenAI나 Anthropic이 Tier로 모델을 구분하는 방식과 유사하지만, DeepSeek는 가중치 자체를 공개한다는 점에서 결이 다르다.
1.6T 베이스 모델을 그대로 받아 쓰기는 어렵지만, 학계와 대형 인프라 보유 기업에는 사후학습 실험의 기반이 될 수 있다.
158B Flash는 단일 노드 추론 환경에서 실용적인 옵션으로 자리잡을 가능성이 높다.

## 결론

컬렉션 페이지가 노출하는 정보 자체는 모델명, 파라미터, 다운로드/좋아요 수치 정도로 한정적이다.
라이선스, 정확한 출시일, 벤치마크 점수 같은 세부 사항은 각 모델 카드와 추후 공식 발표를 통해 확인해야 한다.
다만 컬렉션 구성만으로도 DeepSeek가 Flash/Pro × Base/Instruct의 4분할 라인업을 V4의 표준 형태로 가져갔다는 점은 분명하다.

## Reference

- [DeepSeek-V4 Collection on Hugging Face](https://huggingface.co/collections/deepseek-ai/deepseek-v4){:target="_blank"}
