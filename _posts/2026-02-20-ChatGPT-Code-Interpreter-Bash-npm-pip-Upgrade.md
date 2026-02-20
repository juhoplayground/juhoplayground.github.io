---
layout: post
title: "ChatGPT가 진짜 개발 환경이 됐다, Bash·npm·pip 설치까지 지원"
author: 'Juho'
date: 2026-02-20 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, OpenAI]
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
2. [주요 변화 4가지](#주요-변화-4가지)
3. [보안 및 제한사항](#보안-및-제한사항)
4. [실무 활용 예시](#실무-활용-예시)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

OpenAI가 ChatGPT의 Code Interpreter를 대폭 업그레이드했다.
공식 발표 없이 조용히 배포되었으며, 개발자 Simon Willison이 이를 발견하고 분석했다.
이전까지 Python 코드만 실행 가능했던 ChatGPT가 이제 Bash 명령 실행, 다중 언어 지원, 패키지 설치까지 가능한 본격적인 개발 환경으로 진화했다.

## 주요 변화 4가지

### Bash 명령 직접 실행

이전에는 Python 코드만 실행할 수 있었다.
이제 터미널 명령을 직접 실행할 수 있게 되었다.
셸 스크립트 작성과 시스템 명령어 활용이 가능해진 것이다.

### 10개 이상 프로그래밍 언어 지원

Python만 지원하던 것에서 대폭 확대되었다.
현재 지원되는 언어는 다음과 같다.

- Node.js
- Ruby
- Perl
- PHP
- Go
- Java
- Swift
- Kotlin
- C
- C++

다양한 언어로 코드를 작성하고 즉시 실행 결과를 확인할 수 있다.

### npm과 pip 패키지 설치

OpenAI의 커스텀 프록시 메커니즘을 통해 패키지 설치가 가능해졌다.
npm과 pip를 통해 외부 라이브러리를 설치하고 활용할 수 있다.
이를 통해 실제 프로젝트에서 사용하는 라이브러리를 ChatGPT 환경에서도 테스트할 수 있게 되었다.

### container.download 기능

ChatGPT가 웹에서 URL을 찾으면, 그 파일을 컨테이너로 직접 다운로드할 수 있다.
외부 데이터를 가져와서 분석하거나 처리하는 워크플로우가 가능해진 것이다.

## 보안 및 제한사항

ChatGPT의 개발 환경에는 다음과 같은 보안 장치가 적용되어 있다.

- 컨테이너는 외부 네트워크에 직접 접근할 수 없다.
- 20분 미사용 시 컨테이너가 만료된다.
- URL 접근은 사용자 입력 또는 검색 결과에만 제한된다.
- 프롬프트 인젝션 공격으로부터 보호된다.

## 실무 활용 예시

이번 업그레이드로 다양한 실무 시나리오가 가능해졌다.

- CSV 파일을 pandas로 분석하고 시각화
- npm 패키지를 설치하여 ASCII 아트 생성
- 웹에서 최신 데이터를 다운로드한 후 비교 분석

무료 사용자도 이 기능을 완전히 사용할 수 있다는 점이 주목할 만하다.

## 결론

ChatGPT의 Code Interpreter 업그레이드는 단순한 코드 실행 도구에서 본격적인 개발 환경으로의 전환을 의미한다.
Bash 명령 실행, 다중 언어 지원, 패키지 설치, 파일 다운로드 기능이 추가되면서 활용 범위가 크게 넓어졌다.
다만 OpenAI가 공식 문서를 업데이트하지 않은 점은 아쉬운 부분이다.

## Reference

- [ChatGPT Containers can now run bash, pip/npm install packages, and download files - Simon Willison](https://simonwillison.net/2026/Jan/26/chatgpt-containers/)
