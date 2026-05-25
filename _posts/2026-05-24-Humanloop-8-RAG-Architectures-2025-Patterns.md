---
layout: post
title: "Humanloop가 정리한 2025 RAG 아키텍처 8가지 — Simple부터 Agentic까지"
author: 'Juho'
date: 2026-05-24 00:00:00 +0900
categories: [LangChain]
tags: [LangChain, LLM, Knowledge]
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
2. [8가지 RAG 아키텍처](#8가지-rag-아키텍처)
   - [Simple RAG](#simple-rag)
   - [Simple RAG with Memory](#simple-rag-with-memory)
   - [Branched RAG](#branched-rag)
   - [HyDe](#hyde)
   - [Adaptive RAG](#adaptive-rag)
   - [Corrective RAG](#corrective-rag)
   - [Self-RAG](#self-rag)
   - [Agentic RAG](#agentic-rag)
3. [RAG와 파인튜닝의 구분](#rag와-파인튜닝의-구분)
4. [패턴 선택 가이드](#패턴-선택-가이드)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Humanloop는 2025년 RAG(Retrieval-Augmented Generation) 아키텍처 패턴 여덟 가지를 분류해서 정리했다.
같은 RAG라는 이름 아래 들어가 있어도, 구조는 단순한 정적 DB 조회부터 멀티 에이전트 오케스트레이션까지 폭넓게 갈라진다.
아키텍처 선택은 결국 질의 복잡도, 데이터 소스 수, 답변 품질 요구치에 따라 달라진다.

## 8가지 RAG 아키텍처

### Simple RAG

가장 기본적인 형태다.
정적 데이터베이스에서 관련 문서를 검색해 모델에 함께 넣어 준다.
문서 집합이 제한적이고 변동이 적은 FAQ 시스템 같은 곳에 적합하다.

### Simple RAG with Memory

Simple RAG에 영속적 컨텍스트 저장소를 더한 형태다.
다중 턴 대화를 지원하고, 사용자별 개인화된 인터랙션이 필요한 챗봇에 사용된다.

### Branched RAG

질의를 평가한 뒤 특정 데이터 소스를 선택적으로 조회한다.
가용한 소스 전부를 한 번에 뒤지지 않고, 필요한 분기만 타고 들어간다.
도메인이 명확히 나뉜 멀티 소스 환경에 잘 맞는다.

### HyDe

Hypothetical Document Embedding의 줄임말이다.
검색 단계 전에 가상의 이상적인 문서를 먼저 생성하고, 그 임베딩을 기반으로 실제 검색을 수행한다.
질의가 모호하거나 탐색 성격이 강한 리서치 질문에 유용하다.

### Adaptive RAG

질의 복잡도에 따라 검색 전략을 동적으로 조정한다.
간단한 질문에는 단일 소스, 복잡한 질문에는 복수 소스 검색으로 분기한다.
같은 시스템 안에서 단순/복잡 질의를 모두 다뤄야 할 때 적합하다.

### Corrective RAG

CRAG라고도 한다.
"knowledge strip" 단위로 검색 결과를 잘라낸 뒤, 자체 평가(self-grading)로 품질을 매긴다.
원문은 다음과 같이 설명한다.
"어떤 strip도 관련성 임계값을 통과하지 못하면, 모델은 추가 정보를 찾는다. 종종 웹 검색을 활용한다."
검색 품질이 답변 품질을 좌우하는 환경에 적합한 패턴이다.

### Self-RAG

생성 도중에 모델이 스스로 추가 검색 쿼리를 만들어낸다.
정보 갭이 발견되면 자체적으로 메우는 구조다.
탐색형 리서치 과제처럼 답변 과정 자체가 다단계인 경우에 어울린다.

### Agentic RAG

Document Agent를 각 소스마다 두고, Meta-Agent가 그 위에서 다단계 검색과 합성을 오케스트레이션한다.
복잡한 멀티 소스/멀티 스텝 과제를 다루기 위한 가장 정교한 형태다.

## RAG와 파인튜닝의 구분

원문은 RAG의 장점을 파인튜닝과 대비해 정리한다.

| 비교 항목 | RAG | 파인튜닝 |
|-----------|-----|-----------|
| 신선 데이터 반영 | 검색 시점에 반영 | 재학습 필요 |
| 환각 대응 | 근거 문서 기반으로 줄임 | 모델 지식에 의존 |
| 도메인 적응 | 인덱스 갱신으로 빠름 | 학습 파이프라인 필요 |

핵심은 RAG가 모델 재학습 없이 실시간 데이터에 접근할 수 있다는 점, 그리고 검색된 근거가 환각을 줄이는 데 기여한다는 점이다.

## 패턴 선택 가이드

여덟 가지 패턴을 어떤 기준으로 골라야 할까.
원문에 명시된 분류를 풀어 보면 다음과 같이 정리된다.

| 상황 | 권장 패턴 |
|------|------------|
| FAQ 수준의 정적 문서 검색 | Simple RAG |
| 사용자별 다중 턴 챗봇 | Simple RAG with Memory |
| 도메인이 분명히 나뉜 멀티 소스 | Branched RAG |
| 모호한 탐색형 리서치 질의 | HyDe |
| 단순/복잡 질의를 한 시스템에서 모두 처리 | Adaptive RAG |
| 검색 품질 자체가 결과를 좌우 | Corrective RAG (CRAG) |
| 답변 과정에서 정보 갭을 채워야 함 | Self-RAG |
| 멀티 소스/멀티 스텝 복잡 과제 | Agentic RAG |

## 결론

2025년의 RAG는 단일한 아키텍처가 아니다.
검색을 단순히 끼워 넣는 단계는 끝났고, 메모리 결합, 소스 분기, 검색 품질 평가, 자체 질의 생성, 에이전트 오케스트레이션까지 다양한 형태로 분화되었다.
Humanloop의 분류는 자기 서비스가 어디에 위치해야 하는지 가늠하는 시작점으로 쓸 수 있다.
Simple RAG로 시작해 데이터와 사용 패턴이 커지면 Corrective, Self, Agentic 쪽으로 옮겨가는 진화 경로가 자연스럽다.

## Reference

- [RAG Architectures — Humanloop](https://humanloop.com/blog/rag-architectures){:target="_blank"}
