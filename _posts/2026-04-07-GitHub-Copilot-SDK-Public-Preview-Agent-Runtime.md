---
layout: post
title: "GitHub Copilot SDK 퍼블릭 프리뷰: 에이전트 런타임을 직접 통합하는 방법"
author: 'Juho'
date: 2026-04-07 00:00:00 +0900
categories: [Dev]
tags: [AI, Agent, Dev]
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
2. [Copilot SDK란 무엇인가](#copilot-sdk란-무엇인가)
3. [핵심 기능](#핵심-기능)
4. [지원 언어 및 설치 방법](#지원-언어-및-설치-방법)
5. [시스템 프롬프트 커스터마이징](#시스템-프롬프트-커스터마이징)
6. [권한 프레임워크와 보안](#권한-프레임워크와-보안)
7. [BYOK 지원](#byok-지원)
8. [가용성 및 요금 체계](#가용성-및-요금-체계)
9. [결론](#결론)

## 개요
GitHub이 Copilot SDK를 퍼블릭 프리뷰로 출시했다.
이 SDK는 GitHub Copilot 클라우드 에이전트와 Copilot CLI에서 사용되는 동일한 에이전트 런타임을 개발자에게 직접 제공한다.
개발자는 이제 자체 애플리케이션, 워크플로우, 플랫폼에 Copilot의 에이전틱 기능을 직접 통합할 수 있게 되었다.
이번 글에서는 Copilot SDK의 핵심 기능, 지원 언어, 설정 방법 등을 상세히 살펴본다.

## Copilot SDK란 무엇인가
Copilot SDK는 GitHub Copilot의 에이전트 런타임을 외부 애플리케이션에서 사용할 수 있도록 패키징한 소프트웨어 개발 키트이다.
기존에는 Copilot의 에이전틱 기능을 GitHub 생태계 내부에서만 사용할 수 있었다.
이제는 SDK를 통해 자체 서비스에 Copilot의 코드 생성, 도구 호출, 에이전트 실행 기능을 직접 통합할 수 있다.
GitHub Copilot 클라우드 에이전트와 Copilot CLI에서 사용되는 것과 동일한 런타임이므로, 프로덕션 수준의 안정성을 기대할 수 있다.

## 핵심 기능
Copilot SDK가 제공하는 주요 기능은 다음과 같다.

### 커스텀 도구 및 에이전트 정의
개발자가 직접 도구(tool)와 에이전트(agent)를 정의하여 Copilot 런타임에 등록할 수 있다.
특정 비즈니스 로직에 맞는 도구를 만들어 에이전트가 자동으로 호출하도록 설정할 수 있다.

### 토큰 단위 스트리밍
응답을 토큰 단위로 스트리밍하여 사용자에게 실시간으로 결과를 보여줄 수 있다.
대화형 인터페이스나 코드 생성 UI에서 사용자 경험을 크게 향상시킬 수 있는 기능이다.

### 이미지 및 바이너리 인라인 첨부
텍스트뿐만 아니라 이미지와 바이너리 파일을 인라인으로 첨부하여 전달할 수 있다.
멀티모달 작업이 필요한 경우에 유용하다.

### OpenTelemetry 분산 트레이싱
OpenTelemetry 기반의 분산 트레이싱을 기본 지원한다.
에이전트의 실행 흐름을 추적하고, 성능 병목을 파악하며, 디버깅에 활용할 수 있다.

### 민감 작업용 권한 프레임워크
민감한 작업에 대해 권한 프레임워크를 제공한다.
에이전트가 수행할 수 있는 작업의 범위를 제한하여 보안을 강화할 수 있다.

## 지원 언어 및 설치 방법
Copilot SDK는 5개 프로그래밍 언어를 지원한다.

| 언어 | 설치 명령어 |
|------|------------|
| Node.js / TypeScript | npm install @github/copilot-sdk |
| Python | pip install github-copilot-sdk |
| Go | go get github.com/github/copilot-sdk/go |
| .NET | dotnet add package GitHub.Copilot.SDK |
| Java | Maven 의존성 추가 |

주요 언어를 폭넓게 지원하므로, 기존 기술 스택에 맞춰 SDK를 선택할 수 있다.
Node.js/TypeScript와 Python이 가장 먼저 안정화될 것으로 예상된다.

## 시스템 프롬프트 커스터마이징
Copilot SDK는 시스템 프롬프트를 유연하게 수정할 수 있는 기능을 제공한다.
지원되는 수정 방식은 4가지이다.

| 방식 | 설명 |
|------|------|
| replace | 기존 시스템 프롬프트를 완전히 교체 |
| append | 기존 프롬프트 뒤에 내용 추가 |
| prepend | 기존 프롬프트 앞에 내용 추가 |
| transform | 기존 프롬프트를 변환 함수로 가공 |

이를 통해 에이전트의 행동 패턴, 응답 스타일, 도메인 특화 지식 등을 세밀하게 조정할 수 있다.
replace 방식은 완전히 새로운 시스템 프롬프트를 설정할 때 사용하고, append나 prepend는 기존 프롬프트를 유지하면서 추가 지침을 부여할 때 적합하다.
transform은 기존 프롬프트를 프로그래밍적으로 변환해야 할 때 활용할 수 있다.

## 권한 프레임워크와 보안
Copilot SDK는 민감한 작업에 대한 권한 프레임워크를 내장하고 있다.
에이전트가 파일 시스템 접근, 외부 API 호출, 코드 실행 등 민감한 작업을 수행할 때 권한 검증 단계를 거치도록 설정할 수 있다.
이를 통해 에이전트의 자율성과 보안 사이의 균형을 맞출 수 있다.

OpenTelemetry 분산 트레이싱과 결합하면, 에이전트가 어떤 권한을 요청했고 어떤 작업을 수행했는지 전체 이력을 추적할 수 있다.
프로덕션 환경에서 에이전트를 운영할 때 필수적인 기능이다.

## BYOK 지원
BYOK(Bring Your Own Key) 기능을 통해 GitHub Copilot 기본 모델 외에 다른 모델 제공자의 API 키를 사용할 수 있다.
지원되는 외부 제공자는 다음과 같다.

- OpenAI
- Azure AI Foundry
- Anthropic

자체 API 키를 사용하면 모델 선택의 자유도가 높아지고, 특정 모델에 대한 비용 관리를 직접 할 수 있다.
조직의 기존 AI 인프라와 통합하여 사용하기에도 적합하다.

## 가용성 및 요금 체계
Copilot SDK는 Copilot Free를 포함한 모든 GitHub Copilot 사용자가 접근할 수 있다.
무료 사용자도 SDK를 활용하여 에이전트 기반 애플리케이션을 구축할 수 있다는 점이 주목할 만하다.

사용량은 프리미엄 쿼타(premium quota) 기반으로 관리된다.
할당된 쿼타 내에서 API 호출이 가능하며, 추가 사용량이 필요한 경우 별도의 요금이 부과될 수 있다.

## 결론
GitHub Copilot SDK의 퍼블릭 프리뷰 출시는 AI 에이전트 개발 생태계에 중요한 변화를 가져올 수 있다.
기존에는 Copilot의 에이전틱 기능이 GitHub 내부에 한정되어 있었지만, SDK를 통해 어떤 애플리케이션에서든 동일한 에이전트 런타임을 활용할 수 있게 되었다.
5개 언어 지원, 시스템 프롬프트 커스터마이징, BYOK, OpenTelemetry 트레이싱, 권한 프레임워크 등 프로덕션 수준의 기능이 갖춰져 있다.
Copilot Free 사용자까지 접근 가능하므로, AI 에이전트 통합을 검토하는 개발자라면 한 번 살펴볼 가치가 있다.

## Reference
- [Copilot SDK Changelog](https://github.blog/changelog/2026-04-02-copilot-sdk-in-public-preview/)
