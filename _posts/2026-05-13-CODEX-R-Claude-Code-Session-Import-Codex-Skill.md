---
layout: post
title: "CODEX-R: Claude Code 세션을 Codex로 가져오는 codex -r 픽커 스킬"
author: 'Juho'
date: 2026-05-13 00:00:00 +0900
categories: [VibeCoding]
tags: [VibeCoding, Skill, Skill Development]
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
   - [왜 codex -r이 필요했나](#왜-codex--r이-필요했나)
   - [네 가지 실무적 이슈](#네-가지-실무적-이슈)
3. [핵심 내용](#핵심-내용)
   - [기본 사용법](#기본-사용법)
   - [설치](#설치)
   - [동작 원리](#동작-원리)
4. [의미와 시사점](#의미와-시사점)
   - [안전 계약](#안전-계약)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

CODEX-R은 OpenAI Codex CLI에 `claude -r`과 동일한 워크플로우를 제공하는 Codex 스킬이다.
Claude Code 세션을 픽커로 고르면 Codex로 임포트되고, 임포트된 Codex 세션으로 바로 진입한다.

```bash
codex -r
```

Codex CLI 0.128.0은 외부 에이전트 세션 임포트를 추가했지만, 첫 진입 경로가 잘 보이지 않는다.
시작 프롬프트는 `external_migration` 피처 플래그와 신뢰 온보딩 플로우에 의존하기 때문에, 이미 신뢰된 프로젝트에서는 프롬프트가 영영 뜨지 않을 수 있다.
대용량 `~/.claude/projects` 디렉토리에서는 홈 전역 탐색이 느려진다는 문제도 있다.

CODEX-R은 이 두 도구를 오가는 사용자를 위해 세션 마이그레이션을 픽커 한 번으로 줄인다.

## 배경

### 왜 codex -r이 필요했나

Claude Code 사용자는 최근 작업을 이어가기 위해 `claude -r`을 자주 쓴다.
Codex에는 외부 에이전트 세션 임포트 경로가 있지만, 이 스킬을 만드는 시점까지 `codex import claude`나 `codex -r` 같은 단순한 명령으로 노출되어 있지 않았다.

이 간극을 메우기 위해 CODEX-R은 로컬에서 `codex` 래퍼와 Claude Code 세션 픽커, 안전 검증 명령을 묶어 제공한다.

### 네 가지 실무적 이슈

저자가 첫 로컬 실험에서 발견한 네 가지 문제는 다음과 같다.

- Codex CLI 0.128.0은 외부 에이전트 세션 임포트 기능을 포함한다.
- TUI 프롬프트는 `external_migration`이 켜져 있고 세션이 신뢰 온보딩 플로우에 진입할 때만 나타난다.
- 기존 셸 alias가 `codex -r`을 진짜 Codex 바이너리로 라우팅하면 `unexpected argument '-r'`로 종료된다.
- 실제 임포트와 셋업 검증은 분리되어야 한다.
에이전트는 `--list`와 `--dry-run`으로 테스트하고, 실제 임포트는 사람이 결정해야 한다.

CODEX-R은 이 네 가지 교훈을 재사용 가능한 Codex 스킬 형태로 패키징한다.

## 핵심 내용

### 기본 사용법

`codex -r`은 인자에 따라 동작이 달라진다.

```bash
codex -r                    # 픽커 열기, 선택 시 임포트
codex -r                    # 기본은 현재 디렉토리와 정확히 일치하는 세션
codex -r daybreak           # ~/ws/daybreak 디렉토리가 있으면 해당 세션 열기
codex -r --cwd ~/ws/kb      # 특정 디렉토리의 세션 열기
codex -r --recursive        # 현재 디렉토리의 자식 디렉토리 포함
codex -r --all daybreak     # 텍스트로 모든 세션 검색
codex -r --list --limit 5   # 목록만, 임포트 안 함
codex -r --dry-run --limit 1 # 검사만, 임포트 안 함
```

기본값으로 현재 디렉토리에 정확히 일치하는 세션만 보여주는 점이 핵심이다.
홈 전역 탐색을 피해 대용량 프로젝트 폴더에서도 빠르게 동작한다.

### 설치

스킬은 Markdown 단일 파일로 배포되며 별도의 설치 스크립트가 없다.
저장소를 클론하고 Codex 스킬 디렉토리에 심볼릭 링크를 거는 방식이다.

```bash
git clone https://github.com/thedalbee/codex-r.git ~/ws/codex-r
ln -sfn ~/ws/codex-r ~/.codex/skills/codex-r
```

새 Codex 세션을 시작한 뒤 `$codex-r`로 호출하면 Codex가 `SKILL.md`를 읽고 셸 alias와 PATH를 점검한다.
필요하면 로컬 래퍼를 패치하고, 실제 임포트 없이 결과를 검증한다.

### 동작 원리

CODEX-R은 Codex의 실험적 외부 마이그레이션 경로를 사용한다.

| 항목 | 값 |
|---|---|
| 피처 플래그 | external_migration |
| Claude 세션 소스 | ~/.claude/projects/**/*.jsonl |
| 임포트 RPC | externalAgentConfig/import |
| 완료 이벤트 | externalAgentConfig/import/completed |
| 임포트 원장 | ~/.codex/external_agent_session_imports.json |

선택된 세션은 `SESSIONS` 마이그레이션 아이템으로 한 건씩 임포트된다.
임포트가 끝나면 원장을 읽고 `codex resume <threadId>`를 시작하며, 스레드 ID를 찾지 못하면 `codex resume --all`로 폴백한다.

CODEX-R은 Claude의 설정, MCP 서버, 플러그인, 스킬을 복사하지 않는다.
선택된 Claude 세션 JSONL 파일만 Codex 자체 app-server 마이그레이션 API로 가져온다.

## 의미와 시사점

CODEX-R은 도구 사이를 오가는 사용자에게 컨텍스트 단절 비용을 줄여준다.
Claude Code에서 진행하던 멀티턴 작업을 Codex의 모델로 이어 보고 싶거나, 그 반대인 경우 세션 자체가 자산이 된다.

스킬이 Markdown만으로 구성된다는 점도 흥미롭다.
설치 스크립트가 없으므로 사용자가 코드 실행 동의를 별도로 하지 않아도 되고, Codex가 직접 자기 환경을 점검·패치한다.

### 안전 계약

CODEX-R은 한 가지 규칙을 엄격히 지킨다.
셋업 검증이 절대 세션을 임포트해서는 안 된다.

- `codex -r --list`는 임포트하지 않는다.
- `codex -r --dry-run`은 번호를 선택해도 임포트하지 않는다.
- `codex -r`은 사용자가 세션을 선택한 뒤에만 임포트한다.
- 기본적으로 `codex -r`은 기록된 cwd가 현재 디렉토리와 정확히 일치하는 Claude 세션만 보여준다.
자식 디렉토리는 `--recursive`, 전역 검색은 `--all`을 사용한다.
- Codex가 공식 `-r` 지원을 추가하면 래퍼는 공식 동작에 위임해야 한다.

이 계약은 에이전트가 사람의 동의 없이 세션을 옮기지 않도록 한다.
공식 명령이 나오면 비켜주는 가역적(reversible) 설계라는 점도 명시되어 있다.

## 결론

CODEX-R은 OpenAI의 공식 도구가 아니다.
Codex의 실험적 마이그레이션 API에 기대는 로컬 스킬이며, 안정적인 공식 명령이 나올 때까지의 다리 역할을 자처한다.

사용 대상이 명확하다.
Claude Code와 OpenAI Codex 사이를 오가고, `~/.claude/projects` 아래에 세션이 쌓여 있으며, 가역적인 로컬 셋업을 원하는 사용자다.
Codex만 쓰거나 Claude 세션이 없거나 안정적인 공식 명령을 기다릴 사용자는 굳이 쓸 필요가 없다.

스킬 자체가 Markdown 한 장이라는 점, 그리고 검증과 실제 임포트를 명시적으로 분리한 점이 이 작은 도구를 안전하고 빠르게 평가할 수 있게 만든다.

## Reference

- [thedalbee/codex-r](https://github.com/thedalbee/codex-r/)
