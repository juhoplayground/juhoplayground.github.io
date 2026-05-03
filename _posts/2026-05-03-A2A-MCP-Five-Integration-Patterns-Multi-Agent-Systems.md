---
layout: post
title: "A2A와 MCP가 함께 동작하는 방식 - 멀티 에이전트 시스템을 위한 5가지 통합 패턴"
author: 'Juho'
date: 2026-05-03 00:00:00 +0900
categories: [A2A]
tags: [A2A, MCP, Agent, AI]
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
   - [Pattern 1: Agent Card Discovery](#pattern-1-agent-card-discovery)
   - [Pattern 2: Delegated Specialization](#pattern-2-delegated-specialization)
   - [Pattern 3: Tool Bridge (MCP)](#pattern-3-tool-bridge-mcp)
   - [Pattern 4: Cross-Organization Federation](#pattern-4-cross-organization-federation)
   - [Pattern 5: Ambient Event Mesh](#pattern-5-ambient-event-mesh)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google Cloud가 Cloud Next 26에서 A2A와 MCP의 통합 패턴을 정리해 발표했다.
A2A는 에이전트 간 통신을 위한 프로토콜이고, MCP는 에이전트와 도구/데이터 간 통신을 위한 프로토콜이다.
두 프로토콜을 결합해 엔터프라이즈 규모의 멀티 에이전트 시스템을 구축하는 5가지 패턴을 제시한다.

발표는 Addy Osmani와 Shubham Saboo가 공동 작성했으며, 결론은 단호하다.
"고립된 에이전트를 만드는 회사는 12개월 안에 리팩터링하게 될 것이고, 상호운용 가능한 에이전트 생태계를 만드는 회사는 매일 우위를 복리로 쌓아간다."

## 배경

어떤 조직도 필요한 모든 에이전트를 처음부터 직접 만들지는 않는다.
실제 가치는 다른 팀, 다른 언어, 다른 조직이 만든 에이전트를 발견하고 조립하는 데서 나온다.
A2A는 에이전트 간 통신을, MCP는 에이전트와 도구 간 통신을 담당하는 두 축이다.

Cloud Next 26에서 Google은 이 통합을 엔터프라이즈 스케일에서 실용화하는 인프라를 출시했다.
ADK는 Python, TypeScript, Go, Java에서 A2A를 지원하고 MCP를 네이티브로 처리한다.
GCP 데이터베이스에는 매니지드 MCP 지원이 제공되며, Gemini Enterprise의 Agent Gallery에는 100개 이상의 검증된 파트너 에이전트가 등록되어 있다.

## 핵심 내용

### Pattern 1: Agent Card Discovery

에이전트가 다른 에이전트에 작업을 위임하려면 그 존재와 능력, 통신 방법을 먼저 알아야 한다.
A2A는 이를 Agent Card로 해결한다.
모든 A2A 호환 에이전트는 well-known URL에 자신의 능력, 인증 요구사항, 레이트 리밋을 기술한 JSON 문서를 게시한다.

OpenAPI 스펙과 비슷하지만 클라이언트-서버 통신이 아닌 에이전트 간 상호작용을 위해 설계되었다는 점이 다르다.
ADK에서 자기 에이전트를 A2A 서버로 노출하는 것은 단일 설정이며, 정의로부터 Agent Card가 자동 생성된다.
원격 에이전트를 소비하는 쪽은 RemoteA2aAgent 컴포넌트를 사용한다.

이 컴포넌트는 인증, 직렬화, 에러 처리, 결과 스트리밍을 모두 처리한다.
조정자(coordinator)의 관점에서 다른 팀이 다른 언어로 만든 원격 에이전트는 로컬 에이전트와 동일하게 보인다.

Agent Registry는 이 효과를 증폭한다.
중앙 레지스트리에 에이전트를 등록하면 조직 전체가 특정 URL을 모르고도 능력을 발견할 수 있다.
레지스트리는 에이전트 생태계의 서비스 메시 역할을 한다.

### Pattern 2: Delegated Specialization

가장 흔한 통합 패턴은 위임이다.
자기 에이전트가 자신의 전문 영역을 벗어난 작업을 만나면 전문가에게 넘긴다.
Coordinator-Dispatcher 아키텍처를 팀과 프레임워크 경계 너머로 확장한 형태다.

전문가 에이전트는 ADK로 만들지 않아도 되고, 같은 언어로 짤 필요도 없으며, 같은 팀이 유지보수할 필요도 없다.
A2A를 말할 줄만 알면 된다.

다섯 팀과 네 언어에 걸친 고객 온보딩 워크플로를 보자.
Python으로 작성된 조정자가 보안팀의 Go 에이전트에 신원 확인을, 리스크팀의 Java 에이전트에 신용 평가를, 플랫폼팀의 Go 에이전트에 계정 프로비저닝을, 법무팀의 Python 에이전트에 컴플라이언스 문서화를, 마케팅팀의 TypeScript 에이전트에 환영 커뮤니케이션을 위임한다.

각 전문가는 다른 팀이 소유하고 독립적으로 유지보수되며 자체 일정으로 배포된다.
보안팀이 새 문서 타입을 지원하도록 검증 에이전트를 업데이트해도 조정자는 변경할 필요가 없다.
이는 마이크로서비스를 성공시킨 원칙과 동일하다 - 독립적 배포, 독립적 스케일링, 독립적 진화.

### Pattern 3: Tool Bridge (MCP)

A2A가 에이전트끼리 연결한다면, MCP는 에이전트를 도구와 데이터에 연결한다.
Tool Bridge 패턴은 MCP를 사용해 에이전트가 데이터 소스, API, 엔터프라이즈 시스템에 안전하게 접근하도록 한다.

MCP가 없으면 모든 데이터 연결마다 커스텀 통합 코드가 필요하다.
REST API용 Python 커넥터, 다른 데이터베이스용 다른 커넥터, 레거시 시스템용 또 다른 커넥터.
세 데이터 소스는 세 개의 커넥터를 의미한다.

MCP는 모든 도구가 구현할 수 있는 단일 프로토콜로 이를 제거한다.
ADK 통합 생태계는 GitHub, Notion, Hugging Face, AgentOps, Stripe 등 60개 이상의 즉시 사용 가능한 도구를 제공한다.

데이터베이스 연결에 한정하면 MCP Toolbox for Databases가 30개 이상의 데이터 소스를 단일 MCP 인터페이스로 연결한다.
Apigee API Hub는 한 발 더 나간다 - Apigee에 문서화된 기존 REST API를 자동으로 에이전트가 접근 가능한 도구로 전환한다.
기존 API 트래픽을 관리하던 거버넌스 레이어가 그대로 적용되어 레이트 리밋, 인증, 로깅, 접근 제어를 인프라가 담당한다.

| 통합 대상 | 도구/플랫폼 | 비고 |
|----------|------------|------|
| SaaS 도구 | GitHub, Notion, Stripe 등 60개+ | ADK 즉시 사용 |
| 데이터베이스 | MCP Toolbox for Databases | 30개+ 데이터 소스 |
| 기존 REST API | Apigee API Hub | 자동 도구화 |

### Pattern 4: Cross-Organization Federation

가장 야심찬 패턴은 서로 다른 조직의 에이전트가 공유 작업을 협력하는 것이다.
A2A의 오픈 프로토콜 설계가 본질적으로 필요한 영역이다.
상대 조직이 같은 프레임워크, 같은 언어, 같은 클라우드를 쓴다고 가정할 수 없기 때문이다.

Gemini Enterprise의 Agent Gallery가 이를 실용화한다.
Adobe, ServiceNow, Workday, Salesforce 등에서 검증된 100개 이상의 파트너 에이전트가 플랫폼 안에서 직접 사용 가능하다.
모든 파트너 에이전트는 Google Cloud에서 보안과 상호운용성을 검증한다.

핵심 속성은 각 조직이 자체 거버넌스를 유지한다는 점이다.
자기 조직의 Agent Gateway 정책이 자기 에이전트가 파트너 에이전트와 공유할 데이터, 파트너 응답에 따라 취할 수 있는 행동, 요청 가능한 정보를 제어한다.
파트너 에이전트는 자체 보안 모델 아래에서 동작하며 Google Cloud의 통합 요구사항으로 검증된다.

자기 에이전트는 Salesforce의 데이터 모델이나 ServiceNow의 내부 아키텍처를 이해할 필요가 없다.
A2A 프로토콜로 통신하면 양쪽이 각자의 보안 경계를 독립적으로 시행한다.

### Pattern 5: Ambient Event Mesh

마지막 패턴은 A2A를 이벤트 주도 아키텍처와 결합해 백그라운드에서 지속적으로 이벤트에 반응하는 ambient 에이전트 메시를 만든다.
Gemini Enterprise Agent Platform의 Batch & Event-Driven Agents가 BigQuery 테이블과 Pub/Sub 스트림에 직접 연결된다.
이를 A2A와 결합하면 이벤트를 듣고, 처리하고, 필요시 전문가 에이전트에 위임하는 에이전트 네트워크가 백그라운드에서 동작한다.

메시의 각 에이전트는 독립적으로 실행된다.
이벤트가 도착하면 수신 에이전트는 로컬에서 처리할지, A2A로 전문가에 위임할지, Mission Control을 통해 사람에게 에스컬레이션할지를 결정한다.
메시는 자기 조직화된다 - 새 전문가 에이전트(예: 사기 탐지) 추가는 Agent Registry에 등록하고 관련 ambient 에이전트의 라우팅 로직만 업데이트하면 된다.

이 패턴은 이벤트 볼륨이 큰 조직에 특히 강력하다.
하루 수백만 건의 업로드를 처리하는 콘텐츠 플랫폼, 수백만 건의 거래를 처리하는 금융 기관, 수백만 건의 고객 상호작용을 처리하는 이커머스 플랫폼이 모두 ambient 에이전트 메시로 지능적 처리가 필요한 롱테일 이벤트를 다룰 수 있다.

거버넌스가 결정적이다.
메시 안의 모든 에이전트는 Agent Identity로 자체 정체성을 갖고, 모든 도구 접근은 Agent Gateway로 통제되며, 모든 상호작용은 Agent Observability로 트레이스된다.
메시는 블랙박스가 아니라 완전히 관찰 가능하고 거버넌스되는 전문 에이전트 네트워크다.

## 의미와 시사점

다섯 패턴은 단순한 사용 사례 나열이 아니라 관계 기반의 분류다.
같은 팀 안에서는 위임 패턴이 충분하지만, 팀 경계를 넘으면 Agent Card Discovery가 전제 조건이 된다.
조직 경계를 넘으면 Agent Gateway 정책 분리가 필수다.

A2A와 MCP의 역할 분담도 명확해졌다.
A2A는 에이전트가 능동적으로 다른 에이전트에 작업을 넘길 때, MCP는 에이전트가 외부 도구나 데이터에 닿을 때 사용된다.
두 프로토콜이 직교(orthogonal)하므로 동일 시스템에서 함께 작동한다.

마지막 패턴인 Ambient Event Mesh는 사용자 요청을 기다리지 않는 백그라운드 에이전트라는 새로운 운영 모델을 제시한다.
Pub/Sub과 BigQuery에 직접 연결되는 이벤트 주도 에이전트가 표준화되면, 이커머스나 금융처럼 이벤트 볼륨이 큰 도메인의 운영 비용 구조가 달라진다.
사람이 모든 이벤트를 처리하지 않고 전문가 에이전트가 자기 자리에서 자기 조각만 처리한다.

## 결론

A2A와 MCP는 각자의 역할이 다르며, 다섯 패턴은 그 조합을 관계 형태별로 정리한 지도다.
Agent Card Discovery, Delegated Specialization, Tool Bridge, Cross-Organization Federation, Ambient Event Mesh의 다섯 가지를 출발점으로 잡고 시작하면 된다.
멀티 에이전트 시스템을 설계하는 팀이라면 어느 경계 위에 서 있는지를 먼저 확인하고 그에 맞는 패턴을 채택하는 것이 합리적이다.

## Reference

- [Google for Startups AI Agents Challenge](https://cloud.google.com/products/gemini-enterprise-agent-platform)
- [InstaVibe ADK Multi-Agents Codelab](https://codelabs.developers.google.com/instavibe-adk-multi-agents)
- [ADK Samples GitHub](https://github.com/google/adk-samples)
- [Agent Development Kit](https://adk.dev)
