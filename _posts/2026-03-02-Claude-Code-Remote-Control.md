---
layout: post
title: "Claude Code Remote Control - 로컬 세션을 어디서든 이어받기"
author: 'Juho'
date: 2026-03-02 00:00:00 +0900
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
2. [요구 사항](#요구-사항)
3. [Remote Control 세션 시작](#remote-control-세션-시작)
   - [새 세션 시작](#새-세션-시작)
   - [기존 세션에서 전환](#기존-세션에서-전환)
   - [다른 기기에서 연결](#다른-기기에서-연결)
4. [연결 및 보안](#연결-및-보안)
5. [Remote Control vs 웹의 Claude Code](#remote-control-vs-웹의-claude-code)
6. [제한 사항](#제한-사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Anthropic이 Claude Code에 원격 제어(Remote Control) 기능을 추가했습니다.
이 기능을 사용하면 로컬 컴퓨터에서 실행 중인 Claude Code 세션을 스마트폰, 태블릿, 다른 컴퓨터의 브라우저에서 이어받아 계속 작업할 수 있습니다.
현재 Pro 및 Max 플랜에서 연구 미리보기(research preview)로 제공되며, Team 또는 Enterprise 플랜에서는 아직 지원되지 않습니다.

Remote Control의 핵심은 코드와 파일 시스템이 클라우드로 올라가지 않는다는 점입니다.
Claude는 컴퓨터에서 로컬로 실행되며, 웹/모바일 인터페이스는 단지 해당 로컬 세션으로의 창(window) 역할을 합니다.

## 요구 사항

Remote Control을 사용하기 전에 다음 조건을 충족해야 합니다.

| 항목 | 설명 |
|------|------|
| 구독 플랜 | Pro 또는 Max 플랜 필요 (API 키 미지원) |
| 인증 | claude.ai를 통해 로그인 (/login 명령 사용) |
| 작업 공간 신뢰 | 프로젝트 디렉토리에서 최소 한 번 claude 실행 후 작업 공간 신뢰 대화 수락 필요 |

## Remote Control 세션 시작

### 새 세션 시작

프로젝트 디렉토리에서 다음 명령을 실행합니다.

```bash
claude remote-control
```

실행하면 터미널에서 세션이 유지되며 원격 연결을 기다리는 상태가 됩니다.
터미널에는 세션 URL이 표시되고, 스페이스바를 눌러 QR 코드를 표시할 수 있습니다.

이 명령은 다음 플래그를 지원합니다.

| 플래그 | 설명 |
|--------|------|
| --verbose | 자세한 연결 및 세션 로그 표시 |
| --sandbox | 세션 중 파일 시스템 및 네트워크 격리 활성화 |
| --no-sandbox | 샌드박싱 명시적 비활성화 (기본값) |

### 기존 세션에서 전환

이미 Claude Code 세션이 실행 중이라면 `/remote-control` (또는 `/rc`) 명령을 사용합니다.

```
/remote-control
```

현재 대화 기록을 이어받아 Remote Control 세션을 시작합니다.
세션 URL과 QR 코드가 표시됩니다.

연결 전에 `/rename` 명령으로 세션에 설명적인 이름을 지정하면 기기 목록에서 찾기 쉬워집니다.

### 다른 기기에서 연결

다른 기기에서 연결하는 방법은 세 가지입니다.

| 방법 | 설명 |
|------|------|
| 세션 URL 열기 | 터미널에 표시된 URL을 모든 브라우저에서 열기 |
| QR 코드 스캔 | 표시된 QR 코드를 Claude 앱에서 스캔 |
| 세션 목록에서 찾기 | claude.ai/code 또는 Claude 앱에서 이름으로 세션 검색 |

모든 세션에 대해 Remote Control을 자동으로 활성화하려면 `/config`에서 **Enable Remote Control for all sessions**를 `true`로 설정합니다.

## 연결 및 보안

Remote Control의 보안 구조는 다음과 같습니다.
로컬 Claude Code 세션은 아웃바운드 HTTPS 요청만 수행하며 컴퓨터에서 인바운드 포트를 열지 않습니다.
Remote Control을 시작하면 Anthropic API에 등록하고 작업을 폴링하는 방식으로 동작합니다.

모든 트래픽은 TLS를 통해 Anthropic API를 경유하며, 이는 모든 Claude Code 세션에서 동일하게 적용되는 전송 보안 수준입니다.
각 연결은 단일 목적으로 범위가 지정되고 독립적으로 만료되는 단기 자격 증명을 사용합니다.

## Remote Control vs 웹의 Claude Code

두 기능 모두 claude.ai/code 인터페이스를 사용하지만, 세션이 실행되는 위치가 다릅니다.

| 항목 | Remote Control | 웹의 Claude Code |
|------|------|------|
| 실행 위치 | 로컬 컴퓨터 | Anthropic 클라우드 인프라 |
| 로컬 파일 시스템 | 접근 가능 | 접근 불가 |
| MCP 서버 | 로컬 MCP 서버 사용 가능 | 미지원 |
| 프로젝트 설정 | 유지됨 | 별도 설정 필요 |
| 적합한 사용 사례 | 작업 중 다른 기기에서 이어받기 | 로컬 설정 없이 새로 시작 |

## 제한 사항

Remote Control 사용 시 주의해야 할 제한 사항입니다.

- **한 번에 하나의 원격 세션**: 각 Claude Code 세션은 하나의 원격 연결만 지원합니다.
- **터미널은 열려 있어야 함**: 터미널을 닫거나 claude 프로세스를 중지하면 세션이 종료됩니다.
- **장시간 네트워크 중단**: 약 10분 이상 네트워크에 도달할 수 없으면 세션이 타임아웃됩니다.

## 결론

Claude Code의 Remote Control 기능은 로컬 개발 환경의 편의성과 멀티 디바이스 접근성을 결합한 기능입니다.
코드와 파일 시스템이 클라우드로 올라가지 않으면서도 어디서든 작업을 이어받을 수 있어, 자리를 이동하거나 화면을 전환해야 하는 상황에서 유용합니다.
현재 Pro/Max 플랜의 연구 미리보기 단계이지만, 향후 더 넓은 플랜으로 확장될 가능성이 있습니다.

## Reference

- [모든 기기에서 로컬 세션 계속하기 (Remote Control)](https://code.claude.com/docs/ko/remote-control)
