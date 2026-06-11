---
layout: post
title: "Hermes Agent: 모든 메신저에 붙는 Nous Research의 AI 데스크톱"
author: 'Juho'
date: 2026-06-10 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, MCP]
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
3. [여섯 가지 핵심 역량](#여섯-가지-핵심-역량)
4. [지원 환경과 구독 티어](#지원-환경과-구독-티어)
5. [의미와 시사점](#의미와-시사점)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Hermes는 Nous Research가 공개한 오픈소스 AI 에이전트 데스크톱 애플리케이션이다.
MIT 라이선스로 배포되며 여러 플랫폼에서 동작한다.
다양한 커뮤니케이션 채널과 통합되는 다재다능한 어시스턴트를 지향한다.

## 배경

기존의 AI 어시스턴트는 특정 앱이나 단일 인터페이스 안에 갇혀 있는 경우가 많았다.
Hermes는 사용자가 일상에서 사용하는 여러 메신저와 커뮤니케이션 채널에 직접 붙어 동작하도록 설계되었다.
즉 하나의 에이전트가 여러 표면(surface)에 걸쳐 작동하며, 통합된 기억을 유지한다.

## 여섯 가지 핵심 역량

Hermes는 크게 여섯 가지 핵심 역량으로 정리된다.

### Multi-Platform Integration

Telegram, Discord, Slack, WhatsApp, Signal, Email, CLI 등 점점 늘어나는 플랫폼과 연결된다.
모든 표면(surface)에 걸쳐 통합된 기억을 제공한다.

### Memory System

에이전트가 사용자의 프로젝트를 학습한다.
스킬을 자동으로 생성한다.
문제를 어떻게 풀었는지 절대 잊지 않는다.

### Automation

리포트, 백업, 브리핑을 자연어로 스케줄링할 수 있다.
게이트웨이를 통해 무인으로 실행된다.

### Task Delegation

자체 대화, 터미널, Python RPC 스크립트를 가진 격리된 서브에이전트를 사용한다.
이를 통해 작업을 위임할 수 있다.

### Web Capabilities

웹 검색과 브라우저 자동화를 지원한다.
비전 분석, 이미지 생성, 텍스트-음성 변환 기능을 제공한다.

### Sandboxing

다섯 가지 백엔드 환경에서 동작한다.
local, Docker, SSH, Singularity, Modal을 지원한다.
컨테이너 하드닝과 네임스페이스 격리를 적용한다.

## 지원 환경과 구독 티어

Hermes는 주요 데스크톱 운영체제를 모두 지원하며, 무료 및 유료 구독 티어를 제공한다.
현재 버전은 v0.16.0이다.

지원 환경 및 티어 요약

| 항목 | 내용 |
| --- | --- |
| 지원 OS | macOS 12+, Windows 10/11, Linux |
| 현재 버전 | v0.16.0 |
| 구독 티어 | Free, Plus, Super, Ultra |
| 유료 플랜 혜택 | 월 크레딧, 300개 이상 모델(도구 사용 통합) 접근 |

유료 플랜은 월 크레딧을 포함한다.
또한 도구 사용이 통합된 300개 이상의 모델에 접근할 수 있다.

## 의미와 시사점

Hermes는 AI 에이전트를 단일 앱이 아니라 사용자가 이미 쓰고 있는 여러 메신저 위에서 동작시킨다는 점에서 차별화된다.
통합 기억과 자동 스킬 생성은 반복 작업에서 누적된 맥락을 잃지 않게 한다.
서브에이전트를 통한 작업 위임과 다섯 가지 샌드박스 백엔드는 안전한 격리 실행을 가능하게 한다.
MIT 라이선스 오픈소스라는 점은 개발자가 직접 확장하고 검증할 수 있는 여지를 넓힌다.

## 결론

Hermes는 여러 플랫폼 통합, 기억 시스템, 자동화, 작업 위임, 웹 역량, 샌드박싱이라는 여섯 가지 역량을 갖춘 오픈소스 AI 에이전트 데스크톱이다.
macOS, Windows, Linux를 지원하며 Free부터 Ultra까지의 구독 티어를 제공한다.
하나의 에이전트가 여러 커뮤니케이션 채널에 걸쳐 통합적으로 동작한다는 점이 핵심이다.

## Reference

- [Hermes Agent Desktop](https://hermes-agent.nousresearch.com/desktop/)
