---
layout: post
title: "Voicebox, 내 컴퓨터에서 돌아가는 오픈소스 음성 합성 스튜디오"
author: 'Juho'
date: 2026-05-07 00:00:00 +0900
categories: [AI]
tags: [AI, Python, FastAPI]
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
   - [통합 TTS 엔진과 다국어 지원](#통합-tts-엔진과-다국어-지원)
   - [기술 스택과 GPU 가속](#기술-스택과-gpu-가속)
   - [API와 설치 방법](#api와-설치-방법)
4. [의미와 시사점](#의미와-시사점)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

박재홍이 위키독스에 공개한 [Voicebox 소개 글](https://wikidocs.net/blog/@jaehong/12235/){:target="_blank"}은 클라우드 의존도를 낮춘 로컬 우선 음성 합성 도구를 소개한다.
Voicebox는 텍스트를 입력하면 사람과 유사한 음성을 만들어주는 오픈소스 프로젝트이며, 데이터가 외부 서버로 나가지 않는 점이 핵심 차별점이다.
글은 ElevenLabs 같은 클라우드 유료 서비스의 대안으로 Voicebox가 어디까지 왔는지를 정리한다.

## 배경

음성 합성은 그동안 ElevenLabs를 비롯한 SaaS형 서비스가 시장을 주도해왔다.
사용량 단위 과금, 데이터 외부 전송, 사용량 제한 같은 클라우드 모델 특유의 제약이 따른다.
저자는 "오픈소스 TTS 엔진의 품질이 일정 수준을 넘으면서, 음성 합성이 클라우드에서 로컬 도구로 내려올 수 있는 시점이 왔다"고 평가한다.
Voicebox는 그 흐름을 한 데 묶어 데스크톱 스튜디오 형태로 제공하려는 시도다.

## 핵심 내용

### 통합 TTS 엔진과 다국어 지원

Voicebox는 자체 TTS 모델을 새로 만들지 않는다.
대신 이미 공개된 우수한 오픈소스 엔진들을 하나의 인터페이스로 묶는다.
포함된 엔진은 Qwen3, LuxTTS, Chatterbox, Kokoro 등 7종이며, 23개 언어를 지원한다.
음성 복제, 후처리 이펙트, 멀티트랙 편집까지 내장해 단순 음성 생성기를 넘어선 음성 편집 스튜디오를 지향한다.

### 기술 스택과 GPU 가속

기술 스택은 데스크톱 앱과 백엔드 API가 분리된 구조다.
프론트엔드는 React, TypeScript, Tailwind CSS로 구성되며 Tauri 프레임워크 위에서 네이티브 앱으로 패키징된다.
백엔드는 Python의 FastAPI가 담당한다.
가속 백엔드는 다양한 하드웨어를 폭넓게 지원한다.

| 항목 | 사용 기술 |
|------|-----------|
| 프론트엔드 | React, TypeScript, Tailwind CSS, Tauri |
| 백엔드 | FastAPI 기반 Python |
| GPU 가속 | CUDA, ROCm, MLX, IPEX |
| TTS 엔진 | Qwen3, LuxTTS, Chatterbox, Kokoro 등 7종 |
| 언어 지원 | 23개 언어 |

NVIDIA뿐 아니라 AMD ROCm, Apple Silicon용 MLX, Intel용 IPEX까지 커버하는 점이 특징이다.

### API와 설치 방법

Voicebox는 로컬 HTTP API 형태로 음성 생성을 제공한다.
다음과 같이 텍스트와 프로필 ID를 지정해 곧바로 호출할 수 있다.

```bash
# 음성 생성 API 호출
curl -X POST http://localhost:17493/generate \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world", "profile_id": "abc123", "language": "en"}'
```

설치는 GitHub 저장소를 클론한 뒤 just 빌드 도구로 처리한다.

```bash
# 설치 및 실행
git clone https://github.com/jamiepine/voicebox.git
just setup
just dev
```

## 의미와 시사점

저자가 강조하는 차이는 세 가지다.
비용은 무료, 데이터는 로컬에 머무르며, 사용량 제한이 없다.
프로토타이핑이나 기업 내부 용도, 즉 외부에 음성을 보내기 어려운 환경에서 특히 유리하다.
GPU 백엔드를 폭넓게 지원하기 때문에 사내 워크스테이션이나 Mac Studio 같은 실리콘 환경에서도 무리 없이 동작한다.
다만 단일 SaaS의 안정성과 일관된 품질을 기대하던 사용자에게는 7종 엔진 운영의 복잡성이 부담이 될 수 있다.

## 결론

Voicebox는 오픈소스 TTS 엔진 생태계가 성숙해진 현 시점을 가장 잘 활용한 프로젝트 중 하나다.
React, FastAPI, Tauri라는 흔한 스택 위에서 7개 TTS 엔진을 한 데 묶고 GPU 가속까지 통합했다.
음성 합성을 클라우드 외주로 두고 있던 팀이 내부 도구로 가져올 때 가장 먼저 검토할 만한 후보다.

## Reference

- [Voicebox, 내 컴퓨터에서 돌아가는 오픈소스 음성 합성 스튜디오 by 박재홍](https://wikidocs.net/blog/@jaehong/12235/){:target="_blank"}
