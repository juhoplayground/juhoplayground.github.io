---
layout: post
title: "멀티 에이전트 오케스트레이션 실전: Microsoft Agent Framework로 만드는 AI 팟캐스트 스튜디오"
author: 'Juho'
date: 2026-02-20 02:00:00 +0900
categories: [AI]
tags: [AI, LLM, Ollama]
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
2. [로컬 우선 접근의 장점](#로컬-우선-접근의-장점)
3. [멀티 에이전트 오케스트레이션 패턴](#멀티-에이전트-오케스트레이션-패턴)
4. [기술 구현](#기술-구현)
5. [시스템 요구사항](#시스템-요구사항)
6. [한계점](#한계점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Microsoft의 기술 에반젤리스트 kinfey가 공개한 이 프로젝트는 여러 AI 에이전트가 협업하는 완전 자동화된 팟캐스트 제작 파이프라인을 구현한다.
모든 처리가 로컬에서 이루어지며, Ollama 기반 Qwen-3-8B 모델과 VibeVoice 음성 합성을 활용한다.
이 글은 멀티 에이전트 오케스트레이션의 실전 구현 방법을 정리한다.

## 로컬 우선 접근의 장점

클라우드 LLM이 강력하지만, 창작 작업에는 마찰이 있다.
로컬 우선 접근은 다음과 같은 이점을 제공한다.

- 토큰 생성의 즉각성
- 일회성 하드웨어 투자로 추가 비용 없음
- 인터넷 없이 작동 가능
- 프라이버시가 보장되는 로컬 처리

## 멀티 에이전트 오케스트레이션 패턴

시스템은 순차적 및 병렬 처리를 조합하여 팟캐스트를 제작한다.
각 에이전트의 역할은 다음과 같다.

| 에이전트 | 역할 |
|---------|------|
| SearchAgent | 웹에서 최신 뉴스 검색 |
| GenerateScriptAgent | 10분 팟캐스트 대본 작성 |
| ReviewExecutor | 품질 검토 및 피드백 루프 |
| VibeVoice | 자연스러운 다성 음성 합성 |

ReviewExecutor와 GenerateScriptAgent 사이에는 피드백 루프가 존재하여, 품질이 기준에 미달하면 대본을 재작성한다.

## 기술 구현

### 로컬 모델 클라이언트 초기화

```python
from agent_framework.ollama import OllamaChatClient

chat_client = OllamaChatClient(
    model_id="qwen3:8b",
    endpoint="http://localhost:11434"
)
```

### 전문 에이전트 정의

```python
from agent_framework import ChatAgent

researcher_agent = client.create_agent(
    name="SearchAgent",
    instructions="최신 뉴스를 검색엔진으로 찾아줘",
    tools=[web_search]
)
```

### 워크플로우 연결

```python
workflow = (
    WorkflowBuilder()
    .set_start_executor(search_executor)
    .add_edge(search_executor, gen_script_executor)
    .add_edge(gen_script_executor, review_executor)
    .add_edge(review_executor, gen_script_executor)  # 재작성 루프
    .build()
)
```

순차적 파이프라인에 피드백 루프를 추가하여 품질을 자동으로 보장하는 구조이다.

## 시스템 요구사항

| 항목 | 요구사항 |
|------|---------|
| RAM | 최소 16GB, 권장 32GB |
| 하드웨어 | GPU/NPU 권장 |
| 소프트웨어 | Python 3.10+, Ollama, Microsoft Agent Framework |

## 한계점

- 로컬 모델의 성능이 최신 클라우드 모델에 미치지 못할 수 있다.
- 하드웨어 초기 투자 비용이 필요하다.
- 기술 설정에 일정 수준의 지식이 요구된다.
- 팟캐스트에 최적화된 구조이므로 다른 용도에는 커스터마이징이 필요하다.

## 결론

이 프로젝트는 개발자들이 코드를 작성하는 시대에서 AI 에이전트 생태계를 연출하는 시대로 넘어가고 있음을 보여준다.
Microsoft Agent Framework와 Ollama를 활용하면 클라우드 의존 없이 로컬에서 멀티 에이전트 시스템을 구축할 수 있다.
프라이버시와 비용 효율성을 중시하는 팀이라면 로컬 우선 멀티 에이전트 오케스트레이션을 검토해볼 가치가 있다.

## Reference

- [Engineering a Local-First Agentic Podcast Studio - Microsoft Tech Community](https://techcommunity.microsoft.com/blog/azuredevcommunityblog/engineering-a-local-first-agentic-podcast-studio-a-deep-dive-into-multi-agent-or/4488947)
- [EdgeAI for Beginners - GitHub](https://github.com/microsoft/edgeai-for-beginners)
- [Microsoft Agent Framework - GitHub](https://github.com/microsoft/agent-framework)
- [Microsoft Agent Framework Samples - GitHub](https://github.com/microsoft/agent-framework-samples)
