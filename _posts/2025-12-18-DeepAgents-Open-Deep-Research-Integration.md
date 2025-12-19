---
layout: post
title: DeepAgents + Open Deep Research - AI 기반 자동화 리서치 시스템 구축하기
author: 'Juho'
date: 2025-12-18 09:00:00 +0900
categories: [LangChain]
tags: [Python, LangChain, LangGraph, AI, Agent, DeepAgents, Research, OpenAI, Anthropic]
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
2. [DeepAgents 핵심 기능](#deepagents-핵심-기능)
3. [Open Deep Research 핵심 기능](#open-deep-research-핵심-기능)
4. [서브 에이전트 통합 아키텍처](#서브-에이전트-통합-아키텍처)
5. [기본 구현: 서브 에이전트 설정](#기본-구현-서브-에이전트-설정)
6. [서브 에이전트 설정 옵션 상세](#서브-에이전트-설정-옵션-상세)
7. [시나리오별 서브 에이전트 구성](#시나리오별-서브-에이전트-구성)
8. [실전 시나리오: 기술 트렌드 분석 시스템](#실전-시나리오-기술-트렌드-분석-시스템)
9. [디버깅 및 트러블슈팅](#디버깅-및-트러블슈팅)
10. [베스트 프랙티스](#베스트-프랙티스)
11. [성능 최적화 및 모니터링](#성능-최적화-및-모니터링)
12. [마무리](#마무리)

## 개요

AI 에이전트를 활용한 자동화 리서치 시스템은 현대 데이터 분석과 의사결정에서 핵심적인 역할을 합니다. 이번 글에서는 LangChain의 두 가지 강력한 프로젝트인 DeepAgents와 Open Deep Research를 함께 사용하여, 복잡한 장기 작업을 처리하면서도 심층적인 리서치 능력을 갖춘 AI 시스템을 구축하는 방법을 소개합니다.

### 왜 두 프로젝트를 함께 사용하는가?

DeepAgents는 범용 에이전트 프레임워크로서 다음과 같은 강점을 가집니다:
- 복잡한 작업의 계획 및 분해
- 파일시스템 백엔드를 통한 컨텍스트 관리
- 서브 에이전트 위임을 통한 병렬 작업 처리

Open Deep Research는 리서치 특화 에이전트로서:
- 다양한 LLM 제공자 지원 (OpenAI, Anthropic, 로컬 모델)
- 강력한 검색 통합 (Tavily, MCP, 네이티브 웹 검색)
- Deep Research Bench에서 검증된 성능 (6위, RACE 점수 0.4344)

이 두 프로젝트를 조합하면:
1. DeepAgents의 작업 분해 및 관리 능력으로 복잡한 워크플로우 구성
2. Open Deep Research의 전문적인 리서치 기능을 서브 에이전트로 활용
3. 각 시스템의 강점을 살린 최적의 자동화 리서치 시스템 구축

### 프로젝트 정보

| 프로젝트 | GitHub | 라이센스 | 주요 기능 |
|---------|--------|----------|-----------|
| DeepAgents | [langchain-ai/deepagents](https://github.com/langchain-ai/deepagents){:target="_blank"} | MIT | 작업 계획, 파일시스템, 서브 에이전트 |
| Open Deep Research | [langchain-ai/open_deep_research](https://github.com/langchain-ai/open_deep_research){:target="_blank"} | MIT | 심층 리서치, 다중 모델, 검색 통합 |

## DeepAgents 핵심 기능

DeepAgents는 LangChain과 LangGraph 기반의 에이전트 프레임워크로, 복잡한 장기 작업을 처리하도록 설계되었습니다.

### 주요 특징

1. 작업 계획 시스템
```python
# 에이전트가 자동으로 작업을 분해하고 관리
agent.invoke({
    "messages": [{
        "role": "user",
        "content": "복잡한 프로젝트 분석"
    }]
})
# 내부적으로 write_todos, read_todos 도구 사용
```

2. 파일시스템 백엔드
- 대용량 컨텍스트를 메모리가 아닌 파일로 관리
- 컨텍스트 윈도우 오버플로우 방지
- 10가지 파일 작업 도구 제공 (ls, read_file, write_file, edit_file, glob, grep 등)

3. 서브 에이전트 위임
   ```python
   # 전문화된 서브 에이전트 생성
   research_subagent = {
       "name": "research-agent",
       "description": "깊이 있는 질문 조사용",
       "prompt": "전문 연구원입니다",
       "tools": [internet_search],
       "model": "openai:gpt-4o"
   }

   agent = create_deep_agent(subagents=[research_subagent])
   ```

4. 자동 컨텍스트 관리
- 170K 토큰 초과 시 자동 요약
- Anthropic 모델 사용 시 프롬프트 캐싱으로 비용 절감

## Open Deep Research 핵심 기능

Open Deep Research는 자동화된 심층 리서치를 수행하는 특화 에이전트입니다.

### 주요 특징

1. 다중 모델 지원

| 작업 단계 | 기본 모델 | 역할 |
|----------|----------|------|
| 요약 | gpt-4.1-mini | 검색 결과 요약 |
| 연구 | gpt-4.1 | 에이전트 구동 |
| 압축 | gpt-4.1 | 발견사항 압축 |
| 최종 보고서 | gpt-4.1 | 리포트 작성 |

2. 검색 통합
- Tavily 검색 API (기본)
- MCP (Model Context Protocol) 서버
- Anthropic/OpenAI 네이티브 웹 검색

3. 검증된 성능
- Deep Research Bench 리더보드 6위
- RACE 점수: 0.4344
- 100개의 박사급 연구 과제 데이터셋으로 평가

## 서브 에이전트 통합 아키텍처

DeepAgents와 Open Deep Research를 서브 에이전트 모델로 통합하는 아키텍처입니다.

### 아키텍처 개요

```
┌─────────────────────────────────────────────────────┐
│         DeepAgents (메인 에이전트)                    │
│                                                     │
│  역할:                                              │
│  - 사용자 요청 수신 및 분석                          │
│  - write_todos로 작업 계획 수립                     │
│  - 작업을 서브 에이전트에게 위임                     │
│  - 파일시스템을 통한 결과 저장 및 관리                │
│  - 최종 결과 통합 및 보고서 생성                     │
└────────────┬────────────────────────────────────────┘
             │ task 도구로 위임
             │
    ┌────────┴────────┬───────────┬──────────────┐
    │                 │           │              │
    ▼                 ▼           ▼              ▼
┌────────┐      ┌──────────┐  ┌──────────┐  ┌──────────┐
│Research│      │Research  │  │Analysis  │  │Synthesis │
│Agent 1 │      │Agent 2   │  │Agent     │  │Agent     │
└────────┘      └──────────┘  └──────────┘  └──────────┘
    │                 │           │              │
    │ 병렬 실행        │           │              │
    └─────────────────┴───────────┴──────────────┘
                      │
                      ▼
              ┌──────────────┐
              │ 파일시스템    │
              │ (중간 결과)   │
              └──────────────┘
                      │
                      ▼
              ┌──────────────┐
              │ 메인 에이전트 │
              │ (결과 통합)   │
              └──────────────┘
```

### 주요 구성 요소

1. 메인 에이전트 (Orchestrator)
- 전체 워크플로우 조율
- 작업 계획 수립 (write_todos)
- 서브 에이전트 호출 (task 도구)
- 결과 통합 및 최종 보고서 생성

2. 리서치 서브 에이전트 (Research Agents)
- Open Deep Research 스타일의 심층 조사
- 웹 검색 및 정보 수집
- 구조화된 결과 반환
- 병렬 실행 가능

3. 분석 서브 에이전트 (Analysis Agent)
- 리서치 결과 분석
- 트렌드 및 패턴 식별
- 인사이트 도출

4. 종합 서브 에이전트 (Synthesis Agent)
- 여러 서브 에이전트 결과 통합
- 전략적 권장사항 제시
- 최종 결론 도출

### 통신 및 데이터 흐름

```
1. 사용자 요청
   └─► 메인 에이전트

2. 작업 분해 (write_todos)
   └─► TODO 리스트 생성

3. 서브 에이전트 호출 (병렬)
   ├─► Research Agent 1 → internet_search → write_file(result_1.md)
   ├─► Research Agent 2 → internet_search → write_file(result_2.md)
   └─► Research Agent 3 → internet_search → write_file(result_3.md)

4. 분석 단계
   └─► Analysis Agent → read_file(results) → 분석 → write_file(analysis.md)

5. 종합 단계
   └─► Synthesis Agent → read_file(all) → 통합 → write_file(synthesis.md)

6. 최종 보고서
   └─► 메인 에이전트 → read_file(all) → 보고서 작성 → write_file(final.md)
```

## 기본 구현: 서브 에이전트 설정

DeepAgents의 서브 에이전트 기능을 활용하여 Open Deep Research 스타일의 리서치 시스템을 구축합니다.

### 환경 설정

```bash
# 필수 패키지 설치
pip install deepagents tavily-python langchain-openai langchain-anthropic

# Open Deep Research 클론 (참조용)
git clone https://github.com/langchain-ai/open_deep_research.git
cd open_deep_research
uv sync
```

### 환경 변수 설정

```bash
# .env 파일
OPENAI_API_KEY=your-openai-key
ANTHROPIC_API_KEY=your-anthropic-key
TAVILY_API_KEY=your-tavily-key
LANGCHAIN_API_KEY=your-langsmith-key  # 선택사항
LANGCHAIN_TRACING_V2=true  # 선택사항
```

### 구현 코드

```python
import os
from deepagents import create_deep_agent
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# 1. 검색 도구 설정
internet_search = TavilySearchResults(
    max_results=5,
    api_key=os.environ["TAVILY_API_KEY"]
)

# 2. Open Deep Research 스타일의 서브 에이전트 정의
deep_research_subagent = {
    "name": "deep-research-agent",
    "description": """
    복잡한 주제에 대한 심층 리서치를 수행합니다.
    다음 작업에 특화되어 있습니다:
    - 다각도 정보 수집
    - 출처 신뢰성 검증
    - 구조화된 보고서 작성
    - 발견사항 요약 및 압축
    """,
    "prompt": """
    당신은 전문 리서치 에이전트입니다.

    작업 프로세스:
    1. 주어진 주제를 세부 질문으로 분해합니다.
    2. 각 질문에 대해 웹 검색을 수행합니다.
    3. 신뢰할 수 있는 출처를 우선시합니다.
    4. 발견한 정보를 체계적으로 정리합니다.
    5. 마크다운 형식의 보고서를 작성합니다.
    6. 모든 출처를 명시합니다.

    보고서 구조:
    ## 요약
    - 핵심 발견사항 3-5개 항목

    ## 상세 분석
    - 주제별 심층 분석
    - 데이터 및 통계

    ## 출처
    - 모든 참조 자료 목록
    """,
    "tools": [internet_search],
    "model": "openai:gpt-4o"  # 또는 "anthropic:claude-3-5-sonnet-20241022"
}

# 3. 분석 전문 서브 에이전트
analysis_subagent = {
    "name": "analysis-agent",
    "description": "리서치 결과를 분석하고 인사이트를 도출합니다.",
    "prompt": """
    당신은 데이터 분석 전문가입니다.

    리서치 결과를 받아서:
    1. 주요 트렌드를 식별합니다
    2. 패턴과 상관관계를 분석합니다
    3. 실행 가능한 인사이트를 도출합니다
    4. 데이터 시각화 권장사항을 제시합니다
    """,
    "tools": [],
    "model": "openai:gpt-4o"
}

# 4. 메인 에이전트 생성
main_agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    subagents=[deep_research_subagent, analysis_subagent],
    tools=[internet_search],
    system_prompt="""
    당신은 프로젝트 관리자이자 리서치 오케스트레이터입니다.

    작업 방식:
    1. write_todos로 전체 프로젝트 계획을 수립합니다.
    2. 복잡한 리서치 작업은 deep-research-agent에게 위임합니다.
    3. 분석 작업은 analysis-agent에게 위임합니다.
    4. 서브 에이전트의 결과를 통합합니다.
    5. 최종 보고서를 파일로 저장합니다.

    파일 관리:
    - 각 서브 에이전트의 결과는 별도 파일로 저장
    - 최종 통합 보고서는 final_report.md로 저장
    """
)

# 5. 에이전트 실행
result = main_agent.invoke({
    "messages": [{
        "role": "user",
        "content": """
        2025년 생성형 AI의 주요 트렌드를 조사하고 분석해주세요.

        다음 내용을 포함해주세요:
        1. 주요 모델 발전 현황 (GPT-5, Claude 4, Gemini 2 등)
        2. 산업별 적용 사례 (헬스케어, 금융, 교육)
        3. 윤리적 이슈 및 규제 동향
        4. 향후 6개월 전망

        각 섹션마다 최소 3개 이상의 신뢰할 수 있는 출처를 인용하고,
        데이터 기반 인사이트를 도출해주세요.
        """
    }]
})

print(result)
```

### 실행 흐름

```
1. 메인 에이전트: write_todos로 작업 계획 수립
   ├─ TODO 1: 주요 모델 발전 현황 조사
   ├─ TODO 2: 산업별 적용 사례 조사
   ├─ TODO 3: 윤리 및 규제 조사
   ├─ TODO 4: 향후 전망 분석
   └─ TODO 5: 최종 보고서 작성

2. task 도구로 deep-research-agent 호출 (TODO 1-3 병렬 실행)
   ├─ SubAgent 1: 모델 발전 현황 (internet_search 사용)
   ├─ SubAgent 2: 산업별 사례 (internet_search 사용)
   └─ SubAgent 3: 윤리 및 규제 (internet_search 사용)

3. write_file로 각 결과 저장
   ├─ research_models.md
   ├─ research_industries.md
   └─ research_ethics.md

4. task 도구로 analysis-agent 호출 (TODO 4)
   └─ 저장된 리서치 파일들을 read_file로 읽고 분석

5. write_file로 최종 보고서 저장
   └─ final_report.md
```

### 스트리밍으로 실시간 진행 상황 확인

```python
# 실시간 진행 상황 모니터링
for chunk in main_agent.stream({
    "messages": [{
        "role": "user",
        "content": "양자 컴퓨팅의 현재와 미래에 대해 조사해주세요."
    }]
}):
    # 메시지 출력
    if "messages" in chunk:
        for message in chunk["messages"]:
            if hasattr(message, "content"):
                print(f"\n[메시지] {message.content}")

    # 도구 호출 모니터링
    if "tool_calls" in chunk:
        for tool_call in chunk["tool_calls"]:
            print(f"[도구 사용] {tool_call['name']}")

    # 서브 에이전트 활동 모니터링
    if "task" in chunk:
        print(f"[서브 에이전트] {chunk['task']['subagent_name']} 실행 중...")
```

## 서브 에이전트 설정 옵션 상세

서브 에이전트의 각 파라미터를 상세히 살펴봅니다.

### 필수 파라미터

#### 1. name (문자열, 필수)

서브 에이전트의 고유 식별자입니다.

```python
subagent = {
    "name": "research-agent",  # 고유한 이름, 대시(-) 또는 언더스코어(_) 사용 권장
    # ...
}
```

주의사항:
- 같은 메인 에이전트 내에서 중복된 이름 사용 불가
- 공백 대신 대시(-) 또는 언더스코어(_) 사용
- 명확하고 설명적인 이름 권장 (예: `tech-research-agent`, `competitor-analyst`)

#### 2. description (문자열, 필수)

서브 에이전트가 수행할 작업에 대한 설명입니다. 메인 에이전트가 이 설명을 바탕으로 적절한 서브 에이전트를 선택합니다.

```python
subagent = {
    "name": "deep-research-agent",
    "description": """
    복잡한 주제에 대한 심층 리서치를 수행합니다.

    특화 영역:
    - 학술 논문 및 연구 자료 분석
    - 다각도 정보 수집 및 검증
    - 출처 신뢰성 평가
    - 구조화된 보고서 작성

    최적 사용 사례:
    - 기술 동향 분석
    - 경쟁사 분석
    - 시장 조사
    - 학술 연구
    """,
    # ...
}
```

베스트 프랙티스:
- 서브 에이전트의 역할을 명확하게 기술
- 특화된 능력과 제한사항 명시
- 사용 사례 예시 포함
- 200-500자 권장

#### 3. prompt (문자열, 필수)

서브 에이전트의 시스템 프롬프트입니다. 서브 에이전트의 행동 방식, 작업 프로세스, 출력 형식을 정의합니다.

```python
subagent = {
    "name": "research-agent",
    "description": "심층 리서치 수행",
    "prompt": """
    당신은 전문 리서치 에이전트입니다.

    ## 역할
    주어진 주제에 대해 심층적이고 체계적인 리서치를 수행합니다.

    ## 작업 프로세스
    1. 주제 분석 및 세부 질문 도출
    2. 각 질문에 대해 웹 검색 수행
    3. 신뢰할 수 있는 출처 우선 선택
       - 학술 논문, 공식 문서, 정부 기관 자료
       - 최신 데이터 (2년 이내)
    4. 정보 교차 검증
    5. 구조화된 보고서 작성

    ## 출력 형식
    ### 요약
    - 핵심 발견사항 3-5개 (각 1-2문장)

    ### 상세 분석
    #### [주제 1]
    - 주요 내용
    - 데이터 및 통계
    - 출처: [링크]

    #### [주제 2]
    ...

    ### 결론
    - 종합적인 결론 (3-5문장)

    ### 참고 자료
    1. [제목] - [URL] - [날짜]
    2. ...

    ## 주의사항
    - 모든 주장에는 출처 명시
    - 데이터는 최신 정보 우선
    - 편향되지 않은 객관적 분석
    - 불확실한 정보는 명시적으로 표시
    """,
    # ...
}
```

프롬프트 작성 팁:
- 역할, 프로세스, 출력 형식을 명확히 구분
- 구체적인 지시사항 제공
- 예상되는 edge case 처리 방법 명시
- 품질 기준 설정

#### 4. tools (리스트, 필수)

서브 에이전트가 사용할 수 있는 도구 목록입니다.

```python
from langchain_community.tools.tavily_search import TavilySearchResults
from langchain.tools import tool

# 검색 도구
internet_search = TavilySearchResults(max_results=5)

# 커스텀 도구
@tool
def calculate_metrics(data: dict) -> str:
    """데이터 메트릭 계산"""
    # ...
    return result

subagent = {
    "name": "research-agent",
    "description": "리서치 수행",
    "prompt": "...",
    "tools": [internet_search, calculate_metrics],  # 여러 도구 가능
    "model": "openai:gpt-4o"
}

# 도구 없는 서브 에이전트 (분석만 수행)
analysis_agent = {
    "name": "analysis-agent",
    "description": "데이터 분석",
    "prompt": "...",
    "tools": [],  # 빈 리스트
    "model": "openai:gpt-4o"
}
```

도구 선택 가이드:
- 리서치 에이전트: 검색 도구 (Tavily, DuckDuckGo 등)
- 분석 에이전트: 계산/통계 도구
- 데이터 에이전트: DB 접근 도구
- 종합 에이전트: 도구 불필요 (기존 결과만 통합)

### 선택적 파라미터

#### 5. model (문자열, 선택)

서브 에이전트가 사용할 LLM 모델을 지정합니다.

```python
subagent = {
    "name": "research-agent",
    "description": "리서치 수행",
    "prompt": "...",
    "tools": [internet_search],
    "model": "openai:gpt-4o"  # 형식: "provider:model-name"
}
```

지원 모델:

| 프로바이더 | 모델 예시 | 권장 용도 |
|----------|---------|----------|
| OpenAI | `openai:gpt-4o` | 범용, 고성능 |
| OpenAI | `openai:gpt-4o-mini` | 빠른 처리, 비용 절감 |
| Anthropic | `anthropic:claude-3-5-sonnet-20241022` | 긴 컨텍스트, 분석 |
| Anthropic | `anthropic:claude-3-5-haiku-20241022` | 빠른 응답, 간단한 작업 |
| Google | `google:gemini-2.0-flash-exp` | 멀티모달 |

모델 선택 전략:

```python
# 비용 최적화 전략
subagents = [
    {
        "name": "simple-research",
        "model": "openai:gpt-4o-mini",  # 간단한 작업
        # ...
    },
    {
        "name": "deep-analysis",
        "model": "anthropic:claude-3-5-sonnet-20241022",  # 복잡한 분석
        # ...
    },
    {
        "name": "final-synthesis",
        "model": "openai:gpt-4o",  # 최종 보고서
        # ...
    }
]
```

미지정 시 동작:
- 서브 에이전트의 모델을 지정하지 않으면 메인 에이전트의 모델 사용
- 메인 에이전트 모델도 미지정 시 기본값: `claude-sonnet-4-5-20250929`

### 설정 예시 모음

#### 예시 1: 최소 설정

```python
minimal_subagent = {
    "name": "quick-search",
    "description": "빠른 웹 검색",
    "prompt": "주어진 질문에 대해 웹 검색을 수행하고 요약합니다.",
    "tools": [internet_search]
    # model 미지정 → 메인 에이전트 모델 사용
}
```

#### 예시 2: 완전한 설정

```python
complete_subagent = {
    "name": "comprehensive-researcher",
    "description": """
    포괄적인 기술 리서치를 수행합니다.
    - 학술 논문 분석
    - 산업 동향 파악
    - 경쟁사 분석
    """,
    "prompt": """
    전문 기술 리서치 에이전트입니다.

    작업 절차:
    1. 주제를 5-7개 세부 질문으로 분해
    2. 각 질문에 대해 최소 3개 출처 확인
    3. 정보 교차 검증 및 신뢰도 평가
    4. 구조화된 마크다운 보고서 작성

    출력 형식:
    ## 요약 (200자 이내)
    ## 주요 발견사항 (3-5개 항목)
    ## 상세 분석
    ## 데이터 및 통계
    ## 결론 및 시사점
    ## 참고 자료 (최소 5개)
    """,
    "tools": [
        internet_search,
        arxiv_search,  # 학술 논문 검색
        company_data_tool  # 기업 데이터 조회
    ],
    "model": "anthropic:claude-3-5-sonnet-20241022"
}
```

## 시나리오별 서브 에이전트 구성

다양한 사용 사례별로 최적화된 서브 에이전트 구성을 소개합니다.

### 시나리오 1: 학술 연구 (Academic Research)

```python
from langchain_community.tools import ArxivQueryRun

# arXiv 검색 도구
arxiv_search = ArxivQueryRun()

# 학술 연구용 서브 에이전트들
academic_subagents = [
    {
        "name": "literature-review-agent",
        "description": "최신 학술 논문을 조사하고 문헌 리뷰를 수행합니다.",
        "prompt": """
        학술 문헌 리뷰 전문가입니다.

        작업 방식:
        1. arXiv 및 학술 검색 엔진에서 관련 논문 검색
        2. 최근 3년 이내 논문 우선 (최신 연구)
        3. 인용 횟수가 높은 영향력 있는 논문 식별
        4. 연구 방법론, 주요 발견사항, 한계점 분석
        5. 연구 갭(gap) 식별

        출력:
        ## 주요 논문 (5-10개)
        - [제목] (저자, 연도)
          - 핵심 기여
          - 방법론
          - 결과
          - arXiv ID 또는 DOI

        ## 연구 동향
        - 주요 연구 방향
        - 새로운 기법

        ## 연구 갭
        - 아직 해결되지 않은 문제
        - 향후 연구 방향
        """,
        "tools": [arxiv_search, internet_search],
        "model": "anthropic:claude-3-5-sonnet-20241022"  # 긴 논문 분석에 유리
    },
    {
        "name": "methodology-analysis-agent",
        "description": "연구 방법론을 분석하고 비교합니다.",
        "prompt": """
        연구 방법론 분석 전문가입니다.

        수집된 논문들의 방법론을 분석하여:
        1. 각 연구의 실험 설계 평가
        2. 데이터셋 및 벤치마크 비교
        3. 통계적 유의성 검토
        4. 재현 가능성 평가
        5. 방법론적 강점과 약점 식별

        출력:
        ## 방법론 비교표
        | 논문 | 데이터셋 | 방법 | 성능 | 한계 |

        ## 권장 방법론
        - 이 연구에 적합한 방법
        - 근거
        """,
        "tools": [],  # 파일 읽기만 사용
        "model": "openai:gpt-4o"
    }
]

# 메인 에이전트
academic_main_agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    subagents=academic_subagents,
    tools=[arxiv_search, internet_search],
    system_prompt="""
    학술 연구 오케스트레이터입니다.

    작업 흐름:
    1. 연구 주제 수신
    2. write_todos로 리뷰 계획 수립
    3. literature-review-agent에게 논문 조사 위임
    4. 결과를 papers.md에 저장
    5. methodology-analysis-agent에게 방법론 분석 위임
    6. 최종 문헌 리뷰 보고서 작성 (academic_review.md)
    """
)
```

사용 예시:

```python
result = academic_main_agent.invoke({
    "messages": [{
        "role": "user",
        "content": """
        Transformer 모델의 효율성 개선 기법에 대한 문헌 리뷰를 작성해주세요.

        초점:
        - 최근 3년 (2023-2025) 연구
        - Attention 메커니즘 최적화
        - 추론 속도 개선
        - 메모리 효율성
        """
    }]
})
```

### 시나리오 2: 비즈니스 인텔리전스

```python
# 비즈니스 분석용 서브 에이전트들
business_subagents = [
    {
        "name": "market-research-agent",
        "description": "시장 규모, 성장률, 주요 플레이어를 조사합니다.",
        "prompt": """
        시장 조사 전문가입니다.

        분석 항목:
        1. 시장 규모 (TAM, SAM, SOM)
        2. 연평균 성장률 (CAGR)
        3. 주요 시장 세그먼트
        4. 지역별 시장 분포
        5. 성장 동인 및 저해 요인

        출력 형식:
        ## 시장 개요
        - 현재 시장 규모: $X billion
        - 예상 규모 (2030): $Y billion
        - CAGR: Z%

        ## 주요 플레이어
        1. [회사명] - 시장 점유율 X%
           - 강점
           - 약점

        ## 시장 동향
        - 성장 동인
        - 도전 과제

        모든 수치는 출처와 함께 명시
        """,
        "tools": [internet_search],
        "model": "openai:gpt-4o"
    },
    {
        "name": "competitor-intel-agent",
        "description": "경쟁사의 전략, 제품, 재무 상태를 분석합니다.",
        "prompt": """
        경쟁 인텔리전스 전문가입니다.

        분석 프레임워크:
        1. 제품/서비스 포트폴리오
        2. 가격 전략
        3. 마케팅 채널
        4. 기술 스택
        5. 최근 뉴스 (투자, 인수, 파트너십)
        6. 재무 성과 (매출, 이익)
        7. 채용 동향 (기술 투자 방향 추론)

        출력:
        ## [경쟁사 1]
        ### 개요
        - 설립: 연도
        - 직원 수
        - 본사

        ### 제품/서비스
        - 주력 제품
        - 가격: $X/month
        - 특징

        ### 최근 동향
        - 뉴스 1 (출처, 날짜)
        - 뉴스 2

        ### SWOT 분석
        - Strengths
        - Weaknesses
        - Opportunities
        - Threats

        ### 위협도 평가: HIGH/MEDIUM/LOW
        """,
        "tools": [internet_search],
        "model": "anthropic:claude-3-5-sonnet-20241022"
    },
    {
        "name": "strategic-synthesis-agent",
        "description": "시장 조사와 경쟁사 분석을 종합하여 전략적 인사이트를 도출합니다.",
        "prompt": """
        전략 컨설턴트입니다.

        시장 조사와 경쟁사 분석 결과를 바탕으로:
        1. 시장 기회 식별
        2. 경쟁 포지셔닝 권장
        3. 차별화 전략 제시
        4. 진입 장벽 평가
        5. 실행 가능한 권장사항

        출력:
        ## 핵심 인사이트 (3-5개)

        ## 전략적 권장사항
        ### 단기 (3-6개월)
        - 액션 1
        - 액션 2

        ### 중기 (6-12개월)
        - 액션 1

        ## 리스크 및 완화 전략
        - 리스크 1 → 완화책

        ## 성공 지표 (KPI)
        - 지표 1
        - 목표 값
        """,
        "tools": [],
        "model": "openai:gpt-4o"
    }
]

# 메인 에이전트
business_main_agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    subagents=business_subagents,
    tools=[internet_search],
    system_prompt="""
    비즈니스 인텔리전스 오케스트레이터입니다.

    작업 흐름:
    1. 분석 목표 수신
    2. write_todos로 분석 계획 수립
    3. market-research-agent와 competitor-intel-agent 병렬 실행
    4. 각 결과를 개별 파일로 저장
    5. strategic-synthesis-agent에게 종합 분석 위임
    6. 최종 비즈니스 인텔리전스 보고서 작성

    파일 관리:
    - market_analysis.md
    - competitor_analysis.md
    - strategic_insights.md
    - final_bi_report.md
    """
)
```

사용 예시:

```python
result = business_main_agent.invoke({
    "messages": [{
        "role": "user",
        "content": """
        AI 기반 코드 어시스턴트 시장에 대한 비즈니스 인텔리전스 보고서를 작성해주세요.

        분석 범위:
        - 시장: AI 코드 어시스턴트 (GitHub Copilot, Cursor, Codeium 등)
        - 경쟁사: GitHub, Cursor, Codeium, Tabnine, Amazon CodeWhisperer
        - 지역: 글로벌
        - 목표: 신규 진입 가능성 평가
        """
    }]
})
```

### 시나리오 3: 기술 실사 (Technical Due Diligence)

```python
# 기술 실사용 서브 에이전트들
tech_dd_subagents = [
    {
        "name": "tech-stack-analyzer",
        "description": "대상 기업의 기술 스택을 분석하고 평가합니다.",
        "prompt": """
        기술 스택 분석 전문가입니다.

        분석 항목:
        1. 프론트엔드 기술 (프레임워크, 라이브러리)
        2. 백엔드 기술 (언어, 프레임워크, API)
        3. 데이터베이스 및 스토리지
        4. 클라우드 인프라 (AWS, GCP, Azure)
        5. DevOps 및 CI/CD
        6. 보안 및 컴플라이언스
        7. 모니터링 및 관측성

        평가 기준:
        - 확장성 (Scalability)
        - 유지보수성 (Maintainability)
        - 최신성 (Modernity)
        - 보안성 (Security)
        - 비용 효율성 (Cost Efficiency)

        출력:
        ## 기술 스택 요약
        | 레이어 | 기술 | 버전 | 평가 | 비고 |

        ## 강점
        ## 약점
        ## 기술 부채 평가
        ## 현대화 권장사항
        """,
        "tools": [internet_search],
        "model": "anthropic:claude-3-5-sonnet-20241022"
    },
    {
        "name": "ip-patent-analyzer",
        "description": "지적재산권과 특허를 조사합니다.",
        "prompt": """
        지적재산권 분석 전문가입니다.

        조사 항목:
        1. 등록된 특허 (Patent)
        2. 출원 중인 특허
        3. 상표 (Trademark)
        4. 저작권 (Copyright)
        5. 오픈소스 라이선스 준수
        6. 제3자 IP 리스크

        출력:
        ## 특허 포트폴리오
        - 등록 특허: X건
        - 출원 중: Y건
        - 핵심 특허 목록

        ## 오픈소스 사용 현황
        - 라이선스: MIT, Apache 2.0 등
        - 컴플라이언스 이슈: 없음/있음

        ## IP 리스크 평가
        - 잠재적 침해 리스크
        - 완화 방안
        """,
        "tools": [internet_search],
        "model": "openai:gpt-4o"
    },
    {
        "name": "team-capability-assessor",
        "description": "개발팀의 역량과 조직 구조를 평가합니다.",
        "prompt": """
        엔지니어링 팀 평가 전문가입니다.

        평가 영역:
        1. 팀 크기 및 구조
        2. 핵심 인재 (LinkedIn, GitHub 프로필 기반)
        3. 기술 스킬셋 분포
        4. 오픈소스 기여도
        5. 최근 채용 동향
        6. 이직률 추정

        출력:
        ## 팀 구성
        - 총 엔지니어: X명
        - Frontend: Y명
        - Backend: Z명
        - DevOps/SRE: N명

        ## 핵심 인재
        1. [이름] - [직책]
           - 경력: XX년
           - 전문 분야
           - GitHub 기여

        ## 기술 역량 평가
        - 전반적 수준: High/Medium/Low
        - 강점 영역
        - 약점 영역

        ## 조직 건강도
        - 채용 트렌드
        - 인재 유지율 추정
        """,
        "tools": [internet_search],
        "model": "anthropic:claude-3-5-sonnet-20241022"
    }
]

# 메인 에이전트
tech_dd_main_agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    subagents=tech_dd_subagents,
    tools=[internet_search],
    system_prompt="""
    기술 실사 오케스트레이터입니다.

    작업 흐름:
    1. 대상 기업 정보 수신
    2. write_todos로 실사 계획 수립
    3. 3개 서브 에이전트 병렬 실행:
       - tech-stack-analyzer
       - ip-patent-analyzer
       - team-capability-assessor
    4. 각 결과를 개별 파일로 저장
    5. 종합 평가 수행
    6. 최종 기술 실사 보고서 작성

    보고서 구조:
    - Executive Summary
    - 기술 스택 평가
    - IP 포트폴리오
    - 팀 역량
    - 리스크 및 기회
    - 가치 평가 (Valuation Impact)
    - 권장사항
    """
)
```

사용 예시:

```python
result = tech_dd_main_agent.invoke({
    "messages": [{
        "role": "user",
        "content": """
        다음 스타트업에 대한 기술 실사를 수행해주세요.

        회사: [회사명]
        웹사이트: [URL]
        GitHub: [org URL]

        실사 목적: 시리즈 B 투자 검토

        중점 확인 사항:
        - 기술 스택의 확장성
        - 핵심 인재 유지
        - IP 포트폴리오 가치
        - 기술 부채 수준
        """
    }]
})
```

## 실전 시나리오: 기술 트렌드 분석 시스템

실무에서 활용할 수 있는 완전한 시스템을 구축해봅시다.

### 시나리오

매주 자동으로 최신 AI 기술 트렌드를 조사하고, 경쟁사 분석을 포함한 보고서를 생성하는 시스템을 만듭니다.

### 전체 시스템 아키텍처

```python
from deepagents import create_deep_agent
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore
from langchain_openai import ChatOpenAI
from langchain_community.tools.tavily_search import TavilySearchResults
import os
from datetime import datetime

# 1. 장기 메모리 설정
store = InMemoryStore()
backend = CompositeBackend({
    "memories": StoreBackend(store=store),  # /memories/* 경로
    "default": StateBackend()  # 나머지
})

# 2. 검색 도구
internet_search = TavilySearchResults(max_results=5)

# 3. 전문 서브 에이전트들
subagents = [
    {
        "name": "trend-research-agent",
        "description": "최신 기술 트렌드를 조사합니다.",
        "prompt": """
        기술 트렌드 리서치 전문가입니다.

        조사 영역:
        - 새로운 논문 및 연구
        - 주요 기업의 기술 발표
        - 오픈소스 프로젝트 동향
        - 개발자 커뮤니티 반응

        출력 형식:
        ## 주요 트렌드
        1. [트렌드 1]
           - 설명
           - 영향도: HIGH/MEDIUM/LOW
           - 출처

        2. [트렌드 2]
           ...
        """,
        "tools": [internet_search],
        "model": "openai:gpt-4o"
    },
    {
        "name": "competitor-analysis-agent",
        "description": "경쟁사의 기술 전략을 분석합니다.",
        "prompt": """
        경쟁사 분석 전문가입니다.

        분석 항목:
        - 최근 제품 출시 및 업데이트
        - 채용 동향 (기술 스택 추론)
        - 특허 및 논문
        - 투자 및 인수합병

        출력 형식:
        ## [회사명]
        - 주요 동향
        - 기술 스택 변화
        - 전략적 방향
        - 위협도 평가
        """,
        "tools": [internet_search],
        "model": "openai:gpt-4o"
    },
    {
        "name": "synthesis-agent",
        "description": "리서치 결과를 종합하고 인사이트를 도출합니다.",
        "prompt": """
        전략 분석가입니다.

        역할:
        1. 트렌드와 경쟁사 분석 결과 통합
        2. 우리 조직에 미치는 영향 평가
        3. 실행 가능한 권장사항 제시
        4. 이전 보고서와 비교 분석 (장기 메모리 활용)

        출력 형식:
        ## 종합 분석
        - 핵심 인사이트
        - 영향 평가
        - 권장 조치

        ## 이전 대비 변화
        - 새로운 트렌드
        - 지속되는 트렌드
        - 약화된 트렌드
        """,
        "tools": [],  # 검색 없이 종합만
        "model": "openai:gpt-4o"
    }
]

# 4. 메인 오케스트레이터 에이전트
orchestrator = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    backend=backend,
    subagents=subagents,
    tools=[internet_search],
    system_prompt=f"""
    당신은 주간 기술 트렌드 분석 시스템의 오케스트레이터입니다.
    오늘 날짜: {datetime.now().strftime('%Y-%m-%d')}

    작업 프로세스:
    1. write_todos로 이번 주 분석 계획 수립
    2. trend-research-agent에게 트렌드 조사 위임
    3. competitor-analysis-agent에게 경쟁사 분석 위임 (병렬)
    4. 각 결과를 개별 파일로 저장
    5. synthesis-agent에게 종합 분석 위임
    6. /memories/previous_reports/ 에서 이전 보고서 읽기
    7. 최종 보고서 생성 및 저장
    8. 이번 보고서를 /memories/previous_reports/{date}.md 에 저장

    파일 구조:
    - trends_{date}.md: 트렌드 리서치 결과
    - competitors_{date}.md: 경쟁사 분석 결과
    - synthesis_{date}.md: 종합 분석
    - final_report_{date}.md: 최종 보고서
    - /memories/previous_reports/{date}.md: 장기 보존용

    경쟁사 목록: OpenAI, Google DeepMind, Anthropic, Meta AI
    """
)

# 5. 실행 함수
def run_weekly_analysis():
    """주간 트렌드 분석 실행"""

    date_str = datetime.now().strftime('%Y-%m-%d')

    result = orchestrator.invoke({
        "messages": [{
            "role": "user",
            "content": f"""
            {date_str} 주간 AI 기술 트렌드 분석을 수행해주세요.

            분석 범위:
            1. 최신 AI 모델 및 기술 (지난 7일)
            2. 주요 경쟁사 동향 (OpenAI, Google DeepMind, Anthropic, Meta AI)
            3. 오픈소스 프로젝트 트렌드
            4. 산업 적용 사례

            이전 보고서와 비교하여 변화 추이를 분석해주세요.
            """
        }]
    })

    return result

# 6. 실행
if __name__ == "__main__":
    result = run_weekly_analysis()
    print("Weekly analysis completed!")
    print(result)
```

### 실행 결과 예시

```
주간 AI 기술 트렌드 분석 완료
================================================================================

작업 계획 (write_todos):
✓ 1. 최신 AI 트렌드 조사
✓ 2. 경쟁사 분석
✓ 3. 종합 분석
✓ 4. 이전 보고서 비교
✓ 5. 최종 보고서 작성

서브 에이전트 실행:
  [trend-research-agent] 실행 완료
    → 5개 주요 트렌드 식별
    → trends_2025-12-15.md 저장

  [competitor-analysis-agent] 실행 완료
    → 4개 경쟁사 분석 완료
    → competitors_2025-12-15.md 저장

  [synthesis-agent] 실행 완료
    → 이전 4주 보고서와 비교
    → synthesis_2025-12-15.md 저장

최종 보고서:
  → final_report_2025-12-15.md (생성 완료)
  → /memories/previous_reports/2025-12-15.md (장기 보존)

핵심 인사이트:
1. Gemini 2.0의 멀티모달 기능 강화
2. OpenAI의 실시간 API 베타 출시
3. Anthropic의 컴퓨터 사용 기능 확대
4. 오픈소스 모델의 품질 향상 (Llama 3.3)

권장 조치:
- 멀티모달 기능 통합 검토
- 실시간 처리 파이프라인 개선
- 비용 절감을 위한 오픈소스 모델 테스트
```

### 자동화 및 스케줄링

```python
# cron 또는 Airflow로 주간 실행 자동화
from apscheduler.schedulers.blocking import BlockingScheduler

scheduler = BlockingScheduler()

@scheduler.scheduled_job('cron', day_of_week='mon', hour=9)
def scheduled_analysis():
    """매주 월요일 오전 9시 실행"""
    print(f"Starting weekly analysis: {datetime.now()}")
    result = run_weekly_analysis()

    # 결과를 Slack이나 이메일로 전송
    send_notification(result)
    print("Analysis completed and notification sent")

if __name__ == "__main__":
    scheduler.start()
```

## 디버깅 및 트러블슈팅

서브 에이전트 시스템을 운영하면서 발생할 수 있는 문제와 해결 방법을 소개합니다.

### 문제 1: 서브 에이전트가 호출되지 않음

증상:
```
메인 에이전트가 서브 에이전트를 호출하지 않고 직접 작업을 수행
```

원인:
- 서브 에이전트의 `description`이 불명확
- 메인 에이전트 프롬프트에 서브 에이전트 사용 지시 누락

해결책:

```python
# ❌ 나쁜 예
subagent = {
    "name": "research-agent",
    "description": "리서치",  # 너무 짧고 모호함
    # ...
}

# ✅ 좋은 예
subagent = {
    "name": "deep-research-agent",
    "description": """
    복잡한 주제에 대해 웹 검색을 통한 심층 리서치를 수행합니다.

    다음 경우에 이 에이전트를 사용하세요:
    - 다각도 정보 수집이 필요한 경우
    - 최소 5개 이상의 출처가 필요한 경우
    - 학술적이거나 기술적인 주제
    - 데이터 기반 분석이 필요한 경우

    사용하지 말아야 할 경우:
    - 간단한 팩트 체크
    - 이미 알려진 정보 확인
    """,
    # ...
}

# 메인 에이전트 프롬프트도 명확히
main_agent = create_deep_agent(
    subagents=[subagent],
    system_prompt="""
    ...

    서브 에이전트 사용:
    - 복잡한 리서치 작업은 반드시 deep-research-agent에게 위임하세요
    - task 도구를 사용하여 서브 에이전트를 호출하세요
    - 직접 검색하지 말고 서브 에이전트를 활용하세요
    """
)
```

### 문제 2: 서브 에이전트 결과가 파일에 저장되지 않음

증상:
```
서브 에이전트가 실행되지만 결과가 파일로 저장되지 않음
```

원인:
- 서브 에이전트 프롬프트에 파일 저장 지시 누락
- 메인 에이전트가 결과를 파일로 저장하지 않음

해결책:

```python
# 서브 에이전트 프롬프트
subagent = {
    "name": "research-agent",
    "prompt": """
    ...

    중요: 작업 완료 후 결과를 반환하기만 하세요.
    파일 저장은 메인 에이전트가 처리합니다.
    """,
    # ...
}

# 메인 에이전트 프롬프트
main_agent = create_deep_agent(
    subagents=[subagent],
    system_prompt="""
    ...

    작업 흐름:
    1. task 도구로 research-agent 호출
    2. 반환된 결과를 즉시 write_file로 저장
       예: write_file("research_result.md", 결과 내용)
    3. 다음 단계 진행
    """
)
```

### 문제 3: 병렬 실행이 순차 실행됨

증상:
```
여러 서브 에이전트를 호출했지만 순차적으로 실행됨
```

원인:
- 서브 에이전트 간 의존성이 있어 병렬 실행 불가
- 메인 에이전트가 한 번에 하나씩만 호출

해결책:

```python
# 메인 에이전트 프롬프트 수정
main_agent = create_deep_agent(
    subagents=[research1, research2, research3],
    system_prompt="""
    ...

    병렬 실행 방법:
    독립적인 작업들은 한 번에 모두 호출하세요.

    예시:
    1. task("research-agent-1", "주제 A 조사")
    2. task("research-agent-2", "주제 B 조사")
    3. task("research-agent-3", "주제 C 조사")

    위 3개를 하나의 응답에서 모두 호출하면 병렬 실행됩니다.
    각각을 별도 응답으로 호출하면 순차 실행됩니다.
    """
)
```

### 문제 4: 토큰 제한 초과

증상:
```
Error: Token limit exceeded
```

원인:
- 서브 에이전트가 너무 많은 정보 수집
- 파일 크기가 너무 큼

해결책:

```python
# 방법 1: 서브 에이전트 출력 제한
subagent = {
    "name": "research-agent",
    "prompt": """
    ...

    출력 제한:
    - 요약: 최대 200자
    - 주요 발견사항: 최대 5개 항목
    - 상세 분석: 각 주제당 최대 300자
    - 참고 자료: 최대 5개

    총 출력 길이: 2000자 이내로 제한
    """,
    # ...
}

# 방법 2: 압축 서브 에이전트 추가
compression_agent = {
    "name": "compression-agent",
    "description": "긴 리서치 결과를 압축합니다",
    "prompt": """
    주어진 리서치 결과를 다음 기준으로 압축하세요:
    - 중복 제거
    - 핵심 정보만 유지
    - 최대 1000자로 압축
    """,
    "tools": [],
    "model": "openai:gpt-4o"
}

# 방법 3: 더 큰 컨텍스트 모델 사용
main_agent = create_deep_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-20241022"),  # 200K 컨텍스트
    # ...
)
```

### 문제 5: 서브 에이전트 간 정보 공유 안 됨

증상:
```
서브 에이전트 A의 결과를 서브 에이전트 B가 참조하지 못함
```

해결책:

```python
# 파일시스템을 통한 정보 공유
main_agent = create_deep_agent(
    subagents=[agent_a, agent_b],
    system_prompt="""
    정보 공유 방법:

    1. 서브 에이전트 A 호출
    2. 결과를 파일로 저장 (write_file)
    3. 서브 에이전트 B의 입력에 파일 경로 포함
       예: "agent_a_result.md 파일을 읽고 분석하세요"
    4. 서브 에이전트 B는 read_file로 이전 결과 읽기

    예시:
    - task("agent-a", "주제 조사")
    - write_file("result_a.md", 결과)
    - task("agent-b", "result_a.md 파일의 내용을 분석하세요")
    """
)
```

## 베스트 프랙티스

서브 에이전트 시스템을 효과적으로 사용하기 위한 권장 사항입니다.

### 1. 서브 에이전트 설계 원칙

#### 단일 책임 원칙 (Single Responsibility)

각 서브 에이전트는 하나의 명확한 역할만 수행해야 합니다.

```python
# ❌ 나쁜 예: 너무 많은 역할
bad_subagent = {
    "name": "do-everything-agent",
    "description": "리서치, 분석, 보고서 작성, 데이터 시각화, 이메일 전송",
    # 너무 많은 책임!
}

# ✅ 좋은 예: 명확한 단일 역할
research_agent = {
    "name": "research-agent",
    "description": "웹에서 정보를 수집하고 요약합니다",
}

analysis_agent = {
    "name": "analysis-agent",
    "description": "리서치 결과를 분석하고 인사이트를 도출합니다",
}

report_agent = {
    "name": "report-agent",
    "description": "분석 결과를 바탕으로 보고서를 작성합니다",
}
```

#### 적절한 입도 (Granularity)

너무 세분화하지도, 너무 통합하지도 않은 적절한 크기로 설계합니다.

```python
# ❌ 너무 세분화
google_search_agent = {"name": "google-search", ...}
bing_search_agent = {"name": "bing-search", ...}
tavily_search_agent = {"name": "tavily-search", ...}

# ✅ 적절한 입도
web_search_agent = {
    "name": "web-search-agent",
    "description": "다양한 검색 엔진을 사용하여 웹 검색 수행",
    "tools": [google_search, bing_search, tavily_search],
}
```

### 2. 프롬프트 작성 가이드

#### 구조화된 프롬프트

```python
subagent = {
    "name": "research-agent",
    "prompt": """
    # 역할
    [명확한 역할 정의]

    # 작업 프로세스
    1. [단계 1]
    2. [단계 2]
    3. [단계 3]

    # 출력 형식
    [구체적인 형식 지정]

    # 품질 기준
    - [기준 1]
    - [기준 2]

    # 주의사항
    - [주의할 점 1]
    - [주의할 점 2]

    # 예시
    [구체적인 예시]
    """,
}
```

#### 명시적 지시

```python
# ❌ 모호한 지시
"정보를 조사하세요"

# ✅ 명확한 지시
"""
다음 절차에 따라 정보를 조사하세요:
1. 먼저 공식 문서 확인
2. 없으면 학술 논문 검색
3. 마지막으로 기술 블로그 참조
4. 각 출처의 신뢰도를 평가하고 명시
5. 최소 3개 이상의 독립적인 출처 확보
"""
```

### 3. 도구 선택 전략

#### 최소 권한 원칙

서브 에이전트에게 필요한 최소한의 도구만 제공합니다.

```python
# ❌ 너무 많은 도구
all_tools_agent = {
    "name": "agent",
    "tools": [
        internet_search, file_write, file_delete,
        database_write, api_call, send_email, ...
    ],  # 위험!
}

# ✅ 필요한 도구만
safe_research_agent = {
    "name": "research-agent",
    "tools": [internet_search],  # 검색만 가능
}

safe_analysis_agent = {
    "name": "analysis-agent",
    "tools": [],  # 도구 없음, 분석만
}
```

### 4. 에러 처리

#### 재시도 로직

```python
# 메인 에이전트 프롬프트
main_agent = create_deep_agent(
    system_prompt="""
    에러 처리:

    서브 에이전트 호출 실패 시:
    1. 에러 메시지 확인
    2. 1회에 한해 재시도
    3. 재시도 시 입력을 더 명확하게 수정
    4. 2회 실패 시 사용자에게 보고

    예시:
    - 첫 시도: task("agent", "복잡한 질문")
    - 실패 → 재시도: task("agent", "더 간단하고 구체적인 질문")
    - 실패 → 사용자에게 "해당 작업 수행 불가" 보고
    """
)
```

### 5. 성능 최적화

#### 캐싱 활용

```python
# Anthropic 모델 사용 시 프롬프트 캐싱
agent = create_deep_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-20241022"),
    subagents=[...],  # 서브 에이전트 프롬프트도 캐싱됨
    system_prompt="..."  # 캐싱되어 비용 절감
)
```

#### 병렬 실행 최대화

```python
# 메인 에이전트 프롬프트
main_agent = create_deep_agent(
    system_prompt="""
    병렬 실행 전략:

    독립적인 작업들을 식별하고 한 번에 모두 호출하세요.

    좋은 예:
    - task("agent-1", "주제 A")
    - task("agent-2", "주제 B")
    - task("agent-3", "주제 C")
    → 3개 모두 병렬 실행

    나쁜 예:
    - task("agent-1", "주제 A")
    - [결과 대기]
    - task("agent-2", "주제 B")
    - [결과 대기]
    → 순차 실행, 느림
    """
)
```

### 6. 모니터링 및 로깅

#### LangSmith 활용

```python
import os

# LangSmith 추적 활성화
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-key"
os.environ["LANGCHAIN_PROJECT"] = "my-project"

# 이제 모든 서브 에이전트 호출이 추적됨
agent = create_deep_agent(...)

# LangSmith에서 확인 가능:
# - 각 서브 에이전트 실행 시간
# - 토큰 사용량
# - 성공/실패율
# - 병목 구간
```

### 7. 테스트 및 검증

#### 단위 테스트

```python
# 각 서브 에이전트를 독립적으로 테스트
def test_research_agent():
    agent = create_deep_agent(
        subagents=[research_subagent],
        tools=[internet_search]
    )

    result = agent.invoke({
        "messages": [{
            "role": "user",
            "content": "test query"
        }]
    })

    assert "research" in result
    assert len(result["sources"]) >= 3
```

#### 통합 테스트

```python
# 전체 시스템 테스트
def test_full_system():
    main_agent = create_deep_agent(
        subagents=[research_agent, analysis_agent, synthesis_agent],
        tools=[...]
    )

    result = main_agent.invoke({
        "messages": [{
            "role": "user",
            "content": "comprehensive research task"
        }]
    })

    # 모든 파일이 생성되었는지 확인
    assert os.path.exists("research_result.md")
    assert os.path.exists("analysis_result.md")
    assert os.path.exists("final_report.md")
```

## 성능 최적화 및 모니터링

### LangSmith를 통한 모니터링

```python
import os

# LangSmith 설정
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-key"
os.environ["LANGCHAIN_PROJECT"] = "hybrid-research-system"

# 이제 모든 에이전트 실행이 LangSmith에 기록됩니다
# - 각 도구 호출 시간
# - 토큰 사용량
# - 비용 분석
# - 성능 병목 지점
```

### 캐싱을 통한 비용 절감

```python
from deepagents import create_deep_agent
from langchain_anthropic import ChatAnthropic

# Anthropic 모델 사용 시 프롬프트 캐싱 자동 활성화
agent = create_deep_agent(
    model=ChatAnthropic(model="claude-3-5-sonnet-20241022"),
    # AnthropicPromptCachingMiddleware가 자동으로 추가됨
    system_prompt="..."  # 이 프롬프트는 캐싱되어 비용 절감
)
```

### 병렬 처리 최적화

```python
# 서브 에이전트를 병렬로 실행하여 성능 향상
# DeepAgents는 자동으로 독립적인 작업을 병렬 처리합니다

# 예: 4개 경쟁사 분석을 동시에 실행
# task 도구가 각 경쟁사마다 별도 서브 에이전트를 생성
# 순차 실행: ~20분 → 병렬 실행: ~5분
```

### Human-in-the-Loop 설정 (선택사항)

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()

# 중요한 작업에 대해서만 승인 요청
agent = create_deep_agent(
    interrupt_on={
        "write_file": True,  # 파일 작성 시 승인 요청
        "execute": True,  # 명령 실행 시 승인 요청
    },
    checkpointer=checkpointer,
    system_prompt="..."
)

# 실행
config = {"configurable": {"thread_id": "weekly-analysis-1"}}
result = agent.invoke({"messages": [...]}, config=config)

# 중단점 처리
if "action_requests" in result:
    # 사용자에게 승인 요청
    # 승인 후 재개
    pass
```

## 마무리

### 주요 장점

통합 시스템의 강점:

1. DeepAgents의 장점 활용
   - 복잡한 워크플로우 관리
   - 파일시스템을 통한 효율적인 컨텍스트 관리
   - 서브 에이전트를 통한 병렬 처리

2. Open Deep Research의 장점 활용
   - 검증된 리서치 프로세스
   - 다양한 모델 선택 가능
   - 강력한 검색 통합

3. 시너지 효과
   - 각 시스템의 강점을 최대한 활용
   - 복잡한 리서치 작업의 자동화
   - 확장 가능한 아키텍처

### 적용 가능한 분야

- 기업 인텔리전스: 경쟁사 분석, 시장 조사, 트렌드 모니터링
- 학술 연구: 문헌 조사, 연구 동향 분석, 논문 리뷰
- 투자 분석: 산업 분석, 기업 실사, 시장 전망
- 제품 개발: 기술 조사, 사용자 피드백 분석, 경쟁 제품 분석

### 다음 단계

1. 환경 구축
   ```bash
   pip install deepagents tavily-python langchain-openai
   ```

2. 기본 예제 실행
   - 구현 예제 1부터 시작
   - 점진적으로 복잡도 증가

3. 커스터마이징
   - 자신의 도메인에 맞게 시스템 프롬프트 조정
   - 필요한 도구 추가
   - 서브 에이전트 특화

4. 프로덕션 배포
   - LangSmith 설정
   - 모니터링 및 알림 구성
   - 자동화 스케줄 설정

### 추가 리소스

DeepAgents:
- [GitHub 리포지토리](https://github.com/langchain-ai/deepagents){:target="_blank"}
- [공식 문서](https://docs.langchain.com/oss/python/deepagents/overview){:target="_blank"}
- [Quickstarts](https://github.com/langchain-ai/deepagents-quickstarts){:target="_blank"}

Open Deep Research:
- [GitHub 리포지토리](https://github.com/langchain-ai/open_deep_research){:target="_blank"}
- [LangGraph Platform 문서](https://docs.langchain.com/langgraph){:target="_blank"}

참고 자료:
- [DeepAgents 블로그 포스트](https://blog.langchain.com/deep-agents/){:target="_blank"}
- [DataCamp 튜토리얼](https://www.datacamp.com/tutorial/deep-agents){:target="_blank"}
- [MarkTechPost 가이드](https://www.marktechpost.com/2025/10/20/meet-langchains-deepagents-library-and-a-practical-example-to-see-how-deepagents-actually-work-in-action/){:target="_blank"}

두 프로젝트를 결합하여 더 강력하고 효율적인 AI 리서치 시스템을 구축해보세요!
