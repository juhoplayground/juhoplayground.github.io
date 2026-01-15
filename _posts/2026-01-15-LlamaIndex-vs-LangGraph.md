---
layout: post
title: LlamaIndex vs LangGraph - RAG와 멀티 에이전트 워크플로우의 선택
author: 'Juho'
date: 2026-01-15 09:00:00 +0900
categories: [LangChain]
tags: [Python, LangChain, LangGraph, LlamaIndex, Agent, RAG]
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
1. [들어가며](#들어가며)
2. [LlamaIndex 개요](#llamaindex-개요)
3. [LangGraph 개요](#langgraph-개요)
4. [핵심 차이점 비교](#핵심-차이점-비교)
5. [아키텍처 패턴 차이](#아키텍처-패턴-차이)
6. [실제 사용 경험과 한계](#실제-사용-경험과-한계)
7. [언제 무엇을 선택해야 하나](#언제-무엇을-선택해야-하나)
8. [코드 비교](#코드-비교)
9. [결론](#결론)
10. [참고 자료](#참고-자료)

## 들어가며

LLM 애플리케이션을 개발할 때 가장 많이 접하게 되는 두 가지 프레임워크가 있습니다. 바로 **LlamaIndex**와 **LangGraph**입니다. 두 프레임워크 모두 LLM 기반 애플리케이션 개발을 지원하지만, 그 접근 방식과 강점은 완전히 다릅니다.

이 글에서는 두 프레임워크의 차이점을 실제 사용 경험을 바탕으로 비교하고, 어떤 상황에서 어떤 프레임워크를 선택해야 하는지 알아보겠습니다.

## LlamaIndex 개요

### LlamaIndex란?

LlamaIndex는 **"Context-Augmented LLM Applications Framework"**로, 사용자 데이터를 LLM과 연결하는 것에 특화된 프레임워크입니다.

공식 문서에서는 다음과 같이 정의합니다:
> "LlamaIndex is the leading framework for building LLM-powered agents over your data with LLMs and workflows."

### 핵심 문제와 솔루션

**문제**: 대규모 언어 모델은 공개 정보로 사전 학습되어 있지만, 사용자의 프라이빗 데이터나 도메인 특화 데이터에는 접근할 수 없습니다.

**솔루션**: LlamaIndex는 **컨텍스트 증강(Context Augmentation)**을 통해 추론 시점에 프라이빗 데이터를 LLM에 제공합니다.

### 핵심 컴포넌트

LlamaIndex는 다음과 같은 모듈식 아키텍처를 제공합니다:

| 컴포넌트 | 역할 |
|---------|------|
| **Data Connectors** | API, PDF, SQL, 클라우드 스토리지 등에서 데이터 수집 |
| **Data Indexes** | LLM 소비에 최적화된 중간 표현으로 데이터 변환 |
| **Query Engines** | 질문-답변(RAG 플로우)을 위한 자연어 인터페이스 |
| **Chat Engines** | 멀티턴 대화를 위한 인터페이스 |
| **Agents** | 도구로 강화된 LLM 기반 워커 |
| **Workflows** | 복잡한 다단계 프로세스를 위한 이벤트 기반 시스템 |

### 간단한 예제

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 데이터 로드
documents = SimpleDirectoryReader("data").load_data()

# 인덱스 생성
index = VectorStoreIndex.from_documents(documents)

# 쿼리 엔진 생성
query_engine = index.as_query_engine()

# 질문하기
response = query_engine.query("우리 회사의 매출은?")
print(response)
```

단 5줄의 코드로 RAG 시스템을 구축할 수 있습니다. 이것이 LlamaIndex의 가장 큰 강점입니다.

## LangGraph 개요

### LangGraph란?

LangGraph는 **"Low-Level Agent Orchestration Framework"**로, 장기 실행되는 상태 기반 에이전트를 구축, 관리, 배포하는 데 특화된 프레임워크입니다.

공식 문서에서는 다음과 같이 강조합니다:
> "LangGraph is very low-level, and focused entirely on agent orchestration"

Klarna, Replit, Elastic 같은 회사들이 실제 프로덕션에서 사용하고 있습니다.

### 핵심 철학

LangGraph는 **에이전트 오케스트레이션에만 집중**하며, 프롬프트를 숨기거나 아키텍처를 강제하지 않습니다. 개발자에게 세밀한 제어권을 제공합니다.

### 5가지 핵심 기능

| 기능 | 설명 |
|------|------|
| **Durable Execution** | 실패에도 지속되며 체크포인트에서 재개 가능 |
| **Human-in-the-Loop** | 모든 지점에서 에이전트 상태 검사 및 수정 가능 |
| **Comprehensive Memory** | 단기 작업 메모리 + 장기 세션 간 스토리지 |
| **Deep Debugging** | LangSmith와 통합된 시각화 및 실행 추적 |
| **Production-Ready** | 상태 기반 장기 실행 워크플로우를 위한 확장 가능한 인프라 |

### 핵심 아키텍처 컴포넌트

LangGraph는 그래프 기반 접근 방식을 사용합니다:

- **StateGraph**: 에이전트 워크플로우 정의를 위한 핵심 컨테이너
- **Nodes**: 개별 계산 단계 또는 에이전트 함수
- **Edges**: 노드 간 실행 흐름을 정의하는 연결

### 간단한 예제

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

# 그래프 생성
graph = StateGraph(MessagesState)
graph.add_node(mock_llm)
graph.add_edge(START, "mock_llm")
graph.add_edge("mock_llm", END)

# 컴파일 및 실행
graph = graph.compile()
```

## 핵심 차이점 비교

### 추상화 레벨

| 구분 | LlamaIndex | LangGraph |
|------|-----------|-----------|
| **추상화 레벨** | High-Level | Low-Level |
| **제어 수준** | 제한적 (편의성 우선) | 완전한 제어 |
| **학습 곡선** | 완만함 | 가파름 |
| **유연성** | 보통 | 매우 높음 |

### 주요 사용 사례

| LlamaIndex | LangGraph |
|-----------|-----------|
| RAG (Retrieval-Augmented Generation) | 멀티 에이전트 워크플로우 |
| 간단한 Q&A 챗봇 | 장기 실행 에이전트 |
| 데이터 인덱싱 및 검색 | 복잡한 상태 관리 |
| 빠른 프로토타이핑 | Human-in-the-Loop 시스템 |
| 단일 에이전트 시스템 | 멀티 에이전트 협업 |

### 아키텍처 비교

```
┌─────────────────────────────────────────────────────┐
│ LlamaIndex: 중앙 집중형 오케스트레이터 패턴        │
│                                                     │
│        ┌──────────────────┐                        │
│        │ Central Agent    │                        │
│        │  (Orchestrator)  │                        │
│        └────────┬─────────┘                        │
│                 │                                   │
│        ┌────────┼────────┐                         │
│        │        │        │                         │
│    ┌───▼──┐ ┌──▼───┐ ┌─▼────┐                    │
│    │Tool1 │ │Tool2 │ │Tool3 │                    │
│    └──────┘ └──────┘ └──────┘                    │
│                                                     │
│ - 단일 에이전트가 모든 도구 호출 제어              │
│ - 간단하고 예측 가능                               │
│ - 복잡한 워크플로우에서는 확장성 한계              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ LangGraph: 분산 그래프 패턴                         │
│                                                     │
│   ┌──────┐      ┌──────┐      ┌──────┐            │
│   │Agent1│─────▶│Agent2│─────▶│Agent3│            │
│   └───┬──┘      └───┬──┘      └───┬──┘            │
│       │             │             │                │
│       │             ▼             │                │
│       │         ┌──────┐          │                │
│       └────────▶│Agent4│◀─────────┘                │
│                 └──────┘                           │
│                                                     │
│ - 각 에이전트가 독립적으로 결정                     │
│ - 유연하고 확장 가능                               │
│ - 복잡한 협업 워크플로우에 적합                    │
└─────────────────────────────────────────────────────┘
```

## 아키텍처 패턴 차이

### LlamaIndex: 중앙 집중형 오케스트레이터

LlamaIndex는 **중앙 집중형 오케스트레이터 패턴**을 사용합니다:

**특징**:
- 단일 에이전트가 모든 도구와 데이터 소스를 관리
- 명확한 제어 흐름
- 간단하고 예측 가능한 동작

**한계**:
- 복잡한 멀티 에이전트 시스템에서는 확장성 제한
- 에이전트 간 복잡한 협업 패턴 구현 어려움
- 서브 에이전트가 많아질수록 관리 복잡도 증가

### LangGraph: 분산 그래프 패턴

LangGraph는 **분산 그래프 패턴**을 사용합니다:

**특징**:
- 각 노드(에이전트)가 독립적으로 실행
- 상태 기반 실행 흐름
- 체크포인트와 재시작 지원

**강점**:
- 복잡한 멀티 에이전트 워크플로우 구현 용이
- 각 에이전트의 역할과 책임이 명확
- Human-in-the-Loop 통합 쉬움

## 실제 사용 경험과 한계

### LlamaIndex의 강점과 한계

**강점**:
- ✅ RAG 패턴 구현에 최적화
- ✅ 데이터 로딩, 인덱싱, 검색이 매우 쉬움
- ✅ 빠른 프로토타이핑
- ✅ 초보자 친화적인 API

**한계**:
- ❌ 멀티 에이전트 워크플로우 구현 시 추상화 구조의 제약
- ❌ 서브 에이전트 수가 증가할수록 관리 어려움
- ❌ 복잡한 상태 관리 필요 시 한계 명확
- ❌ 중앙 집중형 패턴의 확장성 한계

### 실제 경험담

처음에는 LlamaIndex의 간편함에 매력을 느껴 사용했지만, 프로젝트가 복잡해질수록 한계가 느껴졌습니다:

**시나리오**: 복잡한 딥 리서치 워크플로우 구축 (서브 에이전트 약 20개)

- LlamaIndex: 중앙 오케스트레이터가 모든 서브 에이전트를 관리하려다 보니 코드가 복잡해지고 디버깅이 어려웠습니다.
- LangGraph: 각 서브 에이전트를 독립적인 노드로 구성하고, 명확한 엣지로 연결하니 구현과 유지보수가 훨씬 수월했습니다.

### LangGraph로 마이그레이션하는 이유

"이것저것 구현하다 보니 어쩔 수 없이 LangChain/LangGraph에 손이 가게 되는" 이유는:

1. **상태 관리의 명확성**: 각 노드의 상태가 명확하게 정의됨
2. **체크포인트 기능**: 장기 실행 작업에서 중간 상태 저장 및 재개 가능
3. **Human-in-the-Loop**: 에이전트 실행 중 사람의 개입이 자연스러움
4. **디버깅 용이성**: LangSmith와의 통합으로 시각화 및 추적 가능
5. **확장성**: 에이전트를 추가해도 아키텍처가 깨지지 않음

## 언제 무엇을 선택해야 하나

### LlamaIndex를 선택해야 할 때

✅ **다음과 같은 경우 LlamaIndex가 적합합니다**:

1. **RAG 기반 Q&A 시스템**
   - 문서 기반 질문-답변 챗봇
   - 지식 베이스 검색 시스템
   - PDF/문서 기반 정보 추출

2. **빠른 프로토타이핑**
   - POC(Proof of Concept) 단계
   - MVP(Minimum Viable Product) 개발
   - 데모 애플리케이션

3. **단순한 에이전트 시스템**
   - 단일 에이전트로 충분한 경우
   - 복잡한 상태 관리가 필요 없는 경우
   - 도구 호출이 단순한 경우

4. **데이터 중심 애플리케이션**
   - 데이터 인덱싱이 핵심인 경우
   - 다양한 데이터 소스 통합이 필요한 경우
   - 벡터 DB 최적화가 중요한 경우

### LangGraph를 선택해야 할 때

✅ **다음과 같은 경우 LangGraph가 적합합니다**:

1. **복잡한 멀티 에이전트 시스템**
   - 서브 에이전트가 10개 이상인 경우
   - 에이전트 간 협업이 필요한 경우
   - 각 에이전트의 역할이 명확히 구분되는 경우

2. **장기 실행 워크플로우**
   - 실행 시간이 긴 작업(몇 분 이상)
   - 중간 체크포인트 저장이 필요한 경우
   - 실패 시 재시작이 필요한 경우

3. **Human-in-the-Loop 시스템**
   - 사람의 승인이 필요한 단계가 있는 경우
   - 중간 결과를 검토해야 하는 경우
   - 수동 개입이 필요한 경우

4. **복잡한 상태 관리**
   - 에이전트 간 상태 공유가 필요한 경우
   - 조건부 분기가 복잡한 경우
   - 순환 워크플로우가 있는 경우

5. **프로덕션 배포**
   - 안정성과 신뢰성이 중요한 경우
   - 확장 가능한 아키텍처가 필요한 경우
   - 상세한 모니터링과 디버깅이 필요한 경우

### 의사결정 플로우차트

```
시작
  │
  ▼
RAG 중심 애플리케이션인가?
  │
  ├─ Yes ──▶ LlamaIndex 사용
  │
  └─ No
      │
      ▼
  에이전트가 5개 이하인가?
      │
      ├─ Yes ──▶ LlamaIndex 또는 LangGraph (선택)
      │
      └─ No
          │
          ▼
  복잡한 상태 관리가 필요한가?
          │
          ├─ Yes ──▶ LangGraph 사용
          │
          └─ No
              │
              ▼
  Human-in-the-Loop이 필요한가?
              │
              ├─ Yes ──▶ LangGraph 사용
              │
              └─ No ──▶ LlamaIndex 시도 후 필요시 마이그레이션
```

## 코드 비교

### 시나리오: 문서 기반 리서치 에이전트

동일한 작업(문서를 읽고, 분석하고, 요약하는 리서치 에이전트)을 두 프레임워크로 구현해보겠습니다.

### LlamaIndex 구현

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.core.agent import ReActAgent
from llama_index.core.tools import QueryEngineTool, ToolMetadata

# 1. 문서 로드 및 인덱싱
documents = SimpleDirectoryReader("research_papers").load_data()
index = VectorStoreIndex.from_documents(documents)

# 2. 쿼리 엔진을 도구로 래핑
query_engine = index.as_query_engine()
query_tool = QueryEngineTool(
    query_engine=query_engine,
    metadata=ToolMetadata(
        name="research_papers",
        description="논문 데이터베이스에서 정보를 검색합니다."
    )
)

# 3. 에이전트 생성
agent = ReActAgent.from_tools([query_tool])

# 4. 실행
response = agent.chat("최신 AI 연구 동향을 요약해줘")
print(response)
```

**특징**:
- 간결하고 직관적
- RAG 패턴에 최적화
- 단일 에이전트 구조

### LangGraph 구현

```python
from langgraph.graph import StateGraph, MessagesState, START, END
from langchain_core.messages import HumanMessage, AIMessage
from langchain_community.vectorstores import FAISS
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from typing import TypedDict, Annotated
import operator

# 1. 상태 정의
class ResearchState(TypedDict):
    messages: Annotated[list, operator.add]
    documents: list
    summary: str
    current_step: str

# 2. 노드 함수 정의
def load_documents(state: ResearchState):
    """문서 로드 노드"""
    # 문서 로딩 로직
    docs = ["doc1", "doc2", "doc3"]  # 실제로는 벡터 DB에서 검색
    return {"documents": docs, "current_step": "loaded"}

def analyze_documents(state: ResearchState):
    """문서 분석 노드"""
    llm = ChatOpenAI()
    analysis = llm.invoke(f"다음 문서를 분석해줘: {state['documents']}")
    return {
        "messages": [AIMessage(content=analysis.content)],
        "current_step": "analyzed"
    }

def create_summary(state: ResearchState):
    """요약 생성 노드"""
    llm = ChatOpenAI()
    summary = llm.invoke(f"다음을 요약해줘: {state['messages'][-1].content}")
    return {
        "summary": summary.content,
        "current_step": "completed"
    }

# 3. 그래프 구성
graph = StateGraph(ResearchState)

# 노드 추가
graph.add_node("load", load_documents)
graph.add_node("analyze", analyze_documents)
graph.add_node("summarize", create_summary)

# 엣지 추가
graph.add_edge(START, "load")
graph.add_edge("load", "analyze")
graph.add_edge("analyze", "summarize")
graph.add_edge("summarize", END)

# 4. 컴파일
app = graph.compile()

# 5. 실행
result = app.invoke({
    "messages": [HumanMessage(content="최신 AI 연구 동향을 요약해줘")],
    "documents": [],
    "summary": "",
    "current_step": "init"
})

print(result["summary"])
```

**특징**:
- 명확한 상태 관리
- 각 단계가 독립적인 노드
- 확장 및 수정 용이

### 복잡한 멀티 에이전트 시나리오: 딥 리서치

서브 에이전트 20개를 사용하는 복잡한 리서치 시스템을 비교해보겠습니다.

### LlamaIndex의 한계

```python
# 중앙 오케스트레이터가 모든 서브 에이전트 관리
agent = ReActAgent.from_tools([
    search_tool, scrape_tool, summarize_tool, fact_check_tool,
    citation_tool, compare_tool, analyze_tool, extract_tool,
    translate_tool, categorize_tool, rank_tool, filter_tool,
    merge_tool, validate_tool, format_tool, export_tool,
    visualize_tool, report_tool, email_tool, archive_tool
])  # 20개 도구를 단일 에이전트가 관리

# 문제점:
# 1. 에이전트가 어떤 도구를 언제 사용할지 혼란
# 2. 도구 간 의존성 관리 어려움
# 3. 디버깅 시 어디서 문제가 발생했는지 추적 어려움
# 4. 확장 시 복잡도 기하급수적 증가
```

### LangGraph의 강점

```python
# 각 기능을 독립적인 노드로 구성
graph = StateGraph(DeepResearchState)

# 리서치 파이프라인을 명확한 단계로 구분
graph.add_node("search", search_agent)
graph.add_node("scrape", scrape_agent)
graph.add_node("extract", extract_agent)
graph.add_node("fact_check", fact_check_agent)
graph.add_node("summarize", summarize_agent)
# ... 15개 더

# 명확한 워크플로우 정의
graph.add_edge(START, "search")
graph.add_edge("search", "scrape")
graph.add_edge("scrape", "extract")
graph.add_conditional_edges(
    "extract",
    should_fact_check,
    {
        "yes": "fact_check",
        "no": "summarize"
    }
)
# ... 나머지 엣지

# 장점:
# 1. 각 에이전트의 역할이 명확
# 2. 실행 흐름을 시각적으로 이해 가능
# 3. 체크포인트로 중간 상태 저장
# 4. 특정 노드만 수정/테스트 가능
# 5. Human-in-the-Loop 쉽게 추가 가능
```

## 결론

### 핵심 요약

| 기준 | LlamaIndex | LangGraph |
|------|-----------|-----------|
| **적합한 사용 사례** | RAG, Q&A, 데이터 검색 | 멀티 에이전트, 복잡한 워크플로우 |
| **추상화 레벨** | High-Level | Low-Level |
| **학습 난이도** | 쉬움 | 보통-어려움 |
| **확장성** | 제한적 | 매우 높음 |
| **상태 관리** | 간단 | 강력함 |
| **디버깅** | 기본적 | 고급 (LangSmith 통합) |
| **프로덕션 준비도** | 중간 | 높음 |

### 실용적 조언

1. **시작은 LlamaIndex로**: RAG 기반 애플리케이션이라면 LlamaIndex로 시작하세요. 빠르고 쉽습니다.

2. **복잡해지면 LangGraph로**: 서브 에이전트가 5-10개를 넘어가거나, 복잡한 상태 관리가 필요하면 LangGraph로 마이그레이션을 고려하세요.

3. **병행 사용도 가능**: LlamaIndex의 데이터 로딩/인덱싱 기능을 LangGraph의 노드에서 사용할 수 있습니다. 두 프레임워크는 상호 배타적이지 않습니다.

4. **프로덕션은 LangGraph**: 장기 실행, 안정성, 모니터링이 중요하다면 LangGraph의 체크포인트와 상태 관리 기능을 활용하세요.

### 최종 추천

- **입문자 / POC 단계**: LlamaIndex
- **중급 개발자 / MVP 단계**: LlamaIndex (단순) 또는 LangGraph (복잡)
- **고급 개발자 / 프로덕션**: LangGraph
- **RAG 중심 애플리케이션**: LlamaIndex
- **멀티 에이전트 시스템**: LangGraph

기억하세요: "어쩔 수 없이 손이 가게 된다"는 것은 잘못된 선택이 아닙니다. 프로젝트가 성장하면서 요구사항이 변하는 것은 자연스러운 과정입니다. 중요한 것은 각 단계에서 적절한 도구를 선택하는 것입니다.

## 참고 자료

- [LlamaIndex 공식 문서](https://developers.llamaindex.ai/python/framework/){:target="_blank"}
- [LangGraph 공식 문서](https://docs.langchain.com/oss/python/langgraph/overview){:target="_blank"}
- [LangGraph GitHub](https://github.com/langchain-ai/langgraph){:target="_blank"}
- [LlamaIndex GitHub](https://github.com/run-llama/llama_index){:target="_blank"}
