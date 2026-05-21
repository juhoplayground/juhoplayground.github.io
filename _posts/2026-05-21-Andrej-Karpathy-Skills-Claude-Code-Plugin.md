---
layout: post
title: "Andrej Karpathy Skills - LLM 코딩 함정을 줄이는 Claude Code 플러그인"
author: 'Juho'
date: 2026-05-21 00:00:00 +0900
categories: [AI]
tags: [AI, Skill, Skill Development, Agent]
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
2. [네 가지 핵심 원칙](#네-가지-핵심-원칙)
   - [Think Before Coding](#think-before-coding)
   - [Simplicity First](#simplicity-first)
   - [Surgical Changes](#surgical-changes)
   - [Goal-Driven Execution](#goal-driven-execution)
3. [Claude Code Skills와의 연결](#claude-code-skills와의-연결)
4. [설치와 사용](#설치와-사용)
5. [의미와 시사점](#의미와-시사점)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

`multica-ai/andrej-karpathy-skills`는 Andrej Karpathy가 X에 공유한 LLM 코딩 관찰을 Claude Code용 가이드라인으로 옮긴 오픈소스 저장소입니다.
핵심은 단일 `CLAUDE.md` 파일로, 플러그인으로 설치하거나 프로젝트에 직접 추가해 사용할 수 있습니다.
저장소는 Karpathy가 지적한 LLM 코딩 함정을 네 가지 실행 가능한 원칙으로 재정리합니다.
LLM이 모델의 가정을 명시하지 않고, 코드를 과도하게 복잡하게 만들며, 불필요한 리팩터링을 시도하는 경향에 대한 처방을 담고 있습니다.

## 네 가지 핵심 원칙

저장소는 LLM 코딩에서 반복적으로 발생하는 문제를 네 가지 원칙으로 분류합니다.

### Think Before Coding

구현에 들어가기 전에 가정과 모호함을 먼저 다루는 원칙입니다.
Karpathy는 "모델이 잘못된 가정을 하고, 명확화를 요청하지 않으며, 모순을 드러내지 않는다"고 지적했습니다.
이 원칙은 모델이 이해하지 못한 부분을 침묵으로 덮지 않고, 표면화하도록 유도합니다.

### Simplicity First

코드 복잡도를 최소화하고, 가설적인 미래 요구사항에 대비하는 기능을 추가하지 않는 원칙입니다.
LLM은 "혹시 모를" 분기를 추가해 코드를 과도하게 복잡하게 만드는 경향이 있습니다.
필요한 만큼만 작성하라는 원칙은 이러한 경향을 차단합니다.

### Surgical Changes

요청된 부분만 정확하게 수정하고, 기존 스타일을 보존하는 원칙입니다.
LLM은 작업 범위를 넘어 주변 코드를 리팩터링하거나, 무관한 부분의 스타일을 바꾸는 일이 잦습니다.
"외과적 변경"이라는 표현은 이러한 경향에 대한 명시적 제약입니다.

### Goal-Driven Execution

명령형 지시 대신 검증 가능한 성공 기준과 테스트 루프를 정의하는 원칙입니다.
Karpathy는 "LLM은 특정 목표를 향해 루프 도는 것에 능하다"고 강조했습니다.
이를 활용해 명령을 나열하는 대신, 목표 상태와 검증 방법을 제공하면 모델이 자율적으로 성공 기준에 도달할 때까지 반복합니다.

## Claude Code Skills와의 연결

이 저장소는 Claude Code 플러그인 마켓플레이스에 `forrestchang/andrej-karpathy-skills`로 등록돼 있습니다.
Claude Code의 워크플로우 관리 시스템과 직접 통합돼, 설치 후 자동으로 적용됩니다.
저장소 구조는 Claude Code뿐 아니라 Cursor IDE까지 함께 지원합니다.

| 파일 | 역할 |
|------|------|
| CLAUDE.md | 메인 가이드라인 |
| CURSOR.md | Cursor IDE 통합 지침 |
| EXAMPLES.md | 사용 예시 |
| .cursor/rules/ | Cursor 프로젝트 규칙 |
| .claude-plugin/ | 플러그인 설정 |

라이선스는 MIT이며, 누구나 자유롭게 활용할 수 있습니다.

## 설치와 사용

설치는 두 가지 방식으로 가능합니다.

첫 번째는 Claude Code 플러그인으로 추가하는 방식입니다.

```bash
/plugin marketplace add forrestchang/andrej-karpathy-skills
```

두 번째는 프로젝트 단위로 `CLAUDE.md`를 가져오는 방식입니다.
저장소의 `CLAUDE.md` 파일을 curl로 받아 프로젝트 루트에 두거나, 기존 `CLAUDE.md`에 내용을 추가하면 됩니다.

## 의미와 시사점

이 저장소는 "LLM 코딩 베스트 프랙티스가 단일 파일에 응축될 수 있는가"라는 질문에 실용적인 답을 제시합니다.
복잡한 시스템 프롬프트나 멀티 에이전트 구성 없이, 네 가지 원칙만으로도 LLM의 흔한 실패 모드를 줄일 수 있다는 가설을 검증하는 시도입니다.

또한 Claude Code 플러그인 마켓플레이스가 단순 도구 확장을 넘어, 코딩 철학과 규범을 배포하는 채널로 활용되고 있음을 보여줍니다.
"외과적 변경", "목표 지향 실행" 같은 용어가 표준화된다면, LLM 코딩 협업에서의 공통 어휘 역할을 할 수 있습니다.

## 결론

Andrej Karpathy Skills는 LLM 코딩의 반복적 함정을 네 가지 원칙으로 정리한 미니멀한 Claude Code 플러그인입니다.
Think Before Coding, Simplicity First, Surgical Changes, Goal-Driven Execution이라는 네 축은 그 자체로 LLM 협업 체크리스트로 활용할 수 있습니다.
단일 `CLAUDE.md` 파일로 배포된다는 점에서 부담 없이 도입할 수 있으며, MIT 라이선스로 자유롭게 수정할 수 있습니다.
Karpathy의 관찰이 X 포스트에서 시작해 플러그인 형태로 응축된 흐름은, LLM 코딩 노하우가 빠르게 표준화되고 있음을 보여주는 사례입니다.

## Reference

- [multica-ai/andrej-karpathy-skills - GitHub](https://github.com/multica-ai/andrej-karpathy-skills/)
