---
layout: post
title: "Gemini 앱이 Docs Sheets Slides DOCX XLSX PDF를 직접 생성하기 시작했다"
author: 'Juho'
date: 2026-05-04 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Documentation]
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
2. [발표 내용](#발표-내용)
   - [지원 파일 포맷](#지원-파일-포맷)
   - [사용 시나리오](#사용-시나리오)
3. [출시 범위와 출력 옵션](#출시-범위와-출력-옵션)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google이 Gemini 앱에서 직접 파일을 생성하는 기능을 글로벌 출시했다.
사용자가 프롬프트만 입력하면 Gemini가 Google Docs, Sheets, Slides는 물론 Microsoft Word(.docx), Excel(.xlsx), PDF, CSV, LaTeX, TXT, RTF, Markdown 형식의 완성된 파일을 만들어 준다.
기존에는 Gemini의 응답을 복사해서 다른 도구로 옮긴 뒤 다시 포맷을 정리해야 했는데, 이 단계를 모두 제거한 것이 핵심이다.
Google은 [공식 블로그](https://blog.google/innovation-and-ai/products/gemini-app/generate-files-in-gemini/){:target="_blank"}에서 "브레인스토밍에서 완성된 파일까지 Gemini 앱을 떠나지 않고 빠르게 이동할 수 있다"고 설명한다.

## 발표 내용

이번 업데이트는 Gemini 앱을 단순한 답변기에서 산출물 제작 도구로 확장하는 변화다.
이전에도 Gemini는 텍스트, 표, 코드 등을 응답으로 보여줄 수 있었지만 실제 업무에서 쓰려면 파일로 옮기는 추가 작업이 필요했다.
이번 업데이트는 그 마지막 단계를 Gemini 자체에서 처리한다.

### 지원 파일 포맷

Gemini 앱이 직접 생성할 수 있는 파일 포맷은 크게 세 갈래로 나뉜다.

| 분류 | 지원 포맷 |
|------|------|
| Google Workspace | Docs, Sheets, Slides |
| Microsoft Office | Word(.docx), Excel(.xlsx) |
| 범용 포맷 | PDF, CSV, LaTeX, TXT, RTF, Markdown |

Workspace 내부 포맷뿐 아니라 Office 계열과 범용 텍스트 포맷까지 함께 지원한다는 점이 특징이다.
조직마다 다른 도구를 쓰더라도 Gemini의 출력물을 곧바로 받을 수 있도록 설계되어 있다.

### 사용 시나리오

Google이 공식 글에서 든 사례는 세 가지다.

| 시나리오 | 설명 |
|------|------|
| 예산안 → Excel | 예산 제안을 그대로 .xlsx 스프레드시트로 추출 |
| 아이디어 → 초안 | 흩어진 아이디어를 불릿 형태의 초안 문서로 정리 |
| 긴 협업 기록 → 1쪽 PDF | 길게 이어진 대화나 자료를 한 페이지 PDF나 Word 문서로 통합 |

세 사례 모두 "프롬프트 한 번에 완성형 파일이 나오는" 패턴을 공통으로 보여준다.
복사·붙여넣기와 재포매팅 작업을 줄이는 것이 이번 기능의 핵심 가치다.

## 출시 범위와 출력 옵션

이번 기능은 글로벌 단위로 모든 Gemini 앱 사용자에게 동시 제공된다.
"이 기능은 이제 전 세계 모든 Gemini 앱 사용자에게 제공된다"는 것이 Google의 공식 안내다.

생성된 파일은 두 가지 경로로 활용할 수 있다.

| 출력 옵션 | 설명 |
|------|------|
| 디바이스 다운로드 | 대부분의 포맷에서 로컬 디바이스로 직접 다운로드 |
| Google Drive 내보내기 | Drive로 곧바로 내보내 기존 협업 환경에 합류 |

이 구조 덕분에 Workspace 사용자가 아니더라도 Gemini의 산출물을 자기 업무 흐름으로 가져갈 수 있다.
공식 글에서는 파일 크기, 복잡도, 사용량 같은 별도 제약은 명시되지 않았다.

## 의미와 시사점

Gemini의 이번 변화는 LLM 챗봇이 "답변자"에서 "산출물 생산자"로 이동하는 흐름의 또 다른 예시다.
사용자가 ChatGPT, Claude, Gemini 같은 도구에서 가장 자주 하는 실제 작업이 결국 문서·시트·자료 정리라는 점을 감안하면, 이 기능은 일상 업무 시나리오를 그대로 한 번의 프롬프트로 압축한다.

특히 Microsoft Office 포맷(.docx, .xlsx)을 함께 지원한다는 점이 눈에 띈다.
Workspace 진영 외부 사용자에게도 Gemini 출력을 즉시 활용할 수 있게 만들기 때문에, Microsoft 365 환경에서 일하는 사용자도 Gemini를 도큐먼트 드래프트 도구로 끌어들일 여지가 생긴다.
Geek News 큐레이션도 "문서 요약, 데이터 분석, 보고서·자료 초안 작성 같은 반복 업무를 자동화해 생산성을 높이는 데 초점이 맞춰져 있다"고 정리한다.

다만 출시 발표 글은 한계나 제한 사항을 거의 다루지 않았다.
대용량 데이터에서 어떻게 동작하는지, 복잡한 표 구조의 정확도는 어느 정도인지, 디자인 일관성은 어떻게 유지되는지 등은 실제 사용 사례에서 확인할 영역으로 남는다.

## 결론

Gemini 앱의 직접 파일 생성은 LLM 워크플로의 마지막 1마일을 좁히는 변화다.
Workspace · Office · 범용 텍스트 포맷을 모두 지원하고, 디바이스 다운로드와 Drive 내보내기를 동시에 제공하면서 글로벌 사용자에게 일괄 출시되었다.
앞으로 Gemini 앱을 다루는 방식은 "답변을 받는 곳"이 아니라 "초안을 만드는 곳"으로 점점 더 옮겨갈 가능성이 높다.

## Reference

- [Generate files in the Gemini app](https://blog.google/innovation-and-ai/products/gemini-app/generate-files-in-gemini/)
