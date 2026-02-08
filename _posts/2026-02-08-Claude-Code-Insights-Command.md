---
layout: post
title: Claude Code /insights 명령어 - 사용 패턴 분석과 워크플로 개선 제안
author: 'Juho'
date: 2026-02-08 02:00:00 +0900
categories: [AI Tools]
tags: [Claude Code, AI, LLM, Coding, Productivity]
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
2. [/insights 명령어란](#insights-명령어란)
3. [주요 기능](#주요-기능)
4. [사용 방법](#사용-방법)
5. [정리](#정리)

## 개요

Claude Code v2.1.30부터 새로운 `/insights` 명령어가 추가되었다.
이 기능은 사용자의 Claude Code 활용 방식을 분석하고 개선점을 제안하는 역할을 한다.

## /insights 명령어란

`/insights` 명령어를 실행하면 Claude Code가 지난 한 달간의 메시지 기록을 읽어들인다.
이를 바탕으로 사용자의 프로젝트를 요약하고, Claude Code 사용 방식을 분석하며, 워크플로 개선을 위한 제안을 제공한다.

공식 발표에 따르면 다음과 같이 설명된다.

> We've added a new command to Claude Code called /insights.
> When you run it, Claude Code will read your message history from the past month.
> It'll summarize your projects, how you use Claude Code, and give suggestions on how to improve your workflow.

## 주요 기능

### 프로젝트 요약

지난 한 달간 진행한 프로젝트들의 개요를 정리해준다.
어떤 작업을 수행했는지, 주요 변경 사항이 무엇이었는지 파악할 수 있다.

### 사용 패턴 분석

사용자가 Claude Code를 어떻게 활용하고 있는지 분석한다.
자주 사용하는 기능, 작업 유형, 시간대별 사용량 등을 파악할 수 있다.

### 워크플로 개선 제안

분석 결과를 바탕으로 작업 효율성을 높이기 위한 조언을 제공한다.
사용자 평가에 따르면 "마치 인간 관리자가 보고서를 작성해주는 정도의 느낌"이라고 한다.

## 사용 방법

Claude Code에서 다음 명령어를 입력하면 된다.

```bash
/insights
```

명령어 실행 후 Claude Code가 로컬에 저장된 세션 데이터를 분석하여 결과를 출력한다.

## 정리

`/insights` 명령어는 Claude Code 사용자에게 자신의 작업 패턴을 돌아보고 개선점을 찾을 수 있는 기회를 제공한다.
지난 한 달간의 메시지 기록을 분석하여 프로젝트 요약, 사용 패턴 분석, 워크플로 개선 제안을 제공한다.
정기적으로 이 기능을 활용하면 Claude Code를 더 효과적으로 사용하는 방법을 발견할 수 있을 것이다.
