---
layout: post
title: "AI 에이전트가 자면서 코딩한다, Ralph Wiggum 기법 실전 가이드"
author: 'Juho'
date: 2026-02-19 02:00:00 +0900
categories: [AI]
tags: [AI, LLM, Dev]
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
2. [문제 정의](#문제-정의)
3. [루프 기반 아키텍처](#루프-기반-아키텍처)
4. [핵심 혁신 요소](#핵심-혁신-요소)
5. [실무 결과](#실무-결과)
6. [개발자 역할 변화](#개발자-역할-변화)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

AI 코딩 에이전트를 연속 루프로 실행하여 자율적으로 개발을 진행하는 방식이 주목받고 있다.
Google 엔지니어 Addy Osmani가 공개한 이 기법은 개발자 커뮤니티에서 Ralph Wiggum 기법으로 불린다.
이 글은 루프 기반 자율 코딩의 작동 원리와 실전 적용 방법을 정리한다.

## 문제 정의

거대한 단일 프롬프트는 AI 모델의 컨텍스트 윈도우를 오버플로우시킨다.
그 결과 품질 저하와 환각(hallucination)이 발생한다.
이를 해결하기 위해 작업을 작은 단위로 나누고 반복 실행하는 접근법이 필요하다.

## 루프 기반 아키텍처

Geoffrey Huntley가 제안한 기본 구조는 다음과 같다.

```bash
while :; do cat PROMPT.md | claude-code ; done
```

각 반복은 다음 사이클을 따른다.

1. 태스크 선택
2. 코딩 실행
3. 테스트 검증
4. 커밋
5. 상태 업데이트
6. 컨텍스트 리셋

매 사이클마다 컨텍스트가 완전히 리셋되므로, 컨텍스트 윈도우 오버플로우 문제를 근본적으로 해결한다.

## 핵심 혁신 요소

### AGENTS.md 파일을 통한 학습 메커니즘

매 사이클마다 메모리를 리셋하면서도 학습을 유지하기 위해 중요 정보를 디스크에 기록한다.
AGENTS.md 파일에는 다음 내용이 포함된다.

- 프로젝트 규칙 및 패턴
- 발견한 주의사항
- 과거 실수 사항

이를 통해 에이전트가 과거 결정을 반복하지 않게 된다.

### 테스트 기반 검증

각 반복마다 테스트를 실행하여 실패 시 태스크를 미완료 상태로 처리한다.
Simon Willison은 코드베이스의 고품질 테스트가 에이전트의 새 테스트 품질에도 영향을 미친다고 지적했다.
기존 테스트의 품질이 높을수록 에이전트가 생성하는 테스트의 품질도 높아진다.

### 인간 감독 단계

완전 자율 방식은 아니다.
매일 아침 생성된 PR을 인간이 리뷰하는 QA 단계가 포함된다.
이를 통해 품질을 보장하고 위험한 변경사항을 사전에 차단한다.

## 실무 결과

Geoffrey Huntley는 이 방식으로 LLM 학습 데이터에 없는 새로운 프로그래밍 언어를 구현했다.
보고된 사례에 따르면, 일반적으로 5만 달러 규모의 프로젝트를 몇백 달러의 API 호출로 완수할 수 있었다.
Ryan Carson은 창업자들이 매일 이런 루프를 10개 이상 운영하게 될 것이라고 예측했다.

## 개발자 역할 변화

이 기법의 확산은 개발자의 역할을 근본적으로 바꾸고 있다.
코드 작성에서 프로세스 큐레이션으로 전환되고 있다.

- 사양 작성
- 결과 리뷰
- 고수준 가이드 제공

개발자는 점차 AI 팀의 엔지니어링 매니저 역할로 변모하고 있다.

## 결론

Ralph Wiggum 기법은 AI 에이전트의 컨텍스트 한계를 루프와 디스크 기반 학습으로 극복하는 실용적인 접근법이다.
테스트 하네스의 품질과 인간 감독이 성공의 핵심 요소이다.
AI 에이전트를 자율 개발에 활용하려는 팀이라면 이 패턴을 검토해볼 가치가 있다.

## Reference

- [Self-Improving Coding Agents - Addy Osmani](https://addyosmani.com/blog/self-improving-agents/)
- [Ralph Wiggum as a software engineer - Geoffrey Huntley](https://ghuntley.com/ralph/)
- [Coding agents and tests - Simon Willison](https://simonwillison.net/2026/Jan/26/tests/)
