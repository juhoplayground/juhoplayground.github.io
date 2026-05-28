---
layout: post
title: "Cloudflare Skills - Workers와 Agents SDK 개발을 위한 에이전트 스킬 모음"
author: 'Juho'
date: 2026-05-28 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Skill]
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
2. [스킬이란 무엇인가](#스킬이란-무엇인가)
3. [설치와 사용](#설치와-사용)
   - [설치 방법](#설치-방법)
   - [제공되는 명령과 스킬](#제공되는-명령과-스킬)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

`cloudflare/skills`는 Cloudflare 개발자 플랫폼 전반에 걸쳐 빌드를 돕는 에이전트 스킬(Agent Skills) 모음입니다.
저장소 설명에 따르면 "Cloudflare, Workers, Agents SDK, 그리고 더 넓은 Cloudflare Developer Platform 위에서 빌드하기 위한 에이전트 스킬 모음"입니다.
이 스킬들은 Agent Skills 표준을 지원하는 모든 에이전트에서 동작하도록 설계되었습니다.

이 저장소는 Apache-2.0 라이선스로 공개되어 있으며, 스타 1.6천 개, 포크 133개, 커밋 37개 규모로 관리되고 있습니다.
Claude Code뿐 아니라 Cursor 등 여러 에이전트 환경에서 활용할 수 있는 것이 특징입니다.

## 스킬이란 무엇인가

스킬은 대화 맥락에 따라 자동으로 로드되는 컨텍스트 모듈로 동작합니다.
요청이 특정 스킬의 목적과 일치하면, 에이전트가 관련 가이드와 정보를 자동으로 적용합니다.
즉 개발자가 매번 긴 지침을 직접 붙여 넣지 않아도, 작업 맥락에 맞는 전문 지식이 필요한 순간에 주입되는 방식입니다.

이 접근은 에이전트가 Cloudflare 플랫폼의 방대한 API와 베스트 프랙티스를 일관되게 따르도록 돕습니다.
플랫폼별 관례나 배포 절차를 매번 모델의 일반 지식에 의존하지 않고, 검증된 스킬 정의로 안내하는 것입니다.

## 설치와 사용

### 설치 방법

저장소는 여러 설치 방식을 지원합니다.

| 환경 | 설치 방법 |
|------|-----------|
| Claude Code | 플러그인 마켓플레이스에서 plugin marketplace add cloudflare/skills |
| Cursor | Cursor Marketplace 또는 수동 설정 |
| npx skills CLI | npx skills add 명령으로 저장소 URL 추가 |
| 수동 설치 | 스킬 폴더를 에이전트별 디렉터리에 복사 |

수동 설치 시에는 스킬 폴더를 `~/.claude/skills/`, `~/.cursor/skills/` 등 각 에이전트가 인식하는 디렉터리로 복사합니다.

### 제공되는 명령과 스킬

저장소는 사용자가 직접 호출하는 명령(command)과 맥락에 따라 자동 로드되는 스킬(skill)을 함께 제공합니다.

명령 예시는 다음과 같습니다.

| 명령 | 설명 |
|------|------|
| cloudflare:build-agent | Agents SDK로 AI 에이전트 생성 |
| cloudflare:build-mcp | MCP 서버 개발 |

스킬은 플랫폼 전반을 폭넓게 다룹니다.

| 영역 | 다루는 내용 |
|------|-------------|
| 코어 플랫폼 | Workers, Pages, KV, D1, R2 |
| 에이전트 | Agents SDK 및 Durable Objects |
| 코드 실행 | Sandbox SDK |
| 배포 | Wrangler 배포 도구 |
| 성능 | 웹 성능 감사(auditing) |
| 프로토콜 | MCP 서버 개발 |

## 의미와 시사점

이 저장소는 에이전트가 특정 플랫폼 위에서 안정적으로 작업하도록 돕는 "스킬"이라는 패턴이 실제 제품 생태계로 확산되고 있음을 보여줍니다.
Cloudflare는 자사 개발자 플랫폼의 베스트 프랙티스를 스킬 형태로 패키징하여, 에이전트가 Workers나 Agents SDK를 다룰 때 일관된 품질을 유지하도록 했습니다.

특히 Claude Code, Cursor 등 서로 다른 에이전트 환경에서 동일한 스킬을 재사용할 수 있다는 점이 중요합니다.
Agent Skills 표준을 따르면 벤더 종속 없이 동일한 지식 자산을 여러 도구에 적용할 수 있기 때문입니다.

## 결론

`cloudflare/skills`는 Cloudflare Developer Platform 위에서 에이전트가 빌드를 수행할 때 필요한 지식을 자동 로드형 스킬과 사용자 호출형 명령으로 묶어 제공합니다.
Workers, Agents SDK, MCP 서버 개발 등 핵심 영역을 포괄하며, 다양한 설치 경로와 표준 호환성을 통해 여러 에이전트 환경에서 재사용할 수 있습니다.
플랫폼 제공자가 직접 관리하는 스킬 모음은 에이전트 기반 개발의 일관성과 신뢰성을 높이는 실용적인 사례입니다.

## Reference

- [cloudflare/skills (GitHub)](https://github.com/cloudflare/skills/)
