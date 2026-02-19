---
layout: post
title: "Claude 에이전트 팀, 2주 만에 리눅스 컴파일러 제작한 방법"
author: 'Juho'
date: 2026-02-19 01:00:00 +0900
categories: [AI]
tags: [AI, LLM, Dev]
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
2. [에이전트 팀의 작동 방식](#에이전트-팀의-작동-방식)
3. [리눅스 컴파일러 개발 과정](#리눅스-컴파일러-개발-과정)
4. [핵심 설계 원칙](#핵심-설계-원칙)
5. [성과와 한계](#성과와-한계)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Anthropic 연구원 Nicholas Carlini는 16개의 Claude AI 인스턴스를 병렬로 실행하여 2주 만에 10만 줄의 C 컴파일러를 작성했다.
이 프로젝트에는 2만 달러의 API 비용과 2,000개의 Claude Code 세션이 투입되었다.
이 글은 에이전트 팀이 어떻게 협업하여 대규모 소프트웨어 프로젝트를 완성했는지, 그 과정에서 얻은 교훈을 정리한다.

## 에이전트 팀의 작동 방식

여러 Claude 인스턴스가 병렬로 작동하며 공유 태스크 리스트를 사용한다.
Git 기반 파일 잠금으로 동시 작업 충돌을 방지한다.
각 에이전트가 독립적인 컨텍스트에서 작업하고 결과를 통합한다.
마스터 에이전트 없이 자율적으로 다음 작업을 결정하는 구조이다.

## 리눅스 컴파일러 개발 과정

### 초기 병렬화 성공

초반에는 if문 파싱, 함수 정의 등 독립적인 문제들을 각 에이전트가 나누어 해결했다.
각 에이전트가 서로 다른 기능을 담당하면서 빠르게 진행할 수 있었다.

### 후기 병목 현상

테스트 통과율이 99%에 도달하자 모든 에이전트가 동일한 버그를 발견하고 서로의 수정사항을 덮어쓰는 문제가 발생했다.
병렬 작업의 한계가 드러난 지점이다.

### 해결책

GCC를 알려진 정답 기준으로 설정하여, 커널의 대부분은 GCC로 컴파일하고 Claude 컴파일러가 특정 파일만 담당하도록 구조화했다.
이를 통해 에이전트 간 충돌을 최소화할 수 있었다.

## 핵심 설계 원칙

Carlini가 강조한 핵심 교훈은 다음과 같다.

### 완벽한 테스트 하네스

Claude는 주어진 문제를 자율적으로 해결하려 하지만, 문제 검증기가 부정확하면 잘못된 문제를 풀 위험이 있다.
정확한 테스트 하네스가 필수적이다.

### 맥락 유지

광범위한 README와 진행 상황 파일을 유지하여 새 컨테이너의 에이전트 적응 시간을 단축했다.
에이전트가 프로젝트 상태를 빠르게 파악할 수 있도록 하는 것이 중요하다.

### LLM 한계 고려

컨텍스트 오염 방지를 위해 테스트는 파일로 기록했다.
속도를 위해 `--fast` 옵션으로 전체 테스트의 1~10% 샘플만 실행하는 방식을 채택했다.

### 에이전트 전문화

한 에이전트는 중복 코드 통합을, 다른 에이전트는 성능 최적화를 담당하도록 역할을 분리했다.
전문화를 통해 각 에이전트의 효율성을 높일 수 있었다.

## 성과와 한계

### 성과

- Linux 6.9를 x86, ARM, RISC-V에서 부팅 가능
- QEMU, FFmpeg, SQLite, Redis 컴파일 성공
- GCC torture 테스트에서 99% 통과율 달성
- Doom 게임도 컴파일 및 실행 가능

### 한계

- 16비트 x86 컴파일러 부재로 real mode 부팅 시 GCC에 의존해야 한다.
- 어셈블러와 링커에 다수의 버그가 존재한다.
- 생성된 코드의 효율성이 부족하여, 최적화를 적용해도 GCC 비최적화 코드보다 느리다.
- 새 기능 추가 시마다 기존 기능이 손상되는 현상이 반복되었다.

Carlini는 Opus 4.6의 능력이 거의 한계에 도달했다고 평가했다.

## 결론

이 프로젝트는 AI와의 협업 방식이 진화하는 과정을 보여준다.
동시에 개발자가 직접 검증하지 않은 소프트웨어를 배포하는 것에 대한 보안 우려도 제기된다.
AI 에이전트 팀을 활용한 대규모 소프트웨어 개발은 가능성과 한계를 동시에 보여주며, 새로운 전략이 필요한 시대에 진입했음을 시사한다.

## Reference

- [Building a C compiler with a team of parallel Claudes - Anthropic Engineering Blog](https://www.anthropic.com/engineering/building-c-compiler)
- [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Claude's C Compiler - GitHub](https://github.com/anthropics/claudes-c-compiler)
