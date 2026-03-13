---
layout: post
title: "Google Gemini Embedding 2 - 최초의 네이티브 멀티모달 임베딩 모델"
author: 'Juho'
date: 2026-03-13 00:00:00 +0900
categories: [AI]
tags: [AI, Embedding, LLM]
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
   - [입력 지원 범위](#입력-지원-범위)
   - [기술 혁신](#기술-혁신)
   - [성능 개선 사례](#성능-개선-사례)
   - [접근성 및 통합 지원](#접근성-및-통합-지원)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google이 텍스트, 이미지, 비디오, 오디오, 문서를 하나의 임베딩 공간에 매핑하는 최초의 완전 멀티모달 임베딩 모델인 [Gemini Embedding 2](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/){:target="_blank"}를 공개했다.
이 모델은 다양한 콘텐츠 유형을 동일한 시맨틱 공간의 임베딩으로 변환하여, 크로스 모달 검색과 멀티모달 시맨틱 검색 등 다양한 활용이 가능하다.

## 배경

기존 임베딩 모델은 텍스트 중심으로 설계되어, 이미지나 비디오 등 다른 모달리티의 콘텐츠를 동일한 시맨틱 공간에서 처리하기 어려웠다.
Gemini Embedding 2는 텍스트, 시각 미디어, 비디오, 오디오, 문서 등 다양한 콘텐츠 유형을 하나의 임베딩 공간으로 통합하여 이 문제를 해결한다.

## 핵심 내용

### 입력 지원 범위

| 모달리티 | 지원 사양 |
|----------|-----------|
| 텍스트 | 최대 8,192 토큰 |
| 이미지 | 요청당 최대 6개 (PNG, JPEG) |
| 비디오 | 최대 120초 (MP4, MOV) |
| 오디오 | 네이티브 임베딩 지원 |
| 문서 | 최대 6페이지 PDF |

### 기술 혁신

Gemini Embedding 2는 100개 이상의 언어에서 시맨틱 의도를 포착한다.
Matryoshka Representation Learning 기법을 적용하여 기본 3,072 차원의 임베딩을 1,536 또는 768 차원으로 유연하게 축소할 수 있다.
이를 통해 성능과 저장 비용 사이에서 유연한 선택이 가능하다.

### 성능 개선 사례

| 기업 | 개선 내용 |
|------|-----------|
| Sparkonomy | 지연 시간 최대 70% 감소, 텍스트-이미지 유사도 점수가 0.4에서 0.8로 향상 |
| Mindlid | 개인 웰니스 앱에서 top-1 리콜 20% 향상 |

### 접근성 및 통합 지원

Gemini Embedding 2는 Gemini API와 Vertex AI를 통해 사용할 수 있다.
LangChain, LlamaIndex, Weaviate 등 주요 프레임워크와의 통합도 지원한다.

## 의미와 시사점

Gemini Embedding 2는 텍스트 설명과 일치하는 이미지를 찾는 크로스 모달 검색, 멀티모달 시맨틱 검색, 다양한 형식을 결합한 문서 및 미디어 분석 등에 활용할 수 있다.
하나의 임베딩 공간에서 여러 모달리티를 처리할 수 있기 때문에, 멀티모달 RAG 파이프라인이나 검색 시스템 구축이 크게 단순화될 것으로 보인다.

## 결론

Google Gemini Embedding 2는 텍스트, 이미지, 비디오, 오디오, 문서를 하나의 시맨틱 공간에 매핑하는 최초의 완전 멀티모달 임베딩 모델이다.
Matryoshka Representation Learning을 통한 유연한 차원 축소와 100개 이상의 언어 지원을 제공하며, 주요 AI 프레임워크와의 통합도 지원한다.

## Reference

- [Gemini Embedding 2](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-embedding-2/)
