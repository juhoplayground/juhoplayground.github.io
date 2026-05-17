---
layout: post
title: "Anthropic가 공개한 Claude for Financial Services: 금융 워크플로우 에이전트·스킬·커넥터 오픈소스"
author: 'Juho'
date: 2026-05-17 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, MCP]
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
2. [Claude for Financial Services란](#claude-for-financial-services란)
   - [두 가지 배포 방식](#두-가지-배포-방식)
   - [저장소 구조](#저장소-구조)
3. [10가지 워크플로우 에이전트](#10가지-워크플로우-에이전트)
4. [버티컬 플러그인과 MCP 데이터 커넥터](#버티컬-플러그인과-mcp-데이터-커넥터)
5. [설치와 사용](#설치와-사용)
6. [의미와 시사점](#의미와-시사점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Anthropic이 금융 서비스 업무 자동화를 위한 레퍼런스 구현체 `claude-for-financial-services`를 오픈소스로 공개했습니다.
사전 구축된 AI 에이전트, 스킬, 데이터 커넥터를 모아 둔 저장소로, 피치 덱 작성부터 어닝 분석, 원장 조정(GL reconciliation), KYC 스크리닝까지 금융권 실무 흐름을 그대로 담았습니다.
핵심 전제는 명확합니다.
모든 산출물은 "사람의 검토를 위한 초안(draft work product)"이며, 에이전트는 투자 자문을 하지 않고 거래를 실행하거나 리스크를 확정하지도 않습니다.
라이선스는 Apache 2.0입니다.

## Claude for Financial Services란

### 두 가지 배포 방식

이 저장소는 두 가지 방식으로 배포할 수 있습니다.

첫째, Claude Cowork 플러그인으로 설치하는 방식입니다.
둘째, Claude Managed Agents API(`/v1/agents`)를 통해 헤드리스로 배포하는 방식입니다.
같은 시스템 프롬프트와 스킬을 공유하면서, 대화형 환경(Cowork)과 자동화 파이프라인(Managed Agents) 양쪽 모두에서 동일한 워크플로우를 돌릴 수 있도록 설계되어 있습니다.

또한 Microsoft 365 애드인 프로비저닝 도구(`claude-for-msft-365-install/`)가 포함되어 있어, Excel·PowerPoint 같은 오피스 환경에서 Claude를 호출할 수 있습니다.

### 저장소 구조

저장소는 크게 플러그인, Managed Agent 쿡북, 설치 도구, 스크립트로 나뉩니다.

```
plugins/
  ├── agent-plugins/           # 10개의 독립 워크플로우 에이전트
  ├── vertical-plugins/        # 금융 버티컬별 스킬 + 커맨드 번들
  │   ├── financial-analysis/  # 핵심 모델링·Excel·덱 QC (필수)
  │   ├── investment-banking/
  │   ├── equity-research/
  │   ├── private-equity/
  │   ├── wealth-management/
  │   ├── fund-admin/
  │   └── operations/
  └── partner-built/           # LSEG, S&P Global 파트너 플러그인
managed-agent-cookbooks/       # 헤드리스 배포 템플릿 (에이전트별)
claude-for-msft-365-install/   # Microsoft 365 애드인 프로비저닝
scripts/
  ├── deploy-managed-agent.sh
  ├── orchestrate.py
  ├── sync-agent-skills.py
  └── check.py
```

설계상 에이전트는 자신의 워크플로우를 시스템 프롬프트와 번들된 스킬로 처음부터 끝까지 소유하고, 버티컬 플러그인은 스킬의 단일 출처(single source of truth) 역할을 합니다.
`sync-agent-skills.py`가 버티컬 스킬을 각 에이전트에 전파하고, `check.py`가 매니페스트를 린트하며 스킬 드리프트를 감지합니다.
빌드 단계 없이 마크다운 스킬과 JSON 매니페스트만으로 동작하는 파일 기반 구조입니다.

## 10가지 워크플로우 에이전트

각 에이전트는 자체 스킬을 번들로 포함하며, 하나의 플러그인으로 설치됩니다.

### 에이전트 목록

| 에이전트 | 기능 | 사용 사례 |
|----------|------|-----------|
| Pitch Agent | 비교기업·선례거래·LBO 분석을 브랜드 피치 덱으로 | M&A 자문 |
| Meeting Prep Agent | 미팅 전 브리핑 패키지 작성 | 고객 응대 |
| Market Researcher | 섹터 개요, 경쟁 환경, 동종업체 비교 | 리서치·스크리닝 |
| Earnings Reviewer | 어닝 콜 + 공시 분석 후 모델 업데이트 및 노트 | 주식 리서치 |
| Model Builder | DCF, LBO, 3-statement, 비교기업 모델을 Excel로 | 재무 모델링 |
| Valuation Reviewer | GP 패키지를 밸류에이션 템플릿으로 정리해 LP 리포팅 | 펀드 관리 |
| GL Reconciler | 차이(break) 탐지, 근본 원인 추적, 승인 라우팅 | 재무 운영 |
| Month-End Closer | 발생주의 처리, 롤포워드, 분산 코멘트 작성 | 펀드 회계 |
| Statement Auditor | LP 명세서 배포 전 감사 | 펀드 관리 |
| KYC Screener | 문서 파싱, 룰 엔진 실행, 누락 항목 플래깅 | 온보딩 |

## 버티컬 플러그인과 MCP 데이터 커넥터

핵심 플러그인인 `financial-analysis`에는 비교기업 분석, DCF·LBO·3-statement 모델, Excel 감사, 경쟁 분석, 덱 리프레시, PPTX·XLSX 작성 등 스킬과 `/comps`, `/dcf`, `/lbo`, `/3-statement-model`, `/debug-model` 등 슬래시 커맨드가 포함됩니다.
버티컬별로는 투자은행(CIM·티저·바이어 리스트·머저 모델), 주식 리서치(어닝 분석·커버리지 개시·테제 추적), 사모펀드(소싱·스크리닝·DD 체크리스트·IC 메모), 자산관리(고객 리뷰·재무 플랜·리밸런싱·택스 로스 하베스팅), 펀드 관리(GL 조정·NAV 타이아웃), 운영(KYC 문서 파싱) 스킬이 추가됩니다.

특히 11개 금융 데이터 제공업체의 MCP 커넥터를 `financial-analysis` 플러그인의 `.mcp.json`에 중앙 집중식으로 관리합니다.

### MCP 데이터 커넥터

| 제공업체 | 설명 |
|----------|------|
| Daloopa | 재무 데이터 |
| Morningstar | 투자 리서치 데이터 |
| S&P Global (Kensho) | 시장·기업 데이터 |
| FactSet | 금융 데이터 |
| Moody's | GenAI-ready 데이터 |
| MT Newswires | 뉴스와이어 |
| Aiera | 어닝·이벤트 데이터 |
| LSEG | 분석 데이터 |
| PitchBook | 사모·벤처 데이터 |
| Chronograph | PE 포트폴리오 데이터 |
| Egnyte | 문서 관리 |

기업 환경에 맞게 `.mcp.json`의 커넥터를 교체하거나, 스킬 파일에 회사 고유 용어와 프로세스를 추가하거나, `/ppt-template`로 브랜드 레이아웃을 학습시키는 식의 커스터마이징이 가능합니다.

## 설치와 사용

Claude Code CLI에서는 마켓플레이스를 추가한 뒤 플러그인을 설치합니다.

```bash
# 마켓플레이스 추가
claude plugin marketplace add anthropics/claude-for-financial-services

# 핵심 스킬 + 커넥터 (먼저 설치)
claude plugin install financial-analysis@claude-for-financial-services

# 명명된 에이전트
claude plugin install pitch-agent@claude-for-financial-services
claude plugin install gl-reconciler@claude-for-financial-services
claude plugin install market-researcher@claude-for-financial-services

# 버티컬 번들
claude plugin install investment-banking@claude-for-financial-services
claude plugin install equity-research@claude-for-financial-services
```

Managed Agents API로 헤드리스 배포할 때는 다음과 같이 합니다.

```bash
export ANTHROPIC_API_KEY=sk-ant-...
scripts/deploy-managed-agent.sh gl-reconciler
```

이 경우 공유 시스템 프롬프트(`agent.yaml`), depth-1 서브에이전트(`callable_agents`), 오케스트레이션 이벤트 루프(`scripts/orchestrate.py`)와 함께 `/v1/agents`에 배포됩니다.
서브에이전트 위임은 아직 프리뷰 단계 기능이고, 에이전트 간 핸드오프에는 steering 이벤트가 쓰입니다.
Cowork에서는 설정 → 플러그인 → 플러그인 추가에서 저장소 URL을 붙여넣거나 `plugins/` 디렉터리 zip을 업로드하면 됩니다.

## 의미와 시사점

이 저장소는 단순한 데모가 아니라 "에이전트가 워크플로우를 끝까지 소유한다"는 패턴을 금융 도메인에 구체화한 사례입니다.
버티컬 스킬을 단일 출처로 두고 에이전트가 복사본을 동기화하는 구조, MCP 커넥터를 한곳에서 공유하는 구조, Cowork 플러그인과 Managed Agent 쿡북이 동일한 시스템 프롬프트·스킬을 참조하는 구조는 다른 도메인의 에이전트 시스템을 설계할 때도 참고할 만합니다.
또한 출력물을 명시적으로 "사람 검토용 초안"으로 못박고 거래 실행·리스크 확정을 배제한 점은, 규제 산업에서 LLM 에이전트를 어떻게 책임 범위 안에 가둘지에 대한 실용적 선택을 보여줍니다.

## 결론

`claude-for-financial-services`는 10개의 워크플로우 에이전트, 7개 버티컬 플러그인, 11개 금융 데이터 MCP 커넥터, Microsoft 365 애드인 도구를 묶은 Apache 2.0 오픈소스입니다.
Claude Code CLI 플러그인이나 Managed Agents API 어느 쪽으로도 배포할 수 있으며, 파일 기반 구조라 회사 고유 용어·프로세스·브랜드 템플릿을 얹어 확장하기 쉽습니다.
금융권에서 LLM 에이전트 도입을 검토 중이라면 출발점으로 살펴볼 가치가 있습니다.

## Reference

- [anthropics/financial-services (GitHub)](https://github.com/anthropics/financial-services)
