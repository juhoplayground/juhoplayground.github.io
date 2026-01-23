---
layout: post
title: Mantic.sh - AI 에이전트를 위한 맥락 인식 코드 검색 엔진
author: 'Juho'
date: 2026-01-23 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Coding, MCP]
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
2. [핵심 목적](#핵심-목적)
3. [주요 기능](#주요-기능)
4. [성능 지표](#성능-지표)
5. [설치 방법](#설치-방법)
6. [사용법](#사용법)
7. [다른 도구와의 비교](#다른-도구와의-비교)
8. [라이선스](#라이선스)

## 개요

Mantic.sh는 AI 에이전트를 위해 설계된 맥락 인식 코드 검색 엔진이다.
원시 속도보다 결과의 관련성을 우선시하며, 로컬에서만 실행되어 데이터 유출 걱정이 없다.
파일 구조와 메타데이터로부터 의도를 유추하여 불필요한 컨텍스트 검색 오버헤드를 제거하는 인프라 계층 역할을 한다.

## 핵심 목적

AI 에이전트가 코드베이스를 탐색할 때 가장 큰 문제는 관련 없는 파일들로 인한 컨텍스트 낭비다.
Mantic.sh는 이 문제를 해결하기 위해 다음을 목표로 한다.

- 의미 기반 검색으로 정확한 파일 찾기
- 토큰 사용량 최적화
- 로컬 실행으로 보안 유지

## 주요 기능

### 의도 기반 검색

"stripe payment integration" 같은 자연어 쿼리로 정확한 파일을 찾을 수 있다.
단순 문자열 매칭이 아닌 의미를 파악하여 관련 파일을 반환한다.

### CamelCase 감지

ScriptController를 검색하면 script_controller도 함께 매칭된다.
다양한 명명 규칙을 자동으로 인식한다.

### 경로 시퀀스 매칭

다중 용어 쿼리가 연속적인 경로 구성요소와 일치하는지 확인한다.
예를 들어 "api user controller"는 api/user/controller.js를 찾아낸다.

### 세션 관리

이전에 조회한 파일에 자동으로 가중치를 적용한다.
작업 맥락을 유지하여 연관 파일 검색 정확도를 높인다.

### 제로 쿼리 모드

검색어 없이도 수정된 파일과 관련 의존성을 자동으로 제시한다.
현재 작업 중인 코드와 연관된 파일을 즉시 파악할 수 있다.

### 영향 분석

코드 변경이 영향을 미치는 범위를 계산한다.
리팩토링이나 버그 수정 시 놓칠 수 있는 의존성을 파악하는 데 유용하다.

## 성능 지표

Mantic.sh의 성능은 다음과 같다.

### 속도

대부분의 저장소에서 500ms 이하로 검색이 완료된다.
Chromium처럼 481K 파일이 있는 대규모 저장소에서도 4초 이하로 처리된다.

### 효율성

토큰 사용량을 최대 63% 감소시킨다.
관련 없는 파일을 필터링하여 AI 에이전트가 더 효율적으로 작동하도록 돕는다.

### 정확도

5개 저장소 테스트 결과 grep이나 ripgrep보다 관련성 측면에서 우수한 결과를 보였다.

## 설치 방법

### CLI 설치

설치 없이 즉시 실행할 수 있다.

```bash
npx mantic.sh@latest "검색 쿼리"
```

소스에서 설치하려면 다음 명령어를 사용한다.

```bash
git clone https://github.com/marcoaapfortes/Mantic.sh.git
cd Mantic.sh && npm install && npm run build && npm link
```

### MCP 서버 설치

Claude Desktop, Cursor, VS Code에 통합하여 사용할 수 있다.
MCP 설정 파일에 다음을 추가한다.

```json
{
  "mcpServers": {
    "mantic": {
      "command": "npx",
      "args": ["-y", "mantic.sh@latest", "server"]
    }
  }
}
```

## 사용법

### 기본 검색

```bash
mantic "stripe payment integration"
```

### 고급 옵션

다양한 옵션을 조합하여 검색 결과를 세밀하게 조절할 수 있다.

| 옵션 | 설명 |
|------|------|
| --code | 코드 파일만 검색 |
| --session id | 세션 기반 컨텍스트 유지 |
| --impact | 의존성 분석 포함 |
| --markdown | 보기 좋은 터미널 출력 |

### 예시

```bash
# 코드 파일만 검색
mantic --code "authentication middleware"

# 세션 유지하며 검색
mantic --session my-feature "user validation"

# 의존성 분석 포함
mantic --impact "database connection"
```

## 다른 도구와의 비교

Mantic.sh를 기존 검색 도구와 비교하면 다음과 같다.

| 기능 | Mantic | ripgrep | fzf |
|------|--------|---------|-----|
| 관련성 순위 | 우수 | 없음 | 기본 |
| 경로 구조 인식 | 완벽 | 없음 | 부분 |
| CamelCase 감지 | 지원 | 미지원 | 미지원 |
| 텍스트 검색 속도 | 느림 | 최속 | 매우 빠름 |

ripgrep은 원시 속도에서 가장 빠르지만 관련성 순위가 없다.
Mantic.sh는 속도를 약간 희생하는 대신 검색 결과의 품질을 크게 향상시킨다.
AI 에이전트 환경에서는 관련성이 속도보다 중요하기 때문에 Mantic.sh가 더 적합하다.

## 라이선스

Mantic.sh는 이중 라이선스 구조를 가진다.

### AGPL-3.0

개인 사용이나 오픈소스 프로젝트 내부 사용은 무료다.

### Commercial

독점 소프트웨어에 임베딩할 경우 유료 라이선스가 필요하다.
연간 $500부터 시작한다.

### 비용 절감 효과

벡터 임베딩 방식을 사용하면 회사 규모 100명 기준 연간 $1,680에서 $46,800 이상의 비용이 발생한다.
Mantic.sh는 완전히 로컬에서 실행되므로 이러한 비용을 $0로 줄일 수 있다.
보안 측면에서도 코드가 외부 서버로 전송되지 않아 안전하다.

## Reference

- [Mantic.sh GitHub](https://github.com/marcoaapfortes/Mantic.sh)
