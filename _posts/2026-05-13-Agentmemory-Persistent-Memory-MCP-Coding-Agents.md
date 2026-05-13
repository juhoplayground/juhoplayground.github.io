---
layout: post
title: "agentmemory: AI 코딩 에이전트를 위한 영구 메모리 MCP 서버"
author: 'Juho'
date: 2026-05-13 00:00:00 +0900
categories: [MCP]
tags: [MCP, Agent, Knowledge, Embedding]
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
   - [왜 또 다른 메모리인가](#왜-또-다른-메모리인가)
   - [지원 에이전트](#지원-에이전트)
3. [핵심 내용](#핵심-내용)
   - [벤치마크](#벤치마크)
   - [경쟁 솔루션 비교](#경쟁-솔루션-비교)
   - [빠른 시작](#빠른-시작)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

agentmemory는 Claude Code, Cursor, Gemini CLI, Codex CLI, OpenCode를 비롯해 MCP를 지원하는 모든 에이전트가 같은 메모리를 공유하도록 만드는 영구 메모리 엔진이다.
하나의 서버가 떠 있으면 모든 에이전트가 동일한 기억을 읽고 쓴다.

```bash
npx @agentmemory/agentmemory
```

Karpathy의 LLM Wiki 패턴을 확장해 신뢰도 점수, 라이프사이클, 지식 그래프, 하이브리드 검색을 더한 디자인 문서가 GitHub Gist에서 1,050 스타와 150 포크를 얻었고, agentmemory는 그 구현체다.

## 배경

### 왜 또 다른 메모리인가

CLAUDE.md나 .cursorrules 같은 빌트인 메모리는 200줄 안팎에서 한계를 드러낸다.
파일은 빠르게 낡고, 매 세션마다 같은 아키텍처를 다시 설명하고, 같은 버그를 다시 발견하고, 같은 선호를 다시 가르친다.

agentmemory는 에이전트가 하는 일을 조용히 캡처해 검색 가능한 메모리로 압축하고, 다음 세션이 시작될 때 적절한 컨텍스트를 주입한다.
저자는 다음과 같은 시나리오를 든다.

세션 1에서 JWT 인증을 셋업한다.
세션 2에서 레이트 리미팅을 요청한다.
이때 에이전트는 인증이 `src/middleware/auth.ts`의 jose 미들웨어를 쓰고 있고, 테스트가 토큰 검증을 커버하며, Edge 호환 때문에 jsonwebtoken이 아닌 jose를 골랐다는 사실을 이미 알고 있다.

다시 설명하지 않고, 복사 붙여넣기 없이, 에이전트가 그냥 안다.

### 지원 에이전트

agentmemory는 MCP 또는 REST API를 지원하는 모든 에이전트와 호환된다.

| 에이전트 | 통합 방식 |
|---|---|
| Claude Code | 12 hooks + MCP + skills |
| Cursor | MCP server |
| Gemini CLI | MCP server |
| OpenCode | MCP server |
| Codex CLI | MCP server |
| Cline | MCP server |
| Goose | MCP server |
| Kilo Code | MCP server |
| Aider | REST API |
| Claude Desktop | MCP server |
| Windsurf | MCP server |
| Roo Code | MCP server |
| Claude SDK | AgentSDKProvider |
| 기타 | REST API (104 엔드포인트) |

서버는 한 개고, 메모리는 모든 에이전트가 공유한다.

## 핵심 내용

### 벤치마크

검색 정확도와 토큰 절감 두 축에서 모두 효율을 보고한다.

LongMemEval-S (ICLR 2025, 500 questions) 결과는 다음과 같다.

| System | R@5 | R@10 | MRR |
|---|---|---|---|
| agentmemory | 95.2% | 98.6% | 88.2% |
| BM25-only fallback | 86.2% | 94.6% | 71.5% |

토큰 비용 비교는 다음과 같다.

| 접근 방식 | 연간 토큰 | 연간 비용 |
|---|---|---|
| 전체 컨텍스트 붙여넣기 | 19.5M+ | 컨텍스트 윈도우 초과로 불가능 |
| LLM 요약 | 약 650K | 약 $500 |
| agentmemory | 약 170K | 약 $10 |
| agentmemory + 로컬 임베딩 | 약 170K | $0 |

임베딩 모델은 `all-MiniLM-L6-v2`이며 로컬에서 무료로 동작한다.
세션당 약 1,900 토큰 수준의 메모리 주입 비용이 든다.

### 경쟁 솔루션 비교

mem0, Letta/MemGPT, CLAUDE.md와의 비교를 보면 차별화 지점이 분명하다.

| 항목 | agentmemory | mem0 | Letta / MemGPT | CLAUDE.md |
|---|---|---|---|---|
| 유형 | 메모리 엔진 + MCP 서버 | 메모리 레이어 API | 풀 에이전트 런타임 | 정적 파일 |
| Retrieval R@5 | 95.2% | 68.5% (LoCoMo) | 83.2% (LoCoMo) | N/A (grep) |
| 자동 캡처 | 12 hooks (수동 작업 없음) | 수동 add 호출 | 에이전트 자가 편집 | 수동 편집 |
| 검색 | BM25 + Vector + Graph (RRF 융합) | Vector + Graph | Vector (archival) | 전체를 컨텍스트에 적재 |
| 멀티 에이전트 | MCP + REST + leases + signals | API (조정 없음) | Letta 런타임 내부 한정 | 에이전트별 파일 |
| 외부 종속성 | 없음 (SQLite + iii-engine) | Qdrant / pgvector | Postgres + vector DB | 없음 |
| 메모리 라이프사이클 | 4단계 통합 + 감쇠 + 자동 망각 | 수동 추출 | 에이전트 관리 | 수동 가지치기 |
| 토큰 효율 | 세션당 약 1,900 토큰 ($10/yr) | 통합에 따라 다름 | core memory가 컨텍스트 차지 | 240 obs에서 22K 이상 |
| 실시간 뷰어 | 있음 (port 3113) | 클라우드 대시보드 | 클라우드 대시보드 | 없음 |
| 셀프 호스팅 | 기본 | 선택 | 선택 | 가능 |

핵심은 BM25, 벡터, 그래프 검색을 RRF(Reciprocal Rank Fusion)로 융합하고, 자동 캡처를 12개 훅으로 처리하며, 외부 DB 없이 SQLite와 자체 엔진으로 구동한다는 점이다.

### 빠른 시작

서버 기동과 데모 시드는 두 줄이면 된다.

```bash
# 터미널 1: 서버 시작
npx @agentmemory/agentmemory

# 터미널 2: 샘플 데이터 시드 후 회상 동작 확인
npx @agentmemory/agentmemory demo
```

호환성은 `iii-sdk` `^0.11.0`과 iii-engine v0.11.x 안정 버전을 대상으로 한다.
v0.9.0에서는 [agent-memory.dev](https://agent-memory.dev){:target="_blank"} 랜딩 사이트, 파일시스템 커넥터(`@agentmemory/fs-watcher`), 표준 MCP의 실행 중 서버 프록시화, 모든 삭제 경로에 코드화된 감사 정책이 도입되었다.
작은 Node 프로세스에서 헬스 체크가 `memory_critical`을 잘못 띄우던 이슈도 정리됐다.

## 의미와 시사점

agentmemory가 흥미로운 지점은 두 가지다.

첫째, 메모리를 에이전트가 아니라 메모리 자체가 가진다.
mem0가 메모리 레이어 API에 가깝고 Letta가 메모리를 가진 풀 에이전트 런타임이라면, agentmemory는 어떤 에이전트라도 붙어 쓰는 공유 서버다.
한 명의 사용자가 여러 도구를 옮겨다니는 현 상황에 적합한 모델이다.

둘째, 정적 메모리(CLAUDE.md)의 한계를 정량으로 짚는다.
240 관측치만 쌓여도 22K 토큰을 컨텍스트에 부담시키는 정적 파일과, 세션당 1,900 토큰만 주입하는 검색 기반 메모리의 차이가 분명하다.
하이브리드 검색의 R@5 95.2%는 단순 BM25 폴백 86.2%와도 약 9%포인트 차이가 난다.

물론 평가 데이터셋이 LongMemEval-S로 한정되어 있고, 경쟁 솔루션은 자체 LoCoMo 결과를 보고하는 만큼 직접 비교에는 한계가 있다.

## 결론

agentmemory는 코딩 에이전트의 망각 비용을 정면으로 다룬다.
영구적이고, 에이전트 간 공유 가능하며, 외부 DB 없이 셀프 호스팅되고, 12개 훅으로 자동 캡처된다.

여러 에이전트를 오가는 개발자라면 한 번 띄워두고 시간 변화를 관찰할 만하다.
mem0를 쓰던 사용자도 자동 캡처와 RRF 기반 하이브리드 검색이 자기 업무와 맞는지 비교해볼 가치가 있다.

## Reference

- [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory/)
- [agent-memory.dev](https://agent-memory.dev/)
