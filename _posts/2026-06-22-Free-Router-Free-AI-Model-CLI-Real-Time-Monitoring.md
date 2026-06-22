---
layout: post
title: "Free-Router: 무료 AI 모델을 실시간 모니터링하고 전환하는 CLI 도구"
author: 'Juho'
date: 2026-06-22 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Dev]
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
2. [어떤 문제를 해결하는가](#어떤-문제를-해결하는가)
3. [주요 기능](#주요-기능)
   - [실시간 TUI 모니터링](#실시간-tui-모니터링)
   - [Tier와 Verdict 척도](#tier와-verdict-척도)
   - [대상 도구 통합](#대상-도구-통합)
4. [설치와 사용법](#설치와-사용법)
   - [API 키와 설정 파일](#api-키와-설정-파일)
   - [최고 모델 자동 선택](#최고-모델-자동-선택)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Free-Router는 "free AI model router, API for everyday AI work"를 표방하는 CLI 도구로, 무료 AI 모델을 발견하고 비교하며 쉽게 전환할 수 있게 해준다.
여러 무료 제공자가 제공하는 모델의 성능을 실시간으로 모니터링하고, 선택한 모델을 개발 도구에 자동 구성하는 것이 핵심이다.
현재 버전은 1.2.1이며 Apache 2.0 라이선스로 배포된다.

## 어떤 문제를 해결하는가

무료 AI 모델은 많지만, 막상 쓰려고 하면 여러 장벽에 부딪힌다.
Free-Router는 다음과 같은 불편을 겨냥한다.

| 문제 | 설명 |
|------|------|
| 제공자 선택의 어려움 | 여러 무료 제공자 간 어떤 모델을 쓸지 판단하기 어려움 |
| 성능 비교와 모니터링 | 모델의 지연시간·가용성을 비교하고 추적하기 불편함 |
| API 키 관리 | 키 설정과 관리가 복잡함 |
| 도구 통합 | OpenCode, OpenClaw, Hermes Agent 등에 모델을 연결하는 번거로움 |

## 주요 기능

### 실시간 TUI 모니터링

Free-Router는 대화형 터미널 UI를 제공하며, 모든 모델에 대해 2초마다 병렬로 핑을 보낸다.
기본 정렬은 가용성, 상위 tier, 낮은 지연시간 순이며, 화면에는 다음과 같은 정보가 컬럼으로 표시된다.

| 컬럼 | 의미 |
|------|------|
| Tier | SWE-bench 점수 기반 등급 |
| Provider | NIM 또는 OpenRouter |
| Ctx | 컨텍스트 윈도우 크기 |
| AA | Arena Elo 지능 점수 |
| Avg / Lat | 평균 지연시간 / 최신 핑 지연시간 |
| Up% | 가용성 비율 |
| Verdict | 상태 요약 |

`/` 키로 모델을 검색·필터링하고, `T` 키로 tier 필터를 순환할 수 있다.
제공자 배지는 정상(✓), 만료(✗), 누락(○)으로 상태를 표시한다.

### Tier와 Verdict 척도

모델 등급은 SWE-bench Verified 점수를 기준으로 매겨진다.

| Tier | 점수 | 설명 |
|------|------|------|
| S+ | 70% 이상 | 최고급 frontier |
| S | 60-70% | 우수 |
| A+ | 50-60% | 좋음 |
| A | 40-50% | 양호 |
| A- | 35-40% | 괜찮음 |
| B+ | 30-35% | 평균 |
| B | 20-30% | 평균 이하 |
| C | 20% 미만 | 경량/엣지 |

Verdict는 핑 결과를 바탕으로 모델의 현재 상태를 요약한다.

| 상태 | 트리거 | 의미 |
|------|--------|------|
| Overloaded | HTTP 429 | 과부하 |
| Unstable | 성공 후 실패 | 불안정 |
| Not Active | 응답 없음 | 비활성 |
| Perfect | 평균 400ms 미만 | 완벽 |
| Normal | 평균 1000ms 미만 | 정상 |
| Unusable | 평균 5000ms 이상 | 사용 불가 |

### 대상 도구 통합

Free-Router의 차별점은 선택한 모델을 곧바로 다른 도구의 설정 파일에 써주는 핸드오프(handoff) 기능이다.
리스트에서 `Enter`를 누르면 현재 모델이 대상 도구에 자동 구성된다.

| 대상 도구 | 설정 파일 |
|------|------|
| OpenCode | ~/.config/opencode/opencode.json |
| OpenClaw | ~/.openclaw/openclaw.json |
| Hermes Agent | ~/.hermes/config.yaml |

기존 대상 설정은 쓰기 전에 자동으로 백업되며, 키가 누락된 경우 provider 변환 시 추가 여부를 확인한다.
미지원 모델은 메타데이터 기반으로 확인 후 다른 모델로 폴백한다.

## 설치와 사용법

npm, npx, bun 등 다양한 방식으로 설치하거나 즉시 실행할 수 있다.

```bash
# npm으로 전역 설치
npm i -g @bytonylee/free-router

# npx로 즉시 실행
npx @bytonylee/free-router

# bun 사용
bunx @bytonylee/free-router
bun install -g @bytonylee/free-router
```

설치 후 `free-router` 명령으로 실행하며, 첫 실행 시 설정 마법사에서 API 키를 입력한다(ESC로 건너뛰기 가능).

### API 키와 설정 파일

지원하는 제공자는 NVIDIA NIM과 OpenRouter 두 가지이며, 각각 무료 키를 발급받을 수 있다.

| Provider | 무료 키 발급처 | 키 접두사 |
|------|------|------|
| NVIDIA NIM | build.nvidia.com | nvapi- |
| OpenRouter | openrouter.ai | sk-or- |

API 키는 환경변수, `~/.free-router.json`, 키리스 핑 순으로 우선 적용된다.
환경변수로 키를 전달하는 예시는 다음과 같다.

```bash
NVIDIA_API_KEY=nvapi-xxx free-router
OPENROUTER_API_KEY=sk-or-xxx free-router
```

설정 파일은 `~/.free-router.json`에 권한 0600으로 저장되며, 키와 provider 활성화 상태, UI 옵션을 담는다.

```json
{
  "apiKeys": {
    "nvidia": "nvapi-xxx",
    "openrouter": "sk-or-xxx"
  },
  "providers": {
    "nvidia": { "enabled": true },
    "openrouter": { "enabled": true }
  },
  "ui": {
    "scrollSortPauseMs": 1500
  }
}
```

### 최고 모델 자동 선택

스크립트에서 활용할 수 있도록 최상의 모델 ID만 출력하는 옵션도 제공한다.

```bash
free-router --best
```

약 10초간 분석한 뒤 최적 모델 ID를 출력하므로, 이를 다른 명령의 입력으로 연결할 수 있다.

```bash
# 스크립트에서 최고 모델 사용
MODEL=$(free-router --best)
curl -X POST https://api.example.com \
  -H "Authorization: Bearer $API_KEY" \
  -d "{\"model\": \"$MODEL\"}"
```

## 결론

Free-Router는 흩어져 있는 무료 AI 모델을 한 화면에서 실시간으로 비교하고, 가장 좋은 모델을 골라 개발 도구에 곧바로 연결해주는 CLI 도구다.
2초 간격의 병렬 핑으로 지연시간과 가용성을 모니터링하고, SWE-bench 기반 tier로 품질을 가늠하며, OpenCode·OpenClaw·Hermes Agent에 자동 구성까지 처리한다.
무료 모델을 자주 갈아타며 쓰는 개발자라면 키 관리와 도구 통합의 번거로움을 줄이는 실용적인 선택지가 될 수 있다.

## Reference

- [Free-Router GitHub](https://github.com/bytonylee/free-router/)
