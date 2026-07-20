---
layout: post
title: "OpenScience: 문헌부터 실험·보고서까지 자동화하는 AI 연구 워크벤치"
author: 'Juho'
date: 2026-07-09 00:00:00 +0900
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
2. [핵심 기능](#핵심-기능)
   - [연구 루프 자동화](#연구-루프-자동화)
   - [다중 전문가 에이전트와 스킬](#다중-전문가-에이전트와-스킬)
3. [기술 스택과 아키텍처](#기술-스택과-아키텍처)
   - [설치와 실행](#설치와-실행)
4. [특징과 확장성](#특징과-확장성)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

OpenScience는 synthetic-sciences가 공개한 AI 기반 과학 연구 워크벤치다.
사용자가 목표를 제시하면, 문헌을 읽고, 코드를 작성·실행하며, 실험을 수행하고, 결과를 정리하는 방식으로 동작한다.
문헌 검토부터 가설 수립, 코드 실행, 실험, 분석, 보고서 작성까지 이어지는 연구 루프를 하나의 세션에서 완결하는 것이 목표다.

## 핵심 기능

### 연구 루프 자동화

OpenScience의 핵심은 연구의 전 과정을 자동화하는 것이다.
문헌 검토, 가설 수립, 코드 작성 및 실행, 실험, 분석, 보고서 작성이 하나의 세션 안에서 순차적으로 진행된다.
브라우저 UI에서는 파일 트리, 편집기, 터미널, 세션 히스토리를 제공하며, 분자·구조·게놈·그래프를 인라인으로 렌더링한다.

### 다중 전문가 에이전트와 스킬

기본 research 에이전트 외에도 biology, physics, ml 전문가 에이전트가 존재한다.
비평과 문헌검색을 담당하는 하위 에이전트를 두고, 계획 전용 읽기 모드도 제공한다.
여기에 290개 이상의 스킬과 30여 개의 과학 데이터베이스 통합이 결합된다.

| 구분 | 내용 |
|------|------|
| 머신러닝 스킬 | DeepSpeed, PEFT, TRL 등 훈련 도구 |
| 생물·화학 스킬 | 분자/임상 생물학, 케미인포매틱스 |
| 문서·인프라 스킬 | LaTeX 및 논문 처리, 클라우드 컴퓨팅(Modal, Tinker) |
| 데이터베이스 | UniProt, PDB, Ensembl, ChEMBL, PubChem, arXiv, OpenAlex, Semantic Scholar 등 |

## 기술 스택과 아키텍처

TypeScript(54.4%)와 Python(31.2%), TeX(8.6%)로 구성되며 Bun 1.3 이상 런타임에서 동작한다.
모노레포 관리에는 Turbo를 사용한다.

로컬 서버가 UI, 에이전트 런타임, 도구 레이어를 호스팅하고, 에이전트는 셸·편집기·LSP·MCP 서버·과학 커넥터를 사용한다.

```
backend/cli          → CLI, 서버, 제공자 통합, 세션, 스킬
frontend/workspace   → 브라우저 UI
frontend/docs        → 문서 및 공유 사이트
tooling/sdk/js       → TypeScript SDK
tooling/plugin       → 플러그인 런타임
```

### 설치와 실행

전역 설치 후 바로 실행하거나, npx로 일회성 실행이 가능하다.

```bash
# 전역 설치
npm install -g @synsci/openscience
openscience

# 일회용 실행
npx synsci

# 특정 프로젝트에서 시작
openscience ~/code/my-project
```

소스에서 개발할 때는 Bun 명령을 사용한다.

```bash
bun install
bun dev                          # 소스에서 워크스페이스 실행
bun run typecheck                # 타입 체크
bun run --cwd backend/cli test
bun run --cwd backend/cli build  # 플랫폼 바이너리 생성
```

## 특징과 확장성

OpenScience는 모델 독립적이다.
Anthropic, OpenAI, Google 등 수십 개 제공자의 최신 또는 오픈가중 모델을 지원하며, 사용자의 API 키를 사용하고 별도 계정이 필요 없다.
"Atlas"라는 관리형 플랫폼 옵션도 선택적으로 제공된다.

확장성 측면에서는 LSP 통합, MCP 서버, 플러그인을 지원하고, 커스텀 에이전트·명령·도구를 추가할 수 있으며 TypeScript SDK가 제공된다.
설정은 전역(`~/.config/openscience/openscience.json`)과 프로젝트(`openscience.json` 또는 `.openscience/` 디렉토리) 두 수준으로 관리한다.

한 가지 유의할 점은 에이전트가 샌드박스 처리되지 않으며 대신 권한 시스템으로 제어된다는 것이다.
또한 이 프로젝트는 Anthropic과 무관한 독립 프로젝트로, Claude 호환성만 표시한다.
라이선스는 Apache License 2.0이다.

## 결론

OpenScience는 연구자가 목표만 제시하면 문헌 조사부터 실험, 보고서까지 한 세션에서 처리하도록 설계된 오픈소스 연구 워크벤치다.
290개 이상의 스킬과 30여 개의 과학 데이터베이스, 다중 전문가 에이전트를 결합하고, 모델 독립성과 MCP·플러그인 기반 확장성을 갖춘 점이 특징이다.
반복적인 과학 연구 파이프라인을 에이전트로 자동화하려는 연구자와 개발자에게 살펴볼 만한 프로젝트다.

## Reference

- [synthetic-sciences/openscience](https://github.com/synthetic-sciences/openscience/)
