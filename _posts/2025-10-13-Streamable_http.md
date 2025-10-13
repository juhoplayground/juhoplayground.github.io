---
layout: post
title: Streamable HTTP란 무엇인가?
author: 'Juho'
date: 2025-10-13 09:00:00 +0900
categories: [MCP]
tags: [MCP, Streamable HTTP, SSE, FastMCP, Spring AI]
pin: True
toc : True
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
1. [Streamable HTTP란 무엇인가?](#streamable-http란-무엇인가)  
2. [SSE(Server-Sent Events)와 무엇이 다른가?](#sseserver-sent-events와-무엇이-다른가)
3. [Streamable HTTP의 장점](#streamable-http의-장점)
4. [Streamable HTTP의 단점 및 주의사항](#streamable-http의-단점-및-주의사항)
5. [언제 Streamable HTTP를 쓰면 좋나?](#언제-streamable-http를-쓰면-좋나)
6. [Python-FastMCP](#python-fastmcp)  
7. [Java-Spring AI](#java-spring-ai)

## Streamable HTTP란 무엇인가?
Streamable HTTP는 Model Context Protocol(MCP)의 표준 전송 방식 중 하나로  
MCP 서버를 HTTP 기반의 독립 프로세스로 띄우고 여러 클라이언트가 동시에 접속해 양방향 통신을 할 수 있게 하는 프로토콜이다.  
요청→서버: 각 JSON-RPC 메시지를 개별 HTTP POST로 보낸다.  
서버→클라이언트: 서버는 응답을 단일 JSON으로 돌려줄 수도 있고 SSE(text/event-stream)로 스트림을 열어 다중 메시지를 연속 전송할 수도 있다.  
또한 클라이언트가 별도의 HTTP GET으로 서버의 알림 스트림을 구독할 수도 있다.  
MCP 사양은 기존 `HTTP+SSE` 운용 방식을 대체하는 것으로 Streamable HTTP를 명시하고 보안 권고 및 세션/재전송 등 운용 세부를 규정한다.  

## SSE(Server-Sent Events)와 무엇이 다른가?
전통적인 SSE는 서버→클라이언트 단방향 스트리밍이다.  
클라이언트의 상호작용은 보통 별도 HTTP 요청/폴링이나 다른 채널이 필요해 완전한 양방향 흐름을 만들기 번거롭다.  
반면 Streamable HTTP는 POST(클라이언트→서버) 와 SSE/JSON(서버→클라이언트) 를 하나의 MCP 엔드포인트에서 통합 운용한다.  
요청-응답과 실시간 스트림을 한 프로토콜로 다루므로 도구 실행 중간 진행 이벤트, 리소스/프롬프트 변경 알림, 추가 쿼리 요청 같은 서버발 메시지를 자연스럽게 끼워 넣을 수 있다.  

## Streamable HTTP의 장점
- 진짜 양방향 흐름  
  - 클라이언트 요청은 POST로, 서버 응답은 JSON 또는 SSE 스트림으로 다중 메시지 전달이 가능해서 중간 진행 상황(progress)·로그·부분 결과 등을 실시간으로 보낼 수 있다.  
  - 별도 웹소켓 없이도 실시간 상호작용 UX를 만들기 쉽다.  
- 멀티 클라이언트·네트워크 배포에 적합  
  - 하나의 서버가 동시에 여러 클라이언트를 처리하고, 표준 HTTP 인프라(리버스 프록시, 인증 게이트웨이, 로깅, 모니터링)와 잘 맞는다.  
- 세션·재전송·프로토콜 버전 협상  
  - 사양이 세션 ID, Last-Event-ID 기반 스트림 재개, MCP-Protocol-Version 헤더를 정의해 연결 끊김 회복·하위 호환 시나리오를 표준화한다.  
- 레거시 SSE 대비 이관이 쉬움  
 - 사양이 백워드 컴패티빌리티 가이드를 제공해 구형 HTTP+SSE 서버/클라이언트와의 공존 전략을 제시한다.
  
## Streamable HTTP의 단점 및 주의사항  
- 구현 복잡도 증가  
  - 단순 SSE보다 요청 경로(POST)·스트림 경로(GET/SSE)·세션·재전송 등 고려할 지점이 많아진다.  
  - 잘못 구현하면 중복 전송/유실/순서 보장 문제가 생길 수 있으니 사양의 제약(응답을 스트림에서 반드시 완료 후 종료 등)을 지켜야 한다.  
- SSE의 인프라 제약은 여전히 존재  
  - 서버→클라이언트 스트림은 SSE를 활용하므로, 프록시/로드밸런서의 타임아웃, HTTP/2 전환, 헤더 버퍼 같은 이슈를 점검해야 한다.  
- 보안 고려 필수  
  - 사양은 Origin 헤더 검증, 로컬 바인딩(127.0.0.1), 인증 적용을 강하게 권고한다.  
  - 특히 DNS Rebinding 공격에 취약할 수 있으므로 방어가 필요하다.  
- 레거시 클라이언트 호환성  
  - 구형 SSE 전용 클라이언트와는 바로 호환되지 않을 수 있다.  
  - 사양이 제시한 폴백 절차(405/404 시 구형 프로토콜로 전환)를 구현해야 무중단 전환이 가능하다.  

## 언제 Streamable HTTP를 쓰면 좋나?  
툴 실행 과정의 중간 단계, 로그, 토큰 스트리밍 등 부분 결과를 즉시 보여줘야 하는 UX  
멀티 사용자·멀티 세션 환경, 중앙 서버로 MCP를 네트워크에 공개해야 하는 경우  
리소스/도구/프롬프트 변경 알림 같이 서버발 이벤트가 중요한 시스템  
이미 HTTP 인프라(인증 프록시, API 게이트웨이, 관측·모니터링)를 보유한 조직  
  
  
반대로 단일 사용자/로컬에서 간단히 붙이는 용도라면 STDIO가 더 간단하고 정말 단순한 서버→클라이언트 알림만 필요하다면 구형 SSE도(레거시지만) 충분할 수 있다.  

## Python-FastMCP
Python의 FastMCP가 Streamable HTTP 전송으로 제공하고 SSE 전송은 레거시로 분류해 신규 프로젝트에선 HTTP 사용을 권장한다.  
### Streamable HTTP MCP 서버  
```python
from fastmcp import FastMCP

mcp = FastMCP("MyServer")

@mcp.tool
def hello(name: str) -> str:
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="http", host="127.0.0.1", port=8000)
```
  
### Streamable HTTP MCP 클라이언트   
```python
from fastmcp.client.transports import StreamableHttpTransport

transport = StreamableHttpTransport(url="https://api.example.com/mcp")
client = Client(transport)

transport = StreamableHttpTransport(
    url="https://api.example.com/mcp",
    headers={
        "Authorization": "Bearer your-token-here",
        "X-Custom-Header": "value"
    }
)
client = Client(transport)
```
  

## Java-Spring AI  
Spring AI가 STREAMABLE 프로토콜을 제공하며 WebMVC/WebFlux 스타터로 스트리머블 서버를 간단히 구성하고 도구/리소스/프롬프트 변경 알림, keep-alive(ping) 등의 운용 옵션을 지원한다.  

### Streamable HTTP MCP 서버  
- 의존성 추가 (maven 방식)  
```xml
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client-webflux</artifactId>
  </dependency>
```
  

- 의존성 추가 (gradle 방식)  
```groovy
  dependencies {
      implementation 'org.springframework.ai:spring-ai-starter-mcp-server-webmvc:1.1-SNAPSHOT'
      implementation 'org.springframework.ai:spring-ai-starter-mcp-client-webflux:1.1-SNAPSHOT'
  }
```

- 서버 설정  
```yml
spring:
  ai:
    mcp:
      server:
        protocol: STREAMABLE
        name: streamable-mcp-server
        version: 1.0.0
        type: SYNC
        instructions: "This streamable server provides real-time notifications"
        resource-change-notification: true
        tool-change-notification: true
        prompt-change-notification: true
        streamable-http:
          mcp-endpoint: /api/mcp
          keep-alive-interval: 30s
```
  

- sever 구현 코드  
  
```java 
@Service
public class WeatherService {

    @Tool(description = "Get weather information by city name")
    public String getWeather(String cityName) {
        // Implementation
    }
}

@SpringBootApplication
public class McpServerApplication {

    private static final Logger logger = LoggerFactory.getLogger(McpServerApplication.class);

    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }

	@Bean
	public ToolCallbackProvider weatherTools(WeatherService weatherService) {
		return MethodToolCallbackProvider.builder().toolObjects(weatherService).build();
	}
}
```

### Streamable HTTP MCP 클라이언트  
- 의존성 추가 (maven 방식)  
```xml
  <dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
  </dependency>
```
  

- 의존성 추가 (gradle 방식)  
```groovy
  dependencies {
      implementation 'org.springframework.ai:spring-ai-starter-mcp-client'
  }
```

- 클라이언트 설정  
```yml
spring:
  ai:
    mcp:
      client:
        streamable-http:
          connections:
            server1:
              url: http://localhost:8080
            server2:
              url: http://otherserver:8081
              endpoint: /custom-sse
```

- 클라이언트 구현 코드
```java
@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}	

	@Value("${ai.user.input}")
	private String userInput;

	@Bean
	public CommandLineRunner predefinedQuestions(ChatClient.Builder chatClientBuilder, ToolCallbackProvider tools,
			ConfigurableApplicationContext context) {

		return args -> {

			var chatClient = chatClientBuilder
					.defaultToolCallbacks(tools)
					.build();

			System.out.println("\n>>> QUESTION: " + userInput);
			System.out.println("\n>>> ASSISTANT: " + chatClient.prompt(userInput).call().content());

			context.close();
		};
	}
}
```

---  

참고 문서  
- FastMCP  
  - https://gofastmcp.com/deployment/running-server  
  - https://gofastmcp.com/clients/transports  
- Spring AI  
  - https://docs.spring.io/spring-ai/reference/1.1-SNAPSHOT/api/mcp/mcp-streamable-http-server-boot-starter-docs.html  
  - https://docs.spring.io/spring-ai/reference/1.1-SNAPSHOT/api/mcp/mcp-client-boot-starter-docs.html  
- MCP  
  - https://modelcontextprotocol.io/specification/2025-06-18/basic/transports