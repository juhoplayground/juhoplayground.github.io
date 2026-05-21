---
layout: post
title: "Qwen3.7-Max 공개: 에이전트 시대를 겨냥한 알리바바의 프런티어 모델"
author: 'Juho'
date: 2026-05-21 00:00:00 +0900
categories: [LLM]
tags: [LLM, Agent, Benchmark]
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
2. [핵심 에이전트 역량](#핵심-에이전트-역량)
   - [코딩과 사무 자동화](#코딩과-사무-자동화)
   - [크로스 하네스 일반화](#크로스-하네스-일반화)
3. [벤치마크 성능](#벤치마크-성능)
   - [코딩 에이전트](#코딩-에이전트)
   - [범용 에이전트](#범용-에이전트)
   - [추론과 일반 능력](#추론과-일반-능력)
4. [35시간 자율 커널 최적화](#35시간-자율-커널-최적화)
5. [학습 방법: 환경 스케일링](#학습-방법-환경-스케일링)
6. [사용 방법](#사용-방법)
   - [API 호출](#api-호출)
   - [코딩 어시스턴트 연동](#코딩-어시스턴트-연동)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Qwen 팀이 에이전트 시대를 겨냥한 최신 독점 모델 Qwen3.7-Max를 공개했다.
이 모델은 코드 작성과 디버깅, 사무 워크플로우 자동화, 그리고 수백에서 수천 단계에 이르는 자율 실행을 모두 수행할 수 있는 범용 에이전트 기반(agent foundation)으로 설계되었다.

가장 강조되는 특징은 에이전트 역량의 폭과 깊이다.
프런트엔드 프로토타이핑부터 복잡한 다중 파일 엔지니어링까지 코딩 에이전트로 작동하고, MCP 통합과 멀티 에이전트 오케스트레이션을 통해 사무·생산성 보조자 역할을 한다.
특히 1,000건 이상의 도구 호출로 구성된 35시간의 완전 자율 커널 최적화 실행으로, 극도로 긴 작업 지평에서 일관된 추론을 유지함을 보였다.
Claude Code, OpenClaw, Qwen Code 등 어떤 에이전트 프레임워크에 배포되든 일관되게 동작하는 크로스 스캐폴드 일반화도 핵심이다.
모델은 곧 Alibaba Cloud Model Studio를 통해 API로 제공될 예정이다.

## 핵심 에이전트 역량

Qwen3.7-Max는 단일 프레임워크에 최적화하지 않고, 다양한 에이전트 시스템에 그대로 꽂아 쓸 수 있는 백본을 지향한다.

### 코딩과 사무 자동화

코딩 측면에서는 단일 프롬프트로 Three.js 3D 씬, Canvas 애니메이션, 전체 페이지 레이아웃, 동적 SVG 같은 풍부한 인터랙티브 웹 애플리케이션을 생성할 수 있다.
사무 보조자로서는 도구 통합을 통해 작동한다.
예를 들어 대학 논문 양식 규격을 읽고 엉망인 초안을 자동으로 재포맷해, 페이지 레이아웃·제목 스타일·폰트·여백·목차·참고문헌 형식을 office-cli 도구 호출만으로 수정한다.

장기 지평 작업에서는 자율 계획과 다중 시간대에 걸친 연속 실행을 지원한다.
수천 번의 도구 호출과 수십 번의 정제 반복을 거치며 출력 품질을 점진적으로 끌어올린다.
일반적으로 전문 팀이 1~2주 걸리는 복잡한 프로젝트를 몇 시간 안에 끝까지 처리할 수 있다는 것이 Qwen 팀의 주장이다.

### 크로스 하네스 일반화

Qwen 팀의 Rollout 환경 인프라는 각 학습 인스턴스를 Task(작업), Harness(하네스), Verifier(검증기)라는 세 직교 요소로 분리한다.
이 세 요소를 자유롭게 재조합할 수 있어, 같은 작업을 다양한 종류·버전의 하네스 및 검증기와 최소 비용으로 짝지을 수 있다.

더 중요한 점은 이 설계가 크로스 하네스·크로스 검증기 강화학습을 가능하게 한다는 것이다.
모델이 동일한 작업을 서로 다른 하네스 구성에서 마주하게 되면서, 특정 하네스에 맞춘 지름길이 아니라 일반화 가능한 문제 해결 전략을 학습하도록 강제된다.
그 결과 QwenClawBench와 CoWorkBench 전반에서 평가 시 어떤 하네스를 쓰든 일관된 성능을 낸다.

## 벤치마크 성능

비교 대상은 Opus-4.6 Max, K2.6 Thinking, GLM-5.1 Thinking, DS-V4-Pro Max, 그리고 직전 모델인 Qwen3.6-Plus다.
표의 빈 칸(--)은 아직 점수가 공개되지 않은 항목이다.

### 코딩 에이전트

Qwen3.7-Max는 SWE-Pro(60.6), SWE-Multilingual(78.3), SciCode(53.5), QwenSVG(1608)에서 강세를 보인다.
Terminal Bench 2.0-Terminus(69.7)에서는 DS-V4-Pro Max(67.9)를 앞섰고, SWE-Verified(80.4)에서는 Opus-4.6 Max(80.8), DS-V4-Pro Max(80.6)와 대등하다.

| 벤치마크 | Opus-4.6 | K2.6 | GLM-5.1 | DS-V4-Pro | Qwen3.6+ | Qwen3.7-Max |
|------|------|------|------|------|------|------|
| Terminal Bench 2.0-Terminus | 65.4 | 66.7 | 63.5 | 67.9 | 61.6 | 69.7 |
| SWE-Verified | 80.8 | 80.2 | -- | 80.6 | 78.8 | 80.4 |
| SWE-Pro | 57.3 | 59.5 | 58.8 | 59.0 | 56.6 | 60.6 |
| SWE-Multilingual | 77.5 | 76.7 | -- | 76.2 | 73.8 | 78.3 |
| NL2repo | 47.6 | 42.8 | 41.0 | 35.5 | 34.4 | 47.2 |
| SciCode | 51.9 | 52.2 | 45.1 | -- | 41.4 | 53.5 |
| QwenWebDev | 1617 | -- | 1564 | 1570 | 1500 | 1568 |
| QwenSVG | 1541 | 1325 | 1605 | 1506 | 1432 | 1608 |

### 범용 에이전트

범용 에이전트에서 개선이 더 두드러진다.
MCP-Mark(60.8, GLM-5.1의 57.5 대비), MCP-Atlas(76.4, Opus-4.6의 75.8 대비), Skillsbench(59.2, K2.6의 56.2 대비)에서 우위를 보인다.
Kernel Bench L3에서는 1.98배 중앙값 속도 향상과 96% 승률로 강력한 GPU 커널 최적화 능력을 입증했다.
사무 자동화 벤치마크 SpreadSheetBench-v1에서는 87점으로 최상위권을 기록했다.

| 벤치마크 | Opus-4.6 | K2.6 | GLM-5.1 | DS-V4-Pro | Qwen3.6+ | Qwen3.7-Max |
|------|------|------|------|------|------|------|
| Qwenclaw | 65.5 | 54.7 | 58.7 | 59.2 | 57.2 | 64.3 |
| CoWorkBench | 68.2 | 58.2 | 66.0 | 66.3 | 64.5 | 67.2 |
| ClawEval | 70.4 | 61.5 | 62.7 | 58.4 | 57.1 | 65.2 |
| Skillsbench | -- | 56.2 | 53.1 | 52.3 | 45.7 | 59.2 |
| BFCL-V4 | 76.7 | 71.3 | 70.9 | 70.6 | 68.9 | 75.0 |
| MCP-Mark | 56.7 | 55.9 | 57.5 | 57.1 | 48.2 | 60.8 |
| MCP-Atlas | 75.8 | 66.6 | 71.8 | 73.6 | 74.1 | 76.4 |
| SpreadSheetBench-v1 | 89.3 | 84.5 | 85.2 | 84.9 | 80.2 | 87.0 |
| Kernel Bench L3 (속도/승률) | 2.63 / 98% | 1.41 / 80% | 2.00 / 78% | 1.07 / 54% | 1.03 / 48% | 1.98 / 96% |
| HLE w/ tools | 53.0 | 54.0 | 52.3 | 48.2 | 50.2 | 53.5 |
| QwenWorldBench | 56.1 | 50.9 | 50.2 | 52.3 | 47.6 | 57.3 |

### 추론과 일반 능력

추론에서 Qwen3.7-Max는 GPQA Diamond(92.4), HLE(41.4), HMMT 2026 Feb(97.1), IMOAnswerBench(90.0), Apex(44.5)에서 선두 결과를 달성하며 가장 어려운 추론 벤치마크에서 강세를 보였다.
일반 능력과 다국어에서는 IFBench(79.1)로 정밀한 지시 따르기를 보였고, WMT24++(85.8)와 MAXIFE(89.2)로 최상위 다국어 이해·번역 품질을 확인했다.

| 벤치마크 | Opus-4.6 | K2.6 | GLM-5.1 | DS-V4-Pro | Qwen3.6+ | Qwen3.7-Max |
|------|------|------|------|------|------|------|
| GPQA Diamond | 91.3 | 90.5 | 86.2 | 90.1 | 90.4 | 92.4 |
| HLE | 40.0 | 36.4 | 34.7 | 37.7 | 28.8 | 41.4 |
| LiveCodeBench | 88.8 | 89.6 | -- | 93.5 | 87.1 | 91.6 |
| HMMT 2026 Feb | 96.2 | 92.7 | 89.4 | 95.2 | 87.8 | 97.1 |
| IMOAnswerBench | 75.3 | 86.0 | 83.8 | 89.8 | 83.8 | 90.0 |
| Apex | 34.5 | 24.0 | 11.5 | 38.3 | 8.8 | 44.5 |
| MMLU-Pro | 89.7 | 87.1 | 86.3 | 87.5 | 88.5 | 89.6 |
| SuperGPQA | 72.5 | 71.3 | 68.0 | 69.9 | 71.6 | 73.6 |
| IFBench | 62.5 | 76.0 | 76.0 | 77.0 | 74.2 | 79.1 |
| MRCR-v2 128k | 84.0 | 63.1 | 62.0 | 74.4 | 85.9 | 90.4 |
| WMT24++ | 82.7 | 81.6 | 81.8 | 82.2 | 84.3 | 85.8 |
| MAXIFE | 81.3 | 87.7 | 87.7 | 88.9 | 88.2 | 89.2 |
| PolyMATH | 80.2 | 82.7 | 67.6 | 72.0 | 77.4 | 86.5 |

## 35시간 자율 커널 최적화

이번 발표에서 가장 인상적인 사례는 실제 프로덕션 커널을 자율 최적화한 실험이다.
대상은 SGLang의 Extend Attention으로, 새로 생성된 토큰과 최대 32K 항목의 prefix KV 캐시 사이의 어텐션 점수를 MTP와 함께 계산하는, 메모리 바운드이자 지연에 민감한 커널이다.

Qwen 팀은 학습 중 본 적 없는 하드웨어인 T-Head ZW-M890 PPU가 장착된 ECS 인스턴스에서 이 커널을 최적화하도록 모델에 맡겼다.
모델에는 사전 프로파일링 데이터, 하드웨어 문서, 예제 커널이 전혀 주어지지 않았다.
작업 설명, 기존 SGLang 구현, 평가 스크립트만 담긴 빈 작업 공간에서 시작했다.

약 35시간의 연속 자율 실행 동안 모델은 1,158건의 도구 호출에 걸쳐 432회의 커널 평가를 수행했다.
컴파일 실패를 진단하고, 정확성 버그를 고치며, 런타임 프로파일링으로 병목을 찾아 커널 아키텍처를 여러 번 재설계했다.
최종 결과는 Triton 레퍼런스 대비 10.0배의 기하 평균 속도 향상이었고, 30시간이 지난 뒤에도 의미 있는 개선을 계속 찾아냈다.

동일 조건에서 다른 모델들의 결과는 다음과 같다.

| 모델 | 속도 향상 |
|------|------|
| Qwen3.7-Max | 10.0배 |
| GLM 5.1 | 7.3배 |
| Kimi K2.6 | 5.0배 |
| DeepSeek V4 Pro | 3.3배 |
| Qwen3.6-Plus | 1.1배 |

조기 종료한 모델들은 5라운드 연속으로 도구 호출을 하지 않아, 더 이상 진전을 낼 수 없다고 스스로 판단하고 세션을 자발적으로 끝낸 경우다.
이 결과는 1,000건이 넘는 도구 호출에 걸쳐 컨텍스트를 잃거나 퇴행하지 않고 일관된 최적화 전략을 유지하는 장기 지평 추론과, 기억된 하드웨어 지식이 아니라 런타임 피드백에 의존해 처음 보는 아키텍처에 경쟁력 있는 커널을 만들어내는 강력한 인컨텍스트 일반화라는 두 속성을 보여준다.

## 학습 방법: 환경 스케일링

Qwen3.7-Max는 Qwen3.5에서 도입한 환경 스케일링(environment scaling) 접근을 이어받아, 에이전트 학습 환경의 품질과 다양성을 공격적으로 확장했다.
언어 모델이 다양한 사전학습 텍스트에서 일반화하듯, 에이전트 능력은 다양한 학습 환경에서 일반화한다는 것이 핵심 관찰이다.

환경 스케일링은 명확하고 일관된 개선 궤적을 만들어내며, Qwen3.7-Max는 Claude-4.6-Opus-Max에 근접하는 top-3 평균 순위를 달성했다.
평가에 쓰인 모든 벤치마크는 학습에 전혀 포함되지 않은 도메인 밖(out-of-domain) 환경이라는 점이 중요하다.

이 외에도 두 가지 장기 지평 사례가 제시되었다.
SWE 작업의 강화학습 모니터링에 Qwen3.7-Max를 투입해, 80시간이 넘는 실험에서 1만 건 이상의 호출로 학습 궤적을 재생하며 보상 해킹(reward hacking) 패턴을 자율 탐지하고 13개의 새 휴리스틱 규칙을 추가, 1,618건의 해킹 사례를 정확히 표시했다.
또한 스타트업 1년 생애주기를 시뮬레이션하는 YC-Bench에서 총 208만 달러 매출을 올려 Qwen3.6-Plus(105만 달러)의 두 배, Qwen3.5-Plus(35.2만 달러)의 5.9배 성과로 237개 작업을 완료했다.

## 사용 방법

Qwen3.7-Max는 곧 Alibaba Cloud Model Studio를 통해 제공되며, 주요 에이전트 프레임워크 및 코딩 어시스턴트와 통합할 수 있다.
에이전트 작업을 위해 이전 턴들의 thinking 내용을 메시지에 보존하는 `preserve_thinking` 기능을 권장한다.

### API 호출

Model Studio는 OpenAI 사양과 호환되는 chat completions·responses API, 그리고 Anthropic 호환 인터페이스를 지원한다.

```python
"""
환경 변수:
  DASHSCOPE_API_KEY: https://modelstudio.console.alibabacloud.com 에서 발급한 API Key
  DASHSCOPE_BASE_URL: (선택) compatible-mode API의 Base URL
    - 베이징: https://dashscope.aliyuncs.com/compatible-mode/v1
    - 싱가포르: https://dashscope-intl.aliyuncs.com/compatible-mode/v1
    - 미국(버지니아): https://dashscope-us.aliyuncs.com/compatible-mode/v1
"""
from openai import OpenAI
import os

api_key = os.environ.get("DASHSCOPE_API_KEY")
if not api_key:
    raise ValueError(
        "DASHSCOPE_API_KEY is required. "
        "Set it via: export DASHSCOPE_API_KEY='your-api-key'"
    )

client = OpenAI(
    api_key=api_key,
    base_url=os.environ.get(
        "DASHSCOPE_BASE_URL",
        "https://dashscope-intl.aliyuncs.com/compatible-mode/v1",
    ),
)

messages = [{"role": "user", "content": "Write a Python function to merge two sorted linked lists."}]

completion = client.chat.completions.create(
    model="qwen3.7-max",
    messages=messages,
    extra_body={
        "enable_thinking": True,
        # "preserve_thinking": True,
    },
    stream=True
)
```

### 코딩 어시스턴트 연동

Qwen API는 Anthropic API 프로토콜을 지원해 Claude Code에서 바로 사용할 수 있다.

```bash
npm install -g @anthropic-ai/claude-code

export ANTHROPIC_MODEL="qwen3.7-max"
export ANTHROPIC_SMALL_FAST_MODEL="qwen3.7-max"
export ANTHROPIC_BASE_URL=https://dashscope-intl.aliyuncs.com/apps/anthropic
export ANTHROPIC_AUTH_TOKEN=<your_api_key>

claude
```

Qwen 시리즈에 깊이 최적화된 Qwen Code도 사용할 수 있다.

```bash
npm install -g @qwen-code/qwen-code@latest
qwen
```

OpenClaw의 경우 `~/.openclaw/openclaw.json`에서 모델 프로바이더를 설정하며, 컨텍스트 윈도우는 1,000,000 토큰까지 지정할 수 있다.

## 결론

Qwen3.7-Max는 코딩과 사무 자동화부터 장기 지평 자율 작업까지 폭넓게 다루는 에이전트 중심 모델이다.
프런티어급 추론과 견고한 크로스 스캐폴드 일반화, 그리고 장시간에 걸쳐 생산적 실행을 지속하는 능력을 결합했다.
특히 35시간 자율 커널 최적화에서 Triton 대비 10배 속도 향상을 달성한 사례는, 처음 보는 하드웨어에서도 런타임 피드백만으로 일관된 장기 추론을 유지할 수 있음을 보여준다.
모델은 Alibaba Cloud Model Studio를 통해 곧 제공될 예정이다.

## Reference

- [Qwen3.7: The Agent Frontier (Qwen Blog)](https://qwen.ai/blog?id=qwen3.7)
