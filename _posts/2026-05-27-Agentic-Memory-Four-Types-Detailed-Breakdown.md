---
layout: post
title: "Agentic Memory 4가지 유형 해부: 컨텍스트, 외부, 에피소딕, 파라메트릭 메모리 설계"
author: 'Juho'
date: 2026-05-27 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, Embedding, Knowledge, Management]
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
   - [4가지 메모리 유형](#4가지-메모리-유형)
   - [메모리 흐름과 에이전트 루프](#메모리-흐름과-에이전트-루프)
   - [MemoryStore 구현](#memorystore-구현)
   - [EpisodicLogger 구현](#episodiclogger-구현)
   - [메모리 큐레이션 전략](#메모리-큐레이션-전략)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

LLM 은 매 세션마다 기억을 잃는다.
챗봇이라면 큰 문제가 아니지만, 의사결정을 내리고 작업을 수행하며 시간이 지날수록 더 똑똑해져야 하는 에이전트에는 치명적이다.
Agentic Memory 는 단순한 storage 가 아니라 Continuity(정체성), Context(현재 과제), Learning(개선) 세 역할을 동시에 수행하는 시스템이다.
이 글에서는 @techwith_ram 이 정리한 4가지 메모리 유형, 메모리 흐름, 그리고 Python + ChromaDB 로 구현하는 메모리 레이어를 정리한다.

## 배경

좋은 에이전트의 “지능”은 응답 품질만이 아니라 기억하고 학습하며 누적된 경험 위에 새 결정을 쌓는 능력에서 나온다.
메모리는 stateless 한 시스템을 진화 가능한 시스템으로 바꾸는 핵심이다.
세 가지 역할로 분리하면 설계가 명확해진다.

| 역할 | 내용 |
|------|------|
| Continuity | 사용자가 누구이고 무엇을 선호하며 이전에 무엇을 함께 만들었는지 기억 |
| Context | 지금 진행 중인 멀티스텝 워크플로우의 직전 상태와 다음 행동 |
| Learning | 무엇이 잘 됐고 무엇이 실패했는지를 누적해 다음 결정을 개선 |

## 핵심 내용

### 4가지 메모리 유형

저자는 메모리를 in-context, external, episodic, semantic/parametric 의 네 유형으로 구분한다.
각각은 인간 뇌의 다른 영역처럼 서로 다른 역할을 한다.

| 유형 | 특성 | 보관 위치 | 한계 |
|------|------|----------|------|
| In-context | 즉시 접근 가능한 작업대 | 모델 컨텍스트 윈도우 | 토큰 비용, 세션 종료 시 소멸 |
| External | 세션 경계를 넘어 영속화 | DB, 벡터스토어, 파일 | 검색 단계가 병목 |
| Episodic | 과거 행동의 결과 기록 | 구조화 로그 + 벡터 | 평가 점수의 신뢰성 |
| Semantic/Parametric | 학습 시 가중치에 인코딩 | 모델 weights | frozen, 환각 |

**In-context memory** 는 시스템 프롬프트, 대화 이력, 툴 호출 결과, 검색된 메모리, 스크래치패드를 담는다.
긴 대화에서는 누적으로 오버플로우가 발생하므로 요약 압축, 선택적 보존, 외부 메모리 오프로드 같은 전략이 필요하다.

**External memory** 는 두 가지 흐름이 있다.
PostgreSQL/Redis/SQLite 같은 구조화 저장소는 정확 조회에 강하고, Pinecone/Chroma/pgvector 같은 벡터 저장소는 의미 검색에 강하다.
저자는 “좋은 메모리 아키텍처는 storage 20%, retrieval 80%”라고 강조한다.

**Episodic memory** 는 가장 저평가된 유형이다.
사실(facts) 이 아니라 사건(events), 특히 과거 행동의 outcome 을 저장한다.
각 에피소드는 task, approach, outcome, duration, token cost, quality score, embedding 등을 담은 구조화 레코드다.
새 작업이 들어오면 의미적으로 유사한 과거 에피소드를 검색해 전략 선택에 활용한다.
즉 핸드크래프트 데이터셋이 아니라 개인 이력에서 few-shot 학습을 하는 것이다.

**Semantic/Parametric memory** 는 모델 weights 에 내재된 일반 지식이다.
학습 시점 이후의 사건, 도메인 특수 지식, 사적 정보에는 의존하지 말아야 하며, 시간 민감하거나 도메인 특수한 영역은 external retrieval 로 보강해야 한다.

### 메모리 흐름과 에이전트 루프

에이전트의 매 요청 처리에는 메모리 연산이 LLM 호출을 앞뒤로 둘러싼다.
호출 전에 retrieval, 호출 후에 writing 이 일어난다.
모델 자체는 stateless 이고 메모리 시스템이 “상태가 있는 듯한 환상”을 만든다.

### MemoryStore 구현

ChromaDB + OpenAI 임베딩으로 영속 벡터 메모리를 구현한다.

```python
import chromadb
from openai import OpenAI
from datetime import datetime
import json, uuid

class MemoryStore:
    """Persistent vector memory for an AI agent."""

    def __init__(self, agent_id: str, persist_dir: str = "./memory_db"):
        self.agent_id = agent_id
        self.openai = OpenAI()

        self.client = chromadb.PersistentClient(path=persist_dir)
        self.collection = self.client.get_or_create_collection(
            name=f"agent_{agent_id}_memories",
            metadata={"hnsw:space": "cosine"}
        )

    def _embed(self, text: str) -> list[float]:
        response = self.openai.embeddings.create(
            model="text-embedding-3-small",
            input=text
        )
        return response.data[0].embedding

    def remember(self, content: str, memory_type: str = "general", metadata: dict = None) -> str:
        memory_id = str(uuid.uuid4())
        embedding = self._embed(content)

        meta = {
            "type": memory_type,
            "timestamp": datetime.utcnow().isoformat(),
            "agent_id": self.agent_id,
            **(metadata or {})
        }

        self.collection.add(
            ids=[memory_id],
            embeddings=[embedding],
            documents=[content],
            metadatas=[meta]
        )
        return memory_id

    def recall(self, query: str, k: int = 5, memory_type: str = None, min_relevance: float = 0.6) -> list[dict]:
        query_embedding = self._embed(query)
        where = {"type": memory_type} if memory_type else None

        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=k,
            where=where,
            include=["documents", "metadatas", "distances"]
        )

        memories = []
        for doc, meta, dist in zip(
            results["documents"][0],
            results["metadatas"][0],
            results["distances"][0]
        ):
            relevance = 1 - dist
            if relevance >= min_relevance:
                memories.append({
                    "content": doc,
                    "metadata": meta,
                    "relevance": round(relevance, 3)
                })

        return sorted(memories, key=lambda x: x["relevance"], reverse=True)

    def forget(self, memory_id: str):
        self.collection.delete(ids=[memory_id])
```

### EpisodicLogger 구현

MemoryStore 위에 에피소드 로깅 레이어를 얹어 과거 작업의 outcome 을 의미 검색 가능한 형태로 저장한다.

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class Episode:
    task: str
    approach: str
    outcome: str
    duration_ms: int
    token_cost: int
    quality_score: float
    notes: str = ""
    error: Optional[str] = None


class EpisodicLogger:
    def __init__(self, memory_store: MemoryStore):
        self.store = memory_store

    def log(self, episode: Episode):
        doc = (
            f"Task: {episode.task}\n"
            f"Approach: {episode.approach}\n"
            f"Outcome: {episode.outcome}\n"
            f"Notes: {episode.notes}"
        )
        self.store.remember(
            content=doc,
            memory_type="episode",
            metadata={
                "outcome": episode.outcome,
                "quality_score": episode.quality_score,
                "duration_ms": episode.duration_ms,
                "token_cost": episode.token_cost,
            }
        )

    def recall_similar(self, task: str, k: int = 3) -> list[dict]:
        return self.store.recall(
            query=task,
            k=k,
            memory_type="episode",
            min_relevance=0.65
        )
```

전체를 묶으면 메모리 보강 에이전트가 된다.
요청이 들어오면 의미 검색으로 관련 메모리 + 유사 과거 에피소드를 가져와 system prompt 에 주입하고, 응답 후에는 상호작용을 메모리에 기록하고 에피소드 로그를 남긴다.

### 메모리 큐레이션 전략

메모리는 단순 축적이 아니라 큐레이션이 필요하다.
저자는 세 가지 전략을 제시한다.

**1. Time-based decay**: Generative Agents 논문에서 영감을 받은 공식으로, relevance, importance, recency 를 가중합한다.

```python
def memory_score(relevance, importance, created_at,
                 recency_weight=0.3, decay_factor=0.995):
    hours_old = (datetime.utcnow() - created_at).total_seconds() / 3600
    recency = math.pow(decay_factor, hours_old)
    return (relevance * 0.4 + importance * 0.3 + recency * recency_weight)
```

**2. Importance scoring at write time**: 저장 시점에 LLM 에게 0.0~1.0 점수로 중요도를 평가시키고 임계치 이상만 저장한다.
인사말은 0.0, 사용자 선호·에러·결정은 1.0 으로 분류한다.

**3. Periodic consolidation**: 주기적으로 유사도 0.92 이상인 메모리 그룹을 단일 canonical summary 로 병합한다.
인간의 수면 중 기억 통합과 비슷한 메커니즘이다.

벡터 DB 선택은 로컬 개발에서는 ChromaDB, Postgres 기반 운영에서는 pgvector, 본격 스케일이 필요할 때는 Pinecone 또는 Qdrant 를 권장한다.

## 의미와 시사점

저자는 “모델 자체보다 무엇을 기억하고 무엇을 잊으며 그 정보를 어떻게 사용하는지를 설계하는 것이 진짜 힘”이라고 정리한다.
실용적으로는 다음을 의미한다.
첫째, 단일 “메모리 솔루션”은 존재하지 않으며 4가지 유형을 각자에 맞는 저장소로 분리해야 한다.
둘째, 검색 설계(retrieval)가 저장 설계(storage) 보다 압도적으로 중요하다.
셋째, 시간 감쇠와 importance 평가, 통합 작업을 통해 메모리를 능동적으로 큐레이션해야 노이즈와 지연이 누적되지 않는다.

## 결론

Agentic Memory 는 in-context, external, episodic, parametric 4가지 유형이 서로 다른 storage 와 검색 전략으로 협업할 때 비로소 동작한다.
MemoryStore + EpisodicLogger 패턴은 ChromaDB 와 임베딩 모델만으로도 출발 가능한 최소 구현이다.
메모리 레이어를 제대로 설계하면 그 위의 모든 추론이 더 똑똑해진다.

## Reference

- [Agentic Memory: A Detailed Breakdown - @techwith_ram](https://x.com/techwith_ram/status/2037499938574110770)
