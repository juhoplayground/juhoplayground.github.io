---
layout: post
title: Claude Research Plugin - 체계적인 리서치를 위한 범용 플러그인
author: 'Juho'
date: 2026-01-16 09:00:00 +0900
categories: [Claude]
tags: [MCP, Research, Plugin, LLM]
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
1. [Claude Research Framework 개요](#claude-research-framework-개요)
2. [핵심 특징](#핵심-특징)
3. [5단계 리서치 파이프라인](#5단계-리서치-파이프라인)
4. [설치 방법](#설치-방법)
5. [사용 시나리오](#사용-시나리오)
6. [기술적 아키텍처](#기술적-아키텍처)
7. [실제 사용 예시](#실제-사용-예시)
8. [MCP 서버 통합](#mcp-서버-통합)
9. [참고 자료](#참고-자료)

## Claude Research Framework 개요

Claude Research Framework는 Claude Code를 위한 범용 리서치 플러그인입니다. 이 플러그인은 사용자가 광범위한 탐색에서 실행 가능한 인사이트까지 대화형 발견을 통해 도메인별 리서치를 수행할 수 있도록 지원합니다.

기존의 폼 기반 접근 방식 대신 자연스러운 대화를 통해 리서치 범위를 정의하고, 제조 AI부터 헬스케어, 핀테크, 교육까지 다양한 분야에 적용할 수 있는 5단계 리서치 파이프라인을 구현했습니다.

## 핵심 특징

### 대화형 발견 (Conversational Discovery)

딱딱한 설문지 방식을 벗어나 자연스러운 대화를 통해 리서치 범위를 정의합니다. 사용자의 의도와 상황에 맞춰 적응적으로 질문하며, 불필요한 질문을 최소화하는 것을 원칙으로 합니다.

### 도메인 독립적 설계 (Domain-Agnostic Design)

특정 산업에 맞춘 커스터마이징 없이 모든 분야에 적용 가능합니다. 제조업, 의료, 금융, 교육 등 어떤 도메인에서도 동일한 프레임워크를 사용할 수 있습니다.

### 체계적인 파이프라인 (Systematic Pipeline)

질문 생성부터 액션 플랜까지 5단계로 구성된 체계적인 프로세스를 제공합니다. 각 단계는 명확한 목적과 결과물을 가지고 있어 리서치의 진행 상황을 쉽게 파악할 수 있습니다.

### 실시간 정보 통합 (Real-time Integration)

MCP 서버(WebSearch, Sequential)를 활용하여 최신 정보를 실시간으로 수집하고 분석합니다. 학술 논문, 산업 트렌드, 실무 사례 등 다양한 출처의 정보를 통합합니다.

### 실무자 중심 (Practitioner-Focused)

학술적 엄격함보다는 실제 적용 가능성에 중점을 둡니다. 연구 결과를 실행 가능한 액션 아이템으로 변환하고, 성공 지표를 명확히 정의합니다.

## 5단계 리서치 파이프라인

### Step 0 - 의도 분석 (Intent Analysis)

개방형 대화를 통해 리서치 맥락을 파악합니다. 사용자의 배경, 목표, 제약사항을 이해하고 적응적으로 질문합니다.

**주요 활동**
- 사용자의 리서치 동기 파악
- 현재 지식 수준 확인
- 이해관계자 및 제약사항 식별
- 리서치 범위 초기 정의

### Step 1 - 핵심 질문 생성 (Key Questions)

테스트 가능한 5개의 리서치 질문을 생성하고 각각에 대한 근거를 제시합니다.

**출력 형식**
- 질문 1: 구체적이고 측정 가능한 질문
  - 근거: 왜 이 질문이 중요한가
- 질문 2-5: 동일한 형식

### Step 2 - 갭 식별 (Gap Identification)

연구가 부족한 영역과 새로운 기회를 매핑합니다. 기존 연구의 한계와 미래 가능성을 분석합니다.

**분석 영역**
- 문헌에서 다뤄지지 않은 영역
- 모순되는 연구 결과
- 새롭게 떠오르는 기회
- 실무와 이론의 갭

### Step 3 - 인사이트 추출 (Insight Extraction)

개별 출처를 깊이 있게 분석하여 실용적인 시사점을 도출합니다.

**추출 내용**
- 핵심 발견사항
- 실무 적용 가능성
- 제한사항 및 주의사항
- 추가 탐구가 필요한 부분

### Step 4 - 다중 출처 통합 (Multi-Source Synthesis)

여러 출처의 발견사항을 통합하고 모순을 해결합니다.

**통합 프로세스**
- 공통 패턴 식별
- 상충되는 주장 조정
- 종합적인 결론 도출
- 신뢰도 평가

### Step 5 - 액션 플랜 (Action Planning)

리서치 결과를 실행 가능한 항목으로 변환하고 성공 지표를 정의합니다.

**액션 아이템 구성**
- 구체적인 실행 단계
- 우선순위 및 의존성
- 성공 측정 기준
- 리스크 및 완화 전략

## 설치 방법

### 방법 1 - 플러그인 마켓플레이스 (권장)

```bash
/plugin marketplace add hongsw/plugin-for-claude-research
/plugin install domain-research
```

가장 간단하고 빠른 방법입니다. Claude Code의 플러그인 마켓플레이스를 통해 직접 설치할 수 있습니다.

### 방법 2 - 로컬 클론 및 등록

```bash
git clone https://github.com/hongsw/plugin-for-claude-research.git
```

저장소를 로컬에 클론한 후 Claude Code 설정에서 플러그인 경로를 등록합니다.

### 방법 3 - 작업 디렉토리로 추가

클론한 저장소를 Claude Code 내에서 작업 디렉토리로 직접 추가하여 사용할 수 있습니다.

## 사용 시나리오

Claude Research Framework는 네 가지 사용자 프로필에 맞춰 동작합니다.

### 1. 명확한 관심사 (Clearly Defined Interests)

**예시**: "중소기업의 품질 검사를 위한 컴퓨터 비전을 연구하고 싶습니다"

이 경우 2-3개의 명확화 질문만으로 리서치를 시작할 수 있습니다. 사용자가 이미 구체적인 목표를 가지고 있으므로 빠르게 파이프라인으로 진입합니다.

### 2. 탐색적 사용자 (Exploratory Users)

**예시**: "우리 공장에 AI를 적용하고 싶습니다"

4-6개의 발견 질문을 통해 구체적인 리서치 방향을 함께 찾아갑니다. 사용자의 상황, 제약사항, 우선순위를 파악하며 점진적으로 범위를 좁혀갑니다.

### 3. 지시받은 리서치 (Mandated Research)

**예시**: "상사가 블록체인에 대한 보고서를 요청했습니다"

이해관계자 정렬을 다룹니다. 보고서의 목적, 대상 독자, 기대 결과물을 명확히 하고 리서치를 진행합니다.

### 4. 일반적인 탐색 (General Exploration)

**예시**: "AI가 교육 분야에 어떻게 활용될 수 있을까요?"

점진적으로 초점을 좁혀갑니다. 광범위한 주제에서 시작하여 대화를 통해 구체적인 리서치 질문으로 발전시킵니다.

## 기술적 아키텍처

### 저장소 구조

```
plugin-for-claude-research/
├── plugins/
│   └── domain-research/     # 메인 플러그인 로직
├── skills/
│   └── domain-research/
│       └── SKILL.md         # 방법론 및 프롬프트 템플릿
├── examples/                # 실제 사용 사례
│   ├── manufacturing/
│   ├── healthcare/
│   └── fintech/
└── templates/               # 재사용 가능한 템플릿
```

### 핵심 컴포넌트

**Plugin Core**: 플러그인의 메인 로직이 위치한 `plugins/domain-research/` 디렉토리

**Skill Definitions**: 방법론과 프롬프트 템플릿을 문서화한 `skills/domain-research/SKILL.md`

**Examples**: 제조업, 헬스케어, 핀테크의 완전한 사용 사례 제공

**Templates**: 다양한 도메인에 재사용 가능한 템플릿

### 기술 스택

| 항목 | 내용 |
|------|------|
| 언어 | JavaScript 68.5%, HTML 31.5% |
| 라이선스 | MIT |
| 기여자 | 2명 claude, hongsw |
| 통계 | 9 stars, 5 forks, 9 commits |

## 실제 사용 예시

### 제조 AI (Manufacturing AI - 2026)

중소기업의 AI 도입 경로를 연구합니다.

**리서치 컨텍스트**
- 대상: 중소 제조업체
- 목표: AI 기반 품질 검사 시스템 도입
- 제약: 제한된 예산과 기술 인력

**핵심 질문**
- 중소기업에 적합한 컴퓨터 비전 기술은?
- ROI를 입증할 수 있는 파일럿 프로젝트는?
- 기존 시스템과의 통합 방안은?

### 헬스케어 스케줄링 (Healthcare Scheduling)

병원 직원 최적화를 위한 스케줄링 시스템을 연구합니다.

**리서치 컨텍스트**
- 대상: 중형 병원
- 목표: 간호사 스케줄링 자동화
- 제약: 노동법 준수, 공정성 유지

**핵심 질문**
- 다양한 근무 형태를 고려한 최적화 알고리즘은?
- 직원 선호도와 병원 요구사항의 균형점은?
- 기존 인력 관리 시스템과의 통합 방안은?

### 핀테크 및 블록체인 (FinTech/Blockchain)

국경 간 결제 시스템을 연구합니다.

**리서치 컨텍스트**
- 대상: 핀테크 스타트업
- 목표: 블록체인 기반 송금 서비스
- 제약: 규제 준수, 거래 속도

**핵심 질문**
- 어떤 블록체인 플랫폼이 적합한가?
- 각국 규제 요구사항을 충족하는 방법은?
- 기존 금융 시스템과의 연동 방안은?

## MCP 서버 통합

### WebSearch MCP 서버

실시간 논문 발견 및 트렌드 검증을 제공합니다.

**활용 단계**
- Step 1 (질문 생성): 최신 연구 동향 파악
- Step 2 (갭 식별): 미탐구 영역 발견

**제공 기능**
- 학술 논문 검색
- 산업 트렌드 분석
- 실시간 정보 수집

### Sequential MCP 서버

복잡한 다단계 분석을 가능하게 합니다.

**활용 단계**
- Step 4 (통합): 다중 출처 분석
- Step 5 (액션 플랜): 단계별 실행 계획 수립

**제공 기능**
- 순차적 추론
- 다단계 분석
- 의존성 관리

## 참고 자료

- [Claude Research Plugin GitHub Repository](https://github.com/hongsw/plugin-for-claude-research)
- [MCP Protocol Documentation](https://modelcontextprotocol.io)
- [Claude Code Official Guide](https://claude.ai/code)
