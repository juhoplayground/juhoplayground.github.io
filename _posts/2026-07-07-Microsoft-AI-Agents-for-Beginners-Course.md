---
layout: post
title: "Microsoft AI Agents for Beginners: AI 에이전트 입문 12강 무료 코스"
author: 'Juho'
date: 2026-07-07 00:00:00 +0900
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
2. [커리큘럼 구성](#커리큘럼-구성)
   - [핵심 기초](#핵심-기초)
   - [디자인 패턴과 기법](#디자인-패턴과-기법)
   - [고급 주제](#고급-주제)
3. [사용 기술과 사전 요건](#사용-기술과-사전-요건)
4. [학습 방식](#학습-방식)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Microsoft의 AI Agents for Beginners는 AI 에이전트를 구축하기 위한 기초 지식을 가르치는 무료 교육 프로그램이다.
"12 Lessons to Get Started Building AI Agents"라는 부제 그대로, 에이전트 개발을 시작하는 데 필요한 내용을 구조화된 강의로 제공한다.

각 레슨은 서면 콘텐츠, 코드 예제, 동영상 강의를 결합한 형태다.
기초부터 고급 주제까지 총 18개 모듈로 구성되어 있다.

## 커리큘럼 구성

### 핵심 기초

첫 세 개 레슨은 AI 에이전트의 개념과 뼈대를 다룬다.

| 레슨 | 주제 |
|------|------|
| 1 | AI 에이전트 소개와 활용 사례 |
| 2 | 에이전트 프레임워크 탐색 |
| 3 | 디자인 패턴 이해 |

### 디자인 패턴과 기법

레슨 4부터 9까지는 실제 에이전트를 만들 때 쓰는 패턴과 기법을 다룬다.

| 레슨 | 주제 |
|------|------|
| 4 | 도구 사용(Tool Use) 패턴 |
| 5 | Agentic RAG (검색 증강 생성) |
| 6 | 신뢰할 수 있는 에이전트 구축 |
| 7 | 계획 수립(Planning) 접근법 |
| 8 | 멀티 에이전트 시스템 |
| 9 | 메타인지(Metacognition) 패턴 |

### 고급 주제

레슨 10부터 15까지는 프로덕션과 프로토콜 등 실전 배포로 넘어간다.

| 레슨 | 주제 |
|------|------|
| 10 | 프로덕션 배포 |
| 11 | 에이전트 프로토콜 (MCP, A2A, NLWeb) |
| 12 | 컨텍스트 엔지니어링 |
| 13 | 에이전트 메모리 관리 |
| 14 | Microsoft Agent Framework 탐색 |
| 15 | 컴퓨터 사용(Computer Use) 에이전트, 브라우저 자동화 |

이후 특화 주제로 프로덕션 확장성, 로컬 AI 에이전트 개발, 보안 고려사항을 추가로 다룬다.

## 사용 기술과 사전 요건

코스는 다음 기술을 중심으로 진행된다.

| 기술 | 설명 |
|------|------|
| Microsoft Agent Framework (MAF) | 코스의 중심이 되는 에이전트 프레임워크 |
| Microsoft Foundry Agent Service V2 | 에이전트 서비스 실행 환경 |
| OpenAI 호환 제공자 | MiniMax 등 대안 제공자 지원 |

사전 요건은 다음과 같다.
Microsoft Foundry 접근을 위한 Azure 계정, Python 환경이 필요하다.
생성형 AI에 대한 기본 지식이 있으면 좋으며, 부족한 경우 보충용 코스가 별도로 제공된다.

## 학습 방식

각 레슨은 여러 형태의 자료를 함께 제공한다.
서면 자료로 개념을 설명하고, Python 코드 샘플로 직접 구현해볼 수 있게 하며, 동영상 강의와 추가 참고 링크가 붙는다.

기초 개념 설명에서 시작해 도구 사용, RAG, 멀티 에이전트, 프로토콜, 컴퓨터 사용까지 점진적으로 난이도를 올리는 구조라 입문자가 순서대로 따라가기 좋다.

## 결론

Microsoft AI Agents for Beginners는 에이전트 개발의 전체 지형을 처음부터 훑어보고 싶은 사람에게 적합한 무료 코스다.
개념, 디자인 패턴, 프로토콜, 프로덕션 배포까지 12강에 걸쳐 폭넓게 다룬다.

Microsoft Agent Framework와 Foundry를 중심으로 하지만, OpenAI 호환 제공자도 지원해 특정 생태계에 완전히 종속되지 않고 학습할 수 있다.

## Reference

- [Microsoft AI Agents for Beginners GitHub](https://github.com/microsoft/ai-agents-for-beginners)
