---
layout: post
title: "Agent Middleware: 6개의 훅으로 에이전트 하네스 커스터마이징하기"
author: 'Juho'
date: 2026-05-08 00:00:00 +0900
categories: [LangChain]
tags: [LangChain, Agent, Skill]
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
2. [미들웨어란 무엇인가](#미들웨어란-무엇인가)
3. [6개의 라이프사이클 훅](#6개의-라이프사이클-훅)
4. [내장 미들웨어 목록](#내장-미들웨어-목록)
5. [커스터마이징 패턴](#커스터마이징-패턴)
6. [구현 방법](#구현-방법)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

LangChain의 에이전트 미들웨어는 에이전트 코어 루프의 각 단계 전후에 커스텀 로직을 실행할 수 있게 하는 훅 시스템이다.
프롬프트와 도구 정의 같은 기본 커스터마이징은 단순하지만, 미들웨어는 코어 루프 자체를 수정하지 않고도 깊이 있는 조작을 가능하게 한다.
이는 안정적인 기반을 유지하면서 동시에 비즈니스 로직, 컴플라이언스, 동적 제어, 컨텍스트 관리, 프로덕션 안정성을 달성하는 핵심 추상화다.

## 미들웨어란 무엇인가

에이전트 하네스는 LLM을 환경에 연결하는 단순한 루프다.
모델이 반복 실행되며 도구를 호출한다.
미들웨어는 이 루프의 각 단계에 훅을 노출해, 프롬프트나 도구만으로는 표현하기 어려운 결정적 정책과 동적 동작을 삽입할 수 있게 한다.

## 6개의 라이프사이클 훅

| 훅 | 시점 | 주요 용도 |
|------|------|------|
| before_agent | 에이전트 시작 시 | 메모리 로딩, 입력 검증 |
| before_model | 매 모델 호출 전 | 히스토리 트리밍, PII 필터링 |
| wrap_model_call | 모델 호출 전체 래핑 | 캐싱, 재시도, 동적 도구 주입 |
| wrap_tool_call | 도구 실행 래핑 | 컨텍스트 주입, 결과 가로채기 |
| after_model | 응답 후 실행 전 | Human-in-the-loop 워크플로우 |
| after_agent | 완료 시 | 정리, 알림 |

각 훅은 특정 라이프사이클 시점에 결정적 로직을 삽입할 수 있는 진입점이다.
`wrap_model_call`과 `wrap_tool_call`은 단순한 전후 훅이 아니라 호출 전체를 래핑하므로, 캐싱이나 재시도 같은 흐름 제어에 적합하다.

## 내장 미들웨어 목록

LangChain은 즉시 사용 가능한 미들웨어들을 제공한다.

- **PIIMiddleware**: 입출력 전반에서 민감 데이터 마스킹과 편집
- **LLMToolSelectorMiddleware**: 빠른 LLM으로 관련 도구를 동적 필터링
- **SummarizationMiddleware**: 토큰 인식 요약으로 컨텍스트 오버플로 관리
- **ModelRetryMiddleware**: 설정 가능한 백오프와 함께 재시도 로직 구현
- **ShellToolMiddleware**: 셸 리소스 초기화와 정리 관리
- **FilesystemMiddleware**: 파일 기반 컨텍스트 관리
- **SubagentMiddleware**: 컨텍스트 격리를 가진 서브에이전트 활성화
- **SkillsMiddleware**: 능력의 점진적 공개

이 목록은 에이전트가 프로덕션에서 실제로 마주하는 문제들에 대한 답을 제시한다.
PII 처리는 프롬프트에 넣을 수 없는 컴플라이언스 요구사항이고, SummarizationMiddleware는 컨텍스트 윈도우를 초과하는 작업에서 필수적이다.

## 커스터마이징 패턴

미들웨어의 사용 사례는 네 가지로 분류된다.

| 패턴 | 설명 |
|------|------|
| Business Logic & Compliance | PII 편집, 콘텐츠 모더레이션 같이 프롬프트에 둘 수 없는 결정적 정책 |
| Dynamic Agent Control | 도구 주입과 모델 스왑을 포함한 런타임 재구성 |
| Context Management | 토큰 한계 처리와 장황한 출력 트리밍 |
| Production Readiness | 재시도 로직, 폴백, Human interrupt |

특히 첫 번째 패턴이 중요하다.
"can't live in a prompt"라는 표현이 강조하듯, 컴플라이언스는 결정적이어야 하며 LLM의 확률적 동작에 맡길 수 없다.

## 구현 방법

LangChain은 두 가지 진입점을 제공한다.

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware, SummarizationMiddleware

agent = create_agent(
    model="gpt-5.2-codex",
    tools=[...],
    middleware=[
        PIIMiddleware(),
        SummarizationMiddleware(max_tokens=8000),
    ],
)
```

`create_agent`는 미들웨어를 지정해 최소한의 커스터마이징을 적용한다.
`create_deep_agent`는 의견이 반영된 미들웨어 스택이 포함된 배터리 포함 하네스를 제공한다.

커스텀 미들웨어가 필요하면 `AgentMiddleware`를 서브클래싱한다.

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def before_model(self, state, runtime):
        return state

    def after_model(self, state, runtime):
        return state
```

미들웨어는 합성 가능하므로 여러 개를 조합해 유연한 동작을 구성할 수 있다.

## 결론

미들웨어는 에이전트 하네스의 확장 표면이다.
6개의 훅은 라이프사이클의 모든 의미 있는 시점을 덮으며, 내장 미들웨어들은 PII 처리부터 서브에이전트 격리까지 즉시 사용 가능한 솔루션을 제공한다.
프롬프트로 표현할 수 없는 결정적 동작, 런타임에 에이전트를 재구성하는 동적 제어, 프로덕션에 필요한 재시도와 폴백을 모두 한 추상화 안에서 다룰 수 있다는 점이 핵심이다.

## Reference

- [How Middleware Lets You Customize Your Agent Harness](https://www.langchain.com/blog/how-middleware-lets-you-customize-your-agent-harness/)
