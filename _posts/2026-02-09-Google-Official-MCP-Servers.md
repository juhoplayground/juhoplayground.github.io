---
layout: post
title: Google 공식 MCP 서버 모음 - Google Cloud 서비스 통합
author: 'Juho'
date: 2026-02-22 01:00:00 +0900
categories: [MCP]
tags: [MCP, GCP, Google Cloud]
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
2. [원격 MCP 서버](#원격-mcp-서버)
3. [오픈소스 MCP 서버](#오픈소스-mcp-서버)
4. [MCP Toolbox for Databases](#mcp-toolbox-for-databases)
5. [예제 프로젝트](#예제-프로젝트)
6. [배포 옵션](#배포-옵션)
7. [정리](#정리)
8. [Reference](#reference)

## 개요

Google이 공식 MCP(Model Context Protocol) 서버 모음을 GitHub에 공개했다.
이 저장소는 Google이 관리하는 원격 및 오픈소스 MCP 서버들의 공식 모음이다.
Google Cloud 서비스와의 통합, 배포 가이드, 시작을 위한 예제를 제공한다.

라이선스는 Apache 2.0이며, 데모 및 학습 목적으로 제공되는 참고용 저장소다.
공식 Google 제품이 아님에 유의해야 한다.

## 원격 MCP 서버

Google이 관리하는 원격 MCP 서버는 주요 Google Cloud 서비스와 연동된다.

### 지원 서비스

| 서비스 | 설명 |
|--------|------|
| Google Maps | Grounding Lite 기반 지도 서비스 |
| BigQuery | 대규모 데이터 분석 |
| GKE | Kubernetes Engine |
| GCE | Compute Engine |

## 오픈소스 MCP 서버

로컬 실행 또는 Google Cloud 배포가 가능한 15개 이상의 오픈소스 MCP 서버를 제공한다.

### Google Workspace 통합

- Google Docs
- Google Sheets
- Google Slides
- Google Calendar
- Gmail

### Google Cloud 서비스

- Firebase
- Cloud Run
- Google Analytics
- Google Cloud Storage
- gcloud CLI

### AI 및 미디어

- Genmedia (Imagen/Veo 모델)
- Google Cloud Security 도구

### 개발 도구

- Flutter/Dart
- Chrome DevTools

## MCP Toolbox for Databases

MCP Toolbox는 데이터베이스 관리를 위한 도구다.
다음 서비스를 지원한다.

- BigQuery
- Cloud SQL
- Spanner

## 예제 프로젝트

### Launch My Bakery

Agent Development Kit(ADK) 기반 MCP 서버 활용 방식을 보여주는 샘플 에이전트다.
원격 MCP 서버를 사용하여 지도 및 데이터 분석 통합을 시연한다.

## 배포 옵션

MCP 서버를 Google Cloud에 배포하는 방법을 안내한다.

### Cloud Run 배포

Cloud Run에서 MCP 서버를 빠르게 배포할 수 있다.
블로그 및 코드랩 자료가 제공된다.

### GKE 배포

Kubernetes Engine에서 MCP 서버를 운영할 수 있다.
멀티 에이전트 아키텍처 가이드가 포함되어 있다.

### 보안 구현

안전한 MCP 서버 구현을 위한 코드랩이 제공된다.

## 정리

Google의 공식 MCP 서버 모음은 Google Cloud 서비스와 AI 에이전트의 통합을 쉽게 만든다.
원격 서버로 Google Maps, BigQuery, GKE, GCE 등을 바로 사용할 수 있다.
오픈소스 서버로 Google Workspace, Firebase, Analytics 등 다양한 서비스와 통합할 수 있다.
Cloud Run이나 GKE에 배포하여 프로덕션 환경에서 운영할 수 있다.
Launch My Bakery 예제를 통해 MCP 서버 활용 방식을 학습할 수 있다.

## Reference

- [Google MCP GitHub Repository](https://github.com/google/mcp)
