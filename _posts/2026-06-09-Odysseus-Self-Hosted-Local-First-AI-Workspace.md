---
layout: post
title: "Odysseus: 내 하드웨어에서 돌리는 셀프호스팅 AI 워크스페이스"
author: 'Juho'
date: 2026-06-09 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Ollama]
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
3. [핵심 기능](#핵심-기능)
4. [기술 스택](#기술-스택)
5. [작동 방식](#작동-방식)
6. [의미와 시사점](#의미와-시사점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Odysseus는 ChatGPT나 Claude 같은 사용 경험을 자체 하드웨어에서 재현하도록 설계된 셀프호스팅 애플리케이션이다.
로컬 우선(local-first)과 프라이버시 중심을 핵심 원칙으로 삼는다.
외부 서비스에 의존하지 않고 데이터를 완전히 통제하면서 AI 모델과 상호작용하는 것을 목적으로 한다.

## 배경

상용 AI 챗봇은 편리하지만 대화 내용과 데이터가 외부 서비스 제공자의 인프라를 거친다.
Odysseus는 이러한 의존성을 제거하고, 사용자가 자신의 데이터를 직접 소유하고 관리할 수 있는 환경을 제공한다.
로컬 모델 서빙을 중심에 두면서도 필요할 경우 외부 API와도 연결할 수 있도록 설계되어 있다.

## 핵심 기능

Odysseus는 단일 채팅 기능을 넘어 다양한 작업을 하나의 워크스페이스에서 처리한다.

Chat & Agents는 로컬 모델 또는 API 연결을 지원한다.
vLLM, llama.cpp, Ollama, OpenAI 등 여러 백엔드와 연동할 수 있다.

Cookbook은 하드웨어 스캐너를 통해 호환 모델을 추천하고 다운로드한다.

Deep Research는 소스를 수집하고 종합해 리포트로 만드는 다단계 분석 기능이다.

Compare는 모델을 나란히 테스트하고 블라인드 평가를 수행한다.

Documents는 마크다운, HTML, CSV용 AI 보조 텍스트 에디터를 제공한다.

Memory/Skills는 벡터 기반 지속 기억을 제공하며 시간이 지나며 진화한다.

Email은 IMAP/SMTP 통합과 AI 분류(triage)를 지원한다.

Calendar & Tasks는 CalDAV 동기화와 스케줄링을 처리한다.

Mobile-Ready는 반응형 디자인과 PWA를 지원한다.

## 기술 스택

Odysseus의 구성 요소는 다음과 같다.

기술 스택 구성

| 영역 | 사용 기술 |
| --- | --- |
| 백엔드 | Python + FastAPI |
| 프런트엔드 | JavaScript |
| 저장소 | SQLite, 임베딩에 ChromaDB |
| 서비스 | SearXNG(검색), ntfy(알림), Radicale(캘린더) |
| 배포 | Docker Compose, 선택적 GPU 지원(NVIDIA/AMD) |

전체 코드 비율은 Python 45.6%, JavaScript 43.0%, CSS 9.1%로 구성되어 있다.
백엔드는 Python과 FastAPI로 작성되었고 프런트엔드는 JavaScript 기반이다.
데이터 저장에는 SQLite를 사용하며, 임베딩 처리에는 ChromaDB를 활용한다.

## 작동 방식

Odysseus는 Docker, 네이티브 설치(Linux/macOS/Windows), Apple Silicon으로 배포할 수 있다.
로컬 모델 서빙을 관리하고, 에이전트를 통한 도구 실행을 처리한다.
대화 히스토리를 유지하며 외부 서비스 통합도 담당한다.
이 모든 기능은 선택적 인증 뒤에서 처리된다.

## 의미와 시사점

Odysseus는 상용 AI 서비스의 사용 경험을 자체 하드웨어로 옮겨오려는 시도다.
데이터 통제권을 사용자에게 두면서도 로컬 모델과 외부 API를 모두 선택지로 제공한다.
검색, 알림, 캘린더 같은 주변 서비스까지 자체 호스팅 구성요소로 묶어 하나의 통합 워크스페이스를 구성한다.
프라이버시를 중시하거나 외부 의존성을 줄이려는 사용자에게 실용적인 대안이 될 수 있다.

## 결론

Odysseus는 로컬 우선·프라이버시 중심 원칙 아래 채팅, 에이전트, 리서치, 문서, 이메일, 캘린더를 하나로 묶은 셀프호스팅 AI 워크스페이스다.
Docker Compose 기반 배포와 다양한 모델 백엔드 연동을 통해 사용자가 자신의 환경에서 AI를 통제하며 활용할 수 있도록 한다.

## Reference

- [pewdiepie-archdaemon/odysseus](https://github.com/pewdiepie-archdaemon/odysseus/)
