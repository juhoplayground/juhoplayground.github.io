---
layout: post
title: "PixelRAG: 문서를 스크린샷으로 검색하는 픽셀 네이티브 RAG"
author: 'Juho'
date: 2026-06-18 00:00:00 +0900
categories: [AI]
tags: [AI, Embedding, FAISS, FastAPI]
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
2. [PixelRAG가 푸는 문제](#pixelrag가-푸는-문제)
3. [아키텍처와 동작 방식](#아키텍처와-동작-방식)
   - [핵심 구성 요소](#핵심-구성-요소)
   - [모듈형 파이프라인](#모듈형-파이프라인)
4. [사용 방법](#사용-방법)
   - [호스팅 인덱스 검색](#호스팅-인덱스-검색)
   - [페이지 렌더링과 Claude 플러그인](#페이지-렌더링과-claude-플러그인)
5. [모델과 학습 데이터](#모델과-학습-데이터)
6. [기술 세부 정보](#기술-세부-정보)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

PixelRAG는 문서를 스크린샷으로 렌더링하고, 파싱된 텍스트가 아니라 이미지를 검색하여 정보를 찾는 오픈소스 시스템입니다.
저장소는 이를 "웹 파싱의 종말, 확장 가능한 픽셀 네이티브 검색의 시작"이라고 표현합니다.
핵심 아이디어는 웹 페이지를 텍스트로 변환하는 대신 시각적 타일로 캡처하는 것입니다.
이 방식은 기존 HTML 파싱이 버리는 레이아웃, 표, 차트, 인포그래픽을 그대로 보존합니다.

PixelRAG는 [StarTrail-org/PixelRAG](https://github.com/StarTrail-org/PixelRAG){:target="_blank"} 저장소에서 공개되었으며, 라이선스는 Apache-2.0입니다.

## PixelRAG가 푸는 문제

전통적인 RAG 시스템은 문서를 텍스트로 파싱하면서 시각적 구조를 잃어버립니다.
문서는 이를 구체적인 예시로 설명합니다.
텍스트 기반 RAG는 페이지를 텍스트 청크로 파싱하면서 표를 잃어버리고, 그 결과 읽는 쪽은 답을 찾을 수 없게 됩니다.

PixelRAG는 시각 정보를 보존하여 이미지에서 직접 답을 추출할 수 있도록 합니다.
표, 차트, 인포그래픽처럼 텍스트 변환 과정에서 사라지던 정보가 검색 단계까지 그대로 유지됩니다.

## 아키텍처와 동작 방식

### 핵심 구성 요소

PixelRAG의 파이프라인은 네 단계로 구성됩니다.

| 단계 | 설명 |
|------|------|
| Rendering | Playwright/CDP를 사용해 문서를 스크린샷 타일로 변환 |
| Embedding | Qwen3-VL-Embedding-2B 모델을 스크린샷 데이터로 LoRA 파인튜닝하여 임베딩 생성 |
| Indexing | 벡터 검색을 위한 FAISS 인덱스 구축 |
| Serving | 검색을 위한 FastAPI 엔드포인트 제공 |

임베딩 모델은 웹페이지 스크린샷에 특화되어 학습되었습니다.
그 결과 시각적 콘텐츠가 효과적으로 검색되는 의미 공간을 만들어냅니다.

### 모듈형 파이프라인

각 단계는 독립적으로 동작하도록 설계되어, 필요한 부분만 설치할 수 있습니다.

| 명령 | 목적 | 설치 |
|------|------|------|
| pixelshot | 문서를 이미지 타일로 변환 | pip install pixelrag |
| pixelrag embed | 타일을 벡터로 변환 | pip install 'pixelrag[embed]' |
| pixelrag index | 전체 오케스트레이션 | pip install 'pixelrag[index]' |
| pixelrag serve | 검색 API | pip install 'pixelrag[serve]' |

학습은 train/ 디렉터리의 별도 uv 프로젝트로 분리되어 있습니다.
의존성은 torch 2.9.1+cu129, transformers 4.57.1 버전으로 고정되어 있습니다.

## 사용 방법

### 호스팅 인덱스 검색

PixelRAG는 별도 설정 없이 사용할 수 있는 사전 구축 Wikipedia 인덱스를 제공합니다.
이 인덱스는 828만 개의 Wikipedia 페이지를 포함하며, 호스팅 API를 통해 검색할 수 있습니다.

```bash
curl -X POST https://api.pixelrag.ai/search \
  -H "Content-Type: application/json" \
  -d '{"queries": [{"text": "What is the capital of France?"}], "n_docs": 5}'
```

검색은 텍스트 쿼리와 이미지 쿼리를 모두 지원합니다.

### 페이지 렌더링과 Claude 플러그인

특정 페이지를 직접 이미지 타일로 렌더링할 수 있습니다.

```bash
pixelshot https://en.wikipedia.org/wiki/Python --output ./tiles
```

PixelRAG는 Claude Code용 pixelbrowse 스킬로도 제공됩니다.

```bash
pip install pixelrag
claude plugin marketplace add StarTrail-org/PixelRAG
claude -p "screenshot https://news.ycombinator.com and summarize stories"
```

## 모델과 학습 데이터

PixelRAG는 임베딩에 Qwen/Qwen3-VL-Embedding-2B 모델을 LoRA로 파인튜닝하여 사용합니다.

| 항목 | 값 |
|------|-----|
| 임베딩 모델 | Qwen/Qwen3-VL-Embedding-2B (LoRA 파인튜닝) |
| 학습된 어댑터 | Chrisyichuan/wiki-screenshot-embedding-lora |
| 학습 데이터셋 | Chrisyichuan/screenshot-training-natural-filtered-v2 |

학습 데이터셋은 다른 모델에 적용할 수 있도록 공개되었습니다.

## 기술 세부 정보

PixelRAG의 주요 기술 사양은 다음과 같습니다.

| 항목 | 값 |
|------|-----|
| 사전 구축 Wikipedia 인덱스 크기 | 약 217GB |
| Wikipedia 페이지 수 | 828만 개 |
| 라이선스 | Apache-2.0 |
| 주요 언어 구성 | Python 73.7%, Markdown 16%, TypeScript 6.7% |

시스템은 각 단계가 독립적으로 동작하도록 설계되어 있습니다.
사용자는 호스팅 API를 활용하거나, 자신의 문서로 직접 인덱스를 구축할 수 있습니다.
범용 파이프라인으로 웹 페이지, PDF, 이미지를 모두 처리할 수 있습니다.

## 결론

PixelRAG는 문서를 텍스트로 파싱하던 기존 RAG의 한계를 시각적 검색으로 대체하려는 시도입니다.
문서를 스크린샷 타일로 렌더링하고, 시각 임베딩 모델과 FAISS 인덱스로 검색하며, FastAPI로 서빙하는 구조를 갖췄습니다.
사전 구축된 828만 페이지 규모의 Wikipedia 인덱스와 호스팅 API를 통해 별도 설정 없이 바로 사용할 수 있습니다.
표나 차트처럼 텍스트 변환에서 사라지던 정보를 보존해야 하는 검색 작업에서 의미 있는 접근법을 제시합니다.

## Reference

- [StarTrail-org/PixelRAG](https://github.com/StarTrail-org/PixelRAG/)
