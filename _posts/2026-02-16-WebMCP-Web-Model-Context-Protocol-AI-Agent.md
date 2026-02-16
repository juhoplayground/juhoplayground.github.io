---
layout: post
title: "WebMCP : 웹사이트가 AI 에이전트에게 도구를 제공하는 새로운 웹 표준"
author: 'Juho'
date: 2026-02-16 00:00:00 +0900
categories: [MCP]
tags: [MCP, AI]
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
2. [WebMCP란 무엇인가](#webmcp란-무엇인가)
3. [기존 방식의 문제점](#기존-방식의-문제점)
4. [WebMCP의 두 가지 API](#webmcp의-두-가지-api)
5. [핵심 API 상세](#핵심-api-상세)
6. [실전 예제 : 우표 데이터베이스](#실전-예제--우표-데이터베이스)
7. [사용자 상호작용 요청](#사용자-상호작용-요청)
8. [MCP와의 관계](#mcp와의-관계)
9. [사용 사례](#사용-사례)
10. [설계의 장점과 한계](#설계의-장점과-한계)
11. [현재 상태 및 향후 방향](#현재-상태-및-향후-방향)
12. [Reference](#reference)

## 개요

Google과 Microsoft가 공동으로 제안한 WebMCP(Web Model Context Protocol)가 사전 체험판(Early Preview)으로 공개되었다.
WebMCP는 웹사이트가 AI 에이전트에게 구조화된 도구(tool)를 직접 노출할 수 있게 해주는 새로운 웹 표준이다.
기존에 AI 에이전트가 웹사이트와 상호작용하려면 스크린샷 분석이나 DOM 추론에 의존해야 했지만, WebMCP를 통해 웹사이트가 자신의 기능을 명시적으로 선언할 수 있게 된다.
Chrome 146+ 버전에서 프로토타이핑이 가능하며, W3C Web Machine Learning Working Group에서 표준화가 진행 중이다.

## WebMCP란 무엇인가

WebMCP는 웹 개발자가 자신의 웹 애플리케이션 기능을 "도구(tool)"로 노출할 수 있게 해주는 JavaScript API이다.
여기서 도구란 자연어 설명과 구조화된 스키마를 갖춘 JavaScript 함수로, AI 에이전트, 브라우저 어시스턴트, 보조 기술이 호출할 수 있다.

도구를 정의함으로써 에이전트에게 사이트와 상호작용하는 방법과 위치를 알려줄 수 있다.
항공편 예약, 지원 티켓 제출, 복잡한 데이터 탐색 등 다양한 작업에 활용된다.
이 직접 통신 채널은 모호성을 제거하고 더 빠르고 견고한 에이전트 워크플로를 가능하게 한다.

### 핵심 계약 구조

WebMCP는 세 부분으로 구성된 계약(contract)을 통해 작동한다.

| 구성 요소 | 설명 |
|-----------|------|
| Discovery | 페이지에서 사용 가능한 도구 목록 |
| JSON Schema | 명확한 입력/출력 사양 |
| State | 현재 페이지 컨텍스트 공유 |

## 기존 방식의 문제점

현재 AI 에이전트가 웹사이트와 상호작용하는 방식에는 여러 문제가 있다.

스크린샷 기반 분석은 버튼의 목적이나 폼의 구조를 추측해야 한다.
DOM 기반 자동화는 UI가 변경되면 쉽게 깨진다.
이러한 방식은 토큰 사용량이 많고 환각(hallucination) 발생률이 높다.

WebMCP를 사용하면 스크린샷/DOM 분석 방식 대비 약 10% 수준의 토큰만 사용한다.
또한 UI 변경에도 안정적으로 동작하며, 환각 현상이 크게 감소한다.

## WebMCP의 두 가지 API

WebMCP는 브라우저 에이전트가 사용자를 대신하여 작업을 수행할 수 있도록 두 가지 API를 제안한다.

### 선언적 API (Declarative API)

HTML 폼에서 직접 정의할 수 있는 표준 작업을 위한 API이다.
기존 HTML 폼에 속성을 추가하는 간단한 방식으로, 단순한 UI 패턴에 적합하다.

### 명령형 API (Imperative API)

JavaScript 실행이 필요한 복잡하고 동적인 상호작용을 위한 API이다.
비동기 작업이나 복잡한 로직이 필요한 경우에 사용한다.

## 핵심 API 상세

### modelContext 인터페이스

`window.navigator.modelContext` 인터페이스를 통해 사이트가 AI 에이전트에게 기능을 선언한다.
브라우저가 접근을 중재하며, `provideContext` 메서드로 사용 가능한 도구를 업데이트한다.

```javascript
window.navigator.modelContext.provideContext({
  tools: [
    {
      name: "add-todo",
      description: "Add a new todo item to the list",
      inputSchema: {
        type: "object",
        properties: {
          text: { type: "string", description: "The text of the todo item" }
        },
        required: ["text"]
      },
      execute: ({ text }, agent) => {
        // Todo 항목을 추가하고 UI를 업데이트한다.
        return { content: [{ type: "text", text: "Todo added!" }] };
      }
    }
  ]
});
```

`provideContext`는 여러 번 호출할 수 있으며, 이후 호출 시 기존 도구가 초기화된 후 새로운 도구가 등록된다.
이는 SPA(Single Page Application)에서 UI 상태에 따라 다른 도구를 제공할 때 유용하다.

### registerTool / unregisterTool

상태를 초기화하지 않고 도구를 개별적으로 추가하거나 제거할 수도 있다.

```javascript
window.navigator.modelContext.registerTool({
  name: "add-todo",
  description: "Add a new todo item to the list",
  inputSchema: {
    type: "object",
    properties: {
      text: { type: "string", description: "The text of the todo item" }
    },
    required: ["text"]
  },
  execute: ({ text }, agent) => {
    // Todo 항목을 추가하고 UI를 업데이트한다.
    return { content: [{ type: "text", text: "Todo added!" }] };
  }
});

// 도구를 제거할 때
window.navigator.modelContext.unregisterTool("add-todo");
```

### 도구 호출의 특성

도구 호출은 순차적으로 실행되며, 한 번에 하나씩 처리된다.
각 도구 호출 후 UI가 상태 변경을 반영할 수 있다.
간단한 애플리케이션은 페이지 스크립트에서 직접 처리하고, 복잡한 경우에는 Worker에 위임할 수 있다.

## 실전 예제 : 우표 데이터베이스

WebMCP 명세에 포함된 우표 데이터베이스 예제를 통해 실제 적용 방법을 살펴본다.

### 기존 코드

기존에 HTML 폼의 submit 핸들러로 우표를 추가하는 코드가 있다.

```javascript
document.getElementById('addStampForm').addEventListener('submit', (event) => {
  event.preventDefault();

  const stampName = document.getElementById('stampName').value;
  const stampDescription = document.getElementById('stampDescription').value;
  const stampYear = document.getElementById('stampYear').value;
  const stampImageUrl = document.getElementById('stampImageUrl').value;

  addStamp(stampName, stampDescription, stampYear, stampImageUrl);
});
```

핵심 로직은 `addStamp` 헬퍼 함수로 분리되어 있다.

```javascript
function addStamp(stampName, stampDescription, stampYear, stampImageUrl) {
  stamps.push({
    name: stampName,
    description: stampDescription,
    year: stampYear,
    imageUrl: stampImageUrl || null
  });

  document.getElementById('confirmationMessage').textContent =
    `Stamp "${stampName}" added successfully!`;
  renderStamps();
}
```

### WebMCP 도구 추가

기존 헬퍼 함수를 재사용하면서 WebMCP 도구로 등록하는 코드를 추가한다.

```javascript
if ("modelContext" in window.navigator) {
  window.navigator.modelContext.provideContext({
    tools: [
      {
        name: "add-stamp",
        description: "Add a new stamp to the collection",
        inputSchema: {
          type: "object",
          properties: {
            name: { type: "string", description: "The name of the stamp" },
            description: {
              type: "string",
              description: "A brief description of the stamp"
            },
            year: {
              type: "number",
              description: "The year the stamp was issued"
            },
            imageUrl: {
              type: "string",
              description: "An optional image URL for the stamp"
            }
          },
          required: ["name", "description", "year"]
        },
        execute({ name, description, year, imageUrl }, agent) {
          addStamp(name, description, year, imageUrl);

          return {
            content: [
              {
                type: "text",
                text: `Stamp "${name}" added successfully! ` +
                      `The collection now contains ${stamps.length} stamps.`
              }
            ]
          };
        }
      }
    ]
  });
}
```

기존 JavaScript 코드를 최소한으로 변경하면서 AI 에이전트가 사용할 수 있는 도구를 등록할 수 있다.
`modelContext` 지원 여부를 먼저 확인하는 점진적 향상(progressive enhancement) 패턴을 사용한다.

## 사용자 상호작용 요청

`agent` 인터페이스는 도구 실행 중 사용자 입력을 비동기적으로 요청할 수 있는 `requestUserInteraction` API를 제공한다.
이를 통해 구매 확인 같은 중요한 작업에서 사용자 동의를 받을 수 있다.

```javascript
async function buyProduct({ product_id }, agent) {
  const confirmed = await agent.requestUserInteraction(async () => {
    return new Promise((resolve) => {
      const confirmed = confirm(
        `Buy product ${product_id}?\nClick OK to confirm, Cancel to abort.`
      );
      resolve(confirmed);
    });
  });

  if (!confirmed) {
    throw new Error("Purchase cancelled by user.");
  }

  executePurchase(product_id);
  return `Product ${product_id} purchased.`;
}
```

이 API는 도구 실행 중 여러 번 호출할 수 있으며, 에이전트의 수명은 도구 실행 범위로 제한된다.
이를 통해 인간이 개입하는(human-in-the-loop) 워크플로를 구현할 수 있다.

## MCP와의 관계

WebMCP와 기존 MCP(Model Context Protocol)는 보완적인 관계이다.

| 비교 항목 | MCP | WebMCP |
|-----------|-----|--------|
| 실행 환경 | 서버 사이드 | 클라이언트 사이드 (브라우저) |
| 통신 방식 | JSON-RPC (stdio, SSE) | JavaScript API |
| 인증 | 별도 인증 필요 | 브라우저 세션 활용 |
| UI 연동 | 없음 | 직접 연동 가능 |
| 사용 시나리오 | 백엔드 통합 | 프론트엔드 에이전트 상호작용 |

MCP는 서버 측에서 도구를 제공하는 프로토콜이고, WebMCP는 브라우저 내에서 동작하는 클라이언트 측 에이전트를 위한 것이다.
WebMCP는 MCP를 대체하는 것이 아니라 웹 프론트엔드 영역으로 확장하는 표준이다.

## 사용 사례

### 고객 지원

에이전트가 자동으로 기술 세부 정보를 입력하고 지원 티켓을 제출할 수 있다.
사용자가 상황을 설명하면 에이전트가 적절한 폼 필드를 채워 넣는다.

### 전자상거래

사용자의 자연어 설명과 개인 선호도에 따라 제품을 필터링하고 선택할 수 있다.
에이전트가 제품 검색, 옵션 구성, 결제 처리를 수행한다.

### 코드 리뷰

개발자가 에이전트를 활용하여 테스트 실패를 분석하고, 특수 UI 기능에 접근하며, 수정 제안을 생성할 수 있다.

### 창작 작업

사용자가 에이전트와 협업하여 디자인 작업을 수행하고, 템플릿을 필터링하며, 디자인을 편집할 수 있다.
시각적 피드백과 제어권을 유지하면서 작업한다.

## 설계의 장점과 한계

### 장점

| 항목 | 설명 |
|------|------|
| 익숙한 언어 | 웹 개발자가 JavaScript로 도구를 구현 |
| 코드 재사용 | 기존 JavaScript 함수를 최소 변경으로 도구로 노출 |
| 로컬 실행 | 별도 서버나 인증 없이 AI 에이전트와 통합 |
| 세밀한 권한 제어 | 브라우저가 도구 호출을 중재하고 사용자가 동의 |
| 접근성 | 접근성 트리(accessibility tree)의 대안 인터페이스 제공 |

### 한계

| 항목 | 설명 |
|------|------|
| 브라우징 컨텍스트 필요 | 브라우저 탭이나 웹뷰가 열려 있어야 함 |
| UI 동기화 | 개발자가 인간 상호작용과 도구 호출의 상태를 UI에 반영해야 함 |
| 도구 발견 불가 | 에이전트가 사이트 방문 없이 도구를 발견하는 메커니즘 없음 |
| 헤드리스 미지원 | 인간 관찰 없는 헤드리스 브라우징 시나리오 미지원 |

## 현재 상태 및 향후 방향

WebMCP는 현재 Chrome 146+ 버전에서 사전 체험판으로 제공되고 있다.
Chrome Built-in AI Early Preview Program에 등록하면 문서, 데모, 최신 변경 사항에 접근할 수 있다.

향후 탐구 방향은 다음과 같다.

**PWA 통합** : WebMCP를 Progressive Web App의 오프라인 기능 및 백그라운드 기능과 통합하는 것이 검토되고 있다.

**매니페스트 기반 도구 선언** : 페이지 렌더링 없이 HTTP GET으로 매니페스트를 가져와 도구를 발견할 수 있는 선언적 방식이 논의 중이다.

**Worker 기반 처리** : 대량의 도구 호출 시 메인 스레드를 차단하지 않도록 Worker로 처리를 위임하는 방식이 계획되어 있다.

**보안 고려사항** : 악성 도구, 모델 포이즈닝, 교차 출처 격리, 권한 프레임워크 등이 해결해야 할 과제로 남아 있다.

WebMCP는 Google과 Microsoft가 공동으로 개발하고 있으며, W3C Web Machine Learning Working Group에서 표준화가 진행 중이다.
이 표준이 자리잡으면 웹사이트가 AI 에이전트와 상호작용하는 방식에 근본적인 변화가 올 것으로 예상된다.

## Reference

- [WebMCP is available for early preview - Chrome for Developers](https://developer.chrome.com/blog/webmcp-epp)
- [WebMCP GitHub Repository](https://github.com/webmachinelearning/webmcp)
- [WebMCP Specification](https://webmachinelearning.github.io/webmcp/)
