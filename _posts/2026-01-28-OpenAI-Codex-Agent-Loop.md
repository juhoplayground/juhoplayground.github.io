---
layout: post
title: OpenAI Codex Agent Loop - 에이전트 루프의 내부 동작 원리
author: 'Juho'
date: 2026-01-28 00:00:00 +0900
categories: [OpenAI]
tags: [AI, LLM, Agent, Coding]
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
2. [에이전트 루프란](#에이전트-루프란)
3. [모델 추론](#모델-추론)
4. [초기 프롬프트 구성](#초기-프롬프트-구성)
5. [첫 번째 턴 실행](#첫-번째-턴-실행)
6. [멀티턴 대화](#멀티턴-대화)
7. [성능 최적화](#성능-최적화)
8. [컨텍스트 윈도우 관리](#컨텍스트-윈도우-관리)

## 개요

Codex CLI는 OpenAI의 크로스 플랫폼 로컬 소프트웨어 에이전트다.
사용자 머신에서 안전하고 효율적으로 동작하면서 고품질의 신뢰할 수 있는 소프트웨어 변경을 생성하도록 설계되었다.
이 글에서는 Codex의 핵심인 에이전트 루프(Agent Loop)의 내부 동작 원리를 상세히 살펴본다.

OpenAI에서 "Codex"는 Codex CLI, Codex Cloud, Codex VS Code 확장을 포함하는 소프트웨어 에이전트 제품군을 의미한다.
이 글에서는 모든 Codex 경험의 기반이 되는 핵심 에이전트 루프와 실행 로직을 제공하는 Codex harness에 초점을 맞춘다.

## 에이전트 루프란

모든 AI 에이전트의 핵심에는 에이전트 루프가 있다.
에이전트 루프는 사용자, 모델, 모델이 호출하는 도구 간의 상호작용을 조율하는 핵심 로직이다.

### 기본 동작 흐름

에이전트 루프의 기본 흐름은 다음과 같다.

1. 에이전트가 사용자 입력을 받아 프롬프트에 포함
2. 모델에 쿼리를 보내 응답 생성 (추론)
3. 모델이 최종 응답을 생성하거나 도구 호출을 요청
4. 도구 호출 시 에이전트가 실행하고 출력을 프롬프트에 추가
5. 새로운 입력으로 모델을 다시 쿼리
6. 모델이 도구 호출 대신 사용자 메시지를 생성할 때까지 반복

### 추론 과정

추론(Inference) 중에 텍스트 프롬프트는 먼저 입력 토큰 시퀀스로 변환된다.
이 토큰들은 모델의 어휘에 대한 인덱스 정수다.
토큰을 사용해 모델을 샘플링하여 새로운 출력 토큰 시퀀스를 생성한다.
출력 토큰은 다시 텍스트로 변환되어 모델의 응답이 된다.

### 턴(Turn)의 개념

사용자 입력부터 에이전트 응답까지의 과정을 대화의 한 턴이라고 한다.
Codex에서는 이를 스레드(thread)라고 부른다.
한 턴 내에서 모델 추론과 도구 호출 사이에 여러 번의 반복이 있을 수 있다.
기존 대화에 새 메시지를 보낼 때마다 이전 턴의 메시지와 도구 호출을 포함한 대화 히스토리가 새 턴의 프롬프트에 포함된다.

### 에이전트의 출력

에이전트는 도구 호출을 실행하여 로컬 환경을 수정할 수 있다.
따라서 에이전트의 "출력"은 어시스턴트 메시지에 국한되지 않는다.
소프트웨어 에이전트의 주요 출력은 머신에 작성하거나 편집하는 코드인 경우가 많다.
그럼에도 각 턴은 항상 어시스턴트 메시지로 끝나며, 이는 에이전트 루프의 종료 상태를 나타낸다.

## 모델 추론

Codex CLI는 Responses API에 HTTP 요청을 보내 모델 추론을 실행한다.

### Responses API 엔드포인트

Codex CLI가 사용하는 Responses API 엔드포인트는 설정 가능하다.

| 인증 방식 | 엔드포인트 |
|-----------|------------|
| ChatGPT 로그인 | https://chatgpt.com/backend-api/codex/responses |
| API 키 인증 (OpenAI 호스팅) | https://api.openai.com/v1/responses |
| --oss 플래그 (ollama/LM Studio) | http://localhost:11434/v1/responses |
| Azure 등 클라우드 | 클라우드 제공자 엔드포인트 |

### Responses API 호환성

Responses API를 구현하는 모든 엔드포인트와 함께 사용할 수 있다.
ollama 0.13.4+ 또는 LM Studio 0.3.39+에서 gpt-oss를 사용할 때 로컬에서 실행된다.

## 초기 프롬프트 구성

Responses API를 쿼리할 때 프롬프트를 직접 지정하지 않는다.
대신 다양한 입력 타입을 쿼리의 일부로 지정하면 서버가 이를 프롬프트로 구조화한다.

### Role 우선순위

초기 프롬프트의 모든 항목은 role과 연관된다.
role은 해당 콘텐츠가 가져야 할 가중치를 나타낸다.

| 우선순위 | Role |
|----------|------|
| 1 (최고) | system |
| 2 | developer |
| 3 | user |
| 4 (최저) | assistant |

### Responses API 주요 파라미터

Responses API는 여러 파라미터를 가진 JSON 페이로드를 받는다.
핵심 파라미터 세 가지는 다음과 같다.

**instructions**

모델 컨텍스트에 삽입되는 system (또는 developer) 메시지다.
~/.codex/config.toml의 model_instructions_file에서 읽거나 모델과 연관된 base_instructions를 사용한다.
모델별 지시사항은 Codex 레포지토리에 있으며 CLI에 번들된다.

**tools**

모델이 응답 생성 중 호출할 수 있는 도구 정의 목록이다.

```json
[
  {
    "type": "function",
    "name": "shell",
    "description": "Runs a shell command and returns its output...",
    "strict": false,
    "parameters": {
      "type": "object",
      "properties": {
        "command": {"type": "array", "description": "The command to execute"},
        "workdir": {"description": "The working directory..."},
        "timeout_ms": {"description": "The timeout for the command..."}
      },
      "required": ["command"]
    }
  },
  {
    "type": "function",
    "name": "update_plan",
    "description": "Updates the task plan...",
    "parameters": {
      "type": "object",
      "properties": {"plan": {}, "explanation": {}},
      "required": ["plan"]
    }
  },
  {
    "type": "web_search",
    "external_web_access": false
  }
]
```

도구는 세 가지 출처에서 제공된다.

- Codex CLI 제공 도구 (shell, update_plan 등)
- Responses API 제공 도구 (web_search 등)
- 사용자 제공 도구 (주로 MCP 서버를 통해)

**input**

모델에 대한 텍스트, 이미지, 파일 입력 목록이다.

### input 필드 구성

Codex는 사용자 메시지를 추가하기 전에 다음 항목들을 input에 삽입한다.

**1. 권한 지시사항 (role=developer)**

Codex 제공 shell 도구에만 적용되는 샌드박스를 설명한다.
MCP 서버 등 다른 도구는 Codex에 의해 샌드박싱되지 않으며 자체 가드레일을 적용해야 한다.

```
<permissions instructions>
  - 파일 권한과 네트워크 접근을 설명하는 샌드박스 설명
  - 셸 명령 실행을 위해 사용자에게 권한을 요청할 시점에 대한 지시
  - Codex가 쓸 수 있는 폴더 목록 (있는 경우)
</permissions instructions>
```

**2. 개발자 지시사항 (role=developer, 선택)**

사용자의 config.toml 파일에서 읽은 developer_instructions 값이다.

**3. 사용자 지시사항 (role=user, 선택)**

여러 소스에서 집계된 "사용자 지시사항"이다.
일반적으로 더 구체적인 지시사항이 나중에 나타난다.

- $CODEX_HOME의 AGENTS.override.md와 AGENTS.md 내용
- Git/프로젝트 루트부터 현재 작업 디렉토리까지 각 폴더에서 AGENTS.override.md, AGENTS.md 또는 config.toml의 project_doc_fallback_filenames에 지정된 파일명 확인 (기본 32 KiB 제한)
- 스킬이 설정된 경우 스킬 관련 프리앰블과 메타데이터

**4. 환경 컨텍스트 (role=user)**

에이전트가 현재 작동하는 로컬 환경을 설명한다.

```xml
<environment_context>
  <cwd>/Users/mbolin/code/codex5</cwd>
  <shell>zsh</shell>
</environment_context>
```

### 최종 프롬프트 구조

서버가 요청을 받으면 JSON을 사용해 다음과 같이 프롬프트를 구성한다.

```
[System Message]
    ↓
[Tools]
    ↓
[Instructions]
    ↓
[Input Items]
    - Developer 메시지 (권한 지시사항)
    - Developer 메시지 (개발자 지시사항) - 선택
    - User 메시지 (사용자 지시사항) - 선택
    - User 메시지 (환경 컨텍스트)
    - User 메시지 (사용자 입력)
```

처음 세 항목의 순서는 클라이언트가 아닌 서버가 결정한다.

## 첫 번째 턴 실행

Responses API에 대한 HTTP 요청이 대화의 첫 번째 "턴"을 시작한다.

### SSE 스트리밍

서버는 Server-Sent Events (SSE) 스트림으로 응답한다.
각 이벤트의 데이터는 "response"로 시작하는 "type"을 가진 JSON 페이로드다.

```
data: {"type":"response.reasoning_summary_text.delta","delta":"ah ", ...}
data: {"type":"response.reasoning_summary_text.delta","delta":"ha!", ...}
data: {"type":"response.reasoning_summary_text.done", "item_id":...}
data: {"type":"response.output_item.added", "item":{...}}
data: {"type":"response.output_text.delta", "delta":"forty-", ...}
data: {"type":"response.output_text.delta", "delta":"two!", ...}
data: {"type":"response.completed","response":{...}}
```

Codex는 이벤트 스트림을 소비하여 클라이언트가 사용할 수 있는 내부 이벤트 객체로 재발행한다.

### 이벤트 유형

| 이벤트 | 용도 |
|--------|------|
| response.output_text.delta | UI에서 스트리밍 지원 |
| response.output_item.added | 후속 API 호출의 input에 추가할 객체로 변환 |

### 도구 호출 처리

첫 번째 요청에서 type=reasoning과 type=function_call 두 개의 response.output_item.done 이벤트가 있다고 가정하면, 도구 호출 응답과 함께 모델을 다시 쿼리할 때 input 필드에 이들을 표현해야 한다.

```json
[
  {
    "type": "reasoning",
    "summary": [
      "type": "summary_text",
      "text": "**Adding an architecture diagram for README.md**\n\nI need to..."
    ],
    "encrypted_content": "gAAAAABpaDWNMxMeLw..."
  },
  {
    "type": "function_call",
    "name": "shell",
    "arguments": "{\"command\":\"cat README.md\",\"workdir\":\"/Users/mbolin/code/codex5\"}",
    "call_id": "call_8675309..."
  },
  {
    "type": "function_call_output",
    "call_id": "call_8675309...",
    "output": "<p align=\"center\"><code>npm i -g @openai/codex</code>..."
  }
]
```

### 턴 종료

추론과 도구 호출 사이에 많은 반복이 있을 수 있다.
최종적으로 어시스턴트 메시지를 받으면 턴이 종료된다.

```
data: {"type":"response.output_text.done","text": "I added a diagram to explain...", ...}
data: {"type":"response.completed","response":{...}}
```

Codex CLI는 어시스턴트 메시지를 사용자에게 표시하고 composer에 포커스를 맞춰 사용자의 "턴"임을 나타낸다.

## 멀티턴 대화

사용자가 응답하면 이전 턴의 어시스턴트 메시지와 사용자의 새 메시지 모두 새 턴을 시작하기 위한 Responses API 요청의 input에 추가되어야 한다.

```json
[
  {
    "type": "message",
    "role": "assistant",
    "content": [
      {
        "type": "output_text",
        "text": "I added a diagram to explain the client/server architecture."
      }
    ]
  },
  {
    "type": "message",
    "role": "user",
    "content": [
      {
        "type": "input_text",
        "text": "That's not bad, but the diagram is missing the bike shed."
      }
    ]
  }
]
```

대화가 계속됨에 따라 Responses API에 보내는 input의 길이도 계속 증가한다.

## 성능 최적화

대화 과정에서 Responses API에 보내는 JSON 양이 2차적으로 증가하는 문제가 있다.

### previous_response_id 미사용 이유

Responses API는 이 문제를 완화하기 위한 previous_response_id 파라미터를 지원한다.
하지만 Codex는 현재 이를 사용하지 않는다.

| 이유 | 설명 |
|------|------|
| 무상태 유지 | 모든 요청을 완전히 무상태로 유지 |
| ZDR 지원 | Zero Data Retention 구성 지원 |

ZDR 고객은 이전 턴의 독점 추론 메시지 혜택을 포기하지 않는다.
연관된 encrypted_content를 서버에서 복호화할 수 있기 때문이다.
OpenAI는 ZDR 고객의 복호화 키는 유지하지만 데이터는 유지하지 않는다.

### Prompt Caching

모델 샘플링 비용이 네트워크 트래픽 비용보다 훨씬 크다.
따라서 샘플링이 효율성 노력의 주요 대상이다.
Prompt Caching을 통해 이전 추론 호출의 계산을 재사용할 수 있다.

캐시 히트를 얻으면 모델 샘플링이 2차가 아닌 선형이 된다.

### Prompt Caching 원리

캐시 히트는 프롬프트 내에서 정확한 접두사 일치에서만 가능하다.
캐싱 이점을 얻으려면 다음 규칙을 따라야 한다.

- 지시사항과 예시 같은 정적 콘텐츠를 프롬프트 시작에 배치
- 사용자별 정보 같은 가변 콘텐츠를 끝에 배치
- 이미지와 도구도 요청 간에 동일해야 함

### 캐시 미스 원인

Codex에서 캐시 미스를 유발할 수 있는 작업은 다음과 같다.

| 원인 | 설명 |
|------|------|
| 도구 변경 | 대화 중간에 모델에 사용 가능한 도구 변경 |
| 모델 변경 | Responses API 요청 대상 모델 변경 |
| 샌드박스 설정 변경 | 샌드박스 구성, 승인 모드 또는 현재 작업 디렉토리 변경 |

### MCP 도구 주의사항

초기 MCP 도구 지원에서 도구를 일관된 순서로 열거하지 못하는 버그가 있었다.
이로 인해 캐시 미스가 발생했다.

MCP 서버는 notifications/tools/list_changed 알림을 통해 제공하는 도구 목록을 즉시 변경할 수 있다.
긴 대화 중간에 이 알림을 처리하면 비용이 많이 드는 캐시 미스가 발생할 수 있다.

### 캐시 히트 유지 전략

가능한 경우 대화 중간에 발생하는 구성 변경은 이전 메시지를 수정하는 대신 input에 새 메시지를 추가하여 처리한다.

**샌드박스 구성 또는 승인 모드 변경 시**

원래 permissions instructions 항목과 동일한 형식의 새로운 role=developer 메시지를 삽입한다.

**현재 작업 디렉토리 변경 시**

원래 environment_context와 동일한 형식의 새로운 role=user 메시지를 삽입한다.

## 컨텍스트 윈도우 관리

대화가 길어지면 프롬프트 길이도 증가한다.
모든 모델에는 한 번의 추론 호출에 사용할 수 있는 최대 토큰 수인 컨텍스트 윈도우가 있다.
이 윈도우에는 입력과 출력 토큰이 모두 포함된다.

### Compaction이란

컨텍스트 윈도우 소진을 방지하기 위한 일반적인 전략은 토큰 수가 임계값을 초과하면 대화를 압축하는 것이다.
input을 대화를 대표하는 새롭고 더 작은 항목 목록으로 교체한다.
이를 통해 에이전트가 지금까지 일어난 일을 이해하면서 계속 진행할 수 있다.

### 초기 Compaction 구현

초기 구현에서는 사용자가 /compact 명령을 수동으로 호출해야 했다.
기존 대화와 요약을 위한 커스텀 지시사항을 사용해 Responses API를 쿼리했다.
Codex는 요약이 포함된 결과 어시스턴트 메시지를 후속 대화 턴의 새 input으로 사용했다.

### 현재 Compaction 구현

이후 Responses API는 compaction을 더 효율적으로 수행하는 특별한 /responses/compact 엔드포인트를 지원하게 되었다.

엔드포인트가 반환하는 항목 목록은 다음을 포함한다.

- 이전 input을 대체하여 대화를 계속하면서 컨텍스트 윈도우를 확보할 수 있는 항목들
- 원래 대화에 대한 모델의 잠재적 이해를 보존하는 불투명한 encrypted_content 항목이 있는 특별한 type=compaction 항목

### 자동 Compaction

현재 Codex는 auto_compact_limit가 초과되면 이 엔드포인트를 사용해 자동으로 대화를 압축한다.

```
대화 진행 중
    ↓
토큰 수가 auto_compact_limit 초과
    ↓
/responses/compact 엔드포인트 호출
    ↓
압축된 항목 목록 수신 (type=compaction 포함)
    ↓
새 input으로 대화 계속
```

## Reference

- [Unrolling the Codex agent loop - OpenAI](https://openai.com/index/unrolling-the-codex-agent-loop/)
- [Codex CLI GitHub Repository](https://github.com/openai/codex)
- [OpenAI Responses API Documentation](https://platform.openai.com/docs/api-reference/responses)
- [OpenAI Prompt Caching Documentation](https://platform.openai.com/docs/guides/prompt-caching)
