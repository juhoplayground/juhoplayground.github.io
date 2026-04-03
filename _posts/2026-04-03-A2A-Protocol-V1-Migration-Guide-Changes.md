---
layout: post
title: "A2A Protocol v1.0 마이그레이션 가이드 - 주요 변경사항과 전환 전략"
author: 'Juho'
date: 2026-04-03 00:00:00 +0900
categories: [A2A]
tags: [A2A, Agent, Migration]
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
2. [네이밍 변경사항](#네이밍-변경사항)
   - [Enum 컨벤션 통일](#enum-컨벤션-통일)
   - [오퍼레이션 네이밍 변경](#오퍼레이션-네이밍-변경)
3. [구조적 변경사항](#구조적-변경사항)
   - [Part 객체 통합](#part-객체-통합)
   - [스트리밍 이벤트 래퍼 변경](#스트리밍-이벤트-래퍼-변경)
   - [에러 응답 포맷 변경](#에러-응답-포맷-변경)
4. [설계 변경사항](#설계-변경사항)
   - [Proto 파일 정규 스펙 격상](#proto-파일-정규-스펙-격상)
   - [AgentCard 재설계](#agentcard-재설계)
   - [멀티 바인딩 지원](#멀티-바인딩-지원)
5. [마이그레이션 우선순위와 전략](#마이그레이션-우선순위와-전략)
   - [마이그레이션 난이도 분류](#마이그레이션-난이도-분류)
   - [점진적 전환 전략](#점진적-전환-전략)
   - [마이그레이션 체크리스트](#마이그레이션-체크리스트)
6. [실전 권장사항](#실전-권장사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Google의 A2A(Agent-to-Agent) Protocol이 v1.0에 도달하면서 프로토콜 전반이 대폭 재설계되었다.
v0.3에서 v1.0으로의 전환은 단순한 버전 업그레이드가 아니라 네이밍, 구조, 설계 철학까지 변경된 대규모 마이그레이션이다.
이 가이드에서는 [neocode24의 마이그레이션 가이드](https://blog.neocode24.com/blog/a2a-protocol-v1-spec-migration/){:target="_blank"}를 바탕으로 변경사항을 분류하고 실전 전환 전략을 정리한다.

변경사항은 크게 세 가지로 분류할 수 있다.
첫째, 검색/치환으로 기계적으로 처리 가능한 네이밍 변경이다.
둘째, 파싱 로직을 다시 작성해야 하는 구조적 변경이다.
셋째, 아키텍처 수준의 검토가 필요한 설계 변경이다.

## 네이밍 변경사항

### Enum 컨벤션 통일

v1.0에서 모든 Enum 값이 타입 접두사를 포함한 `SCREAMING_SNAKE_CASE`로 통일되었다.
이 변경은 기계적으로 검색/치환이 가능하므로 가장 쉽게 적용할 수 있다.

```
# v0.3
"submitted", "completed", "user"

# v1.0
"TASK_STATE_SUBMITTED", "TASK_STATE_COMPLETED", "ROLE_USER"
```

타입 접두사가 추가됨으로써 Enum 값만 보고도 어떤 타입에 속하는지 즉시 파악할 수 있게 되었다.
JSON 직렬화 시에도 이 포맷을 그대로 사용해야 한다.

### 오퍼레이션 네이밍 변경

API 오퍼레이션 이름이 슬래시 기반에서 PascalCase로 변경되었다.

| v0.3 | v1.0 | 비고 |
|------|------|------|
| message/send | SendMessage | Async/sync 옵션 |
| message/stream | SendStreamingMessage | SSE 기반 |
| tasks/get | GetTask | - |
| tasks/cancel | CancelTask | - |
| - | ListTasks | 신규, 커서 기반 페이지네이션 |

`ListTasks`는 v1.0에서 새로 추가된 오퍼레이션으로 커서 기반 페이지네이션을 지원한다.

## 구조적 변경사항

### Part 객체 통합

v0.3에서는 `TextPart`, `FilePart`, `DataPart`가 각각 별도 타입으로 존재하고 `kind` 필드로 구분했다.
v1.0에서는 단일 `Part` 구조체로 통합되며 필드 존재 여부로 타입을 판별하는 방식으로 변경되었다.

v0.3 방식:

```json
{% raw %}{"kind": "text", "text": "Hello"}{% endraw %}
{% raw %}{"kind": "file", "file": {"url": "https://...", "mimeType": "image/png"}}{% endraw %}
```

v1.0 방식:

```json
{% raw %}{"text": "Hello"}{% endraw %}
{% raw %}{"url": "https://...", "mediaType": "image/png"}{% endraw %}
{% raw %}{"data": {"key": "value"}}{% endraw %}
```

핵심 변경 포인트는 다음과 같다.
- `kind` 필드가 완전히 제거되었다.
- `file` 중첩 객체가 평탄화되어 `url`, `mediaType` 등이 최상위로 올라왔다.
- `mimeType`이 `mediaType`으로 이름이 변경되었다.

이 변경으로 인해 기존에 `kind` 필드를 기준으로 분기하던 파싱 로직을 필드 존재 여부 체크 방식으로 전면 재작성해야 한다.

### 스트리밍 이벤트 래퍼 변경

스트리밍 이벤트의 구분 방식이 `kind` 문자열에서 래퍼 객체 키 방식으로 변경되었다.
또한 스트림 종료를 알리던 `final` 필드가 제거되었다.

v0.3 방식:

```json
{% raw %}{"kind": "status-update", "taskId": "...", "state": "completed"}{% endraw %}
```

v1.0 방식:

```json
{% raw %}{"taskStatusUpdate": {"taskId": "...", "state": "TASK_STATE_COMPLETED"}}{% endraw %}
```

v1.0에서는 스트림 연결 종료 자체가 완료 신호이므로 별도의 `final` 필드가 필요하지 않다.
이벤트 타입 판별 시 `kind` 문자열 비교 대신 `taskStatusUpdate` 등의 래퍼 객체 키 존재 여부를 확인해야 한다.

### 에러 응답 포맷 변경

에러 응답이 RFC 9457 Problem Details 형식에서 Google의 `google.rpc.Status` 형식으로 변경되었다.

v0.3 방식:

```json
{% raw %}{"type": "about:blank", "title": "Unauthorized", "status": 401}{% endraw %}
```

v1.0 방식:

```json
{% raw %}{"code": 16, "message": "Unauthorized", "details": [...]}{% endraw %}
```

HTTP 상태 코드 대신 gRPC 상태 코드 체계를 사용하며, `details` 배열을 통해 구조화된 에러 정보를 전달할 수 있다.

## 설계 변경사항

### Proto 파일 정규 스펙 격상

v1.0에서는 `a2a.proto` 파일이 JSON Schema를 대체하여 정규(canonical) 스펙으로 격상되었다.
이를 통해 자동 코드 생성의 이점을 얻을 수 있지만, Protobuf 툴체인 의존성과 더 엄격한 스키마 진화 규칙이 따른다.

Proto 채택이 정당화되는 경우는 다음과 같다.
- 다중 언어 팀에서 일관된 타입 생성이 필요한 경우
- 외부 파트너와의 통합에서 명확한 계약이 필요한 경우

반면 소규모 단일 언어 배포에서는 JSON 스키마만으로도 충분하다.

### AgentCard 재설계

AgentCard의 구조가 단일 URL 방식에서 `supportedInterfaces[]` 배열 방식으로 재설계되었다.
이를 통해 하나의 에이전트가 여러 프로토콜과 버전을 동시에 지원할 수 있게 되었다.

v0.3 방식:

```json
{
  "url": "https://...",
  "protocolVersion": "0.3.0",
  "preferredTransport": "JSONRPC"
}
```

v1.0 방식:

```json
{
  "supportedInterfaces": [
    {
      "url": "https://...",
      "protocolBinding": "jsonrpc+http",
      "protocolVersion": "1.0"
    },
    {
      "url": "https://...",
      "protocolBinding": "grpc",
      "protocolVersion": "1.0"
    }
  ]
}
```

이 구조 변경은 점진적 마이그레이션의 핵심 메커니즘이 된다.
v0.3 인터페이스와 v1.0 인터페이스를 동시에 광고할 수 있기 때문이다.

### 멀티 바인딩 지원

v1.0에서 세 가지 프로토콜 바인딩이 공식 지원된다.

| 바인딩 | 트랜스포트 | 스트리밍 | 적합한 사용 사례 |
|--------|-----------|---------|----------------|
| JSON+HTTP | HTTP POST | SSE | 웹, 범용 |
| gRPC | HTTP/2 | gRPC stream | 마이크로서비스, 고성능 |
| JSON-RPC | HTTP POST | 폴링/웹훅 | 레거시 호환 |

기본적으로 단일 바인딩으로 시작하는 것이 권장되며, JSON+HTTP가 대부분의 시나리오에 적합하다.

## 마이그레이션 우선순위와 전략

### 마이그레이션 난이도 분류

| 난이도 | 작업 항목 | 설명 |
|--------|----------|------|
| 높음 | Part 파싱 로직 전환 | kind 기반에서 필드 존재 기반으로 전면 재작성 필요 |
| 중간 | AgentCard/에러 응답 포맷 | 구조 변경에 따른 직렬화/역직렬화 수정 |
| 낮음 | Enum 상수 업데이트 | 검색/치환으로 기계적 처리 가능 |

### 점진적 전환 전략

AgentCard의 `supportedInterfaces[]` 배열을 활용하면 v0.3과 v1.0 인터페이스를 동시에 광고할 수 있다.
클라이언트는 `A2A-Version` 헤더를 통해 원하는 프로토콜 버전을 선택한다.

```json
{
  "supportedInterfaces": [
    {
      "url": "https://agent.example.com/v0",
      "protocolBinding": "jsonrpc+http",
      "protocolVersion": "0.3.0"
    },
    {
      "url": "https://agent.example.com/v1",
      "protocolBinding": "jsonrpc+http",
      "protocolVersion": "1.0"
    }
  ]
}
```

이 방식을 통해 기존 클라이언트의 호환성을 유지하면서 새로운 클라이언트부터 v1.0으로 전환할 수 있다.
권장되는 듀얼 버전 유지 기간은 최대 6개월이며, 명시적인 서비스 종료 일자를 설정해야 한다.

### 마이그레이션 체크리스트

실제 전환 시 다음 순서로 진행하는 것이 효과적이다.

1. SDK 업그레이드: `pip install -U "a2a-sdk>=1.0.0"` 적용 후 정적 타입 체커(`mypy`)로 브레이킹 체인지를 식별한다.
2. Part 파싱 로직 전환: `kind` 필드 조건 분기를 필드 존재 여부 체크로 변경하고, `file` 중첩 객체를 평탄화한다.
3. Enum 상수 업데이트: 모든 Enum 참조를 `SCREAMING_SNAKE_CASE` 포맷으로 갱신한다.
4. AgentCard 구조 변경: `url`, `protocolBinding`, `protocolVersion`을 `supportedInterfaces[]` 배열로 래핑한다.
5. 에러 응답 변환: RFC 9457 파싱을 `google.rpc.Status` 구조로 전환한다.
6. 버전 협상 설정(선택): 양쪽 버전을 동시에 광고하고 `A2A-Version` 헤더로 프로토콜 협상을 구현한다.

## 실전 권장사항

JSON+HTTP 단일 바인딩으로 시작하는 것을 권장한다.
대부분의 사용 사례에서 이것만으로 충분하다.

멀티 바인딩은 다음 경우에만 도입을 검토한다.
- 내부 마이크로서비스 간 고빈도 통신이 필요하여 gRPC가 정당화되는 경우
- 레거시 시스템과의 호환성을 위해 JSON-RPC가 필요한 경우

Multi-Tenancy 지원(`tenant` 필드)은 SaaS 형태로 에이전트를 제공하는 경우에만 적용한다.
AgentCard JWS 서명 검증은 신뢰할 수 없는 외부 파트너와 협업하는 경우에만 구현하면 된다.

인증 관련해서는 `ImplicitOAuthFlow`와 `PasswordOAuthFlow`가 폐기(deprecated)되었다.
브라우저가 없는 환경을 위해 RFC 8628 기반의 `DeviceCodeOAuthFlow`가 새로 추가되었으며, AgentCard에 `pkce_required` 필드도 도입되었다.

프로토콜 설계는 Stateless, Layered Architecture 패턴을 따르므로 기존 인프라(로드 밸런서, API 게이트웨이, 관측 도구)를 수정 없이 그대로 재사용할 수 있다.

## 결론

A2A Protocol v1.0은 v0.3 대비 네이밍, 구조, 설계 전반에 걸쳐 대폭 변경되었다.
특히 Part 파싱 로직 전환이 가장 높은 난이도를 가지므로 우선적으로 착수해야 한다.

점진적 전환을 위해 AgentCard의 `supportedInterfaces[]`를 활용하여 양쪽 버전을 동시에 지원하는 전략이 효과적이다.
JSON+HTTP 단일 바인딩으로 시작하고, 실제 필요가 발생할 때만 멀티 바인딩이나 Multi-Tenancy를 도입하는 것이 현실적인 접근법이다.

canary 배포와 함께 최대 6개월의 듀얼 버전 유지 기간을 설정하고, 명확한 서비스 종료 일자를 공지하여 클라이언트의 전환을 유도하는 것을 권장한다.

## Reference

- [A2A Protocol v1.0 스펙 마이그레이션 가이드](https://blog.neocode24.com/blog/a2a-protocol-v1-spec-migration/)
