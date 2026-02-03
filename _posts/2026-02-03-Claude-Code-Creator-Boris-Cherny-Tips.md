---
layout: post
title: Claude Code 창시자 Boris Cherny가 공개한 10가지 실전 사용 팁
author: 'Juho'
date: 2026-02-03 02:00:00 +0900
categories: [Claude]
tags: [Claude Code, Productivity, Coding]
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
2. [Git Worktree로 병렬 작업 실행](#git-worktree로-병렬-작업-실행)
3. [복잡한 작업은 Plan 모드부터 시작](#복잡한-작업은-plan-모드부터-시작)
4. [CLAUDE.md에 투자하기](#claudemd에-투자하기)
5. [반복 작업을 커스텀 Skill로 만들기](#반복-작업을-커스텀-skill로-만들기)
6. [버그 수정은 Claude에게 위임](#버그-수정은-claude에게-위임)
7. [프롬프팅 수준 높이기](#프롬프팅-수준-높이기)
8. [터미널과 환경 설정 최적화](#터미널과-환경-설정-최적화)
9. [서브에이전트 활용](#서브에이전트-활용)
10. [데이터 분석에 활용](#데이터-분석에-활용)
11. [Claude와 함께 학습하기](#claude와-함께-학습하기)
12. [Reference](#reference)

## 개요

Claude Code 팀의 Boris Cherny가 Anthropic 엔지니어링 팀이 실제로 Claude Code를 사용하는 방법을 공개했다.
단순한 코드 자동완성을 넘어, AI 기반 개발 워크플로우를 극대화하는 10가지 실전 팁을 담고 있다.
병렬 작업, 계획 모드, 서브에이전트 활용 등 생산성을 크게 향상시킬 수 있는 구체적인 방법론을 제시한다.

## Git Worktree로 병렬 작업 실행

Boris가 꼽은 가장 큰 생산성 향상 방법은 git worktree를 활용한 병렬 작업이다.
3~5개의 git worktree를 동시에 실행하고 각각에 별도의 Claude 세션을 운영하는 방식이다.

git worktree는 하나의 저장소에서 여러 브랜치를 서로 다른 디렉토리에 동시에 체크아웃할 수 있는 기능이다.
이를 통해 여러 기능을 병렬로 개발할 수 있다.

셸 별칭(alias)을 za, zb, zc 등으로 설정하면 한 번의 키 입력으로 worktree 간 전환이 가능하다.
각 worktree에서 독립적인 Claude 세션이 동작하므로, 하나의 작업이 완료되기를 기다리지 않고 다른 작업을 진행할 수 있다.

## 복잡한 작업은 Plan 모드부터 시작

복잡한 작업을 시작할 때는 반드시 Plan 모드로 시작해야 한다.
Plan 모드는 Claude가 코드베이스를 읽기 전용으로 분석하여 계획을 수립하는 단계이다.

계획 단계에 충분한 에너지를 투자하면 Claude가 한 번에 구현을 완료할 가능성이 크게 높아진다.
문제가 발생했을 때는 기존 구현을 수정하기보다 처음부터 다시 계획하는 것이 더 효과적이다.
다른 Claude 세션을 열어 계획 자체를 검토받는 것도 좋은 방법이다.

## CLAUDE.md에 투자하기

CLAUDE.md 파일을 지속적으로 관리하면 Claude의 실수율을 크게 줄일 수 있다.
수정 작업 후마다 Claude에게 "CLAUDE.md를 업데이트해서 다시 같은 실수를 하지 마"라고 지시하는 것이 핵심이다.

Claude는 자신의 규칙을 작성하는 데 뛰어난 능력을 보인다.
반복적인 업데이트를 통해 프로젝트에 특화된 규칙이 축적되면, 시간이 지날수록 오류율이 감소한다.

## 반복 작업을 커스텀 Skill로 만들기

하루에 한 번 이상 반복하는 작업은 Skill이나 슬래시 명령어로 변환해야 한다.
Skill은 자주 사용하는 워크플로우를 자동화하는 기능이다.

활용 예시는 다음과 같다.
- `/techdebt` : 코드베이스에서 중복 코드를 찾아 정리하는 명령어
- 협업 도구 동기화 명령어
- dbt 모델 작성 에이전트

반복 작업을 Skill로 만들면 프롬프트를 매번 작성하는 시간을 절약할 수 있다.

## 버그 수정은 Claude에게 위임

버그 수정 작업은 Claude에게 위임하는 것이 효율적이다.
Slack 버그 스레드를 Claude에 붙여넣고 "fix"라고만 말하면 된다.
CI 테스트 실패 로그나 Docker 로그 분석도 마찬가지로 위임 가능하다.

Slack MCP를 연결하면 Slack 채널의 버그 리포트를 직접 가져와 처리할 수 있다.
핵심은 과정을 미시 관리(micromanage)하지 않는 것이다.
Claude에게 문제를 던지고 결과만 확인하는 방식이 가장 효과적이다.

## 프롬프팅 수준 높이기

Claude의 출력 품질은 프롬프트의 품질에 비례한다.
상세한 지시와 스펙을 작성하면 모호함이 줄어들고 결과물의 품질이 향상된다.

효과적인 프롬프트 예시는 다음과 같다.
- "이 변경사항을 비판적으로 검토하고 내가 테스트를 통과할 때까지 PR을 만들지 마"
- "지금 알고 있는 모든 것을 고려해 처음부터 우아한 해결책으로 구현해"

프롬프트에 구체적인 제약 조건, 기대 결과, 품질 기준을 명시하면 더 나은 결과를 얻을 수 있다.

## 터미널과 환경 설정 최적화

Boris는 Ghostty 터미널 사용을 추천한다.
`/statusline` 명령어를 활용하면 컨텍스트 사용량과 현재 git 브랜치를 실시간으로 확인할 수 있다.

음성 입력도 강력한 생산성 도구이다.
Boris에 따르면 말하는 속도는 타이핑 속도의 3배이며, 그 결과 프롬프트가 훨씬 더 상세해진다.
macOS에서는 fn 키를 두 번 누르면 음성 받아쓰기를 사용할 수 있다.

터미널 탭에 색상을 구분하여 설정하면 여러 worktree 간 전환 시 혼동을 줄일 수 있다.

## 서브에이전트 활용

"use subagents"라고 명령하면 Claude가 더 많은 계산 리소스를 작업에 투입한다.
서브에이전트는 복잡한 작업을 분산 처리하면서 메인 컨텍스트 윈도우를 깔끔하게 유지하는 역할을 한다.

추가로 Opus 4.5를 통한 권한 요청 자동 승인을 Hook으로 설정하면, 안전한 작업에 대해 사람의 개입 없이 자동으로 진행되도록 할 수 있다.
이를 통해 Claude가 작업 중 멈추는 빈도를 줄일 수 있다.

## 데이터 분석에 활용

bq CLI나 데이터베이스 CLI, MCP, API를 Claude와 연결하면 데이터 분석 작업을 자동화할 수 있다.
Boris는 6개월 이상 SQL을 직접 작성하지 않았다고 언급했다.

Claude에게 분석하고 싶은 내용을 자연어로 설명하면, 필요한 쿼리를 자동으로 생성하고 실행하여 결과를 분석해 준다.
데이터 엔지니어링과 분석 업무에서 큰 시간 절약이 가능하다.

## Claude와 함께 학습하기

`/config`에서 출력 스타일을 "Explanatory" 또는 "Learning"으로 설정하면 Claude가 변경사항의 이유를 함께 설명해 준다.
단순히 코드를 생성하는 것을 넘어, 왜 그런 결정을 내렸는지 이해할 수 있다.

추가 활용 방법은 다음과 같다.
- HTML 시각 자료 생성 : Claude에게 발표 자료나 다이어그램을 HTML로 생성하도록 요청
- ASCII 코드베이스 다이어그램 : 코드베이스의 구조를 ASCII 아트로 시각화
- 간격 반복 학습(Spaced Repetition) : 학습 스킬을 만들어 새로운 기술을 효율적으로 습득

## Reference

- [Boris Cherny - Claude Code Tips (X/Twitter)](https://x.com/bcherny/status/2017742741636321619){:target="_blank"}
- [10 Claude Code tips from Boris, summarized](https://ykdojo.github.io/claude-code-tips/content/boris-claude-code-tips){:target="_blank"}
- [Inside Claude Code: How Anthropic's Engineers Are Rewriting the Playbook - WebProNews](https://www.webpronews.com/inside-claude-code-how-anthropics-engineers-are-rewriting-the-playbook-for-ai-assisted-development/){:target="_blank"}
