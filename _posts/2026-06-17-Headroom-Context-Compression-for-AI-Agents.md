---
layout: post
title: "Headroom: AI 에이전트를 위한 컨텍스트 압축 레이어"
author: 'Juho'
date: 2026-06-17 00:00:00 +0900
categories: [AI]
tags: [Agent, LLM, Caching]
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
2. [해결하려는 문제](#해결하려는-문제)
3. [아키텍처](#아키텍처)
   - [핵심 구성 요소](#핵심-구성-요소)
   - [출력 토큰 절감](#출력-토큰-절감)
   - [실패 학습](#실패-학습)
4. [설치와 사용법](#설치와-사용법)
   - [설치](#설치)
   - [사용 예시](#사용-예시)
   - [설정](#설정)
5. [벤치마크와 호환성](#벤치마크와-호환성)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Headroom은 AI 에이전트 워크플로의 토큰 소비를 줄이기 위해 설계된 컨텍스트 압축 레이어다.
도구 출력, 로그, RAG 청크, 파일, 대화 기록을 언어 모델에 전달하기 전에 압축한다.
출력 품질을 유지하면서 토큰 소비를 60~95% 줄이는 것을 목표로 한다.

[Headroom GitHub 저장소](https://github.com/chopratejas/headroom){:target="_blank"}는 라이브러리, HTTP 프록시, 에이전트 래퍼, MCP 서버 등 여러 배포 모드를 제공한다.
라이선스는 Apache 2.0이며, Python 3.10 이상을 요구한다.

## 해결하려는 문제

AI 에이전트는 검색 결과, 로그, 파일 내용, 도구 출력 등 대량의 컨텍스트를 LLM에 전송한다.
이 과정에서 비싼 토큰이 소비된다.

Headroom은 이 컨텍스트를 가로채 지능적으로 압축한다.
정확도를 희생하지 않으면서 비용과 지연 시간을 줄이는 것이 핵심 목표다.

## 아키텍처

압축 파이프라인은 여러 단계를 거쳐 흐른다.
Claude Code, Cursor, Aider 등 에이전트나 앱에서 보내는 프롬프트, 도구 출력, 로그, RAG 결과, 파일이 입력으로 들어온다.
Headroom은 로컬에서 이를 처리한 뒤, 압축된 프롬프트와 검색 도구를 Anthropic, OpenAI, Bedrock 같은 LLM 제공자에게 전달한다.

```
Your agent/app (Claude Code, Cursor, Aider, etc.)
    | (prompts, tool outputs, logs, RAG results, files)
    v
+-----------------------------------------+
| Headroom (local processing)             |
| - CacheAligner                          |
| - ContentRouter                         |
|   - SmartCrusher (JSON)                 |
|   - CodeCompressor (AST)                |
|   - Kompress-base (text/HF model)       |
| - CCR (reversible compression)          |
+-----------------------------------------+
    | (compressed prompt + retrieval tool)
    v
LLM provider (Anthropic, OpenAI, Bedrock, etc.)
```

### 핵심 구성 요소

Headroom은 콘텐츠 유형에 따라 서로 다른 압축 알고리즘을 적용한다.
JSON, 코드(AST 기반), 산문(prose) 각각에 맞는 방식을 사용한다.

아래 표는 주요 구성 요소를 정리한 것이다.

| 구성 요소 | 역할 |
|-----------|------|
| ContentRouter | 콘텐츠 유형을 감지하고 적절한 압축기를 선택 |
| SmartCrusher | 배열, 중첩 객체, 혼합 타입을 다루는 범용 JSON 압축 |
| CodeCompressor | Python, JavaScript, Go, Rust, Java, C++의 AST 인식 압축 |
| Kompress-base | 에이전트 트레이스로 학습된 HuggingFace 모델 |
| CacheAligner | 제공자 KV 캐시 적중을 위해 프리픽스를 안정화 |
| IntelligentContext | 학습된 중요도 기반 점수로 컨텍스트를 맞춤 |

이미지 압축은 ML 라우터를 통해 40~90% 절감을 제공한다.
CCR(가역 압축)은 원본 콘텐츠를 로컬에 캐시해 필요할 때 다시 가져올 수 있게 한다.
Claude, Copilot, Codex, Gemini 사이에서 압축된 컨텍스트를 공유하는 크로스 에이전트 메모리도 지원한다.

### 출력 토큰 절감

Headroom은 입력 토큰뿐 아니라 출력 토큰도 줄인다.
환경 변수로 출력 트리밍을 활성화한 뒤 프록시를 실행한다.

```bash
export HEADROOM_OUTPUT_SHAPER=1         # Enable output trimming
headroom proxy --port 8787
```

verbosity steering은 시스템 프롬프트에 간결함 가이드를 덧붙이며, 이때 프롬프트 캐시 적중을 유지한다.
effort routing은 도구 결과 후 재개하는 일상적 단계에서는 모델의 사고를 낮추고, 새로운 질문에는 전체 노력을 유지한다.

최적의 간결함을 학습하고 절감량을 측정할 수 있다.

```bash
headroom learn --verbosity              # Preview (dry run)
headroom learn --verbosity --apply      # Save preferences
headroom output-savings
# Output: Reduction: 31.7%  (95% CI 27.7% ... 35.7%)  [estimated]
```

측정된 수치를 얻으려면 컨트롤 그룹을 사용한다.
환경 변수 HEADROOM_OUTPUT_HOLDOUT=0.1은 10%를 가공하지 않은 베이스라인으로 둔다.

### 실패 학습

headroom learn은 실패한 세션을 분석해 자동으로 교정한다.
분석 결과를 에이전트 설정 파일(CLAUDE.md, AGENTS.md)에 실행 가능한 교정 사항으로 기록한다.

```bash
headroom learn                          # Preview corrections
headroom learn --apply                  # Save to CLAUDE.md / AGENTS.md
headroom learn --verbosity              # Optimize terseness
```

## 설치와 사용법

### 설치

Python 환경에서는 pip로 설치한다.
필요한 기능만 골라 설치할 수 있는 extras를 제공한다.

```bash
pip install "headroom-ai[all]"          # Everything
pip install "headroom-ai[proxy]"        # Proxy only
pip install "headroom-ai[mcp]"          # MCP server
```

TypeScript/Node와 Docker도 지원한다.

```bash
npm install headroom-ai
docker pull ghcr.io/chopratejas/headroom:latest
```

SSL 검사 환경에서는 빌드 전에 Rust를 먼저 설치해야 한다.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup default stable
```

### 사용 예시

코드 변경 없이 코딩 에이전트를 감쌀 수 있다.

```bash
headroom wrap claude                    # Claude Code
headroom wrap codex                     # Codex
headroom wrap cursor                    # Cursor
headroom wrap aider                     # Aider
headroom copilot-auth login
headroom wrap copilot --subscription -- --model gpt-4o
```

언어에 구애받지 않는 HTTP 프록시 모드도 있다.

```bash
headroom proxy --port 8787
# Point your app to http://localhost:8787
```

라이브러리로 직접 통합할 수도 있다.

```python
from headroom import compress

compressed = compress(messages, model="claude-3-5-sonnet")
```

```typescript
import { compress } from 'headroom-ai';

const compressed = await compress(messages, { model: 'gpt-4o' });
```

Anthropic, OpenAI SDK 래퍼와 LangChain 통합도 제공한다.

```python
# Anthropic
from headroom import withHeadroom
from anthropic import Anthropic

client = withHeadroom(Anthropic())

# OpenAI
from openai import OpenAI
client = withHeadroom(OpenAI())

# LangChain
from headroom.integrations.langchain import HeadroomChatModel

llm = HeadroomChatModel(your_llm)
```

MCP 서버는 headroom_compress, headroom_retrieve, headroom_stats 도구를 제공한다.

```bash
headroom mcp install
```

### 설정

주요 환경 변수는 다음과 같다.

| 환경 변수 | 설명 |
|-----------|------|
| HEADROOM_OUTPUT_SHAPER=1 | 출력 트리밍 활성화 |
| HEADROOM_OUTPUT_HOLDOUT=0.1 | 컨트롤 그룹 비율 |
| HEADROOM_UPDATE_CHECK=off | 업데이트 알림 생략 |
| HEADROOM_EMBEDDER_RUNTIME=pytorch_mps | Apple GPU 오프로드 |
| HF_HUB_OFFLINE=1 | 캐시된 모델 사용 |
| GITHUB_COPILOT_TOKEN | Copilot 인증 토큰 |

extras로 proxy, mcp, ml, code, memory, relevance, image, langchain, evals 등을 granular하게 설치할 수 있다.

## 벤치마크와 호환성

실제 워크로드에서의 압축 결과는 아래와 같다.

| 워크로드 | Before | After | Savings |
|----------|--------|-------|---------|
| Code search (100 results) | 17,765 | 1,408 | 92% |
| SRE incident debugging | 65,694 | 5,118 | 92% |
| GitHub issue triage | 54,174 | 14,761 | 73% |
| Codebase exploration | 78,502 | 41,254 | 47% |

정확도 보존을 측정한 결과는 다음과 같다.

| Benchmark | Category | N | Baseline | Headroom | Delta |
|-----------|----------|---|----------|----------|-------|
| GSM8K | Math | 100 | 0.870 | 0.870 | 0.000 |
| TruthfulQA | Factual | 100 | 0.530 | 0.560 | +0.030 |
| SQuAD v2 | QA | 100 | — | 97% | 19% 압축 |
| BFCL | Tools | 100 | — | 97% | 32% 압축 |

벤치마크는 다음 명령으로 재현할 수 있다.

```bash
python -m headroom.evals suite --tier 1
```

지원하는 에이전트와 제공자도 폭넓다.

| 에이전트 | 비고 |
|----------|------|
| Claude Code | memory, code-graph 옵션 포함 |
| Codex | Claude와 메모리 공유 |
| Cursor | 수동 설정용 config 출력 |
| Aider | 프록시 자동 시작 |
| Copilot CLI | 프록시 자동 시작, 구독 모드 |
| Any OpenAI-compatible | headroom proxy 경유 |

지원 제공자는 Anthropic(Claude), OpenAI(GPT-4, GPT-4o), Google Gemini, AWS Bedrock, 그리고 OpenAI 호환 API 전반이다.

## 결론

Headroom은 에이전트가 LLM에 보내는 컨텍스트를 로컬에서 압축해 입력과 출력 토큰을 모두 줄인다.
콘텐츠 유형별 압축, 가역 압축(CCR), 크로스 에이전트 메모리, 캐시 정렬, 실패 학습을 하나의 레이어로 묶었다.
라이브러리, 프록시, 에이전트 래퍼, MCP 서버 등 다양한 배포 모드를 제공해 기존 워크플로에 코드 변경 없이 붙일 수 있다.
벤치마크상 정확도를 거의 그대로 유지하면서 실제 워크로드에서 47~92%의 토큰 절감을 보고한다.

## Reference

- [Headroom: Context Compression for AI Agents (GitHub)](https://github.com/chopratejas/headroom/)
- [Headroom Documentation](https://headroom-docs.vercel.app/docs/)
- [Kompress-v2-base model (HuggingFace)](https://huggingface.co/chopratejas/kompress-v2-base/)
