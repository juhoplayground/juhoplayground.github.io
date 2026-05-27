---
layout: post
title: "SpecGuard - AI 코딩 전 명세를 검증하는 Validation-First Workflow 도구"
author: 'Juho'
date: 2026-05-27 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Documentation]
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
2. [SpecGuard가 해결하는 문제](#specguard가-해결하는-문제)
3. [핵심 기능과 워크플로우](#핵심-기능과-워크플로우)
   - [2단계 리뷰 프로세스](#2단계-리뷰-프로세스)
   - [통합 옵션](#통합-옵션)
4. [설치와 빠른 시작](#설치와-빠른-시작)
5. [성능 벤치마크](#성능-벤치마크)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

한국 개발자 KoreaNirsa가 SpecGuard라는 오픈소스 도구를 공개했다.
AI 코딩 에이전트가 결함 있는 코드를 만들기 전에 약한 명세를 차단하는 CLI 검증 도구다.
이 도구는 Validation-First Workflow(VFW)라는 새로운 패러다임을 구현하며, 명세를 강화해 외부 코딩 에이전트로 핸드오프하기 전에 검증한다.

저자의 문제 의식은 명확하다.
AI 생성 코드의 문제는 "코딩 능력의 한계"가 아니라 "AI에 전달되는 불완전한 명세"에서 발원한다는 관찰이다.
결함 있는 명세는 코드 품질 저하와 요구사항과 구현 간 격차 증가로 이어진다.

## SpecGuard가 해결하는 문제

SpecGuard는 AI 보조 개발 과정에서 발생하는 핵심 격차들을 다룬다.

| 격차 유형 | 설명 |
|----------|------|
| 모호하거나 불완전한 요구사항 | AI가 추측으로 채워 잘못 구현 |
| 명시되지 않은 가정 | 코드와 의도 사이 불일치 |
| 누락된 접근 제어 정의 | 보안 취약점 가능성 |
| 약한 합격 기준 | 검수 불가능한 결과 |
| 정의되지 않은 오류 처리/상태 전환 | 예외 케이스에서 실패 |
| 계약과 동작의 불일치 | 통합 시 버그 |

도구는 구현 전 검증에 집중한다.
이를 통해 코드 생성이 시작되기 전에 명세의 약점을 잡아낼 수 있다.

## 핵심 기능과 워크플로우

SpecGuard의 가치는 두 단계의 리뷰와 다양한 통합 옵션에서 나온다.

### 2단계 리뷰 프로세스

**SpecGuard Review**는 구현 전 휴리스틱 분석으로 작동한다.
중대한 발견은 차단하고, 다른 이슈는 경고로 보고한다.

**SpecGuard PR Review**는 승인된 명세와 PR 변경 사항을 비교한다.
조언성 댓글을 게시해 명세와 실제 구현이 어긋나지 않는지 검증한다.

결과는 세 가지 범주로 나뉜다.

| 결과 | 의미 |
|------|------|
| READY | 구현 진행 가능 |
| READY_WITH_WARNINGS | 주의해서 진행 |
| NOT_READY | 명세 수정 필요 |

### 통합 옵션

SpecGuard는 세 가지 방식으로 사용 가능하다.

| 통합 방식 | 설명 |
|----------|------|
| 독립 CLI 도구 | 로컬에서 명세 검증 |
| Codex 앱 플러그인 | Python 3.11-3.13 호환, 에디터 내 검사 |
| GitHub Actions | PR 자동 검증 워크플로우 |

검증 차원은 인증과 권한 경계, 데이터 소유권 범위, 멱등성과 경쟁 조건 처리, 상태 전환 규칙, 외부 부작용과 재시도 정책을 포함한다.

언어 지원은 영어 문서가 1차이며, 한국어는 결정론적 안전 체크를 포함한다.

## 설치와 빠른 시작

설치와 초기 설정은 단순하다.

```bash
pip install spec-guard
specguard auth setup --mode codex --model gpt-5.4
specguard init your-feature-name
specguard run specs/your-feature-name
```

도구는 패키지된 예시 명세를 포함해 프로덕션 명세 생성 전 테스트가 가능하다.

워크플로우 전체는 다음 순서로 흐른다.
Discovery → Spec Package → Technical Design → SpecGuard Review → Test → Contract → Implementation Handoff → External AI Implementation → Pull Request → SpecGuard PR Review

사용자는 명세 소유권을 유지하면서, SpecGuard가 구현 핸드오프까지 검증 경로를 지킨다.

## 성능 벤치마크

벤치마크는 영어와 한국어로 각각 98개의 명세 패키지를 현실적 도메인에 걸쳐 평가했다.

| 지표 | 영어 | 한국어 |
|------|------|--------|
| 약한 명세 차단율 | 96.9% | 100.0% |
| 거짓 양성률 | 0.0% | 0.0% |

한국어 100% 차단율은 한국 개발자가 작성한 도구의 강점을 보여준다.

아키텍처는 Python으로 작성됐다.
빠른 로컬 검증을 위해 휴리스틱 분석을 사용하고, 상세 리뷰를 위한 LLM 통합은 선택 사항이다.
구성은 OpenAI API 자격증명과 모델 선택을 요구한다.

최신 기능(v0.4.0)에는 Codex 플러그인 통합, GitHub Actions PR 리뷰 워크플로우, CLI 기반 휴리스틱 체크가 기본 빠른 경로로 포함된다.

## 결론

AI 코딩의 다음 단계는 "더 좋은 코드 생성"이 아니라 "더 좋은 명세 검증"일 수 있다.
SpecGuard는 코드 생성 단계에서 후속 검증하는 대신, 명세 단계에서 사전 차단하는 접근을 구현한다.
한국어 100%, 영어 96.9%의 차단율은 휴리스틱 기반 접근법의 실효성을 입증한다.
명세를 작성하고 → 검증하고 → 필요시 강화하고 → 구현 에이전트에게 핸드오프하는 워크플로우가 정착되면, AI 코딩의 신뢰성이 한 단계 올라갈 수 있다.

## Reference

- [SpecGuard - GitHub Repository](https://github.com/KoreaNirsa/spec-guard)
