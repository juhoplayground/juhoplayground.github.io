---
layout: post
title: "MCP 2026-07-28 스펙: 세션을 걷어낸 스테이트리스 프로토콜로의 전환"
author: 'Juho'
date: 2026-08-18 00:00:00 +0900
categories: [MCP]
tags: [MCP, Streamable HTTP, FastMCP]
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
3. [핵심 변경사항](#핵심-변경사항)
   - [스테이트리스 코어 전환](#스테이트리스-코어-전환)
   - [server/discover와 버전 협상](#serverdiscover와-버전-협상)
   - [MRTR: 다중 왕복 요청 패턴](#mrtr-다중-왕복-요청-패턴)
   - [알림 채널의 재설계](#알림-채널의-재설계)
   - [인프라 친화적 헤더와 캐싱](#인프라-친화적-헤더와-캐싱)
   - [인증과 보안 강화](#인증과-보안-강화)
   - [Tasks 확장으로의 분리](#tasks-확장으로의-분리)
4. [Deprecated 기능과 수명주기 정책](#deprecated-기능과-수명주기-정책)
5. [SDK 대응 현황](#sdk-대응-현황)
   - [Python SDK v2](#python-sdk-v2)
   - [TypeScript SDK v2](#typescript-sdk-v2)
   - [Go SDK와 C# SDK](#go-sdk와-c-sdk)
6. [사용자별 대응 전략](#사용자별-대응-전략)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Model Context Protocol이 2026년 7월 28일 새로운 사양 개정을 공식 릴리스했다.
이번 개정은 MCP 역사상 가장 큰 규모의 변경으로 평가된다.
핵심은 양방향 상태유지(stateful) 프로토콜에서 요청/응답 기반 무상태(stateless) 프로토콜로의 전환이다.
Lead Maintainer인 David Soria Parra와 Den Delimarsky가 발표를 주도했다.

이전 개정 버전인 `2025-11-25` 대비 세션 제거, 핸드셰이크 제거, 서버 주도 요청 방식 폐기 등 프로토콜의 근간에 해당하는 항목이 대거 변경되었다.
RC는 5월 21일부터 약 10주간 검증을 거쳤고, Tier 1 SDK 베타는 6월 29일에 공개되었다.
Google Cloud, AWS, Microsoft, Cloudflare, Figma 등 주요 기업이 지지를 표명했다.

## 배경

기존 MCP는 클라이언트가 `initialize`를 호출해 세션을 열고, 이후 모든 요청에 `Mcp-Session-Id` 헤더를 붙이는 구조였다.
이 구조에는 운영상 문제가 있었다.
세션이 특정 서버 인스턴스에 묶이기 때문에 로드밸런서 뒤에서 스티키 세션이 필수였고, 장애 복구가 까다로웠다.
서버 개발자는 세션 수명주기를 직접 관리해야 했으며, 리스트 응답이 커넥션마다 달라질 수 있어 캐싱도 어려웠다.

또한 서버가 클라이언트에게 요청을 보내는 기능(`roots/list`, `sampling/createMessage`, `elicitation/create`)은 양방향 스트림을 전제로 했다.
이는 게이트웨이·WAF·서버리스 환경과 잘 맞지 않았다.
2026-07-28 개정은 이 두 가지 구조적 제약을 동시에 제거하는 방향으로 설계되었다.

## 핵심 변경사항

### 스테이트리스 코어 전환

`initialize` / `notifications/initialized` 핸드셰이크가 제거되었다.
프로토콜 수준 세션과 Streamable HTTP 전송의 `Mcp-Session-Id` 헤더도 함께 사라졌다.
이제 모든 요청이 자체적으로 프로토콜 버전과 클라이언트 기능을 `_meta` 필드에 담아 전달한다.

사용되는 `_meta` 키는 다음과 같다.

| 키 | 방향 | 용도 |
|------|------|------|
| io.modelcontextprotocol/protocolVersion | 클라이언트 to 서버 | 요청의 프로토콜 버전 |
| io.modelcontextprotocol/clientCapabilities | 클라이언트 to 서버 | 클라이언트 기능 선언 |
| io.modelcontextprotocol/clientInfo | 클라이언트 to 서버 | 클라이언트 신원(SHOULD) |
| io.modelcontextprotocol/serverInfo | 서버 to 클라이언트 | 결과의 서버 신원(SHOULD) |
| io.modelcontextprotocol/logLevel | 클라이언트 to 서버 | 요청별 로그 레벨 |

버전이 맞지 않으면 서버는 `UnsupportedProtocolVersionError`를 반환한다.

실제 요청은 다음과 같은 형태가 된다.

```http
POST /mcp HTTP/1.1
MCP-Protocol-Version: 2026-07-28
Mcp-Method: tools/call
Mcp-Name: search

{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"search","arguments":{"q":"otters"},
 "_meta":{"io.modelcontextprotocol/clientInfo":
 {"name":"my-app","version":"1.0"}}}}
```

리스트 엔드포인트(`tools/list`, `resources/list`, `prompts/list`)는 더 이상 커넥션별로 달라지지 않는다.
호출 간 상태가 필요한 서버는 서버가 발급한 명시적 핸들(handle)을 일반 도구 인자로 주고받는 방식을 사용한다.
이 변경은 SEP-2567과 SEP-2575에서 다뤄졌다.

또한 `ping`, `logging/setLevel`, `notifications/roots/list_changed`가 제거되었다.
로그 레벨은 요청별로 `_meta`의 `io.modelcontextprotocol/logLevel`로 지정하며, 이 필드가 없는 요청에 대해 서버는 `notifications/message`를 방출해서는 안 된다.

Streamable HTTP에서 SSE 스트림 재개(resumability)와 메시지 재전송도 제거되었다.
`Last-Event-ID` 헤더와 SSE 이벤트 ID가 사라졌으므로, 응답 스트림이 끊기면 진행 중이던 요청은 유실된다.
클라이언트는 새 요청 ID로 요청을 다시 발행해야 한다.

### server/discover와 버전 협상

핸드셰이크가 사라진 대신 `server/discover` RPC가 추가되었다.
서버는 이 메서드를 반드시 구현해야 하며, 지원하는 프로토콜 버전·기능·신원을 광고한다.
클라이언트는 다른 요청 이전에 선택적으로 호출해 버전을 미리 선택할 수 있고, STDIO에서는 하위 호환성 프로브로도 사용할 수 있다.

### MRTR: 다중 왕복 요청 패턴

서버가 클라이언트에게 요청을 보내던 방식이 MRTR(Multi Round-Trip Requests) 패턴으로 대체되었다.
도구 실행 중 사용자 확인이나 추가 정보가 필요할 때, 서버는 스트림으로 역방향 요청을 보내는 대신 중간 결과를 반환한다.

동작 순서는 다음과 같다.

1. 클라이언트가 도구를 호출한다.
2. 서버가 `resultType: "input_required"`인 `InputRequiredResult`를 반환한다. 필요한 정보 요청은 `inputRequests` 필드에 담기고, 서버는 `requestState`에 자체 식별자를 인코딩할 수 있다.
3. 클라이언트가 응답을 `inputResponses`에 담고 `requestState`를 되돌려주며 원 요청을 재시도한다.
4. 서버가 `requestState`를 검증한 뒤 처리를 이어간다.

이에 맞춰 모든 결과에 `resultType` 필드가 필수로 추가되었다.
일반 결과는 `"complete"`, MRTR 중간 결과는 `"input_required"`다.
이전 프로토콜 서버가 이 필드를 생략한 경우, 클라이언트는 반드시 `"complete"`로 취급해야 한다.

MRTR 도입으로 `notifications/elicitation/complete` 알림과 URL 모드 엘리시테이션의 `elicitationId` 필드는 제거되었다.
클라이언트가 원 요청을 재시도하면서 결과를 확인하므로, 서버가 완료 신호를 보낼 이유가 없어졌기 때문이다.

### 알림 채널의 재설계

HTTP GET 엔드포인트와 `resources/subscribe` / `resources/unsubscribe`가 `subscriptions/listen` 하나로 통합되었다.
이는 단일 장수명 POST 응답 스트림이며, 서버에서 클라이언트로 가는 변경 알림을 옵트인 방식으로 전달한다.

클라이언트가 옵트인할 수 있는 타입은 `toolsListChanged`, `promptsListChanged`, `resourcesListChanged`, `resourceSubscriptions`다.
서버는 구독을 확인한 뒤 알림에 `io.modelcontextprotocol/subscriptionId`를 태깅한다.

주의할 점은 요청 범위 알림의 처리다.
`notifications/progress`와 `notifications/message`는 `subscriptions/listen` 스트림이 아니라, 해당 요청의 응답 스트림으로 계속 흐른다.

### 인프라 친화적 헤더와 캐싱

Streamable HTTP POST 요청에 표준 MCP 요청 헤더가 필수화되었다.

```http
POST /mcp HTTP/1.1
Mcp-Method: tools/call
Mcp-Name: file_write
```

게이트웨이나 WAF가 JSON 본문을 파싱하지 않고 헤더만으로 라우팅, 로깅, 필터링, 속도 제한을 수행할 수 있다.
도구 파라미터에서 커스텀 헤더를 전달하는 `x-mcp-header` 지원도 추가되었다(SEP-2243).

캐싱도 표준화되었다.
`tools/list`, `prompts/list`, `resources/list`, `resources/read`, `resources/templates/list` 결과에 새로운 `CacheableResult` 인터페이스가 적용되어 `ttlMs`와 `cacheScope` 필드가 필수가 되었다.

| 필드 | 의미 |
|------|------|
| ttlMs | 신선도 힌트(밀리초). 클라이언트가 응답을 캐싱해 폴링을 줄이는 데 사용 |
| cacheScope | public 또는 private. 공유 중개자가 응답을 캐싱해도 되는지 제어 |

두 필드는 기존 `listChanged` 알림을 대체하는 것이 아니라 보완한다.
추가로 서버는 `tools/list` 결과를 결정적인 순서로 반환해야 한다(SHOULD).
클라이언트 측 캐싱과 LLM 프롬프트 캐시 적중률을 높이기 위한 조치다.

OpenTelemetry 트레이스 컨텍스트 전파 규약도 문서화되었다.
`_meta` 키로 `traceparent`, `tracestate`, `baggage`를 사용한다(SEP-414).

### 인증과 보안 강화

인가 서버는 RFC 9207에 따라 인가 응답에 `iss` 파라미터를 포함해야 하며(SHOULD), MCP 클라이언트는 인가 코드를 교환하기 전에 존재하는 `iss` 값을 기록된 발급자와 반드시 대조해야 한다(SEP-2468).

클라이언트 자격증명은 발급한 인가 서버에 바인딩된다.
클라이언트는 지속화한 자격증명을 발급자 식별자로 키잉해야 하며, 다른 인가 서버와 재사용해서는 안 되고, 인가 서버가 바뀌면 재등록해야 한다(SEP-2352).

동적 클라이언트 등록(DCR, RFC 7591)은 클라이언트 ID 메타데이터 문서(CIMD)를 선호하는 방향으로 deprecated 되었다.
CIMD를 지원하지 않는 인가 서버와의 하위 호환을 위해 DCR 자체는 남아 있다.
DCR을 사용할 때는 OpenID Connect 리다이렉트 URI 충돌을 피하기 위해 적절한 `application_type`을 지정해야 한다(SEP-837).

에러 코드 체계도 정비되었다.
리소스를 찾지 못했을 때의 코드가 JSON-RPC 사양에 맞춰 `-32002`에서 `-32602`(Invalid Params)로 변경되었다.
그리고 JSON-RPC 서버 에러 범위를 분할하는 할당 정책이 정의되었다.

| 범위 | 용도 |
|------|------|
| -32000 ~ -32019 | 구현 정의 영역. 기존 SDK 사용분은 그대로 인정 |
| -32020 ~ -32099 | MCP 사양 예약 영역 |

이 정책에 따라 이번 드래프트에서 도입된 코드가 재번호되었다.
`HeaderMismatch`는 -32001에서 -32020으로, `MissingRequiredClientCapability`는 -32003에서 -32021로, `UnsupportedProtocolVersion`은 -32004에서 -32022로 변경되었다.
전송 계층 산문에만 존재하던 `HeaderMismatchError`가 스키마에도 추가되었다.

스키마 관련 변경으로 `inputSchema`와 `outputSchema`가 JSON Schema 2020-12의 모든 키워드를 허용하도록 완화되었고, `structuredContent`는 임의의 JSON 값을 허용한다.
`$ref` 해석 요구사항과 합성 키워드에 대한 리소스 상한도 추가되었다(SEP-2106).
또한 `ClientCapabilities`와 `ServerCapabilities`에 `extensions` 필드가 추가되어 코어 프로토콜 밖의 선택적 확장을 선언할 수 있게 되었다.

### Tasks 확장으로의 분리

실험 단계였던 tasks 기능이 코어 프로토콜에서 빠져 공식 확장 `io.modelcontextprotocol/tasks`로 이동했다(SEP-2663).
재설계된 확장의 변경점은 다음과 같다.

- 블로킹 방식의 `tasks/result` 메서드를 폴링 기반 `tasks/get`으로 대체
- 클라이언트가 서버로 입력을 보내는 `tasks/update` 메서드 신설
- `tasks/list` 제거
- 서버가 요청별 옵트인 없이도 태스크 핸들을 자발적으로 반환 가능

코어를 최소화하고 기능을 버전 관리되는 확장으로 분리하는 방향성의 일부다.
같은 맥락에서 채팅 UI 렌더링을 담당하는 MCP Apps, 기업 관리형 인증을 담당하는 EMA도 별도 확장으로 존재한다.

## Deprecated 기능과 수명주기 정책

이번 개정에서 기능 수명주기 및 폐기 정책이 도입되었다(SEP-2596).
Active, Deprecated, Removed 세 가지 상태를 정의하고, 최소 12개월의 폐기 유예 기간을 보장한다.
현재 Deprecated 상태인 기능은 별도 레지스트리로 추적된다.

Deprecated 처리된 항목과 권장 마이그레이션 경로는 다음과 같다.

| 기능 | 권장 대안 |
|------|------|
| Roots | 디렉터리나 파일을 도구 파라미터, 리소스 URI, 서버 설정으로 전달 |
| Sampling | LLM 제공자 API와 직접 통합 |
| Logging | stdio 환경에서는 stderr, 그 외에는 OpenTelemetry 사용 |
| HTTP+SSE 전송 | Streamable HTTP로 이전 |
| includeContext의 thisServer / allServers 값 | 필드를 생략하거나 none 사용 |
| OAuth 2.0 DCR | 클라이언트 ID 메타데이터 문서(CIMD) |

Roots, Sampling, Logging은 SEP-2577에서 폐기가 결정되었다.
폐기 기간 중에는 완전히 동작하지만, 새 구현은 이들에 대한 지원을 추가하지 않아야 한다.
HTTP+SSE 전송은 `2025-03-26`부터 이미 deprecated였으며, 이번에 수명주기 정책상의 Deprecated로 재분류되었다.
`includeContext`의 `"thisServer"`, `"allServers"` 값은 Sampling 기능 자체보다 늦지 않게 제거된다.

프로세스 측면에서는 SEP 워크플로가 공식화되었다.
`seps/` 디렉터리의 마크다운 파일, PR에서 파생된 번호 체계, 스폰서 책임, PR 라벨을 통한 상태 관리가 정의되었다(SEP-1850).

## SDK 대응 현황

Tier 1 SDK인 Python, TypeScript, Go, C# 네 가지의 베타가 2026년 6월 29일 공개되었고, 사양 확정일인 7월 28일에 맞춰 지원이 완료되었다.
Rust SDK도 베타 수준으로 지원한다.

베타 기간은 약 4주였으며, 안정 버전까지 공개 API가 바뀔 수 있으므로 정확한 버전 고정이 권장되었다.
라이브러리 개발자는 `mcp>=1.27,<2` 같은 상한선을 두어 사용자를 보호하는 방식이 안내되었다.

기존 서버와 클라이언트는 계속 동작한다.
7월 28일 이후에도 변경 없이 사용할 수 있으며, 새 버전은 선택적 업그레이드다.

### Python SDK v2

가장 눈에 띄는 변화는 클래스명이다.
`FastMCP`가 `MCPServer`로 바뀌었다.
데코레이터 API는 유지되며, JSON Schema를 직접 작성할 필요가 없다.

```python
from mcp.server import MCPServer

mcp = MCPServer("Demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    """Greet someone by name."""
    return f"Hello, {name}!"
```

설치는 베타 기간 중 정확한 버전을 고정하는 방식이 권장되었다.

```bash
uv add "mcp[cli]==2.0.0b1"
pip install "mcp[cli]==2.0.0b1"
```

v1에서 v2로의 전환은 메이저 버전 변경이며 공식 마이그레이션 가이드가 제공된다.
v2 서버는 기존 `initialize` 핸드셰이크도 동시에 지원한다.
인메모리 테스트를 위해 서브프로세스 없이 `async with Client(mcp) as client:` 형태를 사용할 수 있다.
v1은 v2 출시 후 최소 6개월간 보안 패치를 받는다.

### TypeScript SDK v2

단일 패키지였던 `@modelcontextprotocol/sdk`가 분할 패키지 구조로 바뀌었다.
`@modelcontextprotocol/server`, `@modelcontextprotocol/client` 등으로 나뉜다.
Node.js 20 이상, Bun, Deno를 지원하며 ESM 전용이다.
Zod v4, Valibot, ArkType 등 Standard Schema 라이브러리를 사용할 수 있다.

```typescript
import { McpServer } from "@modelcontextprotocol/server";
import { StdioServerTransport } from "@modelcontextprotocol/server/stdio";
import * as z from "zod/v4";

const server = new McpServer({ name: "greeting-server", version: "1.0.0" });

server.registerTool(
  "greet",
  {
    description: "Greet someone by name",
    inputSchema: z.object({ name: z.string() }),
  },
  async ({ name }) => ({
    content: [{ type: "text", text: `Hello, ${name}!` }],
  }),
);
```

설치와 마이그레이션 도구는 다음과 같다.

```bash
npm install @modelcontextprotocol/server@beta
npm install @modelcontextprotocol/client@beta

npx @modelcontextprotocol/codemod@beta v1-to-v2 .
```

HTTP 배포는 `createMcpHandler`로 스테이트리스 프로토콜을 구현한다.
v1은 v2 출시 후 최소 6개월간 버그 수정과 보안 업데이트를 받는다.

### Go SDK와 C# SDK

Go SDK는 `v1.7.0-pre.1`로 제공되며 모듈 경로가 동일하다.

```bash
go get github.com/modelcontextprotocol/go-sdk@v1.7.0-pre.1
```

기존 API를 유지해 점진적 업그레이드가 가능하다.
HTTP 배포 시에는 `StreamableHTTPOptions.Stateless = true`로 명시적으로 활성화한다.

C# SDK는 `2.0.0-preview.1`이다.

```bash
dotnet add package ModelContextProtocol --prerelease
```

v1 안정 API 호환성을 유지하며, 폐기 예정 기능인 roots, sampling, logging은 `[Obsolete]`로 표시된다.
변경은 실험적 API에 한정된다.

## 사용자별 대응 전략

MCP 서버를 사용하기만 하는 입장이라면 특별히 할 일이 없다.
클라이언트와 SDK가 자동으로 폴백을 처리한다.
다만 SSE 전송으로 고정 설정한 경우에는 Streamable HTTP로 변경해야 한다.

서버 개발자는 마이그레이션을 시작할 시점이다.

- 세션 관리 코드를 삭제한다
- 필요한 상태는 서버가 발급한 핸들이나 외부 저장소로 이관한다
- 엘리시테이션을 MRTR 패턴으로 재구성한다
- 장기 실행 작업을 Tasks 확장으로 이전한다
- Roots, Sampling, Logging 의존을 걷어낸다

배포·운영 측면에서는 구성이 단순해진다.
스티키 세션이 더 이상 필요 없어 표준 라운드로빈 로드밸런싱을 쓸 수 있다.
쿠버네티스 서비스에 평범한 구성을 그대로 적용할 수 있고, 게이트웨이(agentgateway 1.4.0 이상)를 활용해 카나리 배포도 가능하다.

호환성은 양방향으로 유지된다.
구 스펙 서버와 신 스펙 클라이언트, 그 반대 조합 모두 동작하도록 양쪽 지원이 요구된다.
강제 전환 시점은 없으며 최소 12개월의 유예 기간이 보장된다.

## 결론

2026-07-28 개정의 방향은 명확하다.
세션과 핸드셰이크, 서버 주도 요청처럼 상태를 전제하던 요소를 제거하고, 요청 하나가 자체적으로 완결되도록 만들었다.
그 결과 로드밸런싱, 캐싱, 게이트웨이 라우팅, 관측성 같은 기존 HTTP 인프라의 자산을 그대로 활용할 수 있게 되었다.

동시에 코어는 최소화하고 Tasks, MCP Apps, EMA 같은 기능은 버전 관리되는 확장으로 분리했다.
기능 수명주기 정책과 최소 12개월 폐기 유예 기간이 함께 도입되어, 사양 변경이 구현체를 갑자기 깨뜨리지 않도록 하는 장치도 마련되었다.
실험적 프로토콜에서 운영 가능한 인프라로 넘어가는 단계로 볼 수 있다.

## Reference

- [The Next Generation of MCP](https://blog.modelcontextprotocol.io/posts/2026-07-28/)
- [MCP Specification 2026-07-28 Changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [MCP SDK Betas for 2026-07-28](https://blog.modelcontextprotocol.io/posts/sdk-betas-2026-07-28/)
- [MCP 새로운 스펙 총정리: 무엇을 결정하고 바꿔야 할까?](https://yozm.wishket.com/magazine/detail/3893/)
