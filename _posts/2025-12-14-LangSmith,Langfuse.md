---
layout: post
title: LangSmith vs Langfuse
author: 'Juho'
date: 2025-12-14 00:00:00 +0900
categories: [LLM]
tags: [LangSmith, Langfuse, LLM, Observability, Tracing, Evaluation]
pin: True
toc: true
---

## 목차

1. [LangSmith란?](#langsmith란)
2. [Langfuse란?](#langfuse란)
3. [LangSmith vs Langfuse: 공통점](#langsmith-vs-langfuse-공통점)
4. [LangSmith vs Langfuse: 핵심 차이점](#langsmith-vs-langfuse-핵심-차이점)
5. [어떤 상황에서 사용해야 할까?](#어떤-상황에서-사용해야-할까)
6. [설치 및 사용 방법](#설치-및-사용-방법)
7. [실전 사용 예시](#실전-사용-예시)
8. [비용 고려사항](#비용-고려사항)
9. [결론](#결론)
10. [참고 자료](#참고-자료)

## 개요

LLM(Large Language Model) 애플리케이션을 개발할 때, 성능 모니터링, 디버깅, 품질 평가는 필수적입니다.  
LangSmith와 Langfuse는 이러한 요구사항을 해결하기 위한 대표적인 플랫폼입니다.  
이 글에서는 두 플랫폼의 차이점과 각각을 어떤 상황에서 사용해야 하는지 그리고 기본적인 사용 방법에 대해 알아보겠습니다.  

## LangSmith란?

### 핵심 개념

LangSmith는 LangChain 팀이 개발한 LLM 애플리케이션 개발, 디버깅, 배포를 위한 통합 플랫폼입니다.
로컬 프로토타이핑에서 프로덕션 배포까지 전체 라이프사이클을 지원하며, **Python/JS SDK 및 REST API를 통해 프레임워크에 무관하게 사용할 수 있습니다.**
LangChain 없이도 독립적으로 사용 가능하며, LangChain/LangGraph와 사용 시 추가적인 자동 통합 기능을 제공합니다.

### 주요 기능

#### 1. Observability (관찰성)
- 애플리케이션의 모든 단계를 상세히 추적하고 시각화
- 병목 지점 파악 및 성능 최적화 지원
- 비용 및 지연시간 추적

#### 2. Tracing (추적)
- Python/JS SDK 및 REST API를 통한 직접 계측
- LangChain, LangGraph 사용 시 자동 추적 지원
- OpenAI, Anthropic 등 주요 LLM 프로바이더 직접 지원
- 수동 API를 통한 커스텀 계측 가능

#### 3. Evaluation (평가)
- 데이터셋 생성 및 평가자 설정
- 실험 비교를 통한 품질 측정
- 시간에 따른 품질 추적으로 신뢰성 확보

#### 4. Prompt Engineering
- 프롬프트 버전 관리 및 협업
- A/B 테스트 기능
- 실시간 프롬프트 개선

#### 5. 추가 기능
- Agent Server: 프로덕션 배포 지원
- Agent Builder: 코드 작성 없이 시각적으로 에이전트 설계 (베타)
- Studio: 엔드-투-엔드 애플리케이션 디자인 및 테스트

### 보안 및 규정 준수
- SOC 2 Type 2, GDPR 준수
- 엔터프라이즈 표준 충족

---

## Langfuse란?

### 핵심 개념

Langfuse는 오픈소스 LLM 엔지니어링 플랫폼으로, 팀이 LLM 애플리케이션을 협력하여 디버깅, 분석, 반복할 수 있도록 지원합니다.
자체 호스팅이 가능하며 확장 가능한 구조를 갖추고 있습니다.  

### 주요 기능

#### 1. Observability (관찰성)
- Traces: LLM 호출, 검색, 임베딩, API 호출 등 모든 상호작용 기록
- Multi-turn 대화 추적: 세션과 사용자 추적 지원
- Agent 시각화: 복잡한 워크플로우를 그래프로 표현
- 다양한 통합: Python/JS 네이티브 SDK, 다양한 프레임워크, OpenTelemetry 지원

#### 2. Prompt Management (프롬프트 관리)
- 버전 관리 및 협업 편집
- LLM Playground에서 상호작용식 테스트
- 데이터셋 대비 실험 실행
- 프로덕션 배포 (레이블 기반, 코드 변경 불필요)

#### 3. Evaluation (평가)
- LLM-as-a-Judge: 자동 평가
- 사용자 피드백 수집
- 수동 라벨링 (Annotation Queue)
- 커스텀 점수 시스템

### 주요 장점
- 오픈소스 기반
- 최소 성능 오버헤드로 설계
- 텍스트와 이미지 등 다양한 형식 지원
- 프레임워크 독립적 (Framework-agnostic)

---

## LangSmith vs Langfuse: 공통점

두 플랫폼은 LLM 애플리케이션 개발과 운영에 필수적인 핵심 기능들을 공유하고 있습니다.

### 1. Observability (관찰성)

공통 기능:
- LLM 호출의 전체 라이프사이클 추적
- 실시간 모니터링 및 시각화
- 복잡한 워크플로우와 Agent 동작 추적
- 멀티턴 대화 및 세션 관리

목적: 애플리케이션의 모든 단계를 투명하게 파악하여 빠른 디버깅과 성능 최적화를 지원합니다.

### 2. Tracing (추적)

공통 기능:
- 분산 추적(Distributed Tracing) 지원
- 각 요청의 상세한 실행 경로 기록
- 비용 및 지연시간(Latency) 추적
- 입력/출력 데이터 로깅
- 에러 및 예외 상황 캡처

목적: 병목 지점을 파악하고 성능 문제를 신속하게 해결할 수 있습니다.

### 3. Evaluation (평가)

공통 기능:
- 데이터셋 생성 및 관리
- 자동화된 평가 시스템
- 사용자 피드백 수집
- 수동 라벨링 및 주석 달기
- 커스텀 평가 메트릭 정의

목적: LLM 출력의 품질을 지속적으로 측정하고 개선할 수 있습니다.

### 4. Prompt Management (프롬프트 관리)

공통 기능:
- 프롬프트 버전 관리
- 팀 협업 지원
- 프롬프트 테스트 환경 (Playground)
- 프로덕션 배포 기능
- A/B 테스트 지원

목적: 프롬프트 개발 프로세스를 체계화하고 팀 간 협업을 원활하게 합니다.

### 5. 통합 및 SDK

공통 기능:
- Python 및 JavaScript/TypeScript SDK 제공
- 주요 LLM 프로바이더 통합 (OpenAI, Anthropic 등)
- REST API 제공
- LangChain 프레임워크 지원
- 다양한 프레임워크 및 라이브러리 통합

목적: 기존 워크플로우에 쉽게 통합하여 즉시 사용할 수 있습니다.

### 6. 분석 및 대시보드

공통 기능:
- 사용량 통계 및 분석
- 비용 추적 및 예측
- 성능 메트릭 시각화
- 시계열 데이터 분석
- 필터링 및 검색 기능

목적: 데이터 기반 의사결정을 통해 애플리케이션을 지속적으로 개선할 수 있습니다.

### 7. 협업 기능

공통 기능:
- 팀 멤버 관리
- 프로젝트/조직 단위 관리
- 공유 및 권한 관리
- 주석 및 코멘트 기능

목적: 여러 팀원이 효율적으로 협업할 수 있는 환경을 제공합니다.

### 8. 배포 옵션

공통 기능:
- 클라우드 호스팅 옵션 제공
- API를 통한 프로그래밍 방식 접근
- 다양한 환경 지원 (개발, 스테이징, 프로덕션)

목적: 개발 단계부터 프로덕션까지 일관된 도구 사용이 가능합니다.

### 핵심 가치 제안

두 플랫폼 모두 다음과 같은 핵심 가치를 제공합니다:

- 투명성: LLM 애플리케이션의 "블랙박스"를 열어 내부 동작 이해
- 신뢰성: 지속적인 모니터링과 평가를 통한 안정적인 서비스 제공
- 효율성: 자동화된 추적과 분석으로 개발 시간 단축
- 품질 보증: 체계적인 평가를 통한 일관된 출력 품질 유지
- 비용 최적화: 상세한 사용량 추적을 통한 비용 관리

---

## LangSmith vs Langfuse: 핵심 차이점

| 비교 항목 | LangSmith | Langfuse |
|---------|----------|----------|
| 라이선스 | 클로즈드 소스 | 오픈소스 (MIT 라이선스) |
| 클라우드 호스팅 | SaaS 제공 (무료 티어 + 유료 플랜) | SaaS 제공 (Free/Pro/Enterprise) |
| 자체 호스팅 | 엔터프라이즈 플랜에서 제공 | 오픈소스로 무료 자체 호스팅 가능 |
| SDK 지원 | Python/JS SDK, REST API | Python/JS SDK, REST API, OpenTelemetry |
| 프레임워크 통합 | 프레임워크 독립적. LangChain과 특히 긴밀한 통합 | 프레임워크 독립적, 다양한 통합 제공 |
| 대시보드 | 자동 생성 대시보드, 성숙한 모니터링/알림 | 커스터마이징 가능한 대시보드 |
| 데이터 제어 | SaaS (자체 호스팅은 엔터프라이즈 플랜) | SaaS 또는 완전한 자체 호스팅 가능 |
| 비용 효율성 | 소규모~중간 규모 적합 | 대규모 시 자체 호스팅으로 비용 절감 가능 |
| 투명성 | 제한적 (클로즈드 소스) | 높음 (오픈소스) |

---

## 어떤 상황에서 사용해야 할까?

### LangSmith를 선택해야 하는 경우

1. **관리형 SaaS 선호**: 인프라 관리 부담을 줄이고 즉시 사용하고 싶은 경우
2. **LangChain/LangGraph 사용**: LangChain 기반 프로젝트라면 자동 통합으로 더 빠른 설정 가능
3. **엔터프라이즈급 모니터링**: 미리 구축된 대시보드와 알림 시스템이 필요한 경우
4. **빠른 도입**: 초기 설정이 간단하고 즉시 사용 가능
5. **어떤 프레임워크든 사용 가능**: Python/JS SDK로 프레임워크 독립적 통합

### Langfuse를 선택해야 하는 경우

1. **오픈소스 우선**: 투명성, 커스터마이징, 데이터 제어가 중요한 경우
2. **무료 자체 호스팅**: 인프라가 준비되어 있고 비용 효율성을 원하는 경우
3. **벤더 독립성**: 장기적으로 벤더 종속성을 피하고 싶은 경우
4. **표준 프로토콜**: OpenTelemetry 등 표준 기반 통합 선호
5. **클라우드 옵션도 가능**: 자체 호스팅이 부담스러우면 클라우드 SaaS 이용 가능

### 사용 시기별 권장사항

| 단계 | LangSmith | Langfuse |
|-----|----------|----------|
| 프로토타이핑 | SDK로 빠른 시작 (LangChain 시 더 빠름) | 오픈소스로 자유로운 실험 |
| 개발 | 협업 및 프롬프트 테스트 | 버전 관리 및 실험 실행 |
| 프로덕션 | SaaS로 자동 모니터링 및 알림 | SaaS 또는 자체 호스팅 선택 가능 |

---

## 설치 및 사용 방법

### LangSmith 시작하기

#### 1. 계정 생성 및 API 키 발급

1. [smith.langchain.com](https://smith.langchain.com)에서 가입 (Google, GitHub, 이메일 지원)
2. 설정 → API Keys → Create API Key에서 API 키 발급

#### 2. 환경 변수 설정

```bash
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY="ls-..."
```

#### 3. Python SDK 설치

```bash
pip install -U langsmith
```

#### 4. 기본 사용 예시

##### LangChain 사용 시 (자동 추적)

```python
from langchain_openai import ChatOpenAI

# 환경 변수만 설정하면 자동으로 추적됨
llm = ChatOpenAI(model="gpt-4")
response = llm.invoke("Hello, world!")
```

##### LangChain 없이 직접 SDK 사용

```python
from langsmith import traceable
from openai import OpenAI

client = OpenAI()

@traceable
def my_llm_function(text: str):
    response = client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": text}]
    )
    return response.choices[0].message.content

# 자동으로 LangSmith에 추적됨
result = my_llm_function("Hello, world!")
```

#### 5. 추적 확인

[smith.langchain.com](https://smith.langchain.com) 대시보드에서 실시간으로 추적 내용 확인

---

### Langfuse 시작하기

#### 1. Python SDK 설치

```bash
pip install langfuse
```

#### 2. JavaScript/TypeScript SDK 설치

```bash
npm install langfuse
```

#### 3. 환경 변수 설정

```bash
export LANGFUSE_SECRET_KEY="sk-lf-..."
export LANGFUSE_PUBLIC_KEY="pk-lf-..."
export LANGFUSE_BASE_URL="https://cloud.langfuse.com"  # 또는 자체 호스팅 URL
```

#### 4. 기본 사용 예시

##### Python - 데코레이터 방식

```python
from langfuse import observe, get_client

@observe
def my_llm_function():
    # LLM 호출 로직
    return "Hello, world!"

# 함수 실행 시 자동으로 추적됨
my_llm_function()

# 데이터 전송 완료
langfuse = get_client()
langfuse.flush()
```

##### Python - OpenAI 통합

```python
from langfuse.openai import openai

# OpenAI 클라이언트를 Langfuse로 래핑
completion = openai.chat.completions.create(
    name="test-chat",
    model="gpt-4o",
    messages=[{"role": "user", "content": "1 + 1 = "}]
)

# 자동으로 모델 호출 기록
```

##### Python - 컨텍스트 매니저 방식

```python
from langfuse import get_client

langfuse = get_client()

with langfuse.start_as_current_observation(
    as_type="span",
    name="process-request"
) as span:
    # 작업 수행
    result = process_data()
    span.update(output=result)
```

#### 5. 자체 호스팅 (선택사항)

Docker를 사용한 자체 호스팅:

```bash
docker run -d \
  -p 3000:3000 \
  -e DATABASE_URL="postgresql://..." \
  langfuse/langfuse:latest
```

Kubernetes, VM 등 다양한 배포 옵션 지원

---

## 실전 사용 예시

### LangSmith - LangChain 애플리케이션 추적

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.schema.output_parser import StrOutputParser
import os

# 환경 변수 설정
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls-..."

# 체인 구성
prompt = ChatPromptTemplate.from_template("Tell me a joke about {topic}")
model = ChatOpenAI(model="gpt-4")
output_parser = StrOutputParser()

chain = prompt | model | output_parser

# 실행 - 자동으로 LangSmith에 추적됨
result = chain.invoke({"topic": "AI"})
print(result)
```

### Langfuse - 커스텀 LLM 애플리케이션 추적

```python
from langfuse import observe
import openai

@observe(name="summarize-article")
def summarize_article(article_text: str):
    """기사 요약 함수"""

    response = openai.chat.completions.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "You are a helpful summarizer."},
            {"role": "user", "content": f"Summarize this article:\n\n{article_text}"}
        ]
    )

    return response.choices[0].message.content

# 사용
article = "Long article text here..."
summary = summarize_article(article)

# Langfuse 대시보드에서 자동으로 추적 확인 가능
```

### Langfuse - 평가(Evaluation) 예시

```python
from langfuse import get_client

langfuse = get_client()

# 평가 점수 추가
langfuse.score(
    trace_id="trace-id-123",
    name="user-feedback",
    value=1,  # 1 = positive, 0 = negative
    comment="Great response!"
)
```

---

## 비용 고려사항

### LangSmith
- **클라우드 SaaS**: 무료 티어 + 사용량 기반 유료 플랜
- **자체 호스팅**: 엔터프라이즈 플랜에서 제공 (별도 문의)
- **장점**: 초기~중간 규모에서 관리 부담 없이 빠른 도입
- **비용 구조**: 추적 횟수, 데이터 저장량 기반 과금

### Langfuse
- **클라우드 SaaS**: Free / Pro / Enterprise 플랜 제공
- **자체 호스팅**: 오픈소스로 무료 (인프라 비용만 발생)
- **장점**: 대규모 시 자체 호스팅으로 비용 절감 가능
- **고려사항**: 자체 호스팅 시 운영 인력 및 인프라 필요

> **참고**: 가격 정책은 변경될 수 있으므로 최신 정보는 각 플랫폼의 공식 웹사이트를 참조하세요.

---

## 결론

### 빠른 선택 가이드

**LangSmith는 이런 경우에:**
- 관리형 SaaS로 빠른 시작 선호
- LangChain/LangGraph 사용 시 자동 통합 활용
- 엔터프라이즈급 모니터링 및 알림 필요
- 프레임워크 독립적 사용도 가능 (Python/JS SDK)

**Langfuse는 이런 경우에:**
- 오픈소스 및 투명성 중시
- 무료 자체 호스팅으로 데이터 완전 제어
- 벤더 종속성 최소화
- 대규모 시 비용 효율성 추구
- SaaS 클라우드 옵션도 제공

두 플랫폼 모두 2025년 현재 활발히 개발되고 있으며, LLM 애플리케이션의 관찰성, 평가, 모니터링에 강력한 도구를 제공합니다. 프로젝트의 요구사항, 팀의 기술 스택, 예산, 그리고 장기적인 전략을 고려하여 선택하시면 됩니다.

---

## 참고 자료

- [LangSmith 공식 문서](https://docs.langchain.com/langsmith)
- [Langfuse 공식 문서](https://langfuse.com/docs)