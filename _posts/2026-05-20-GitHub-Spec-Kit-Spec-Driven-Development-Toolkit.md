---
layout: post
title: "GitHub Spec Kit 명세서가 실행 가능한 산출물이 되는 개발 방법론"
author: 'Juho'
date: 2026-05-20 00:00:00 +0900
categories: [VibeCoding]
tags: [VibeCoding, Documentation, AI]
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
2. [환경 설정](#환경-설정)
   - [설치 방법](#설치-방법)
   - [지원 에이전트](#지원-에이전트)
3. [Spec-Driven 워크플로우](#spec-driven-워크플로우)
   - [6단계 명령어](#6단계-명령어)
   - [프로젝트 구조](#프로젝트-구조)
4. [확장과 커스터마이징](#확장과-커스터마이징)
5. [주의사항](#주의사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

GitHub Spec Kit은 명세서를 실행 가능한 산출물로 다루는 오픈소스 툴킷이다.
기존 개발 방식에서 명세서는 코드를 짤 때 참고하는 보조 문서에 가까웠다.
Spec Kit은 그 순서를 뒤집어, 잘 작성된 명세서가 구현을 직접 끌고 나가는 1차 자료가 되도록 만든다.
"intent-driven development"라는 표현이 핵심을 잘 보여준다.
무엇을 만들지(what)를 먼저 정확히 정의하고, 어떻게 만들지(how)는 그다음에 결정한다.
현재 저장소는 97.9k의 GitHub 스타와 8.5k의 포크, 144개의 릴리스를 기록하고 있고 최신 버전은 v0.8.9다.

## 환경 설정

Spec Kit은 Python 기반 CLI 도구로 제공된다.
사전 요구사항은 Python 3.11 이상, Git, uv 또는 pipx, 그리고 지원되는 AI 에이전트다.

### 설치 방법

영구 설치를 원하면 uv tool로 설치한다.

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git@vX.Y.Z
```

pipx를 선호하면 동일한 명령을 pipx로 대체할 수 있다.

```bash
pipx install git+https://github.com/github/spec-kit.git@vX.Y.Z
```

한 번만 실행하려면 uvx로 즉시 실행 가능하다.

```bash
uvx --from git+https://github.com/github/spec-kit.git@vX.Y.Z specify init <PROJECT_NAME>
```

PyPI에 유사한 이름의 패키지가 있을 수 있으나, 공식 패키지는 GitHub 저장소의 것뿐이다.

### 지원 에이전트

Spec Kit은 30종 이상의 코딩 에이전트와 통합된다.
주력은 GitHub Copilot이고, Claude Code, Google Gemini CLI, Cursor, Codex CLI, Tabnine, opencode 등이 모두 지원된다.
각 에이전트는 슬래시 명령 또는 스킬 모드로 Spec Kit 명령을 노출한다.
에이전트 자동 감지가 실패하면 --ignore-agent-tools 플래그를 사용해 우회할 수 있다.

## Spec-Driven 워크플로우

Spec Kit의 핵심은 6단계로 정형화된 명령 시퀀스다.
각 단계는 별도의 슬래시 명령으로 호출되고, 그 결과물은 모두 파일로 남는다.

### 6단계 명령어

| 단계 | 명령 | 산출물 |
|------|------|--------|
| 1 | /speckit.constitution | 프로젝트 원칙과 거버넌스 |
| 2 | /speckit.specify | 기능 명세와 사용자 스토리 |
| 3 | /speckit.clarify | 모호한 요구사항 정리 |
| 4 | /speckit.plan | 기술 구현 전략 |
| 5 | /speckit.tasks | 의존성 포함된 작업 분할 |
| 6 | /speckit.implement | 실제 구현 실행 |

추가 명령으로 /speckit.analyze는 산출물 간 일관성을 검사하고 /speckit.checklist는 품질 검증 체크리스트를 만들어준다.

예제 워크플로우는 다음과 같이 흐른다.

```text
/speckit.specify
Build an application organizing photos into albums grouped by date,
with drag-and-drop reorganization and tile-preview interface within albums.

/speckit.plan
Use Vite with vanilla HTML, CSS, JavaScript. Store metadata in local SQLite.

/speckit.tasks

/speckit.implement
```

specify 단계에서는 기술 스택을 언급하지 않는다는 점이 중요하다.
스택 결정은 plan 단계로 미루고, specify는 사용자가 보는 동작과 결과만 정의한다.

### 프로젝트 구조

specify init이 실행되면 다음 구조가 만들어진다.

```text
.specify/
  memory/
    constitution.md
  scripts/
    check-prerequisites.sh
    create-new-feature.sh
  specs/
    [FEATURE-NAME]/
      spec.md
      plan.md
      tasks.md
  templates/
    spec-template.md
    plan-template.md
    tasks-template.md
```

각 기능은 specs 하위의 별도 폴더에 spec/plan/tasks 세 파일로 누적된다.
constitution.md는 프로젝트 전역의 원칙 문서로 모든 후속 명세에 영향을 준다.

## 확장과 커스터마이징

Spec Kit은 100개 이상의 커뮤니티 확장과 프리셋을 제공한다.
영역별 대표 확장으로 Architecture Guard(아키텍처 거버넌스), MAQA(다중 에이전트 오케스트레이션), 그리고 Jira·Azure DevOps·GitHub Issues 연동 CI/CD 통합이 있다.

```bash
specify extension search
specify extension add <extension-name>
specify preset search
specify preset add <preset-name>
```

템플릿 해석은 우선순위를 가진다.
프로젝트 로컬 오버라이드(.specify/templates/overrides/)가 가장 높고, 그다음 설치된 프리셋, 설치된 확장, 마지막으로 Spec Kit 기본 템플릿 순서다.
프로젝트별 규칙을 강제하고 싶다면 로컬 오버라이드를 두는 것이 가장 명확하다.

지원 시나리오는 그린필드(0에서 1로 새 프로젝트 생성), 창의적 탐색(다양한 스택의 병렬 구현), 브라운필드(기존 코드베이스에 기능 추가) 세 가지를 모두 포함한다.

## 주의사항

리눅스 환경에서는 Git 인증 문제가 자주 발생하므로 Git Credential Manager v2.6.1 이상을 설치해야 한다.
/speckit.implement는 npm, dotnet 같은 로컬 CLI를 직접 호출하므로 필요한 도구가 사전에 설치되어 있어야 한다.
에이전트가 슬래시 명령이나 스킬 모드를 지원하는지 확인하고, init 이후 /speckit.constitution이 정상적으로 노출되는지 검증해야 한다.
PyPI에서 유사 이름 패키지를 설치하지 않도록 주의해야 하며, 공식 저장소(github/spec-kit) 외 채널은 관리되지 않는다.

## 결론

GitHub Spec Kit은 명세서를 부수적인 산출물이 아니라 1차 자료로 삼는 개발 방식을 코드로 구현한 툴킷이다.
constitution에서 implement까지 이어지는 6단계 명령은 무엇을 만들지와 어떻게 만들지를 분리하면서도 산출물을 파일로 남겨 검토 가능하게 만든다.
30개 이상의 코딩 에이전트가 모두 동일한 워크플로우 위에서 동작하므로, 도구를 바꿔도 프로세스는 유지된다.
"AI에게 막연히 코드를 짜달라"는 단계에서 한 발 나아가, AI가 따라야 할 명세 자체를 협업으로 정련하고 싶을 때 적합한 선택지다.

## Reference

- [GitHub Spec Kit](https://github.com/github/spec-kit){:target="_blank"}
