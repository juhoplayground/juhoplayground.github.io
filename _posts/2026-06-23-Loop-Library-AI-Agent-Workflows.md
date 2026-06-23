---
layout: post
title: "Loop Library: 검증과 멈춤 조건을 갖춘 AI 에이전트 루프 모음"
author: 'Juho'
date: 2026-06-23 00:00:00 +0900
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
2. [Loop Library의 구조](#loop-library의-구조)
   - [분류 체계](#분류-체계)
   - [루프의 공통 구성 요소](#루프의-공통-구성-요소)
3. [대표적인 루프 사례](#대표적인-루프-사례)
   - [엔지니어링과 평가 루프](#엔지니어링과-평가-루프)
   - [고급 멀티 에이전트 루프](#고급-멀티-에이전트-루프)
4. [결론](#결론)
5. [Reference](#reference)

## 개요

Loop Library는 Forward Future가 운영하는, 반복 가능한 AI 에이전트 워크플로우 모음이다.
핵심 컨셉은 "명확한 검증 조건과 멈춤 조건을 갖춘 실용적인 AI 에이전트 프롬프트"를 복사해서 쓸 수 있게 하는 것이다.
즉 단발성 프롬프트가 아니라, AI가 어떤 기준에 도달할 때까지 스스로 반복하는 루프(loop)를 패턴으로 정리한 라이브러리다.
웹사이트에서 직접 검색해 복사할 수도 있고, 다음과 같이 스킬로 설치할 수도 있다.

```bash
npx skills add Forward-Future/loop-library --skill loop-library -g
```

## Loop Library의 구조

### 분류 체계

루프들은 직무와 목적에 따라 다섯 가지 카테고리로 나뉜다.

| 카테고리 | 대상 |
|------|------|
| Engineering | 테스트 커버리지, 성능, 에러 처리, 로깅 |
| Evaluation | 전체 제품 평가, UX 검증, 약속 검증 |
| Operations | 데이터 정리, 배포 관리, 복구 검증 |
| Content | SEO 개선, 팟캐스트 제작, 콘텐츠 최적화 |
| Design | UI/UX 개선, 접근성, 시각적 재구성 |

### 루프의 공통 구성 요소

모든 루프는 비슷한 골격을 공유한다.

| 요소 | 설명 |
|------|------|
| 목적 | 루프의 최종 목표 |
| 실행 방식 | 단계별 절차 |
| 검증 조건 | 성공과 실패의 기준 |
| 멈춤 조건 | 언제 종료할 것인가 |
| 결과물 | 최종 산출물 |

이를 반복 구조로 풀면 다음 패턴이 된다.

```
① 현재 상태 평가
② 가장 약한 부분 또는 영향도 높은 부분 식별
③ 한 가지 변경 실행
④ 검증과 측정
⑤ 기준 충족 또는 더 이상 진전이 없을 때까지 반복
```

핵심은 "한 번에 하나씩 변경하고, 매번 검증한다"는 점이다.
검증 조건과 멈춤 조건이 명시되어 있어야 AI가 폭주하지 않고 수렴한다.

## 대표적인 루프 사례

### 엔지니어링과 평가 루프

라이브러리에는 61개 이상의 루프가 있으며, 실무에서 바로 쓸 만한 사례가 많다.

| 루프 | 목적과 방식 |
|------|------|
| The Docs Sweep | 코드와 문서 동기화 유지, 전체 검토 후 오래된 문서 업데이트하고 PR 개설 |
| The Production Error Sweep | 프로덕션 로그 검토로 근본 원인 추적, 수정 후 PR 제출 |
| The 100% Test Coverage Loop | 100% 커버리지 도달까지 반복적으로 테스트 추가 |
| The Full Product Evaluation Loop | 프로덕션 규모 로컬 데이터 구축, 실제 사용자처럼 테스트 후 공통 원인 통합 수정 |
| The Promise-to-Proof Loop | 마케팅과 문서의 모든 주장을 나열하고 현재 제품과 대조해 위험도 순으로 수정 |

각 루프는 "약한 곳을 찾아 하나 고치고 다시 검증"이라는 동일한 리듬을 따른다.

### 고급 멀티 에이전트 루프

여러 AI를 엮어 서로를 검증하게 하는 루프도 정리되어 있다.

| 루프 | 핵심 아이디어 |
|------|------|
| The Autonomy-Loop Builder-Reviewer Loop | 빌더가 변경하고 빨강에서 초록으로 테스트, 검토자가 게이트 재실행, 둘 다 통과해야 승인 |
| The Multi-LLM Convergence Loop | 서로 다른 두 AI가 같은 버전을 승인할 때까지 교대로 검토와 수정 반복 |
| The Codex Completion-Contract Loop | 장기 작업에서 완료 요건을 사전 정의하고 모두 증명되어야 완료 처리 |
| The Goal Forge Loop | 아이디어를 SPEC.md 완료 조건과 GOAL.md 작업 계획으로 변환 후 승인받고 시작 |

특히 Builder-Reviewer나 Multi-LLM Convergence 패턴은, AI가 자기 작업을 스스로 옹호하는 편향을 깨기 위해 별도 인스턴스가 검증하게 만든다는 점에서 의미가 있다.

## 결론

Loop Library의 가치는 개별 프롬프트가 아니라 사고방식에 있다.
"한 번 잘 시키면 끝"이 아니라, 검증 게이트를 정의하고 그 기준에 도달할 때까지 AI가 반복하게 만드는 구조다.
사람의 역할은 작업을 직접 수행하는 것에서, 검증 조건과 멈춤 조건을 설계하고 루프의 경계를 관리하는 것으로 이동한다.
직무별로 정리된 61개 이상의 루프는, 이 사고방식을 자신의 업무에 옮겨 적용하기 위한 좋은 출발점이 된다.

## Reference

- [Loop Library — Forward Future](https://signals.forwardfuture.ai/loop-library/)
