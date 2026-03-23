---
layout: post
title: "OpenJarvis - 스탠포드가 만든 로컬 디바이스 개인용 AI 스택"
author: 'Juho'
date: 2026-03-23 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM, Ollama]
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
2. [핵심 아이디어](#핵심-아이디어)
3. [기술 스택](#기술-스택)
4. [설치 및 사용법](#설치-및-사용법)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

OpenJarvis는 스탠포드 대학의 Scaling Intelligence Lab에서 개발한 오픈소스 프로젝트이다.
"Personal AI, On Personal Devices"라는 슬로건 아래, 클라우드 API에 의존하지 않고 로컬에서 실행되는 개인용 AI 에이전트를 구축할 수 있도록 설계되었다.

로컬 언어 모델이 단일 턴 채팅과 추론 쿼리의 88.7%를 처리할 수 있으며, 2023년부터 2025년 사이에 지능 효율성이 5.3배 개선되었다는 기술적 배경을 기반으로 한다.

프로젝트는 GitHub에서 Stars 1.1k, Forks 162를 기록하고 있으며, Apache 2.0 라이선스로 공개되어 있다.
스탠포드 SAIL, Hazy Research, Scaling Intelligence Lab 소속이며, Laude Institute, Stanford Marlowe, Google Cloud Platform, Lambda Labs, Ollama, IBM Research, Stanford HAI 등이 스폰서로 참여하고 있다.

## 핵심 아이디어

OpenJarvis는 세 가지 핵심 아이디어를 중심으로 설계되었다.

### 공유 프리미티브

온디바이스 에이전트 구축을 위한 표준화된 기본 단위를 제공한다.
이를 통해 개발자들이 일관된 방식으로 로컬 AI 에이전트를 구성할 수 있다.

### 평가 체계

에너지, FLOPs, 지연시간, 비용을 정확성과 동등하게 고려하는 평가 체계를 갖추고 있다.
단순히 모델의 정확도만 추구하는 것이 아니라, 실제 디바이스에서의 실용성을 함께 측정한다.

### 학습 루프

로컬 추적 데이터를 활용한 지속적 모델 개선이 가능하다.
사용자의 디바이스에서 발생하는 데이터를 기반으로 모델이 점진적으로 개선되는 구조이다.

## 기술 스택

OpenJarvis의 코드베이스는 다음과 같은 언어 비율로 구성되어 있다.

| 언어 | 비율 | 용도 |
|:---:|:---:|:---:|
| Python | 77.6% | 핵심 로직 |
| Rust | 14.4% | 성능 최적화 |
| TypeScript | 7.6% | 프론트엔드 |

지원하는 로컬 추론 백엔드는 다음과 같다.

| 백엔드 |
|:---:|
| Ollama |
| vLLM |
| SGLang |
| llama.cpp |

## 설치 및 사용법

### 설치

```bash
git clone https://github.com/open-jarvis/OpenJarvis.git
cd OpenJarvis
uv sync
```

### 빠른 시작 (Ollama 기준)

OpenJarvis를 클론하고 설치한 뒤, 아래 단계를 따른다.

먼저 하드웨어 자동 감지 및 설정을 수행한다.

```bash
uv run jarvis init
```

Ollama를 설치하고 실행한 뒤, 원하는 모델을 다운로드한다.

```bash
ollama pull qwen3:8b
```

쿼리를 실행하여 AI에게 질문할 수 있다.

```bash
uv run jarvis ask "질문"
```

설정에 문제가 있는지 진단하려면 다음 명령어를 사용한다.

```bash
uv run jarvis doctor
```

## 결론

OpenJarvis는 클라우드에 의존하지 않는 로컬 기반 개인용 AI 에이전트 구축을 목표로 하는 프로젝트이다.
스탠포드 대학의 Scaling Intelligence Lab에서 개발되었으며, 공유 프리미티브, 실용적 평가 체계, 로컬 학습 루프라는 세 가지 핵심 아이디어를 중심으로 설계되었다.
Ollama, vLLM, SGLang, llama.cpp 등 다양한 로컬 추론 백엔드를 지원하므로, 자신의 환경에 맞는 백엔드를 선택하여 활용할 수 있다.

## Reference

- [OpenJarvis GitHub](https://github.com/open-jarvis/OpenJarvis/){:target="_blank"}
