---
layout: post
title: "Codebase Memory MCP: 코드베이스를 영속적 지식 그래프로 인덱싱하는 코드 인텔리전스 서버"
author: 'Juho'
date: 2026-06-25 00:00:00 +0900
categories: [MCP]
tags: [MCP, Knowledge, Agent]
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
2. [작동 방식](#작동-방식)
   - [3계층 아키텍처](#3계층-아키텍처)
   - [지식 그래프 데이터 모델](#지식-그래프-데이터-모델)
3. [성능과 토큰 효율](#성능과-토큰-효율)
4. [설치와 연동](#설치와-연동)
   - [설치 방법](#설치-방법)
   - [MCP 연동 설정](#mcp-연동-설정)
5. [사용법](#사용법)
   - [제공 도구](#제공-도구)
   - [CLI와 설정](#cli와-설정)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

codebase-memory-mcp는 코드베이스 전체를 영속적인 지식 그래프로 인덱싱하는 고성능 코드 분석 엔진이다.
프로젝트는 스스로를 "AI 코딩 에이전트를 위한 가장 빠르고 효율적인 코드 인텔리전스 엔진"이라고 소개한다.
런타임 의존성이 없는 단일 정적 바이너리로 동작하며, Model Context Protocol(MCP)을 통해 구조적 코드 쿼리를 제공한다.

핵심 목적은 함수, 클래스, 임포트, 콜 체인, HTTP 라우트, 서비스 간 의존성 같은 코드 구조를 탐색 가능한 그래프로 만드는 것이다.
이를 통해 AI 에이전트가 파일을 하나씩 열어보지 않고도 코드베이스를 이해할 수 있게 한다.
README에 따르면 평균적인 저장소를 밀리초 단위로 풀 인덱싱하며, 리눅스 커널(2800만 LOC, 7만 5천 파일)을 3분 만에 인덱싱한다.

## 작동 방식

codebase-memory-mcp는 AST 파싱, 시맨틱 분석, 그래프 저장이라는 세 계층을 거쳐 코드를 지식 그래프로 변환한다.

### 3계층 아키텍처

첫째 계층은 Tree-sitter 기반 AST 파싱이다.
158개 언어 전체에 걸쳐 구문 구조를 추출한다.

둘째 계층은 하이브리드 LSP 시맨틱 레이어다.
타입 추론과 임포트 추적을 통해 Python, TypeScript/JavaScript, PHP, C#, Go, C, C++, Java, Kotlin, Rust에서 심볼 해석을 정교화한다.

셋째 계층은 지식 그래프 저장이다.
SQLite 기반의 영속 그래프로 저장되며, Cypher 유사 문법으로 쿼리한다.

인덱싱 파이프라인은 파일을 병렬로 처리하고, 메모리 내에서 LZ4로 데이터를 압축한 뒤, 마지막에 단일 SQLite 데이터베이스로 덤프한다.
인덱싱이 끝나면 메모리는 해제된다.

언어 지원은 정확도 티어로 구분된다.

| 티어 | 언어 |
|------|------|
| Excellent (90% 이상) | Lua, Kotlin, C++, C, Bash, Zig, Swift, CSS, YAML, TOML, HTML, SCSS, HCL, Dockerfile |
| Good (75-89%) | Python, TypeScript, Go, Rust, Java, JavaScript, PHP, C#, Ruby |
| Functional (75% 미만) | OCaml, Haskell |

위 언어 외에도 Tree-sitter가 지원하는 100여 개 언어가 추가로 포함되어 총 158개 언어를 다룬다.

### 지식 그래프 데이터 모델

그래프는 노드와 엣지로 코드 구조를 표현한다.
노드 라벨은 Project, Package, Folder, File, Module, Class, Function, Method, Interface, Enum, Type, Route, Resource이다.

엣지 타입은 코드 요소 사이의 관계를 나타낸다.
CONTAINS_PACKAGE, CONTAINS_FOLDER, CONTAINS_FILE, DEFINES, IMPORTS, CALLS, HTTP_CALLS, ASYNC_CALLS, IMPLEMENTS, HANDLES, USAGE, CONFIGURES, WRITES, MEMBER_OF, TESTS, USES_TYPE, FILE_CHANGES_WITH 등이 있다.

쿼리는 openCypher의 읽기 서브셋을 지원한다.
MATCH, WHERE, RETURN, UNION과 집계 함수(count, sum, avg, min, max, collect), 그리고 labels, type, size, length, substring, 정규식 매칭 같은 함수를 사용할 수 있다.

## 성능과 토큰 효율

README가 제시하는 성능 수치는 다음과 같다.

| 항목 | 수치 |
|------|------|
| 리눅스 커널 인덱싱 | 3분 (2800만 LOC, 7만 5천 파일, 481만 노드, 772만 엣지) |
| Django 인덱싱 | 약 6초 (4만 9천 노드, 19만 6천 엣지) |
| 관계 탐색 쿼리 속도 | 1ms 미만 |
| 토큰 절감 | 99.2% 감소 |

토큰 효율은 직접 비교로 제시된다.
다섯 개의 구조적 쿼리를 codebase-memory-mcp로 실행하면 약 3,400 토큰을 소비한다.
같은 작업을 파일 단위 grep 탐색으로 수행하면 약 412,000 토큰이 든다.
즉 파일 단위 탐색 대비 99.2%의 토큰을 절감한다.

## 설치와 연동

설치는 단일 바이너리를 받는 방식이며, 설치 스크립트가 주요 에이전트의 MCP 설정을 자동 구성한다.

### 설치 방법

macOS와 Linux에서는 한 줄 명령으로 설치한다.

```bash
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash
```

그래프 시각화 UI를 함께 설치하려면 옵션을 추가한다.

```bash
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh | bash -s -- --ui
```

Windows에서는 PowerShell로 설치 스크립트를 내려받아 실행한다.

```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.ps1 -OutFile install.ps1
.\install.ps1
```

소스에서 직접 빌드하려면 C 컴파일러, C++ 컴파일러, zlib가 필요하다.

```bash
git clone https://github.com/DeusData/codebase-memory-mcp.git
cd codebase-memory-mcp
scripts/build.sh
scripts/build.sh --with-ui
```

### MCP 연동 설정

설치 스크립트는 11개 에이전트를 자동 감지해 설정한다.
대상은 Claude Code, Codex CLI, Gemini CLI, Zed, OpenCode, Antigravity, Aider, KiloCode, VS Code, OpenClaw, Kiro이다.

수동으로 설정하려면 `~/.claude/.mcp.json`에 다음과 같이 추가한다.

```json
{
  "mcpServers": {
    "codebase-memory-mcp": {
      "command": "/path/to/codebase-memory-mcp",
      "args": []
    }
  }
}
```

## 사용법

codebase-memory-mcp는 인덱싱과 쿼리를 위한 MCP 도구를 제공하며, 동일한 기능을 CLI로도 호출할 수 있다.

### 제공 도구

인덱싱 관련 도구는 다음과 같다.

| 도구 | 설명 |
|------|------|
| index_repository | 코드베이스를 인덱싱 |
| list_projects | 인덱싱된 프로젝트 목록 표시 |
| delete_project | 프로젝트 데이터 삭제 |
| index_status | 인덱싱 진행 상태 확인 |

쿼리 관련 도구는 다음과 같다.

| 도구 | 설명 |
|------|------|
| search_graph | 라벨, 이름 패턴, 파일 패턴으로 구조적 검색 |
| trace_path | 콜 그래프의 BFS 탐색 (깊이 1-5) |
| detect_changes | git diff를 영향받는 심볼에 매핑 |
| query_graph | Cypher 유사 읽기 전용 쿼리 실행 |
| get_graph_schema | 노드/엣지 수와 패턴 반환 |
| get_code_snippet | 정규화된 이름으로 소스 조회 |
| get_architecture | 코드베이스 개요 (언어, 패키지, 라우트, 클러스터) |
| search_code | grep 유사 텍스트 검색 |
| manage_adr | 아키텍처 결정 기록 CRUD |
| ingest_traces | HTTP 라우트 엣지 검증 |

### CLI와 설정

CLI에서는 도구 이름과 JSON 인자를 함께 전달한다.

```bash
codebase-memory-mcp cli index_repository '{"repo_path": "/path/to/repo"}'
codebase-memory-mcp cli search_graph '{"name_pattern": ".*Handler.*", "label": "Function"}'
codebase-memory-mcp cli trace_path '{"function_name": "Search", "direction": "both"}'
codebase-memory-mcp cli query_graph '{"query": "MATCH (f:Function) RETURN f.name LIMIT 5"}'
```

자동 인덱싱은 설정 명령으로 켤 수 있다.

```bash
codebase-memory-mcp config set auto_index true
codebase-memory-mcp config set auto_index_limit 50000
```

환경 변수로 동작을 조정할 수 있다.

| 환경 변수 | 설명 |
|-----------|------|
| CBM_CACHE_DIR | 데이터베이스 저장 위치 (기본값 ~/.cache/codebase-memory-mcp) |
| CBM_LOG_LEVEL | 로깅 상세도 (debug, info, warn, error, none) |
| CBM_WORKERS | 병렬 인덱싱 워커 수 |
| CBM_DIAGNOSTICS | 주기적 진단 출력 활성화 |

커스텀 파일 확장자는 `.codebase-memory.json`으로 매핑한다.

```json
{"extra_extensions": {".blade.php": "php", ".mjs": "javascript"}}
```

그래프 시각화 UI는 내장 명령으로 실행한다.

```bash
codebase-memory-mcp --ui=true --port=9749
```

실행 후 http://localhost:9749 에서 인터랙티브 3D 탐색이 가능하다.

또한 프로젝트는 zstd로 압축된 SQLite 스냅샷인 `.codebase-memory/graph.db.zst`를 내보내 저장소에 커밋할 수 있다.
팀원은 전체 재인덱싱을 건너뛰고, 증분 인덱싱으로 로컬 변경분만 채운다.

## 결론

codebase-memory-mcp는 Tree-sitter AST 파싱, 하이브리드 LSP 시맨틱 레이어, SQLite 지식 그래프를 결합해 158개 언어의 코드 구조를 영속적 그래프로 만든다.
1ms 미만의 관계 탐색과 파일 단위 탐색 대비 99.2% 토큰 절감은 AI 코딩 에이전트가 대규모 코드베이스를 다룰 때 직접적인 효율로 이어진다.
런타임 의존성 없는 단일 바이너리, 로컬 처리, 11개 에이전트 자동 설정, 팀 공유 스냅샷까지 제공해 실무 도입 장벽을 낮춘다.

## Reference

- [codebase-memory-mcp (GitHub)](https://github.com/DeusData/codebase-memory-mcp/)
