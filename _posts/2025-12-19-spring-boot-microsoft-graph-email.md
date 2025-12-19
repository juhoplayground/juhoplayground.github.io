---
layout: post
title: Spring Boot에서 Microsoft Graph API로 이메일 보내기
author: 'Juho'
date: 2025-12-19 09:00:00 +0900
categories: [SpringBoot]
tags: [SpringBoot, Microsoft Graph API, Azure, email, Java, API]
toc: True
---

## 개요

기존 SMTP 방식 대신 Microsoft Graph API를 사용하면 더 안전하고 강력한 방식으로 이메일을 발송할 수 있습니다.  
이 글에서는 Azure Portal에서 앱을 등록하고 Spring Boot 프로젝트에서 Microsoft Graph API를 통해 이메일을 보내는 방법을 단계별로 알아보겠습니다.  

## 왜 Microsoft Graph API를 사용해야 할까?

- 보안: OAuth 2.0 기반 인증으로 안전한 접근  
- 통합: Office 365 생태계와의 완벽한 통합  
- 기능: 단순 메일 발송뿐만 아니라 첨부파일, 회신, 받은편지함 관리 등 다양한 기능  
- 관리: Azure Portal에서 중앙 집중식 권한 관리  

## 1. Azure Portal 설정

### 1.1 Azure Portal 접속 및 앱 등록

