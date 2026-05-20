---
layout: post
title: "Code with Claude SF 컨퍼런스 정리 - 19개 세션이 보여준 Anthropic의 차세대 개발 청사진"
author: 'Juho'
date: 2026-05-20 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM, Skill]
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
2. [컨퍼런스 구성](#컨퍼런스-구성)
   - [세 가지 트랙](#세-가지-트랙)
   - [주요 연사](#주요-연사)
3. [Claude의 진화 방향](#claude의-진화-방향)
4. [핵심 발표 내용](#핵심-발표-내용)
   - [Claude Code 업데이트](#claude-code-업데이트)
   - [Claude Managed Agents](#claude-managed-agents)
   - [엔터프라이즈 사례](#엔터프라이즈-사례)
5. [의미와 시사점](#의미와-시사점)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Anthropic이 2026년 5월 6일 샌프란시스코에서 "Code with Claude" 개발자 컨퍼런스를 개최했습니다.
현장과 가상 참가 옵션을 모두 제공했으며, 총 19개의 세션이 진행됐습니다.
세션은 Research, Claude Platform, Claude Code 세 트랙으로 구성됐으며, 행사 종료 후 모든 녹화본이 검색 가능한 아카이브 형태로 공개됐습니다.
이번 컨퍼런스는 코드 작성을 넘어 검증, 보안, 메모리 관리, 에이전트 조율로 옮겨가는 차세대 개발 환경의 청사진을 제시했습니다.

## 컨퍼런스 구성

행사는 키노트, 기술 세션, 파이어사이드 챗 등 다양한 포맷으로 진행됐습니다.
참가자는 현장 참석과 라이브스트림 등록 중 선택할 수 있었습니다.

### 세 가지 트랙

| 트랙 | 주요 주제 |
|------|----------|
| Research | 모델 연구, 메모리, 정렬 |
| Claude Platform | API, 캐싱, Managed Agents |
| Claude Code | 개발자 도구, 워크플로우, 에이전트 |

### 주요 연사

오프닝 키노트는 Boris Cherny, Ami Vora, Dianne Penn이 함께 진행했습니다.
Dickson Tsai는 "What's new in Claude Code" 세션에서 신규 기능을 소개했습니다.
Mahesh Murag는 에이전트 메모리 시스템을 다뤘습니다.
Mario Rodriguez와 Brad Abrams는 대규모 환경에서의 캐싱 전략을 발표했습니다.
Jess Yan과 Lance Martin은 Claude Managed Agents를 활용한 프로덕션 배포를 다뤘습니다.
파이어사이드 챗에는 Anthropic 공동 창업자 Dario Amodei와 Daniela Amodei가 등장했습니다.

## Claude의 진화 방향

컨퍼런스 전반에서 강조된 Claude의 진화 경로는 네 가지로 요약됩니다.
첫째, 더 긴 작업을 일관성 있게 수행하는 능력입니다.
둘째, 세션을 가로지르는 장기 메모리 시스템 구축입니다.
셋째, 도구 활용 범위의 확대입니다.
넷째, 작업 결과를 자동으로 검증하는 체계의 강화입니다.

이러한 방향은 단순히 "더 똑똑한 모델"이 아니라, 장시간 자율적으로 운영되는 에이전트를 안전하게 관리하기 위한 인프라에 가깝습니다.

## 핵심 발표 내용

19개 세션 중 개발자 워크플로우와 직접적으로 연결되는 발표를 살펴봅니다.

### Claude Code 업데이트

Claude Code는 데스크탑 IDE 중심에서 분산된 워크플로우 환경으로 확장되고 있습니다.

| 기능 | 설명 |
|------|------|
| 원격 제어(Remote Control) | 기기 간 세션 연속성 제공 |
| 자동 모드(Auto Mode) | 안전한 도구 자동 실행 |
| 워크트리(Worktree) | 병렬 작업 지원 |
| 자동 메모리 | 세션 간 컨텍스트 유지 |

자동 모드는 사용자 개입 없이 도구를 안전하게 실행하는 정책 기반 실행 환경을 제공합니다.
워크트리 지원은 동일한 저장소에서 여러 변경을 병렬로 시도하는 패턴을 공식화합니다.

### Claude Managed Agents

Claude Managed Agents는 장시간 운영되는 에이전트를 위해 설계된 호스팅 환경입니다.

핵심 기능은 다음과 같습니다.

- 장시간 운영 에이전트 지원
- 보안 및 접근 제어 통합
- 감사 가능한 메모리 관리

이는 자율 에이전트가 프로덕션에서 실제 비즈니스 자산에 접근해야 할 때 필요한 거버넌스 계층을 표준화하려는 시도입니다.

### 엔터프라이즈 사례

GitHub Copilot 운영 사례는 대규모 환경에서의 캐싱 효과를 보여줬습니다.
프롬프트 캐싱을 통해 비용과 지연을 동시에 줄였으며, 목표 캐시 적중률 94-96%를 달성한 것으로 보고됐습니다.

Asana는 "AI teammates" 개념으로 조직 협업 워크플로우에 Claude를 통합했습니다.
Datadog는 엔지니어의 90%가 AI 도구를 활용한다고 공유했습니다.

## 의미와 시사점

이번 컨퍼런스는 두 가지 메시지를 분명히 드러냅니다.
첫째, 모델 자체의 성능보다 검증, 보안, 메모리 관리, 에이전트 조율이 새로운 병목으로 떠올랐습니다.
둘째, 개발 도구 생태계가 단일 IDE 중심에서 분산된 에이전트 협업 환경으로 빠르게 재편되고 있습니다.

캐싱 적중률 94-96%라는 수치는 자율 에이전트 운영의 경제성이 캐싱 전략에 의해 결정된다는 점을 시사합니다.
또한 Datadog의 90% 도구 활용률은 AI 보조 개발이 이미 일부 조직에서 표준 워크플로우가 됐음을 보여줍니다.

## 결론

Code with Claude SF는 코드 작성 중심에서 검증·메모리·조율 중심으로 옮겨가는 개발 환경의 전환점을 압축적으로 제시했습니다.
원격 제어, 자동 모드, 워크트리, 자동 메모리 등 Claude Code의 신규 기능은 이러한 전환을 도구 수준에서 뒷받침합니다.
Managed Agents의 등장은 프로덕션 자율 에이전트 운영을 위한 인프라 계층을 표준화하려는 시도로 읽힙니다.
모든 세션 녹화본이 공개돼 있어 트랙별 관심사에 맞춰 학습 자료로 활용할 수 있습니다.

## Reference

- [Code with Claude San Francisco](https://claude.com/code-with-claude/san-francisco/)
