---
layout: post
title: "AI 에이전트 프로토콜 개발자 가이드 - MCP, A2A 등 6가지 프로토콜 총정리"
author: 'Juho'
date: 2026-03-25 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, MCP, A2A]
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
   - [레스토랑 공급망 시나리오](#레스토랑-공급망-시나리오)
   - [MCP - Model Context Protocol](#mcp---model-context-protocol)
   - [A2A - Agent2Agent Protocol](#a2a---agent2agent-protocol)
   - [UCP - Universal Commerce Protocol](#ucp---universal-commerce-protocol)
   - [AP2 - Agent Payments Protocol](#ap2---agent-payments-protocol)
   - [A2UI - Agent-to-User Interface Protocol](#a2ui---agent-to-user-interface-protocol)
   - [AG-UI - Agent-User Interaction Protocol](#ag-ui---agent-user-interaction-protocol)
   - [6가지 프로토콜 요약](#6가지-프로토콜-요약)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google Developers Blog에서 2026년 3월 18일에 발행한 AI 에이전트 프로토콜 개발자 가이드를 정리한다.
이 가이드는 Shubham Saboo와 Kristopher Overholt가 작성했으며, 6가지 AI 에이전트 프로토콜을 하나의 레스토랑 공급망 에이전트 시나리오로 묶어 설명한다.
MCP, A2A, UCP, AP2, A2UI, AG-UI 총 6개 프로토콜이 각각 어떤 역할을 하며, 실제 에이전트 개발에서 어떻게 조합되는지를 다룬다.
Google의 Agent Development Kit(ADK)을 활용한 실전 구현 방식도 함께 소개한다.

## 배경

AI 에이전트가 단순히 텍스트를 생성하는 것을 넘어, 도구를 호출하고, 다른 에이전트와 협력하며, 결제를 수행하고, UI를 구성하는 등 복잡한 작업을 수행하게 되면서 표준화된 프로토콜의 필요성이 대두되었다.
기존에는 각 API마다 커스텀 통합 코드를 작성해야 했고, 에이전트 간 통신, 상거래, 결제, UI 구성 등에 통일된 표준이 없었다.
이 가이드는 이러한 문제를 해결하기 위해 등장한 6가지 프로토콜을 하나의 일관된 시나리오 안에서 체계적으로 설명한다.

## 핵심 내용

### 레스토랑 공급망 시나리오

이 가이드는 모든 프로토콜을 레스토랑의 주방 매니저 에이전트 시나리오로 설명한다.
사용자가 다음과 같은 요청을 한다고 가정한다.

"연어 재고를 확인하고, 오늘의 도매가와 품질 등급을 조회한 뒤, 재고가 부족하면 Example Wholesale에서 10파운드를 주문하고 결제를 승인해줘."

이 하나의 요청을 처리하기 위해 3단계로 나뉜다.

| 단계 | 역할 | 사용 프로토콜 |
|------|------|--------------|
| Stage 1 - Gather | MCP로 재고 DB 조회, A2A로 원격 가격/품질 에이전트 쿼리 | MCP, A2A |
| Stage 2 - Transact | UCP로 공급업체 체크아웃, AP2로 결제 승인 | UCP, AP2 |
| Stage 3 - Present | A2UI로 대시보드 구성, AG-UI로 프론트엔드 스트리밍 | A2UI, AG-UI |

### MCP - Model Context Protocol

MCP는 각 API마다 커스텀 코드를 작성하는 대신, 단일 표준 연결 패턴을 제공하는 프로토콜이다.
"eliminates busywork by giving you a single standard connection pattern for hundreds of servers"라고 설명한다.
MCP 서버가 자체적으로 도구 정의를 관리하며, 에이전트는 이를 자동으로 디스커버리하여 사용한다.
에이전트 측 코드 업데이트 없이도 새로운 도구를 연결할 수 있다.

레스토랑 시나리오에서는 주방 매니저 에이전트가 PostgreSQL을 통해 재고를 조회하고, Notion MCP로 레시피를 검색하며, Mailgun MCP로 공급업체에 이메일을 발송한다.
ADK에서는 McpToolset을 통해 MCP를 네이티브로 지원한다.

### A2A - Agent2Agent Protocol

A2A는 서로 다른 프레임워크, 팀, 서버에 있는 에이전트 간의 표준 디스커버리 및 통신을 담당한다.
각 A2A 에이전트는 {% raw %}`/.well-known/agent-card.json`{% endraw %}에 Agent Card를 발행하여 자신의 기능과 엔드포인트를 공개한다.
주방 매니저 에이전트는 이 Agent Card를 가져와 런타임에 적절한 원격 에이전트로 쿼리를 라우팅한다.

레스토랑 시나리오에서는 가격 조회, 품질 등급 확인, 배송 일정 확인 등을 각각 별도의 원격 에이전트에 동시에 쿼리한다.
수동 코드 변경이나 재배포 없이 새로운 공급업체 에이전트를 추가할 수 있다.

### UCP - Universal Commerce Protocol

UCP는 공급업체마다 다른 API를 가진 주문 프로세스를 표준화하는 프로토콜이다.
강타입 요청/응답 스키마를 통해 쇼핑 라이프사이클을 모듈식 기능으로 표준화한다.
공급업체는 {% raw %}`/.well-known/ucp`{% endraw %}에 UCP 프로필을 발행하고, 에이전트는 이를 디스커버리하여 일관된 패턴으로 주문을 처리한다.

레스토랑 시나리오에서는 주방 매니저가 여러 도매 공급업체의 카탈로그를 탐색하고, 연어 10파운드, 올리브유 3병 등을 통합 체크아웃 플로우로 주문한다.
기반 전송 방식에 관계없이 일관된 주문 패턴을 사용할 수 있다.

### AP2 - Agent Payments Protocol

AP2는 결제 인가와 감사 추적을 담당하는 프로토콜이다.
typed mandates를 통해 부인 불가능한 의도 증명을 제공하고, configurable guardrails로 모든 거래를 제어한다.

레스토랑 시나리오에서의 작동 흐름은 다음과 같다.

| 단계 | 설명 |
|------|------|
| IntentMandate 설정 | 레스토랑 매니저가 지출 한도와 승인 공급업체 목록을 설정 |
| PaymentMandate 생성 | 에이전트가 특정 장바구니와 금액(예: $294)에 바인딩된 결제 위임장 생성 |
| 가드레일 적용 | 설정된 한도를 초과하면 명시적 승인을 요구 |
| PaymentReceipt 발행 | 감사 추적을 위한 암호학적 증명이 포함된 영수증 생성 |

### A2UI - Agent-to-User Interface Protocol

A2UI는 에이전트가 18개의 안전한 컴포넌트 프리미티브로 동적 UI를 구성할 수 있게 하는 프로토콜이다.
선언적 JSON 포맷으로 평면 컴포넌트 리스트를 전송하며, 각 컴포넌트는 ID로 서로를 참조한다.
데이터 페이로드는 별도로 분리되어 있고, 클라이언트 측 렌더러가 JSON을 Lit, Flutter, Angular 등의 네이티브 UI로 변환한다.

레스토랑 시나리오에서는 재고 대시보드, 주문 양식, 공급업체 비교 화면을 동일한 18개 프리미티브로 구성한다.
추가적인 프론트엔드 코드 없이 다양한 레이아웃을 동적으로 생성할 수 있다.
ADK에서는 adk web 명령어로 개발 중에 A2UI 컴포넌트를 테스트할 수 있다.

### AG-UI - Agent-User Interaction Protocol

AG-UI는 프레임워크별 원시 이벤트를 표준화된 SSE(Server-Sent Events) 스트림으로 변환하는 미들웨어 역할을 한다.
TEXT_MESSAGE_CONTENT, TOOL_CALL_START, TOOL_CALL_RESULT 등의 타입이 지정된 이벤트를 통해 프론트엔드 보일러플레이트를 제거한다.

레스토랑 시나리오에서는 주방 매니저 에이전트의 도구 호출과 텍스트 응답을 실시간으로 프론트엔드에 스트리밍한다.
ADK 에이전트를 ag_ui_adk 패키지로 래핑하고 FastAPI 앱에 마운트하여 구현한다.

주요 SSE 이벤트 타입은 다음과 같다.

| 이벤트 | 역할 |
|--------|------|
| RUN_STARTED | 실행 시작 알림 |
| TEXT_MESSAGE_CONTENT | 점진적 텍스트 생성 |
| TOOL_CALL_START | 도구 호출 시작 |
| TOOL_CALL_RESULT | 도구 호출 결과 반환 |
| RUN_FINISHED | 실행 완료 알림 |

### 6가지 프로토콜 요약

| 프로토콜 | 역할 | 연결 대상 |
|----------|------|-----------|
| MCP | 도구와 데이터 연결 | 에이전트 - 도구/데이터 |
| A2A | 에이전트 간 통신 | 에이전트 - 에이전트 |
| UCP | 상거래 표준화 | 에이전트 - 공급업체 |
| AP2 | 결제 인가와 감사 | 에이전트 - 결제 시스템 |
| A2UI | UI 렌더링 정의 | 에이전트 - 프론트엔드 |
| AG-UI | 실시간 스트리밍 전달 | 에이전트 - 프론트엔드 |

## 의미와 시사점

이 가이드가 강조하는 핵심 메시지는 "첫날부터 6개 전부 필요하지 않다"는 점이다.
대부분의 에이전트는 MCP로 데이터 접근부터 시작하고, 요구사항이 증가함에 따라 점진적으로 프로토콜을 추가하는 것이 권장된다.

각 프로토콜의 경계가 명확하다는 점도 중요하다.
MCP는 에이전트를 도구와 데이터에 연결하고, A2A는 에이전트를 다른 에이전트에 연결하며, UCP는 상거래를 표준화하고, AP2는 결제 인가를 처리하며, A2UI는 무엇을 렌더링할지를 정의하고, AG-UI는 어떻게 스트리밍할지를 정의한다.

Google의 ADK가 이 모든 프로토콜에 대한 통합 지원을 제공한다는 점에서, 에이전트 개발의 진입 장벽이 낮아지고 있다.
공식 SDK와 샘플 코드를 먼저 확인한 후 커스텀 구현을 검토하는 것이 효율적이다.

## 결론

Google Developers Blog의 이 가이드는 AI 에이전트 개발에 필요한 6가지 프로토콜을 하나의 일관된 시나리오로 체계적으로 정리한 자료이다.
각 프로토콜이 담당하는 영역이 명확히 구분되어 있으며, 레스토랑 공급망이라는 구체적인 예시를 통해 실전 적용 방식을 보여준다.
에이전트 개발을 시작하려는 개발자에게는 MCP부터 시작하여 점진적으로 필요한 프로토콜을 추가해 나가는 전략이 가장 현실적인 접근법이다.

## Reference

- [A Developer's Guide to AI Agent Protocols](https://developers.googleblog.com/developers-guide-to-ai-agent-protocols/)
