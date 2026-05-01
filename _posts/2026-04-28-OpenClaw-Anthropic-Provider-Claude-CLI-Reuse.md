---
layout: post
title: "OpenClaw Anthropic Provider 복귀: Claude CLI 재사용과 API 키 설정 완벽 가이드"
author: 'Juho'
date: 2026-04-28 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, OpenAI, Caching]
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
2. [두 가지 접근 방식](#두-가지-접근-방식)
   - [API 키 기반 설정](#api-키-기반-설정)
   - [Claude CLI 백엔드](#claude-cli-백엔드)
3. [핵심 지원 기능](#핵심-지원-기능)
   - [Thinking 모드와 Fast Mode](#thinking-모드와-fast-mode)
   - [프롬프트 캐싱과 1M 컨텍스트](#프롬프트-캐싱과-1m-컨텍스트)
4. [차별점과 문제 해결](#차별점과-문제-해결)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

OpenClaw가 Anthropic Provider를 다시 사용할 수 있게 되었다.
Anthropic 담당자가 OpenClaw 스타일의 Claude CLI 사용이 다시 허용된다고 통보했으며, API 키와 Claude CLI 재사용을 함께 지원하고 기존 Anthropic 토큰도 계속 인정한다.
OpenClaw는 Claude 모델 제품군에 대해 Anthropic API 키를 통한 표준 API 접근과 Claude CLI 재사용이라는 두 가지 접근 방식을 지원한다.
Adaptive Thinking, Fast Mode, Prompt Caching, 1M 컨텍스트 등 Claude 4.6 주요 기능을 활용할 수 있도록 설정 경로를 제공한다.

## 두 가지 접근 방식

### API 키 기반 설정

Anthropic API 키 사용이 가장 명확한 운영 경로다.
CLI 온보딩을 통해 대화형으로 설정하거나 환경변수로 비대화형 구성이 가능하다.

```bash
openclaw onboard
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

설정 파일은 JSON5 형식으로 작성한다.

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

API 키 방식은 사용량 기반 청구로 비용 추적이 명확하고, 프로덕션 환경에서 안정성이 높다.

### Claude CLI 백엔드

Claude CLI 백엔드는 레거시 토큰 인증을 지원하지만 API 키 방식이 더 안정적이라고 문서는 권장한다.
자세한 설정은 gateway cli-backends 경로를 참조하라고 안내한다.
이 방식은 기존 Claude 구독자가 별도 API 결제 없이 도구를 사용할 수 있도록 하는 경로지만, 프롬프트 캐싱, Fast Mode, 1M 컨텍스트 같은 고급 기능은 미지원이다.

## 핵심 지원 기능

### Thinking 모드와 Fast Mode

Claude 4.6 모델은 기본적으로 적응형 사고가 활성화된다.
명시적으로 레벨을 지정하거나 adaptive로 설정할 수 있다.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { thinking: "adaptive" }
        },
      },
    },
  },
}
```

메시지별 오버라이드는 `/think:<level>` 명령으로 가능하다.
Fast Mode는 `/fast` 토글로 Anthropic API의 우선순위 처리를 활성화한다.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-6": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

`/fast on`은 service tier를 auto로 설정해 우선 처리하고, `/fast off`는 standard_only로 전환한다.
프록시를 통한 라우팅 시에는 서비스 티어가 적용되지 않는다는 제한이 있다.

### 프롬프트 캐싱과 1M 컨텍스트

프롬프트 캐싱은 응답 시간 단축과 비용 최적화를 위한 핵심 기능이다.

| 값 | 기간 | 설명 |
|---|------|------|
| none | 비활성화 | 캐싱 사용 안 함 |
| short | 5분 | API 키 기본값 |
| long | 1시간 | 장시간 캐싱 |

기본 설정은 다음과 같이 구성한다.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

에이전트별로 캐싱 정책을 오버라이드할 수도 있다.

```json5
{
  agents: {
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } },
    ],
  },
}
```

1M 컨텍스트 윈도우는 베타 기능으로 명시적 활성화가 필요하다.

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

레거시 토큰 인증에서는 작동하지 않으므로 1M 컨텍스트를 사용하려면 API 키 방식이 필수다.

## 차별점과 문제 해결

두 방식의 기능 차이는 실무 선택에 직접 영향을 준다.

| 항목 | API 키 | CLI 재사용 |
|------|--------|----------|
| 청구 방식 | 사용량 기반 | Claude 구독 |
| 프로덕션 안정성 | 높음 | 중간 |
| 프롬프트 캐싱 | 지원 | 미지원 |
| Fast Mode | 지원 | 미지원 |
| 1M 컨텍스트 | 지원 | 미지원 |

자주 발생하는 문제와 해결 방법도 정리되어 있다.

| 에러 | 원인 | 해결 |
|------|------|------|
| 401 에러 | 토큰 만료/취소 | API 키로 마이그레이션 |
| No API key found | 에이전트별 독립 인증 | 각 에이전트 온보딩 |
| No available auth profile | 속도 제한 쿨다운 | 다른 프로필 추가 또는 대기 |

상태 확인은 `openclaw models status` 또는 `openclaw models status --json`으로 가능하다.
Hacker News 커뮤니티에서는 Anthropic의 잦은 정책 변경과 공지 불일치로 인한 신뢰 문제, Codex 같은 대안으로의 전환 고려, 구독 모델의 경제성 문제가 주요 우려로 제기되었다.

## 결론

OpenClaw의 Anthropic Provider 재개방은 Claude 기반 에이전트 도구를 사용하는 개발자에게 중요한 변화다.
API 키 방식은 Adaptive Thinking, Fast Mode, 프롬프트 캐싱, 1M 컨텍스트 같은 고급 기능을 모두 활용할 수 있는 프로덕션 경로이며, CLI 재사용은 기존 구독자에게 편의를 제공하되 기능 제약이 있다.
에이전트별 설정 오버라이드로 캐싱 전략을 세분화할 수 있어 비용과 응답 속도를 함께 최적화할 수 있다.
다만 Anthropic 정책이 일관되지 않다는 커뮤니티 우려는 여전하며, 프로덕션 도입 시 인증 방식과 청구 체계의 변화를 예의주시할 필요가 있다.

## Reference

- [OpenClaw Anthropic Provider Documentation](https://docs.openclaw.ai/providers/anthropic)