먼저 [Azure Portal](https://portal.azure.com){:target="_blank"}에 접속합니다.  

1. Microsoft Entra ID (구 Azure Active Directory) 메뉴로 이동  
2. 좌측 메뉴에서 App registrations 선택  
3. + New registration 클릭  
4. 앱 이름 입력  
5. Supported account types 선택 (보통 "Single tenant" 선택)  
6. Register 클릭  

### 1.2 필수 정보 수집

앱 등록이 완료되면 Overview 페이지에서 다음 정보를 확인할 수 있습니다:  

- 디렉터리(테넌트) ID (Directory/Tenant ID)  
- 애플리케이션(클라이언트) ID (Application/Client ID)  

이 두 값을 복사해서 안전한 곳에 보관합니다.  

### 1.3 클라이언트 시크릿 생성

1. 좌측 메뉴에서 Certificates & secrets 선택  
2. Client secrets 탭에서 + New client secret 클릭  
3. 설명 입력
4. 만료 기간 선택 (6개월, 12개월, 24개월 등)  
5. Add 클릭  
6. 생성된 Value 값을 즉시 복사 (이 페이지를 벗어나면 다시 볼 수 없습니다!)  

### 1.4 API 권한 설정

이메일 발송을 위한 권한을 부여합니다:  

1. 좌측 메뉴에서 API permissions 선택  
2. + Add a permission 클릭  
3. Microsoft Graph 선택  
4. Application permissions 선택 (사용자 로그인 없이 백그라운드에서 작동)  
5. Mail 섹션에서 Mail.Send 권한 찾아서 체크  
6. Add permissions 클릭  
7. 중요: Grant admin consent for [조직명] 버튼 클릭 (관리자 권한 필요)  
   - 이 단계를 건너뛰면 "Insufficient privileges" 오류가 발생합니다  

권한이 정상적으로 부여되면 Status 열에 녹색 체크 표시가 나타납니다.  

## 2. Spring Boot 프로젝트 설정  

### 2.1 build.gradle 의존성 추가  

`build.gradle` 파일에 다음 의존성을 추가합니다:

```gradle
dependencies {
    // 기존 Spring Boot 의존성들...

    // Microsoft Graph API
    implementation 'com.microsoft.azure:msal4j:1.15.1'
    implementation 'com.microsoft.graph:microsoft-graph:6.24.0'
    implementation 'com.azure:azure-core:1.45.0'
}
```

의존성 설명:
- `msal4j`: Microsoft Authentication Library - OAuth 인증 처리
- `microsoft-graph`: Microsoft Graph API Java SDK
- `azure-core`: Azure 공통 라이브러리

### 2.2 application.yml 설정

`src/main/resources/application.yml` 파일에 Azure 설정을 추가합니다  

```yaml
microsoft:
  graph:
    tenant-id: your-tenant-id          # 디렉터리(테넌트) ID
    client-id: your-client-id          # 애플리케이션(클라이언트) ID
    client-secret: your-client-secret  # 클라이언트 시크릿
    from: jhpark@cellkey.co.kr         # 발신 이메일 주소
    subject: Email via Microsoft Graph  # 기본 제목 (선택사항)
    body: This email was sent using Microsoft Graph API.  # 기본 본문 (선택사항)
```

> 보안 주의: 실제 프로덕션 환경에서는 시크릿 값을 환경 변수나 AWS Secrets Manager, Azure Key Vault 등에 저장하고 참조해야 합니다.  

## 3. Spring Boot 코드 구현

### 3.1 설정 클래스 작성

```java
import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Getter
@Setter
@Configuration
@ConfigurationProperties(prefix = "microsoft.graph")
public class MicrosoftGraphConfig {
    private String tenantId;
    private String clientId;
    private String clientSecret;
    private String from;
    private String subject;
    private String body;
}
```

### 3.2 이메일 서비스 구현

```java
import com.azure.core.credential.AccessToken;
import com.azure.core.credential.TokenCredential;
import com.azure.core.credential.TokenRequestContext;
import com.microsoft.graph.authentication.TokenCredentialAuthProvider;
import com.microsoft.graph.models.BodyType;
import com.microsoft.graph.models.EmailAddress;
import com.microsoft.graph.models.ItemBody;
import com.microsoft.graph.models.Message;
import com.microsoft.graph.models.Recipient;
import com.microsoft.graph.requests.GraphServiceClient;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Mono;

import java.util.Arrays;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class MicrosoftGraphEmailService {

    private final MicrosoftGraphConfig config;

    /
     * Microsoft Graph API를 통해 이메일 발송
     *
     * @param to 수신자 이메일 주소
     * @param subject 메일 제목
     * @param bodyContent 메일 본문
     */
    public void sendEmail(String to, String subject, String bodyContent) {
        try {
            // 1. GraphServiceClient 생성
            GraphServiceClient<okhttp3.Request> graphClient = createGraphClient();

            // 2. 메시지 구성
            Message message = buildMessage(to, subject, bodyContent);

            // 3. 이메일 발송
            graphClient.users(config.getFrom())
                    .sendMail(message, false)  // false = 보낸 편지함에 저장 안 함
                    .buildRequest()
                    .post();

            log.info("이메일 발송 성공: {} -> {}", config.getFrom(), to);

        } catch (Exception e) {
            log.error("이메일 발송 실패: {}", e.getMessage(), e);
            throw new RuntimeException("이메일 발송 중 오류가 발생했습니다.", e);
        }
    }

    /
     * 여러 수신자에게 이메일 발송
     */
    public void sendEmailToMultiple(List<String> recipients, String subject, String bodyContent) {
        try {
            GraphServiceClient<okhttp3.Request> graphClient = createGraphClient();
            Message message = buildMessageForMultiple(recipients, subject, bodyContent);

            graphClient.users(config.getFrom())
                    .sendMail(message, false)
                    .buildRequest()
                    .post();

            log.info("이메일 발송 성공: {} -> {}", config.getFrom(), recipients);

        } catch (Exception e) {
            log.error("이메일 발송 실패: {}", e.getMessage(), e);
            throw new RuntimeException("이메일 발송 중 오류가 발생했습니다.", e);
        }
    }

    /
     * GraphServiceClient 생성
     */
    private GraphServiceClient<okhttp3.Request> createGraphClient() {
        // 커스텀 TokenCredential 구현
        TokenCredential tokenCredential = new TokenCredential() {
            @Override
            public Mono<AccessToken> getToken(TokenRequestContext request) {
                return Mono.fromCallable(() -> getAccessToken());
            }
        };

        // TokenCredentialAuthProvider 생성
        TokenCredentialAuthProvider authProvider =
                new TokenCredentialAuthProvider(
                        Arrays.asList("https://graph.microsoft.com/.default"),
                        tokenCredential
                );

        // GraphServiceClient 생성
        return GraphServiceClient.builder()
                .authenticationProvider(authProvider)
                .buildClient();
    }

    /
     * MSAL4J를 사용하여 Access Token 획득
     */
    private AccessToken getAccessToken() {
        try {
            com.microsoft.aad.msal4j.ConfidentialClientApplication app =
                com.microsoft.aad.msal4j.ConfidentialClientApplication.builder(
                        config.getClientId(),
                        com.microsoft.aad.msal4j.ClientCredentialFactory.createFromSecret(config.getClientSecret())
                )
                .authority("https://login.microsoftonline.com/" + config.getTenantId())
                .build();

            com.microsoft.aad.msal4j.ClientCredentialParameters params =
                com.microsoft.aad.msal4j.ClientCredentialParameters.builder(
                        java.util.Collections.singleton("https://graph.microsoft.com/.default")
                )
                .build();

            com.microsoft.aad.msal4j.IAuthenticationResult result = app.acquireToken(params).join();

            return new AccessToken(
                    result.accessToken(),
                    result.expiresOnDate().toInstant().atOffset(java.time.ZoneOffset.UTC)
            );

        } catch (Exception e) {
            log.error("Access Token 획득 실패: {}", e.getMessage(), e);
            throw new RuntimeException("인증 토큰 획득에 실패했습니다.", e);
        }
    }

    /
     * 단일 수신자용 메시지 구성
     */
    private Message buildMessage(String to, String subject, String bodyContent) {
        Message message = new Message();
        message.subject = subject;

        // 본문 설정
        ItemBody body = new ItemBody();
        body.contentType = BodyType.HTML;  // 또는 BodyType.TEXT
        body.content = bodyContent;
        message.body = body;

        // 수신자 설정
        Recipient recipient = new Recipient();
        EmailAddress emailAddress = new EmailAddress();
        emailAddress.address = to;
        recipient.emailAddress = emailAddress;
        message.toRecipients = Arrays.asList(recipient);

        return message;
    }

    /
     * 다중 수신자용 메시지 구성
     */
    private Message buildMessageForMultiple(List<String> recipients, String subject, String bodyContent) {
        Message message = new Message();
        message.subject = subject;

        // 본문 설정
        ItemBody body = new ItemBody();
        body.contentType = BodyType.HTML;
        body.content = bodyContent;
        message.body = body;

        // 여러 수신자 설정
        List<Recipient> recipientList = recipients.stream()
                .map(email -> {
                    Recipient recipient = new Recipient();
                    EmailAddress emailAddress = new EmailAddress();
                    emailAddress.address = email;
                    recipient.emailAddress = emailAddress;
                    return recipient;
                })
                .toList();
        message.toRecipients = recipientList;

        return message;
    }
}
```

### 3.3 컨트롤러 예제

```java
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/email")
@RequiredArgsConstructor
public class EmailController {

    private final MicrosoftGraphEmailService emailService;

    @PostMapping("/send")
    public ResponseEntity<String> sendEmail(@RequestBody EmailRequest request) {
        emailService.sendEmail(
                request.getTo(),
                request.getSubject(),
                request.getBody()
        );
        return ResponseEntity.ok("이메일이 성공적으로 발송되었습니다.");
    }

    @PostMapping("/send-multiple")
    public ResponseEntity<String> sendEmailToMultiple(@RequestBody EmailMultipleRequest request) {
        emailService.sendEmailToMultiple(
                request.getRecipients(),
                request.getSubject(),
                request.getBody()
        );
        return ResponseEntity.ok("이메일이 성공적으로 발송되었습니다.");
    }
}

// DTO 클래스
@Getter
@Setter
class EmailRequest {
    private String to;
    private String subject;
    private String body;
}

@Getter
@Setter
class EmailMultipleRequest {
    private List<String> recipients;
    private String subject;
    private String body;
}
```

## 4. 테스트

### 4.1 Postman으로 테스트

단일 수신자:
```json
POST http://localhost:8080/api/email/send
Content-Type: application/json

{
  "to": "recipient@example.com",
  "subject": "테스트 메일입니다",
  "body": "<h1>안녕하세요</h1><p>Microsoft Graph API로 발송된 메일입니다.</p>"
}
```

다중 수신자:
```json
POST http://localhost:8080/api/email/send-multiple
Content-Type: application/json

{
  "recipients": [
    "user1@example.com",
    "user2@example.com"
  ],
  "subject": "공지사항",
  "body": "<p>전체 공지사항입니다.</p>"
}
```

### 4.2 JUnit 테스트

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class MicrosoftGraphEmailServiceTest {

    @Autowired
    private MicrosoftGraphEmailService emailService;

    @Test
    void testSendEmail() {
        emailService.sendEmail(
                "test@example.com",
                "JUnit 테스트 메일",
                "<h1>테스트</h1><p>정상 작동합니다.</p>"
        );
    }
}
```

## 5. 문제 해결 (Troubleshooting)

### 5.1 "Insufficient privileges to complete the operation"

원인: API 권한이 부여되지 않았거나 Admin Consent가 승인되지 않음

해결:
1. Azure Portal → API permissions 확인
2. "Grant admin consent" 버튼이 클릭되었는지 확인
3. Status가 "Granted for [조직명]"인지 확인

### 5.2 "The tenant for tenant guid 'xxx' does not exist"

원인: Tenant ID가 잘못되었거나 복사 시 공백이 포함됨

해결:
1. Azure Portal에서 Tenant ID 재확인
2. application.yml에서 앞뒤 공백 제거

### 5.3 "Invalid client secret provided"

원인: Client Secret이 만료되었거나 잘못됨

해결:
1. Azure Portal → Certificates & secrets에서 만료 여부 확인
2. 새로운 Client Secret 생성 후 application.yml 업데이트

### 5.4 "ErrorAccessDenied: Access is denied"

원인: 발신 계정(from)에 메일함이 없거나 권한 부족

해결:
1. from에 지정된 이메일이 실제 Office 365 계정인지 확인
2. 해당 계정에 로그인하여 메일함이 활성화되어 있는지 확인

## 6. 보안 Best Practices

### 6.1 시크릿 관리

```yaml
# application.yml - 개발 환경
microsoft:
  graph:
    tenant-id: ${AZURE_TENANT_ID}
    client-id: ${AZURE_CLIENT_ID}
    client-secret: ${AZURE_CLIENT_SECRET}
```

환경 변수로 관리:
```bash
export AZURE_TENANT_ID=your-tenant-id
export AZURE_CLIENT_ID=your-client-id
export AZURE_CLIENT_SECRET=your-client-secret
```

### 6.2 최소 권한 원칙

필요한 권한만 부여:
- 메일 발송만 필요: `Mail.Send`
- 메일 읽기도 필요: `Mail.Read` 추가
- 모든 사용자 메일박스 접근: `Mail.Send.Shared`

### 6.3 로깅 주의사항

```java
// 잘못된 예 - 민감 정보 노출
log.info("Token: {}", accessToken);

// 올바른 예
log.info("이메일 발송 시도: from={}, to={}", from, to);
```

## 7. 고급 기능

### 7.1 첨부파일 추가

```java
import com.microsoft.graph.models.Attachment;
import com.microsoft.graph.models.FileAttachment;

private Message buildMessageWithAttachment(String to, String subject, String bodyContent, byte[] fileBytes, String fileName) {
    Message message = buildMessage(to, subject, bodyContent);

    // 첨부파일 생성
    FileAttachment attachment = new FileAttachment();
    attachment.name = fileName;
    attachment.contentBytes = fileBytes;
    attachment.contentType = "application/octet-stream";

    message.attachments = Arrays.asList(attachment);
    return message;
}
```

### 7.2 HTML 템플릿 사용

```java
import org.springframework.core.io.ClassPathResource;
import java.nio.charset.StandardCharsets;

public String loadEmailTemplate(String templateName, Map<String, String> variables) throws IOException {
    ClassPathResource resource = new ClassPathResource("templates/email/" + templateName + ".html");
    String template = new String(resource.getInputStream().readAllBytes(), StandardCharsets.UTF_8);

    // 변수 치환
    for (Map.Entry<String, String> entry : variables.entrySet()) {
        template = template.replace("{{" + entry.getKey() + "}}", entry.getValue());
    }

    return template;
}
```

### 7.3 CC, BCC 추가

```java
private Message buildMessageWithCcBcc(String to, List<String> ccList, List<String> bccList, String subject, String body) {
    Message message = buildMessage(to, subject, body);

    // CC 추가
    if (ccList != null && !ccList.isEmpty()) {
        message.ccRecipients = ccList.stream()
            .map(this::createRecipient)
            .toList();
    }

    // BCC 추가
    if (bccList != null && !bccList.isEmpty()) {
        message.bccRecipients = bccList.stream()
            .map(this::createRecipient)
            .toList();
    }

    return message;
}

private Recipient createRecipient(String email) {
    Recipient recipient = new Recipient();
    EmailAddress emailAddress = new EmailAddress();
    emailAddress.address = email;
    recipient.emailAddress = emailAddress;
    return recipient;
}
```

## 마무리

Microsoft Graph API를 사용한 이메일 발송 시스템 구축을 완료했습니다. 기존 SMTP 방식에 비해 다음과 같은 장점이 있습니다:  

- 보안: OAuth 2.0 기반 인증으로 비밀번호 노출 위험 없음  
- 확장성: Office 365의 다양한 기능 활용 가능  
- 관리 용이성: Azure Portal에서 중앙 집중식 관리  
- 안정성: Microsoft의 인프라를 활용한 높은 가용성  

프로덕션 환경에 배포하기 전에는 반드시 시크릿 관리, 에러 핸들링, 로깅 전략을 철저히 점검하시기 바랍니다.  

## 참고 자료

- [Microsoft Graph API 공식 문서](https://learn.microsoft.com/en-us/graph/overview)
- [Microsoft Graph Java SDK](https://github.com/microsoftgraph/msgraph-sdk-java)
- [Azure AD 앱 등록 가이드](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)
- [MSAL4J 문서](https://github.com/AzureAD/microsoft-authentication-library-for-java)
