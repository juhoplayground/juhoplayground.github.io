---
layout: post
title: "Spring SSE 안정적인 연결 관리 가이드 - Heartbeat, 재연결, Thread-safe 종료까지"
author: 'Juho'
date: 2026-03-29 00:00:00 +0900
categories: [SpringBoot]
tags: [SpringBoot, SSE, thread, threadpool]
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
2. [Heartbeat/Keepalive 메커니즘](#heartbeatkeepalive-메커니즘)
3. [재연결/재시도 로직](#재연결재시도-로직)
4. [상태 머신 기반 Thread-safe 관리](#상태-머신-기반-thread-safe-관리)
5. [안전한 Emitter 종료](#안전한-emitter-종료)
6. [클라이언트 연결 끊김 감지](#클라이언트-연결-끊김-감지)
   - [방식 1: IOException 기반 감지 (ClientAbortException / Broken pipe)](#방식-1-ioexception-기반-감지-clientabortexception--broken-pipe)
   - [방식 2: Spring 6.2+ AsyncRequestNotUsableException](#방식-2-spring-62-asyncrequestnotusableexception)
   - [방식 3: Keepalive 전송 실패 감지](#방식-3-keepalive-전송-실패-감지)
7. [타임아웃 설정](#타임아웃-설정)
8. [커넥션 풀 관리](#커넥션-풀-관리)
9. [SSE 전용 스레드 풀 (AsyncConfig)](#sse-전용-스레드-풀-asyncconfig)
10. [결론](#결론)

## 개요

Spring에서 SSE(Server-Sent Events)를 운영 환경에서 사용하다 보면, 단순히 `SseEmitter`를 생성해서 이벤트를 보내는 것만으로는 부족하다.
연결이 끊어졌는지 모르고 계속 이벤트를 보내거나, 타임아웃 처리가 미흡하거나, 멀티스레드 환경에서 동시 접근 문제가 발생할 수 있다.

이 글에서는 외부 AI API와의 SSE 프록시 스트리밍 서비스를 구현하면서 적용한 8가지 핵심 포인트를 정리한다.
전체 구조는 외부 API로부터 SSE 이벤트를 수신(WebClient + Reactor)하고, 이를 클라이언트에게 `SseEmitter`로 중계하는 프록시 엔진 형태다.

## Heartbeat/Keepalive 메커니즘

SSE 연결은 HTTP 기반의 단방향 스트림이기 때문에, 중간 프록시나 로드밸런서가 유휴 연결을 끊어버릴 수 있다.
특히 외부 AI API의 응답이 느릴 때(Deep Research 등) 클라이언트 연결이 먼저 끊기는 문제가 발생한다.

이를 방지하기 위해 `ScheduledExecutorService`로 5초마다 전체 활성 스트림을 순회하고, 마지막 이벤트로부터 15초 이상 지나면 keepalive comment를 전송한다.
스트림마다 개별 스케줄 태스크를 생성하는 대신, 단일 스케줄 태스크에서 전체 스트림을 순회하는 방식을 사용한다.
이렇게 하면 스케줄 태스크 수가 동시 스트림 수와 무관하게 항상 1개만 존재한다.

```java
private final ScheduledExecutorService keepaliveScheduler =
    Executors.newSingleThreadScheduledExecutor();
private static final long KEEPALIVE_THRESHOLD_MS = 15_000;

@PostConstruct
void startGlobalKeepalive() {
    keepaliveScheduler.scheduleAtFixedRate(() -> {
        long now = System.currentTimeMillis();
        sseStreamRegistry.getAllContexts().forEach((chatId, ctx) -> {
            long elapsed = now - ctx.lastEventTime().get();
            if (elapsed >= KEEPALIVE_THRESHOLD_MS
                && ctx.state().get() == StreamState.ACTIVE) {
                try {
                    ctx.emitter().send(SseEmitter.event().comment("keepalive"));
                    ctx.lastEventTime().set(now);
                } catch (Exception e) {
                    if (ChatSseUtils.isClientDisconnectException(e)) {
                        handleClientDisconnect(ctx, chatId);
                    }
                }
            }
        });
    }, 5, 5, TimeUnit.SECONDS);
}

@PreDestroy
void shutdownKeepaliveScheduler() {
    keepaliveScheduler.shutdownNow();
}
```

핵심 포인트는 다음과 같다.
- SSE event가 아니라 comment(`emitter.send(SseEmitter.event().comment("keepalive"))`)로 전송한다.
- comment는 클라이언트의 `onmessage` 핸들러를 트리거하지 않으므로, 비즈니스 이벤트와 간섭이 없다.
- `lastEventTime`을 `AtomicLong`으로 관리하여 이벤트 수신 시마다 갱신하고, 15초 이상 유휴 상태일 때만 keepalive를 전송한다.
- keepalive 전송 실패 시 `isClientDisconnectException`으로 클라이언트 단절을 감지한다.
- `@PreDestroy`로 애플리케이션 종료 시 스케줄러를 정리한다.

## 재연결/재시도 로직

외부 API 서버의 커넥션 풀에서 유휴 커넥션이 만료되면 `PrematureCloseException`이 발생할 수 있다.
이 경우에만 1회 재시도하고, 그 외의 에러는 재시도 없이 즉시 실패 처리한다.

Reactor의 `retryWhen`을 사용해서 선언적으로 재시도 조건을 정의한다.

```java
Flux<ServerSentEvent<String>> sseFlux =
    webClient
        .post()
        .uri(streamUrl)
        .header("X-User-API-Key", apiKey)
        .contentType(MediaType.APPLICATION_JSON)
        .bodyValue(externalRequest)
        .retrieve()
        .bodyToFlux(new ParameterizedTypeReference<ServerSentEvent<String>>() {})
        .retryWhen(
            Retry.max(1)
                .filter(ChatSseUtils::isPrematureCloseException)
                .onRetryExhaustedThrow((spec, signal) -> signal.failure())
                .doBeforeRetry(
                    signal ->
                        log.warn(
                            "Chat stream connection lost (stale connection),"
                                + " retrying (attempt {}): {}",
                            signal.totalRetries() + 1,
                            signal.failure().getMessage())))
        .timeout(Duration.ofSeconds(TIMEOUT_SECONDS));
```

`PrematureCloseException` 판별은 `WebClientRequestException`으로 래핑된 경우도 처리해야 한다.

```java
public static boolean isPrematureCloseException(Throwable throwable) {
    if (throwable instanceof reactor.netty.http.client.PrematureCloseException) {
        return true;
    }
    if (throwable
        instanceof org.springframework.web.reactive.function.client.WebClientRequestException) {
        Throwable cause = throwable.getCause();
        return cause instanceof reactor.netty.http.client.PrematureCloseException;
    }
    return false;
}
```

핵심 포인트는 다음과 같다.
- `Retry.max(1)`: 최대 1회만 재시도한다. 무한 재시도는 외부 서버에 부담을 줄 수 있다.
- `.filter(ChatSseUtils::isPrematureCloseException)`: stale connection으로 인한 에러만 재시도 대상으로 허용한다. 네트워크 에러, 서버 에러 등은 재시도하지 않는다.
- `.onRetryExhaustedThrow((spec, signal) -> signal.failure())`: 재시도 횟수 초과 시 원래 예외를 그대로 던진다.

## 상태 머신 기반 Thread-safe 관리

SSE 프록시 엔진에서는 여러 스레드가 동시에 하나의 Emitter에 접근한다.
Keepalive 스케줄러 스레드, Reactor 이벤트 수신 스레드, Spring의 onTimeout/onCompletion/onError 콜백 스레드가 모두 같은 Emitter 상태를 변경할 수 있다.

독립된 `AtomicBoolean` 여러 개로 상태를 관리하면, 이론상 플래그 수만큼의 조합이 가능해지고 유효하지 않은 상태 조합이 생길 수 있다.
`AtomicReference<StreamState>` 하나로 상태를 관리하면 허용된 전이만 발생하도록 보장할 수 있다.

```java
enum StreamState {
    ACTIVE,              // 정상 스트리밍 중
    CLIENT_DISCONNECTED, // 클라이언트 끊김, 외부 구독 유지
    STOPPED,             // 사용자 수동 중지
    TRUNCATED,           // 응답 길이 초과
    ERROR,               // 에러 발생
    FINALIZED            // 종료 완료
}
```

스트림 컨텍스트를 record로 묶어서 관리한다.

```java
record SseStreamContext(
    SseEmitter emitter,
    AtomicReference<StreamState> state,
    AtomicBoolean errorEventSent,       // 전송 여부 가드 (상태와 독립)
    AtomicReference<Disposable> subscriptionRef,
    AtomicLong lastEventTime,
    AtomicInteger promptTokens,
    AtomicInteger completionTokens,
    AtomicReference<String> report,
    AtomicReference<String> references
) {
    static SseStreamContext create(SseEmitter emitter) {
        return new SseStreamContext(
            emitter,
            new AtomicReference<>(StreamState.ACTIVE),
            new AtomicBoolean(false),
            new AtomicReference<>(),
            new AtomicLong(System.currentTimeMillis()),
            new AtomicInteger(0),
            new AtomicInteger(0),
            new AtomicReference<>(""),
            new AtomicReference<>("")
        );
    }
}
```

`errorEventSent`는 상태가 아니라 "전송 여부"를 추적하는 가드이므로 별도의 `AtomicBoolean`으로 유지한다.
상태 전이는 `compareAndSet`으로 제어하며, 허용된 전이만 발생하도록 보장한다.

```java
// ACTIVE → CLIENT_DISCONNECTED 전이
if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.CLIENT_DISCONNECTED)) {
    log.warn("클라이언트 단절 감지, 외부 구독 유지 - 세션 ID: {}", chatId);
}

// ACTIVE → STOPPED 전이
if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.STOPPED)) {
    log.info("사용자 수동 중지 - 세션 ID: {}", chatId);
}

// ACTIVE → ERROR 전이
if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.ERROR)) {
    ChatSseUtils.sendSseErrorEventSafely(ctx.emitter(), errorMessage, ctx.errorEventSent());
    ChatSseUtils.disposeSubscription(ctx.subscriptionRef());
}
```

핵심은 `CLIENT_DISCONNECTED`와 `ERROR`의 분리다.
클라이언트가 연결을 끊어도 외부 API 구독은 유지해야 한다.
외부 API 응답이 끝까지 도달해야 DB에 결과를 저장할 수 있기 때문이다.

`CLIENT_DISCONNECTED` 상태에서는 `subscriptionRef`를 dispose하지 않는다.
`ERROR` 상태에서는 외부 구독까지 즉시 정리한다.

종료 처리는 어떤 상태에서든 `FINALIZED`로 전이하여 1회 실행을 보장한다.

```java
StreamState current = ctx.state().get();
if (current != StreamState.FINALIZED
    && ctx.state().compareAndSet(current, StreamState.FINALIZED)) {
    try {
        // DB 저장, Emitter 종료 등 후처리
        ChatSseUtils.completeEmitterSafely(ctx.emitter(), ctx.state());
    } finally {
        sseStreamRegistry.unregister(chatId);
    }
}
```

## 안전한 Emitter 종료

`emitter.complete()`는 여러 경로에서 호출될 수 있다.
onTimeout 콜백, onError 콜백, 스트림 정상 완료, 에러 이벤트 수신 등 다양한 종료 경로가 있다.
상태 머신에서 `FINALIZED` 전이가 한 번만 성공하므로, 중복 호출이 원천 방지된다.

```java
public static void completeEmitterSafely(
    SseEmitter emitter, AtomicReference<StreamState> state) {
    if (state.get() == StreamState.FINALIZED) {
        try {
            emitter.complete();
        } catch (Exception ignored) {
            // 클라이언트 단절로 이미 complete된 경우 무시
        }
    }
}
```

중요한 설계 결정으로, `completeWithError()`는 사용하지 않는다.
SSE 응답이 `text/event-stream`으로 이미 커밋된 상태에서 `completeWithError()`를 호출하면 `GlobalExceptionHandler`가 재호출되어 의도하지 않은 오류가 발생한다.
대신 에러는 SSE error 이벤트로 먼저 전송한 뒤, `emitter.complete()`를 호출하는 패턴을 사용한다.

```java
public static void sendSseErrorEventSafely(
    SseEmitter emitter, String message, AtomicBoolean errorEventSent) {
    if (!errorEventSent.compareAndSet(false, true)) {
        return; // 이미 전송됨
    }
    try {
        emitter.send(SseEmitter.event().name("error").data(message));
    } catch (Exception e) {
        log.error("에러 이벤트 전송 실패: {}", e.getMessage(), e);
    }
}
```

리소스 정리도 안전하게 처리해야 한다.
Reactor `Disposable`을 `AtomicReference`로 관리하여, `getAndSet(null)` 패턴으로 중복 정리를 방지한다.

```java
public static void disposeSubscription(AtomicReference<Disposable> subscriptionRef) {
    Disposable disposable = subscriptionRef.getAndSet(null);
    if (disposable != null && !disposable.isDisposed()) {
        disposable.dispose();
    }
}
```

## 클라이언트 연결 끊김 감지

클라이언트가 브라우저를 닫거나 네트워크가 끊어져도, 서버는 이를 즉시 알 수 없다.
3가지 방식을 조합하여 연결 끊김을 감지한다.

### 방식 1: IOException 기반 감지 (ClientAbortException / Broken pipe)

`emitter.send()` 호출 시 `IOException`이 발생하면 연결이 끊어진 것으로 판단한다.
Tomcat이 EPIPE를 감지하면 `ClientAbortException`으로 래핑하고, 래핑되지 않는 엣지 케이스에서는 "Broken pipe" 메시지를 확인한다.

```java
public static boolean isClientAbortException(IOException e) {
    if (e instanceof org.apache.catalina.connector.ClientAbortException) {
        return true;
    }
    String message = e.getMessage();
    if (message == null) {
        return false;
    }
    return message.toLowerCase().contains("broken pipe");
}
```

주의할 점으로, "Connection reset"(ECONNRESET)은 클라이언트 단절 판별에 포함하지 않는다.
ECONNRESET은 클라이언트 자발적 종료 외에도 로드밸런서/프록시의 idle timeout 리셋, 네트워크 장비의 TCP RST 등 실제 오류 상황에서도 발생한다.
잘못 판별하면 실제 오류를 에러 히스토리 없이 무시하는 버그가 된다.

### 방식 2: Spring 6.2+ AsyncRequestNotUsableException

Spring 6.2부터 클라이언트 SSE 연결 끊김 시 `AsyncRequestNotUsableException`이 발생한다.
기존 IOException 기반 판별과 통합하여 단일 메서드로 감지한다.

```java
public static boolean isClientDisconnectException(Exception e) {
    if (e instanceof AsyncRequestNotUsableException) {
        return true;
    }
    if (e instanceof IOException ioEx) {
        return isClientAbortException(ioEx);
    }
    return false;
}
```

`GlobalExceptionHandler`에서도 `AsyncRequestNotUsableException`을 별도로 처리해야 한다.

```java
@ExceptionHandler(AsyncRequestNotUsableException.class)
public void handleAsyncRequestNotUsableException(
    AsyncRequestNotUsableException e, HttpServletRequest request) {
    log.debug("클라이언트 단절로 응답 불가 - URI: {}", request.getRequestURI());
}
```

### 방식 3: Keepalive 전송 실패 감지

앞서 설명한 keepalive 스케줄러가 5초마다 동작하면서, keepalive 전송 실패 시 클라이언트 단절을 감지한다.
이벤트 전송이 없는 유휴 기간에도 능동적으로 감지할 수 있다는 장점이 있다.

이벤트 전달 시점에서도 같은 패턴으로 감지한다.

```java
if (ctx.state().get() == StreamState.ACTIVE) {
    try {
        emitter.send(sseEventBuilder);
    } catch (Exception e) {
        if (ChatSseUtils.isClientDisconnectException(e)) {
            if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.CLIENT_DISCONNECTED)) {
                log.warn("클라이언트 단절 감지, 외부 구독 유지 - 세션 ID: {}", chatId);
            }
        } else {
            if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.ERROR)) {
                ChatSseUtils.disposeSubscription(ctx.subscriptionRef());
                ChatSseUtils.completeEmitterSafely(ctx.emitter(), ctx.state());
            }
        }
    }
}
```

클라이언트 단절과 실제 에러를 구분하는 것이 중요하다.
`CLIENT_DISCONNECTED` 전이 시에는 외부 API 구독(`subscriptionRef`)을 dispose하지 않고, `ERROR` 전이 시에는 즉시 정리한다.
이렇게 해야 클라이언트가 떠나더라도 외부 API 응답을 끝까지 수신하여 DB에 저장할 수 있다.

## 타임아웃 설정

타임아웃은 두 가지 레이어에서 설정해야 한다.
클라이언트 쪽 `SseEmitter` 타임아웃과, 외부 API 쪽 WebClient 타임아웃이다.

```java
// 컨트롤러: SseEmitter 타임아웃 없음 (0L)
SseEmitter emitter = new SseEmitter(0L);
emitter.onTimeout(() -> emitter.complete());
emitter.onError(error -> log.debug("SSE onError (클라이언트 단절): {}", error.getMessage()));
```

`SseEmitter`를 `0L`로 생성하여 서블릿 레벨 타임아웃을 비활성화한다.
대신 외부 API 쪽 타임아웃을 600초(10분)로 설정하여, AI API의 Deep Research 등 긴 응답을 수용한다.

```java
private static final int TIMEOUT_SECONDS = 600;

// WebClient: 외부 API 응답 타임아웃
HttpClient httpClient =
    HttpClient.create(connectionProvider)
        .responseTimeout(Duration.ofSeconds(TIMEOUT_SECONDS));

// Reactor Flux: 스트림 전체 타임아웃
.timeout(Duration.ofSeconds(TIMEOUT_SECONDS));
```

Emitter 타임아웃 콜백에서는 상태를 `ERROR`로 전이한 뒤, 에러 이벤트 전송과 리소스 정리를 수행한다.

```java
emitter.onTimeout(() -> {
    if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.ERROR)) {
        ChatSseUtils.sendSseErrorEventSafely(
            emitter, "채팅 스트림 타임아웃이 발생했습니다.", ctx.errorEventSent());
        ChatSseUtils.disposeSubscription(ctx.subscriptionRef());
        ChatSseUtils.completeEmitterSafely(emitter, ctx.state());
        sseStreamRegistry.unregister(chatId);
    }
});
```

onCompletion과 onError 콜백도 각각 역할이 다르다.

```java
emitter.onCompletion(() -> {
    // ACTIVE 상태에서만 외부 구독 정리 (CLIENT_DISCONNECTED면 유지)
    if (ctx.state().get() == StreamState.ACTIVE) {
        ChatSseUtils.disposeSubscription(ctx.subscriptionRef());
    }
});

emitter.onError(error -> {
    boolean isClientDisconnect =
        ctx.state().get() == StreamState.CLIENT_DISCONNECTED
            || (error instanceof Exception ex
                && ChatSseUtils.isClientDisconnectException(ex));

    if (isClientDisconnect) {
        ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.CLIENT_DISCONNECTED);
        // 외부 구독은 유지: DB 저장을 위해 응답을 끝까지 수신
    } else {
        if (ctx.state().compareAndSet(StreamState.ACTIVE, StreamState.ERROR)) {
            ChatSseUtils.disposeSubscription(ctx.subscriptionRef());
            ChatSseUtils.completeEmitterSafely(emitter, ctx.state());
            sseStreamRegistry.unregister(chatId);
        }
    }
});
```

## 커넥션 풀 관리

외부 AI API에 SSE 요청을 보내는 WebClient는 Reactor Netty의 `ConnectionProvider`로 커넥션 풀을 관리한다.
풀을 적절히 설정하지 않으면 stale connection으로 인한 `PrematureCloseException`이 빈번하게 발생한다.

```java
ConnectionProvider connectionProvider =
    ConnectionProvider.builder("chat-sse-pool")
        .maxConnections(50)
        .maxIdleTime(Duration.ofSeconds(20))
        .maxLifeTime(Duration.ofMinutes(10))
        .evictInBackground(Duration.ofSeconds(5))
        .build();

HttpClient httpClient =
    HttpClient.create(connectionProvider)
        .responseTimeout(Duration.ofSeconds(TIMEOUT_SECONDS));

WebClient webClient = webClientBuilder
    .baseUrl(baseUrl)
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

각 설정값의 의미는 다음과 같다.

| 설정 | 값 | 의미 |
|------|-----|------|
| maxConnections | 50 | 동시 최대 연결 수 |
| maxIdleTime | 20초 | 유휴 커넥션 유지 시간 (서버 idle timeout보다 짧게) |
| maxLifeTime | 10분 | 커넥션 최대 수명 (오래된 커넥션 순환) |
| evictInBackground | 5초 | 백그라운드 커넥션 정리 주기 |

서비스별로 커넥션 풀을 분리하는 것도 중요하다.
메인 API와 SDF API가 별도 풀을 사용하도록 구성하여, 한쪽 서버의 장애가 다른 쪽에 영향을 주지 않도록 했다.

```java
// 메인 API 풀
ConnectionProvider mainPool =
    ConnectionProvider.builder("chat-sse-pool")
        .maxConnections(50)
        .maxIdleTime(Duration.ofSeconds(20))
        .maxLifeTime(Duration.ofMinutes(10))
        .evictInBackground(Duration.ofSeconds(5))
        .build();

// SDF API 풀 (별도 분리)
ConnectionProvider sdfPool =
    ConnectionProvider.builder("chat-sse-sdf-pool")
        .maxConnections(50)
        .maxIdleTime(Duration.ofSeconds(20))
        .maxLifeTime(Duration.ofMinutes(10))
        .evictInBackground(Duration.ofSeconds(5))
        .build();
```

활성 SSE 스트림을 추적하는 `SseStreamRegistry`는 `ConcurrentHashMap`으로 Thread-safe하게 관리한다.
`finally` 블록에서 `unregister`를 호출하지만, 모든 예외 경로를 완벽하게 커버하기는 어렵다.
TTL 기반 자동 정리를 적용하여, 비정상 종료 시에도 항목이 영구 등록 상태로 남지 않도록 방어한다.

```java
@Component
public class SseStreamRegistry {

    private final Map<String, StreamInfo> activeStreams = new ConcurrentHashMap<>();

    record StreamInfo(String chatType, Instant registeredAt, SseStreamContext context) {}

    public void register(String chatId, String chatType, SseStreamContext context) {
        activeStreams.put(chatId, new StreamInfo(chatType, Instant.now(), context));
    }

    public void unregister(String chatId) {
        activeStreams.remove(chatId);
    }

    public boolean isActive(String chatId) {
        return activeStreams.containsKey(chatId);
    }

    public String getChatType(String chatId) {
        StreamInfo info = activeStreams.get(chatId);
        return info != null ? info.chatType() : null;
    }

    public Map<String, SseStreamContext> getAllContexts() {
        Map<String, SseStreamContext> result = new ConcurrentHashMap<>();
        activeStreams.forEach((id, info) -> result.put(id, info.context()));
        return result;
    }

    /** TTL 초과 항목 자동 정리 - 스트림 최대 수명(TIMEOUT_SECONDS)보다 여유 있게 설정 */
    @Scheduled(fixedRate = 60_000)
    void evictStaleStreams() {
        Instant cutoff = Instant.now().minus(Duration.ofMinutes(15));
        activeStreams.entrySet().removeIf(entry -> {
            boolean stale = entry.getValue().registeredAt().isBefore(cutoff);
            if (stale) {
                log.warn("TTL 초과로 SSE 스트림 레지스트리 정리 - chatId: {}", entry.getKey());
            }
            return stale;
        });
    }
}
```

TTL은 외부 API 타임아웃(600초) + 여유분으로 15분을 설정한다.
정상적인 스트림은 `finally`에서 먼저 해제되고, TTL 정리는 비정상 종료 시의 안전망 역할을 한다.

## SSE 전용 스레드 풀 (AsyncConfig)

SSE 스트리밍 작업을 다른 비동기 작업과 격리하기 위해 전용 스레드 풀을 구성한다.
Java 21+ 환경에서는 Virtual Thread를 사용하여 스레드 풀 크기 관리를 근본적으로 해결할 수 있다.

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "sseTaskExecutor")
    public TaskExecutor sseTaskExecutor() {
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

Virtual Thread는 블로킹 I/O 대기 시 OS 스레드를 반환하므로, `emitter.send()` 같은 블로킹 호출이 많은 SSE 스트리밍에 적합하다.
`corePoolSize`, `maxPoolSize`, `queueCapacity` 같은 튜닝 파라미터를 관리할 필요가 없고, 동시 수천 개 스트림도 OS 스레드 부담 없이 처리할 수 있다.

MDC 컨텍스트는 `TaskDecorator`와 동일한 패턴으로 전파한다.

```java
@Bean(name = "sseTaskExecutor")
public TaskExecutor sseTaskExecutor() {
    TaskExecutorAdapter adapter =
        new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    adapter.setTaskDecorator(runnable -> {
        Map<String, String> context = MDC.getCopyOfContextMap();
        return () -> {
            if (context != null) {
                MDC.setContextMap(context);
            }
            try {
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    });
    return adapter;
}
```

작업 등록 시점에 MDC를 복사하고, 실행 시점에 복원하고, 완료 시 정리하는 패턴이다.
Virtual Thread에서도 `ThreadLocal` 기반 MDC가 정상 동작하므로, 기존 로깅 인프라를 그대로 사용할 수 있다.

Java 21 미만 환경에서는 `ThreadPoolTaskExecutor`를 사용한다.
이 경우 동시 스트림 수와 AI 응답 시간을 고려하여 `maxPoolSize`를 설정해야 한다.

```java
@Bean(name = "sseTaskExecutor")
public TaskExecutor sseTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(16);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("sse-");
    executor.setTaskDecorator(runnable -> {
        Map<String, String> context = MDC.getCopyOfContextMap();
        return () -> {
            if (context != null) {
                MDC.setContextMap(context);
            }
            try {
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    });
    executor.initialize();
    return executor;
}
```

컨트롤러에서는 `@Qualifier`로 SSE 전용 Executor를 주입받아 사용한다.

```java
@RestController
public class ChatController {

    private final TaskExecutor sseTaskExecutor;

    public ChatController(
        @Qualifier("sseTaskExecutor") TaskExecutor sseTaskExecutor,
        // ... 기타 의존성
    ) {
        this.sseTaskExecutor = sseTaskExecutor;
    }

    @PostMapping(value = "/chat/stream/standalone",
        consumes = MediaType.MULTIPART_FORM_DATA_VALUE,
        produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamChat(...) {
        SseEmitter emitter = new SseEmitter(0L);
        emitter.onTimeout(() -> emitter.complete());
        emitter.onError(error ->
            log.debug("SSE onError (클라이언트 단절): {}", error.getMessage()));

        // SSE 전용 스레드에서 비동기 실행
        sseTaskExecutor.execute(() -> {
            chatSseStreamService.processExternalChatStream(
                request, emitter, chatType, onComplete, onDocumentNotFound);
        });

        return emitter;
    }
}
```

`SseEmitter`를 즉시 반환하고, 실제 스트리밍은 `sseTaskExecutor`에서 비동기로 실행한다.
이렇게 하면 Tomcat 워커 스레드는 즉시 반환되어 다른 요청을 처리할 수 있다.

## 결론

SSE를 운영 환경에서 안정적으로 사용하려면 단순한 `SseEmitter` 생성 이상의 처리가 필요하다.

핵심 설계 원칙을 정리하면 다음과 같다.
- Keepalive는 SSE event가 아닌 comment로 전송하여 비즈니스 로직과 간섭을 없앤다.
- 전체 활성 스트림을 단일 스케줄 태스크에서 순회하여, 스케줄 태스크 수를 동시 스트림 수와 분리한다.
- 재시도는 `PrematureCloseException`(stale connection)에 대해서만, 최대 1회로 제한한다.
- 독립된 `AtomicBoolean` 여러 개 대신 `AtomicReference<StreamState>` 상태 머신으로 유효하지 않은 상태 조합을 원천 방지한다.
- `completeWithError()`는 사용하지 않고, error 이벤트 전송 후 `complete()`를 호출한다.
- 클라이언트 단절과 실제 에러를 구분하여, 단절 시에도 외부 API 응답을 끝까지 수신한다.
- "Connection reset"은 클라이언트 단절 판별에 포함하지 않는다.
- 커넥션 풀을 서비스별로 분리하여 장애 격리를 확보한다.
- `SseStreamRegistry`에 TTL 기반 자동 정리를 적용하여 비정상 종료 시의 누수를 방어한다.
- Java 21+ Virtual Thread로 스레드 풀 크기 관리 없이 SSE 스트리밍을 처리하고, MDC 컨텍스트를 전파하여 로그 추적성을 유지한다.
