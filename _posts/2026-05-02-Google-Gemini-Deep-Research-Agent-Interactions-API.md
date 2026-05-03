---
layout: post
title: "Google Gemini Deep Research Agent - 장기 리서치 과제를 자동 수행하는 Interactions API"
author: 'Juho'
date: 2026-05-02 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, LLM, MCP]
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
   - [설정과 첫 실행](#설정과-첫-실행)
   - [협업 계획 모드](#협업-계획-모드)
   - [차트와 인포그래픽 자동 생성](#차트와-인포그래픽-자동-생성)
   - [원격 MCP 서버와 도구 구성](#원격-mcp-서버와-도구-구성)
   - [멀티모달 그라운딩](#멀티모달-그라운딩)
   - [실시간 스트리밍](#실시간-스트리밍)
4. [결론](#결론)
5. [Reference](#reference)

## 개요

Google이 Gemini Deep Research Agent를 공개했다.
에이전트가 장기 리서치 과제를 자율적으로 계획, 검색, 종합해 인용이 포함된 상세 보고서를 생성한다.
Deep Research는 백그라운드에서 실행되며 Interactions API를 통해서만 제공된다.
기존의 `generate_content`가 아니라 `client.interactions.create`로 호출하는 점이 중요한 차이다.

## 배경

두 가지 버전이 동시에 제공된다.

| 버전 | 특징 |
|------|------|
| deep-research-preview-04-2026 | 속도·효율 중심. 클라이언트 UI 스트리밍에 적합 |
| deep-research-max-preview-04-2026 | 자동 컨텍스트 수집·종합을 위한 최대 포괄성 |

이번 릴리스에서 함께 도입된 신규 역량은 다음과 같다.

- Collaborative planning: 실행 전 리서치 계획을 검토하고 다듬는다
- Native charts & infographics: 에이전트가 차트, 그래프, 인포그래픽을 직접 생성한다
- Remote MCP server: Model Context Protocol로 외부 도구 연결
- Extended tooling: Google Search, URL Context, Code Execution, MCP, File Search
- Multimodal research grounding: 이미지, PDF, 오디오를 리서치 컨텍스트로 전달

## 핵심 내용

### 설정과 첫 실행

Python SDK를 설치하고 API 키를 환경 변수에 설정한다.

```bash
pip install google-genai
export GEMINI_API_KEY="your-api-key"
```

Deep Research는 비동기 실행이므로 `background=True`와 폴링을 함께 사용한다.

```python
import time
from google import genai

client = genai.Client()

interaction = client.interactions.create(
    input="Research the history of Google TPUs.",
    agent="deep-research-preview-04-2026",
    background=True,
)

while True:
    interaction = client.interactions.get(interaction.id)
    if interaction.status == "completed":
        print(interaction.outputs[-1].text)
        break
    elif interaction.status == "failed":
        print(f"Research failed: {interaction.error}")
        break
    time.sleep(10)
```

### 협업 계획 모드

`collaborative_planning=True`로 두면 즉시 실행 대신 리서치 계획만 반환된다.
`previous_interaction_id`로 대화를 이어 계획을 수정할 수 있고, 준비되면 `collaborative_planning=False`로 전환해 실제 실행을 시작한다.

```python
import time
from google import genai

client = genai.Client()

plan = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Research Google TPUs vs competitor hardware.",
    agent_config={"type": "deep-research", "collaborative_planning": True},
    background=True,
)

while (result := client.interactions.get(id=plan.id)).status != "completed":
    time.sleep(5)

print(result.outputs[-1].text)
```

계획 수정은 다음과 같이 이어진다.

```python
refined = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Add a section comparing power efficiency.",
    agent_config={"type": "deep-research", "collaborative_planning": True},
    previous_interaction_id=plan.id,
    background=True,
)
```

마지막 턴에서 반드시 `collaborative_planning=False`로 바꿔야 보고서 생성이 시작된다.
"go ahead" 같은 단순 승인 메시지만으로는 실행되지 않는다.

```python
report = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Plan looks good!",
    agent_config={"type": "deep-research", "collaborative_planning": False},
    previous_interaction_id=refined.id,
    background=True,
)
```

### 차트와 인포그래픽 자동 생성

`visualization="auto"`를 설정하면 에이전트가 base64 인코딩 이미지 형태로 차트와 인포그래픽을 반환한다.

```python
import base64
from google import genai

client = genai.Client()

interaction = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Analyze global semiconductor market trends. Include charts showing market share changes.",
    agent_config={"type": "deep-research", "visualization": "auto"},
    background=True,
)

while (result := client.interactions.get(id=interaction.id)).status != "completed":
    time.sleep(5)

for output in result.outputs:
    if output.type == "text":
        print(output.text)
    elif output.type == "image" and output.data:
        image_bytes = base64.b64decode(output.data)
```

기능을 켜는 것만으로는 최적 결과를 얻기 어려우며 프롬프트에서 원하는 시각화를 명시하는 것이 권장된다.

### 원격 MCP 서버와 도구 구성

원격 MCP 서버를 연결해 외부 도구에 접근할 수 있다.
No-auth, bearer token, OAuth를 모두 지원하며 `allowed_tools`로 사용할 도구를 제한할 수 있다.

```python
interaction = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Research how recent geopolitical events influenced USD interest rates",
    tools=[
        {
            "type": "mcp_server",
            "name": "Finance Data Provider",
            "url": "https://finance.example.com/mcp",
            "headers": {"Authorization": "Bearer my-token"},
        }
    ],
    background=True,
)
```

기본 도구는 `google_search`, `url_context`, `code_execution`이다.
선택적으로 `mcp_server`와 `file_search`를 추가할 수 있다.
예를 들어 공용 웹 검색만 허용하려면 다음과 같이 제한한다.

```python
interaction = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input="Latest developments in quantum computing.",
    tools=[{"type": "google_search"}],
    background=True,
)
```

### 멀티모달 그라운딩

텍스트 프롬프트와 함께 이미지, PDF, 문서를 전달해 리서치의 근거로 삼을 수 있다.

```python
interaction = client.interactions.create(
    agent="deep-research-preview-04-2026",
    input=[
        {"type": "text", "text": "What has been the impact of this research paper?"},
        {"type": "document", "uri": "https://arxiv.org/pdf/1706.03762", "mime_type": "application/pdf"},
    ],
    background=True,
)
```

### 실시간 스트리밍

`stream=True`와 `thinking_summaries="auto"`를 결합하면 중간 추론 요약, 텍스트, 생성 이미지를 실시간으로 받아볼 수 있다.
연결이 끊기면 `last_event_id`를 넘겨 재연결한다.

```python
import base64
from google import genai
from IPython.display import Image, display

client = genai.Client()

interaction_id = None
last_event_id = None
is_complete = False

def process_stream(stream):
    global interaction_id, last_event_id, is_complete
    for chunk in stream:
        if chunk.event_type == "interaction.start":
            interaction_id = chunk.interaction.id
        if chunk.event_id:
            last_event_id = chunk.event_id
        if chunk.event_type == "content.delta":
            if chunk.delta.type == "text":
                print(chunk.delta.text, end="", flush=True)
            elif chunk.delta.type == "thought_summary":
                print(f"\n{chunk.delta.content.text}", flush=True)
            elif chunk.delta.type == "image" and chunk.delta.data:
                image_bytes = base64.b64decode(chunk.delta.data)
                display(Image(data=image_bytes))
        elif chunk.event_type in ("interaction.complete", "error"):
            is_complete = True

stream = client.interactions.create(
    input="Research AI chip market trends. Include charts comparing vendors.",
    agent="deep-research-preview-04-2026",
    background=True,
    stream=True,
    agent_config={
        "type": "deep-research",
        "thinking_summaries": "auto",
        "visualization": "auto",
    },
)
process_stream(stream)

while not is_complete and interaction_id:
    status = client.interactions.get(interaction_id)
    if status.status != "in_progress":
        break
    stream = client.interactions.get(
        id=interaction_id, stream=True, last_event_id=last_event_id,
    )
    process_stream(stream)
```

## 결론

Gemini Deep Research Agent는 단일 응답 대신 장기 실행 에이전트를 공식 API로 제공하는 사례다.
협업 계획, 네이티브 시각화, 원격 MCP 통합, 멀티모달 그라운딩은 리서치 전 과정을 한 에이전트 안에 담는 방향성을 보여준다.
Interactions API라는 새로운 표층을 통해 Google은 `generate_content`와 별개로 상태 기반 장기 과제를 위한 공식 경로를 마련했다.

## Reference

- [Google AI Studio - Deep Research Agent](https://x.com/GoogleAIStudio/status/2046633796380876930)
