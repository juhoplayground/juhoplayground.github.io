---
layout: post
title: "Anthropic Managed Agents: 두뇌와 손을 분리하는 에이전트 아키텍처"
author: 'Juho'
date: 2026-04-13 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, LLM]
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
3. [핵심 아키텍처](#핵심-아키텍처)
   - [세 가지 가상화 요소](#세-가지-가상화-요소)
   - [Pets vs Cattle 문제](#pets-vs-cattle-문제)
4. [분리의 효과](#분리의-효과)
   - [하네스의 무상태화](#하네스의-무상태화)
   - [보안 강화](#보안-강화)
   - [컨텍스트 관리 전략](#컨텍스트-관리-전략)
5. [성능 개선과 확장성](#성능-개선과-확장성)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Anthropic이 장기 실행 에이전트를 위한 호스팅 서비스인 Managed Agents의 아키텍처를 공개했다.
핵심 설계 원칙은 Claude의 추론 엔진(두뇌)과 코드 실행 환경(손)을 분리하는 것이다.
운영체제가 하드웨어를 추상화하는 방식을 모델로 삼아, 세션, 하네스, 샌드박스를 독립적으로 분리했다.

## 배경

초기 Managed Agents는 모든 구성 요소를 하나의 컨테이너에 통합하는 방식이었다.
모델이 발전하면서 하네스에 인코딩된 AI 능력에 대한 가정들이 낡아졌고, 불필요한 오버헤드를 생성하게 되었다.
컨테이너가 실패하면 세션이 사라지고, 보안 제약으로 인해 디버깅도 불가능했다.
이러한 결합된 아키텍처는 수작업으로 관리해야 하는 Pet 서버를 만들어냈다.

## 핵심 아키텍처

### 세 가지 가상화 요소

Managed Agents는 세 가지 핵심 요소를 가상화하여 분리한다.

| 요소 | 역할 |
|------|------|
| Session | 발생한 모든 이벤트를 추적하는 append-only 이벤트 로그 |
| Harness | Claude를 호출하고 도구 출력을 인프라로 라우팅하는 루프 |
| Sandbox | 코드 실행과 파일 조작을 위한 실행 환경 |

이 설계는 운영체제가 하드웨어를 프로세스와 파일이라는 내구성 있는 추상화로 가상화한 방식과 동일한 패턴이다.

### Pets vs Cattle 문제

초기 결합 아키텍처에서는 하나의 컨테이너가 모든 것을 담당했다.
이로 인해 손실될 수 없는 Pet 서버가 만들어졌다.
컨테이너 실패 시 세션이 사라지고, 사용자 데이터 보안 제약으로 디버깅이 불가능했다.
분리 아키텍처는 이 문제를 근본적으로 해결한다.

## 분리의 효과

### 하네스의 무상태화

하네스는 더 이상 컨테이너 안에 존재하지 않는다.
샌드박스를 execute(name, input) → string 인터페이스로 호출한다.
실패한 하네스는 wake(sessionId)와 getSession(id)를 통해 복구된다.
이벤트는 emitEvent(id, event)를 통해 내구성 있게 기록된다.

### 보안 강화

자격증명은 생성된 코드가 실행되는 샌드박스에서 절대 도달할 수 없다.
Git 토큰은 샌드박스 초기화 중에 번들링된다.
OAuth 토큰은 보안 볼트에 저장되고 MCP 프록시를 통해 접근한다.
하네스는 자격증명에 대해 무관심(credential-agnostic)하게 유지된다.

### 컨텍스트 관리 전략

Claude의 컨텍스트 윈도우 안에 컨텍스트를 저장하는 대신, 세션이 외부 객체로 작동한다.
getEvents() 인터페이스는 유연한 검색을 가능하게 한다.
마지막 읽기 위치에서 이어서 읽거나, 특정 시점 이전으로 되감거나, 특정 행동 전의 컨텍스트를 재검토할 수 있다.
이벤트를 하네스에서 변환한 후 Claude에 전달하는 것도 가능하다.
이 분리는 핵심 인터페이스 변경 없이 미래의 컨텍스트 엔지니어링을 가능하게 한다.

## 성능 개선과 확장성

분리 아키텍처의 성능 개선 효과는 극적이다.

| 지표 | 개선폭 |
|------|--------|
| p50 TTFT | 약 60% 감소 |
| p95 TTFT | 90% 이상 감소 |

컨테이너는 필요할 때만 프로비저닝되어 세션 로그에서 즉시 추론을 시작할 수 있다.

Many Brains 확장에서는 무상태 하네스가 수평적으로 확장된다.
각 두뇌가 전용 인프라를 필요로 한다는 가정이 없다.

Many Hands 확장에서는 각 실행 환경이 도구 인터페이스가 된다.
Claude가 여러 샌드박스, 컨테이너, 외부 시스템에 걸쳐 작업을 보내는 위치를 결정한다.

핵심 설계 원칙은 인터페이스의 형태에 대해서는 의견을 갖되, 그 뒤에서 무엇이 실행되는지에 대해서는 의견을 갖지 않는 것이다.

## 결론

Managed Agents의 분리 아키텍처는 메타 하네스로 기능한다.
특정 구현에 대해서는 비의견적이면서, 현재와 미래의 하네스 설계를 위한 안정적인 인터페이스를 제공한다.
이 접근법은 모델 능력이 진화하더라도 장기 실행 에이전트 작업에서 보안과 안정성을 유지할 수 있게 한다.
두뇌와 손의 분리라는 비유는 단순한 수사가 아니라 실제 프로덕션에서 검증된 아키텍처 패턴이다.

## Reference

- [Scaling Managed Agents: Decoupling the brain from the hands](https://www.anthropic.com/engineering/managed-agents/)
- [Introducing Claude Managed Agents](https://claude.com/blog/claude-managed-agents)
- [Managed Agents Overview - Claude Platform Docs](https://platform.claude.com/docs/ko/managed-agents/overview)
