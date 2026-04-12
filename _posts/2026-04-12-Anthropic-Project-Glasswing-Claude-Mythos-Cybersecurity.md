---
layout: post
title: "Anthropic Project Glasswing 출범, Claude Mythos Preview로 사이버보안 취약점 자동 탐지"
author: 'Juho'
date: 2026-04-12 00:00:00 +0900
categories: [AI]
tags: [AI, Security, Benchmark]
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
2. [Project Glasswing이란](#project-glasswing이란)
   - [참여 파트너](#참여-파트너)
   - [핵심 모델 Claude Mythos Preview](#핵심-모델-claude-mythos-preview)
3. [벤치마크 성능](#벤치마크-성능)
4. [발견된 취약점 사례](#발견된-취약점-사례)
5. [배포와 가격 정책](#배포와-가격-정책)
6. [의미와 시사점](#의미와-시사점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Anthropic이 Project Glasswing이라는 사이버보안 이니셔티브를 공개했다.
이 프로젝트는 미공개 프런티어 모델인 Claude Mythos Preview를 활용하여 핵심 소프트웨어 인프라의 취약점을 악성 행위자보다 먼저 식별하고 패치하는 것을 목표로 한다.
12개의 출범 파트너와 40여 개의 추가 인프라 운영 조직이 모델에 대한 접근 권한을 부여받았다.

## Project Glasswing이란

Glasswing은 Anthropic이 주도하는 방어 중심 사이버보안 프로그램이다.
프런티어 AI 모델의 코드 분석 능력을 활용해 운영체제, 브라우저, 핵심 라이브러리에 잠재된 고위험 취약점을 자동으로 발견하고 보고하는 데 초점을 맞춘다.
Anthropic은 "방어자가 앞서 나가야 한다(defenders must come out ahead)"는 문구로 프로젝트의 방향성을 표현했다.

### 참여 파트너

12개의 출범 파트너는 클라우드, 운영체제, 보안, 금융 인프라 등 여러 분야를 망라한다.

| 분류 | 파트너 |
|------|--------|
| 클라우드 | Amazon Web Services, Google, Microsoft |
| 운영체제/디바이스 | Apple |
| 네트워킹/보안 | Broadcom, Cisco, CrowdStrike, Palo Alto Networks |
| 금융 | JPMorganChase |
| AI/하드웨어 | Anthropic, NVIDIA |
| 오픈소스 | Linux Foundation |

이 외에도 핵심 소프트웨어 인프라를 유지보수하는 40여 개 조직이 모델 접근 권한을 받았다.

### 핵심 모델 Claude Mythos Preview

Claude Mythos Preview는 Anthropic이 아직 일반 공개하지 않은 프런티어 모델이다.
공식 발표에 따르면 이 모델은 "주요 운영체제와 모든 웹 브라우저를 포함하여 수천 개의 고위험(high-severity) 취약점을 발견했다".
Anthropic은 일반 공개 일정을 잡지 않은 채, 이후 출시될 Claude Opus 모델에 새 사이버보안 안전장치를 함께 배포한 뒤에 더 넓은 활용을 검토하겠다는 입장을 밝혔다.

## 벤치마크 성능

Mythos Preview는 사이버보안 및 코드 관련 벤치마크에서 기존 Claude Opus 4.6 대비 큰 폭의 향상을 보였다.

| 벤치마크 | Mythos Preview | Claude Opus 4.6 |
|----------|----------------|-----------------|
| CyberGym (Vulnerability Reproduction) | 83.1% | 66.6% |
| SWE-bench Pro | 77.8% | 53.4% |
| SWE-bench Verified | 93.9% | 80.8% |
| Terminal-Bench 2.0 | 82.0% | 65.4% |

특히 CyberGym은 실제 취약점 재현 능력을 평가하는 벤치마크로, Mythos Preview는 16.5%포인트 향상되었다.
SWE-bench Verified에서도 90% 선을 넘기며 코드 수정 능력 자체가 한 단계 진전했음을 보여준다.

## 발견된 취약점 사례

Mythos Preview는 자동 탐지 과정에서 장기간 발견되지 않았던 취약점들을 독립적으로 찾아냈다.

OpenBSD에서 27년간 잠재되어 있던 핵심 인프라 영향 취약점이 발견되었다.
FFmpeg에서는 수백만 건의 자동화된 테스트가 잡지 못했던 16년 묵은 결함이 드러났다.
또한 Linux 커널에서 권한 상승(privilege escalation)으로 이어지는 다수의 취약점이 식별되었다.

이러한 발견 사례는 모델이 단순히 패턴 매칭이 아니라 깊은 코드 분석과 추론을 통해 취약점을 찾고 있음을 시사한다.

## 배포와 가격 정책

Mythos Preview는 Claude API, Amazon Bedrock, Google Cloud Vertex AI, Microsoft Foundry를 통해 파트너에게 제공된다.
가격은 입력 100만 토큰당 25달러, 출력 100만 토큰당 125달러로 책정되었다.

Anthropic은 프로그램 활성화를 위해 다음과 같은 자금을 약속했다.

| 항목 | 금액 |
|------|------|
| 참여자 모델 사용 크레딧 | 1억 달러 |
| Alpha-Omega 및 OpenSSF | 250만 달러 |
| Apache Software Foundation | 150만 달러 |

오픈소스 보안 재단에 직접 자금을 투입한다는 점에서, 단순한 모델 판매가 아니라 생태계 차원의 방어력 강화를 노린 구조이다.

활용 사례로는 로컬 취약점 탐지, 블랙박스 바이너리 테스트, 엔드포인트 보안 강화, 침투 테스트, 오픈소스 소프트웨어 패치 제안 등이 제시되었다.

## 의미와 시사점

Glasswing 프로젝트는 프런티어 AI 모델의 양면성을 정면으로 다룬다.
같은 기술이 방어자에게는 인프라 보호 도구이지만, 공격자에게는 자동화된 익스플로잇 발견 도구가 될 수 있다.
Anthropic이 일반 공개를 미루고 제한된 파트너에게만 모델을 푼 것은 이 위험을 인식한 결과로 읽힌다.

27년, 16년 묵은 취약점이 자동으로 발견된다는 사실은 기존 정적 분석 도구와 퍼저(fuzzer)가 도달하지 못했던 영역에 LLM 기반 분석이 새로운 진입로를 열었음을 보여준다.
동시에 동일 모델이 공격 측에 흘러갈 경우 방어 측이 패치를 적용하기 전에 대규모 익스플로잇이 발생할 수 있다는 우려도 함께 따른다.

## 결론

Project Glasswing은 프런티어 AI를 사이버보안 방어에 본격적으로 투입한 첫 번째 대규모 산업 협력 프로그램이다.
Claude Mythos Preview의 벤치마크 성능과 실제 발견 사례는 LLM이 보안 연구의 보조 도구를 넘어 1차 탐지자 역할로 전환되는 시점이 다가오고 있음을 시사한다.
다만 모델 자체의 이중 용도 위험과 공개 시점 조율은 여전히 풀어야 할 숙제로 남는다.

## Reference

- [Anthropic Project Glasswing](https://www.anthropic.com/glasswing)
