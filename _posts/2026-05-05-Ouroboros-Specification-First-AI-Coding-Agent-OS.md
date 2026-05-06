---
layout: post
title: "Ouroboros - 한국 개발자가 만든 Specification-First AI 코딩 Agent OS"
author: 'Juho'
date: 2026-05-05 00:00:00 +0900
categories: [VibeCoding]
tags: [Agent, AI, VibeCoding, Skill, Prompt, MCP]
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
   - [Ouroboros Cycle - 4단계 루프](#ouroboros-cycle---4단계-루프)
   - [모호성 점수와 인터뷰 게이트](#모호성-점수와-인터뷰-게이트)
   - [Ralph 진화 루프와 온톨로지 수렴](#ralph-진화-루프와-온톨로지-수렴)
   - [3단계 검증 게이트](#3단계-검증-게이트)
   - [아키텍처와 런타임 통합](#아키텍처와-런타임-통합)
   - [설치와 사용 예시](#설치와-사용-예시)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

한국 개발자가 만든 오픈소스 프로젝트 [Ouroboros](https://github.com/Q00/ouroboros){:target="_blank"}가 AI-assisted discrete-event simulation 벤치마크에서 Claude의 Plan Mode를 능가하며 1위를 기록했다.
프로젝트의 핵심 철학은 "Stop prompting. Start specifying"이다.
모호한 프롬프트 대신 명확한 사양을 먼저 정의하는 specification-first 워크플로우를 자동화한 Agent OS다.
GitHub 저장소는 3.1k 스타와 77개 릴리스를 보유하고 있으며 v0.32.0이 최신이다.

## 배경

전통적인 AI 코딩은 모호한 입력을 AI가 추측하면서 재작업이 늘어나고, 스펙이 부재한 상태에서 아키텍처가 표류하며, "괜찮아 보임" 수준의 수동 QA에 의존한다는 문제가 있다.
Ouroboros는 이 세 가지 문제를 각각 소크라테스식 인터뷰로 명확화하고, 불변 사양으로 의도를 고정하며, 3단계 자동 검증 게이트로 해결하겠다는 접근을 취한다.

광산 운송 시스템을 대상으로 한 고난도 과제에서 단순 코딩 능력을 넘어 시스템 이해, 모델링, 실행 가능한 시뮬레이션 결과까지 완성했다.
MCP 서버 실패 시에도 기술 기반 접근으로 성공적으로 복구하며, 구조화된 워크플로우인 문제 정의 → 계획 → 실행 → 평가 → 복구가 단순한 지침보다 우수하다는 결과를 보여줬다.
애니메이션과 topology diagram까지 포함한 고품질 산출물을 제공한 점이 차별점으로 평가된다.

## 핵심 내용

### Ouroboros Cycle - 4단계 루프

Ouroboros의 기본 루프는 인터뷰, 시드, 실행, 평가의 4단계로 구성된다.
이 루프는 진화적 피드백을 통해 반복된다.

```
Interview → Seed → Execute → Evaluate
    ↑                          ↓
    ←──── Evolutionary Loop ────
```

| 단계 | 역할 | 주요 산출물 |
|------|------|-------------|
| Interview | 숨겨진 가정 12개 이상 노출 | 모호성 점수 0.0~1.0 |
| Seed | 인터뷰 답변을 불변 사양으로 결정화 | 수용 기준, 온톨로지, 제약사항 |
| Execute | Double Diamond 분해법으로 구현 | 계층적 AC 분해 결과 |
| Evaluate | 3단계 검증 수행 | Mechanical, Semantic, Multi-Model Consensus |

### 모호성 점수와 인터뷰 게이트

Interview 단계에서는 모호성 점수가 0.2 이하여야 다음 단계로 진행할 수 있다.
모호성은 다음 공식으로 계산된다.

```
Ambiguity = 1 - Σ(clarity_i × weight_i)
```

greenfield 프로젝트의 경우 가중치는 다음과 같이 배정된다.

| 차원 | 가중치 | 설명 |
|------|--------|------|
| 목표 명확성 | 40% | 목표가 구체적인가 |
| 제약 명확성 | 30% | 한계가 정의되었는가 |
| 성공 기준 | 30% | 결과가 측정 가능한가 |

예시 계산을 보면 Goal 0.9 × 0.4 = 0.36, Constraint 0.8 × 0.3 = 0.24, Success 0.7 × 0.3 = 0.21로 합계 0.81이 된다.
따라서 Ambiguity는 0.19로 0.2 이하 게이트를 통과한다.

### Ralph 진화 루프와 온톨로지 수렴

Ralph는 지속적인 진화 루프를 의미하며, 세대를 거치며 온톨로지가 수렴할 때까지 반복된다.

```
Ralph Cycle 1 → Gen 1 → CONTINUE
Ralph Cycle 2 → Gen 2 → CONTINUE
Ralph Cycle 3 → Gen 3 → CONVERGED (본체 정지)
```

온톨로지 유사도는 다음 공식으로 계산된다.

```
Similarity = 0.5×이름_겹침 + 0.3×타입_일치 + 0.2×정확_일치
```

3세대 연속으로 Similarity가 0.95 이상일 때 수렴으로 판정하고 본체 루프를 정지시킨다.

### 3단계 검증 게이트

Evaluate 단계는 비용 효율을 고려한 단계적 검증을 수행한다.
무료에 가까운 단계부터 시작해 필요할 때만 비싼 검증으로 넘어간다.

| 단계 | 비용 | 역할 |
|------|------|------|
| Mechanical | 무료 | 정적 분석, 컴파일, 테스트 실행 |
| Semantic | 중간 | 의미적 일치성 검증 |
| Multi-Model Consensus | 높음 | 여러 모델의 합의 기반 평가 |

### 아키텍처와 런타임 통합

Ouroboros는 모듈화된 디렉토리 구조를 갖는다.

```
src/ouroboros/
├── bigbang/        인터뷰, 모호성 점수, 기존 코드 탐색
├── routing/        PAL Router (1x/10x/30x 비용 최적화)
├── execution/      Double Diamond, AC 분해
├── evaluation/     3단계 검증 게이트
├── evolution/      Wonder/Reflect 사이클, 수렴 감지
├── resilience/     정체 감지, 5가지 측면 사고 페르소나
├── observability/  표류 측정, 자동 회고
├── persistence/    이벤트 소싱 (SQLAlchemy + aiosqlite)
├── orchestrator/   런타임 추상화 (Claude Code, Codex, OpenCode, Hermes)
├── mcp/            MCP 클라이언트/서버 통합
└── cli/            Typer 기반 CLI
```

기술 스택은 Python 97.3%, Rust 1.9%로 구성되며 LiteLLM을 통해 100개 이상의 모델을 지원한다.
런타임은 Claude Code, Codex CLI, OpenCode, Hermes를 모두 지원하며 MCP 통합을 통해 외부 도구와 연결된다.
이벤트 소싱 기반의 SQLAlchemy + aiosqlite 영속화 계층이 재생 가능한 실행을 보장한다.

### 설치와 사용 예시

단일 명령어로 설치할 수 있다.

```bash
curl -fsSL https://raw.githubusercontent.com/Q00/ouroboros/main/scripts/install.sh | bash
```

pip 기반 선택형 설치도 제공한다.

```bash
pip install ouroboros-ai                    # 기본
pip install ouroboros-ai[claude]            # Claude Code 지원
pip install ouroboros-ai[litellm]           # 100+ 모델 지원
pip install ouroboros-ai[all]               # 전체
ouroboros setup                             # 런타임 구성
```

Claude Code 플러그인 마켓플레이스를 통한 등록도 지원된다.

```bash
claude plugin marketplace add Q00/ouroboros
```

CLI 명령은 `ooo`라는 짧은 alias로 제공된다.

```bash
# 인터뷰 시작
ooo interview "I want to build a task management CLI"

# CLI 직접 사용
ouroboros init start
ouroboros run seed.yaml
ouroboros status executions
```

핵심 명령어는 다음과 같이 정리된다.

| 기능 | 명령어 | 설명 |
|------|--------|------|
| 설정 | ooo setup | 런타임 등록 |
| 인터뷰 | ooo interview | 가정 노출 |
| 실행 | ooo run | Double Diamond 실행 |
| 평가 | ooo evaluate | 3단계 검증 |
| 진화 | ooo evolve | 진화 루프 시작 |
| 진행률 | ooo status | 세션 추적 |
| 도움말 | ooo help | 전체 참조 |

요구사항은 Python 3.12 이상이다.
제거는 `ouroboros uninstall` 한 줄로 모든 설정과 MCP 등록, 데이터를 제거한다.

## 의미와 시사점

Ouroboros는 단순한 프롬프트 입력에 의존하던 AI 코딩 에이전트의 한계를 specification-first 접근으로 극복하려 한다.
모호성 점수, 온톨로지 유사도, 3단계 검증 게이트라는 수학 기반 게이팅이 그 핵심이다.
이는 단순히 프롬프트를 잘 쓰는 vibe coding이 아니라 측정 가능한 명확성을 강제한다는 점에서 차별화된다.

여러 런타임을 추상화한 orchestrator 계층은 Claude Code, Codex CLI, OpenCode, Hermes를 모두 통합한다.
MCP 클라이언트와 서버를 모두 지원해 외부 도구와의 양방향 통합도 가능하다.
이벤트 소싱 기반 영속화는 실행을 재생 가능하게 만들어 디버깅과 회고를 자동화할 수 있는 토대를 제공한다.

광산 운송 시뮬레이션 같은 도메인 특화 과제에서 Plan Mode를 능가했다는 점은 단순한 코드 생성을 넘어 시스템 모델링까지 가능함을 보여준다.
9가지 전문화된 에이전트 페르소나와 5가지 측면 사고 페르소나, Wonder/Reflect 사이클은 단일 모델의 한계를 다중 페르소나 협업으로 보완하려는 시도다.

## 결론

Ouroboros는 AI 코딩의 모호함을 수치화하고 게이팅하는 specification-first 워크플로우를 제안한다.
인터뷰, 시드, 실행, 평가의 4단계 루프와 Ralph 진화 루프, 3단계 검증 게이트가 핵심 메커니즘이다.
한국 개발자가 만든 오픈소스가 글로벌 벤치마크에서 Plan Mode를 능가했다는 점에서 의미가 있다.

## Reference

- [Q00/ouroboros - GitHub](https://github.com/Q00/ouroboros/)
