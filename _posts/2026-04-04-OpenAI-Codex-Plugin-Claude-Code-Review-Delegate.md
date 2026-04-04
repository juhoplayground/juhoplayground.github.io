---
layout: post
title: "OpenAI, Claude Code용 Codex 플러그인 공개 - 코드 리뷰와 작업 위임"
author: 'Juho'
date: 2026-04-04 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, OpenAI]
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
3. [핵심 내용](#핵심-내용)
   - [주요 슬래시 커맨드](#주요-슬래시-커맨드)
   - [백그라운드 작업 관리](#백그라운드-작업-관리)
   - [설치 및 설정](#설치-및-설정)
   - [설정 커스터마이징](#설정-커스터마이징)
4. [활용 시나리오](#활용-시나리오)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

OpenAI가 Claude Code 내에서 Codex를 직접 호출할 수 있는 플러그인 [codex-plugin-cc](https://github.com/openai/codex-plugin-cc){:target="_blank"}를 Apache-2.0 라이선스로 공개했다.
이 플러그인을 통해 Claude Code 사용자는 기존 워크플로우를 벗어나지 않고도 Codex의 코드 리뷰와 작업 위임 기능을 활용할 수 있다.

## 배경

AI 코딩 에이전트 시장에서 경쟁이 치열해지는 가운데, OpenAI는 흥미로운 전략을 택했다.
자사 플랫폼에 사용자를 가두는 대신, 경쟁사 도구인 Claude Code 안에서 Codex를 사용할 수 있도록 플러그인을 제공한 것이다.
"니 플랫폼 써도 되니까, 일은 우리한테 맡겨"라는 접근으로, 사용자가 어떤 환경에서든 Codex를 활용하게 만드는 전략이다.

## 핵심 내용

### 주요 슬래시 커맨드

플러그인은 슬래시 커맨드 기반으로 작동하며, 크게 리뷰와 작업 위임 두 가지 축으로 구성된다.

| 커맨드 | 설명 |
|--------|------|
| /codex:review | 현재 작업에 대한 읽기 전용 Codex 코드 리뷰 실행 |
| /codex:adversarial-review | 설계 결정에 의도적으로 도전하는 적대적 리뷰 |
| /codex:rescue | Codex에 작업을 위임하여 버그 조사 및 수정 |

`/codex:review`는 미커밋 변경사항을 검토하는 표준 리뷰로, `--base <ref>` 옵션으로 특정 브랜치와 비교할 수 있다.
변경 사항을 수정하지 않는 읽기 전용 모드로 동작한다.

`/codex:adversarial-review`는 구현 방식과 설계의 가정, 트레이드오프, 실패 모드를 적극적으로 검토한다.
인증, 데이터 손실, 안정성 등 특정 위험 영역에 포커스를 지정할 수도 있다.

`/codex:rescue`는 Codex에게 직접 작업을 위임하는 커맨드다.
버그 조사, 수정 시도, 이전 Codex 작업 계속, 소형 모델로 빠른 처리 등이 가능하다.

### 백그라운드 작업 관리

장시간 실행되는 작업을 비동기로 처리할 수 있는 관리 커맨드도 제공된다.

| 커맨드 | 설명 |
|--------|------|
| /codex:status | 실행 중이거나 최근 완료된 Codex 작업 표시 |
| /codex:result | 완료된 작업의 최종 출력 표시 |
| /codex:cancel | 활성 백그라운드 작업 취소 |
| /codex:setup | Codex 설치 및 인증 확인, 리뷰 게이트 관리 |

`--background` 옵션으로 작업을 백그라운드에서 시작하고, `/codex:status`와 `/codex:result`로 진행 상황을 확인하는 흐름이다.
완료된 작업의 결과에는 Codex 세션 ID가 포함되어 `codex resume <session-id>`로 직접 재개할 수도 있다.

### 설치 및 설정

요구사항은 ChatGPT 구독(무료 포함) 또는 OpenAI API 키, 그리고 Node.js 18.18 이상이다.

```bash
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Codex가 설치되어 있지 않은 경우 npm으로 자동 설치할 수 있다.

```bash
npm install -g @openai/codex
```

인증이 필요하면 `codex login` 명령을 실행한다.
사용량은 기존 Codex 사용 한도에 포함된다.

### 설정 커스터마이징

프로젝트 루트의 `.codex/config.toml`에서 기본 모델과 추론 노력도를 설정할 수 있다.

```toml
model = "gpt-5.4-mini"
model_reasoning_effort = "xhigh"
```

## 활용 시나리오

배포 전 리뷰가 필요할 때는 `/codex:review`로 빠르게 검토할 수 있다.
CI 빌드가 실패하는 문제를 조사해야 할 때는 `/codex:rescue investigate why the build is failing in CI`처럼 자연어로 작업을 위임할 수 있다.
장시간 작업이 예상되면 `--background` 옵션으로 시작하고 다른 작업을 계속 진행하면 된다.

## 결론

OpenAI의 Claude Code용 Codex 플러그인은 경쟁사 도구 안에서 자사 서비스를 제공하는 파격적인 전략이다.
코드 리뷰, 적대적 리뷰, 작업 위임, 백그라운드 실행 등 실용적인 기능을 갖추고 있어, Claude Code 사용자에게 추가적인 AI 리뷰 레이어를 제공한다.
Apache-2.0 오픈소스로 공개되어 커뮤니티 기여도 가능하다.

## Reference

- [codex-plugin-cc GitHub Repository](https://github.com/openai/codex-plugin-cc/)
