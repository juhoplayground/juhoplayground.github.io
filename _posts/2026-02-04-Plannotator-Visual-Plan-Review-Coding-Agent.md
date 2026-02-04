---
layout: post
title: Plannotator - 코딩 에이전트의 계획을 시각적으로 검토하고 주석을 다는 도구
author: 'Juho'
date: 2026-02-04 01:00:00 +0900
categories: [AI Tools]
tags: [Claude Code, Agent, Coding, Productivity]
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
2. [핵심 워크플로](#핵심-워크플로)
3. [주석 유형](#주석-유형)
4. [설치 방법](#설치-방법)
5. [코드 리뷰 기능](#코드-리뷰-기능)
6. [이미지 주석 기능](#이미지-주석-기능)
7. [메모 앱 연동](#메모-앱-연동)
8. [공유 및 프라이버시](#공유-및-프라이버시)
9. [프로젝트 정보](#프로젝트-정보)
10. [Reference](#reference)

## 개요

Plannotator는 AI 코딩 에이전트가 생성한 개발 계획을 시각적으로 검토하고 주석을 달 수 있는 브라우저 기반 도구이다.
Claude Code와 OpenCode 같은 AI 코딩 도구와 연동되어, 에이전트가 작성한 계획에 대해 팀과 협업하고 한 번의 클릭으로 피드백을 전송할 수 있다.

AI 코딩 에이전트를 사용할 때 계획 단계는 구현 품질을 결정하는 핵심 과정이다.
하지만 기존에는 에이전트의 계획을 텍스트로만 확인해야 했고, 구체적인 수정 피드백을 전달하기 어려웠다.
Plannotator는 이 문제를 해결하여 인간-에이전트 간 계획 검토 루프를 효율적으로 만들어 준다.

## 핵심 워크플로

Plannotator의 기본 워크플로는 다음과 같이 진행된다.

1. AI 에이전트가 Plan 모드에서 계획 수립을 완료한다
2. Claude Code가 ExitPlanMode를 호출하면 Hook이 작동하여 Plannotator UI가 브라우저에서 열린다
3. 사용자가 계획을 시각적으로 검토하고 주석을 단다
4. 승인(Approve)하면 에이전트가 구현을 진행한다
5. 변경 요청(Request Changes)하면 구조화된 피드백이 에이전트에게 자동 전달된다

이 방식은 빠른 Human-in-the-Loop 계획 검토 루프를 생성한다.
사용자는 계획의 특정 부분을 정확히 지정하여 삭제, 수정, 코멘트를 남길 수 있으므로, 모호한 텍스트 피드백보다 훨씬 효과적이다.

## 주석 유형

Plannotator는 다섯 가지 주석 유형을 지원한다.

| 유형 | 코드 | 설명 |
|------|------|------|
| Delete | D | 계획의 특정 부분을 삭제 표시 |
| Replace | R | 원본 텍스트를 대체 텍스트로 교체 제안 |
| Comment | C | 원본 텍스트에 대한 코멘트 추가 |
| Insert | I | 특정 위치에 새로운 텍스트 삽입 제안 |
| Global Comment | G | 계획 전체에 대한 전역 코멘트 |

UI에서 계획의 정확한 부분을 선택한 후 해당 부분에 대해 삭제, 교체, 코멘트, 삽입 등의 주석을 추가할 수 있다.
이러한 구조화된 주석은 에이전트가 피드백을 정확히 이해하고 반영할 수 있게 해준다.

## 설치 방법

### Claude Code 설치

먼저 plannotator 명령을 설치한다.

macOS, Linux, WSL 환경에서는 다음 명령을 사용한다.

```bash
curl -fsSL https://plannotator.ai/install.sh | bash
```

Windows PowerShell 환경에서는 다음 명령을 사용한다.

```powershell
irm https://plannotator.ai/install.ps1 | iex
```

이후 Claude Code 플러그인을 설치한다.

```
/plugin marketplace add backnotprop/plannotator
/plugin install plannotator@plannotator
```

플러그인 설치 후 Claude Code를 재시작해야 한다.

### OpenCode 설치

`opencode.json` 파일에 다음을 추가한다.

```json
{
  "plugin": ["@plannotator/opencode@latest"]
}
```

이후 위와 동일한 설치 스크립트를 실행하고 재시작한다.

## 코드 리뷰 기능

2026년 1월에 추가된 코드 리뷰 기능은 `/plannotator-review` 명령으로 사용할 수 있다.
git diff를 기반으로 변경된 코드를 검토하고 인라인 주석을 달 수 있다.

주요 기능은 다음과 같다.
- 라인 번호를 선택하여 특정 코드 라인에 주석 추가
- diff 뷰 간 전환 지원
- 주석이 완료되면 에이전트에게 피드백 전송

이 기능은 에이전트가 구현한 코드에 대해서도 계획 검토와 동일한 방식으로 피드백을 제공할 수 있게 해준다.

## 이미지 주석 기능

Plannotator는 이미지에 직접 주석을 추가하는 기능도 지원한다.
펜, 화살표, 원형 그리기 도구를 사용하여 시각적 피드백을 제공할 수 있다.

디자인 목업이나 UI 스크린샷에 대한 피드백을 텍스트로 설명하는 대신, 이미지 위에 직접 표시하여 명확하게 전달할 수 있다.

## 메모 앱 연동

승인된 계획을 Obsidian, Bear Notes 등의 메모 앱에 자동으로 저장하는 기능을 제공한다.
이를 통해 프로젝트의 의사결정 이력을 체계적으로 관리할 수 있다.

에이전트와 함께 수립한 계획을 별도의 복사 작업 없이 자동으로 노트 앱에 아카이빙할 수 있어, 프로젝트 문서화에 유용하다.

## 공유 및 프라이버시

Plannotator는 완전히 브라우저에서 실행된다.
계획 데이터는 사용자의 기기를 벗어나지 않는다.

공유 기능은 Textarea.my에서 영감을 받은 방식을 사용한다.
콘텐츠와 주석이 base64 문자열로 압축되어 링크 자체에 저장된다.
별도의 백엔드 서버가 없는 정적 사이트로, 개인정보 보호에 대한 우려 없이 사용할 수 있다.

(https://share.plannotator.ai/){:target="_blank"} 에서 데모를 체험할 수 있다.

## 프로젝트 정보

| 항목 | 내용 |
|------|------|
| GitHub Stars | 1.6k |
| Forks | 103 |
| Contributors | 13 |
| 최신 버전 | v0.6.8 (2026년 1월 29일) |
| 주요 언어 | TypeScript (81.5%) |
| 패키지 매니저 | Bun |
| 라이선스 | Business Source License 1.1 |
| 지원 플랫폼 | macOS, Linux, Windows (WSL, PowerShell) |

## Reference

- [Plannotator - GitHub](https://github.com/backnotprop/plannotator)
- [Plannotator - 공식 웹사이트](https://plannotator.ai/)
- [Plannotator - Product Hunt](https://www.producthunt.com/products/plannotator)
