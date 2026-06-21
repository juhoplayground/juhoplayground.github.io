---
layout: post
title: "16개의 Claude 에이전트가 C 컴파일러를 만들다: 멀티 에이전트 개발 실험"
author: 'Juho'
date: 2026-06-17 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, Compiler]
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
2. [핵심 아키텍처](#핵심-아키텍처)
   - [중앙 저장소 모델](#중앙-저장소-모델)
   - [RALPH 루프](#ralph-루프)
3. [관리 기법](#관리-기법)
   - [태스크 잠금](#태스크-잠금)
   - [테스트 하네스 최적화](#테스트-하네스-최적화)
   - [외부 메모리와 역할 분담](#외부-메모리와-역할-분담)
4. [한계와 평가](#한계와-평가)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Anthropic은 16개의 자율 Claude Opus 4.6 에이전트가 협력해 약 2주에 걸쳐 동작하는 C 컴파일러를 만드는 실험을 진행했다.
이 컴파일러는 Rust로 작성되었고 약 100,000줄 규모이며, API 비용으로 20,000달러 이상이 소요되었다.
결과물은 인상적이지만, 순수한 자율성보다는 광범위한 인간의 스캐폴딩(scaffolding)에 의해 달성되었다.
이 실험은 복잡한 멀티 에이전트 시스템을 관리하는 데 유용한 교훈을 제공한다.

## 핵심 아키텍처

### 중앙 저장소 모델

Git 기반의 `upstream` 디렉토리가 권위 있는 소스 코드 저장소 역할을 했다.
이를 통해 모든 에이전트 작업 공간 간 동기화가 가능했다.
각 에이전트는 자신만의 Docker 컨테이너 안에서 동작하며 별도의 `workspace`를 가졌다.
에이전트는 수정 전에 upstream 저장소를 클론하고, 작업 완료 후 변경 사항을 다시 푸시했다.
이 Docker 격리는 에이전트들이 서로의 작업을 방해하지 않도록 보장했다.

### RALPH 루프

RALPH 루프는 지속적으로 실행되는 Bash 스크립트로, 에이전트가 끊김 없이 동작할 수 있게 했다.
각 태스크마다 새로운 Claude 세션을 시작하는 방식으로, 상태가 없는(stateless) 모델을 지속적으로 일하는 워커로 변환했다.

```bash
while true; do
  COMMIT=$(git rev-parse --short=6 HEAD)
  LOGFILE="agent_logs/agent_${COMMIT}.log"
  AGENT_PROMPT=$(python3 pick_task.py)
  claude --dangerously_skip_permissions \
         -p "$(cat AGENT_PROMPT.md)" \
         --model claude-opus-X-Y \
         &> "$LOGFILE"
done
```

이 루프는 현재 커밋 해시를 읽어 로그 파일명을 만들고, 다음 작업을 선택한 뒤, 새 Claude 세션으로 해당 작업을 실행한다.
세션이 종료되면 루프가 다시 처음으로 돌아가 다음 태스크를 이어받는다.

## 관리 기법

### 태스크 잠금

에이전트들은 `current_tasks/` 디렉토리에 잠금 파일(lock file)을 생성하고 이를 Git에 커밋했다.
각 잠금에 대해 단 하나의 에이전트만 성공적으로 푸시할 수 있어 중복 작업을 방지했다.
작업이 완료되면 해당 잠금 파일은 삭제되었다.
이 방식으로 16개의 에이전트가 동시에 같은 태스크를 붙잡는 충돌을 막았다.

### 테스트 하네스 최적화

테스트 하네스에는 여러 최적화가 적용되었다.
장황한 테스트 출력을 필터링해 토큰 컨텍스트를 보존했다.
`--fast` 플래그를 구현해 빠른 샘플링을 가능하게 했으며, 이는 에이전트별로는 결정론적(deterministic)이고 팀 전체에서는 무작위(random)였다.
또한 GNU Compiler Collection(GCC)을 오라클(oracle)로 사용해 실패하는 코드 구간을 격리했다.

다음은 주요 최적화 기법을 정리한 것이다.

| 기법 | 목적 |
|------|------|
| 출력 필터링 | 장황한 테스트 출력을 줄여 토큰 컨텍스트 보존 |
| fast 플래그 | 빠른 샘플링, 에이전트별 결정론 및 팀 전체 무작위성 |
| GCC 오라클 | 실패하는 코드 구간을 격리 |

### 외부 메모리와 역할 분담

README와 진행 상황 파일이 새 세션을 위한 외부 메모리(external memory) 역할을 했다.
이를 통해 새로 시작되는 세션에 컨텍스트를 제공하고, 회귀(regression)와 중복 작업을 방지했다.
에이전트들은 서로 다른 역할을 맡았다.
리팩토러(Refactorer), 성능 튜너(Performance Tuner), 코드 비평가(Code Critic), 문서 작성자(Documenter) 등으로 전문화되었다.
이러한 역할 분담은 프로젝트의 여러 차원에서 병렬적인 개선을 가능하게 했다.

## 한계와 평가

완성된 컴파일러는 몇 가지 명확한 한계를 가졌다.

| 한계 | 설명 |
|------|------|
| 어셈블러와 링커 부재 | 자체 어셈블러와 링커가 없어 GCC에 의존 |
| 프로세서 모드 | 모든 프로세서 모드를 처리하지 못함 |
| 코드 효율 | GCC보다 훨씬 비효율적인 코드를 생성 |

"인간의 적극적 개입 없이(without active human intervention)"라는 주장은 단서가 필요하다.
에이전트들이 자율적으로 작업한 것은 맞지만, Nicholas Carlini가 전체 아키텍처를 설계했다.
그는 RALPH 스크립트를 만들고, 최적화가 포함된 테스트 하네스를 구축했으며, 오라클 전략을 고안하고, 전문화된 역할을 정의했다.
따라서 이는 순수한 자율 개발이라기보다 인간이 안내하는 AI 자동화(human-guided AI automation)에 가깝다.

## 결론

이 실험은 에이전트 팀 패러다임의 개념 증명(proof-of-concept)으로서 성공적이다.
적절히 설계된 멀티 에이전트 시스템이 상당한 규모의 소프트웨어 프로젝트를 다룰 수 있음을 보여주었다.
진정한 가치는 컴파일러 그 자체가 아니라, 향후 AI 주도 엔지니어링을 위한 운영 청사진에 있다.
상태 없는 모델을 지속적인 워커로 바꾸는 루프, 충돌을 막는 잠금, 컨텍스트를 보존하는 외부 메모리, 역할 분담이라는 구성 요소가 그 핵심이다.

## Reference

- [Multi-Agent AI Development: 16 Claude Agents Build C Compiler](https://betterstack.com/community/guides/ai/anthropic-ai-agents-c-compiler/)
