---
layout: post
title: "LangGraph 기반 차세대 Agentic RAG (2026 에디션)"
author: 'Juho'
date: 2026-06-13 00:00:00 +0900
categories: [LangChain]
tags: [LangChain, Agent, AI]
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
2. [RAG의 진화: 2023-2025에서 2026으로](#rag의-진화-2023-2025에서-2026으로)
   - [기존 RAG의 한계](#기존-rag의-한계)
   - [Agentic RAG의 개선점](#agentic-rag의-개선점)
3. [핵심 기술 구성 요소](#핵심-기술-구성-요소)
   - [LangGraph 기반 상태 오케스트레이션](#langgraph-기반-상태-오케스트레이션)
   - [멀티 에이전트 시스템](#멀티-에이전트-시스템)
   - [그래프 + 벡터 하이브리드 메모리](#그래프--벡터-하이브리드-메모리)
   - [반복 검색과 쿼리 재작성](#반복-검색과-쿼리-재작성)
   - [LLM-as-Judge와 자기 반성](#llm-as-judge와-자기-반성)
   - [비용 인식 적응형 모델 라우팅](#비용-인식-적응형-모델-라우팅)
4. [시스템 아키텍처](#시스템-아키텍처)
   - [오케스트레이션 레이어](#오케스트레이션-레이어)
   - [에이전트 통신 프로토콜](#에이전트-통신-프로토콜)
5. [관찰 가능성과 평가 프레임워크](#관찰-가능성과-평가-프레임워크)
   - [LangSmith 통합](#langsmith-통합)
   - [평가 지표](#평가-지표)
6. [Agentic RAG를 사용하지 말아야 할 경우](#agentic-rag를-사용하지-말아야-할-경우)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

[LangGraph 기반 차세대 Agentic RAG (2026 에디션)](https://medium.com/@vinodkrane/next-generation-agentic-rag-with-langgraph-2026-edition-d1c4c068d2b8){:target="_blank"} 아티클은 2023-2025년의 단순 RAG 파이프라인에서 2026년의 자율 에이전트 시스템으로의 진화를 정리한다.
기존의 정적인 "임베딩-검색-생성" 워크플로우 대신, LangGraph 오케스트레이션을 통해 계획, 추론, 비판, 반복 학습을 수행하는 지능형 에이전트 시스템이 등장하고 있다.
이 글은 해당 아티클의 핵심 내용을 구조적으로 정리한다.

## RAG의 진화: 2023-2025에서 2026으로

### 기존 RAG의 한계

2023-2025년에 사용되던 RAG 시스템은 다음과 같은 구조적 한계를 지닌다.

- 단일 검색 후 폴백 메커니즘이 없는 구조
- 멀티 스텝 추론 능력의 부재
- 자기 수정 또는 오류 감지 기능의 미흡
- 과거 인터랙션으로부터 학습하는 영속적 메모리 부재

이러한 한계는 복잡한 쿼리나 다중 소스를 요구하는 질문에서 품질 저하를 유발한다.

### Agentic RAG의 개선점

2026년 아키텍처는 다음 네 가지 핵심 역량으로 기존 한계를 극복한다.

- 유연한 도구 선택을 가능하게 하는 동적 의사 결정
- 여러 해결 경로를 탐색하는 그래프 오브 사고(Graph-of-Thought) 추론
- LLM-as-Judge 방식의 출력 검증을 통한 반복 반성
- 에피소딕·시맨틱 레이어를 활용하는 메모리 인식 지능

## 핵심 기술 구성 요소

### LangGraph 기반 상태 오케스트레이션

LangGraph는 Agentic RAG의 핵심 오케스트레이션 엔진으로 다음 기능을 제공한다.

- StateGraph: 모든 노드가 접근 가능한 공유 타입 상태
- 조건부 엣지: 상태 조건에 따라 실행 경로를 라우팅
- 영속성 레이어: MemorySaver, AsyncSqliteSaver, PostgresSaver를 통한 체크포인트 복구
- Human-in-the-loop: 중요 분기점에서 인간 개입을 허용하는 인터럽트 게이트
- 서브그래프: 모듈화된 멀티 에이전트 팀 구조

### 멀티 에이전트 시스템

전문화된 에이전트가 각각의 책임 영역을 담당한다.
이 분업 구조는 전체 시스템의 품질과 감사 가능성(auditability)을 향상시킨다.
각 에이전트는 공유 상태(AgentState TypedDict)를 통해 통신하므로 상태 추적과 재현이 용이하다.

### 그래프 + 벡터 하이브리드 메모리

세 가지 상호 보완적인 메모리 레이어가 결합된다.

| 레이어 | 기술 | 역할 |
|--------|------|------|
| 벡터 레이어 | pgvector 등 밀집 임베딩 | 시맨틱 유사도 검색 |
| 그래프 레이어 | Neo4j, FalkorDB | 엔티티 관계 및 출처 추적 |
| 에피소딕 레이어 | 실행 이력 저장소 | 학습 및 패턴 인식 |

### 반복 검색과 쿼리 재작성

단일 검색 패스에 의존하지 않고 다음 기법을 통해 복수 검색 라운드를 수행한다.

- 쿼리 확장 및 재구성
- HyDE(Hypothetical Document Embedding): 합성 컨텍스트 생성
- Stepback 프롬프팅: 더 넓은 개념적 기반 확보
- 크로스 인코더 리랭킹: 관련성 최적화

### LLM-as-Judge와 자기 반성

크리틱(Critic) 에이전트가 출력물을 독립적으로 평가한다.
평가 항목은 증거-답변 정합성, 모순 감지, 하위 목표 커버리지, 추가 검색 필요 여부 등이다.
크리틱 점수가 0.85 미만이면 재작성 사이클이 트리거된다(임계값 설정 가능).

### 비용 인식 적응형 모델 라우팅

태스크 복잡도와 예산 제약에 따라 모델 크기를 자동으로 선택한다.

| 태스크 유형 | 사용 모델 |
|-------------|-----------|
| 쿼리 재작성 | 소형 모델 |
| 추론 합성 | 중형 모델 |
| 최종 답변 검증 | 프리미엄 모델 |

이 외에도 불확실성, 레이턴시 요건, 결정론적 필요성을 기준으로 모델을 라우팅한다.

## 시스템 아키텍처

### 오케스트레이션 레이어

프로덕션 아키텍처는 네 개의 주요 레이어로 구성되며, 오케스트레이션 레이어("두뇌")의 각 노드는 다음 역할을 담당한다.

- Planner: 사용자 쿼리를 실행 가능한 서브 태스크로 분해
- Retriever: 외부 소스에서 데이터 수집
- Context Fusion: MMR과 압축을 활용한 중복 제거 및 컨텍스트 최적화
- Tool Agent: Python, SQL, API 등 기술적 연산 실행
- Reasoner: 고급 프롬프팅 기법을 통한 컨텍스트 합성
- Critic: 출력 검증 및 신뢰도 임계값 미달 시 재작성 트리거
- Formatter: 최종 출력(JSON, Markdown) 포맷팅

피드백 및 최적화 레이어의 Memory Agent는 실패한 실행 트레이스를 분석하여 장기 행동을 개선한다.

### 에이전트 통신 프로토콜

모든 에이전트는 공유 AgentState TypedDict를 통해서만 통신한다.
이 설계는 다음 이점을 제공한다.

- LangSmith 추적을 통한 완전한 상태 투명성
- 임의 체크포인트에서 정확한 시스템 재현
- 관리된 병합을 통한 안전한 병렬 실행
- 인터럽트 게이트를 통한 인간 개입 포인트 제공

다음은 LangGraph에서 StateGraph를 정의하는 기본 구조 예시다.

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List

class AgentState(TypedDict):
    query: str
    sub_tasks: List[str]
    retrieved_docs: List[str]
    critic_score: float
    iteration_count: int
    final_answer: str

graph = StateGraph(AgentState)

graph.add_node("planner", planner_node)
graph.add_node("retriever", retriever_node)
graph.add_node("reasoner", reasoner_node)
graph.add_node("critic", critic_node)

graph.add_conditional_edges(
    "critic",
    lambda state: "retriever" if state["critic_score"] < 0.85 else END,
)
```

## 관찰 가능성과 평가 프레임워크

### LangSmith 통합

프로덕션 시스템은 LangSmith를 통해 모든 노드, 엣지 전환, 상태 변이를 자동으로 추적한다.
활성화는 환경 변수 `LANGCHAIN_TRACING_V2=true` 설정으로 이루어진다.
각 트레이스에는 `critic_score`, `iteration_count`, `retrieval_round`, `token_budget_used` 등의 커스텀 메타데이터를 첨부해야 한다.
비용 귀속을 위해 노드별 모델 티어 및 토큰 수를 기록하고 세션 단위로 예상 비용을 집계한다.

### 평가 지표

검색 품질 평가에는 BEIR 벤치마크를 활용한다.

| 지표 | 설명 |
|------|------|
| nDCG@10 | 관련 청크의 적절한 랭킹에 보상을 부여 |
| MRR | 첫 번째 결과 위치를 측정하는 평균 역순위 |
| Recall@K | 상위 K 결과 내 관련 청크 비율 |

엔드 투 엔드 품질 평가에는 RAGAS 프레임워크를 사용한다.

| 지표 | 설명 |
|------|------|
| Faithfulness | 검색된 컨텍스트 기반 답변 vs. 환각 비율 |
| Answer Relevance | 응답이 실제 쿼리를 다루는 정도 |
| Context Precision/Recall | 올바른 증거 검색 및 활용 비율 |

## Agentic RAG를 사용하지 말아야 할 경우

프론티어 모델이 100만 토큰 이상의 컨텍스트 윈도우를 지원하게 되면서, 더 단순한 접근 방식이 충분한 경우도 많아졌다.
500페이지 매뉴얼이나 사내 위키 같은 소-중형 지식 베이스에서는 단순 `load-and-ask` 방식이 구축은 빠르고 비용은 저렴하며 디버깅도 쉽다.

Agentic RAG의 복잡성이 정당화되는 조건은 다음과 같다.

- 지식 베이스가 컨텍스트 윈도우 용량을 초과할 때
- 알 수 없는 소스에 걸친 멀티 홉 추론이 필요할 때
- SQL, 코드, API 등의 도구 사용이 필수적일 때
- 레이턴시 및 비용 최적화가 엔지니어링 투자를 정당화할 때

## 결론

이 아키텍처는 단순한 점진적 개선이 아닌 구조적 전환을 의미한다.
2023-2024년의 "검색 후 생성" 패턴과 달리, 2026년의 Agentic RAG 시스템은 계획 수립, 반복 검색, 분기 추론 로직, 자기 비판, 실패 기반 학습, 경제적 모델 선택 능력을 갖춘다.
LangGraph의 StateGraph, 조건부 엣지, 영속성 레이어, 멀티 에이전트 분업 구조는 이 자율 정보 시스템을 구현하는 핵심 기반 기술이다.
소형 지식 베이스에는 단순 RAG가 더 실용적이지만, 복잡한 멀티 소스 추론이 필요한 엔터프라이즈 환경에서는 이 아키텍처가 강력한 선택지가 된다.

## Reference

- [Next-Generation Agentic RAG with LangGraph (2026 Edition)](https://medium.com/@vinodkrane/next-generation-agentic-rag-with-langgraph-2026-edition-d1c4c068d2b8/)
- [GitHub: GenerativeAI-Lab / agentic-rag](https://github.com/vinodkrane/GenerativeAI-Lab/tree/main/agentic-rag/)
