---
layout: post
title: "안전한 자연어 기반 API 구축 방법 - 프로덕션 환경을 위한 아키텍처 가이드"
author: 'Juho'
date: 2026-02-14 02:00:00 +0900
categories: [AI]
tags: [AI, LLM, LangChain, FastAPI]
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
2. [자연어 API의 프로덕션 위험](#자연어-api의-프로덕션-위험)
3. [핵심 설계 원칙 - 자연어는 실행이 아닌 구조로 변환](#핵심-설계-원칙---자연어는-실행이-아닌-구조로-변환)
4. [2단계 API 레이어 아키텍처](#2단계-api-레이어-아키텍처)
5. [안전 메커니즘 상세](#안전-메커니즘-상세)
6. [LangGraph를 활용한 구현](#langgraph를-활용한-구현)
7. [성능 지표](#성능-지표)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

자연어를 API 입력으로 받아들이는 것은 강력한 사용자 경험을 제공하지만, 프로덕션 환경에서는 심각한 위험을 수반한다.
Microsoft 기술 블로그에서 제안하는 핵심 원칙은 명확하다.
"자연어는 직접 실행되어서는 안 되며, 반드시 구조화된 형태로 변환되어야 한다."
이 글에서는 자연어 기반 API를 안전하게 구축하기 위한 2단계 레이어 아키텍처와 핵심 안전 메커니즘을 정리한다.

## 자연어 API의 프로덕션 위험

자연어를 API 계약(contract)으로 직접 사용할 경우 다음과 같은 프로덕션 위험이 발생한다.

- 비결정적 동작(Nondeterministic Behavior): 같은 입력에 대해 다른 결과가 나올 수 있다
- 프롬프트 기반 비즈니스 로직: 비즈니스 규칙이 프롬프트에 묶여 버전 관리와 테스트가 어렵다
- 디버깅 난이도 증가: 오류 발생 시 원인 추적이 극도로 어렵다
- 사일런트 실패(Silent Failures): 잘못된 해석이 오류 없이 실행되어 데이터 오염을 유발한다

이러한 위험을 해결하기 위해 의도 발견(Intent Discovery)과 결정론적 실행(Deterministic Execution)을 분리하는 아키텍처가 필요하다.

## 핵심 설계 원칙 - 자연어는 실행이 아닌 구조로 변환

이 아키텍처의 핵심은 LLM의 역할을 발견과 추출에만 한정하는 것이다.
LLM은 사용자의 자연어 입력에서 의도(Intent)와 엔티티(Entity)를 추출하는 역할만 수행한다.
추출된 결과는 사전 정의된 스키마로 변환되며, 실제 비즈니스 로직 실행은 검증된 구조화 데이터로만 이루어진다.

핵심 비유: LLM을 엔진이 아닌 컴파일러로 생각해야 한다.

## 2단계 API 레이어 아키텍처

### 1단계: Semantic Parse API (Language → Structure)

사용자의 자연어 입력을 구조화된 요청으로 변환하는 단계다.

- 사용자 텍스트를 수신
- LLM을 통해 의도(Intent)와 엔티티(Entity)를 추출
- 사전 정의된 스키마를 완성
- 정보가 부족하면 명확화 질문(Clarification) 요청
- 검증된 정규화 구조 요청(Canonical Structured Request) 반환

### 2단계: Structured Execution API (Structure → Action)

검증된 구조화 데이터만 받아 실행하는 단계다.

- 검증된 구조화 입력만 수용
- 다운스트림 시스템 호출
- 결정론적(Deterministic)이고 버전 관리 가능
- 자연어 처리 없음
- 완전한 테스트와 재실행(Replay)이 가능

### 레이어 비교

| 항목 | Semantic Parse API | Structured Execution API |
|------|-------------------|------------------------|
| 입력 | 자연어 텍스트 | 구조화된 JSON |
| 역할 | 의도 추출, 스키마 완성 | 비즈니스 로직 실행 |
| LLM 사용 | 사용 | 미사용 |
| 결정론성 | 비결정론적 | 결정론적 |
| 테스트 | 평가 기반 | 단위/통합 테스트 가능 |

## 안전 메커니즘 상세

### Canonical Schemas (정규 스키마)

각 지원되는 의도(Intent)는 코드 기반 스키마로 정의된다.
스키마에는 필수 필드, 허용 범위, 유효성 검사 규칙이 명시된다.
이 스키마가 프롬프트가 아닌 실질적인 API 계약이 된다.

```python
class FlightSearchSchema(BaseModel):
    origin: str
    destination: str
    departure_date: date
    return_date: Optional[date] = None
    passengers: int = Field(ge=1, le=9)
    cabin_class: Literal["economy", "business", "first"]
```

### Confidence Gates (신뢰도 게이트)

LLM의 엔티티 추출 결과에는 신뢰도 점수가 포함된다.
신뢰도가 임계값 이하로 떨어지면 잘못된 해석으로 진행하는 대신 사용자에게 명확화 질문을 요청한다.
이를 통해 사일런트 실패를 방지한다.

### Schema Completion (스키마 완성)

자유형 대화(Free-form Chat) 대신 구조화된 명확화 프로세스를 사용한다.
필수 필드가 누락되면 해당 필드에 대한 타겟 질문을 생성하면서 기존 상태를 유지한다.
대화가 진행될수록 스키마가 점진적으로 완성되는 구조다.

### Lightweight Ontologies (경량 온톨로지)

코드 수준의 매핑 규칙으로 사용자 용어를 정규 값으로 변환한다.

```python
PRICE_ONTOLOGY = {
    "cheaper": {"price_bias": -2},
    "budget-friendly": {"price_bias": -2},
    "affordable": {"price_bias": -1},
    "premium": {"price_bias": 2},
    "luxury": {"price_bias": 3},
}
```

이러한 매핑을 통해 다양한 사용자 표현을 표준화된 수치 값으로 변환할 수 있다.

## LangGraph를 활용한 구현

Semantic Parse API의 워크플로우 구현에는 LangGraph가 권장된다.
LangGraph를 사용하면 다음과 같은 파이프라인을 명시적으로 구성할 수 있다.

1. 의도 분류(Intent Classification)
2. 엔티티 추출(Entity Extraction)
3. 스키마 상태 병합(Schema State Merging)
4. 필드 유효성 검사(Field Validation)
5. 완성 또는 명확화 라우팅(Completion or Clarification Routing)

각 단계가 그래프의 노드로 표현되며, 조건부 분기를 통해 정보가 충분할 때만 실행 단계로 넘어간다.

## 성능 지표

Semantic Parse 레이어의 지연 시간은 채팅 인터페이스에서 충분히 수용 가능한 수준이다.

| 단계 | 지연 시간 |
|------|----------|
| 의도 분류 | 약 40ms |
| 엔티티 추출 | 약 200ms |
| 유효성 검사 및 라우팅 | 약 1ms |
| 전체 오버헤드 | 약 250-300ms |

전체 오버헤드가 250-300ms 수준으로, 채팅 UX에서 사용자 체감에 큰 영향을 주지 않는다.

## 결론

자연어 기반 API를 프로덕션에 안전하게 배포하려면 자연어 처리와 실행을 반드시 분리해야 한다.
LLM은 자연어를 구조화된 요청으로 변환하는 컴파일러 역할에만 한정하고, 실제 실행은 검증된 구조 데이터로만 수행해야 한다.
Canonical Schemas, Confidence Gates, Schema Completion, Lightweight Ontologies라는 4가지 안전 메커니즘을 결합하면 비결정적 동작과 사일런트 실패 위험을 효과적으로 완화할 수 있다.
이 아키텍처는 AI 시대에 자연어 인터페이스의 편의성과 프로덕션 안정성을 동시에 달성할 수 있는 실용적인 접근법이다.

## Reference

- [How to Build Safe Natural Language-Driven APIs - Microsoft Tech Community](https://techcommunity.microsoft.com/blog/azuredevcommunityblog/how-to-build-safe-natural-language-driven-apis/4488509/)
