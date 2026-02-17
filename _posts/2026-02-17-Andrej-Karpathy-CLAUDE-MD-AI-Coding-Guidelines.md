---
layout: post
title: "Andrej Karpathy의 CLAUDE.md - AI 코딩 실수를 줄이는 65줄 가이드라인"
author: 'Juho'
date: 2026-02-17 01:00:00 +0900
categories: [AI]
tags: [AI, Prompt, LLM]
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
2. [CLAUDE.md란 무엇인가](#claudemd란-무엇인가)
3. [4가지 핵심 원칙](#4가지-핵심-원칙)
4. [왜 이 파일이 주목받았는가](#왜-이-파일이-주목받았는가)
5. [실무 적용 방법](#실무-적용-방법)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Andrej Karpathy가 영감을 준 65줄짜리 마크다운 파일이 GitHub에서 하루 만에 400개 이상의 스타를 받으며 화제가 되었습니다.
이 파일은 LLM이 코딩할 때 자주 저지르는 실수를 줄이기 위한 행동 가이드라인을 담고 있습니다.
프로젝트에 구애받지 않는 범용 템플릿으로, 프로젝트별 지침과 함께 사용하도록 설계되었습니다.

## CLAUDE.md란 무엇인가

CLAUDE.md는 Claude Code가 프로젝트 디렉토리에서 자동으로 읽어들이는 설정 파일입니다.
이 파일에 작성된 지침은 AI가 코드를 생성할 때 따라야 할 규칙으로 작동합니다.
Karpathy 스타일의 CLAUDE.md는 특정 프로젝트가 아닌, AI 코딩 보조 도구의 공통적인 문제점을 해결하는 데 초점을 맞춥니다.

핵심 철학은 속도보다 신중함을 우선시하는 것입니다.
사소한 작업에는 판단력을 사용하되, 복잡한 작업에서는 반드시 이 가이드라인을 따르도록 권장합니다.

## 4가지 핵심 원칙

### 1. 코드 작성 전에 생각하기 (Think Before Coding)

가정을 명시적으로 밝히고, 불확실하면 질문하라는 원칙입니다.
AI가 혼란스러운 부분을 숨기지 않고, 여러 해석이 가능한 경우 모든 선택지를 제시해야 합니다.

핵심 지침은 다음과 같습니다:
- 가정을 명시적으로 밝힐 것
- 불확실한 경우 반드시 질문할 것
- 조용히 결정하지 말고 트레이드오프를 표면화할 것
- 불명확한 요구사항에 대해 우려를 제기할 것

### 2. 단순함 우선 (Simplicity First)

요청받은 것만 해결하는 최소한의 코드를 작성하라는 원칙입니다.
추측성 기능이나 한 번만 사용할 추상화를 만들지 않아야 합니다.

핵심 지침은 다음과 같습니다:
- 요청된 것 이상의 기능을 추가하지 않을 것
- 일회성 코드에 대한 추상화를 만들지 않을 것
- 불필요한 에러 핸들링을 추가하지 않을 것
- 코드가 과도하게 복잡한지 스스로 질문할 것

### 3. 정밀한 변경 (Surgical Changes)

기존 코드를 수정할 때 필요한 부분만 정확하게 변경하라는 원칙입니다.
자신이 만든 코드만 정리하고, 관련 없는 코드를 리팩토링하지 않아야 합니다.

핵심 지침은 다음과 같습니다:
- 필요한 부분만 수정할 것
- 기존 코드 스타일을 따를 것
- 자신의 변경으로 불필요해진 코드만 제거할 것
- 관련 없는 코드를 건드리지 않을 것

### 4. 목표 중심 실행 (Goal-Driven Execution)

모호한 작업을 검증 가능한 목표로 변환하라는 원칙입니다.
구현 전에 성공 기준을 정의하고, 각 단계마다 검증 체크포인트를 설정해야 합니다.

핵심 지침은 다음과 같습니다:
- 작업을 검증 가능한 목표로 변환할 것
- 간략한 다단계 계획을 세울 것
- 각 단계마다 검증 체크포인트를 설정할 것
- 실수 후가 아닌 구현 전에 명확화 질문을 할 것

## 왜 이 파일이 주목받았는가

이 파일이 4,000개 이상의 스타를 받은 이유는 AI 코딩 도구 사용자들이 공통적으로 겪는 문제를 정확히 짚었기 때문입니다.

### AI 코딩 도구의 흔한 문제점

| 문제 | 해결하는 원칙 |
|------|---------------|
| 과도한 기능 추가 | 단순함 우선 |
| 불필요한 리팩토링 | 정밀한 변경 |
| 숨겨진 가정 | 코드 작성 전에 생각하기 |
| 방향 없는 구현 | 목표 중심 실행 |

사용자들은 이 파일을 적용한 후 과도한 창의성과 불필요한 리팩토링이 줄어들었다고 보고했습니다.
거대한 AI 모델 자체의 개선보다 간단한 프롬프트 엔지니어링이 더 효과적일 수 있다는 점을 시사합니다.

### 성공 지표

이 가이드라인을 적용했을 때 기대할 수 있는 결과는 다음과 같습니다:
- 불필요한 diff 변경 감소
- 과도한 복잡성으로 인한 재작성 감소
- 실수 후가 아닌 구현 전에 명확화 질문 발생
- 기존 코드 스타일과의 일관성 유지

## 실무 적용 방법

이 가이드라인을 프로젝트에 적용하려면 CLAUDE.md 파일을 프로젝트 루트에 생성하면 됩니다.

```markdown
# CLAUDE.md

## Think Before Coding
- State your assumptions explicitly
- If uncertain, ask

## Simplicity First
- No features beyond what was asked
- No abstractions for single-use code

## Surgical Changes
- Touch only what you must
- Match existing code style

## Goal-Driven Execution
- Convert tasks into verifiable objectives
- State a brief multi-step plan with verification checkpoints
```

이 4가지 원칙을 프로젝트별 지침과 함께 사용하면 AI 코딩 도구의 출력 품질을 크게 향상시킬 수 있습니다.
프로젝트별 환경 변수, 빌드 명령어, 코드 컨벤션 등의 구체적인 지침도 함께 추가하는 것이 권장됩니다.

## 결론

65줄의 마크다운 파일이 AI 코딩의 품질을 개선할 수 있다는 것은 프롬프트 엔지니어링의 위력을 보여주는 사례입니다.
핵심은 AI에게 무엇을 하라가 아니라, 무엇을 하지 말라를 알려주는 것입니다.
속도보다 신중함을 우선시하고, 단순함을 유지하며, 필요한 부분만 정확히 변경하는 것이 이 가이드라인의 철학입니다.

## Reference

- [Andrej Karpathy Skills - CLAUDE.md](https://github.com/forrestchang/andrej-karpathy-skills/blob/main/CLAUDE.md/)
- [65 Lines of Markdown - a Claude Code Sensation](https://tildeweb.nl/~michiel/65-lines-of-markdown-a-claude-code-sensation.html/)
