---
layout: post
title: "RTK: AI 코딩 도구의 토큰 소비를 60~90% 줄이는 Rust CLI 프록시"
author: 'Juho'
date: 2026-04-09 00:00:00 +0900
categories: [Dev]
tags: [Dev, AI, LLM]
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
2. [RTK란 무엇인가](#rtk란-무엇인가)
3. [핵심 기능](#핵심-기능)
   - [4가지 토큰 최적화 전략](#4가지-토큰-최적화-전략)
   - [자동 재작성 훅](#자동-재작성-훅)
4. [토큰 절감 효과](#토큰-절감-효과)
5. [설치 및 사용법](#설치-및-사용법)
   - [설치](#설치)
   - [AI 도구별 설정](#ai-도구별-설정)
   - [주요 명령어](#주요-명령어)
6. [지원 AI 도구](#지원-ai-도구)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

RTK(Rust Token Killer)는 AI 코딩 도구가 실행하는 CLI 명령어 출력을 필터링하고 압축하여 토큰 소비를 60~90% 줄여주는 Rust 기반 프록시 도구이다.
단일 바이너리로 배포되며, 외부 의존성 없이 100개 이상의 명령어를 지원한다.
Claude Code, Cursor, Gemini CLI 등 10개 이상의 AI 코딩 도구와 호환된다.

## RTK란 무엇인가

AI 코딩 도구는 셸 명령어를 실행하고 그 출력을 LLM에 전달하여 컨텍스트로 활용한다.
문제는 `git status`, `cargo test` 같은 명령어의 출력에 LLM이 필요로 하지 않는 보일러플레이트, 중복 로그, 공백 등이 대량으로 포함된다는 점이다.
RTK는 이러한 명령어 출력을 LLM에 전달하기 전에 스마트하게 필터링하여 필요한 정보만 남긴다.
10ms 미만의 오버헤드로 동작하므로 체감 속도 저하가 없다.

## 핵심 기능

### 4가지 토큰 최적화 전략

RTK는 4가지 전략을 조합하여 토큰을 절감한다.

| 전략 | 설명 |
|------|------|
| 스마트 필터링 | 주석, 공백, 보일러플레이트 제거 |
| 그룹핑 | 유사 항목 집계 (디렉토리별 파일, 오류 타입별) |
| 트렁케이션 | 관련 내용만 유지하고 나머지 절단 |
| 중복 제거 | 반복 로그 라인을 개수로 표현 |

JSON, YAML, 텍스트 등 포맷을 자동 감지하여 적절한 전략을 적용한다.
또한 AWS 자격증명, API 키 등 비밀 정보를 자동으로 제거하는 보안 기능도 포함되어 있다.

### 자동 재작성 훅

가장 효과적인 사용 방법은 자동 재작성 훅이다.
Bash 명령어를 투명하게 RTK 버전으로 변환한다.

```
Claude → "git status" → RTK 훅 → "rtk git status" → 실행
                        (투명한 재작성)
```

사용자나 AI 도구가 별도로 RTK 명령어를 사용할 필요 없이 100% 자동으로 최적화된다.

## 토큰 절감 효과

30분 세션 기준 토큰 절감 데이터이다.

| 작업 | 빈도 | 일반 토큰 | RTK 토큰 | 절감률 |
|------|------|-----------|----------|--------|
| git status | 10회 | 3,000 | 600 | 80% |
| cargo test | 5회 | 25,000 | 2,500 | 90% |
| 전체 세션 | - | 약 118,000 | 약 23,900 | 80% |

테스트 러너 출력에서는 실패 케이스만 표시하여 최대 90%까지 절감할 수 있다.
`rtk gain` 명령어로 실제 절감 통계를 실시간으로 확인할 수 있다.

## 설치 및 사용법

### 설치

```bash
# Homebrew (권장)
brew install rtk

# 빠른 설치
curl -fsSL https://raw.githubusercontent.com/rtk-ai/rtk/refs/heads/master/install.sh | sh

# Cargo
cargo install --git https://github.com/rtk-ai/rtk
```

### AI 도구별 설정

```bash
rtk init -g                    # Claude Code용
rtk init -g --agent cursor     # Cursor용
rtk init --agent windsurf      # Windsurf용
rtk init -g --copilot          # GitHub Copilot용
rtk init -g --gemini           # Gemini CLI용
```

셸 재시작 후 모든 명령어가 자동으로 RTK 버전으로 변환된다.

### 주요 명령어

파일 작업 관련 명령어이다.

```bash
rtk ls .              # 토큰 최적화 디렉토리 트리
rtk read file.rs      # 스마트 파일 읽기
rtk grep "pattern" .  # 그룹화된 검색 결과
rtk find "*.rs" .     # 컴팩트 검색
```

Git 작업 관련 명령어이다.

```bash
rtk git status        # 컴팩트 상태
rtk git log -n 10     # 한 줄 커밋
rtk git diff          # 축약된 차이
```

테스트 및 빌드 관련 명령어이다.

```bash
rtk test cargo test   # 실패만 표시 (-90%)
rtk lint              # ESLint 결과 (규칙/파일별)
rtk err npm run build # 오류/경고만
```

토큰 절감 분석 명령어이다.

```bash
rtk gain              # 요약 통계
rtk gain --graph      # ASCII 그래프 (30일)
rtk discover          # 최적화 기회 탐색
rtk session           # 채택률 조회
```

## 지원 AI 도구

| 도구 | 설치 명령 | 방식 |
|------|-----------|------|
| Claude Code | rtk init -g | PreToolUse 훅 |
| GitHub Copilot | rtk init -g --copilot | 투명한 재작성 |
| Cursor | rtk init -g --agent cursor | hooks.json |
| Windsurf | rtk init --agent windsurf | .windsurfrules |
| Cline/Roo Code | rtk init --agent cline | .clinerules |
| Gemini CLI | rtk init -g --gemini | BeforeTool 훅 |

## 결론

RTK는 AI 코딩 도구 사용 시 발생하는 토큰 비용 문제를 실용적으로 해결한다.
자동 훅 방식으로 수동 개입 없이 모든 명령어 출력을 최적화하며, 10ms 미만의 오버헤드로 체감 속도 저하가 없다.
다만 압축으로 인한 컨텍스트 손실 가능성이 있으므로, 디버깅 등 상세 출력이 필요한 경우에는 `-v` 플래그로 상세도를 조절할 수 있다.
전체 출력을 별도 파일로 저장하는 기능도 제공되어 이러한 트레이드오프를 관리할 수 있다.

## Reference

- [RTK GitHub Repository](https://github.com/rtk-ai/rtk/)
