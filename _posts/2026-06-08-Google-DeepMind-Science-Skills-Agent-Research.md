---
layout: post
title: "Google DeepMind Science Skills: 과학 연구를 위한 에이전트 스킬 모음"
author: 'Juho'
date: 2026-06-08 00:00:00 +0900
categories: [AI]
tags: [AI, Skill, Agent]
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
3. [어떤 도메인을 다루는가](#어떤-도메인을-다루는가)
4. [저장소 구조](#저장소-구조)
5. [설치와 사용 방법](#설치와-사용-방법)
6. [의미와 시사점](#의미와-시사점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

google-deepmind/science-skills는 과학 연구 워크플로를 가속하기 위해 설계된 에이전트 스킬 모음이다.
이 저장소는 AI 에이전트의 역량을 특화된 과학 작업으로 확장하는 구조화된 지침, 스크립트, 리소스를 제공한다.
즉, 범용 AI 에이전트가 곧바로 과학 연구 영역의 작업을 수행할 수 있도록 도와주는 도구 묶음이다.

## 배경

AI 에이전트는 일반적인 작업에는 능하지만, 게놈학이나 구조생물학처럼 도메인 지식과 전용 데이터베이스가 필요한 과학 작업은 그대로 수행하기 어렵다.
science-skills는 이러한 간극을 메우기 위해 각 과학 작업에 맞는 지침과 스크립트, 리소스를 패키지 형태로 제공한다.
이를 통해 에이전트는 특화된 과학 작업으로 역량을 확장할 수 있다.

## 어떤 도메인을 다루는가

science-skills가 다루는 과학 도메인은 다음과 같다.

| 도메인 | 설명 |
| --- | --- |
| 게놈학 (genomics) | 유전체 데이터 분석 관련 작업 |
| 구조생물학 (structural biology) | 단백질 구조 등 분자 구조 관련 작업 |
| 케모인포매틱스 (cheminformatics) | 화합물 정보 분석 작업 |
| 문헌 검색 | 과학 논문 및 학술 문헌 탐색 작업 |

제공되는 스킬은 AlphaGenome, AFDB, UniProt, ClinVar, OpenAlex 등 30개 이상의 데이터베이스 및 도구와 통합된다.

## 저장소 구조

각 스킬은 독립된 디렉터리로 구성되며, 다음 요소를 포함한다.

| 구성 요소 | 역할 |
| --- | --- |
| SKILL.md | YAML frontmatter와 마크다운 문서로 작성된 지침 파일 |
| scripts/ | 헬퍼 스크립트 및 유틸 |
| references/ | 보충 문서 (선택) |

SKILL.md는 스킬의 핵심 지침을 담고, scripts 디렉터리는 작업을 돕는 스크립트와 유틸을 제공하며, references 디렉터리는 필요에 따라 추가되는 보충 문서를 담는다.

## 설치와 사용 방법

스킬 설치는 NPX를 통해 수행한다.

```bash
npx skills add google-deepmind/science-skills/
```

설치된 스킬은 DeepMind의 AI 플랫폼인 Google Antigravity와 통합되어 동작한다.
초기 설정에는 uv 패키지 매니저가 필요하며, 에이전트가 최초 사용 시 자동으로 설치한다.

일부 스킬은 API 키가 필요하다.
AlphaGenome과 OpenAlex는 API 키를 요구하고, ClinVar 등은 키가 있으면 도움이 되지만 필수는 아니다.

| 스킬 | API 키 |
| --- | --- |
| AlphaGenome | 필요 |
| OpenAlex | 필요 |
| ClinVar | 도움되나 필수 아님 |

## 의미와 시사점

science-skills는 범용 AI 에이전트를 과학 연구의 실무 도구로 끌어올리는 시도다.
30개 이상의 데이터베이스 및 도구와의 통합은 에이전트가 흩어진 과학 리소스를 일관된 방식으로 활용할 수 있게 한다.
SKILL.md, scripts, references로 구성된 표준화된 구조는 새로운 과학 작업을 스킬 형태로 추가하기 쉽게 만든다.

저장소의 기타 정보는 다음과 같다.

| 항목 | 내용 |
| --- | --- |
| 언어 | Python 100% |
| 라이선스 | Apache 2.0 (소프트웨어), CC-BY (기타 자료) |
| 스타 | 1.6k (2026년 5월 기준) |
| 포크 | 154 |
| 릴리스 | 3 |

## 결론

google-deepmind/science-skills는 게놈학, 구조생물학, 케모인포매틱스, 문헌 검색 등의 과학 도메인을 다루는 에이전트 스킬 모음이다.
구조화된 지침과 스크립트, 30개 이상의 도구 통합을 통해 AI 에이전트의 역량을 특화된 과학 작업으로 확장한다.
NPX로 손쉽게 설치하고 Google Antigravity와 통합해 사용할 수 있어, 과학 연구 워크플로를 가속하려는 연구자에게 유용한 출발점이 된다.

## Reference

- [google-deepmind/science-skills](https://github.com/google-deepmind/science-skills/)
