---
layout: post
title: "NanoClaw - Docker 컨테이너 격리를 통한 안전한 AI 에이전트 실행 오픈소스 아키텍처"
author: 'Juho'
date: 2026-03-17 00:00:00 +0900
categories: [Dev]
tags: [Agent, AI, Docker, Dev]
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
2. [핵심 아키텍처 - Host와 Container 분리](#핵심-아키텍처---host와-container-분리)
   - [설계 철학](#설계-철학)
   - [메시지 흐름](#메시지-흐름)
3. [통신 프로토콜 - stdin/stdout 파이핑](#통신-프로토콜---stdinstdout-파이핑)
4. [6단계 보안 레이어](#6단계-보안-레이어)
5. [Skills 시스템](#skills-시스템)
   - [스킬 구분](#스킬-구분)
   - [스킬 엔진의 진화](#스킬-엔진의-진화)
6. [컨테이너 기반 설계 선택의 이유](#컨테이너-기반-설계-선택의-이유)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

AI 에이전트에게 Bash 접근 권한을 부여하는 것은 강력하지만, 호스트 시스템의 보안 위협이 될 수 있다.
NanoClaw는 Docker 컨테이너 격리를 통해 AI 에이전트가 안전하게 웹 검색, 코드 실행, git 작업 등을 수행할 수 있도록 하는 오픈소스 아키텍처이다.
Host(메시지 라우팅)와 Container(AI 실행)를 완전히 분리하여 Claude Agent SDK가 호스트 시스템을 보호하면서도 자유롭게 작업할 수 있는 환경을 제공한다.

## 핵심 아키텍처 - Host와 Container 분리

### 설계 철학

NanoClaw의 핵심 원칙은 "Host는 배달부, Container는 두뇌"이다.
이 원칙에 따라 두 영역의 역할이 명확히 구분된다.

| 영역 | 역할 | 담당 기술 |
|------|------|-----------|
| Host | Telegram 연동, 라우팅, SQLite, 스케줄링 | Node.js (grammy) |
| Container | 모든 AI 작업을 Docker 내에서 격리 실행 | Claude Agent SDK |

Host에서는 AI 처리를 일절 수행하지 않는다.
모든 AI 관련 작업은 반드시 Container 내부에서만 이루어진다.

### 메시지 흐름

전체 메시지 처리 과정은 다음과 같다.

1. 사용자가 Telegram으로 메시지를 전송한다.
2. grammy Long Polling을 통해 Host(Node.js)가 메시지를 수신한다.
3. Host가 Docker 컨테이너를 spawn하고, stdin으로 프롬프트와 OAuth 토큰을 전달한다.
4. Container 내부에서 Claude Agent SDK가 작업을 실행한다.
5. 실행 결과가 stdout JSON으로 반환된다.
6. Host가 결과를 Telegram으로 전달한다.
7. Container가 종료된다.

이 방식은 각 요청마다 컨테이너가 생성되고 종료되므로, 세션 간 상태가 유지되지 않는다.

## 통신 프로토콜 - stdin/stdout 파이핑

Host와 Container 간 통신은 stdin/stdout 파이핑 방식을 사용한다.
이 설계에는 다음과 같은 이점이 있다.

- 개방된 포트나 네트워크 노출이 없다.
- 시크릿(OAuth 토큰 등)은 환경 변수나 파일이 아닌 stdin을 통해서만 전달된다.
- child_process.spawn()을 통한 커널 수준 IPC를 사용한다.
- 공인 IP 없이 NAT 뒤에서도 정상 작동한다.

네트워크를 사용하지 않기 때문에 외부 공격 표면이 최소화된다.

## 6단계 보안 레이어

NanoClaw는 6가지 보안 레이어를 통해 AI 에이전트의 실행 환경을 보호한다.

| 레이어 | 보안 항목 | 설명 |
|--------|-----------|------|
| 1 | 파일시스템 분리 | 그룹별 전용 Docker 볼륨 마운트 |
| 2 | 시크릿 처리 | OAuth 토큰은 stdin으로만 전달 |
| 3 | 마운트 허용 목록 | JSON 설정으로 마운트 가능 경로를 제한 |
| 4 | 읽기 전용 코어 | 프로젝트 루트는 읽기 전용으로 마운트 |
| 5 | 그룹 IPC 격리 | 그룹별 별도 디렉토리 사용 |
| 6 | 세션 데이터 분리 | 실행 간 영구 상태를 유지하지 않음 |

각 레이어가 독립적으로 작동하므로, 하나의 레이어가 우회되더라도 나머지 레이어가 보안을 유지한다.

## Skills 시스템

NanoClaw는 "기능보다 스킬(Skills over features)"이라는 철학을 따른다.
코어는 약 500줄 수준으로 최소화하고, 두 가지 병렬 스킬 컨텍스트를 통해 확장한다.

### 스킬 구분

| 구분 | 스킬 유형 | 예시 |
|------|-----------|------|
| Developer Skills | Host에서 실행 | /setup, /add-telegram, /update-nanoclaw |
| Agent Skills | Container에서 실행 | agent-browser, add-image-vision, add-pdf-reader |

Developer Skills는 시스템 설정과 관리에 사용되며, Agent Skills는 AI 에이전트가 수행하는 실제 작업 기능이다.

### 스킬 엔진의 진화

초기 스킬 엔진은 6,300줄 규모였으나, 현재는 마크다운 스펙과 git merge 방식으로 대체되었다.
각 채널이나 기능은 별도의 fork 리포지토리로 관리된다.
AI 에이전트가 merge 충돌을 자동으로 해결하는 방식으로 운영된다.

## 컨테이너 기반 설계 선택의 이유

AI 에이전트 실행 환경으로 여러 가지 선택지가 있다.
NanoClaw가 Docker 컨테이너 기반을 선택한 이유는 다음과 같다.

| 방식 | 시작 시간 | 볼륨 유연성 | 리소스 관리 |
|------|-----------|-------------|-------------|
| In-process | 즉시 | 없음 | 어려움 |
| Container | 수 초 | 높음 | 용이 |
| VM | 수 분 | 높음 | 과도 |

컨테이너 방식은 시작 시간(수 초), 볼륨 유연성, 리소스 관리 측면에서 균형 잡힌 선택이다.
In-process 방식은 빠르지만 격리가 불가능하고, VM은 격리는 완벽하지만 리소스가 과도하다.

## 결론

NanoClaw는 개인 디바이스 우선 설계와 코드 미니멀리즘을 지향하는 오픈소스 프로젝트이다.
Host-Container 분리, stdin/stdout 파이핑, 6단계 보안 레이어를 통해 AI 에이전트에게 안전한 실행 환경을 제공한다.
약 500줄의 코어와 스킬 기반 확장 구조는 유지보수성과 확장성을 모두 확보한다.
스펙이 AI 구현을 가이드하는 철학은 AI 에이전트 시대의 새로운 개발 패러다임을 보여준다.

## Reference

- [안전한 AI 에이전트는 어떻게 만드는가 — NanoClaw 오픈소스 컨테이너 아키텍처](https://blog.neocode24.com/blog/nanoclaw-architecture/)
