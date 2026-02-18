---
layout: post
title: "Bing Webmaster Tools, AI 답변 인용 추적 기능 공개 프리뷰"
author: 'Juho'
date: 2026-02-18 01:00:00 +0900
categories: [AI]
tags: [AI]
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
2. [AI Performance 대시보드란](#ai-performance-대시보드란)
3. [주요 지표](#주요-지표)
4. [GEO와 AI 검색 시대의 의미](#geo와-ai-검색-시대의-의미)
5. [콘텐츠 최적화 가이드](#콘텐츠-최적화-가이드)
6. [현재 상태와 제한 사항](#현재-상태와-제한-사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Microsoft가 Bing Webmaster Tools에 AI Performance 대시보드를 공개 프리뷰로 출시했다.
이 기능은 웹사이트 콘텐츠가 AI 생성 답변에서 얼마나 자주 인용되는지 추적할 수 있게 해준다.
Microsoft Copilot, Bing AI 요약 등 AI 서비스에서 콘텐츠가 소스로 참조되는 빈도를 확인할 수 있다.
생성형 AI 검색 시대에 콘텐츠 가시성을 측정할 수 있는 최초의 공개 도구라는 점에서 의미가 크다.

## AI Performance 대시보드란

AI Performance는 Bing Webmaster Tools에 새롭게 추가된 기능이다.
웹사이트 운영자가 자신의 콘텐츠가 AI 생성 답변에서 소스로 참조되는 현황을 확인할 수 있다.
기존 Search Console이 직접적인 클릭 수를 추적했다면, AI Performance는 AI 시스템이 응답 생성 시 콘텐츠를 인용하는 횟수를 보여준다.
지원되는 AI 경험 전반에 걸쳐 인용 데이터를 통합하여 제공하며, robots.txt 제한 사항을 준수한다.

## 주요 지표

AI Performance 대시보드에서 제공하는 핵심 지표는 다음과 같다.

### 총 인용 횟수 (Total Citations)

선택한 기간 동안 콘텐츠가 AI 답변의 소스로 사용된 총 횟수를 표시한다.

### 평균 인용 페이지 수 (Average Cited Pages)

AI 서비스 전반에서 매일 참조되는 고유 페이지 수를 보여준다.
시간 경과에 따라 집계된 수치로 제공된다.

### 그라운딩 쿼리 (Grounding Queries)

AI 시스템이 콘텐츠를 검색할 때 사용한 핵심 문구를 공개한다.
어떤 검색어로 인해 콘텐츠가 AI 답변에 포함되었는지 파악할 수 있다.

### 페이지별 인용 활동 (Page-Level Citation Activity)

특정 URL별로 인용 횟수를 세분화하여 보여준다.
어떤 페이지가 가장 많이 인용되는지 한눈에 확인 가능하다.

### 가시성 트렌드 (Visibility Trends)

시간에 따른 인용 패턴 변화를 타임라인 시각화로 제공한다.

### 지표 요약

| 지표 | 설명 |
|------|------|
| Total Citations | AI 답변에서 소스로 사용된 총 횟수 |
| Average Cited Pages | 일평균 참조 고유 페이지 수 |
| Grounding Queries | AI가 콘텐츠 검색에 사용한 핵심 문구 |
| Page-Level Activity | URL별 인용 횟수 세분화 |
| Visibility Trends | 인용 패턴 변화 타임라인 |

## GEO와 AI 검색 시대의 의미

AI 검색 시대에는 사용자가 링크를 클릭하지 않고도 AI 답변에서 직접 정보를 얻는다.
이로 인해 기존 SEO(Search Engine Optimization)만으로는 콘텐츠 영향력을 측정하기 어렵다.
GEO(Generative Engine Optimization)라는 새로운 개념이 등장했으며, AI 생성 답변에서의 인용 가시성이 중요한 지표가 되었다.

이전까지는 AI가 콘텐츠를 얼마나 참조하는지 확인할 방법이 없었다.
AI Performance 대시보드는 이러한 보이지 않는 AI 인용을 측정할 수 있게 해주는 첫 번째 공개 도구이다.
Google은 아직 이에 해당하는 기능을 발표하지 않은 상태이다.

## 콘텐츠 최적화 가이드

AI 답변에서 인용 가능성을 높이기 위해 다음과 같은 최적화를 권장한다.

### 전문성 강화

자주 인용되는 주제에 대해 전문적인 깊이를 강화해야 한다.
데이터와 출처를 활용하여 주장을 뒷받침하는 것이 중요하다.

### 구조 개선

명확한 제목, 표, FAQ 섹션 등을 활용하여 콘텐츠 구조를 개선한다.
AI 시스템이 정보를 쉽게 추출할 수 있는 형태로 작성해야 한다.

### 콘텐츠 신선도 유지

정기적인 업데이트를 통해 콘텐츠 신선도를 유지해야 한다.
[IndexNow](https://www.indexnow.org){:target="_blank"}를 활용하면 콘텐츠 변경 사항을 빠르게 알릴 수 있다.

### 이미지-텍스트 일관성

이미지와 텍스트 간의 일관성을 유지하는 것도 중요한 요소이다.

## 현재 상태와 제한 사항

AI Performance 기능은 현재 공개 프리뷰(Public Preview) 단계이다.
Bing 및 Microsoft Copilot 생태계에 한정되어 있다.
Google의 동등한 기능은 아직 발표되지 않았다.
로컬 비즈니스의 경우 [Bing Places for Business](https://www.bingplaces.com){:target="_blank"} 등록도 권장된다.

## 결론

Bing Webmaster Tools의 AI Performance 대시보드는 AI 검색 시대의 콘텐츠 가시성 측정에 있어 중요한 첫 걸음이다.
웹사이트 운영자는 이 도구를 통해 AI 답변에서 콘텐츠가 어떻게 인용되는지 파악하고, GEO 전략을 수립할 수 있다.
아직 Bing/Microsoft 생태계에 한정되어 있지만, AI 검색 최적화의 방향성을 제시하는 의미 있는 기능이다.

## Reference

- [Introducing AI Performance in Bing Webmaster Tools Public Preview](https://blogs.bing.com/webmaster/February-2026/Introducing-AI-Performance-in-Bing-Webmaster-Tools-Public-Preview)
