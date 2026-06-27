---
layout: post
title: "Awesome Claude Plugins: n8n으로 GitHub 플러그인 채택 지표 자동 수집하기"
author: 'Juho'
date: 2026-06-26 00:00:00 +0900
categories: [VibeCoding]
tags: [VibeCoding, Skill, Management]
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
   - [n8n 자동화와 지표 수집](#n8n-자동화와-지표-수집)
   - [추적하는 지표 컬럼](#추적하는-지표-컬럼)
   - [상위 플러그인 저장소 목록](#상위-플러그인-저장소-목록)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

awesome-claude-plugins는 GitHub 저장소 전반에서 Claude Code 플러그인 채택 지표를 자동으로 수집하는 프로젝트입니다.
저장소 스스로 "Automated collection of Claude Code plugin adoption metrics across GitHub repositories using n8n workflows"라고 자신을 소개합니다.
즉 n8n 워크플로우를 사용해 GitHub 생태계를 스캔하고, 플러그인을 보유한 저장소를 골라 순위를 매기는 자동 큐레이션 프로젝트입니다.
이 글은 README의 사실만을 근거로 이 프로젝트의 작동 방식과 수집 지표, 그리고 상위 100개 저장소 목록을 정리합니다.

## 배경

Claude Code 플러그인과 스킬 생태계가 빠르게 커지면서, 어떤 저장소가 실제로 많이 채택되는지 한눈에 보기 어려워졌습니다.
이 프로젝트는 사람이 수작업으로 목록을 관리하는 전형적인 awesome 리스트와 달리, 채택 지표 자체를 자동으로 긁어와 정량적으로 순위를 정합니다.
README에 따르면 인덱싱 규모는 상당히 큽니다.
저장소는 "Last updated: 22.06.2026 with 23121 total repositories indexed"라고 명시하며, 23,000개가 넘는 저장소를 자동 인덱스에 담고 있다고 밝힙니다.
이 가운데 별 수, 구독자 수, 플러그인 채택률 같은 지표로 생태계 영향력을 측정해 상위 100개를 큐레이션합니다.
프로젝트 저장소 자체는 약 888개의 스타를 보유하고 있습니다.

## 핵심 내용

### n8n 자동화와 지표 수집

이 프로젝트의 핵심 엔진은 n8n 워크플로우입니다.
n8n은 노드 기반의 워크플로우 자동화 도구로, 여기서는 GitHub 저장소들로부터 지표를 주기적으로 긁어오는 역할을 담당합니다.
README는 이 워크플로우가 GitHub 생태계를 지속적으로 자동 스캔한다는 점을 "continuous automated scanning"이라는 표현으로 시사합니다.
수집한 데이터는 저장소 이름, 설명, 그리고 정량 지표로 정리되어 순위 테이블 형태로 출력됩니다.
다만 README에 노출된 내용에는 워크플로우의 단계별 동작, 갱신 주기, 저장소 발견 및 필터링 방식, 플러그인의 공식 정의가 별도 문서로 명시되어 있지는 않습니다.
대신 갱신 시점은 "Last updated" 날짜로 표기되며, 추가 자료는 awesomeclaudeplugins.com에서 제공한다고 안내합니다.

### 추적하는 지표 컬럼

큐레이션 테이블은 각 저장소마다 다음 컬럼을 표시합니다.

| 컬럼 | 설명 |
|------|------|
| Repo Name | 저장소 이름과 링크 |
| Description | 프로젝트 설명 |
| Stars | GitHub 스타 수 |
| Subs | 구독자 또는 구독 지표 |
| Plugins | 구현된 플러그인 개수 |

Stars는 저장소의 인기도를, Subs는 watch 기반의 구독 관심도를 나타냅니다.
Plugins 컬럼은 해당 저장소가 담고 있는 플러그인 수를 보여 주며, 순위는 기본적으로 스타 수를 기준으로 정렬됩니다.

### 상위 플러그인 저장소 목록

아래는 README가 제공하는 상위 100개 저장소 순위입니다.
스타 수가 높은 순으로 정렬되어 있으며, 일부 항목은 동일 이름이 중복으로 등장합니다.

| 순위 | 저장소 | 설명 | Stars | Subs | Plugins |
|------|--------|------|-------|------|---------|
| 1 | superpowers | 에이전트형 스킬 프레임워크 및 개발 방법론 | 234,966 | 877 | 1 |
| 2 | ECC | 에이전트 하네스 성능 최적화 시스템 | 219,432 | 1,127 | 1 |
| 3 | andrej-karpathy-skills | CLAUDE.md 파일을 통한 Claude Code 행동 개선 | 179,886 | 1,007 | 1 |
| 4 | prompts.chat | 프롬프트 공유 플랫폼 | 163,992 | 1,639 | 1 |
| 5 | skills | Agent Skills 공개 저장소 | 153,535 | 998 | 3 |
| 6 | claude-code | 터미널 기반 에이전트형 코딩 도구 | 133,548 | 810 | 13 |
| 7 | ui-ux-pro-max-skill | 전문 UI/UX 디자인 인텔리전스 | 94,478 | 447 | 1 |
| 8 | claude-mem | 세션 간 지속 컨텍스트 | 83,471 | 284 | 1 |
| 9 | caveman | 토큰 감소 스킬 (65% 감소) | 75,228 | 165 | 1 |
| 10 | RuView | WiFi 신호 공간 인텔리전스 시스템 | 74,879 | 526 | 1 |
| 11 | open-design | 로컬 우선 디자인 대안 | 68,378 | 230 | 1 |
| 12 | Understand-Anything | 인터랙티브 지식 그래프 생성 | 64,916 | 209 | 1 |
| 13 | agent-skills | 프로덕션급 엔지니어링 스킬 | 64,887 | 388 | 1 |
| 14 | ruflo | 멀티 에이전트 스웜용 메타 하네스 | 60,598 | 413 | 35 |
| 15 | mem0 | AI 에이전트용 범용 메모리 레이어 | 59,018 | 234 | 1 |
| 16 | context7 | LLM용 코드 문서화 플랫폼 | 57,819 | 152 | 1 |
| 17 | mempalace | 오픈소스 AI 메모리 시스템 | 56,117 | 320 | 1 |
| 18 | career-ops | AI 기반 구직 시스템 | 54,945 | 208 | 1 |
| 19 | Understand-Anything | 지식 그래프 탐색기 (중복 항목) | 54,661 | 176 | 1 |
| 20 | BMAD-METHOD | 애자일 AI 주도 개발을 위한 방법론 | 49,471 | 399 | 2 |
| 21 | taste-skill | AI 출력물 디자인 품질 개선 | 47,982 | 137 | 1 |
| 22 | slidev | 프레젠테이션 슬라이드 프레임워크 | 47,305 | 162 | 1 |
| 23 | last30days-skill | 여러 플랫폼에 걸친 리서치 종합 | 45,192 | 149 | 1 |
| 24 | headroom | 토큰 압축 도구 | 45,078 | 141 | 1 |
| 25 | ponytail | 코드 최소화 방법론 | 44,554 | 120 | 1 |
| 26 | chrome-devtools-mcp | 코딩 에이전트용 Chrome DevTools | 44,135 | 169 | 1 |
| 27 | CLI-Anything | 소프트웨어 에이전트 네이티브 CLI 시스템 | 43,555 | 173 | 1 |
| 28 | payload | 풀스택 Next.js 프레임워크 | 43,123 | 157 | 1 |
| 29 | GitNexus | 클라이언트 사이드 지식 그래프 생성기 | 42,581 | 148 | 1 |
| 30 | antigravity-awesome-skills | 1,600개 이상의 에이전트 스킬 라이브러리 | 41,327 | 306 | 59 |
| 31 | mempalace | 최고 점수 AI 메모리 시스템 (중복) | 41,278 | 263 | 1 |
| 32 | impeccable | 디자인 언어 프레임워크 | 39,916 | 72 | 1 |
| 33 | agents | 멀티 하네스 에이전트 플러그인 마켓플레이스 | 37,027 | 314 | 84 |
| 34 | oh-my-claudecode | 팀 우선 멀티 에이전트 오케스트레이션 | 36,727 | 127 | 1 |
| 35 | agent-browser | 브라우저 자동화 CLI | 36,608 | 99 | 1 |
| 36 | obsidian-skills | Obsidian CLI 에이전트 스킬 | 36,334 | 207 | 1 |
| 37 | CopilotKit | 에이전트용 프론트엔드 스택 | 35,343 | 176 | 1 |
| 38 | marketingskills | 마케팅 스킬 모음 | 34,362 | 335 | 1 |
| 39 | academic-research-skills | 연구 작문 워크플로우 스킬 | 33,372 | 108 | 1 |
| 40 | financial-services | Anthropic 금융 서비스 | 32,142 | 250 | 20 |
| 41 | claude-plugins-official | Anthropic 공식 관리 플러그인 디렉터리 | 30,571 | 192 | 234 |
| 42 | ppt-master | AI 파워포인트 생성 | 29,899 | 63 | 1 |
| 43 | hyperframes | HTML을 비디오로 렌더링 | 29,517 | 79 | 1 |
| 44 | claude-task-master | 작업 관리 시스템 | 27,651 | 158 | 1 |
| 45 | qmd | 미니 CLI 검색 엔진 | 26,872 | 90 | 1 |
| 46 | mlflow | 오픈소스 AI 엔지니어링 플랫폼 | 26,647 | 321 | 1 |
| 47 | repomix | LLM용 저장소 패킹 도구 | 26,433 | 68 | 3 |
| 48 | claude-hud | Claude Code 플러그인 활동 표시 | 25,524 | 34 | 1 |
| 49 | beads | 코딩 에이전트용 메모리 업그레이드 | 24,675 | 90 | 1 |
| 50 | planning-with-files | 지속적 파일 기반 플래닝 | 23,696 | 105 | 1 |
| 51 | agentmemory | 지속적 에이전트 메모리 | 23,605 | 65 | 1 |
| 52 | frontend-slides | 웹 기반 프레젠테이션 생성 | 22,443 | 96 | 1 |
| 53 | promptfoo | LLM 테스트 프레임워크 | 22,419 | 59 | 1 |
| 54 | awesome-claude-code-subagents | 100개 이상의 전문 서브에이전트 | 22,221 | 227 | 10 |
| 55 | baoyu-skills | 스킬 모음 | 22,097 | 100 | 1 |
| 56 | compound-engineering-plugin | Compound Engineering 공식 플러그인 | 21,860 | 113 | 1 |
| 57 | knowledge-work-plugins | 지식 노동자 플러그인 | 21,570 | 175 | 68 |
| 58 | codex-plugin-cc | Claude Code용 Codex 통합 | 21,399 | 67 | 1 |
| 59 | ralph | 자율 AI 에이전트 루프 | 20,475 | 108 | 1 |
| 60 | pm-skills | 프로젝트 관리 스킬 (100개 이상) | 20,328 | 163 | 9 |
| 61 | beads | 에이전트 메모리 업그레이드 (중복) | 20,018 | 77 | 1 |
| 62 | daily | daily.dev 전문가 네트워크 | 19,889 | 114 | 2 |
| 63 | claude-skills | 337개 Claude Code 스킬 모음 | 18,651 | 157 | 78 |
| 64 | pua | 고에너지 PIP 스킬 | 18,366 | 34 | 1 |
| 65 | context-mode | 컨텍스트 윈도우 최적화 | 17,893 | 87 | 1 |
| 66 | Anthropic-Cybersecurity-Skills | 754개 사이버보안 스킬 | 17,148 | 128 | 1 |
| 67 | hindsight | 에이전트 학습 중심 메모리 | 16,850 | 51 | 1 |
| 68 | Agent-Skills-for-Context-Engineering | 프로덕션 에이전트 시스템 스킬 | 16,667 | 99 | 1 |
| 69 | deepeval | LLM 평가 프레임워크 | 16,343 | 63 | 1 |
| 70 | browser-harness | 자가 치유 브라우저 하네스 | 15,167 | 42 | 미표기 |
| 71 | prowler | 클라우드 보안 플랫폼 | 14,019 | 119 | 1 |
| 72 | awesome-web-security | 웹 보안 리소스 | 13,497 | 351 | 1 |
| 73 | pipecat | 음성 및 멀티모달 대화형 AI | 12,930 | 75 | 1 |
| 74 | skills | MiniMax AI 스킬 | 12,728 | 52 | 1 |
| 75 | superset | AI 에이전트용 코드 에디터 | 11,988 | 28 | 1 |
| 76 | kubeshark | 쿠버네티스 네트워크 관측성 | 11,961 | 71 | 1 |
| 77 | InsForge | 에이전트형 코딩용 오픈소스 백엔드 | 11,917 | 36 | 1 |
| 78 | tambo | 생성형 UI SDK | 11,151 | 30 | 1 |
| 79 | es-toolkit | 자바스크립트 유틸리티 라이브러리 | 11,150 | 33 | 1 |
| 80 | skills | Hugging Face 생태계 스킬 | 10,708 | 50 | 18 |
| 81 | skypilot | AI 워크로드 관리 시스템 | 10,190 | 74 | 1 |
| 82 | mcp-use | 풀스택 MCP 프레임워크 | 10,128 | 89 | 3 |
| 83 | claude-skills | 66개 전문 풀스택 개발자 스킬 | 10,106 | 73 | 1 |
| 84 | AI-Research-SKILLs | AI 연구 및 엔지니어링 스킬 라이브러리 | 9,911 | 48 | 23 |
| 85 | gsap-skills | GSAP 애니메이션 플랫폼 스킬 | 9,623 | 34 | 1 |
| 86 | claude-seo | 범용 SEO 스킬 | 9,373 | 92 | 1 |
| 87 | skills | Minimalist Entrepreneur 기반 스킬 | 9,181 | 41 | 1 |
| 88 | ginkgo | Go 테스트 프레임워크 | 9,015 | 99 | 1 |
| 89 | Kami | 콘텐츠 문서화 도구 | 9,005 | 20 | 1 |
| 90 | claude-code-tips | Claude Code 팁 43선 | 8,872 | 58 | 1 |
| 91 | visual-explainer | HTML 및 슬라이드 다이어그램 생성 | 8,824 | 40 | 1 |
| 92 | connect | 스트림 처리 도구 | 8,684 | 112 | 1 |
| 93 | modelcontextprotocol | MCP 명세 및 문서 | 8,452 | 171 | 1 |
| 94 | garden-skills | 웹 디자인 및 지식 스킬 | 8,434 | 44 | 5 |
| 95 | claude-for-legal | 법률 워크플로우 플러그인 | 8,408 | 95 | 13 |
| 96 | pysheeet | 파이썬 치트시트 | 8,150 | 193 | 1 |
| 97 | open-code-review | 하이브리드 코드 리뷰 도구 | 8,138 | 30 | 1 |
| 98 | financial-services-plugins | 금융 서비스 플러그인 | 7,866 | 88 | 8 |
| 99 | web-access | Claude Code용 웹 브라우징 스킬 | 7,777 | 14 | 1 |
| 100 | awesome-gpt-image-2 | 프롬프트 엔지니어링 템플릿 | 7,761 | 24 | 미표기 |

목록을 보면 단일 플러그인을 담은 저장소가 대다수이지만, 일부 저장소는 플러그인 수가 두드러집니다.
claude-plugins-official은 234개로 가장 많은 플러그인을 담은 공식 디렉터리이며, agents는 84개, claude-skills(337개 스킬 모음)는 78개, knowledge-work-plugins는 68개로 뒤를 잇습니다.
antigravity-awesome-skills는 59개, ruflo는 멀티 에이전트 스웜용 메타 하네스로 35개의 플러그인을 보유합니다.

## 의미와 시사점

이 프로젝트의 가장 큰 의미는 awesome 리스트의 큐레이션을 사람이 아니라 자동화 파이프라인에 맡겼다는 점입니다.
n8n 워크플로우가 GitHub를 지속적으로 스캔하고 지표를 갱신하므로, 목록은 수작업 PR 없이도 최신 채택 현황을 반영합니다.
또한 단순히 스타 수만이 아니라 구독자 수와 플러그인 개수까지 함께 추적해, 생태계 영향력을 다차원으로 본다는 점이 특징입니다.
다만 동일 이름의 저장소가 중복 등장하거나 일부 항목의 Plugins 값이 미표기로 남는 등, 자동 수집에서 비롯된 데이터 정합성의 한계도 함께 드러납니다.
플러그인의 공식 정의나 필터링 기준이 README 본문에 명시되지 않은 점도, 지표를 해석할 때 감안해야 할 부분입니다.

## 결론

awesome-claude-plugins는 n8n 자동화로 23,000개가 넘는 GitHub 저장소를 인덱싱하고, 그중 채택 지표 상위 100개를 추려 순위로 보여 주는 프로젝트입니다.
Stars, Subs, Plugins라는 세 가지 정량 지표를 중심으로 Claude Code 플러그인 생태계의 큰 그림을 한눈에 파악할 수 있게 해 줍니다.
어떤 스킬과 하네스, 메모리 시스템이 실제로 많이 채택되는지 빠르게 살펴보고 싶다면, 이 자동 큐레이션 목록이 좋은 출발점이 됩니다.
추가 자료는 awesomeclaudeplugins.com에서 확인할 수 있습니다.

## Reference

- [awesome-claude-plugins (GitHub)](https://github.com/quemsah/awesome-claude-plugins/)
