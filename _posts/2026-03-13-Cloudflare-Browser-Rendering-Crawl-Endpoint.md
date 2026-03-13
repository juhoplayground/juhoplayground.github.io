---
layout: post
title: "Cloudflare, Browser Rendering /crawl 엔드포인트 공개 베타 출시"
author: 'Juho'
date: 2026-03-13 00:00:00 +0900
categories: [Dev]
tags: [Dev, AI]
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
3. [핵심 내용](#핵심-내용)
   - [다중 출력 형식](#다중-출력-형식)
   - [크롤 범위 제어](#크롤-범위-제어)
   - [증분 크롤링](#증분-크롤링)
   - [정적 모드](#정적-모드)
   - [규칙 준수](#규칙-준수)
   - [API 사용 방법](#api-사용-방법)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Cloudflare가 Browser Rendering 서비스에 새로운 /crawl 엔드포인트를 공개 베타로 출시했다.
단 한 번의 API 호출로 전체 웹사이트를 탐색할 수 있으며, HTML, Markdown, JSON 형식으로 결과를 반환한다.
모델 학습 데이터 수집, RAG 파이프라인 구축, 사이트 모니터링 등 다양한 분야에서 활용할 수 있다.
Free/Paid 플랜 모두에서 사용 가능하다.

## 배경

웹 크롤링은 AI 모델 학습 데이터 수집이나 RAG 파이프라인 구축 등에서 핵심적인 역할을 한다.
기존에는 크롤링을 위해 별도의 도구나 복잡한 설정이 필요했지만, Cloudflare는 Browser Rendering 서비스에 /crawl 엔드포인트를 추가하여 이를 단순화했다.
사이트맵과 페이지 링크에서 자동으로 URL을 탐지하는 Discovery Methods를 제공하여 크롤링 시작을 쉽게 할 수 있다.

## 핵심 내용

### 다중 출력 형식

/crawl 엔드포인트는 HTML, Markdown, 구조화된 JSON 형식을 지원한다.
필요에 따라 원하는 형식으로 크롤링 결과를 받을 수 있다.

### 크롤 범위 제어

크롤링의 깊이, 페이지 수, URL 패턴 필터링을 설정할 수 있다.
Scope Management 기능을 통해 불필요한 페이지를 제외하고 원하는 범위만 크롤링할 수 있다.

### 증분 크롤링

modifiedSince, maxAge 파라미터를 활용하여 변경되지 않은 페이지를 건너뛸 수 있다.
이를 통해 중복 처리를 줄이고 비용을 절감할 수 있다.

### 정적 모드

render: false 설정으로 브라우저 렌더링을 우회할 수 있다.
브라우저 렌더링 없이 빠른 크롤링이 가능하다.

### 규칙 준수

robots.txt 디렉티브와 crawl-delay 설정을 준수한다.
웹사이트 소유자가 설정한 크롤링 규칙을 자동으로 따른다.

### API 사용 방법

크롤링은 비동기로 실행된다.
{% raw %}`POST /client/v4/accounts/{account_id}/browser-rendering/crawl`{% endraw %}로 크롤을 시작하면 job ID를 받게 된다.
{% raw %}`GET /client/v4/accounts/{account_id}/browser-rendering/crawl/{job_id}`{% endraw %}로 결과를 확인할 수 있다.
URL을 제출한 후 job ID를 받고, 이후 결과를 폴링하는 방식으로 동작한다.

## 의미와 시사점

커뮤니티에서는 Cloudflare 보호 페이지에서는 이 크롤링이 작동하지 않을 수 있다는 우려가 있다.
또한 웹사이트 보호 서비스와 크롤링 서비스를 동시에 제공하는 Cloudflare의 이중성을 지적하는 의견도 있다.
robots.txt 준수로 인한 크롤링 범위의 제약이 존재할 수 있다.

## 결론

Cloudflare의 Browser Rendering /crawl 엔드포인트는 단일 API 호출로 웹사이트 전체를 크롤링할 수 있는 간편한 방법을 제공한다.
다중 출력 형식, 범위 제어, 증분 크롤링, 정적 모드 등 다양한 기능을 지원하며, Free/Paid 플랜 모두에서 사용할 수 있다.
다만 Cloudflare 보호 페이지에서의 제한이나 robots.txt 준수로 인한 제약은 고려해야 할 사항이다.

## Reference

- [Browser Rendering's /crawl endpoint](https://developers.cloudflare.com/changelog/post/2026-03-10-br-crawl-endpoint/)
