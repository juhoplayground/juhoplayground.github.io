---
layout: post
title: "Gemini CLI 훅 기능, AI 에이전트에 보안 정책 자동 주입"
author: 'Juho'
date: 2026-02-20 01:00:00 +0900
categories: [AI]
tags: [AI, LLM, Security]
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
2. [훅의 작동 방식](#훅의-작동-방식)
3. [실전 사례: 비밀키 검사](#실전-사례-비밀키-검사)
4. [구성 방법](#구성-방법)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google이 Gemini CLI에 도입한 훅(Hooks) 기능은 AI 에이전트의 자동화와 통제 사이의 딜레마를 해결한다.
코드 수정 없이 특정 시점에 사용자 지정 스크립트를 자동으로 실행할 수 있으며, 보안 정책을 자동으로 강제할 수 있다.
v0.26.0부터 기본 활성화된 이 기능은 단순 편의 기능이 아니라 AI 에이전트 신뢰성 확보의 핵심 메커니즘이다.

## 훅의 작동 방식

훅은 다음과 같은 흐름으로 작동한다.

1. 에이전트가 특정 작업을 시도한다 (파일 쓰기, 도구 실행 등).
2. CLI가 해당 이벤트의 훅 스크립트를 호출한다.
3. 훅이 작업 내용을 분석한 후 allow 또는 deny를 반환한다.
4. deny 시 에이전트가 작업을 취소하고 다시 시도한다.

이 구조를 통해 에이전트의 모든 작업을 사전에 검증하고, 위험한 동작을 자동으로 차단할 수 있다.

## 실전 사례: 비밀키 검사

정규식을 통해 민감한 문자열을 사전에 탐지하고 거부하는 방식이다.
다음과 같은 패턴을 감지할 수 있다.

- api_key
- password
- AWS 액세스 키
- 기타 시크릿 토큰

에이전트가 코드에 하드코딩된 비밀키를 작성하려 할 때, 훅이 이를 감지하고 자동으로 차단한다.

## 구성 방법

`.gemini/settings.json` 파일에서 이벤트별 훅을 설정한다.
`matcher` 속성으로 특정 도구에만 훅을 적용할 수 있다.

```json
{% raw %}{
  "hooks": {
    "preToolUse": [
      {
        "matcher": "WriteFile",
        "command": "python check_secrets.py"
      }
    ]
  }
}{% endraw %}
```

이벤트 타입에 따라 다양한 시점에 훅을 삽입할 수 있다.

- preToolUse: 도구 실행 전 검증
- postToolUse: 도구 실행 후 후처리
- preEdit: 파일 편집 전 검사

## 결론

Gemini CLI 훅 기능은 AI 에이전트의 자율성을 유지하면서도 보안 정책을 자동으로 적용할 수 있는 실용적인 메커니즘이다.
특히 비밀키 유출 방지, 파일 접근 제한, 위험한 명령어 차단 등 다양한 보안 시나리오에 활용할 수 있다.
AI 에이전트를 프로덕션 환경에서 운영하는 팀이라면 훅 기반 보안 정책 적용을 검토해볼 필요가 있다.

## Reference

- [Tailor Gemini CLI to your workflow with hooks - Google Developers Blog](https://developers.googleblog.com/en/tailor-gemini-cli-to-your-workflow-with-hooks/)
