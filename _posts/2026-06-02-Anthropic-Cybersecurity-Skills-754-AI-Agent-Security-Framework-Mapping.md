---
layout: post
title: "Anthropic Cybersecurity Skills - AI 에이전트용 754개 사이버보안 스킬 저장소"
author: 'Juho'
date: 2026-06-02 00:00:00 +0900
categories: [MCP]
tags: [Skill, Skill Development, Security, Agent]
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
2. [스킬 구성](#스킬-구성)
3. [프레임워크 매핑](#프레임워크-매핑)
4. [핵심 도메인](#핵심-도메인)
5. [사용 방법](#사용-방법)
   - [설치](#설치)
   - [호환 플랫폼](#호환-플랫폼)
6. [스킬 구조](#스킬-구조)
7. [라이선스와 커뮤니티](#라이선스와-커뮤니티)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

mukul975가 공개한 Anthropic-Cybersecurity-Skills는 AI 에이전트가 전문가 수준으로 보안 작업을 수행할 수 있도록 설계된 754개 사이버보안 스킬 저장소다.
일반 웹 검색에 의존하는 대신, 도메인별로 잘 구조화된 워크플로에 에이전트가 접근할 수 있도록 한다.
agentskills.io 표준을 따르며, Claude Code, GitHub Copilot, Cursor, Gemini CLI 등 다양한 에이전트 플랫폼에서 활용 가능하다.

## 스킬 구성

- 26개 보안 도메인 — 클라우드 보안부터 디셉션 기술까지
- 754개 프로덕션 등급 스킬 — agentskills.io 표준 준수
- 각 스킬은 YAML frontmatter(에이전트 발견용)와 Markdown 본문(단계별 실행)으로 구성

규모가 754개라는 점은 단순한 데모가 아니라, 실제 SOC/Red Team 워크플로를 폭넓게 커버하려는 의도를 보여준다.

## 프레임워크 매핑

이 저장소의 차별점은 다섯 가지 주요 보안 프레임워크에 스킬을 매핑한다는 점이다.

| 프레임워크 | 커버리지 | 목적 |
|------------|----------|------|
| MITRE ATT&CK v18 | 14 tactics, 200+ techniques | 적대적 TTP |
| NIST CSF 2.0 | 6 functions, 22 categories | 조직 보안 자세 |
| MITRE ATLAS v5.4 | 16 tactics, 84 techniques | AI/ML 위협 |
| MITRE D3FEND v1.3 | 7 categories, 267 techniques | 방어 대응 기술 |
| NIST AI RMF 1.0 | 4 functions, 72 subcategories | AI 리스크 관리 |

특히 ATLAS와 AI RMF가 포함되어 AI 모델 자체에 대한 위협까지 다룬다는 점이 최근 흐름에 부합한다.

## 핵심 도메인

스킬은 26개 도메인에 분포해 있으며, 주요 영역은 다음과 같다.

| 도메인 | 스킬 수 |
|--------|---------|
| Threat Hunting | 55 |
| Threat Intelligence | 50 |
| Web Application Security | 42 |
| Network Security | 40 |
| Malware Analysis | 39 |
| Digital Forensics | 37 |

위협 헌팅과 위협 인텔리전스가 가장 비중이 크다는 점은, 에이전트가 실제로 가장 많이 도움 줄 수 있는 영역이 어디인지를 가늠하게 한다.

## 사용 방법

### 설치

```bash
npx skills add mukul975/Anthropic-Cybersecurity-Skills
```

또는 직접 git clone으로 받을 수 있다.

```bash
git clone https://github.com/mukul975/Anthropic-Cybersecurity-Skills.git
```

npx 방식이 agentskills.io 표준의 권장 흐름이다.

### 호환 플랫폼

Claude Code, GitHub Copilot, Cursor, Gemini CLI를 비롯한 20개 이상의 agentskills.io 호환 플랫폼에서 작동한다.
즉 한 번 작성된 스킬이 여러 코딩/에이전트 도구에서 재사용된다.

## 스킬 구조

각 스킬은 동일한 구조를 따른다.

| 섹션 | 역할 |
|------|------|
| YAML frontmatter | 메타데이터와 프레임워크 매핑 |
| When to Use | 트리거 조건 |
| Prerequisites | 도구 요구 사항과 접근 권한 |
| Workflow | 단계별 절차 |
| Verification | 성공 확인 방법 |
| Reference files | 보조 스크립트와 자료 |

When to Use가 명확하면 에이전트가 스킬을 자율적으로 선택할 수 있고, Verification이 포함되어 있어 단순 행동 가이드가 아니라 검증 가능한 워크플로가 된다.

## 라이선스와 커뮤니티

| 항목 | 값 |
|------|-----|
| 라이선스 | Apache 2.0 |
| 인용 | CITATION.cff 제공 |
| 기여 채널 | GitHub Discussions, Issues, Security Policy |
| 부속 자료 | Casky.ai의 인터랙티브 플레이그라운드, GARS-2026 조사 참여 |
| PR 리뷰 SLA | 48시간 내 기술 리뷰 |

저장소는 Anthropic PBC와 제휴 관계가 아닌 독립 커뮤니티 프로젝트라는 점이 명시되어 있다.

## 결론

Anthropic-Cybersecurity-Skills는 보안 워크플로를 에이전트가 다룰 수 있는 단위로 모듈화한 대표 사례다.
ATT&CK, D3FEND, NIST CSF/AI RMF에 매핑된 754개의 스킬은, SOC와 Red Team 워크플로를 AI 에이전트로 확장하려는 조직에 즉시 도움이 된다.
Apache 2.0 라이선스라 사내 보안 자동화 파이프라인에 포크/확장하기에도 부담이 없다.

## Reference

- [Anthropic-Cybersecurity-Skills GitHub](https://github.com/mukul975/Anthropic-Cybersecurity-Skills)
