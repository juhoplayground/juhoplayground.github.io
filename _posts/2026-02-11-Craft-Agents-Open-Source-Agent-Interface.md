---
layout: post
title: "Craft Agents - AI 에이전트를 위한 오픈소스 인터페이스"
author: 'Juho'
date: 2026-02-11 01:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
2. [Craft Agents란](#craft-agents란)
3. [핵심 기능](#핵심-기능)
4. [권한 모드 시스템](#권한-모드-시스템)
5. [워크스페이스와 스킬](#워크스페이스와-스킬)
6. [기술적 특징](#기술적-특징)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

현재 AI 에이전트를 사용하려면 터미널에서 작업하거나, 설정 파일을 작성하고, OAuth 인증 과정을 거쳐야 한다.
새로운 도구를 연동할 때마다 별도의 프로젝트가 필요하고, 컨텍스트 전환이 워크플로우를 방해한다.
Craft Agents는 이러한 문제를 해결하기 위해 만들어진 오픈소스 AI 에이전트 인터페이스다.

## Craft Agents란

Craft Agents는 AI 에이전트 작업을 위한 데스크톱 애플리케이션이다.
터미널이나 코드 에디터에 AI를 덧붙인 형태가 아닌, 에이전트가 실제로 작동하는 방식에 맞춰 처음부터 설계된 문서 중심의 인터페이스를 제공한다.
개발자뿐만 아니라 정보를 다루는 모든 사람을 위해 만들어졌다.

### 개발 철학

Craft 팀은 흥미롭게도 Craft Agents를 Craft Agents 자체로 개발한다.
코드 에디터를 사용하지 않고 프롬프트와 에이전트 자율성에 의존하여 모든 기능과 버그 수정을 진행한다.
이는 에이전트 네이티브 워크플로우가 많은 작업에서 우수할 수 있음을 보여준다.

## 핵심 기능

### 제로 설정 통합

설정 파일이나 토큰 관리 없이 자연어로 통합을 요청할 수 있다.
예를 들어 "add Linear as a source"라고 말하면 에이전트가 자동으로 API를 발견하고, 문서를 읽고, 자격 증명을 구성한다.

### 문서 중심 디자인

세션이 채팅 로그가 아닌 문서처럼 작동한다.
대화에 플래그를 지정하고, 워크플로우 상태(Todo → In Progress → Needs Review → Done)를 할당할 수 있다.
받은 편지함 스타일의 인터페이스에서 병렬 작업을 관리할 수 있다.

### 파일 첨부 기능

이미지, PDF, Office 문서를 대화에 직접 드래그 앤 드롭할 수 있다.
문서는 자동으로 변환되어 처리된다.

### 주요 기능 요약

| 기능 | 설명 |
|------|------|
| 제로 설정 통합 | 자연어로 API 연동 요청, 자동 구성 |
| 문서 중심 디자인 | 세션을 문서처럼 관리, 워크플로우 상태 지원 |
| 파일 첨부 | 드래그 앤 드롭으로 이미지, PDF, Office 문서 처리 |
| 다이어그램 렌더링 | beautiful-mermaid 엔진으로 네이티브 다이어그램 시각화 |
| Craft MCP 통합 | 32개 이상의 Craft 문서 도구 접근 |

## 권한 모드 시스템

Craft Agents는 에이전트의 자율성을 제어하는 3단계 권한 시스템을 제공한다.
사용자는 Shift+Tab으로 모드를 전환할 수 있다.

### 권한 모드

| 모드 | 설명 |
|------|------|
| Explore | 읽기 전용, 안전한 탐색용 |
| Ask to Edit | 각 작업 전 승인 필요 |
| Execute | 신뢰할 수 있는 워크플로우를 위한 완전 자율 |

사용자 정의 규칙으로 권한을 더욱 세밀하게 조정할 수 있다.

## 워크스페이스와 스킬

### 워크스페이스

프로젝트와 환경별로 컨텍스트를 분리하여 유지한다.
각 워크스페이스는 독립적인 설정과 소스를 가질 수 있다.

### 스킬

스킬은 재사용 가능한 지시사항으로, 대화에서 언급되면 에이전트가 따른다.
워크스페이스별로 저장되어 특정 작업에 맞는 전문화된 동작을 정의할 수 있다.

### 소스 연결

다양한 소스와 연결할 수 있다.
- MCP 서버
- REST API (Google, Slack, Microsoft)
- 로컬 파일시스템

## 기술적 특징

### 테마 시스템

앱 수준과 워크스페이스 수준에서 계층적 테마를 적용할 수 있다.
모든 설정은 편집 가능한 파일로 관리된다.

### 멀티 파일 Diff

VS Code 스타일의 창에서 모든 파일 변경 사항을 확인할 수 있다.
에이전트가 수행한 변경 사항을 쉽게 검토할 수 있다.

### 네이티브 다이어그램 렌더링

beautiful-mermaid라는 빠른 Mermaid 렌더링 엔진을 포함하고 있다.
외부 의존성 없이 다이어그램을 시각화할 수 있다.

### 라이선스 및 저장소

| 항목 | 내용 |
|------|------|
| 라이선스 | Apache 2.0 |
| GitHub | github.com/lukilabs/craft-agents-oss |
| 다운로드 | agents.craft.do |

오픈소스로 공개되어 있어 자유롭게 수정하고 사용할 수 있다.

## 결론

Craft Agents는 AI 에이전트 작업의 사용자 경험을 근본적으로 개선하려는 시도다.
터미널이나 설정 파일 없이 자연어로 통합을 요청하고, 문서 중심의 인터페이스에서 작업을 관리할 수 있다.
3단계 권한 시스템으로 에이전트의 자율성을 제어하며, 워크스페이스와 스킬로 컨텍스트를 체계적으로 관리할 수 있다.
Apache 2.0 라이선스로 오픈소스 공개되어 있어 누구나 자유롭게 활용하고 커스터마이징할 수 있다.

## Reference

- [Craft Agents - The Open Source Agent Interface](https://agents.craft.do/)
- [Introducing Craft Agents](https://www.craft.do/blog/introducing-craft-agents/)
- [GitHub - lukilabs/craft-agents-oss](https://github.com/lukilabs/craft-agents-oss/)
