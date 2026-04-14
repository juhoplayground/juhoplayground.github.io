---
layout: post
title: "Agent Skills: AI 코딩 에이전트를 위한 프로덕션급 엔지니어링 스킬"
author: 'Juho'
date: 2026-04-14 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, Skill]
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
2. [핵심 철학](#핵심-철학)
3. [개발 라이프사이클 구조](#개발-라이프사이클-구조)
   - [20개 핵심 스킬](#20개-핵심-스킬)
   - [슬래시 커맨드](#슬래시-커맨드)
4. [스킬 구조와 설계 원칙](#스킬-구조와-설계-원칙)
5. [전문 에이전트 페르소나](#전문-에이전트-페르소나)
6. [설치와 통합](#설치와-통합)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Google Cloud AI 디렉터 Addy Osmani가 개발한 [Agent Skills](https://github.com/addyosmani/agent-skills){:target="_blank"}는 AI 코딩 에이전트가 엔지니어링 모범 사례를 일관되게 따르도록 설계된 구조화된 워크플로우 모음이다.
MIT 라이선스 하에 공개된 이 프로젝트는 6개 개발 단계에 걸쳐 20개의 상호 연결된 스킬을 포함하고 있다.
GitHub에서 13,000개 이상의 스타와 1,600개 이상의 포크를 기록하며 활발한 커뮤니티 채택이 이루어지고 있다.

## 핵심 철학

AI 코딩 에이전트는 기본적으로 가장 짧은 경로를 선택한다.
이는 스펙, 테스트, 보안 리뷰 등 소프트웨어 신뢰성을 만드는 실천들을 건너뛰는 것을 의미한다.
Agent Skills는 시니어 엔지니어가 프로덕션 환경에서 사용하는 워크플로우를 인코딩하여 이 문제를 해결한다.
검증은 협상 불가라는 원칙으로 테스트 통과, 빌드 출력 등 증거 기반의 확인을 강조한다.

## 개발 라이프사이클 구조

프레임워크는 작업을 6개 단계로 조직한다.
DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP 순서이며, 각 단계는 에이전트가 현재 수행 중인 작업에 따라 적절한 스킬이 자동으로 활성화된다.

### 20개 핵심 스킬

Define 단계(2개)에는 idea-refine과 spec-driven-development가 있다.
idea-refine은 모호한 개념을 구조화된 발산/수렴 사고로 구체적인 제안으로 변환한다.
spec-driven-development는 구현 전에 목표, 구조, 코드 스타일, 테스팅 기준을 다루는 상세한 PRD를 작성한다.

Plan 단계(1개)에는 planning-and-task-breakdown이 있다.
스펙을 수락 기준과 의존성 매핑이 포함된 작고 검증 가능한 작업으로 분해한다.

Build 단계(6개)에는 incremental-implementation, test-driven-development, context-engineering, source-driven-development, frontend-ui-engineering, api-and-interface-design이 있다.
incremental-implementation은 피처 플래그와 롤백 친화적인 변경으로 얇은 수직 슬라이스를 구현한다.
test-driven-development는 Red-Green-Refactor 패턴을 테스트 피라미드 원칙과 함께 따른다.
source-driven-development는 공식 문서에 기반하여 인용 검증과 함께 프레임워크 결정을 내린다.
api-and-interface-design은 계약 우선 설계, Hyrum's Law, 경계 검증을 적용한다.

Verify 단계(2개)에는 browser-testing-with-devtools와 debugging-and-error-recovery가 있다.

Review 단계(4개)에는 code-review-and-quality, code-simplification, security-and-hardening, performance-optimization이 있다.
code-review-and-quality는 약 100줄 변경 크기 제한과 5축 리뷰 방법론을 적용한다.
code-simplification은 Chesterton's Fence 원칙으로 행동을 보존하면서 복잡도를 줄인다.
security-and-hardening은 OWASP Top 10 취약점을 다루고 3계층 경계 시스템을 구현한다.

Ship 단계(5개)에는 git-workflow-and-versioning, ci-cd-and-automation, deprecation-and-migration, documentation-and-adrs, shipping-and-launch가 있다.

### 슬래시 커맨드

| 커맨드 | 기능 |
|--------|------|
| /spec | 스펙 정의 |
| /plan | 구현 계획 |
| /build | 점진적 코드 작성 |
| /test | 동작 검증 |
| /review | 품질 게이트 |
| /code-simplify | 코드 단순화 |
| /ship | 배포 |

## 스킬 구조와 설계 원칙

각 스킬은 프로세스 중심의 일관된 구조를 따른다.
Overview는 핵심 목적과 범위, When to Use는 트리거 조건과 적용 가능성, Process는 검증 게이트가 포함된 단계별 워크플로우를 제공한다.
Rationalizations는 흔한 변명과 반론을 문서화하고, Red Flags는 불충분한 실행의 경고 신호를 제시한다.
Verification은 테스트 결과, 빌드 출력, 런타임 데이터 등의 증거 요구사항을 정의한다.

스킬에 내장된 Google 엔지니어링 원칙들이 있다.
API 설계에 Hyrum's Law, 테스팅에 Beyoncé Rule과 테스트 피라미드, 코드 리뷰에 약 100줄 변경 크기 규범, 코드 단순화에 Chesterton's Fence, Git 워크플로우에 트렁크 기반 개발, CI/CD에 Shift Left와 피처 플래그를 적용한다.

## 전문 에이전트 페르소나

세 가지 사전 구성된 페르소나가 대상별 관점을 제공한다.

| 페르소나 | 역할 |
|----------|------|
| code-reviewer | 시니어 스태프 엔지니어 관점의 5축 코드 리뷰 |
| test-engineer | QA 전문가 관점의 테스트 전략과 커버리지 분석 |
| security-auditor | 보안 엔지니어 관점의 취약점 탐지와 위협 모델링 |

4개의 보조 체크리스트도 제공된다.
testing-patterns.md, security-checklist.md, performance-checklist.md, accessibility-checklist.md가 있다.

## 설치와 통합

Claude Code에서 권장되는 설치 방법이다.

```bash
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills
```

Cursor, Gemini CLI, Windsurf, OpenCode, GitHub Copilot, Kiro IDE 등 다양한 에이전트 환경과도 호환된다.

## 결론

Agent Skills는 AI 코딩 에이전트의 근본적인 문제인 엔지니어링 규율 부족을 체계적으로 해결한다.
프로덕션 환경에서 검증된 워크플로우를 구조화하여 에이전트가 시니어 엔지니어 수준의 개발 프로세스를 따르도록 강제한다.
단순히 코드를 생성하는 것을 넘어 스펙 정의부터 배포까지 전체 라이프사이클을 관리하는 프레임워크이다.

## Reference

- [Agent Skills — GitHub](https://github.com/addyosmani/agent-skills/)
