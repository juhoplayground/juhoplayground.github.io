---
layout: post
title: "AI 에이전트의 불편한 진실 - 보안과 유용성은 제로섬 게임"
author: 'Juho'
date: 2026-02-23 01:00:00 +0900
categories: [AI]
tags: [AI, Security, MCP]
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
2. [Claude Desktop Extensions 취약점](#claude-desktop-extensions-취약점)
3. [ClawHub 악성 스킬 사례](#clawhub-악성-스킬-사례)
4. [구조적 취약성의 본질](#구조적-취약성의-본질)
5. [보안과 유용성의 갈등](#보안과-유용성의-갈등)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

AI 에이전트가 강력해질수록 보안 위협도 커진다.
최근 Claude Desktop Extensions(DXT)에서 CVSS 10점 만점의 원격 코드 실행 취약점이 발견됐고, AI 에이전트 플랫폼을 노리는 공급망 공격도 등장했다.
이 글에서는 AI 에이전트의 보안 위협 사례와 그 구조적 원인을 살펴본다.

## Claude Desktop Extensions 취약점

보안 연구팀은 MCP 서버가 샌드박스 없이 시스템 전체 권한으로 실행된다는 사실을 발견했다.
이를 통해 파일 접근, 시스템 명령 실행, 인증 정보 접근이 가능하다.

실제 공격 시나리오는 다음과 같이 작동한다.
구글 캘린더에 악의적 지시를 포함한 일정을 생성하면, Claude가 자율적으로 MCP 확장을 조합해 확인 없이 코드를 실행한다.
사용자 개입이나 확인 대화상자가 전혀 필요하지 않다.

## ClawHub 악성 스킬 사례

OpenClaw의 ClawHub에서는 악성 스킬이 "Google Services" 패키지로 위장한 사례가 발견됐다.
해당 스킬은 SKILL.md 파일에 사회공학 훅을 포함하고, 존재하지 않는 유틸리티를 요청했다.
사용자가 터미널에 명령을 붙여넣기하면 멀웨어가 설치되는 구조였다.

더 심각한 것은 이 악성 스킬이 최다 다운로드 순위에 올라 있었다는 점이다.
조사 결과 수백 개의 유사한 악성 스킬이 추가로 확인됐다.

## 구조적 취약성의 본질

이 문제의 근본 원인은 언어 모델이 콘텐츠와 명령을 구분할 수 없다는 점이다.
마크다운 같은 문서 형식이 실행 지시서로 변환되는 구조 자체가 취약점이다.

공격자는 이 특성을 활용해 일반적인 콘텐츠처럼 보이는 문서 안에 악의적인 명령을 숨길 수 있다.
AI 에이전트는 이를 구별하지 못하고 그대로 실행한다.

## 보안과 유용성의 갈등

Anthropic이 이 문제를 의도적으로 수정하지 않은 이유가 있다.
고치면 에이전트의 자율성이 감소하기 때문이다.

아래 표는 이 딜레마를 명확하게 보여준다.

| 목표 | 필요 조건 | 부작용 |
|-----|---------|-------|
| 보안 강화 | 권한 제한, 사용자 확인, 도구 제약 | 유용성 감소 |
| 유용성 강화 | 광범위한 권한, 자율 실행 | 공격 표면 증가 |

이것이 AI 에이전트의 "불편한 진실"이다.
보안과 유용성은 현재 구조에서 제로섬 게임이다.

## 결론

AI 에이전트를 기업 환경에서 사용할 때는 이 구조적 취약성을 반드시 인지해야 한다.
기업 기기에서 Claude Desktop Extensions, OpenClaw 등을 사용 중이라면 즉시 보안팀에 보고하는 것이 권장된다.
장기적으로는 에이전트의 자율성을 유지하면서도 보안을 강화할 수 있는 아키텍처 연구가 필요하다.
현재 단계에서는 최소 권한 원칙을 적용하고, 에이전트의 행동을 로깅하고 감시하는 것이 최선의 대응책이다.

## Reference

- [Claude Desktop Extensions RCE - LayerX Security](https://layerxsecurity.com/blog/claude-desktop-extensions-rce/)
- [Claude Calendar 악용 사례 - The Register](https://www.theregister.com/2026/02/11/claude_desktop_extensions_prompt_injection/)
- [ClawHub 악성 스킬 분석 - Snyk](https://snyk.io/blog/clawhub-malicious-google-skill-openclaw-malware/)
- [OpenClaw 공격 표면 분석 - 1Password](https://1password.com/blog/from-magic-to-malware-how-openclaws-agent-skills-become-an-attack-surface)
