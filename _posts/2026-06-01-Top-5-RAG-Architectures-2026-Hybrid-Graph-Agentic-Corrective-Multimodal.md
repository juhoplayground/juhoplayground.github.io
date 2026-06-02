---
layout: post
title: "2026년 알아야 할 5가지 RAG 아키텍처 - Hybrid, Graph, Agentic, Corrective, Multimodal"
author: 'Juho'
date: 2026-06-01 00:00:00 +0900
categories: [AI]
tags: [LLM, Embedding, Knowledge]
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
2. [01 Hybrid RAG](#01-hybrid-rag)
3. [02 GraphRAG](#02-graphrag)
4. [03 Agentic RAG](#03-agentic-rag)
5. [04 Corrective RAG (CRAG)](#04-corrective-rag-crag)
6. [05 Multimodal RAG](#05-multimodal-rag)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Brij Kishore Pandey가 공개한 "Top 5 RAG Architectures You Must Know in 2026" 인포그래픽은 2026 시점에서 실무자가 알아야 할 다섯 가지 RAG 아키텍처를 정리한다.
각 아키텍처는 단순 검색-증강 생성을 넘어 서로 다른 문제(검색 정확도, 관계 추론, 자율적 도구 사용, 잘못된 검색 보정, 멀티모달 통합)를 다룬다.
이 글에서는 그래픽에 등장하는 다섯 패턴의 흐름을 차례로 풀어본다.

## 01 Hybrid RAG

Hybrid RAG는 dense 임베딩 모델과 BM25 같은 키워드 기반 검색을 함께 사용하는 접근이다.
인포그래픽은 Query를 임베딩 모델에 통과시킨 결과를 Vector DB로 보내는 경로와, 별도의 BM25/Sparse 검색 경로를 합쳐 reciprocal rank fusion 또는 reranker를 통과한 뒤 Top-k chunk를 생성기에 넣는 흐름으로 묘사된다.
임베딩만으로 잡히지 않는 정확한 키워드 매칭과, 의미적 유사도가 함께 필요한 도메인(예: 법무, 의료, 사내 코드)에서 성능 개선이 두드러진다.

## 02 GraphRAG

GraphRAG는 답변이 단일 문서가 아니라 엔티티 간 관계에 흩어져 있는 질문에 강하다.
인포그래픽은 Query → Entity Extractor → Knowledge Graph → Subgraph Retriever → Answer 구성으로 표현된다.
구조적 메모리(triple 또는 그래프 노드)와 자유서술 텍스트가 결합되며, 멀티홉 추론이나 정책/규정 같이 노드 간 의존이 강한 도메인에 적합하다.
질문에서 엔티티를 뽑아 그래프에서 관련 부분만 잘라낸 뒤, 그 서브그래프와 텍스트 컨텍스트를 함께 LLM에 넣는 방식이다.

## 03 Agentic RAG

Agentic RAG는 검색을 한 번에 끝내지 않고, 에이전트가 필요에 따라 도구를 반복 호출한다.
그래픽에 등장하는 구성은 Query → Planner Agent → 여러 Tool(Vector Search, Web Search, SQL DB) → 결과 통합 → Answer다.
에이전트가 직접 어떤 도구를 어떤 순서로 부를지 결정하기 때문에, 단순 검색-생성보다 복잡한 워크플로(예: 사용자의 의도 파악 → 외부 데이터 조회 → 코드 실행)에 적합하다.
검색의 횟수와 폭이 늘어나는 만큼 비용과 지연 시간 관리가 핵심 과제다.

## 04 Corrective RAG (CRAG)

CRAG는 검색 결과가 부정확하거나 부적합할 때를 가정하고 보정 단계를 명시적으로 추가한 구조다.
인포그래픽은 Query → Retriever → Relevance Evaluator → Confident면 곧바로 Answer, 그렇지 않으면 Web Search 또는 Knowledge Refiner를 거쳐 다시 Reranker 후 Answer로 가는 흐름이다.
검색 품질을 한 번 채점하고, 신뢰가 낮으면 외부 검색이나 정제 과정을 거쳐 다시 답하기 때문에 환각 위험이 큰 도메인에서 효과적이다.

## 05 Multimodal RAG

Multimodal RAG는 텍스트뿐 아니라 이미지, 표, 차트, 음성 같은 다양한 모달리티를 인덱스에 함께 담는다.
그래픽은 Text Chunk와 Visual/Multimedia를 별도로 임베딩한 뒤 통합 Multimodal Vector DB에 저장하고, Query 시점에 Multimodal LLM이 양쪽을 함께 보는 흐름으로 묘사된다.
PDF에 포함된 그림과 표, 제품 카탈로그의 이미지, 회의 녹취 같은 자료를 통합해야 하는 환경에서 자연스러운 선택지가 된다.

