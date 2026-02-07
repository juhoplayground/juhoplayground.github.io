---
layout: post
title: Claude Cowork Legal Plugin - 법률 업무 자동화와 시장 충격
author: 'Juho'
date: 2026-02-07 00:00:00 +0900
categories: [Claude]
tags: [Claude, AI, MCP, Agent]
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
2. [Claude Cowork 플러그인 생태계](#claude-cowork-플러그인-생태계)
3. [플러그인 아키텍처](#플러그인-아키텍처)
4. [Legal 플러그인 상세](#legal-플러그인-상세)
5. [11개 오픈소스 플러그인](#11개-오픈소스-플러그인)
6. [설치 및 사용법](#설치-및-사용법)
7. [커스터마이징](#커스터마이징)
8. [시장 영향](#시장-영향)
9. [시사점](#시사점)
10. [Reference](#reference)

## 개요

Anthropic이 2026년 1월 30일, Claude Cowork에 플러그인 시스템을 리서치 프리뷰로 공개했다.
플러그인은 도구, 스킬, 통합을 번들로 묶어 원클릭 설치를 지원하는 모듈형 확장 기능이다.
11개 오픈소스 플러그인이 함께 공개되었으며, 그중 Legal 플러그인은 계약 검토, NDA 분류, 컴플라이언스 워크플로를 자동화한다.

이 발표는 단순한 기능 추가를 넘어 시장에 큰 파장을 일으켰다.
Thomson Reuters 주가가 급락하고 Nasdaq이 1.4% 하락하는 등 "SaaSpocalypse"라 불리는 시장 충격이 발생했다.

## Claude Cowork 플러그인 생태계

### 플러그인이란

플러그인은 Claude를 특정 직무의 전문가로 특화시키는 확장 도구이다.
스킬, 커넥터, 슬래시 명령어, 서브 에이전트를 번들로 묶어 역할별 맞춤 워크플로를 구성한다.

### 이용 가능 대상

Pro 및 Max 플랜 사용자에게 제공된다.
Claude Cowork 앱 내에서 직접 설치하거나 Claude Code CLI에서 설치할 수 있다.
현재 플러그인은 로컬에 저장되며, 조직 단위 관리 기능은 추후 제공 예정이다.

### 인기 플러그인 현황

| 플러그인 | 설치 수 | Anthropic 인증 |
|----------|---------|---------------|
| Frontend Design | 141,226 | O |
| Context7 | 93,538 | X |
| Code Review | 68,604 | O |
| GitHub | 65,584 | X |
| Feature Dev | 63,312 | O |
| Code Simplifier | 50,966 | O |
| Ralph Loop | 49,856 | O |
| TypeScript LSP | 48,026 | O |
| Playwright | 43,949 | X |
| Commit Commands | 42,189 | O |

## 플러그인 아키텍처

### 디렉토리 구조

플러그인은 마크다운과 JSON 기반의 선언적 구조를 따른다.
코드 컴파일이나 배포 인프라 없이 파일 편집만으로 구성할 수 있다.

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json      # 매니페스트 (메타데이터, 버전, 설명)
├── .mcp.json             # MCP 서버 연결 및 도구 정의
├── commands/             # 슬래시 명령어 (*.md)
│   ├── command-1.md
│   └── command-2.md
└── skills/               # 도메인 전문 지식 (*.md)
    ├── skill-1.md
    └── skill-2.md
```

### 4대 핵심 구성 요소

| 구성 요소 | 파일 | 역할 |
|-----------|------|------|
| Manifest | plugin.json | 플러그인 ID, 버전, 설명, 명령어 및 스킬 목록 정의 |
| Connectors | .mcp.json | MCP 프로토콜을 통한 외부 도구 연결 (CRM, DB 등) |
| Commands | commands/*.md | 사용자가 명시적으로 호출하는 슬래시 명령어 |
| Skills | skills/*.md | Claude가 자동으로 활용하는 도메인 전문 지식 |

### 동작 방식

스킬은 자동 활성화된다.
Claude는 skills 디렉토리의 마크다운 파일을 읽고, 대화 맥락에 관련된 스킬을 자동으로 적용한다.
사용자가 명시적으로 호출할 필요가 없다.

명령어는 사용자가 직접 실행한다.
`/legal:review-contract`와 같이 슬래시 명령어를 입력하면, commands 디렉토리의 해당 정의 파일을 참조하여 워크플로를 실행한다.

외부 도구 연결은 MCP를 통해 이루어진다.
`.mcp.json`에 정의된 MCP 서버가 외부 도구의 기능을 노출하고, Claude는 스킬과 명령어 정의에 따라 도구를 호출한다.

## Legal 플러그인 상세

### 대상 사용자

- 상업 법무 (Commercial Counsel)
- 제품 법무 (Product Counsel)
- 프라이버시/컴플라이언스 팀
- 소송 지원 팀

### 슬래시 명령어

| 명령어 | 기능 |
|--------|------|
| /review-contract | 설정된 협상 플레이북 기준으로 조항별 계약 검토, GREEN/YELLOW/RED 분류 및 수정 제안 |
| /triage-nda | NDA 사전 심사 - 표준 승인, 법무 검토, 전체 검토로 분류 |
| /vendor-check | 벤더 계약 상태 확인 |
| /brief | 맥락에 맞는 브리핑 생성 (일일 요약, 주제 리서치, 인시던트 대응) |
| /respond | 일반적 문의에 대한 템플릿 응답 작성 (정보주체 요청, 디스커버리 홀드 등) |

### 계약 검토 기능 (/review-contract)

계약 검토는 조직의 플레이북을 기준으로 수행된다.

지원하는 계약 유형:
- SaaS 계약
- 전문 서비스 계약
- 라이선스 계약
- 파트너십 계약
- 조달 계약

분석 대상 조항:
- 책임 제한(Limitation of Liability)
- 면책(Indemnification)
- 지식재산권(IP)
- 데이터 조항

### 수정안(Redline) 생성 기능

계약 검토 결과에 기반한 수정안을 자동 생성한다.

- 삽입 가능한 구체적 수정 문구 제안
- 균형 잡힌 상업적으로 합리적인 접근법 제시
- 상대방에게 공유 가능한 전문적 근거 포함
- 협상을 위한 대안적 입장(fallback position) 제시
- 필수 항목과 선호 항목의 우선순위 구분

### 플레이북 설정

로컬 설정 파일에서 조직의 플레이북을 정의한다.

설정 가능한 항목:
- 계약 조건에 대한 표준 입장
- 협상 허용 범위
- 리스크 임계값에 따른 에스컬레이션 트리거

조직 플레이북이 설정되지 않은 경우, 널리 인정된 상업적 표준을 기반으로 검토를 수행한다.

### 외부 도구 연결

MCP를 통해 다음 도구와 연결할 수 있다.

| 도구 | 용도 |
|------|------|
| Slack | 커뮤니케이션 |
| Box | 문서 관리 |
| Egnyte | 문서 관리 |
| Jira | 프로젝트 추적 |
| Microsoft 365 | 이메일, 문서 |

### 중요 면책사항

"모든 출력물은 자격을 갖춘 변호사의 검토를 받아야 한다."
이 플러그인은 법률 자문을 대체하는 것이 아니라, 법무팀의 업무 효율성을 높이는 지원 도구로 설계되었다.

## 11개 오픈소스 플러그인

Anthropic은 GitHub에 11개 플러그인을 Apache 2.0 라이선스로 공개했다.

| 플러그인 | 주요 기능 | 주요 커넥터 |
|----------|----------|------------|
| Productivity | 태스크, 캘린더, 일상 워크플로 관리 | Slack, Notion, Asana, Linear, Jira |
| Sales | 잠재 고객 리서치, 콜 준비, 파이프라인 리뷰 | HubSpot, Close, Clay, ZoomInfo |
| Customer Support | 티켓 분류, 응답 초안, 에스컬레이션 패키징 | Intercom, HubSpot, Guru |
| Product Management | 스펙 작성, 로드맵 계획, 리서치 종합 | Linear, Asana, Figma, Amplitude |
| Marketing | 콘텐츠 초안, 캠페인 계획, 브랜드 보이스 적용 | Canva, Figma, HubSpot, Ahrefs |
| Legal | 계약 검토, NDA 분류, 컴플라이언스 | Box, Egnyte, Jira |
| Finance | 분개, 계정 대사, 재무제표, 차이 분석 | Snowflake, Databricks, BigQuery |
| Data | 쿼리, 시각화, 데이터셋 해석, SQL 작성 | Snowflake, Databricks, Hex |
| Enterprise Search | 이메일, 채팅, 문서, 위키 통합 검색 | Slack, Notion, Guru, Jira |
| Bio Research | 전임상 연구 도구 - 문헌 검색, 유전체학, 타겟 우선순위화 | PubMed, BioRender, ChEMBL, Benchling |
| Plugin Management | 조직 맞춤 플러그인 생성 및 커스터마이징 | - |

## 설치 및 사용법

### Claude Cowork에서 설치

[claude.com/plugins](https://claude.com/plugins){:target="_blank"}에서 직접 설치할 수 있다.
앱 내에서 검색하고 원클릭으로 설치하면 된다.

### Claude Code (CLI)에서 설치

```bash
# 마켓플레이스 추가
claude plugin marketplace add anthropics/knowledge-work-plugins

# 특정 플러그인 설치
claude plugin install legal@knowledge-work-plugins
claude plugin install sales@knowledge-work-plugins
claude plugin install data@knowledge-work-plugins
```

설치 후 스킬은 자동 활성화되며, 슬래시 명령어를 사용할 수 있다.

```
/legal:review-contract
/sales:call-prep
/data:write-query
/finance:reconciliation
/product-management:write-spec
```

## 커스터마이징

### 코드 없이 가능한 커스터마이징

플러그인의 핵심 설계 원칙은 선언적 마크다운과 JSON이다.
코드 컴파일, 배포 인프라, 인증 복잡성 없이 텍스트 편집만으로 커스터마이징할 수 있다.

### 커넥터 교체

`.mcp.json` 파일을 편집하여 조직의 도구 스택에 맞게 커넥터를 교체한다.

```json
{
  "mcpServers": {
    "slack": { },
    "hubspot": { },
    "custom_tool": { }
  }
}
```

### 조직 컨텍스트 추가

`skills/*.md` 파일에 다음 내용을 추가할 수 있다.

- 회사 용어 및 정의
- 조직 구조
- 표준 프로세스 및 워크플로
- 템플릿 형식
- 컴플라이언스 요구사항
- 브랜드 보이스

### 새 플러그인 생성

Plugin Management 플러그인을 사용하거나 표준 디렉토리 구조를 따라 직접 생성할 수 있다.
마크다운과 JSON만으로 기본 플러그인을 구성하며, 고급 기능이 필요한 경우 Python 함수를 추가할 수 있다.

## 시장 영향

### SaaSpocalypse

Legal 플러그인 공개 이후 시장에 큰 충격이 발생했다.
투자자들의 시각이 "AI가 소프트웨어 회사를 돕는 도구"에서 "소프트웨어 회사를 대체하는 도구"로 전환되었다.

### 주요 주가 영향

| 대상 | 변동 |
|------|------|
| Thomson Reuters | 급락 (Westlaw 법률 데이터 서비스 직접 위협) |
| Nasdaq | 1.4% 하락 |
| Infosys ADR | 5.5% 하락 |
| Wipro | 약 5% 하락 |

Thomson Reuters는 Westlaw 법률 데이터베이스를 보유한 기업으로, Claude의 법률 도구가 핵심 사업을 직접 겨냥했기 때문에 특히 큰 타격을 받았다.

### 시장 패닉의 본질

이 패닉은 법률 기술 분야에 국한되지 않았다.
전문 서비스와 소프트웨어 섹터 전반으로 확산되었다.
인도 IT 대기업들(Infosys, Wipro, TCS)의 주가 하락은 FTE(Full-Time Equivalent) 모델의 위기를 반영한다.

에이전틱 AI가 법률, 영업, 마케팅, 백오피스 전반의 다단계 전문 업무를 자동화하면서, 기업들은 "시간과 인력 기반 과금"에서 "성과와 효율성 기반 과금"으로 전환해야 하는 압박에 직면했다.

### Anthropic의 전략적 의미

Anthropic은 모델 공급자에서 애플리케이션 레이어와 워크플로 소유자로 포지셔닝을 전환하고 있다.
Claude Code가 "역대 가장 빠르게 성장한 제품"으로 10억 달러 매출을 달성한 이후, Cowork 플러그인은 엔터프라이즈 영역으로의 확장을 가속화한다.

Anthropic CEO Dario Amodei는 엔터프라이즈가 Anthropic 비즈니스의 80%를 차지한다고 밝힌 바 있다.
Gartner는 2026년 말까지 엔터프라이즈 애플리케이션의 40%가 AI 에이전트를 임베딩할 것으로 전망했다.
이는 2025년 5% 미만에서 급격한 증가이다.

## 시사점

### 법률 기술 생태계의 변화

분석가들은 범용적인 계약 검토를 제공하는 법률 AI 벤더가 존재적 위협에 직면한다고 평가한다.
그러나 커스터마이즈된 분석, 과거 데이터 통합, 시장 인텔리전스를 제공하는 프리미엄 도구는 여전히 우위를 유지한다.
법률 기술 벤더의 핵심 경쟁력은 독점 데이터셋과 도메인 전문성의 결합에 있다.

### 플러그인 아키텍처의 의의

마크다운과 JSON만으로 구성되는 선언적 아키텍처는 진입 장벽을 크게 낮춘다.
개발자가 아닌 도메인 전문가도 직접 플러그인을 만들고 커스터마이징할 수 있다.
Git 기반 버전 관리가 가능하여 팀 협업에도 적합하다.

### 고려할 점

- 현재 리서치 프리뷰 단계로, 조직 단위 관리 기능은 아직 미제공
- Legal 플러그인의 모든 출력물은 반드시 자격을 갖춘 변호사의 검토가 필요
- 플레이북 설정과 MCP 커넥터 구성에 초기 투자가 필요
- 조직 내부 데이터(과거 계약, 판례 등)와의 통합은 현재 제한적

## Reference

- [Claude Legal Plugin - Anthropic](https://claude.com/plugins/legal)
- [Knowledge Work Plugins - GitHub](https://github.com/anthropics/knowledge-work-plugins)
- [Claude Plugins Directory](https://claude.com/plugins)
- [Anthropic Moves Into Legal Tech - Artificial Lawyer](https://www.artificiallawyer.com/2026/02/02/anthropic-moves-into-legal-tech/)
