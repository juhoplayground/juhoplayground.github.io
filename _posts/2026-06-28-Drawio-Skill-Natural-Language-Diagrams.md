---
layout: post
title: "drawio-skill: 자연어로 draw.io 다이어그램을 생성하는 스킬"
author: 'Juho'
date: 2026-06-28 00:00:00 +0900
categories: [AI]
tags: [Skill, AI, Agent]
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
2. [핵심 기능](#핵심-기능)
   - [자연어에서 다이어그램으로](#자연어에서-다이어그램으로)
   - [자가검증과 자동수정](#자가검증과-자동수정)
3. [셰이프와 아이콘 라이브러리](#셰이프와-아이콘-라이브러리)
4. [6개 다이어그램 프리셋](#6개-다이어그램-프리셋)
5. [코드베이스 시각화](#코드베이스-시각화)
6. [설치](#설치)
   - [draw.io 데스크톱 CLI 설치](#drawio-데스크톱-cli-설치)
   - [스킬 설치](#스킬-설치)
7. [사용 예시와 주요 명령어](#사용-예시와-주요-명령어)
8. [추가 기능](#추가-기능)
9. [결론](#결론)
10. [Reference](#reference)

## 개요

drawio-skill은 자연어 설명을 전문적인 draw.io 다이어그램 파일로 변환하는 스킬입니다.
사용자가 만들고 싶은 다이어그램을 대화하듯 설명하면, 스킬이 레이아웃을 계획하고 XML을 생성하며 초안을 내보낸 뒤 스스로 점검합니다.
이 프로젝트는 "자신의 출력을 직접 읽고, 보여주기 전에 자동으로 고치는 유일한 순수 SKILL.md 솔루션"으로 소개됩니다.

생성된 파일은 PNG, SVG, PDF, JPG 형식으로 내보낼 수 있습니다.
Claude Code, Cursor, Copilot 등 Agent Skills 형식을 지원하는 여러 플랫폼에서 동작하는 에이전트 호환 스킬입니다.
별도의 MCP 서버나 백그라운드 데몬 없이 draw.io 데스크톱 CLI만 있으면 사용할 수 있습니다.
라이선스는 MIT입니다.

## 핵심 기능

### 자연어에서 다이어그램으로

사용자는 원하는 다이어그램을 대화형으로 설명하면 됩니다.
스킬은 레이아웃을 계획하고 XML을 생성한 뒤 초안을 내보내고, 자가점검을 수행하며, 반복적인 개선을 지원합니다.
피드백은 최대 5라운드까지 반영할 수 있어 점진적으로 다이어그램을 다듬을 수 있습니다.

### 자가검증과 자동수정

drawio-skill의 가장 큰 특징은 자신이 생성한 PNG 출력을 직접 읽고 문제를 자동으로 교정한다는 점입니다.
이 자가검증은 사용자 검토 전에 최대 2라운드의 개선 과정을 거칩니다.
스킬이 탐지하고 자동으로 수정하는 6가지 이슈 유형은 다음과 같습니다.

| 이슈 유형 | 설명 |
|-----------|------|
| Overlapping shapes | 요소가 서로 겹치는 문제 |
| Clipped labels | 라벨이 잘려 보이는 문제 |
| Stacked edges | 간선이 겹쳐 쌓이는 문제 |
| Layout inconsistencies | 레이아웃이 일관되지 않은 문제 |
| Connector routing problems | 연결선 경로 설정 문제 |
| Structural validation errors | 구조적 검증 오류 |

## 셰이프와 아이콘 라이브러리

drawio-skill은 방대한 셰이프와 브랜드 로고 라이브러리를 제공합니다.
잘못된 셰이프 지정으로 인한 빈 박스 대체 문제를 없애고, 정확한 공식 아이콘을 찾아 사용합니다.

| 라이브러리 | 내용 |
|------------|------|
| 공식 셰이프 | 10000개 이상의 AWS, Azure, GCP, Cisco, Kubernetes, UML, BPMN, P&ID 셰이프와 셰이프 검색 기능 |
| AI/LLM 브랜드 로고 | 표준 draw.io에 없는 321개 로고 (OpenAI, Claude, Gemini, Mistral, Llama, Cohere, DeepSeek, Qwen, Ollama, LangChain, HuggingFace 등) |

브랜드 로고는 MIT 라이선스의 lobe-icons에서 선별되었습니다.
아이콘은 기본적으로 unpkg CDN을 참조하며, 오프라인 사용을 위해 data URI로 임베드할 수도 있습니다.

## 6개 다이어그램 프리셋

drawio-skill은 자주 쓰이는 다이어그램 유형에 대해 6개의 프리셋을 제공합니다.

| 프리셋 | 설명 |
|--------|------|
| ERD | 테이블 컨테이너와 PK/FK 표기를 사용하는 데이터베이스 스키마 (개체-관계 다이어그램) |
| UML Class | 상속, 컴포지션, 애그리게이션 관계를 적절한 화살표 유형으로 표현하는 클래스 다이어그램 |
| Sequence | 라이프라인과 활성화 박스로 상호작용 흐름을 나타내는 시퀀스 다이어그램 |
| Architecture | 티어 기반 스윔레인으로 마이크로서비스와 클라우드 인프라를 표현하는 아키텍처 다이어그램 |
| ML/Deep Learning | 텐서 셰이프 주석(B, C, H, W)과 레이어 유형별 색상 구분을 갖춘 신경망 다이어그램 |
| Flowchart | 평행사변형 입출력, 다이아몬드 결정 등 의미 기반 셰이프로 비즈니스 프로세스와 상태 머신을 표현하는 순서도 |

## 코드베이스 시각화

drawio-skill은 코드 구조를 추출하여 자동 레이아웃된 구조 다이어그램으로 변환할 수 있습니다.
Python, JavaScript/TypeScript, Go, Rust 프로젝트를 대상으로 합니다.

추출기는 총 5종으로, Python, JavaScript/TypeScript, Go, Rust의 임포트 그래프와 Python 클래스 상속 계층을 추출합니다.
자동 레이아웃 파이프라인은 다음 단계를 포함합니다.

| 단계 | 설명 |
|------|------|
| Graphviz 배치 | 노드 주변을 우회하는 직교(orthogonal) 간선 라우팅으로 배치 |
| Transitive reduction | 함축된 간선을 제거 (예시: asyncio가 149개에서 46개 간선으로 축소) |
| 중첩 컨테이너 | 서브패키지 단위로 모듈을 박스로 묶음 |
| 결정론적 검증 | 매달린 간선, 중복 ID, 겹침에 대한 린팅 검증 |

모듈 단위 색상 코딩으로 구조를 한눈에 파악할 수 있도록 돕습니다.

## 설치

설치는 draw.io 데스크톱 CLI 설치와 스킬 설치의 두 단계로 진행됩니다.

### draw.io 데스크톱 CLI 설치

운영체제별 설치 방법은 다음과 같습니다.

| 운영체제 | 방법 |
|----------|------|
| macOS | brew install --cask drawio |
| Windows | GitHub 릴리스에서 인스톨러 다운로드 |
| Linux | 릴리스의 .deb/.rpm 사용, 헤드리스 환경은 sudo apt install xvfb |

설치 후 다음 명령으로 확인합니다.

```bash
drawio --version
```

### 스킬 설치

스킬 설치는 세 가지 방법을 제공합니다.

가장 권장되는 방법은 플러그인 마켓플레이스를 이용하는 것입니다.

```bash
/plugin marketplace add Agents365-ai/365-skills
/plugin install drawio
```

모든 에이전트에서 사용할 수 있는 npx 설치 방법은 다음과 같습니다.

```bash
npx skills add Agents365-ai/365-skills -g
```

수동 설치는 저장소를 직접 클론합니다.
전역 설치는 홈 디렉터리의 스킬 경로에, 프로젝트 단위 설치는 프로젝트의 스킬 경로에 클론합니다.

```bash
git clone https://github.com/Agents365-ai/drawio-skill ~/.claude/skills/drawio-skill
```

Opencode, ClawHub, SkillsMP, Codex 등 플랫폼별 경로도 지원합니다.
업데이트는 /plugin update drawio 명령이나 git pull로 수행합니다.

## 사용 예시와 주요 명령어

자연어 프롬프트의 예시는 다음과 같습니다.
"Mobile/Web/Admin 클라이언트, API Gateway(인증 + 요청 제한 + 라우팅), Auth/User/Order/Product/Payment 서비스, Kafka 메시지 큐, Notification 서비스, 그리고 User DB / Order DB / Product DB / Redis Cache / Stripe API로 구성된 마이크로서비스 이커머스 아키텍처를 만들어줘"

그 결과로 노드 겹침을 피하는 깔끔한 간선 라우팅과 그리드 정렬을 갖춘 전문적인 다이어그램이 생성됩니다.

주요 명령어와 스크립트는 다음과 같습니다.

| 명령어 | 역할 |
|--------|------|
| shapesearch.py "aws lambda" | 공식 셰이프 스타일을 검색하고 해석 |
| aiicons.py "claude" --json 또는 --embed | 브랜드 로고를 해석 |
| autolayout.py graph.json -o diagram.drawio | 코드 구조를 자동 배치 |
| validate.py | .drawio 파일을 결정론적으로 린팅 |

코드베이스 시각화 추출 명령으로는 pyimports.py, jsimports.py, goimports.py, rustimports.py, pyclasses.py, autolayout.py, validate.py가 제공됩니다.

## 추가 기능

drawio-skill은 다음과 같은 부가 기능을 제공합니다.

| 기능 | 설명 |
|------|------|
| 애니메이션 커넥터 | flowAnimation=1 파라미터로 연결선 애니메이션 적용 |
| 스타일 프리셋 | default, corporate, handdrawn 3종 내장, 기존 파일이나 이미지에서 학습한 사용자 정의 프리셋도 지원 |
| 그리드 정렬 레이아웃 | 10px 스냅 간격, 다이어그램 크기에 따라 간격이 조정되며 전용 통로를 갖춘 직교 라우팅 |
| 내보내기 | 네이티브 draw.io CLI를 통한 PNG, SVG, PDF, JPG 출력 |
| 브라우저 폴백 | diagrams.net URL을 생성하는 폴백 지원 |
| 멀티 에이전트 지원 | Claude Code, Cursor, Copilot, OpenClaw, Codex, Hermes 등 Agent Skills 호환 플랫폼 |
| 제로 설정 배포 | 단일 SKILL.md 파일로 동작, Node.js 없이도 사용 가능 (선택적 npx 설치 시에만 필요) |

## 결론

drawio-skill은 자연어 설명만으로 전문적인 draw.io 다이어그램을 생성하고, 자신의 출력을 직접 읽어 6가지 이슈를 자동으로 수정하는 스킬입니다.
10000개 이상의 공식 셰이프와 321개의 AI/LLM 브랜드 로고, 6개의 다이어그램 프리셋, 코드베이스 시각화 기능을 갖추고 있습니다.
MCP 서버나 데몬 없이 draw.io 데스크톱 CLI만으로 동작하며, 단일 SKILL.md 파일로 여러 에이전트 플랫폼에서 사용할 수 있다는 점이 특징입니다.
다이어그램 작성을 자동화하고 싶은 개발자에게 실용적인 선택지가 될 수 있습니다.

## Reference

- [drawio-skill 공식 페이지](https://agents365-ai.github.io/drawio-skill/)
- [Agents365-ai/drawio-skill](https://github.com/Agents365-ai/drawio-skill)
