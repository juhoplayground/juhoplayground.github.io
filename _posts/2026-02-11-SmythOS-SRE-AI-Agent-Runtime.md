---
layout: post
title: "SmythOS SRE - AI 에이전트를 위한 오픈소스 런타임 환경"
author: 'Juho'
date: 2026-02-11 02:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
2. [SmythOS SRE란](#smythos-sre란)
3. [설계 원칙](#설계-원칙)
4. [아키텍처 구성](#아키텍처-구성)
5. [지원 커넥터](#지원-커넥터)
6. [빠른 시작](#빠른-시작)
7. [코드 예제](#코드-예제)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

프로덕션 수준의 AI 에이전트를 배포하는 것은 여전히 어려운 과제다.
LLM, 벡터 데이터베이스, 스토리지, 캐싱 등 다양한 리소스를 관리해야 하고, 보안과 확장성도 고려해야 한다.
SmythOS SRE(Smyth Runtime Environment)는 이러한 문제를 해결하기 위해 만들어진 오픈소스 AI 에이전트 런타임이다.

## SmythOS SRE란

SmythOS SRE는 프로덕션 AI 에이전트를 위한 오픈소스 런타임 및 SDK다.
"AI 에이전트의 Linux"를 표방하며, 운영체제 커널 아키텍처에서 영감을 받아 설계되었다.
LLM, 벡터 데이터베이스, 스토리지, 캐싱 등 AI 리소스에 대한 OS 수준의 추상화를 제공한다.

### 핵심 철학

| 원칙 | 설명 |
|------|------|
| 간단한 배포 | 프로덕션 AI 에이전트 배포가 로켓 과학처럼 어려워서는 안 됨 |
| 자율성과 제어의 공존 | 에이전트의 자율성과 사용자의 제어가 함께 존재해야 함 |
| 내장 보안 | 보안은 추가 기능이 아닌 기본 내장 기능 |
| 개방형 에이전트 인터넷 | 에이전트 인터넷은 모두에게 열려 있어야 함 |

## 설계 원칙

### 통합 리소스 추상화

SmythOS는 모든 리소스에 대해 통합된 인터페이스를 제공한다.
파일을 로컬, S3, 또는 다른 스토리지에 저장하든 동일한 API를 사용한다.
이 원칙은 스토리지뿐만 아니라 VectorDB, 캐시(Redis, RAM), LLM(OpenAI, Anthropic) 등 모든 서비스에 적용된다.

### 주요 장점

| 특징 | 설명 |
|------|------|
| 에이전트 우선 설계 | AI 에이전트 워크로드를 위해 특별히 구축 |
| 개발자 친화적 | 개발부터 프로덕션까지 확장되는 간단한 SDK |
| 모듈형 아키텍처 | 모든 인프라를 위한 확장 가능한 커넥터 시스템 |
| 프로덕션 준비 | 확장 가능하고 관찰 가능하며 검증됨 |
| 엔터프라이즈 보안 | 내장 접근 제어 및 보안 자격 증명 관리 |

## 아키텍처 구성

SmythOS 저장소는 세 가지 주요 패키지로 구성된다.

### SRE Core (packages/core)

SRE는 SmythOS를 구동하는 핵심 런타임 환경이다.
AI 에이전트 운영 체제의 커널 역할을 한다.

핵심 기능은 다음과 같다.
- 모듈형 아키텍처: 모든 서비스를 위한 플러그형 커넥터
- 보안 우선: 안전한 리소스 접근을 위한 내장 Candidate/ACL 시스템
- 리소스 관리: 지능형 메모리, 스토리지, 컴퓨팅 관리
- 에이전트 오케스트레이션: 완전한 에이전트 생명주기 관리
- 40개 이상의 컴포넌트: AI, 데이터 처리, 통합을 위한 프로덕션 준비 컴포넌트

### SDK (packages/sdk)

SDK는 SRE 런타임 위에 깔끔하고 개발자 친화적인 추상화 계층을 제공한다.
간단하면서도 강력한 기능을 제공하도록 설계되었다.

### CLI (packages/cli)

SRE CLI는 스캐폴딩과 프로젝트 관리를 통해 빠르게 시작할 수 있도록 돕는다.

## 지원 커넥터

### 스토리지

| 커넥터 | 설명 |
|--------|------|
| Local | 로컬 파일 시스템 |
| S3 | AWS S3 스토리지 |
| Google Cloud | Google Cloud Storage |
| Azure | Azure Blob Storage |

### LLM

| 커넥터 | 설명 |
|--------|------|
| OpenAI | GPT-4, GPT-4o 등 |
| Anthropic | Claude 모델 |
| Google AI | Gemini 모델 |
| AWS Bedrock | Amazon Bedrock |
| Groq | Groq 추론 엔진 |
| Perplexity | Perplexity AI |

### VectorDB

| 커넥터 | 설명 |
|--------|------|
| Pinecone | Pinecone 벡터 DB |
| Milvus | Milvus 벡터 DB |
| RAMVec | 인메모리 벡터 저장소 |

### 캐시 및 Vault

| 커넥터 | 설명 |
|--------|------|
| RAM | 인메모리 캐시 |
| Redis | Redis 캐시 |
| JSON File | JSON 파일 기반 Vault |
| AWS Secrets Manager | AWS 시크릿 관리 |
| HashiCorp | HashiCorp Vault |

## 빠른 시작

### CLI 사용 (권장)

```bash
npm i -g @smythos/cli
sre create
```

CLI가 단계별로 안내하여 필요에 맞는 SDK 프로젝트를 생성한다.

### 직접 SDK 설치

```bash
npm install @smythos/sdk
```

## 코드 예제

### .smyth 파일에서 에이전트 로드 및 실행

```typescript
async function main() {
    const agentPath = path.resolve(__dirname, 'my-agent.smyth');

    // 에이전트 워크플로우 가져오기
    const agent = Agent.import(agentPath, {
        model: Model.OpenAI('gpt-4o'),
    });

    // 에이전트에 쿼리하고 전체 응답 받기
    const result = await agent.prompt('Hello, how are you ?');

    console.log(result);
}
```

### 스트림 모드

```typescript
const events = await agent.prompt('Hello, how are you ?').stream();
events.on('content', (text) => {
    console.log('content');
});

events.on('end', /* 종료 처리 */)
events.on('usage', /* 에이전트 사용량 데이터 수집 */)
events.on('toolCall', /* 도구 호출 처리 */)
events.on('toolResult', /* 도구 결과 처리 */)
```

### 채팅 모드

```typescript
const chat = agent.chat();

let result = await chat.prompt("Hello, I'm Smyth")
console.log(result);

result = await chat.prompt('Do you remember my name ?');
console.log(result);

// agent.prompt()와 chat.prompt()의 차이점은
// 후자가 대화를 기억한다는 것
```

### Article Writer 에이전트 예제

```typescript
import { Agent, Model } from '@smythos/sdk';

async function main() {
    // 지능형 에이전트 생성
    const agent = new Agent({
        name: 'Article Writer',
        model: 'gpt-4o',
        behavior: 'You are a copy writing assistant.',
    });

    // 여러 AI 기능을 결합한 커스텀 스킬 추가
    agent.addSkill({
        id: 'AgentWriter_001',
        name: 'WriteAndStoreArticle',
        description: 'Writes an article and stores it',
        process: async ({ topic }) => {
            // VectorDB - 관련 컨텍스트 검색
            const vec = agent.vectordb.Pinecone({
                namespace: 'myNameSpace',
                indexName: 'demo-vec',
                pineconeApiKey: process.env.PINECONE_API_KEY,
                embeddings: Model.OpenAI('text-embedding-3-large'),
            });

            const searchResult = await vec.search(topic, {
                topK: 10,
                includeMetadata: true,
            });
            const context = searchResult
                .map((e) => e?.metadata?.text)
                .join('\n');

            // LLM - 기사 생성
            const llm = agent.llm.OpenAI('gpt-4o-mini');
            const result = await llm.prompt(
                `Write an article about ${topic} using: ${context}`
            );

            // Storage - 기사 저장
            const storage = agent.storage.S3({ /* S3 설정 */ });
            const uri = await storage.write('article.txt', result);

            return `Article stored. URI: ${uri}`;
        },
    });

    const result = await agent.prompt('Write about Sakura trees');
    console.log(result);
}
```

## 내장 보안

보안은 SRE의 핵심 원칙이다.
모든 작업은 Candidate/ACL 시스템을 통한 적절한 권한이 필요하다.
에이전트가 허용된 리소스에만 접근하도록 보장한다.

```typescript
const candidate = AccessCandidate.agent(agentId);
const storage = ConnectorService.getStorageConnector().user(candidate);
await storage.write('data.json', content);
```

## 컴포넌트 시스템

40개 이상의 프로덕션 준비 컴포넌트를 제공한다.

| 카테고리 | 컴포넌트 |
|----------|----------|
| AI/LLM | GenAILLM, ImageGen, LLMAssistant |
| External | APICall, WebSearch, WebScrape, HuggingFace |
| Data | DataSourceIndexer, DataSourceLookup, JSONFilter |
| Logic | LogicAND, LogicOR, Classifier, ForEach |
| Storage | LocalStorage, S3 |
| Code | ECMAScript, ServerlessCode |

## 결론

SmythOS SRE는 AI 에이전트 운영을 위한 포괄적인 런타임 환경을 제공한다.
통합된 리소스 추상화로 인해 다양한 제공자 간에 코드 변경 없이 전환할 수 있다.
내장 보안 시스템으로 에이전트의 리소스 접근을 안전하게 제어한다.
MIT 라이선스로 오픈소스 공개되어 있으며, 로컬, 클라우드, 엣지 환경 어디서든 실행할 수 있다.

## Reference

- [SmythOS SRE GitHub](https://github.com/SmythOS/sre/)
- [SDK Documentation](https://smythos.github.io/sre/sdk/)
- [SRE Core Documentation](https://smythos.github.io/sre/core/)
