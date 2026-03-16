---
layout: post
title: "OpenAI AI 에이전트 구축 실용 가이드 - 설계부터 배포까지"
author: 'Juho'
date: 2026-03-16 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, OpenAI]
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
2. [에이전트란 무엇인가](#에이전트란-무엇인가)
3. [에이전트를 구축해야 하는 시점](#에이전트를-구축해야-하는-시점)
4. [핵심 설계 요소](#핵심-설계-요소)
   - [모델 선택](#모델-선택)
   - [도구 정의](#도구-정의)
   - [지침 구성](#지침-구성)
5. [오케스트레이션 패턴](#오케스트레이션-패턴)
   - [단일 에이전트 시스템](#단일-에이전트-시스템)
   - [다중 에이전트 시스템](#다중-에이전트-시스템)
6. [가드레일](#가드레일)
7. [휴먼 인터벤션](#휴먼-인터벤션)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

OpenAI가 AI 에이전트를 구축하기 위한 실용 가이드를 공개했다.
이 가이드는 에이전트의 정의부터 설계 기초, 오케스트레이션 패턴, 가드레일, 휴먼 인터벤션까지 에이전트 구축의 전 과정을 다루고 있다.
에이전트는 사용자를 대신해 독립적으로 작업을 수행하는 시스템으로, 작게 시작해 실제 사용자와 검증하고 시간이 지나며 역량을 성장시키는 반복적 접근이 핵심이다.

## 에이전트란 무엇인가

에이전트는 사용자를 대신하여 독립적으로 작업을 수행하는 시스템이다.
워크플로우가 단순한 단계의 연속이라면, 에이전트는 그 이상의 핵심 특성을 갖추고 있다.

에이전트의 핵심 특성은 다음과 같다.

- LLM을 활용하여 워크플로우를 실행하고 의사결정을 내린다.
- 작업 완료를 인식하고 스스로 수정할 수 있다.
- 가드레일이 적용된 도구에 접근할 수 있다.

## 에이전트를 구축해야 하는 시점

에이전트는 기존 자동화로 해결하기 어려웠던 워크플로우에 우선적으로 도입해야 한다.
다음과 같은 상황에서 에이전트 도입을 고려할 수 있다.

| 상황 | 설명 | 예시 |
|------|------|------|
| 복잡한 의사결정 | 미묘한 판단, 예외 처리, 맥락에 민감한 결정이 필요한 경우 | 환불 승인 |
| 유지보수가 어려운 규칙 | 규칙 세트가 비대하고 관리하기 어려운 경우 | 벤더 보안 리뷰 |
| 비정형 데이터 의존도가 높은 경우 | 자연어, 문서 추출 등 비정형 데이터를 많이 다루는 경우 | 보험 청구 처리 |

## 핵심 설계 요소

에이전트 설계의 기초는 세 가지 핵심 구성 요소로 이루어진다.

1. 모델(Model) - 추론과 의사결정을 담당하는 LLM
2. 도구(Tools) - 실제 동작을 수행하는 외부 함수 및 API
3. 지침(Instructions) - 명시적 가이드라인과 가드레일

### 모델 선택

모델을 선택할 때는 다음과 같은 접근법을 권장한다.

- 가장 능력 있는 모델로 먼저 프로토타입을 구축한다.
- 허용 가능한 수준의 성능이 나오는 곳에서 더 작은 모델로 교체한다.
- 평가(eval)를 설정하고, 정확도에 집중하며, 비용과 지연 시간을 최적화한다.

```python
from agents import Agent

agent = Agent(
    name="refund_agent",
    model="gpt-4o",
    instructions="You are a refund approval agent. Review refund requests and approve or deny based on policy.",
    tools=[check_order_status, process_refund]
)
```

### 도구 정의

도구는 세 가지 유형으로 분류된다.

| 유형 | 설명 | 예시 |
|------|------|------|
| 데이터 도구(Data) | 컨텍스트를 검색하고 조회한다 | 데이터베이스 쿼리, PDF 읽기, 웹 검색 |
| 액션 도구(Action) | 시스템과 상호작용한다 | 이메일 전송, CRM 업데이트 |
| 오케스트레이션 도구(Orchestration) | 에이전트를 다른 에이전트의 도구로 사용한다 | 에이전트 간 위임 |

```python
from agents import Agent, function_tool

@function_tool
def check_order_status(order_id: str) -> str:
    """Check the status of a customer order."""
    return db.get_order(order_id)

@function_tool
def process_refund(order_id: str, amount: float) -> str:
    """Process a refund for the given order."""
    return payment.refund(order_id, amount)

agent = Agent(
    name="support_agent",
    tools=[check_order_status, process_refund]
)
```

### 지침 구성

지침을 효과적으로 구성하기 위한 모범 사례는 다음과 같다.

- 기존 문서(운영 절차, 지원 스크립트 등)를 활용한다.
- 에이전트에게 작업을 세분화하도록 프롬프트한다.
- 모든 단계에 대해 명확한 행동을 정의한다.
- 조건 분기를 통해 엣지 케이스를 포착한다.

```python
agent = Agent(
    name="refund_agent",
    instructions="""You are a refund approval agent. Follow these steps:
1. Verify the order exists using check_order_status
2. Check if the order is eligible for refund (within 30 days)
3. If eligible, process the refund
4. If not eligible, explain the reason to the customer

Edge cases:
- If the order is partially shipped, only refund unshipped items
- If the customer has exceeded 3 refunds this year, escalate to manager
"""
)
```

## 오케스트레이션 패턴

### 단일 에이전트 시스템

단일 에이전트 시스템은 하나의 모델이 종료 조건이 충족될 때까지 루프에서 도구를 사용하는 구조이다.
정책 변수가 포함된 프롬프트 템플릿을 활용하면 유연성을 확보할 수 있다.
도구를 점진적으로 추가하여 단일 에이전트로도 많은 작업을 처리할 수 있다.

```python
from agents import Agent, Runner

agent = Agent(
    name="customer_support",
    instructions="""You handle customer inquiries. Use available tools to resolve issues.
    Continue working until the customer's issue is fully resolved.""",
    tools=[check_order, process_refund, send_email, search_faq]
)

result = Runner.run_sync(agent, "I want to return my order #12345")
print(result.final_output)
```

다중 에이전트 도입을 고려해야 하는 시점은 다음과 같다.

- 복잡한 로직에 많은 조건 분기가 있는 경우
- 도구가 과다하여 유사성이나 중복 문제가 발생하는 경우

### 다중 에이전트 시스템

다중 에이전트 시스템은 두 가지 주요 패턴으로 나뉜다.

#### 매니저 패턴

중앙의 매니저 에이전트가 도구 호출을 통해 전문 에이전트들을 오케스트레이션한다.
매니저가 제어권과 결과 종합을 유지한다.

```python
from agents import Agent

billing_agent = Agent(
    name="billing_agent",
    instructions="Handle billing inquiries and payment issues.",
    tools=[check_balance, update_payment]
)

technical_agent = Agent(
    name="technical_agent",
    instructions="Handle technical support and troubleshooting.",
    tools=[run_diagnostic, check_logs]
)

manager_agent = Agent(
    name="manager",
    instructions="""You are a customer service manager.
    Route inquiries to the appropriate specialist agent.""",
    tools=[billing_agent.as_tool(), technical_agent.as_tool()]
)
```

#### 분산 패턴

에이전트들이 동등한 관계에서 서로 핸드오프를 수행한다.
각 에이전트가 제어권을 넘겨받아 사용자와 직접 상호작용할 수 있다.

```python
from agents import Agent

billing_agent = Agent(
    name="billing_agent",
    instructions="Handle billing inquiries. Transfer to technical agent for tech issues.",
    tools=[check_balance, update_payment],
    handoffs=[technical_agent]
)

technical_agent = Agent(
    name="technical_agent",
    instructions="Handle technical issues. Transfer to billing agent for payment issues.",
    tools=[run_diagnostic, check_logs],
    handoffs=[billing_agent]
)
```

#### 선언적 vs 비선언적 그래프

오케스트레이션 접근 방식은 두 가지로 나뉜다.

| 접근 방식 | 특징 |
|-----------|------|
| 선언적(Declarative) | 모든 분기를 사전에 정의하며, 시각적 명확성이 있지만 복잡해질 수 있다 |
| 코드 우선(Code-first) | Agents SDK를 사용하며, 유연하고 익숙한 프로그래밍 구조를 활용한다 |

## 가드레일

가드레일은 다층 방어 체계로 구성된다.
에이전트 시스템의 안전성과 신뢰성을 확보하기 위해 여러 유형의 가드레일을 조합하여 적용해야 한다.

| 유형 | 설명 |
|------|------|
| 관련성 분류기(Relevance Classifier) | 입력이 에이전트의 범위에 해당하는지 판별한다 |
| 안전성 분류기(Safety Classifier) | 유해하거나 부적절한 콘텐츠를 감지한다 |
| PII 필터 | 개인 식별 정보를 감지하고 제거한다 |
| 모더레이션(Moderation) | 콘텐츠 정책 위반을 검사한다 |
| 도구 안전장치(Tool Safeguards) | 도구에 위험 등급을 부여하여 관리한다 |
| 규칙 기반 보호(Rules-based) | 사전 정의된 규칙으로 보호한다 |
| 출력 검증(Output Validation) | 최종 출력의 품질과 적절성을 검증한다 |

가드레일 구축을 위한 휴리스틱은 다음과 같다.

1. 데이터 프라이버시와 콘텐츠 안전에 먼저 집중한다.
2. 실제 운영에서 발견되는 엣지 케이스를 기반으로 추가한다.
3. 보안과 사용자 경험을 동시에 최적화한다.

## 휴먼 인터벤션

에이전트 시스템에서 사람의 개입이 필요한 두 가지 트리거가 있다.

- 실패 임계값 초과 - 에이전트가 설정된 실패 횟수를 넘어선 경우
- 고위험 행동 - 민감하거나, 되돌릴 수 없거나, 영향이 큰 작업을 수행하려는 경우

이러한 트리거를 적절히 설정하면 에이전트의 자율성과 안전성 사이의 균형을 맞출 수 있다.

## 결론

OpenAI의 에이전트 구축 가이드는 작게 시작하여 실제 사용자와 함께 검증하고, 시간이 지나면서 역량을 키워나가는 반복적 접근법을 강조한다.
에이전트 시스템의 성공적인 구축을 위해서는 적절한 모델 선택, 명확한 도구 정의, 체계적인 지침 구성이 기본이 된다.
단일 에이전트로 시작하여 필요에 따라 다중 에이전트로 확장하고, 가드레일과 휴먼 인터벤션을 통해 안전성을 확보하는 것이 권장된다.
코드 예제는 OpenAI의 [Agents SDK](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/){:target="_blank"}를 기반으로 제공되며, Python을 사용하여 구현할 수 있다.

## Reference

- [A Practical Guide to Building AI Agents](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)
