---
layout: post
title: "단순한 파일이 정교한 도구를 이긴다 - AGENTS.md가 Skills보다 47%p 우수한 이유"
author: 'Juho'
date: 2026-02-21 00:00:00 +0900
categories: [AI]
tags: [AI, Dev, Prompt]
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
2. [문제 정의 - AI 에이전트의 지식 공백](#문제-정의---ai-에이전트의-지식-공백)
3. [두 가지 접근법 비교](#두-가지-접근법-비교)
4. [벤치마크 결과](#벤치마크-결과)
5. [Skills가 실패한 이유](#skills가-실패한-이유)
6. [AGENTS.md의 핵심 설계](#agentsmd의-핵심-설계)
7. [압축 전략 - 40KB를 8KB로](#압축-전략---40kb를-8kb로)
8. [실전 적용 방법](#실전-적용-방법)
9. [Skills와 AGENTS.md의 역할 분담](#skills와-agentsmd의-역할-분담)
10. [결론](#결론)
11. [Reference](#reference)

## 개요

Vercel이 AI 코딩 에이전트에게 Next.js API를 가르치는 두 가지 방식을 비교 평가한 결과를 공개했다.
정교한 도구 기반 시스템인 Skills와 단순한 마크다운 파일인 AGENTS.md를 비교한 결과, AGENTS.md가 47%p 차이로 압도적인 성능을 보였다.
이 결과는 AI 에이전트에게 프레임워크 지식을 전달하는 최적의 방식에 대한 중요한 시사점을 제공한다.

## 문제 정의 - AI 에이전트의 지식 공백

AI 모델의 학습 데이터는 빠르게 구식이 된다.
Next.js 16에서 도입된 `use cache`, `connection()`, `forbidden()` 같은 새로운 API는 현재 모델의 학습 데이터에 포함되어 있지 않다.
이로 인해 에이전트는 레거시 코드를 생성하거나 잘못된 API를 사용하게 된다.

Vercel은 이 문제를 해결하기 위해 두 가지 접근법을 실험했다.

## 두 가지 접근법 비교

### Skills 방식

Skills는 도메인 지식을 프롬프트, 도구, 문서로 패키징한 개방형 표준이다.
에이전트가 필요할 때 호출하는 온디맨드 방식으로 작동한다.
Anthropic의 내부 포맷에서 시작해 Vercel이 배포 레이어를 구축한 패키지 매니저 형태다.

### AGENTS.md 방식

프로젝트 루트에 위치한 정적 마크다운 파일이다.
에이전트가 매번 자동으로 읽으며, 별도의 의사결정이 필요 없다.
시스템 프롬프트와 함께 항상 로드되어 일관된 정보 접근을 보장한다.

## 벤치마크 결과

Vercel은 Next.js 16의 학습 데이터에 없는 API들을 대상으로 평가를 수행했다.

### 전체 통과율

| 구성 | 통과율 | 개선폭 |
|------|--------|--------|
| 베이스라인 (문서 없음) | 53% | - |
| Skills (기본) | 53% | +0pp |
| Skills + 명시적 지시 | 79% | +26pp |
| AGENTS.md 문서 인덱스 | 100% | +47pp |

### 세부 항목별 결과

| 구성 | Build | Lint | Test |
|------|-------|------|------|
| Skills + 지시 | 95% | 100% | 84% |
| AGENTS.md | 100% | 100% | 100% |

AGENTS.md는 Build, Lint, Test 모든 항목에서 만점을 기록했다.

## Skills가 실패한 이유

Skills 방식의 핵심 문제는 에이전트의 도구 호출 판단에 있었다.

**56%의 평가 케이스에서 Skill이 한 번도 호출되지 않았다.**
에이전트가 문서에 접근할 수 있는 상태였음에도 스스로 판단해서 사용하지 않은 것이다.

명시적 지시("이 Skill을 반드시 사용하라")를 추가하면 호출률이 95%까지 올라갔지만, 이 접근법은 근본적으로 취약했다.
지시문의 표현을 조금만 바꿔도 행동이 크게 달라지는 불안정한 결과를 보였다.

세 가지 구조적 원인이 있다.

- 의사결정 지점이 존재한다 - 에이전트가 Skill을 쓸지 말지 매번 판단해야 한다
- 순서 딜레마가 발생한다 - 문서를 먼저 읽을지, 프로젝트를 먼저 분석할지 혼란이 생긴다
- 가용성이 일관적이지 않다 - 호출 여부에 따라 정보 접근이 달라진다

## AGENTS.md의 핵심 설계

AGENTS.md가 성공한 이유는 세 가지로 요약된다.

첫째, 의사결정 지점이 없다.
정보가 항상 존재하므로 에이전트가 "이 도구를 써야 할까?"라는 판단을 할 필요가 없다.

둘째, 일관된 가용성을 보장한다.
시스템 프롬프트와 함께 매번 로드되므로 세션마다 동일한 정보를 제공한다.

셋째, 핵심 지시문이 행동을 결정한다.
AGENTS.md에 포함된 핵심 지시는 다음과 같다.

> "Prefer retrieval-led reasoning over pre-training-led reasoning for any Next.js tasks."

이 한 줄이 에이전트가 학습 데이터 대신 제공된 문서를 우선 참조하도록 유도한다.

## 압축 전략 - 40KB를 8KB로

전체 문서를 AGENTS.md에 넣으면 컨텍스트 윈도우를 과도하게 소비한다.
Vercel은 파이프 구분자 기반 압축 구조를 사용해 40KB 문서를 8KB로 80% 축소했다.

이 압축된 인덱스는 전체 내용을 포함하지 않는다.
대신 검색 가능한 문서 파일들을 가리키는 포인터 역할을 한다.
에이전트는 인덱스를 보고 필요한 문서를 검색해서 읽는 방식으로 동작한다.

압축 후에도 통과율 100%를 유지했다.

## 실전 적용 방법

다음 명령어로 프로젝트에 즉시 적용할 수 있다.

```bash
npx @next/codemod@canary agents-md
```

이 명령어는 세 가지를 수행한다.

1. 프로젝트의 Next.js 버전을 감지한다
2. 해당 버전에 맞는 문서를 `.next-docs/` 디렉토리에 다운로드한다
3. 압축된 인덱스를 AGENTS.md에 주입한다

### 평가 대상 API 목록

평가에 사용된 Next.js 16 API들이다.

| API | 설명 |
|-----|------|
| connection() | 동적 렌더링 트리거 |
| use cache | 캐시 디렉티브 |
| cacheLife() | 캐시 수명 설정 |
| cacheTag() | 캐시 태그 지정 |
| forbidden() | 403 응답 |
| unauthorized() | 401 응답 |
| cookies() / headers() | 비동기 버전 |
| after() | 응답 후 실행 |
| updateTag() | 태그 갱신 |
| proxy.ts | API 프록시 |

## Skills와 AGENTS.md의 역할 분담

이 결과가 Skills의 무용론을 의미하지는 않는다.
Vercel은 두 접근법의 최적 용도가 다르다고 결론지었다.

AGENTS.md는 수평적 개선에 적합하다.
프레임워크 전반에 걸친 일반 지식을 다양한 작업에 적용할 때 효과적이다.

Skills는 수직적, 액션 특화 워크플로우에 적합하다.
버전 업그레이드나 마이그레이션 같은 특정 작업에서 여전히 가치를 발휘한다.

## 결론

Vercel의 실험은 AI 에이전트에게 지식을 전달하는 방식에 대한 반직관적인 결론을 보여준다.
정교한 도구 시스템보다 단순한 마크다운 파일이 더 효과적이었다.

핵심 교훈은 명확하다.
에이전트의 의사결정 부담을 줄이는 것이 도구의 정교함보다 중요하다.
정보가 항상 존재하면 "써야 할까?"라는 판단 자체가 사라지고, 에이전트는 본래 작업에 집중할 수 있다.

## Reference

- [AGENTS.md outperforms skills in our agent evals - Vercel](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [Agent Skills - Vercel Docs](https://vercel.com/docs/agent-resources/skills)
