---
layout: post
title: "Deep-CoreInterface-Skill — Claude Code용 소프트웨어 펀더멘털 스킬 패키지"
author: 'Juho'
date: 2026-05-23 00:00:00 +0900
categories: [Dev]
tags: [Dev, Skill, Skill Development]
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
2. [스킬의 정체성과 핵심 명제](#스킬의-정체성과-핵심-명제)
3. [패키지 구조](#패키지-구조)
   - [V1 Monolith](#v1-monolith)
   - [V2 Bundle](#v2-bundle)
4. [설치와 사용](#설치와-사용)
5. [7단계 워크플로우](#7단계-워크플로우)
6. [네 가지 핵심 실천](#네-가지-핵심-실천)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Deep-CoreInterface-Skill은 Matt Pocock의 발표 "Software Fundamentals Matter More Than Ever"를 기반으로 만들어진 Claude Code용 스킬 패키지다.
여섯 가지 피해야 할 함정과 네 가지 긍정적 실천을 묶고, 그 위에 메타 계획(metaphysical planning)과 신념적 기반 개념을 더한 형태로 구성되어 있다.

## 스킬의 정체성과 핵심 명제

이 스킬의 단일한 명제는 다음과 같다.
"Good intention + clarity = transcendence."
좋은 의도와 명료함이 만나면 코드가 한 차원 위로 올라간다는 의미다.

사용 대상은 셋이다.

- AI 보조 코딩 중에도 펀더멘털을 유지하고 싶은 개발자
- 자기가 짓는 것이 옳은 대상인지 의심하는 기획자
- Claude Code 안에서 자동화된 워크플로우 가이드를 원하는 사용자

엔지니어링, 메타피지컬, 신념적 관점이 한 패키지 안에 묶인다는 점이 이 스킬의 차별점이다.

## 패키지 구조

두 가지 배포 형태로 제공된다.

### V1 Monolith

한 개의 SKILL.md 파일이다.
모든 원칙이 압축되어 있어 빠른 배포에 적합하다.

### V2 Bundle

핵심 SKILL.md에 다섯 개 보조 문서를 더한 형태다.

| 파일 | 역할 |
|------|------|
| PHILOSOPHY.md | 메타피지컬 및 신념적 기반 |
| SIX_PITFALLS.md | 피해야 할 여섯 가지 함정 |
| CORE_PRACTICES.md | 네 가지 긍정적 원칙 |
| GRILL_ME_PROTOCOL.md | 의도 검증 템플릿 |
| DEEP_MODULE_REVIEW.md | Deep/Shallow 평가 체크리스트 |

영문판은 en/ 디렉터리 아래에 따로 제공된다.

## 설치와 사용

| 환경 | 방법 |
|------|------|
| Claude Code | 스킬 파일을 ~/.claude/skills/software-fundamentals/ 로 복사 |
| Claude.ai | SKILL.md를 시스템 프롬프트나 프로젝트 메모리에 통합 |
| 일반 참조 | PHILOSOPHY → SIX_PITFALLS → CORE_PRACTICES → GRILL_ME_PROTOCOL → DEEP_MODULE_REVIEW 순으로 점진 학습 |

라이선스는 MIT이며, 원본 Matt Pocock 발표에 크레딧을 명시한다.

## 7단계 워크플로우

스킬은 작업을 7단계로 나눈다.

1. Grill Me — 의도 검증 단계
2. Spec Lock — 명확한 사양 고정
3. Module Design — Deep Modules와 Vertical Slices
4. Dependency Check — 세 가지 질문으로 의존성 점검
5. Build — TDD 기반 구현
6. Review — 의식적인 코드 리뷰
7. Daily Design — 체계적인 설계 투자 습관

각 단계는 AI 보조 코딩 환경에서도 펀더멘털이 무너지지 않도록 잡아두는 체크포인트 역할을 한다.

## 네 가지 핵심 실천

스킬이 권장하는 긍정적 실천은 다음과 같다.

| 실천 | 기원 |
|------|------|
| Ubiquitous Language | Eric Evans |
| Vertical Slices | XP/Agile |
| Test-Driven Development | Kent Beck |
| Deep Modules | John Ousterhout |

각 항목 모두 새로운 개념은 아니지만, AI가 코드를 빠르게 쏟아내는 시점에 다시 강조될 가치가 있다는 것이 이 스킬의 입장이다.

## 결론

Deep-CoreInterface-Skill은 AI 보조 코딩의 속도를 인정하면서도, 그 안에서 펀더멘털을 지키기 위한 구체적 절차를 묶어둔 패키지다.
함정 6개와 실천 4개를 결합하고, 메타 계획적 시각으로 그 위를 덮는 구조가 특징이다.
Claude Code 사용자는 단순히 SKILL.md를 ~/.claude/skills/ 아래에 두는 것만으로도 7단계 워크플로우와 Deep Module 리뷰 체크리스트를 그대로 가져올 수 있다.
"Good intention + clarity = transcendence."라는 단 한 줄의 명제가, AI가 모든 것을 더 빨리 만들어 주는 시기에 어떤 무게를 가질 수 있는지에 대한 실험이라고 볼 수 있다.

## Reference

- [Deep-CoreInterface-Skill (GitHub)](https://github.com/okneo31/Deep-CoreInterface-Skill){:target="_blank"}
