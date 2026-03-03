---
layout: post
title: "Claude Cowork 반복 작업 스케줄링 기능 출시"
author: 'Juho'
date: 2026-03-03 00:00:00 +0900
categories: [AI]
tags: [AI, LLM, Agent]
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
2. [주요 기능](#주요-기능)
   - [지원하는 작업 유형](#지원하는-작업-유형)
   - [실행 주기 옵션](#실행-주기-옵션)
3. [사용 방법](#사용-방법)
   - [방법 1: /schedule 명령 활용](#방법-1-schedule-명령-활용)
   - [방법 2: Scheduled 페이지 활용](#방법-2-scheduled-페이지-활용)
4. [주의사항](#주의사항)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Anthropic이 Claude Desktop 앱의 Cowork 기능에 반복 작업 스케줄링 기능을 추가했습니다.
한 번 설정해두면 Claude가 정해진 주기에 따라 자동으로 작업을 실행해주는 기능입니다.
이메일 요약부터 보고서 자동 생성까지, 반복적인 업무를 자동화할 수 있습니다.

## 주요 기능

### 지원하는 작업 유형

Claude Cowork의 스케줄 기능은 다양한 반복 업무를 자동화할 수 있습니다.

| 작업 유형 | 예시 |
|----------|------|
| 일일 브리핑 | Slack, 이메일, 캘린더 요약 자동 생성 |
| 주간 보고서 | Google Drive, 스프레드시트 데이터 컴파일 |
| 정기 조사 | 주제, 경쟁사, 업계 뉴스 추적 |
| 파일 정리 | 폴더별 파일 정렬 및 처리 |
| 팀 업데이트 | 프로젝트 관리 도구에서 상태 보고서 생성 |

### 실행 주기 옵션

스케줄 작업의 실행 주기는 다음 중에서 선택할 수 있습니다.

| 주기 | 설명 |
|------|------|
| 매시간 | 매 시간마다 실행 |
| 매일 | 하루 한 번 실행 |
| 평일 | 월요일~금요일 실행 |
| 매주 | 일주일에 한 번 실행 |
| 수동 | 필요할 때 직접 실행 |

## 사용 방법

### 방법 1: /schedule 명령 활용

Cowork 작업 창 내에서 `/schedule` 명령을 입력하면 Claude가 스케줄 설정을 안내합니다.
원하는 작업 내용을 먼저 작성한 후, `/schedule` 명령을 입력하면 실행 주기를 선택할 수 있습니다.

### 방법 2: Scheduled 페이지 활용

Claude Desktop 앱의 "Scheduled" 페이지에서 "+ New task" 버튼을 클릭합니다.
작업명, 작업 설명, 프롬프트, 실행 빈도를 입력하면 스케줄이 등록됩니다.
등록된 작업은 목록에서 보기, 편집, 일시 중지, 재개, 삭제, 수동 실행이 가능합니다.

## 주의사항

스케줄된 작업은 "컴퓨터가 깨어 있고 Claude Desktop 앱이 열려 있을 때만 실행"됩니다.
컴퓨터가 꺼져 있거나 앱이 닫혀 있을 때 건너뛴 작업은, 컴퓨터 재시작 후 자동으로 실행됩니다.
각 작업은 독립적인 Cowork 세션으로 진행되며, 기존 커넥터와 플러그인을 모두 활용할 수 있습니다.

이 기능은 Pro, Max, Team, Enterprise 플랜에서만 지원되며, Claude Desktop 앱이 필요합니다.

## 결론

Claude Cowork의 스케줄 기능은 반복적인 업무를 자동화하여 개발자와 업무 담당자의 생산성을 높여줍니다.
일일 브리핑, 주간 보고서, 뉴스 모니터링 등 정기적으로 수행해야 하는 작업들을 Claude에게 맡길 수 있습니다.
AI 에이전트가 단순한 대화 상대를 넘어 실질적인 업무 자동화 도구로 자리잡아 가는 흐름을 보여주는 기능입니다.

## Reference

- [Schedule Recurring Tasks in Cowork](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-cowork)
