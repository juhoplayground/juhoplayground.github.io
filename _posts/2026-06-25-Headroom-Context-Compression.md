---
layout: post
title: "Headroom: AI 에이전트 토큰을 60-95% 줄이는 컨텍스트 압축 도구"
author: 'Juho'
date: 2026-06-25 00:00:00 +0900
categories: [LLM]
tags: [LLM, Context, Caching]
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
2. [작동 원리](#작동-원리)
   - [전용 압축기 구성](#전용-압축기-구성)
   - [가역 압축과 캐시 정렬](#가역-압축과-캐시-정렬)
3. [사용 방법](#사용-방법)
   - [라이브러리](#라이브러리)
   - [프록시 서버](#프록시-서버)
   - [에이전트 래핑](#에이전트-래핑)
   - [MCP 서버](#mcp-서버)
4. [벤치마크 수치](#벤치마크-수치)
5. [추가 기능과 설정](#추가-기능과-설정)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Headroom은 AI 에이전트와 LLM의 토큰 사용량을 줄이는 컨텍스트 압축 도구다.
프로젝트 설명에 따르면 답변 품질을 유지하면서 토큰 사용량을 60-95% 절감한다.
도구 출력(tool output), 로그, RAG 청크, 파일 등의 콘텐츠가 언어 모델에 도달하기 전에 이를 가공하는 방식으로 동작한다.

핵심은 모델이 실제로 필요로 하지 않는 군더더기를 모델에 보내기 직전에 걸러내는 것이다.
처리는 로컬 우선(local-first)으로 이루어져 데이터가 기기 안에 머무른다.
라이선스는 Apache 2.0이며, 저장소 언어 구성은 Python 79.4%, Rust 16%, TypeScript 2.6%로 이루어져 있다.

## 작동 원리

Headroom은 콘텐츠 종류에 따라 전용 압축기(specialized compressor)로 라우팅한다.
모든 콘텐츠를 동일하게 다루지 않고, JSON인지 코드인지 일반 텍스트인지에 맞춰 서로 다른 압축 경로를 거치게 한다.

### 전용 압축기 구성

다음은 README가 명시하는 압축기 구성이다.

| 압축기 | 역할 |
|--------|------|
| SmartCrusher | JSON과 구조화된 데이터를 처리 |
| CodeCompressor | AST 분석으로 프로그래밍 언어를 처리 |
| Kompress-base | 텍스트 압축용으로 학습된 HuggingFace 모델 |
| CacheAligner | 프로바이더 KV 캐시 적중을 위해 프리픽스를 안정화 |
| CCR | 가역 압축. 원본을 로컬에 저장해 필요 시 재조회 |

SmartCrusher는 API 응답처럼 구조화된 JSON을 다루고, CodeCompressor는 소스 코드를 AST 수준에서 분석해 압축한다.
Kompress-base는 일반 텍스트 압축을 위해 별도로 학습된 모델이다.

### 가역 압축과 캐시 정렬

CacheAligner는 프리픽스를 안정화해 프로바이더의 KV 캐시 적중률을 높인다.
프리픽스가 흔들리지 않으면 동일한 앞부분에 대한 캐시 재사용이 가능해진다.

CCR(reversible compression)은 가역 압축으로, 원본을 로컬에 캐시해 두고 필요할 때 재조회한다.
LLM이 전체 콘텐츠를 다시 필요로 하면 MCP 도구를 통해 설정된 TTL(time-to-live) 안에서 저장된 원본에 접근한다.
즉, 압축은 정보 손실을 강제하지 않으며 원본은 온디맨드로 되살릴 수 있다.

## 사용 방법

Headroom은 네 가지 배포 형태를 제공한다.
직접 함수를 호출하는 라이브러리, 코드 변경이 필요 없는 프록시, 한 번의 명령으로 에이전트를 감싸는 래핑, 그리고 MCP 서버다.

설치 명령은 환경에 따라 다음과 같다.

```bash
pip install "headroom-ai[all]"          # Python
npm install headroom-ai                 # TypeScript
docker pull ghcr.io/chopratejas/headroom:latest
```

### 라이브러리

Python과 TypeScript에서 직접 함수를 호출하는 방식이다.
Python에서는 `from headroom import compress`로 가져와 `messages`와 `model` 파라미터를 받는 형태로 사용한다.

```python
from headroom import compress

compressed = compress(messages, model=...)
```

TypeScript에서는 async API로 호출한다.

```typescript
const compressed = await compress(messages, { model });
```

### 프록시 서버

프록시는 드롭인(drop-in) 서버로, 코드 변경 없이 어떤 언어에서도 동작한다.
다음 명령으로 포트 8787에서 프록시를 띄운다.

```bash
headroom proxy --port 8787
```

프록시는 요청을 가로채 전달하기 전에 압축한다.
OpenAI 호환 클라이언트나 프로바이더별 클라이언트 모두에 대해 드롭인으로 동작한다.

라이브 대시보드는 프록시가 8787에서 동작 중일 때 사용할 수 있다.

```bash
headroom dashboard          # 8787에서 프록시 실행 중일 때 사용
```

README는 프록시가 무상태(stateless) 모드와 호환되며, 루프백 `POST /admin/runtime-env`를 통한 핫싱크(hot-sync) 시 재시작 없이 요청 누락이나 캐시 손실 없이 동작한다고 명시한다.

### 에이전트 래핑

한 번의 명령으로 코딩 에이전트를 감쌀 수 있다.

```bash
headroom wrap claude|codex|cursor|aider|copilot|opencode
```

지원 대상은 Claude Code, Codex, Cursor, Aider, Copilot CLI, OpenCode이며, 프록시를 통하면 모든 OpenAI 호환 클라이언트와도 연동된다.

빠른 시작 흐름은 다음과 같다.

```bash
headroom wrap claude          # 코딩 에이전트 래핑
headroom perf                 # 절감 효과 확인
headroom dashboard            # 라이브 대시보드
```

### MCP 서버

MCP 서버는 다음 세 가지 도구를 제공한다.

| 도구 | 설명 |
|------|------|
| headroom_compress | 콘텐츠를 압축 |
| headroom_retrieve | 저장된 원본을 재조회 |
| headroom_stats | 압축 통계를 조회 |

설치는 다음 명령으로 한다.

```bash
headroom mcp install
```

LLM이 압축된 콘텐츠의 전체 원본이 필요할 때 `headroom_retrieve`를 호출해 설정된 TTL 안에서 저장된 원본에 접근한다.

## 벤치마크 수치

README는 실제 작업에 대해 측정한 토큰 절감 수치를 제시한다.

### 입력 토큰 절감

다음은 벤치마크된 입력 토큰 절감 결과다.

| 작업 | 압축 전 | 압축 후 | 절감률 |
|------|---------|---------|--------|
| 코드 검색(결과 100개) | 17,765 토큰 | 1,408 토큰 | 92% |
| GitHub 이슈 분류 | 54,174 토큰 | 14,761 토큰 | 73% |
| 코드베이스 탐색 | 78,502 토큰 | 41,254 토큰 | 47% |

코드 검색 같은 반복적이고 구조화된 출력에서 92%로 절감폭이 가장 컸다.
콘텐츠가 다양하고 비정형일수록 절감률은 47% 수준으로 내려간다.

### 출력 토큰 절감

Headroom은 입력뿐 아니라 모델 응답 자체도 줄인다.
장황함 조절(verbosity steering)과 effort 라우팅을 통해 모델 응답을 다듬으며, 절감 추정치는 95% 신뢰구간 기준 27.7%에서 35.7%로 보고된다.

## 추가 기능과 설정

Headroom은 압축 외에도 여러 기능을 제공한다.

로컬 우선 처리로 데이터가 기기에 머무르고, 교차 에이전트 메모리 공유(cross-agent memory sharing)에 자동 중복 제거(auto-deduplication)가 적용된다.
공유 메모리는 `SharedContext().put`과 `.get` API로 다룬다.

`headroom learn`은 실패한 세션을 분석해 교정 내용을 추출한다.

```bash
headroom learn                      # 실패한 세션 분석
headroom learn --verbosity          # 간결도 수준 미리보기
headroom learn --verbosity --apply  # 자동 적용
```

추출된 교정 내용은 `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`에 기록된다.

출력 형태를 조정하는 환경 변수도 제공한다.

```bash
export HEADROOM_OUTPUT_SHAPER=1
export HEADROOM_OUTPUT_HOLDOUT=0.1   # 10% 대조군
```

모델 및 런타임 관련 환경 변수는 다음과 같다.

```bash
HEADROOM_EMBEDDER_RUNTIME=pytorch_mps   # Apple GPU 오프로드
HF_HUB_OFFLINE=1                        # 오프라인 HuggingFace 모드
HF_ENDPOINT=<mirror>                    # 커스텀 모델 미러
```

이 밖에 사용 가능한 CLI 명령은 다음과 같다.

```bash
headroom perf                    # 압축 절감 확인
headroom output-savings          # 출력 절감 추정
headroom copilot-auth login      # GitHub Copilot 구독 인증
headroom update                  # 제자리 업그레이드
headroom update --check          # 업데이트 확인
headroom update --pre            # 프리릴리스 포함
```

## 결론

Headroom은 도구 출력, 로그, RAG 청크, 파일을 모델에 보내기 직전에 압축해 입력 토큰을 60-95% 줄이는 것을 목표로 한다.
JSON, 코드, 텍스트를 각각 SmartCrusher, CodeCompressor, Kompress-base로 분리 처리하고, CacheAligner로 KV 캐시 적중을 노리며, CCR로 원본을 로컬에 보존해 가역성을 확보한다.
라이브러리, 프록시, 에이전트 래핑, MCP 서버라는 네 가지 형태로 도입할 수 있어 코드 변경 없이도 적용 가능하다.
실측 벤치마크에서 코드 검색은 92%, 이슈 분류는 73% 절감을 보였고, 출력 토큰도 27.7-35.7% 줄였다.

## Reference

- [Headroom (GitHub)](https://github.com/chopratejas/headroom/)
