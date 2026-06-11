---
layout: post
title: "Microsoft Scout - 상시 작동하는 개인 AI 에이전트"
author: 'Juho'
date: 2026-06-11 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM]
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
   - [일정 및 협업 관리](#일정-및-협업-관리)
   - [통합 플랫폼 범위](#통합-플랫폼-범위)
3. [기술 아키텍처](#기술-아키텍처)
   - [인프라 구성](#인프라-구성)
   - [보안 및 컴플라이언스](#보안-및-컴플라이언스)
4. [이용 가능 여부](#이용-가능-여부)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Microsoft는 2026년 6월 2일 [Microsoft Scout](https://www.microsoft.com/en-us/microsoft-365/blog/2026/06/02/introducing-microsoft-scout-your-always-on-personal-agent/){:target="_blank"}를 공개했다.
Scout는 Microsoft의 첫 번째 Autopilot 에이전트로, 사용자가 반복적으로 명령을 입력하지 않아도 자율적으로 작동하며 사용자를 대신해 업무를 수행하는 상시 AI 에이전트 카테고리에 속한다.
Scout는 하루 동안 누적되는 업무 조율 작업을 처리하고, 사용자의 주의가 다른 곳에 쏠려 있는 동안에도 업무 연속성을 유지한다.

## 핵심 기능

### 일정 및 협업 관리

Scout는 다음과 같은 업무를 자율적으로 수행한다.

- 시간대를 고려하여 회의 시간을 예약하고 조율한다.
- 중요한 회의에 대한 알림을 제공하고 준비 자료를 자동으로 생성한다.
- 다가오는 마감 기한을 파악하고 캘린더에 작업 시간을 자동으로 확보한다.
- 결정이 지연되는 등 업무 진행의 리스크를 사전에 감지하여 차단 요인이 되기 전에 알린다.

### 통합 플랫폼 범위

Scout는 클라우드, 데스크톱, 웹을 아우르며 운영된다.
연결 대상 서비스는 Teams, Outlook, OneDrive, SharePoint이며, 채팅, 이메일, 캘린더, 연락처 등 사용자의 업무 데이터 전반과 연동된다.

## 기술 아키텍처

### 인프라 구성

Scout의 기술 기반은 다음과 같다.

| 구성 요소 | 설명 |
|-----------|------|
| OpenClaw | Scout가 기반으로 삼는 오픈소스 기술 |
| Work IQ | 사용자의 업무 패턴과 우선순위를 시간이 지남에 따라 학습하는 엔진 |
| Entra ID | 각 에이전트는 공유 서비스 계정이 아닌 자체 거버넌스된 Entra ID로 운영됨 |

### 보안 및 컴플라이언스

Scout는 엔터프라이즈 수준의 보안과 접근 통제를 적용한다.
자격 증명은 특정 작업 범위로 제한되며 로그에서 삭제된다.
Microsoft Purview 데이터 보호 정책과 민감도 레이블을 준수한다.
민감한 작업에는 사람의 최종 승인이 필요하며, 조직의 기존 접근 통제 정책을 그대로 적용한다.

## 이용 가능 여부

현재 Scout는 다음 경로를 통해 이용할 수 있다.

- 선정된 고객 대상 비공개 프리뷰
- Frontier 프로그램(실험적 릴리스)

Frontier 프로그램을 통한 이용은 Frontier 등록, Intune 정책 구성, GitHub Copilot 라이선스, 그리고 opt-in 동의가 필요하다.

## 결론

Microsoft Scout는 반복 명령 없이 자율적으로 업무를 처리하는 새로운 개인 AI 에이전트 카테고리를 제시한다.
Teams, Outlook, OneDrive, SharePoint와의 통합, Work IQ 기반의 패턴 학습, Entra ID 기반의 독립 거버넌스, Microsoft Purview 준수를 통해 개인 생산성과 기업 보안을 동시에 지원한다.
현재는 비공개 프리뷰와 Frontier 프로그램을 통해 제한적으로 제공되고 있다.

## Reference

- [Introducing Microsoft Scout: Your always-on personal agent](https://www.microsoft.com/en-us/microsoft-365/blog/2026/06/02/introducing-microsoft-scout-your-always-on-personal-agent/)
