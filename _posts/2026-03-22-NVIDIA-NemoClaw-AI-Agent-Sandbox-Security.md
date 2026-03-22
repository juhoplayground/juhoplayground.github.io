---
layout: post
title: "NVIDIA NemoClaw - AI 에이전트를 위한 샌드박스 보안 스택"
author: 'Juho'
date: 2026-03-23 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Security, Docker]
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
2. [핵심 기능](#핵심-기능)
3. [보안 아키텍처](#보안-아키텍처)
   - [보안 계층](#보안-계층)
4. [구성요소](#구성요소)
5. [사용법](#사용법)
   - [시스템 요구사항](#시스템-요구사항)
   - [설치](#설치)
   - [기본 명령어](#기본-명령어)
6. [제약사항](#제약사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

NVIDIA NemoClaw는 OpenClaw 자동화 어시스턴트를 안전하게 실행하기 위한 오픈소스 스택이다.
NVIDIA OpenShell 런타임을 설치하고, 모든 네트워크 요청과 파일 접근을 선언적 정책으로 제어하는 샌드박스 환경을 제공한다.
AI 에이전트가 시스템 리소스에 접근할 때 발생할 수 있는 보안 위험을 다층 보안 구조로 차단하는 것이 핵심 목적이다.

## 핵심 기능

NemoClaw가 제공하는 주요 기능은 다음과 같다.

첫째, 샌드박스 환경 구성이다.
Landlock, seccomp, 네트워크 네임스페이스를 활용한 다층 보안을 제공한다.

둘째, 정책 기반 제어이다.
선언적 정책으로 모든 네트워크 요청, 파일 접근, 추론 호출을 관리할 수 있다.

셋째, 클라우드 추론이다.
NVIDIA 클라우드 모델(Nemotron-3-super-120b)로의 투명 라우팅을 지원한다.

넷째, 실시간 정책 수정이다.
네트워크 및 추론 정책을 런타임에 변경할 수 있다.

## 보안 아키텍처

NemoClaw는 네 가지 보안 계층을 통해 AI 에이전트의 동작을 제어한다.

### 보안 계층

| 계층 | 보호 대상 | 적용 시점 |
|------|-----------|-----------|
| 네트워크 | 무단 아웃바운드 차단 | 런타임(재로드 가능) |
| 파일시스템 | /sandbox, /tmp 외 접근 차단 | 샌드박스 생성 시 잠금 |
| 프로세스 | 권한 상승, 위험한 시스콜 차단 | 샌드박스 생성 시 잠금 |
| 추론 | 모델 API 호출 제어 | 런타임(재로드 가능) |

네트워크 계층과 추론 계층은 런타임에 정책을 재로드할 수 있다.
파일시스템 계층과 프로세스 계층은 샌드박스 생성 시 잠금되어 이후 변경이 불가능하다.

## 구성요소

NemoClaw의 아키텍처는 네 가지 주요 구성요소로 이루어져 있다.

플러그인은 TypeScript CLI로 구현되어 있으며, launch, connect, status, logs 명령을 제공한다.

블루프린트는 Python 버전 아티팩트로 샌드박스 생성 및 정책 적용을 담당한다.

샌드박스는 OpenShell 컨테이너 내에서 OpenClaw 실행 환경을 구성한다.

추론 계층은 NVIDIA 클라우드를 통한 모델 API 라우팅을 처리한다.

## 사용법

### 시스템 요구사항

NemoClaw를 실행하기 위해서는 다음 환경이 필요하다.

- Ubuntu 22.04 LTS 이상
- Node.js 20+
- npm 10+
- Docker
- NVIDIA OpenShell

### 설치

아래 명령어로 NemoClaw를 설치할 수 있다.

```bash
curl -fsSL https://nvidia.com/nemoclaw.sh | bash
```

### 기본 명령어

설치 후 사용 가능한 주요 명령어는 다음과 같다.

```bash
nemoclaw onboard        # 대화형 설정 마법사
nemoclaw <name> connect # 샌드박스 내부 셸 접속
openshell term          # OpenShell TUI 모니터링
openclaw tui            # 에이전트와 채팅
```

## 제약사항

NemoClaw는 현재 알파 소프트웨어 상태이다.
프로덕션 환경에서의 사용은 지원되지 않는다.
라이선스는 Apache 2.0이다.

## 결론

NVIDIA NemoClaw는 AI 에이전트 실행 시 발생할 수 있는 보안 위협을 다층 샌드박스 구조로 해결하는 오픈소스 스택이다.
Landlock, seccomp, 네트워크 네임스페이스를 활용한 보안 계층과 선언적 정책 기반 제어를 통해 에이전트의 네트워크, 파일시스템, 프로세스, 추론 접근을 체계적으로 관리할 수 있다.
현재 알파 단계이지만, AI 에이전트 보안 분야에서 주목할 만한 프로젝트이다.

## Reference

- [NemoClaw GitHub](https://github.com/NVIDIA/NemoClaw/)
