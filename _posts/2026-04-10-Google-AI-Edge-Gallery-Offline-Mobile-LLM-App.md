---
layout: post
title: "Google AI Edge Gallery: 모바일에서 Gemma 4를 완전 오프라인으로 실행하는 앱"
author: 'Juho'
date: 2026-04-10 00:00:00 +0900
categories: [AI]
tags: [AI, LLM]
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
   - [AI 채팅 및 사고 모드](#ai-채팅-및-사고-모드)
   - [에이전트 스킬](#에이전트-스킬)
   - [멀티모달 기능](#멀티모달-기능)
   - [추가 기능](#추가-기능)
3. [기술 스택 및 지원 환경](#기술-스택-및-지원-환경)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Google AI Edge Gallery는 인터넷 연결 없이 완전 오프라인 환경에서 LLM을 구동하는 iOS/Android 앱이다.
Gemma 4 패밀리 모델을 공식 지원하며, 모든 데이터가 기기에서만 처리되어 프라이버시가 보장된다.
Apache-2.0 라이선스로 공개된 오픈소스 프로젝트이다.

## 주요 기능

### AI 채팅 및 사고 모드

다중 턴 대화가 가능한 AI 챗봇 기능을 제공한다.
새로운 Thinking Mode를 통해 모델의 단계별 추론 과정을 시각적으로 확인할 수 있다.
추론 과정이 투명하게 공개되어 모델의 판단 근거를 이해하는 데 도움이 된다.

### 에이전트 스킬

LLM의 능력을 확장하는 에이전트 스킬 시스템을 탑재했다.
Wikipedia 검색, 인터랙티브 지도 등의 도구를 활용할 수 있다.
GitHub Discussions에서 커뮤니티가 만든 스킬도 로드하여 사용할 수 있다.
기기 내에서 모델이 도구를 호출하고 결과를 활용하는 구조이다.

### 멀티모달 기능

이미지와 음성을 포함한 다양한 입력을 처리할 수 있다.

| 기능 | 설명 |
|------|------|
| Ask Image | 카메라나 갤러리의 이미지를 분석하고 질문에 답변 |
| Audio Scribe | 음성 녹음을 실시간으로 전사 및 번역 |

모든 멀티모달 처리가 기기 내에서 이루어지므로 민감한 이미지나 음성 데이터가 외부로 전송되지 않는다.

### 추가 기능

다양한 부가 기능이 포함되어 있다.

| 기능 | 설명 |
|------|------|
| Prompt Lab | 온디바이스 파라미터 조정이 가능한 프롬프트 테스트 공간 |
| Mobile Actions | FunctionGemma 270m 기반 오프라인 기기 제어 자동화 |
| Tiny Garden | 자연어 기반 미니게임 |
| Model Management | 모델 다운로드, 관리, 하드웨어별 벤치마크 테스트 |

## 기술 스택 및 지원 환경

| 항목 | 내용 |
|------|------|
| 개발 언어 | Kotlin |
| 핵심 기술 | Google AI Edge API, LiteRT(경량 런타임) |
| 모델 소스 | Hugging Face 통합 |
| Android | 12 이상 |
| iOS | 17 이상 |
| 라이선스 | Apache-2.0 |

Google Play, App Store, GitHub 릴리스에서 다운로드할 수 있다.

## 의미와 시사점

Google AI Edge Gallery는 모바일 AI의 방향성을 보여주는 프로젝트이다.
클라우드 의존 없이 기기 내에서 모든 추론을 처리하므로, 네트워크가 불안정한 환경에서도 AI 기능을 활용할 수 있다.
프라이버시가 중요한 의료, 법률, 기업 환경에서 특히 유용하다.
에이전트 스킬 시스템은 로컬 LLM에도 도구 활용 능력을 부여하여, 단순한 대화를 넘어 실용적인 작업 수행이 가능하게 한다.

## 결론

Google AI Edge Gallery는 모바일 기기에서 Gemma 4 모델을 완전 오프라인으로 실행할 수 있는 실용적인 앱이다.
에이전트 스킬, 사고 모드, 멀티모달 기능까지 갖추고 있어 데스크톱 AI 도구 못지않은 기능성을 제공한다.
오픈소스로 공개되어 있어 커뮤니티의 기여와 확장이 가능하다는 점도 장점이다.

## Reference

- [Google AI Edge Gallery - GitHub](https://github.com/google-ai-edge/gallery/)
