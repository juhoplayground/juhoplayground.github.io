---
layout: post
title: "대부분이 모르는 Claude 기능 총정리: 17가지 활용 가이드"
author: 'Juho'
date: 2026-07-07 00:00:00 +0900
categories: [AI]
tags: [AI, Prompt, Caching]
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
2. [일반 Claude의 숨은 기능](#일반-claude의-숨은-기능)
   - [Projects](#projects)
   - [Artifacts](#artifacts)
   - [Adaptive Thinking](#adaptive-thinking)
   - [Memory](#memory)
3. [역할 부여 프롬프트](#역할-부여-프롬프트)
4. [대부분이 모르는 제품 기능](#대부분이-모르는-제품-기능)
   - [Claude in Chrome](#claude-in-chrome)
   - [Claude Cowork](#claude-cowork)
   - [Scheduled Tasks](#scheduled-tasks)
   - [Skills와 CLAUDE.md](#skills와-claudemd)
   - [Claude Code와 Claude Design](#claude-code와-claude-design)
5. [개발자용: Prompt Caching](#개발자용-prompt-caching)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

이 글은 Claude가 할 수 있는데도 대부분의 사용자가 모르는 기능들을 정리한 가이드다.
각 기능이 어디에 있는지, 어떻게 켜는지, 그리고 바로 쓸 수 있는 프롬프트까지 담고 있다.

핵심 메시지는 이렇다.
한 번에 전부 구현할 필요는 없다.
무엇이 존재하는지 아는 것만으로도 절반은 이긴 것이며, 목록에서 딱 하나를 골라 오늘 설정하는 것부터 시작하면 된다.

## 일반 Claude의 숨은 기능

### Projects

일반적인 Claude 대화는 매번 백지에서 시작한다.
사용자의 이름도, 하는 일도, 선호도 아무것도 기억하지 못한다.

Projects는 이 문제를 해결한다.
프로젝트를 만들고 문서를 업로드하고 표준 지침(standing instructions)을 작성해두면, Claude가 그 모든 것을 영구적으로 유지한다.
다음 주에 다시 열어도 정확히 이어서 작업할 수 있다.

프로젝트 지침(Project Instructions) 필드에 붙여넣을 수 있는 예시는 다음과 같다.

```text
You are my content research assistant. I run a newsletter about AI and crypto for a technical audience.

Always assume my readers know the basics. Don't explain what an LLM is or what a blockchain is.

When I share a topic or article, your job is to:
1. Identify the 3 most counterintuitive or surprising angles
2. Find connections to recent events I might have missed
3. Suggest how I could frame this as a story, not a summary

Tone: direct, no corporate language, no filler phrases.
Format: short paragraphs, no bullet points unless I ask.
Never start a response with "Great question" or "Certainly".
```

### Artifacts

많은 사람들이 Claude는 텍스트만 만들 수 있다고 생각한다.
Artifacts는 Claude가 대화 안에서 실제로 동작하는 무언가를 만들어내는 기능이다.

복사해서 다른 곳에 붙여야 하는 코드 덩어리가 아니라, 사이드 패널에서 바로 열고 클릭하고 사용할 수 있는 살아있는 결과물이다.
계산기, 습관 추적기, 게임, 차트가 있는 대시보드 같은 것들이다.
SVG 그래픽, 인터랙티브 차트, Mermaid 다이어그램이 모두 지원되며 무료 플랜에서도 쓸 수 있다.

### Adaptive Thinking

Extended Thinking은 Claude가 답을 내놓기 전에 문제를 단계별로 추론하는 모드다.
그리고 그 추론 과정 전체를 지켜볼 수 있다.

간단한 질문에는 필요 없다.
하지만 복잡한 의사결정, 전략 분석처럼 Claude가 패턴 매칭이 아니라 실제로 사고하길 원하는 상황에서는 출력의 차이가 크다.

### Memory

Memory를 켜면 Claude가 시간이 지나면서 사용자에 대한 프로필을 쌓는다.
직업, 진행 중인 프로젝트, 선호하는 커뮤니케이션 방식, 지금 작업 중인 내용 등을 기억한다.

완전히 새로운 대화를 시작해도 맥락을 이미 알고 있어서 자기소개를 다시 할 필요가 없다.
기본값은 꺼짐이며, 다음과 같은 프롬프트로 정보를 저장하도록 요청할 수 있다.

```text
I want you to remember the following about me so you don't need to ask again:

My name is [name]. I work as [role] at [company or project].
My main focus right now is [what you're working on].
My audience or customers are [who they are].

When I ask for help, always assume this context unless I say otherwise.
My preferred communication style: [direct / detailed / casual / formal].
Things I find annoying in responses: [e.g. bullet points, long intros, excessive caveats].

Save all of this to memory now.
```

## 역할 부여 프롬프트

Claude는 반드시 범용 AI 어시스턴트로만 동작할 필요가 없다.
구체적인 역할을 부여하면 완전히 그 역할에 몰입해, 어떻게 질문하고 무엇에 반박하는지가 달라진다.
가이드가 제시하는 역할 프롬프트는 다음과 같다.

| 역할 | 핵심 목적 |
|------|-----------|
| Personal psychologist | 답을 주기보다 질문을 던지는 CBT 치료사처럼 사고 패턴을 점검 |
| The hard mentor | 검증 대신 스트레스 테스트, 계획의 약한 가정을 찾아냄 |
| Personal trainer | 실제 수치를 기반으로 12주 맞춤 훈련 프로그램 설계 |
| Difficult conversation | 상대방 역할을 연기하며 어려운 대화를 사전 연습 |
| Devil's advocate | 이미 내린 결정에 대해 가장 강한 반대 논거를 구성 |

예를 들어 hard mentor 프롬프트는 다음과 같다.

```text
You are a brutally honest mentor. You've built and failed at multiple companies.

Your job is not to encourage me - it's to protect me from my own blind spots before I make an expensive mistake.

Rules:
- Disagree with me when you think I'm wrong. Be specific about why.
- Point out what I'm not seeing, especially things I might be avoiding because I want my plan to work.
- Ask hard questions I haven't thought to ask myself.
- If something is a bad idea, say it's a bad idea. Don't balance it with "on the other hand..."
- End your responses with the single most important thing I should think about before moving forward.
```

## 대부분이 모르는 제품 기능

### Claude in Chrome

Claude in Chrome은 Claude에게 현재 탭에 대한 완전한 가시성과 조작 능력을 주는 브라우저 확장 프로그램이다.
페이지를 읽고, 링크를 클릭하고, 폼을 채우고, 새 URL로 이동한다.

Chrome 웹 스토어에서 설치하고 Claude 계정으로 로그인한 뒤 확장 아이콘을 클릭하면 사이드바가 열린다.
채용 공고 페이지에서 모든 공고의 직무, 회사, 연봉 범위, 상위 요구사항을 추출해 연봉 순으로 정렬한 비교 표를 만드는 것 같은 작업을 평문으로 지시할 수 있다.

### Claude Cowork

Cowork는 Claude에게 파일 시스템에 대한 직접 접근을 주는 데스크톱 앱이다.
실제 파일을 읽고, 문서를 편집하고, 새로 만들고, 폴더를 정리한다.
채팅창에 아무것도 복사해 붙여넣지 않아도 된다.

### Scheduled Tasks

Scheduled Tasks는 작업을 한 번 설정해두면 Claude가 지정한 시간과 주기에 자동으로 실행하는 기능이다.
매일 아침, 매주 월요일, 매시간 등으로 설정할 수 있고 결과를 폴더에 저장한다.

예시 작업 설명은 다음과 같다.

```text
Every weekday morning at 7:30am, do the following:
1. Search for the top AI and crypto news from the last 24 hours
2. Pick the 5 most important stories - focus on things that are surprising, counterintuitive, or have real implications for builders and investors
3. For each story write: headline, 2-sentence summary, why it matters
4. Save the result as "brief-[date].md" in my /briefs folder
```

### Skills와 CLAUDE.md

Skills는 Cowork 안에서 Claude에게 특정 능력을 부여하는 사전 제작된 지침 세트다.
매번 무엇이 필요한지 설명하는 대신, 스킬을 한 번 설치하면 PowerPoint 파일 생성이나 PDF 작업 같은 유형의 작업을 Claude가 이미 알고 처리한다.
휴대폰의 앱처럼 생각하면 된다.

CLAUDE.md는 프로젝트 폴더에 만들어두는 파일로, Claude가 모든 세션 시작 시 요청하지 않아도 자동으로 읽는다.
코딩 컨벤션, 글쓰기 스타일 규칙, 회사에서 특정 의미를 갖는 용어, 브랜드 보이스 가이드라인을 한 번 적어두면 그 프로젝트의 모든 세션에서 따른다.

### Claude Code와 Claude Design

Claude Code는 채팅창이 아니라 개발 환경 안에서 직접 동작한다.
실제 코드베이스를 읽고, 새 코드를 작성하고, 테스트를 실행하고, 에러 메시지를 읽어 버그를 고치는 루프를 작업이 끝날 때까지 돌린다.
VS Code, JetBrains와 통합되며 GitHub Actions에 넣어 PR을 자동으로 리뷰하거나 작성하게 할 수 있다.

Claude Design은 시각 작업을 위한 별도의 Anthropic Labs 도구다.
제품 원페이저, 피치덱, 프로토타입, 랜딩 페이지 레이아웃을 만들고 PPTX, Canva, PDF, HTML로 내보낸다.
claude.ai/design 에서 접근할 수 있다.

## 개발자용: Prompt Caching

Claude API로 개발하는 사람들을 위한 기능이다.
요청에 긴 시스템 프롬프트, 참조 문서, 코드베이스처럼 반복되는 큰 컨텍스트 블록이 있다면, 매 호출마다 같은 토큰을 다시 처리하는 비용을 내고 있는 셈이다.

Prompt Caching은 그 콘텐츠를 서버 측에 저장한다.
이후 호출은 다시 처리하는 대신 캐시를 재사용해, 캐시된 토큰에 대해 최대 90%의 비용 절감과 더 빠른 응답을 얻는다.
캐싱하려는 콘텐츠 블록에 `cache_control` 필드를 추가하면 된다.

```json
{
  "model": "claude-opus-4-6",
  "system": [
    {
      "type": "text",
      "text": "[your large system prompt or reference document]",
      "cache_control": {"type": "ephemeral"}
    }
  ],
  "messages": [
    {
      "role": "user",
      "content": "[user message - this changes each call]"
    }
  ]
}
```

캐시는 5분간 유지되며 사용할 때마다 타이머가 리셋된다.
시스템 프롬프트, 대용량 문서, 도구 정의(tool definitions)에 모두 적용된다.

## 결론

이 가이드가 전하는 조언은 명확하다.
목록에서 기능 하나를 골라 오늘 설정하는 것이다.

무엇이 존재하는지 아는 것만으로 이미 절반은 이긴 것이다.
Projects로 맥락을 유지하고, Artifacts로 동작하는 결과물을 만들고, Scheduled Tasks로 자동화하고, 개발자라면 Prompt Caching으로 비용을 줄이는 식으로 워크플로우에 맞는 것부터 차근차근 도입하면 된다.

## Reference

- [Claude Can Do All of This (원문 게시물)](https://x.com/AnatoliKopadze/status/2057813254617858078)
