---
layout: post
title: "Agent Skills: AI 코딩 에이전트를 위한 24개 프로덕션급 워크플로우"
author: 'Juho'
date: 2026-06-25 00:00:00 +0900
categories: [VibeCoding]
tags: [VibeCoding, Skill, Agent]
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
3. [핵심 내용](#핵심-내용)
   - [6단계 워크플로우](#6단계-워크플로우)
   - [24개 스킬 목록](#24개-스킬-목록)
   - [에이전트 페르소나와 참조 자료](#에이전트-페르소나와-참조-자료)
   - [설치와 지원 에이전트](#설치와-지원-에이전트)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Agent Skills는 AI 코딩 에이전트를 위한 프로덕션급 엔지니어링 스킬 모음이다.
시니어 엔지니어가 소프트웨어를 만들 때 따르는 워크플로우와 품질 게이트, 베스트 프랙티스를 스킬 형태로 인코딩한 저장소다.
README에 따르면 이 스킬들은 "시니어 엔지니어가 소프트웨어를 구축할 때 사용하는 워크플로우, 품질 게이트, 베스트 프랙티스를 인코딩"한 것이다.
전체 24개 스킬은 스펙 정의부터 배포까지 개발 라이프사이클 전 단계를 다룬다.
라이선스는 MIT이며 프로젝트, 팀, 도구에서 자유롭게 사용할 수 있다.

## 배경

AI 코딩 에이전트는 코드를 빠르게 생성하지만 단계마다 품질을 일관되게 유지하기는 어렵다.
Agent Skills는 이 문제를 해결하기 위해 개발 단계별로 따라야 할 관행을 구조화된 스킬로 묶었다.
각 스킬은 단순한 프롬프트가 아니라 특정 작업에 맞는 워크플로우와 검증 규칙을 담고 있다.
메타 스킬이 들어오는 작업을 적절한 스킬 워크플로우로 매핑하고 공통 운영 규칙을 정의하는 구조다.
이를 통해 에이전트는 스펙 작성, 구현, 테스트, 리뷰, 배포 전 과정에서 일정한 수준을 유지한다.

## 핵심 내용

### 6단계 워크플로우

개발 라이프사이클은 6개의 단계를 거쳐 흐른다.
각 단계는 슬래시 커맨드로 호출된다.

| 단계 | 커맨드 | 역할 |
|------|--------|------|
| DEFINE | /spec | 무엇을 만들지 정의 |
| PLAN | /plan | 작업 계획 수립 |
| BUILD | /build | 구현 |
| VERIFY | /test | 검증과 테스트 |
| REVIEW | /review | 코드 리뷰 |
| SHIP | /ship | 배포 |

이 외에도 웹 성능 감사를 위한 /webperf, 코드 단순화를 위한 /code-simplify 커맨드가 추가로 제공된다.
또한 /build auto 모드는 계획을 생성하고 모든 작업을 한 번의 승인된 패스로 구현한다.
이 자동 빌드 모드는 작업 사이에 사람이 끼어드는 단계를 없애면서도 개별 테스트 주도 커밋과 검증 게이트는 그대로 유지한다.

### 24개 스킬 목록

24개 스킬은 워크플로우 단계에 맞춰 묶여 있다.
메타 스킬 1개를 시작으로 Define 3개, Plan 1개, Build 7개, Verify 2개, Review 4개, Ship 6개로 구성된다.

다음은 메타와 Define, Plan 단계 스킬이다.

| 스킬 | 설명 |
|------|------|
| using-agent-skills | 들어오는 작업을 올바른 스킬 워크플로우로 매핑하고 공통 운영 규칙 정의 |
| interview-me | 한 번에 한 질문씩 인터뷰해 사용자가 원하는 바를 약 95% 확신까지 추출 |
| idea-refine | 발산과 수렴적 사고로 모호한 아이디어를 구체적 제안으로 전환 |
| spec-driven-development | 코드 작성 전에 목표, 커맨드, 구조, 코드 스타일, 테스트, 경계를 담은 PRD 작성 |
| planning-and-task-breakdown | 스펙을 작고 검증 가능한 작업으로 분해하고 수락 기준과 의존성 순서 정의 |

다음은 Build 단계 7개 스킬이다.

| 스킬 | 설명 |
|------|------|
| incremental-implementation | 얇은 수직 슬라이스로 구현, 테스트, 검증, 커밋 |
| test-driven-development | Red-Green-Refactor, 테스트 피라미드(80/15/5), 테스트 크기, DRY보다 DAMP |
| context-engineering | 적절한 시점에 적절한 정보 제공, 규칙 파일, 컨텍스트 패킹, MCP 통합 |
| source-driven-development | 모든 프레임워크 결정을 공식 문서에 근거, 검증과 출처 인용 |
| doubt-driven-development | 진행 중인 모든 비자명한 결정에 대한 적대적 신규 컨텍스트 리뷰 |
| frontend-ui-engineering | 컴포넌트 아키텍처, 디자인 시스템, 상태 관리, 반응형, WCAG 2.1 AA 접근성 |
| api-and-interface-design | 계약 우선 설계, Hyrum의 법칙, One-Version Rule, 에러 의미, 경계 검증 |

다음은 Verify 단계 2개 스킬이다.

| 스킬 | 설명 |
|------|------|
| browser-testing-with-devtools | Chrome DevTools MCP로 실시간 런타임 데이터, DOM 검사, 콘솔 로그, 네트워크 추적 |
| debugging-and-error-recovery | 5단계 분류: 재현, 위치 파악, 축소, 수정, 가드 |

다음은 Review 단계 4개 스킬이다.

| 스킬 | 설명 |
|------|------|
| code-review-and-quality | 5축 리뷰, 변경 크기(약 100줄), 심각도 라벨, 리뷰 속도 규범 |
| code-simplification | Chesterton의 울타리, Rule of 500, 동작 보존하며 복잡도 감소 |
| security-and-hardening | OWASP Top 10 예방, 인증 패턴, 시크릿 관리, 의존성 감사 |
| performance-optimization | 측정 우선 접근, Core Web Vitals 목표, 프로파일링, 번들 분석 |

다음은 Ship 단계 6개 스킬이다.

| 스킬 | 설명 |
|------|------|
| git-workflow-and-versioning | 트렁크 기반 개발, 원자적 커밋, 변경 크기(약 100줄), 커밋을 저장점으로 활용 |
| ci-cd-and-automation | Shift Left, Faster is Safer, 피처 플래그, 품질 게이트 파이프라인 |
| deprecation-and-migration | 코드는 부채라는 관점, 강제 대 권고 폐기, 마이그레이션 패턴 |
| documentation-and-adrs | 아키텍처 결정 기록, API 문서, 인라인 문서화 표준 |
| observability-and-instrumentation | 구조화 로깅, RED 메트릭, OpenTelemetry 추적, 증상 기반 알림 |
| shipping-and-launch | 출시 전 체크리스트, 피처 플래그 라이프사이클, 단계적 롤아웃, 롤백 절차 |

### 에이전트 페르소나와 참조 자료

스킬과 별도로 4개의 전문가 에이전트 페르소나가 제공된다.
각 페르소나는 특정 역할의 시각으로 작업을 수행한다.

| 페르소나 | 역할 |
|----------|------|
| code-reviewer | 시니어 스태프 엔지니어 관점의 코드 리뷰 |
| test-engineer | QA 스페셜리스트로서 테스트 전략과 커버리지 담당 |
| security-auditor | 보안 엔지니어로서 취약점 탐지와 위협 모델링 |
| web-performance-auditor | 웹 성능 엔지니어로서 Core Web Vitals 감사 |

저장소에는 스킬이 참조하는 자료 파일도 포함된다.
정의 완료 기준(definition-of-done.md), 테스트 패턴(testing-patterns.md), 보안 체크리스트(security-checklist.md), 성능 체크리스트(performance-checklist.md), 접근성 체크리스트(accessibility-checklist.md), 관측성 체크리스트(observability-checklist.md), 오케스트레이션 패턴(orchestration-patterns.md)이 들어 있다.

### 설치와 지원 에이전트

Claude Code에서는 마켓플레이스를 통해 설치한다.

```
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills
```

Claude Code 외에도 여러 에이전트를 지원한다.

| 도구 | 설치 방법 |
|------|-----------|
| Cursor | 스킬을 .cursor/rules/ 로 복사 |
| Antigravity CLI | agy plugin install 명령으로 저장소 설치 |
| Gemini CLI | gemini skills install 명령에 경로 지정 |
| Windsurf | rules 설정에 추가 |
| OpenCode | AGENTS.md 사용 |
| GitHub Copilot | agents 디렉터리의 에이전트 정의 사용 |
| Kiro IDE | .kiro/skills/ 아래에 스킬 배치 |
| 로컬 개발 | git clone 후 claude --plugin-dir 사용 |

지원 대상은 Claude Code, Cursor, Antigravity CLI, Gemini CLI, Windsurf, OpenCode, GitHub Copilot, Kiro IDE를 비롯해 시스템 프롬프트나 지시 파일을 받아들이는 다른 에이전트들이다.

## 의미와 시사점

Agent Skills는 AI 코딩의 결과물 품질을 사람의 즉흥적 판단이 아니라 구조화된 워크플로우에 의존하게 만든다.
스펙부터 배포까지 6단계가 명확히 나뉘어 있어 에이전트가 어느 단계에서 무엇을 검증해야 하는지가 분명하다.
특히 source-driven-development와 doubt-driven-development처럼 출처 검증과 적대적 리뷰를 강제하는 스킬은 에이전트의 환각과 즉흥적 결정을 줄이는 장치다.
여러 에이전트 도구를 공통으로 지원하므로 특정 도구에 종속되지 않고 동일한 엔지니어링 관행을 적용할 수 있다.
바이브 코딩이 빠른 생성에서 그치지 않고 프로덕션급 품질로 이어지려면 이런 워크플로우 인코딩이 핵심이 된다.

## 결론

Agent Skills는 24개의 스킬과 6단계 워크플로우, 4개의 전문가 페르소나로 AI 코딩 에이전트의 작업을 프로덕션급으로 끌어올린다.
Define, Plan, Build, Verify, Review, Ship 단계마다 검증 가능한 관행이 스킬로 묶여 있다.
Claude Code 마켓플레이스를 비롯해 Cursor, Gemini CLI 등 다양한 도구에서 설치해 사용할 수 있다.
MIT 라이선스로 공개되어 있어 자신의 프로젝트와 팀 환경에 바로 도입할 수 있다.

## Reference

- [agent-skills (GitHub)](https://github.com/addyosmani/agent-skills/)
