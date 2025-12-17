---
layout: post
title: LangChain Open Deep Research - AI 기반 심층 연구 자동화 도구
author: 'Juho'
date: 2025-12-17 09:00:00 +0900
categories: [LangChain, AI, Research]
tags: [LangChain, LangGraph, Deep Research, OpenAI, Anthropic, Tavily, MCP, Supabase]
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
1. [Open Deep Research란?](#open-deep-research란)
2. [주요 특징](#주요-특징)
3. [시스템 아키텍처](#시스템-아키텍처)
4. [설치 방법](#설치-방법)
5. [환경 설정](#환경-설정)
6. [Configuration 완전 가이드](#configuration-완전-가이드)
7. [실행 방법](#실행-방법)
8. [워크플로우 상세 설명](#워크플로우-상세-설명)
9. [실전 예제](#실전-예제)
10. [성능 평가](#성능-평가)
11. [성능 튜닝 가이드](#성능-튜닝-가이드)
12. [보안 및 인증](#보안-및-인증)
13. [배포 옵션](#배포-옵션)

## Open Deep Research란?

Open Deep Research는 LangChain에서 개발한 완전 오픈소스 AI 기반 심층 연구 자동화 도구입니다. 복잡한 연구 질문에 대해 자동으로 정보를 수집하고, 분석하며, 종합적인 리포트를 생성하는 AI 에이전트입니다.

이 도구는 Deep Research Bench 리더보드에서 6위를 기록하며 (RACE 점수 0.4344), 상용 대안들과 경쟁할 수 있는 성능을 입증했습니다.

### 왜 Open Deep Research를 사용해야 하나?

- 완전 오픈소스: MIT 라이선스로 자유롭게 사용 가능
- 멀티모델 지원: OpenAI, Anthropic, Google, Groq 등 다양한 LLM 프로바이더 지원
- 유연한 검색: Tavily, MCP 서버, Anthropic/OpenAI 네이티브 검색 지원
- 병렬 처리: 여러 연구 작업을 동시에 수행하여 효율성 극대화
- 토큰 최적화: 자동으로 토큰 제한을 관리하고 비용 효율적으로 운영

## 주요 특징

### 1. 멀티모델 아키텍처
시스템은 4가지 역할에 각각 다른 모델을 사용할 수 있습니다:
- 요약 (Summarization): 기본값 `gpt-4.1-mini`
- 연구 에이전트 (Research Agent): 기본값 `gpt-4.1`
- 압축 (Compression): 기본값 `gpt-4.1`
- 보고서 작성 (Report Writing): 기본값 `gpt-4.1`

모든 모델은 구조화된 출력(structured outputs)과 도구 호출(tool calling)을 지원해야 합니다.

### 2. 검색 통합
- Tavily Search: 기본 검색 엔진
- MCP (Model Context Protocol): 다양한 외부 도구 통합
- 네이티브 검색: Anthropic과 OpenAI의 웹 검색 API 지원

### 3. 지능형 워크플로우
```
사용자 질문 → 명확화 → 연구 계획 → 병렬 연구 → 압축 → 최종 보고서
```

## 시스템 아키텍처

Open Deep Research는 LangGraph를 기반으로 한 다단계 파이프라인으로 구성됩니다:

```
┌─────────────────┐
│  사용자 입력     │
└────────┬────────┘
         ▼
┌─────────────────┐
│  명확화 단계     │ ← 질문이 충분히 명확한지 확인
└────────┬────────┘
         ▼
┌─────────────────┐
│  연구 계획 작성  │ ← 구조화된 연구 브리프 생성
└────────┬────────┘
         ▼
┌─────────────────┐
│  슈퍼바이저      │ ← 연구 작업을 여러 주제로 분해
└────────┬────────┘
         ▼
┌─────────────────┐
│  병렬 연구 수행  │ ← 여러 연구자가 동시에 작업
└────────┬────────┘
         ▼
┌─────────────────┐
│  연구 결과 압축  │ ← 수집된 정보를 요약
└────────┬────────┘
         ▼
┌─────────────────┐
│  최종 보고서     │ ← 종합 리포트 생성
└─────────────────┘
```

### 핵심 컴포넌트

#### 1. State Management (state.py)
연구 워크플로우의 상태를 관리합니다:
- `AgentState`: 전체 워크플로우 상태
- `SupervisorState`: 슈퍼바이저 작업 관리
- `ResearcherState`: 개별 연구자 상태

#### 2. Deep Researcher (deep_researcher.py)
핵심 연구 로직을 담당하는 메인 모듈입니다:
- 명확화 노드: 사용자 질문 검증
- 연구 계획 노드: 구조화된 브리프 생성
- 슈퍼바이저 노드: 작업 분배 및 조율
- 연구자 노드: 실제 정보 수집
- 압축 노드: 결과 요약
- 보고서 생성 노드: 최종 문서 작성

#### 3. Prompts (prompts.py)
각 단계에서 사용되는 프롬프트 템플릿을 관리합니다.

#### 4. Utilities (utils.py)
검색, 데이터 처리 등의 헬퍼 함수를 제공합니다.

## 설치 방법

### 사전 요구사항
- Python 3.11 (권장, langgraph.json에서 명시)
  - pyproject.toml은 3.10 이상 지원하지만, 실행 시 3.11 사용
- Git
- uv (Python 패키지 관리자)

### 단계별 설치

#### 1. 저장소 클론
```bash
git clone https://github.com/langchain-ai/open_deep_research.git
cd open_deep_research
```

#### 2. 가상 환경 생성
```bash
uv venv
```

#### 3. 가상 환경 활성화
Linux/Mac:
```bash
source .venv/bin/activate
```

Windows:
```bash
.venv\Scripts\activate
```

#### 4. 의존성 설치
```bash
uv sync
```

또는

```bash
uv pip install -r pyproject.toml
```

### 주요 의존성

프로젝트는 다음과 같은 주요 라이브러리를 사용합니다:

AI/ML 프레임워크:
- `langgraph` (>=0.5.4): 워크플로우 오케스트레이션
- `langchain-*`: 다양한 LLM 프로바이더 통합
  - langchain-openai
  - langchain-anthropic
  - langchain-google-genai
  - langchain-aws
  - langchain-groq

검색 도구:
- `tavily-python`: Tavily 검색 API
- `duckduckgo-search`: DuckDuckGo 검색
- `exa-py`: Exa 검색
- `arxiv`: 학술 논문 검색

데이터 처리:
- `pymupdf`: PDF 처리
- `beautifulsoup4`: HTML 파싱
- `markdownify`: HTML을 마크다운으로 변환
- `pandas`: 데이터 분석

## 환경 설정

### .env 파일 설정

#### 1. 예제 파일 복사
```bash
cp .env.example .env
```

#### 2. 필수 환경 변수 설정

`.env` 파일을 열고 다음 항목을 설정합니다:

```bash
# 필수 API 키
OPENAI_API_KEY=your_openai_api_key_here
ANTHROPIC_API_KEY=your_anthropic_api_key_here
TAVILY_API_KEY=your_tavily_api_key_here

# LangSmith (모니터링 및 디버깅)
LANGSMITH_API_KEY=your_langsmith_api_key_here
LANGSMITH_PROJECT=open_deep_research
LANGSMITH_TRACING=true

# 선택 사항 (Google, AWS 등 추가 프로바이더)
GOOGLE_API_KEY=your_google_api_key_here

# Open Agent Platform 배포 시 필요
SUPABASE_KEY=your_supabase_key_here
SUPABASE_URL=your_supabase_url_here
GET_API_KEYS_FROM_CONFIG=false  # 로컬 개발: false, 프로덕션: true
```

### API 키 발급 방법

1. OpenAI API Key: [platform.openai.com](https://platform.openai.com)
2. Anthropic API Key: [console.anthropic.com](https://console.anthropic.com)
3. Tavily API Key: [tavily.com](https://tavily.com)
4. LangSmith API Key: [smith.langchain.com](https://smith.langchain.com)

## Configuration 완전 가이드

Open Deep Research는 다양한 설정 옵션을 제공하여 성능, 비용, 품질을 세밀하게 조정할 수 있습니다. `src/open_deep_research/configuration.py` 파일에서 모든 설정을 확인할 수 있습니다.

### 일반 설정

#### max_structured_output_retries
- 범위: 1-10
- 기본값: 3
- 설명: 구조화된 출력 호출 실패 시 재시도 횟수
- 권장: 안정성이 중요한 경우 5로 증가

```python
max_structured_output_retries = 3
```

#### allow_clarification
- 타입: boolean
- 기본값: true
- 설명: 연구 시작 전 질문 명확화 허용 여부
- 권장: 사용자 입력이 모호할 수 있는 경우 true 유지

```python
allow_clarification = True
```

#### max_concurrent_research_units
- 범위: 1-20
- 기본값: 5
- 설명: 동시에 실행할 연구 작업 수
- 권장:
  - 빠른 결과: 10-15
  - 비용 절감: 3-5
  - 토큰 제한 우려: 2-3

```python
max_concurrent_research_units = 5
```

### 연구 파라미터

#### max_researcher_iterations
- 범위: 1-10
- 기본값: 6
- 설명: 연구 슈퍼바이저가 연구를 반복하는 횟수
- 권장:
  - 간단한 주제: 3-4
  - 복잡한 주제: 6-8
  - 매우 심층적인 연구: 8-10

```python
max_researcher_iterations = 6
```

#### max_react_tool_calls
- 범위: 1-30
- 기본값: 10
- 설명: 각 연구자가 도구를 호출할 수 있는 최대 횟수
- 권장:
  - 빠른 스캔: 5-7
  - 표준 연구: 10-15
  - 심층 분석: 20-25

```python
max_react_tool_calls = 10
```

### 검색 설정

#### search_api
- 옵션:
  - `"tavily"` (기본값)
  - `"openai_native"`
  - `"anthropic_native"`
  - `"none"`
- 설명: 사용할 검색 API 선택
- 비교:

| 검색 API | 장점 | 단점 |
|---------|------|------|
| Tavily | 높은 정확도, 구조화된 결과 | 별도 API 키 필요 |
| OpenAI Native | OpenAI 통합, 간편 | 제한적인 기능 |
| Anthropic Native | Claude 최적화 | 제한적인 기능 |
| None | MCP만 사용 | 검색 불가 |

```python
search_api = "tavily"
```

### 모델 설정

각 역할별로 독립적인 모델과 토큰 제한을 설정할 수 있습니다:

#### Summarization Model
- 기본값: `openai:gpt-4.1-mini`
- Max Tokens: 8,192
- 용도: 웹 콘텐츠 요약
- 권장: 비용 효율적인 모델 사용

```python
summarization_model = "openai:gpt-4.1-mini"
summarization_max_tokens = 8192
```

#### Research Model
- 기본값: `openai:gpt-4.1`
- Max Tokens: 10,000
- 용도: 연구 작업 수행
- 권장: 고성능 모델 사용 (Claude Sonnet 4, GPT-4.1)

```python
research_model = "openai:gpt-4.1"
research_max_tokens = 10000
```

#### Compression Model
- 기본값: `openai:gpt-4.1`
- Max Tokens: 8,192
- 용도: 연구 결과 압축
- 권장: 중간 성능 모델

```python
compression_model = "openai:gpt-4.1"
compression_max_tokens = 8192
```

#### Final Report Model
- 기본값: `openai:gpt-4.1`
- Max Tokens: 10,000
- 용도: 최종 보고서 작성
- 권장: 최고 품질 모델 사용 (Claude Opus 4, GPT-5)

```python
final_report_model = "openai:gpt-4.1"
final_report_max_tokens = 10000
```

### 콘텐츠 처리

#### max_content_length
- 범위: 1,000 - 200,000 characters
- 기본값: 50,000
- 설명: 웹페이지 콘텐츠가 이 길이를 초과하면 요약 진행
- 권장:
  - 빠른 처리: 30,000-40,000
  - 표준: 50,000
  - 상세 분석: 100,000-150,000

```python
max_content_length = 50000
```

### MCP 설정

#### mcp_config
MCP(Model Context Protocol) 서버 설정:

```python
mcp_config = {
    "server_url": "http://localhost:8080",
    "tools": ["web_browser", "file_reader", "calculator"],
    "auth": {
        "type": "bearer",
        "token": "your_token_here"
    }
}
```

#### mcp_prompt
MCP 도구 사용을 위한 커스텀 프롬프트:

```python
mcp_prompt = """
When using MCP tools, follow these guidelines:
1. Use web_browser for recent information
2. Use file_reader for local documents
3. Use calculator for numerical computations
"""
```

### 설정 예제

#### 비용 최적화 설정
```python
# 최소 비용으로 운영
max_concurrent_research_units = 2
max_researcher_iterations = 4
max_react_tool_calls = 7
summarization_model = "openai:gpt-4.1-mini"
research_model = "openai:gpt-4.1-mini"
compression_model = "openai:gpt-4.1-mini"
final_report_model = "openai:gpt-4.1"
```

#### 최고 품질 설정
```python
# 최고 품질 연구
max_concurrent_research_units = 10
max_researcher_iterations = 8
max_react_tool_calls = 20
max_content_length = 100000
summarization_model = "openai:gpt-4.1"
research_model = "anthropic:claude-sonnet-4"
compression_model = "anthropic:claude-sonnet-4"
final_report_model = "anthropic:claude-opus-4"
```

#### 균형 잡힌 설정 (권장)
```python
# 성능과 비용의 균형
max_concurrent_research_units = 5
max_researcher_iterations = 6
max_react_tool_calls = 10
max_content_length = 50000
summarization_model = "openai:gpt-4.1-mini"
research_model = "openai:gpt-4.1"
compression_model = "openai:gpt-4.1"
final_report_model = "anthropic:claude-sonnet-4"
```

## 실행 방법

### LangGraph Studio 실행

```bash
uvx --refresh --from "langgraph-cli[inmem]" --with-editable . \
  --python 3.11 langgraph dev --allow-blocking
```

이 명령어는 다음을 수행합니다:
- LangGraph CLI를 최신 버전으로 업데이트
- 인메모리 체크포인터 포함
- Python 3.11 사용
- 개발 모드로 서버 실행
- 블로킹 호출 허용

### 접속 방법

서버 실행 후 다음 방법으로 접속할 수 있습니다:

#### 1. LangGraph Studio UI (권장)
```
https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

LangSmith 계정으로 로그인하면 브라우저에서 시각적 인터페이스를 통해 연구 에이전트를 사용할 수 있습니다.

#### 2. API 엔드포인트
```
http://127.0.0.1:2024
```

REST API를 통해 직접 호출할 수도 있습니다.

### 사용 예시

#### LangGraph Studio에서 사용

1. Studio UI에 접속
2. "Manage Assistants" 탭에서 설정 조정
3. `messages` 필드에 연구 질문 입력:
   ```
   "양자 컴퓨팅이 암호화 기술에 미치는 영향에 대해 연구해줘"
   ```
4. Submit 버튼 클릭
5. 워크플로우가 자동으로 실행되며 각 단계별 진행 상황 확인 가능

#### Python 코드로 호출

```python
from langgraph_sdk import get_client

client = get_client(url="http://127.0.0.1:2024")

# 연구 시작
thread = client.threads.create()
run = client.runs.create(
    thread["thread_id"],
    "open_deep_research",
    input={
        "messages": [{
            "role": "user",
            "content": "인공지능의 윤리적 문제에 대해 심층 연구해줘"
        }]
    }
)

# 결과 대기 및 확인
result = client.runs.join(thread["thread_id"], run["run_id"])
print(result["final_report"])
```

## 워크플로우 상세 설명

### 1. 명확화 단계 (Clarification)

목적: 사용자의 연구 질문이 충분히 명확한지 확인

동작 방식:
- `allow_clarification` 옵션이 활성화된 경우 실행
- 구조화된 출력을 사용하여 질문 분석
- 모호한 경우 추가 질문 생성
- 명확한 경우 다음 단계로 진행

예시:
- 질문: "AI에 대해 알려줘"
- 명확화: "어떤 측면의 AI에 대해 알고 싶으신가요? (기술, 윤리, 산업 적용 등)"

### 2. 연구 계획 작성 (Research Brief)

목적: 비구조화된 사용자 입력을 구조화된 연구 브리프로 변환

생성 내용:
- 명확한 연구 주제
- 연구 범위 정의
- 핵심 질문 추출
- `ResearchQuestion` 객체로 구조화

### 3. 슈퍼바이저 (Supervisor)

목적: 연구 작업을 여러 하위 작업으로 분해하고 조율

가용 도구:
- `think_tool`: 전략적 사고 및 계획
- `ConductResearch`: 연구자에게 작업 위임
- `ResearchComplete`: 연구 완료 신호

작업 관리:
- `max_concurrent_research_units` 설정으로 동시 실행 제한
- 병렬 처리로 효율성 극대화
- 각 연구자의 진행 상황 모니터링

예시:
```
슈퍼바이저: "AI 윤리" 연구를 다음 3개 작업으로 분해
→ 연구자 1: "AI 편향성 및 공정성"
→ 연구자 2: "프라이버시 및 데이터 보호"
→ 연구자 3: "AI 규제 및 법적 프레임워크"
```

### 4. 연구자 (Researcher)

목적: 특정 주제에 대한 심층 정보 수집

각 연구자의 작업:
- 할당된 연구 주제 수신
- 검색 도구 사용 (Tavily, MCP 등)
- 최대 `max_react_tool_calls` 반복 수행
- 발견 내용을 구조화된 요약으로 압축

반복 프로세스:
1. 검색 쿼리 생성
2. 정보 수집
3. 결과 분석
4. 추가 질문 생성 (필요시)
5. 반복 또는 종료

사용 도구:
- 웹 검색 (Tavily, DuckDuckGo)
- 학술 검색 (arXiv)
- MCP 서버 도구
- think_tool (전략적 사고)

### 5. 연구 결과 압축 (Compression)

목적: 원시 연구 결과를 간결하고 구조화된 요약으로 변환

처리 방식:
- 모든 연구자의 결과 수집
- 중복 정보 제거
- 핵심 발견 사항 추출
- 토큰 제한 관리
- 재시도 시 점진적 잘라내기

토큰 관리:
```python
# 토큰 초과 시 자동으로 내용 잘라내기
try:
    compressed = compress_notes(notes)
except TokenLimitError:
    # 재시도 시 내용 70%만 사용
    compressed = compress_notes(notes[:int(len(notes) * 0.7)])
```

### 6. 최종 보고서 생성 (Report Generation)

목적: 종합적인 연구 보고서 작성

보고서 구성:
- 서론 (연구 질문 및 범위)
- 주요 발견 사항
- 상세 분석
- 결론 및 시사점
- 참고 자료

특징:
- 마크다운 형식 지원
- 토큰 제한 자동 관리
- 재시도 메커니즘
- 구조화된 형식

## 실전 예제

Open Deep Research 저장소는 `examples/` 디렉토리에 4개의 실전 사용 예제를 제공합니다. 각 예제는 특정 도메인에서 도구를 효과적으로 사용하는 방법을 보여줍니다.

### 1. arXiv 학술 논문 연구 (arxiv.md)

파일 크기: 11.6 KB
도메인: 물리학, 컴퓨터 과학, 수학 등 학술 연구

사용 사례: "비만과 건강 격차에 대한 최신 연구 동향"

핵심 발견:
- 미국 성인의 1/3 이상이 비만
- 건축 환경 특성이 도시 간 비만 변동의 72-90% 설명
- 머신러닝을 통한 위성 이미지 분석으로 88% 정확도 달성

사용된 검색 소스:
- arXiv 논문 검색 (2208.05335, 1711.00885, 2310.07563 등)
- 학술 데이터베이스 쿼리
- 연구 논문 교차 참조

설정 권장:
```python
search_api = "tavily"
max_researcher_iterations = 7  # 학술 연구는 더 많은 반복 필요
max_react_tool_calls = 15
# arXiv 검색 도구 활성화
```

활용 팁:
- 특정 arXiv ID를 질문에 포함하여 정확한 논문 참조
- 연구 분야를 명확히 지정 (예: cs.AI, physics.optics)
- 최신 논문 위주로 검색하려면 연도 명시

### 2. PubMed 의학 연구 (pubmed.md)

파일 크기: 13.6 KB
도메인: 의학, 생명과학, 임상 연구

사용 사례: 특정 질병이나 치료법에 대한 임상 연구 분석

특징:
- PubMed 데이터베이스 전문 검색
- 임상 시험 결과 분석
- 의학 용어 및 MeSH 태그 활용

설정 권장:
```python
max_content_length = 100000  # 의학 논문은 길이가 긴 경우 많음
max_researcher_iterations = 8
research_model = "anthropic:claude-sonnet-4"  # 의학 용어 이해도 높음
```

활용 팁:
- PMID (PubMed ID) 직접 참조 가능
- 임상 시험 단계 명시 (Phase I, II, III)
- 특정 질병의 ICD 코드 사용

### 3. AI 추론 시장 분석 (inference-market.md)

파일 크기: 10.0 KB
도메인: AI 산업, 시장 분석, 비즈니스 인텔리전스

사용 사례: AI 추론 시장의 현황과 트렌드 분석

핵심 내용:
- 시장 규모 및 성장률
- 주요 플레이어 분석
- 기술 트렌드 파악
- 가격 전략 비교

설정 권장:
```python
search_api = "tavily"
max_concurrent_research_units = 8  # 여러 회사 동시 분석
max_researcher_iterations = 6
```

활용 팁:
- 최신 뉴스와 블로그 포스트 활용
- 회사 공식 발표 자료 참조
- 재무 데이터 통합 분석

### 4. GPT-4.5 고급 분석 (inference-market-gpt45.md)

파일 크기: 11.9 KB
도메인: 고급 AI 분석

특징:
- GPT-4.5 모델 사용
- 더 깊이 있는 분석
- 복잡한 추론 작업

설정 예시:
```python
research_model = "openai:gpt-4.5"
final_report_model = "openai:gpt-4.5"
max_researcher_iterations = 10
max_react_tool_calls = 25
max_content_length = 150000
```

### 예제 실행 방법

#### 1. 예제 내용 확인
```bash
cat examples/arxiv.md
```

#### 2. LangGraph Studio에서 실행
1. Studio UI 접속
2. 예제의 연구 질문 복사
3. Configuration에서 추천 설정 적용
4. 실행 및 결과 비교

#### 3. Python 스크립트로 실행
```python
from langgraph_sdk import get_client

client = get_client(url="http://127.0.0.1:2024")

# arXiv 예제 실행
thread = client.threads.create()
run = client.runs.create(
    thread["thread_id"],
    "open_deep_research",
    input={
        "messages": [{
            "role": "user",
            "content": "양자 컴퓨팅 분야의 최근 5년간 주요 돌파구에 대해 arXiv 논문을 중심으로 연구해줘"
        }]
    },
    config={
        "configurable": {
            "search_api": "tavily",
            "max_researcher_iterations": 7
        }
    }
)
```

### 나만의 예제 만들기

저장소의 예제를 참고하여 특정 도메인에 최적화된 설정을 만들 수 있습니다:

```python
# examples/my_domain.py
DOMAIN_CONFIG = {
    # 금융 분석 예제
    "finance": {
        "search_api": "tavily",
        "max_researcher_iterations": 7,
        "max_react_tool_calls": 15,
        "research_model": "anthropic:claude-sonnet-4",
        "sources": ["bloomberg", "reuters", "sec_filings"]
    },

    # 법률 연구 예제
    "legal": {
        "search_api": "tavily",
        "max_researcher_iterations": 10,
        "max_content_length": 150000,
        "research_model": "openai:gpt-4.1",
        "sources": ["legal_databases", "case_law", "statutes"]
    },

    # 기술 문서 분석 예제
    "tech_docs": {
        "search_api": "none",  # MCP만 사용
        "mcp_config": {
            "tools": ["file_reader", "code_analyzer"]
        },
        "max_react_tool_calls": 20
    }
}
```

## 성능 평가

Open Deep Research는 Deep Research Bench에서 평가되었습니다. 이 벤치마크는 22개 분야에 걸친 100개의 박사급 연구 작업으로 구성됩니다.

### RACE 점수 비교

| 모델 구성 | RACE 점수 | 총 토큰 수 | 비용 효율성 |
|----------|-----------|-----------|-----------|
| GPT-5 | 0.4943 | 204,640,896 | 낮음 |
| Open Deep Research (기본) | 0.4309 | 58,015,332 | 높음 |
| Claude Sonnet 4 | 0.4401 | 138,917,050 | 중간 |

### 주요 발견

1. 비용 대비 성능:
   - 기본 설정(GPT-4.1)으로 GPT-5 대비 1/3 토큰만 사용하면서 87% 성능 달성
   - 토큰 기반 비용 모델에서 가장 경제적

2. 유연성:
   - Claude Sonnet 4로 변경 시 성능 향상 가능
   - 요약에는 저비용 모델(gpt-4.1-mini), 핵심 작업에는 고성능 모델 사용

3. 리더보드 순위:
   - Deep Research Bench에서 #6 순위
   - 오픈소스 솔루션 중 최상위권

## 성능 튜닝 가이드

Open Deep Research의 성능을 최적화하려면 워크로드 특성에 맞게 설정을 조정해야 합니다. 다음은 실전에서 검증된 튜닝 전략입니다.

### 시나리오별 최적 설정

#### 1. 빠른 탐색 (Quick Exploration)

목적: 빠르게 개요 파악, 5분 이내 결과

```python
# 설정
max_concurrent_research_units = 3
max_researcher_iterations = 3
max_react_tool_calls = 5
max_content_length = 30000

# 모델
summarization_model = "openai:gpt-4.1-mini"
research_model = "openai:gpt-4.1-mini"
compression_model = "openai:gpt-4.1-mini"
final_report_model = "openai:gpt-4.1"
```

예상 비용: $0.50 - $2.00
예상 시간: 3-7분
적합한 사용 사례: 주제 개요, 초기 조사, 빠른 팩트 체크

#### 2. 표준 연구 (Standard Research)

목적: 균형 잡힌 품질과 속도, 15분 내외 결과

```python
# 설정 (기본값 사용)
max_concurrent_research_units = 5
max_researcher_iterations = 6
max_react_tool_calls = 10
max_content_length = 50000

# 모델
summarization_model = "openai:gpt-4.1-mini"
research_model = "openai:gpt-4.1"
compression_model = "openai:gpt-4.1"
final_report_model = "openai:gpt-4.1"
```

예상 비용: $3.00 - $8.00
예상 시간: 10-20분
적합한 사용 사례: 일반적인 연구 작업, 보고서 작성, 학술 조사

#### 3. 심층 분석 (Deep Analysis)

목적: 최고 품질의 종합적 연구, 30분+ 소요

```python
# 설정
max_concurrent_research_units = 8
max_researcher_iterations = 10
max_react_tool_calls = 20
max_content_length = 100000
max_structured_output_retries = 5

# 모델
summarization_model = "openai:gpt-4.1"
research_model = "anthropic:claude-sonnet-4"
compression_model = "anthropic:claude-sonnet-4"
final_report_model = "anthropic:claude-opus-4"
```

예상 비용: $15.00 - $40.00
예상 시간: 30-60분
적합한 사용 사례: 학술 논문, 전략 보고서, 복잡한 기술 분석

#### 4. 비용 최적화 (Budget Mode)

목적: 최소 비용으로 운영, 품질은 타협

```python
# 설정
max_concurrent_research_units = 2
max_researcher_iterations = 4
max_react_tool_calls = 7
max_content_length = 30000
max_structured_output_retries = 2

# 모델 - 모두 mini 사용
summarization_model = "openai:gpt-4.1-mini"
research_model = "openai:gpt-4.1-mini"
compression_model = "openai:gpt-4.1-mini"
final_report_model = "openai:gpt-4.1-mini"
```

예상 비용: $0.30 - $1.00
예상 시간: 8-15분
적합한 사용 사례: 대량 처리, 간단한 요약, 예산 제한 시

### 병목 구간 식별 및 해결

#### 1. 토큰 제한 초과

증상:
```
Error: Token limit exceeded in compression step
```

원인: 연구자들이 수집한 정보량이 압축 모델의 컨텍스트 윈도우 초과

해결책:
```python
# 방법 1: 콘텐츠 길이 제한
max_content_length = 30000  # 50000에서 감소

# 방법 2: 동시 연구 단위 감소
max_concurrent_research_units = 3  # 5에서 감소

# 방법 3: 더 큰 컨텍스트 모델 사용
compression_model = "anthropic:claude-sonnet-4"  # 200K 컨텍스트
```

#### 2. 느린 실행 속도

증상: 30분 이상 소요

원인: 과도한 반복 또는 느린 검색 API

해결책:
```python
# 방법 1: 반복 횟수 감소
max_researcher_iterations = 4  # 6에서 감소
max_react_tool_calls = 7  # 10에서 감소

# 방법 2: 병렬 처리 증가
max_concurrent_research_units = 8  # 5에서 증가

# 방법 3: 더 빠른 모델 사용
research_model = "openai:gpt-4.1-mini"
```

#### 3. 낮은 결과 품질

증상: 표면적인 분석, 중요 정보 누락

원인: 불충분한 반복 또는 저성능 모델

해결책:
```python
# 방법 1: 반복 횟수 증가
max_researcher_iterations = 8  # 6에서 증가
max_react_tool_calls = 15  # 10에서 증가

# 방법 2: 고성능 모델 사용
research_model = "anthropic:claude-sonnet-4"
final_report_model = "anthropic:claude-opus-4"

# 방법 3: 콘텐츠 길이 증가
max_content_length = 80000  # 50000에서 증가
```

#### 4. 높은 비용

증상: 예상보다 높은 API 비용

원인: 과도한 토큰 사용

해결책:
```python
# 방법 1: 저비용 모델 사용
summarization_model = "openai:gpt-4.1-mini"
research_model = "openai:gpt-4.1-mini"
compression_model = "openai:gpt-4.1-mini"
# 보고서만 고품질 모델
final_report_model = "openai:gpt-4.1"

# 방법 2: 반복 제한
max_researcher_iterations = 4
max_react_tool_calls = 7

# 방법 3: 동시 연구 감소
max_concurrent_research_units = 3
```

### 모델별 성능-비용 트레이드오프

| 모델 | 속도 | 품질 | 비용 (상대적) | 권장 용도 |
|------|------|------|--------------|----------|
| gpt-4.1-mini | ⚡⚡⚡ | ⭐⭐⭐ | $ | 요약, 비용 절감 |
| gpt-4.1 | ⚡⚡ | ⭐⭐⭐⭐ | $$ | 표준 연구 |
| claude-sonnet-4 | ⚡⚡ | ⭐⭐⭐⭐⭐ | $$$ | 심층 분석 |
| claude-opus-4 | ⚡ | ⭐⭐⭐⭐⭐ | $$$$ | 최종 보고서 |
| gpt-4.5 | ⚡⚡ | ⭐⭐⭐⭐⭐ | $$$$ | 복잡한 추론 |

### LangSmith를 활용한 성능 모니터링

#### 1. 추적 활성화
```bash
LANGSMITH_TRACING=true langgraph dev
```

#### 2. 주요 메트릭 확인

LangSmith 대시보드에서 확인할 항목:
- 총 토큰 사용량 (입력/출력 분리)
- 각 노드별 실행 시간
- 도구 호출 횟수 및 패턴
- 에러 발생 지점

#### 3. 최적화 포인트 발견

```python
# LangSmith에서 확인한 데이터 기반 최적화
# 예: compression 노드가 전체 시간의 40% 소요

# Before
compression_model = "anthropic:claude-opus-4"

# After - 더 빠른 모델로 변경
compression_model = "anthropic:claude-sonnet-4"
# 결과: 30% 시간 단축, 품질 차이 미미
```

### 실전 튜닝 체크리스트

연구 작업을 시작하기 전에 다음을 확인하세요:

- [ ] 연구 주제의 복잡도 평가 (간단/보통/복잡)
- [ ] 예산 및 시간 제약 확인
- [ ] 필요한 품질 수준 결정
- [ ] 위 기준에 맞는 프리셋 선택
- [ ] LangSmith 추적 활성화
- [ ] 첫 실행 후 메트릭 검토
- [ ] 병목 구간 식별 및 조정
- [ ] 최종 설정으로 본격 실행

### A/B 테스트 예제

동일한 질문에 대해 다른 설정으로 테스트:

```python
# 테스트 A: 빠르고 저렴
config_a = {
    "max_researcher_iterations": 3,
    "research_model": "openai:gpt-4.1-mini"
}

# 테스트 B: 느리지만 고품질
config_b = {
    "max_researcher_iterations": 8,
    "research_model": "anthropic:claude-sonnet-4"
}

# 비교 지표
# - 비용: A가 70% 저렴
# - 시간: A가 50% 빠름
# - 품질: B가 15% 높음
# → 결정: 대부분의 경우 A, 중요한 연구는 B
```

## 보안 및 인증

Open Deep Research는 프로덕션 환경에서 안전하게 운영하기 위한 보안 기능을 제공합니다. `src/security/auth.py` 모듈이 인증 및 권한 관리를 담당합니다.

### 인증 시스템 개요

#### JWT 토큰 기반 인증

시스템은 JWT (JSON Web Token) 토큰을 사용하여 모든 API 요청을 검증합니다:

```python
# src/security/auth.py 핵심 로직
@auth.authenticate
async def handle_request(request):
    # 모든 요청에 대해 자동으로 토큰 검증
    token = request.headers.get("Authorization")
    user = await verify_token(token)
    return user
```

#### Supabase 통합

인증은 Supabase를 통해 관리됩니다:

```python
# Supabase를 통한 사용자 검증
supabase.auth.get_user(token)
```

환경 변수 설정:
```bash
SUPABASE_KEY=your_supabase_key_here
SUPABASE_URL=https://your-project.supabase.co
GET_API_KEYS_FROM_CONFIG=true  # 프로덕션에서는 true
```

### 인증 흐름

```
1. 클라이언트 요청
   ↓
2. Authorization 헤더 확인
   - "Bearer <token>" 형식
   ↓
3. Supabase 토큰 검증
   - 유효성 확인
   - 만료 시간 체크
   ↓
4. 사용자 정보 추출
   ↓
5. 요청 처리 (또는 401 에러)
```

### 권한 관리 (Authorization)

#### 1. 스레드(Thread) 격리

생성 시:
```python
# 스레드 생성 시 자동으로 소유자 정보 포함
thread = {
    "thread_id": "...",
    "metadata": {
        "creator_id": user.id,
        "creator_email": user.email
    }
}
```

접근 제어:
- 읽기: 자신이 만든 스레드만 조회 가능
- 수정: 자신이 만든 스레드만 수정 가능
- 삭제: 자신이 만든 스레드만 삭제 가능
- 검색: 자신의 스레드만 검색 결과에 포함

#### 2. 어시스턴트(Assistant) 격리

```python
# 어시스턴트에 소유자 태그 부여
assistant = {
    "assistant_id": "...",
    "owner_id": user.id,
    "config": {...}
}
```

접근 제어:
- 사용자는 자신의 어시스턴트만 접근 가능
- 다른 사용자의 어시스턴트는 목록에도 표시되지 않음

#### 3. 데이터 스토어 격리

```python
# 네임스페이스로 데이터 격리
namespace = ("user", user.id)

# 사용자 ID와 불일치하면 접근 거부
if namespace[1] != user.id:
    raise HTTPException(401, "Unauthorized")
```

### Studio 사용자 특권

개발/디버깅 모드에서는 모든 제한이 우회됩니다:

```python
if is_studio_user(user):
    # 모든 스레드 접근 가능
    # 모든 어시스턴트 접근 가능
    # 관리자 권한
    bypass_authorization = True
```

### API 요청 예제

#### 인증된 요청

```python
import requests

# 토큰 발급 (Supabase)
token = supabase.auth.sign_in({
    "email": "user@example.com",
    "password": "password"
})["access_token"]

# 인증 헤더와 함께 요청
response = requests.post(
    "http://localhost:2024/threads",
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    },
    json={
        "messages": [{"role": "user", "content": "연구 질문"}]
    }
)
```

#### 인증 실패 처리

```python
# 401 Unauthorized 에러 처리
try:
    response = client.threads.create()
except HTTPException as e:
    if e.status_code == 401:
        # 토큰 재발급 또는 로그인 유도
        print("인증 필요: 로그인하세요")
```

### 보안 모범 사례

#### 1. 토큰 관리

```python
# ✅ 좋은 예: 환경 변수에 저장
import os
token = os.getenv("SUPABASE_TOKEN")

# ❌ 나쁜 예: 코드에 하드코딩
token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

#### 2. 토큰 갱신

```python
# 토큰 만료 전 자동 갱신
from datetime import datetime, timedelta

def refresh_token_if_needed(token):
    expiry = jwt.decode(token, verify=False)["exp"]
    if datetime.fromtimestamp(expiry) < datetime.now() + timedelta(minutes=5):
        # 만료 5분 전에 갱신
        return supabase.auth.refresh_session()
    return token
```

#### 3. HTTPS 사용

```bash
# 프로덕션에서는 반드시 HTTPS
# ✅ 좋은 예
https://api.yourservice.com

# ❌ 나쁜 예
http://api.yourservice.com
```

#### 4. 민감 정보 로깅 방지

```python
# ✅ 좋은 예: 토큰 마스킹
logger.info(f"Token: {token[:10]}...")

# ❌ 나쁜 예: 전체 토큰 로깅
logger.info(f"Token: {token}")
```

### 로컬 개발 vs 프로덕션

#### 로컬 개발 (인증 비활성화)

```bash
# .env 파일
GET_API_KEYS_FROM_CONFIG=false

# 인증 없이 사용 가능
langgraph dev
```

#### 프로덕션 (인증 필수)

```bash
# .env 파일
GET_API_KEYS_FROM_CONFIG=true
SUPABASE_KEY=your_production_key
SUPABASE_URL=https://your-project.supabase.co

# 모든 요청에 인증 필요
langgraph deploy
```

### 멀티 테넌트 아키텍처

Open Deep Research는 멀티 테넌트 환경을 지원합니다:

```
조직 A
├── 사용자 1
│   ├── 스레드 1, 2, 3
│   └── 어시스턴트 A, B
└── 사용자 2
    ├── 스레드 4, 5
    └── 어시스턴트 C

조직 B
└── 사용자 3
    ├── 스레드 6
    └── 어시스턴트 D
```

각 사용자는 자신의 데이터만 접근 가능하며, 완전히 격리됩니다.

### 감사 로그 (Audit Logging)

LangSmith와 통합하여 모든 활동을 추적할 수 있습니다:

```python
# 자동으로 기록되는 정보
{
    "timestamp": "2025-12-15T09:00:00Z",
    "user_id": "user_123",
    "action": "create_thread",
    "resource_id": "thread_456",
    "ip_address": "192.168.1.1",
    "status": "success"
}
```

## 배포 옵션

### 1. 로컬 개발 (LangGraph Studio)

용도: 개발 및 테스트

장점:
- 즉시 시작 가능
- 시각적 디버깅
- 무료

실행:
```bash
langgraph dev --allow-blocking
```

### 2. 프로덕션 배포 (LangGraph Platform)

용도: 프로덕션 환경

장점:
- 확장 가능한 인프라
- 자동 로드 밸런싱
- 모니터링 및 로깅

배포:
```bash
langgraph deploy
```

### 3. Open Agent Platform (OAP)

용도: 사용자 친화적 웹 인터페이스

장점:
- 별도 설치 불필요
- 공개 데모 사용 가능
- 커스텀 인스턴스 배포 가능

접속:
- 공개 데모: [oap.langchain.com](https://oap.langchain.com)
- 자체 배포: OAP 저장소 참고

## 고급 설정

### 모델 커스터마이징

각 역할에 다른 모델을 사용할 수 있습니다:

```python
# configuration.py 수정
SUMMARIZATION_MODEL = "openai:gpt-4.1-mini"  # 비용 절감
RESEARCH_MODEL = "anthropic:claude-sonnet-4"  # 고성능
COMPRESSION_MODEL = "openai:gpt-4.1"
REPORT_MODEL = "anthropic:claude-opus-4"  # 최고 품질
```

### 동시 실행 제어

```python
# 동시에 실행할 연구 작업 수 제한
max_concurrent_research_units = 3

# 각 연구자의 최대 도구 호출 횟수
max_react_tool_calls = 10
```

### 검색 도구 선택

```python
# MCP 서버 사용
use_mcp = True

# Tavily vs DuckDuckGo
search_provider = "tavily"  # or "duckduckgo"
```

## 실전 사용 팁

### 1. 효과적인 질문 작성

좋은 예:
```
"양자 컴퓨팅이 현대 암호화 알고리즘(RSA, AES)에 미치는 영향을 분석하고,
포스트 양자 암호화 대안을 평가해줘"
```

나쁜 예:
```
"양자 컴퓨터에 대해 알려줘"
```

### 2. 명확화 옵션 활용

모호한 질문의 경우 명확화 기능을 활성화하세요:
```python
allow_clarification = True
```

### 3. 비용 최적화

- 요약 및 압축에는 저비용 모델 사용
- 핵심 연구 및 보고서에만 고성능 모델 사용
- `max_concurrent_research_units` 조정으로 토큰 사용량 제어

### 4. LangSmith로 모니터링

LangSmith를 활용하여:
- 각 단계별 토큰 사용량 추적
- 병목 구간 식별
- 프롬프트 최적화

## 레거시 구현

프로젝트는 두 가지 이전 구현을 `legacy` 폴더에 포함합니다:

### 1. Plan-and-Execute Workflow
- 정확성 강조
- 반복적 개선
- 순차적 처리

### 2. Multi-Agent Supervisor-Researcher
- 동시 처리 최적화
- 슈퍼바이저 기반 조율
- 병렬 실행

현재 메인 구현이 이 두 접근법의 장점을 결합했습니다.

## 문제 해결

### 일반적인 오류

#### 1. API 키 오류
```
Error: Invalid API key
```
해결: `.env` 파일에 올바른 API 키 설정 확인

#### 2. 토큰 제한 초과
```
Error: Token limit exceeded
```
해결: 모델 설정에서 더 큰 컨텍스트 윈도우 모델 사용

#### 3. 포트 충돌
```
Error: Port 2024 already in use
```
해결:
```bash
langgraph dev --port 2025
```

### 디버깅 모드

```bash
# 상세 로깅 활성화
LANGSMITH_TRACING=true langgraph dev
```

## 결론

Open Deep Research는 강력하면서도 유연한 AI 기반 연구 자동화 도구입니다. 주요 장점은:

✅ 완전 오픈소스: 자유롭게 사용 및 수정 가능
✅ 검증된 성능: Deep Research Bench #6 순위
✅ 비용 효율성: 토큰 사용량 최적화
✅ 확장 가능: 다양한 모델 및 검색 도구 지원
✅ 사용자 친화적: LangGraph Studio UI 제공

연구, 보고서 작성, 정보 분석 등 다양한 분야에서 활용할 수 있으며, 특히 심층적인 조사가 필요한 작업에서 큰 도움이 됩니다.

## 참고 자료

- GitHub 저장소: [langchain-ai/open_deep_research](https://github.com/langchain-ai/open_deep_research)
- LangChain 문서: [python.langchain.com](https://python.langchain.com)
- LangGraph 문서: [langchain-ai.github.io/langgraph](https://langchain-ai.github.io/langgraph)
- LangSmith: [smith.langchain.com](https://smith.langchain.com)
- Deep Research Bench: 연구 에이전트 평가 벤치마크
