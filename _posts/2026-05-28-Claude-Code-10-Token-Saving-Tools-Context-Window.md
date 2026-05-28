---
layout: post
title: "Claude Code 컨텍스트 80%를 살리는 토큰 절약 도구 10선"
author: 'Juho'
date: 2026-05-28 00:00:00 +0900
categories: [VibeCoding]
tags: [VibeCoding, Context, MCP, Caching, Embedding]
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
   - [출력 압축 도구](#출력-압축-도구)
   - [코드베이스 탐색 최적화](#코드베이스-탐색-최적화)
   - [컨텍스트 격리와 캐싱](#컨텍스트-격리와-캐싱)
   - [프로젝트 설정 도구](#프로젝트-설정-도구)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

@DataChaz 가 Claude Code 사용자가 컨텍스트 윈도우의 약 80%를 낭비하고 있다고 지적했다.
그는 API 비용을 극적으로 줄여주는 10가지 오픈소스 도구를 정리했다.
대부분 토큰 60~98% 절감을 표방하며, 출력 압축부터 코드베이스 의미 검색, MCP 캐싱까지 영역별로 흩어져 있다.
이 글은 10가지 도구의 핵심과 함께, 어떤 “stack” 으로 조합하면 효과가 큰지를 정리한다.

## 배경

Claude Code 워크플로우의 주요 토큰 낭비 지점은 다음과 같다.
첫째, 모델이 장황한 자연어로 응답하면서 본문이 길어진다.
둘째, 거대한 모노레포 전체를 읽어들이며 무관한 파일이 컨텍스트를 점유한다.
셋째, 터미널 출력과 로그가 그대로 컨텍스트로 들어와 누적된다.
넷째, MCP 도구가 매번 원본 데이터를 통째로 반환한다.
다섯째, 프로젝트 설정/문서가 비효율적이라 매번 같은 컨텍스트를 다시 읽힌다.
이 다섯 가지에 각각 대응하는 도구들이 있다.

## 핵심 내용

### 출력 압축 도구

| 도구 | 핵심 | 절감 효과 |
|------|------|----------|
| Caveman Claude | Claude 를 “원시인 말투”로 강제해 출력 단순화 | 출력 토큰 75% 감소 |
| Claude Token Efficient | CLAUDE.md 한 장으로 strict terseness 강제 | 코드 변경 없이 적용 |
| Token Optimizer | 컨텍스트를 갉아먹는 invisible ghost tokens 추적·제거 | 컨텍스트 품질 회복 |

Caveman Claude 는 정확도 손실 없이 출력 토큰을 75% 줄인다고 주장한다.
[github.com/juliusbrussee/caveman](https://github.com/juliusbrussee/caveman){:target="_blank"} 에서 확인할 수 있다.
Claude Token Efficient 는 [github.com/drona23/claude-token-efficient](https://github.com/drona23/claude-token-efficient){:target="_blank"} 의 CLAUDE.md 파일 하나만 레포에 떨어뜨리면 된다.
Token Optimizer 는 [github.com/alexgreensh/token-optimizer](https://github.com/alexgreensh/token-optimizer){:target="_blank"} 에서 ghost token 진단 기능을 제공한다.

### 코드베이스 탐색 최적화

| 도구 | 핵심 | 절감 효과 |
|------|------|----------|
| Code Review Graph | Tree-sitter 그래프로 “필요한 부분만” 읽음 | 거대 모노레포에서 49배 토큰 감소 |
| Claude Context | Zilliz 의 하이브리드 벡터 검색 MCP | 전체 코드베이스 컨텍스트 비용 40% 감소 |
| Token Savior | 거대 파일이 아니라 심볼 단위 탐색 + 영속 메모리 | 코드 탐색 97% 절감 |

Code Review Graph 는 [github.com/tirth8205/code-review-graph](https://github.com/tirth8205/code-review-graph){:target="_blank"} 에서 사용할 수 있고, 거대 모노레포에 적합하다.
Claude Context 는 [github.com/zilliztech/claude-context](https://github.com/zilliztech/claude-context){:target="_blank"} 가 제공하며 벡터 검색 기반 MCP 다.
Token Savior 는 [github.com/mibayy/token-savior](https://github.com/mibayy/token-savior){:target="_blank"} 에서 심볼 그래프 + 영속 메모리로 동작한다.

### 컨텍스트 격리와 캐싱

| 도구 | 핵심 | 절감 효과 |
|------|------|----------|
| RTK | 터미널 출력을 필터링하는 Rust 프록시 | 60~90% 감소, 의존성 없음 |
| Context Mode | 원시 출력을 컨텍스트가 아닌 SQLite 로 격리 | 로그·GitHub 출력 98% 감소 |
| Token Optimizer MCP | MCP 도구에 공격적 캐싱·압축 추가 | 95%+ 토큰 감소 |

RTK(Rust Token Killer)는 [github.com/rtk-ai/rtk](https://github.com/rtk-ai/rtk){:target="_blank"} 에서 터미널 출력을 가로채는 프록시로, dependency-free 가 특징이다.
Context Mode 는 [github.com/mksglu/context-mode](https://github.com/mksglu/context-mode){:target="_blank"} 가 SQLite 로 raw output 을 보관해 컨텍스트 윈도우를 보호한다.
Token Optimizer MCP 는 [github.com/ooples/token-optimizer-mcp](https://github.com/ooples/token-optimizer-mcp){:target="_blank"} 가 MCP 호출 단에 캐싱과 압축을 끼워넣는다.

### 프로젝트 설정 도구

Claude Token Optimizer 는 [github.com/nadimtuhin/claude-token-optimizer](https://github.com/nadimtuhin/claude-token-optimizer){:target="_blank"} 가 프로젝트별 최적화 prompt 를 생성해, 문서를 11K 에서 1.3K 토큰으로 줄였다고 한다.
즉 “언제 무엇을 압축할지”를 프로젝트 셋업 단계에서 결정해두는 방식이다.

## 의미와 시사점

저자는 “모든 도구를 한 번에 깔지 말고, 본인이 어디서 새고 있는지 진단한 뒤 2~3개만 골라라”라고 권한다.
그가 제시한 god-tier stack 은 다음과 같다.

| 상황 | 추천 조합 |
|------|----------|
| 거대 모노레포 | Code Review Graph + Token Savior |
| 무거운 터미널 출력 | RTK |
| MCP 데이터 덤프 | Context Mode |
| 즉시 효과 필요 | Caveman + Claude Token Efficient |

세션을 새로 시작한 뒤 `/context` 명령으로 어디서 토큰이 새는지 진단하고, 위 조합 중 하나를 적용해 효과를 측정하는 흐름이 가장 합리적이다.
대부분의 도구가 MCP, CLI, CLAUDE.md 라는 표준 인터페이스로 결합되기 때문에, 서로 충돌 없이 누적 적용이 가능하다.

## 결론

Claude Code 토큰 비용은 모델 가격이 아니라 “무엇을 컨텍스트에 넣는가”에 좌우된다.
출력 압축, 코드베이스 의미 검색, 터미널·MCP 격리·캐싱, 프로젝트 설정이라는 4영역에 1개씩만 적용해도 60~90% 절감이 현실적이다.
새 세션을 열고 `/context` 로 누수를 진단한 다음, 본인 워크플로우에 맞는 도구를 골라 한 번에 적용하는 것이 가장 빠른 ROI 다.

## Reference

- [Claude Code Token Tools - @DataChaz](https://x.com/DataChaz/status/2055929071733743693)
- [Caveman Claude](https://github.com/juliusbrussee/caveman)
- [RTK - Rust Token Killer](https://github.com/rtk-ai/rtk)
- [Code Review Graph](https://github.com/tirth8205/code-review-graph)
- [Context Mode](https://github.com/mksglu/context-mode)
- [Claude Token Optimizer](https://github.com/nadimtuhin/claude-token-optimizer)
- [Token Optimizer](https://github.com/alexgreensh/token-optimizer)
- [Token Optimizer MCP](https://github.com/ooples/token-optimizer-mcp)
- [Claude Context](https://github.com/zilliztech/claude-context)
- [Claude Token Efficient](https://github.com/drona23/claude-token-efficient)
- [Token Savior](https://github.com/mibayy/token-savior)
