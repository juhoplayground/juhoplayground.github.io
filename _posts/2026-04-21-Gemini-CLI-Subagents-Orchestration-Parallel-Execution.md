---
layout: post
title: "Gemini CLI Subagents 도입 - 독립 컨텍스트 기반 병렬 오케스트레이션"
author: 'Juho'
date: 2026-04-21 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, Context]
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
2. [Subagents의 동작 방식](#subagents의-동작-방식)
   - [독립 컨텍스트 윈도우](#독립-컨텍스트-윈도우)
   - [기본 제공 Subagents](#기본-제공-subagents)
3. [커스텀 Subagent 제작](#커스텀-subagent-제작)
   - [설정 파일 구조](#설정-파일-구조)
   - [호출 방법](#호출-방법)
4. [병렬 실행과 주의사항](#병렬-실행과-주의사항)
5. [의미와 시사점](#의미와-시사점)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Google이 Gemini CLI에 Subagents 기능을 공식 도입했다.
Subagents는 기본 Gemini CLI 세션과 함께 동작하는 특화된 자율 에이전트다.
각 서브에이전트는 별도의 컨텍스트 윈도우, 커스텀 시스템 지침, 큐레이션된 도구 세트를 갖는다.

기본 에이전트는 전략적 오케스트레이터로 동작하며, 전문 영역이 필요한 순간을 식별해 관련 서브에이전트에 하위 작업을 위임한다.
하위 실행은 단일 응답으로 통합되어 메인 컨텍스트 윈도우 오염을 방지한다.

## Subagents의 동작 방식

### 독립 컨텍스트 윈도우

각 서브에이전트는 독립적으로 동작한다.
자체 도구, MCP 서버, 시스템 지침, 컨텍스트 윈도우를 보유한다.
수많은 도구 호출과 테스트가 내부에서 일어나더라도 최종 결과는 요약된 단일 응답으로 반환된다.

이 구조는 기본 에이전트가 전체 목표, 의사결정, 최종 응답에만 집중할 수 있게 만든다.
원시 출력이 아닌 요약이 메인으로 돌아오기 때문에 컨텍스트 부패(context rot)가 방지된다.

### 기본 제공 Subagents

Gemini CLI는 세 종류의 빌트인 서브에이전트를 기본 제공한다.

| Subagent | 역할 |
|----------|------|
| generalist | 전체 도구 접근 권한을 가진 범용 에이전트 (배치 작업용) |
| cli_help | Gemini CLI 문서와 기능에 대한 전문 에이전트 |
| codebase_investigator | 코드 탐색, 아키텍처 매핑, 버그 분석 전담 |

## 커스텀 Subagent 제작

### 설정 파일 구조

커스텀 서브에이전트는 YAML frontmatter가 포함된 Markdown 파일로 정의된다.
개인 레벨은 `~/.gemini/agents`, 프로젝트 레벨은 `.gemini/agents`에 배치한다.

```yaml
---
name: frontend-specialist
description: Frontend specialist in building high-performance, accessible,
  and scalable web applications using modern frameworks and standards.
tools:
  - read_file
  - grep_search
  - glob
  - list_directory
  - web_fetch
  - google_web_search
model: inherit
---
```

Frontmatter 뒤에는 해당 에이전트의 전문성과 행동 지침을 정의하는 시스템 인스트럭션이 이어진다.

### 호출 방법

두 가지 호출 방식이 지원된다.
첫 번째는 자동 라우팅으로, Gemini CLI가 작업 요구사항에 따라 적절한 서브에이전트를 스스로 선택한다.
두 번째는 `@agent` 구문을 통한 명시적 위임이다.

```bash
@frontend-specialist Can you review our app and flag potential improvements?
@generalist Update the license headers across the whole project.
@codebase_investigator Map out the authentication flow.
```

현재 세션에 설정된 모든 서브에이전트는 `/agents` 명령으로 확인할 수 있다.

## 병렬 실행과 주의사항

여러 서브에이전트가 동시에 실행될 수 있다.
"Run the frontend-specialist on each package in parallel"처럼 명시적으로 요청하면 독립 작업의 완료 시간이 대폭 단축된다.

다만 코드 편집 작업에서의 병렬 실행은 충돌 및 덮어쓰기 위험을 수반한다.
읽기/조사/검증 성격의 작업에는 적합하지만, 동일 파일을 수정할 가능성이 있는 작업은 순차 실행이 안전하다.

## 의미와 시사점

Subagents 패턴은 Claude Code의 Task tool, Cursor의 Background Agents와 유사한 방향성을 공유한다.
프론티어 AI 에이전트 시스템에서 단일 컨텍스트 윈도우의 한계를 극복하기 위한 공통 설계다.

메인 에이전트는 지휘자 역할에 집중하고, 실무는 격리된 컨텍스트에서 처리한 뒤 요약만 전달된다.
이는 장시간 실행되는 복잡한 태스크의 신뢰성과 비용 효율성을 동시에 개선한다.

## 결론

Gemini CLI의 Subagents는 에이전트 오케스트레이션이 이제 선택이 아닌 기본 인프라로 자리잡고 있음을 보여준다.
YAML frontmatter 기반의 간단한 정의 방식과 `@agent` 호출 문법은 개발자 친화적이다.
프로젝트 단위로 `.gemini/agents` 디렉터리를 구성하면 팀 전체가 공유하는 특화 에이전트 세트를 빠르게 구축할 수 있다.

## Reference

- [Subagents have arrived in Gemini CLI](https://developers.googleblog.com/subagents-have-arrived-in-gemini-cli/)
