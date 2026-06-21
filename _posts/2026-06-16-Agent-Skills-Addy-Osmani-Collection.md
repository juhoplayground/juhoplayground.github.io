---
layout: post
title: "Agent Skills: Addy Osmani가 정리한 AI 코딩 에이전트 워크플로우 모음"
author: 'Juho'
date: 2026-06-16 00:00:00 +0900
categories: [AI]
tags: [Agent, Skill, Documentation]
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
2. [핵심 개념](#핵심-개념)
   - [개발 라이프사이클 프레임워크](#개발-라이프사이클-프레임워크)
   - [스킬의 구조](#스킬의-구조)
3. [24개 스킬과 구성 요소](#24개-스킬과-구성-요소)
   - [단계별 스킬 목록](#단계별-스킬-목록)
   - [에이전트 페르소나와 참조 자료](#에이전트-페르소나와-참조-자료)
4. [설치와 사용](#설치와-사용)
5. [설계 철학](#설계-철학)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Agent Skills는 AI 코딩 에이전트가 소프트웨어 개발 전 과정을 일관되게 수행하도록 안내하는 프로덕션 수준의 엔지니어링 워크플로우 모음이다.
Addy Osmani가 만든 이 프로젝트는 시니어 엔지니어의 실무 관행을 구조화되고 검증 가능한 프로세스로 인코딩한다.
저장소는 다음과 같이 설명한다.
"Skills encode the workflows, quality gates, and best practices that senior engineers use when building software."

핵심 의도는 AI 에이전트가 지름길을 택하지 않도록 개발 단계 전반에 걸쳐 규율을 강제하는 것이다.
이 저장소는 64.5k 스타와 7k 포크를 기록하고 있으며, 최신 릴리스는 2026년 6월의 v0.6.2다.
주요 언어는 Shell(67.1%)과 JavaScript(32.9%)이고, MIT 라이선스로 배포된다.

## 핵심 개념

Agent Skills는 참조 문서가 아니라 실행 가능한 워크플로우를 제공한다는 점이 특징이다.
각 스킬은 단계별 절차와 체크포인트를 포함하며, 에이전트가 따라야 할 품질 게이트를 명시한다.

### 개발 라이프사이클 프레임워크

이 시스템은 작업을 여섯 단계로 조직하고, 각 단계에 대응하는 명령을 제공한다.

| 단계 | 명령 | 설명 |
|------|------|------|
| Define | /spec | 코딩 전에 명세를 작성 |
| Plan | /plan | 작업을 작고 원자적인 단위로 분해 |
| Build | /build | 테스트와 함께 점진적으로 구현 |
| Verify | /test | 기능이 동작함을 증명 |
| Review | /review | 머지 전 품질 게이트 |
| Ship | /ship | 자신감을 가지고 배포 |

`/build auto` 명령은 전체 계획을 자율적으로 생성하고 실행하면서도 개별 작업의 검증은 유지한다.

### 스킬의 구조

각 스킬은 일관된 구조를 따른다.

| 구성 요소 | 역할 |
|-----------|------|
| Frontmatter | 이름, 설명, 트리거 조건 |
| Overview | 스킬이 달성하는 목표 |
| When to Use | 구체적인 트리거 시나리오 |
| Process | 체크포인트가 있는 단계별 워크플로우 |
| Rationalizations | 단계를 건너뛰려는 흔한 변명과 반박 |
| Red Flags | 문제의 경고 신호 |
| Verification | 타협 불가능한 증거 요구 사항 |

이 구조에서 Rationalizations 항목은 에이전트가 단계를 생략하려고 사용하는 변명을 표로 정리하고 그에 대한 반론을 함께 제시한다.
Verification 항목은 추측이 아니라 구체적인 증거를 요구한다.

## 24개 스킬과 구성 요소

저장소는 24개의 워크플로우 스킬을 제공하며, 이를 라이프사이클 단계별로 분류한다.

### 단계별 스킬 목록

메타/디스커버리 단계에는 using-agent-skills가 있다.

Define 단계의 스킬은 다음과 같다.

| 스킬 | 단계 |
|------|------|
| interview-me | Define |
| idea-refine | Define |
| spec-driven-development | Define |

Plan 단계에는 planning-and-task-breakdown 하나가 있다.

Build 단계는 가장 많은 7개 스킬을 포함한다.

| 스킬 | 단계 |
|------|------|
| incremental-implementation | Build |
| test-driven-development | Build |
| context-engineering | Build |
| source-driven-development | Build |
| doubt-driven-development | Build |
| frontend-ui-engineering | Build |
| api-and-interface-design | Build |

Verify 단계에는 browser-testing-with-devtools와 debugging-and-error-recovery가 있다.

Review 단계의 스킬은 다음과 같다.

| 스킬 | 단계 |
|------|------|
| code-review-and-quality | Review |
| code-simplification | Review |
| security-and-hardening | Review |
| performance-optimization | Review |

Ship 단계는 6개 스킬로 구성된다.

| 스킬 | 단계 |
|------|------|
| git-workflow-and-versioning | Ship |
| ci-cd-and-automation | Ship |
| deprecation-and-migration | Ship |
| documentation-and-adrs | Ship |
| observability-and-instrumentation | Ship |
| shipping-and-launch | Ship |

### 에이전트 페르소나와 참조 자료

스킬 외에도 사전 제작된 전문 리뷰어 페르소나가 제공된다.

| 페르소나 | 관점 |
|----------|------|
| code-reviewer | 시니어 스태프 엔지니어 |
| test-engineer | QA/테스트 전문가 |
| security-auditor | 취약점과 위협 평가 |
| web-performance-auditor | Core Web Vitals 감사 |

또한 워크플로우를 보조하는 체크리스트 형태의 참조 자료가 포함된다.

| 참조 자료 | 내용 |
|-----------|------|
| testing-patterns.md | 테스트 구조와 예시 |
| security-checklist.md | 인증, 검증, OWASP Top 10 |
| performance-checklist.md | Core Web Vitals 목표 |
| accessibility-checklist.md | WCAG 2.1 AA 표준 |
| observability-checklist.md | 로깅, 메트릭, 트레이싱 |
| orchestration-patterns.md | 다중 페르소나 조정 |

## 설치와 사용

Claude Code에서는 플러그인 마켓플레이스를 통한 설치를 권장한다.

```
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills
```

로컬 개발 환경에서는 저장소를 클론한 뒤 플러그인 디렉터리를 지정해 실행한다.

```
git clone https://github.com/addyosmani/agent-skills.git
claude --plugin-dir /path/to/agent-skills
```

이 외에도 Cursor, Antigravity CLI, Gemini CLI, Windsurf, OpenCode, GitHub Copilot, Kiro IDE 등 다른 플랫폼을 `/docs`에 문서화된 다양한 설정 방법으로 지원한다.

프로젝트 구조는 다음과 같이 구성된다.

```
agent-skills/
├── skills/                    # 24개 워크플로우 스킬
├── agents/                    # 4개 전문 페르소나
├── references/                # 보조 체크리스트
├── hooks/                     # 세션 라이프사이클
├── .claude/commands/          # Claude Code 슬래시 명령
├── .gemini/commands/          # Gemini CLI 명령
├── commands/                  # Antigravity CLI 명령
├── plugin.json                # 플러그인 매니페스트
└── docs/                      # 설정 가이드
```

## 설계 철학

이 프로젝트는 "process, not prose"를 강조한다.
스킬은 참조 문서가 아니라 실행 가능한 워크플로우라는 의미다.
주요 특징은 다음과 같다.

에이전트가 단계를 건너뛰기 위해 사용하는 변명을 정리한 안티-합리화(anti-rationalization) 표가 포함된다.
가정이 아니라 구체적인 증거를 보장하는 검증 요구 사항이 명시된다.
참조 자료를 분리해 토큰 사용을 최소화하는 점진적 공개(progressive disclosure) 방식을 사용한다.

또한 스킬은 *Software Engineering at Google*의 실무 관행을 통합한다.

| 원칙 | 적용 영역 |
|------|-----------|
| Hyrum's Law | API 설계 |
| The Beyonce Rule | 테스트 |
| Chesterton's Fence | 코드 단순화 |
| Trunk-based development | git 워크플로우 |
| Shift Left, feature flags | CI/CD |

## 결론

Agent Skills는 AI 코딩 에이전트가 명세 작성부터 배포까지 시니어 엔지니어의 규율을 따르도록 강제하는 워크플로우 모음이다.
여섯 단계의 라이프사이클, 24개의 스킬, 4개의 전문 페르소나, 그리고 다양한 체크리스트가 하나의 일관된 프로세스로 묶여 있다.
변명을 표로 정리하고 검증 증거를 요구하는 구조는 에이전트가 지름길을 택하지 못하도록 설계의 핵심을 이룬다.
여러 AI 코딩 플랫폼을 지원하므로 자신의 환경에 맞게 도입해 볼 수 있다.

## Reference

- [addyosmani/agent-skills (GitHub)](https://github.com/addyosmani/agent-skills/)
