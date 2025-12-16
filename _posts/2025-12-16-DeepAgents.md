---
layout: post
title: DeepAgents - LangChain 기반 장기 작업 AI 에이전트 프레임워크
author: 'Juho'
date: 2025-12-16 09:00:00 +0900
categories: [LangChain]
tags: [Python, LangChain, LangGraph, AI, Agent, DeepAgents, MCP]
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
1. [DeepAgents란?](#deepagents란)
2. [핵심 특징](#핵심-특징)
3. [설치 방법](#설치-방법)
4. [기본 사용법](#기본-사용법)
5. [내장 도구](#내장-도구)
6. [미들웨어 시스템](#미들웨어-시스템)
7. [고급 설정](#고급-설정)
8. [실전 예제: Deep Research Agent](#실전-예제-deep-research-agent)
9. [프로덕션 배포](#프로덕션-배포)
10. [마무리](#마무리)

## DeepAgents란?

[DeepAgents](https://github.com/langchain-ai/deepagents){:target="_blank"}는 LangChain과 LangGraph 기반의 오픈소스 AI 에이전트 프레임워크로, 복잡하고 장기적인 작업(long-horizon tasks)을 처리하는 에이전트를 구축하기 위한 도구입니다.

Claude Code, Deep Research, Manus와 같은 고급 AI 애플리케이션에서 영감을 받아 개발되었으며, 다음과 같은 상황에서 유용합니다:
- 복잡한 작업을 계획하고 단계별로 분해해야 할 때
- 대규모 컨텍스트를 관리하고 메모리 오버플로우를 방지해야 할 때
- 작업을 서브 에이전트에게 위임하여 컨텍스트를 격리해야 할 때
- 대화 세션 간 정보를 유지해야 할 때

### 프로젝트 정보
- GitHub: [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents){:target="_blank"}
- License: MIT (상업적 사용 가능)
- 활발한 개발: 200+ 커밋, 지속적인 업데이트

## 핵심 특징

DeepAgents는 다음과 같은 강력한 기능을 제공합니다:

### 1. 작업 계획 및 분해
내장된 `write_todos`와 `read_todos` 도구를 통해 에이전트가 복잡한 작업을 개별 단계로 분해하고, 진행 상황을 추적하며, 새로운 정보가 등장할 때 계획을 동적으로 조정할 수 있습니다.

### 2. 파일시스템 백엔드
파일 작업 도구(`ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`)를 통해 에이전트가 큰 컨텍스트를 메모리가 아닌 파일시스템으로 오프로드하여 컨텍스트 윈도우 오버플로우를 방지합니다.

### 3. 서브 에이전트 위임
내장 `task` 도구를 사용하여 전문화된 서브 에이전트를 생성하고, 각 서브 에이전트는 격리된 컨텍스트에서 작업을 수행합니다. 이를 통해 복잡한 작업을 병렬로 처리하고 전체 시스템의 효율성을 높일 수 있습니다.

### 4. Human-in-the-Loop (HITL)
민감한 작업(예: 파일 삭제, 명령 실행)에 대해 사용자의 승인을 요청하는 워크플로우를 구성할 수 있습니다.

### 5. 자동 컨텍스트 관리
컨텍스트가 170K 토큰을 초과하면 자동으로 대화를 요약하여 메모리를 관리합니다.

### 6. MCP (Model Context Protocol) 통합
MCP 도구를 지원하여 외부 시스템과의 통합을 쉽게 구현할 수 있습니다. `langchain-mcp-adapters`를 사용하면 MCP 서버의 도구를 LangChain 도구로 변환하여 DeepAgents에서 사용할 수 있습니다.

```python
from langchain_mcp_adapters import MCPToolkit

# MCP 서버에서 도구 가져오기
mcp_toolkit = MCPToolkit.from_server("path/to/mcp/server")
mcp_tools = mcp_toolkit.get_tools()

# DeepAgents에 MCP 도구 추가
agent = create_deep_agent(
    tools=mcp_tools,
    system_prompt="MCP 도구를 활용하는 에이전트"
)
```

## 설치 방법

DeepAgents를 설치하려면 다음 명령어를 실행합니다:

```bash
pip install deepagents tavily-python
```

`tavily-python`은 웹 검색 기능을 사용하기 위한 선택적 의존성입니다. 웹 검색이 필요하지 않다면 생략할 수 있습니다.

### 필수 요구사항
- Python 3.8 이상
- LangChain
- LangGraph

## 기본 사용법

DeepAgents를 사용하여 간단한 에이전트를 생성하는 방법은 다음과 같습니다:

```python
from deepagents import create_deep_agent

# 간단한 에이전트 생성
agent = create_deep_agent(
    tools=[],  # 커스텀 도구 목록 (선택사항)
    system_prompt="복잡한 작업을 계획하고 실행하는 에이전트입니다."
)

# 에이전트 실행
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "Python으로 간단한 웹 크롤러를 만들어주세요."}
    ]
})

print(result)
```

### 웹 검색 도구 추가

Tavily를 사용하여 웹 검색 기능을 추가할 수 있습니다:

```python
from langchain_community.tools.tavily_search import TavilySearchResults
from deepagents import create_deep_agent

# Tavily 검색 도구 생성
internet_search = TavilySearchResults(max_results=5)

# 웹 검색 기능이 있는 에이전트 생성
agent = create_deep_agent(
    tools=[internet_search],
    system_prompt="웹에서 정보를 검색하고 분석하는 연구 에이전트입니다."
)

# 실행
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "2025년 AI 트렌드에 대해 조사해주세요."}
    ]
})
```

### 스트리밍 모드

실시간으로 에이전트의 작업 과정을 확인할 수 있습니다:

```python
for chunk in agent.stream({
    "messages": [
        {"role": "user", "content": "복잡한 데이터 분석 파이프라인을 설계해주세요."}
    ]
}):
    print(chunk)
```

## 내장 도구

DeepAgents는 10가지 강력한 내장 도구를 제공합니다 (execute는 SandboxBackendProtocol 사용 시에만 사용 가능):

| 도구 | 기능 | 예시 사용 사례 |
|------|------|----------------|
| `write_todos` | 작업 목록 생성 및 관리 | 복잡한 프로젝트를 단계별로 분해 |
| `read_todos` | 현재 작업 목록 조회 | 진행 상황 확인 |
| `ls` | 디렉토리 파일 목록 조회 | 프로젝트 구조 파악 |
| `read_file` | 파일 내용 읽기 (페이지네이션 지원) | 코드 분석, 문서 검토 |
| `write_file` | 파일 생성 또는 덮어쓰기 | 코드 생성, 보고서 작성 |
| `edit_file` | 파일 내용 부분 수정 | 버그 수정, 코드 리팩토링 |
| `glob` | 패턴 기반 파일 검색 | `*.py` 파일 찾기 |
| `grep` | 텍스트 패턴 검색 | 특정 함수나 클래스 찾기 |
| `execute` | 샌드박스 환경에서 셸 명령 실행 (SandboxBackendProtocol 필요) | 테스트 실행, 빌드 수행 |
| `task` | 서브 에이전트에게 작업 위임 | 병렬 작업 처리 |

### 도구 사용 예시

```python
# 에이전트가 자동으로 도구를 선택하여 사용합니다
agent = create_deep_agent(
    system_prompt="""
    당신은 코드 분석 전문가입니다.
    프로젝트의 구조를 파악하고, 버그를 찾아내며, 개선안을 제시합니다.
    """
)

result = agent.invoke({
    "messages": [
        {"role": "user", "content": "현재 디렉토리의 모든 Python 파일을 분석하고 개선안을 제시해주세요."}
    ]
})

# 에이전트는 자동으로 다음과 같이 동작합니다:
# 1. write_todos로 작업 계획 수립
# 2. ls로 디렉토리 구조 파악
# 3. glob으로 *.py 파일 검색
# 4. read_file로 각 파일 내용 읽기
# 5. 분석 후 write_file로 보고서 작성
```

## 미들웨어 시스템

DeepAgents는 `create_deep_agent` 호출 시 7가지 미들웨어를 자동으로 추가하여 에이전트의 기능을 확장합니다:

| 미들웨어 | 기능 | 제공 도구 |
|----------|------|----------|
| TodoListMiddleware | 작업 계획 및 진행 상황 추적 | write_todos, read_todos |
| FilesystemMiddleware | 파일시스템 작업 및 컨텍스트 관리 | ls, read_file, write_file, edit_file, glob, grep, execute |
| SubAgentMiddleware | 서브 에이전트를 통한 작업 위임 | task |
| SummarizationMiddleware | 170K 토큰 초과 시 자동 대화 요약 | - |
| AnthropicPromptCachingMiddleware | 시스템 프롬프트 캐싱으로 API 비용 절감 (Anthropic 모델 전용) | - |
| PatchToolCallsMiddleware | 중단된 도구 호출 복구 처리 | - |
| HumanInTheLoopMiddleware | 민감한 작업에 대한 승인 워크플로우 (interrupt_on 설정 필요) | - |

각 미들웨어는 관련 도구에 대한 사용 지침을 시스템 프롬프트에 자동으로 추가합니다.

### 미들웨어 커스터마이징

```python
from deepagents import create_deep_agent
from deepagents.middleware import CustomMiddleware

# 커스텀 미들웨어 정의
class LoggingMiddleware(CustomMiddleware):
    def before_invoke(self, state):
        print(f"작업 시작: {state['messages'][-1]['content']}")
        return state

    def after_invoke(self, state):
        print(f"작업 완료")
        return state

# 미들웨어 추가
agent = create_deep_agent(
    middleware=[LoggingMiddleware()],
    system_prompt="로깅 기능이 추가된 에이전트"
)
```

## 고급 설정

### 1. 모델 선택

기본적으로 Claude Sonnet 4.5 (claude-sonnet-4-5-20250929)를 사용하지만, 다른 LangChain 호환 모델도 사용할 수 있습니다:

```python
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent

# OpenAI GPT-4o 사용
agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    system_prompt="GPT-4o를 사용하는 에이전트"
)

# Anthropic Claude 3.5 Sonnet 사용
from langchain_anthropic import ChatAnthropic

agent = create_deep_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-20241022"),
    system_prompt="Claude 3.5 Sonnet을 사용하는 에이전트"
)
```

### 2. 백엔드 설정

에이전트의 상태 저장 방식을 선택할 수 있습니다:

```python
from deepagents import create_deep_agent
from deepagents.backends import FilesystemBackend, StateBackend, StoreBackend

# 메모리 기반 (휘발성, 기본값)
agent = create_deep_agent(
    backend=StateBackend(),
    system_prompt="메모리 기반 에이전트"
)

# 파일시스템 기반 (영구 저장)
agent = create_deep_agent(
    backend=FilesystemBackend(base_path="./agent_workspace"),
    system_prompt="파일시스템 기반 에이전트"
)

# LangGraph Store 기반 (영구 저장, 장기 메모리)
agent = create_deep_agent(
    backend=StoreBackend(),
    system_prompt="Store 기반 에이전트"
)

# 하이브리드 (경로 기반 라우팅)
# 예: /memories/* 경로는 StoreBackend로, 나머지는 StateBackend로
from deepagents.backends import CompositeBackend

agent = create_deep_agent(
    backend=CompositeBackend({
        "memory": StateBackend(),
        "persistent": FilesystemBackend(base_path="./workspace")
    }),
    system_prompt="하이브리드 백엔드 에이전트"
)
```

### 3. 장기 기억 (Long-term Memory)

StoreBackend를 사용하면 여러 대화 세션 간에 정보를 유지할 수 있습니다. 에이전트가 `/memories/` 경로에 저장한 파일은 영구적으로 보존되어, 나중에 다른 대화에서도 참조할 수 있습니다.

```python
from deepagents import create_deep_agent
from deepagents.backends import StoreBackend, CompositeBackend, StateBackend
from langgraph.store.memory import InMemoryStore

# Store 생성 (실제로는 PostgreSQL, Redis 등 사용 가능)
store = InMemoryStore()

# 장기 기억용 백엔드 설정
# /memories/* 경로는 StoreBackend에, 나머지는 StateBackend에 저장
backend = CompositeBackend({
    "memories": StoreBackend(store=store),
    "default": StateBackend()
})

agent = create_deep_agent(
    backend=backend,
    system_prompt="""
    사용자와의 중요한 대화 내용은 /memories/ 경로에 저장하세요.
    예: /memories/user_preferences.json, /memories/previous_conversations.md
    """
)

# 첫 번째 대화
result1 = agent.invoke({
    "messages": [{"role": "user", "content": "내 이름은 김철수이고, Python을 좋아해."}]
})

# 나중에 다른 세션에서도 기억을 참조 가능
result2 = agent.invoke({
    "messages": [{"role": "user", "content": "내가 좋아하는 프로그래밍 언어가 뭐였지?"}]
})
# 에이전트는 /memories/에 저장된 정보를 읽어서 답변
```

장기 기억 활용 사례:
- 사용자 선호도 저장
- 이전 대화 내용 참조
- 프로젝트별 컨텍스트 유지
- 학습된 패턴 저장

### 4. Human-in-the-Loop 설정

특정 도구 사용 시 사용자 승인을 요청하도록 설정할 수 있습니다.

`interrupt_on` 파라미터는 도구 이름을 키로 하는 딕셔너리 형식이며, 각 도구별로 허용할 결정 유형을 지정할 수 있습니다:

```python
from langgraph.checkpoint.memory import MemorySaver

# 체크포인터는 필수입니다 (상태 저장용)
checkpointer = MemorySaver()

agent = create_deep_agent(
    interrupt_on={
        "execute": True,  # 기본 설정: approve, edit, reject 모두 허용
        "write_file": {"allowed_decisions": ["approve", "reject"]},  # 승인/거부만 가능
        "edit_file": {"allowed_decisions": ["approve", "edit", "reject"]},  # 모든 옵션
        "read_file": False  # 인터럽트 비활성화
    },
    checkpointer=checkpointer,  # 필수
    system_prompt="안전 모드 에이전트"
)

# 사용자 승인이 필요한 작업 실행
config = {"configurable": {"thread_id": "thread-1"}}
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "시스템 파일을 수정해주세요."}
    ]
}, config=config)

# 중단점에서 의사결정
if "action_requests" in result:
    decisions = []
    for action in result["action_requests"]:
        # 옵션: "approve" (승인), "edit" (수정 후 실행), "reject" (거부)
        decisions.append({
            "decision": "approve",  # 또는 "edit", "reject"
            "args": action["args"]  # edit 선택 시 수정된 인수 전달
        })

    # 결정 적용
    agent.invoke({"decisions": decisions}, config=config)
```

의사결정 옵션:
- `approve`: 원래 인수로 도구 실행
- `edit`: 인수를 수정한 후 실행
- `reject`: 도구 호출 건너뛰기

### 5. 시스템 프롬프트 베스트 프랙티스

효과적인 에이전트를 만들기 위한 시스템 프롬프트 작성 가이드입니다:

권장 사항:
- 긍정적 지시 사용: "~하지 마세요" 대신 "~하세요"로 작성
  ```python
  # 나쁜 예
  "파일을 삭제하지 마세요"

  # 좋은 예
  "파일을 수정하기 전에 항상 백업을 만드세요"
  ```

- 구체적이고 명확한 지시: 모호한 표현 대신 구체적인 행동 지침 제공
  ```python
  system_prompt="""
  작업 수행 절차:
  1. 먼저 write_todos로 작업 계획을 수립합니다
  2. 각 단계를 순서대로 실행합니다
  3. 결과를 파일로 저장합니다
  4. 최종 요약을 제공합니다
  """
  ```

- 단일 책임 원칙: 에이전트에게 하나의 명확한 역할 부여
  ```python
  # 좁고 명확한 범위 (권장)
  "Python 코드 리뷰 전문가"

  # 너무 광범위한 범위 (비권장)
  "모든 프로그래밍 언어의 코드를 작성하고 리뷰하고 배포하는 전문가"
  ```

- 도구 거부 처리: 도구 호출이 거부되었을 때의 행동 지침
  ```python
  system_prompt="""
  도구 호출이 거부되면:
  1. 거부를 즉시 수용하고 동일한 명령을 재시도하지 마세요
  2. 왜 거부되었는지 이해했음을 설명하세요
  3. 대안을 제시하세요
  """
  ```

피해야 할 사항:
- 미들웨어 지시사항 중복 (자동으로 추가됨)
- 기본 지시사항과 모순되는 내용
- 지나치게 긴 프롬프트 (컨텍스트 낭비)

### 6. 커스텀 도구 추가

자신만의 도구를 추가할 수 있습니다:

```python
from langchain.tools import tool

@tool
def calculate_statistics(data: list) -> dict:
    """주어진 데이터의 통계를 계산합니다."""
    import statistics
    return {
        "mean": statistics.mean(data),
        "median": statistics.median(data),
        "stdev": statistics.stdev(data) if len(data) > 1 else 0
    }

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """이메일을 전송합니다."""
    # 실제 이메일 전송 로직
    return f"이메일이 {to}로 전송되었습니다."

# 커스텀 도구와 함께 에이전트 생성
agent = create_deep_agent(
    tools=[calculate_statistics, send_email],
    system_prompt="데이터 분석 및 이메일 전송 에이전트"
)
```

## 실전 예제: Deep Research Agent

DeepAgents의 강력함을 보여주는 실전 예제로, 웹에서 정보를 수집하고 분석하는 연구 에이전트를 만들어봅시다:

```python
from langchain_community.tools.tavily_search import TavilySearchResults
from deepagents import create_deep_agent

# Tavily 웹 검색 도구
internet_search = TavilySearchResults(max_results=5)

# Deep Research Agent 생성
research_agent = create_deep_agent(
    tools=[internet_search],
    system_prompt="""
    당신은 전문 리서치 에이전트입니다.

    작업 방식:
    1. 주어진 주제에 대해 다각도로 조사합니다.
    2. 신뢰할 수 있는 출처를 우선시합니다.
    3. 발견한 정보를 체계적으로 정리합니다.
    4. 최종 보고서를 마크다운 형식으로 작성합니다.
    5. 모든 출처를 명시합니다.

    서브 에이전트 활용:
    - 복잡한 주제는 여러 서브 에이전트에게 병렬로 조사를 위임합니다.
    - 각 서브 에이전트는 특정 측면을 깊이 있게 조사합니다.
    """
)

# 연구 수행
result = research_agent.invoke({
    "messages": [{
        "role": "user",
        "content": """
        2025년 생성형 AI의 주요 트렌드에 대해 포괄적으로 조사하고,
        다음 내용을 포함한 보고서를 작성해주세요:

        1. 주요 모델 발전 현황
        2. 산업별 적용 사례
        3. 윤리적 이슈 및 규제 동향
        4. 향후 전망

        각 섹션마다 최소 3개 이상의 신뢰할 수 있는 출처를 인용해주세요.
        """
    }]
})

print(result)
```

### 병렬 서브 에이전트 활용 예시

```python
# 에이전트가 자동으로 작업을 분해하고 서브 에이전트를 생성합니다:
#
# 1. write_todos로 다음 계획 수립:
#    - AI 모델 발전 조사
#    - 산업별 적용 사례 조사
#    - 윤리 및 규제 조사
#    - 향후 전망 분석
#    - 보고서 작성
#
# 2. task 도구로 3개의 서브 에이전트를 병렬 실행:
#    - SubAgent 1: AI 모델 발전 조사
#    - SubAgent 2: 산업별 적용 사례 조사
#    - SubAgent 3: 윤리 및 규제 조사
#
# 3. 각 서브 에이전트의 결과를 통합
#
# 4. write_file로 최종 보고서 작성
```

### 스트리밍으로 실시간 진행 상황 확인

```python
for chunk in research_agent.stream({
    "messages": [{
        "role": "user",
        "content": "양자 컴퓨팅의 현재와 미래에 대해 조사해주세요."
    }]
}):
    # 실시간으로 에이전트의 사고 과정과 작업 내용 출력
    if "messages" in chunk:
        for message in chunk["messages"]:
            if hasattr(message, "content"):
                print(message.content)

    # 도구 호출 내용도 확인 가능
    if "tool_calls" in chunk:
        print(f"도구 사용: {chunk['tool_calls']}")
```

## 프로덕션 배포

### LangSmith를 통한 배포

DeepAgents는 LangSmith와 완벽하게 통합되어 프로덕션 배포를 지원합니다:

```python
import os
from deepagents import create_deep_agent

# LangSmith 설정
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"
os.environ["LANGCHAIN_PROJECT"] = "deepagents-production"

# 에이전트 생성
agent = create_deep_agent(
    tools=[internet_search],
    system_prompt="프로덕션 연구 에이전트"
)

# LangSmith에서 모든 실행 내역, 성능 지표, 오류를 추적할 수 있습니다
```

### 관찰성 (Observability)

LangSmith를 통해 다음을 모니터링할 수 있습니다:
- 각 도구 호출 시간
- 토큰 사용량
- 오류 발생률
- 성능 병목 지점
- 비용 분석

### 평가 (Evaluation)

```python
from langchain.smith import RunEvalConfig

# 평가 설정
eval_config = RunEvalConfig(
    evaluators=["qa", "context_relevance", "faithfulness"]
)

# 에이전트 평가
results = agent.batch([
    {"messages": [{"role": "user", "content": "테스트 질문 1"}]},
    {"messages": [{"role": "user", "content": "테스트 질문 2"}]},
], config={"run_name": "evaluation_run"})
```

### CLI 도구 사용

DeepAgents는 명령줄 인터페이스도 제공합니다:

```bash
# CLI 설치
pip install deepagents-cli

# 에이전트 실행
deepagents run --prompt "웹에서 최신 AI 뉴스를 조사해주세요"

# 대화형 모드
deepagents chat

# 설정 파일 사용
deepagents run --config agent_config.yaml
```

## 마무리

DeepAgents는 복잡하고 장기적인 AI 에이전트 작업을 구축하기 위한 강력하고 유연한 프레임워크입니다.

### 주요 장점
- LangChain/LangGraph 생태계와의 완벽한 통합
- 작업 계획, 파일시스템 관리, 서브 에이전트 위임 등 강력한 기능
- 프로덕션 배포를 위한 LangSmith 지원
- MIT 라이센스로 상업적 사용 가능

### 사용 권장 사례
- 복잡한 리서치 및 데이터 분석
- 자동화된 코드 리뷰 및 리팩토링
- 멀티 스텝 워크플로우 자동화
- 장기 실행 작업 관리

### 추가 리소스
- [GitHub 리포지토리](https://github.com/langchain-ai/deepagents){:target="_blank"}
- [공식 문서](https://docs.langchain.com/oss/python/deepagents/overview){:target="_blank"}
- [Quickstarts 예제](https://github.com/langchain-ai/deepagents-quickstarts){:target="_blank"}
- [CLI 도구](https://github.com/langchain-ai/deepagents/tree/master/libs/deepagents-cli){:target="_blank"}

DeepAgents를 활용하여 더 똑똑하고 강력한 AI 에이전트를 만들어보세요.
