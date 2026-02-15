---
layout: post
title: "Google Developer Knowledge API와 MCP 서버 : AI 어시스턴트를 위한 공식 문서 게이트웨이"
author: 'Juho'
date: 2026-02-15 02:00:00 +0900
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
2. [Developer Knowledge API란](#developer-knowledge-api란)
3. [MCP 서버와 도구](#mcp-서버와-도구)
4. [REST API 엔드포인트](#rest-api-엔드포인트)
5. [설정 가이드](#설정-가이드)
6. [AI 도구별 설정 방법](#ai-도구별-설정-방법)
7. [사용 사례](#사용-사례)
8. [알려진 제한사항](#알려진-제한사항)
9. [향후 계획](#향후-계획)
10. [Reference](#reference)

## 개요

Google이 Developer Knowledge API와 MCP(Model Context Protocol) 서버의 공개 프리뷰를 발표했다.
이 서비스는 AI 어시스턴트가 Google의 공식 개발자 문서에 직접 접근할 수 있게 해주는 기계 판독 가능한 게이트웨이이다.
Firebase, Android, Google Cloud, Maps 등 다양한 플랫폼의 공식 문서를 마크다운 형식으로 제공한다.
AI 기반 개발 도구에서 발생하는 정보 부정확성 문제를 해결하기 위해 설계되었다.

## Developer Knowledge API란

Developer Knowledge API는 Google의 공개 개발자 문서에 대한 프로그래매틱 접근을 제공하는 API이다.
애플리케이션과 워크플로에 통합할 수 있도록 설계되었다.

### 핵심 특징

| 특징 | 설명 |
|------|------|
| 문서 형식 | 기계 판독 가능한 마크다운(Markdown) |
| 문서 범위 | Firebase, Android, Google Cloud, Maps 등 |
| 재색인 주기 | 공개 프리뷰 기간 중 24시간 이내 |
| 인증 방식 | API 키 기반 |
| API 버전 | v1alpha (실험적) |

### 문서 커버리지

API가 색인하는 문서 소스는 다음과 같다.

- firebase.google.com
- developer.android.com
- docs.cloud.google.com
- 기타 Google 개발자 플랫폼 문서

공개된 페이지만 포함되며, GitHub, OSS 사이트, 블로그, YouTube 콘텐츠는 포함되지 않는다.

## MCP 서버와 도구

Google Developer Knowledge MCP 서버는 원격(remote) MCP 서버로 동작한다.
AI 도구가 Google의 공식 개발자 문서를 검색하고 조회할 수 있게 해준다.

### MCP 도구 목록

MCP 서버는 세 가지 도구를 제공한다.

| 도구 | 설명 |
|------|------|
| search_documents | 쿼리를 기반으로 관련 페이지와 스니펫을 검색 |
| get_document | search_documents 결과의 parent를 사용하여 문서 전체 내용을 조회 |
| batch_get_documents | search_documents 결과의 여러 parent를 사용하여 다수 문서의 전체 내용을 일괄 조회 |

일반적인 워크플로는 먼저 `search_documents`로 관련 문서를 찾은 뒤, `get_document` 또는 `batch_get_documents`로 전체 내용을 가져오는 방식이다.

## REST API 엔드포인트

Developer Knowledge API는 REST v1alpha 버전으로 세 가지 메서드를 제공한다.

### documents 리소스 메서드

| 메서드 | 경로 |
|--------|------|
| searchDocumentChunks | /knowledge/reference/rest/v1alpha/documents/searchDocumentChunks |
| get | /knowledge/reference/rest/v1alpha/documents/get |
| batchGet | /knowledge/reference/rest/v1alpha/documents/batchGet |

`searchDocumentChunks`는 쿼리를 기반으로 관련 페이지 URI와 콘텐츠 스니펫을 반환한다.
`get`과 `batchGet`은 검색 결과의 전체 콘텐츠를 마크다운 형식으로 가져온다.
인증은 `key` 쿼리 파라미터로 API 키를 전달하는 방식을 사용한다.

## 설정 가이드

MCP 서버를 사용하려면 세 단계의 설정이 필요하다.

### 1단계 : API 키 생성

Google Cloud Console에서 API 키를 생성한다.

**Google Cloud Console을 통한 방법 :**

1. Google APIs 라이브러리에서 Developer Knowledge API 페이지를 연다.
2. 올바른 프로젝트가 선택되었는지 확인한다.
3. Enable을 클릭한다. (IAM 역할 불필요)
4. Credentials 페이지로 이동한다.
5. Create > API key를 클릭한다.
6. 키 이름을 지정하고 편집한다.
7. API restrictions에서 Restrict key를 선택한다.
8. Developer Knowledge API를 활성화한다.
9. 저장 후 키를 기록한다.

**gcloud CLI를 통한 방법 :**

```bash
gcloud services enable developerknowledge.googleapis.com --project=PROJECT_ID
gcloud services api-keys create --project=PROJECT_ID --display-name="DK API Key"
```

### 2단계 : MCP 서버 활성화

```bash
gcloud beta services mcp enable developerknowledge.googleapis.com --project=PROJECT_ID
```

### 3단계 : AI 도구에 설정

각 AI 도구의 설정 파일에 MCP 서버 정보를 추가한다.
도구별 상세 설정은 다음 섹션에서 설명한다.

## AI 도구별 설정 방법

### Claude Code

```bash
claude mcp add google-dev-knowledge \
  --transport http \
  https://developerknowledge.googleapis.com/mcp \
  --header "X-Goog-Api-Key: YOUR_API_KEY"
```

### Gemini CLI

```bash
gemini mcp add -t http \
  -H "X-Goog-Api-Key: YOUR_API_KEY" \
  google-developer-knowledge \
  https://developerknowledge.googleapis.com/mcp \
  --scope user
```

### Antigravity

`mcp_config.json` 파일을 편집한다.

```json
{
  "mcpServers": {
    "google-developer-knowledge": {
      "serverUrl": "https://developerknowledge.googleapis.com/mcp",
      "headers": {
        "X-Goog-Api-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### Gemini Code Assist

`.gemini/settings.json` 또는 `~/.gemini/settings.json` 파일을 편집한다.

```json
{
  "mcpServers": {
    "google-developer-knowledge": {
      "httpUrl": "https://developerknowledge.googleapis.com/mcp",
      "headers": {
        "X-Goog-Api-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### Firebase Studio

`.idx/mcp.json` 파일을 생성한다.

```json
{
  "mcpServers": {
    "google-developer-knowledge": {
      "url": "https://developerknowledge.googleapis.com/mcp",
      "headers": {
        "X-Goog-Api-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### Cursor

`.cursor/mcp.json` 또는 `~/.cursor/mcp.json` 파일을 편집한다.

```json
{
  "mcpServers": {
    "google-developer-knowledge": {
      "url": "https://developerknowledge.googleapis.com/mcp",
      "headers": {
        "X-Goog-Api-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### GitHub Copilot (VS Code)

`.vscode/mcp.json` 파일을 편집한다.

```json
{
  "servers": {
    "google-developer-knowledge": {
      "url": "https://developerknowledge.googleapis.com/mcp",
      "headers": {
        "X-Goog-Api-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### Windsurf

`~/.codeium/windsurf/mcp_config.json` 파일을 편집한다.

```json
{
  "mcpServers": {
    "google-developer-knowledge": {
      "url": "https://developerknowledge.googleapis.com/mcp",
      "headers": {
        "X-Goog-Api-Key": "YOUR_API_KEY"
      }
    }
  }
}
```

### 설정 확인

설정이 완료되면 다음과 같은 프롬프트로 테스트한다.
"How do I list Cloud Storage buckets?"라고 질문했을 때 도구 호출이 나타나면 설정이 성공한 것이다.

## 사용 사례

### 구현 가이드

Firebase에서 푸시 알림을 구현하는 최적의 방법 등 구현 관련 질문에 대해 공식 문서를 기반으로 답변을 제공한다.

### 오류 해결

`ApiNotActivatedMapError` 같은 특정 오류의 해결 방법을 공식 문서에서 검색하여 정확한 트러블슈팅 정보를 제공한다.

### 서비스 비교 분석

Cloud Run과 Cloud Functions의 차이점처럼 Google 서비스 간 비교 분석을 공식 문서 기반으로 수행한다.

### AI 코딩 어시스턴트 통합

IDE에 통합된 AI 어시스턴트가 코딩 중 실시간으로 최신 공식 문서를 참조하여 정확한 코드 제안을 제공한다.

## 알려진 제한사항

현재 공개 프리뷰 버전에서 알려진 제한사항은 다음과 같다.

| 제한사항 | 설명 |
|----------|------|
| 공개 페이지만 포함 | 비공개 문서, GitHub, OSS 사이트, 블로그, YouTube 콘텐츠는 제외 |
| 영어만 지원 | 현재 영어 결과만 반환 |
| 비구조적 마크다운 | 현재는 비구조적 마크다운만 제공하며, 코드 샘플이나 API 참조 같은 구조적 콘텐츠는 향후 추가 예정 |
| 마크다운 변환 이슈 | 소스 HTML에서 변환된 마크다운에 간혹 불일치나 서식 문제 발생 가능 |
| 네트워크 의존 | 인터넷 연결과 유효한 API 키 설정 필요 |

Model Armor 사용자는 PIJB 필터를 HIGH_AND_ABOVE로 설정하여 오탐(false positive)을 줄일 수 있다.

## 향후 계획

Google은 정식 출시(GA)를 향해 다음과 같은 개선을 계획하고 있다.

**구조적 콘텐츠 지원** : 코드 샘플 객체와 API 참조 엔티티 같은 구조적 콘텐츠를 추가할 예정이다.

**문서 커버리지 확대** : 더 많은 Google 플랫폼과 서비스의 문서를 포함할 계획이다.

**재색인 지연 감소** : 문서 업데이트 후 색인에 반영되는 시간을 단축할 예정이다.

Developer Knowledge API는 AI 어시스턴트가 Google 공식 문서에 직접 접근할 수 있게 함으로써, 할루시네이션 없는 정확한 개발 정보를 제공하는 중요한 인프라가 될 것으로 보인다.

## Reference

- [Introducing the Developer Knowledge API and MCP Server - Google Developers Blog](https://developers.googleblog.com/introducing-the-developer-knowledge-api-and-mcp-server/)
- [Developer Knowledge API - Google for Developers](https://developers.google.com/knowledge/api)
- [Connect to the Developer Knowledge MCP server - Google for Developers](https://developers.google.com/knowledge/mcp)
