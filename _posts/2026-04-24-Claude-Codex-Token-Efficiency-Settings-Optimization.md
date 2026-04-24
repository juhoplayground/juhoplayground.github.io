---
layout: post
title: "Claude와 Codex 토큰 효율 높이기: 설정 조정으로 누수를 막는 법"
author: 'Juho'
date: 2026-04-24 00:00:00 +0900
categories: [AI]
tags: [LLM, Context, Prompt]
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
2. [토큰 누수의 세 가지 원인](#토큰-누수의-세-가지-원인)
3. [Claude Code 최적화](#claude-code-최적화)
4. [Codex CLI 최적화](#codex-cli-최적화)
5. [커뮤니티 반응](#커뮤니티-반응)
6. [결론](#reference)
7. [Reference](#reference)

## 개요

stdy.blog의 한 개발자가 Claude Code와 Codex에서 토큰 소비를 줄이는 설정 조정법을 공개했다.
두 도구 모두 기본 설정에서는 세션당 상당한 토큰을 자동으로 추가하며, 이것이 누적되면 실사용 비용이 크게 늘어난다.
저자는 공식 문서와 소스 코드를 직접 확인해 실증적으로 검증한 최적화 팁을 정리했다.

## 토큰 누수의 세 가지 원인

저자는 토큰 낭비의 원천을 세 가지로 분류했다.

| 원인 | 설명 |
|------|------|
| 세션/턴 자동 추가 텍스트 | 매 턴마다 시스템이 주입하는 고정 컨텍스트 |
| 도구 출력 누적 | 긴 bash/파일 읽기 결과가 히스토리에 쌓임 |
| 외부 연결 | 검색, 커넥터, IDE 연동으로 들어오는 추가 토큰 |

이 세 경로를 각각 차단하는 것이 최적화의 핵심이다.

## Claude Code 최적화

Claude Code에서 적용할 수 있는 주요 조정 사항이다.

- Git 자동 지시문과 IDE 자동 연결 비활성화
- bash, 파일 읽기, MCP에 출력 길이 제한 설정
- 환경 변수 플래그 활용 (예시 아래 코드)
- `--tools` 파라미터로 필요한 도구만 선택적 활성화
- 커스텀 시스템 프롬프트로 기본 지시문 대체

```bash
export ENABLE_CLAUDEAI_MCP_SERVERS=false
claude --tools "Read,Edit,Bash"
```

세션 시작 시 주입되는 기본 지시문이 상당한 토큰을 차지하므로 필요 없는 기능을 꺼두는 것이 즉각적인 효과를 낸다.

## Codex CLI 최적화

Codex CLI에서는 다음과 같은 조정이 효과적이다.

- MCP 앱과 웹 검색 비활성화
- 출력 토큰 상한 설정
- 프로필 구성과 JSON 출력 모드 활용
- 읽기 전용 샌드박스와 ephemeral 세션 사용

ephemeral 세션은 대화가 히스토리에 누적되지 않도록 하여 장기 세션의 토큰 폭증을 막는다.
읽기 전용 샌드박스는 불필요한 파일 쓰기 확인 절차와 그에 따른 토큰 소비를 제거한다.

## 커뮤니티 반응

GeekNews 토론에서는 다양한 관련 경험이 공유됐다.
한 사용자는 미사용 skills와 비대해진 설정 파일을 스캔해 정리해주는 "claude-slim" 도구를 직접 개발했다고 밝혔다.
다른 사용자는 Claude의 토큰 소비 한도에 불만을 표하며 실사용에서는 Codex를 선호한다고 언급했다.

## 결론

토큰 효율화는 단순히 비용 절감을 넘어 컨텍스트 윈도우를 더 오래 유지하는 실용적 효과를 낸다.
기본 설정을 그대로 두면 보이지 않는 곳에서 토큰이 빠져나가므로 한 번쯤은 환경 변수와 도구 플래그를 점검할 가치가 있다.
자신의 작업 패턴에 맞춰 필요한 도구만 선택적으로 활성화하는 것이 가장 효과적인 최적화 방법이다.

## Reference

- [Increasing Token Efficiency by Setting Adjustment in Claude and Codex](https://www.stdy.blog/increasing-token-efficiency-by-setting-adjustment-in-claude-and-codex/)
