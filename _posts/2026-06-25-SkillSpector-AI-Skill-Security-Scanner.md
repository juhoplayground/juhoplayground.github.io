---
layout: post
title: "SkillSpector: 설치 전에 AI 에이전트 스킬을 검사하는 보안 스캐너"
author: 'Juho'
date: 2026-06-25 00:00:00 +0900
categories: [AI]
tags: [AI, Security, Skill]
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
2. [무엇을 탐지하는가](#무엇을-탐지하는가)
   - [프롬프트 인젝션과 안전장치 우회](#프롬프트-인젝션과-안전장치-우회)
   - [데이터 유출과 권한 상승](#데이터-유출과-권한-상승)
   - [공급망과 과도한 권한](#공급망과-과도한-권한)
   - [코드 분석과 시그니처 탐지](#코드-분석과-시그니처-탐지)
   - [MCP 관련 탐지](#mcp-관련-탐지)
3. [작동 방식](#작동-방식)
   - [분석 파이프라인](#분석-파이프라인)
   - [위험 점수 산정](#위험-점수-산정)
4. [설치와 사용법](#설치와-사용법)
   - [설치](#설치)
   - [스캔 실행](#스캔-실행)
   - [출력 형식과 종료 코드](#출력-형식과-종료-코드)
   - [오탐 억제와 LLM 설정](#오탐-억제와-llm-설정)
   - [MCP 서버](#mcp-서버)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

SkillSpector는 AI 에이전트 스킬을 설치하기 전에 보안 관점에서 평가하는 정적 분석 도구다.
NVIDIA가 공개했으며 Apache 2.0 라이선스로 배포된다.
README에 따르면 스킬의 26.1%가 취약점을 포함하고, 5.2%는 악성 의도를 보이는 것으로 나타났다.
이 때문에 설치 전 사전 검증이 중요하다는 것이 도구의 출발점이다.

핵심 설계 원칙은 스캔 대상 스킬을 절대 실행하지 않는다는 점이다.
모든 분석은 파일 내용을 기반으로 수행된다.
지원하는 입력 형식은 Git 저장소, URL, zip 파일, 디렉터리, 그리고 md나 py 같은 단일 파일이다.

## 무엇을 탐지하는가

SkillSpector는 17개 그룹에 걸쳐 총 68개의 패턴을 점검한다.
각 그룹은 AI 에이전트 스킬이 가질 수 있는 서로 다른 위협 유형을 다룬다.

### 프롬프트 인젝션과 안전장치 우회

프롬프트 인젝션 그룹은 5개 패턴으로 구성된다.

| 패턴 | 설명 |
|------|------|
| Instruction Override | 기존 지시를 덮어쓰려는 시도 |
| Hidden Instructions | 숨겨진 지시문 |
| Exfiltration Commands | 정보 반출 명령 |
| Behavior Manipulation | 에이전트 행동 조작 |
| Harmful Content | 유해 콘텐츠 |

안전장치 우회를 노리는 Anti-Refusal 그룹은 3개 패턴이다.

| 패턴 | 설명 |
|------|------|
| Refusal Suppression | 거부 응답 억제 |
| Disclaimer Suppression | 면책 고지 억제 |
| Safety Policy Nullification | 안전 정책 무력화 |

### 데이터 유출과 권한 상승

Data Exfiltration 그룹은 4개 패턴으로 데이터 반출 시도를 잡아낸다.

| 패턴 | 설명 |
|------|------|
| External Transmission | 외부 전송 |
| Environment Variable Harvesting | 환경 변수 수집 |
| File System Enumeration | 파일 시스템 열거 |
| Context Leakage | 컨텍스트 유출 |

Privilege Escalation 그룹은 3개 패턴으로 권한 상승을 탐지한다.

| 패턴 | 설명 |
|------|------|
| Excessive Permissions | 과도한 권한 요구 |
| Sudo/Root Execution | 루트 권한 실행 |
| Credential Access | 자격 증명 접근 |

### 공급망과 과도한 권한

Supply Chain 그룹은 6개 패턴으로 공급망 위협을 점검한다.
이 중 Known Vulnerable Dependencies는 OSV.dev를 통해 실시간으로 알려진 취약점을 조회한다.

| 패턴 | 설명 |
|------|------|
| Unpinned Dependencies | 버전 고정되지 않은 의존성 |
| External Script Fetching | 외부 스크립트 가져오기 |
| Obfuscated Code | 난독화된 코드 |
| Known Vulnerable Dependencies | OSV.dev 실시간 조회 기반 알려진 취약 의존성 |
| Abandoned Dependencies | 유지보수 중단된 의존성 |
| Typosquatting | 타이포스쿼팅 |

Excessive Agency 그룹은 4개 패턴으로 에이전트의 과도한 자율성을 점검한다.

| 패턴 | 설명 |
|------|------|
| Unrestricted Tool Access | 제한 없는 도구 접근 |
| Autonomous Decision Making | 자율적 의사결정 |
| Scope Creep | 범위 확장 |
| Unbounded Resource Access | 무제한 리소스 접근 |

이 밖에도 Output Handling(3개), System Prompt Leakage(3개), Memory Poisoning(3개), Tool Misuse(3개), Rogue Agent(2개), Trigger Abuse(3개) 그룹이 각각의 위협 유형을 다룬다.
Output Handling은 검증되지 않은 출력 주입과 무제한 출력을 잡고, System Prompt Leakage는 시스템 프롬프트의 직접 유출과 간접 추출을 탐지한다.
Memory Poisoning은 지속적 컨텍스트 주입과 메모리 조작을, Rogue Agent는 자기 수정과 세션 지속성을 점검한다.

### 코드 분석과 시그니처 탐지

Behavioral AST 그룹은 9개 패턴으로 Python AST를 분석해 위험한 실행 호출을 찾는다.

| 패턴 | 설명 |
|------|------|
| exec() Call | exec 함수 호출 |
| eval() Call | eval 함수 호출 |
| Dynamic Import | 동적 임포트 |
| subprocess Call | subprocess 호출 |
| os.system/exec-family | os.system 등 exec 계열 호출 |
| compile() Call | compile 함수 호출 |
| Dynamic getattr() | 동적 getattr |
| Dangerous Execution Chain | 위험한 실행 체인 |
| Reflective getattr() Sink | 리플렉티브 getattr 싱크 |

Taint Tracking 그룹은 5개 패턴으로 오염된 입력이 위험한 지점까지 흐르는 경로를 추적한다.

| 패턴 | 설명 |
|------|------|
| Direct Taint Flow | 직접 오염 흐름 |
| Variable-Mediated Taint Flow | 변수를 거친 오염 흐름 |
| Credential Exfiltration Chain | 자격 증명 반출 체인 |
| File Read to Network Exfiltration | 파일 읽기 후 네트워크 반출 |
| External Input to Code Execution | 외부 입력에서 코드 실행으로 |

YARA Signatures 그룹은 4개 패턴으로 알려진 악성 코드 시그니처와 대조한다.

| 패턴 | 설명 |
|------|------|
| Malware Match | 멀웨어 일치 |
| Webshell Match | 웹셸 일치 |
| Cryptominer Match | 크립토마이너 일치 |
| Hack Tool/Exploit Match | 해킹 도구 또는 익스플로잇 일치 |

### MCP 관련 탐지

MCP Least Privilege 그룹은 4개 패턴으로 MCP 권한 선언의 최소 권한 위반을 점검한다.

| 패턴 | 설명 |
|------|------|
| Underdeclared Capability | 과소 선언된 권한 |
| Wildcard Permission | 와일드카드 권한 |
| Missing Permission Declaration | 권한 선언 누락 |
| Overdeclared Permission | 과다 선언된 권한 |

MCP Tool Poisoning 그룹은 4개 패턴으로 MCP 도구 설명에 숨겨진 조작을 탐지한다.

| 패턴 | 설명 |
|------|------|
| Hidden Instructions | 숨겨진 지시문 |
| Unicode Deception | 유니코드 기만 |
| Parameter Description Injection | 파라미터 설명 인젝션 |
| Description-Behavior Mismatch | 설명과 동작 불일치 |

## 작동 방식

### 분석 파이프라인

SkillSpector의 분석은 두 단계로 나뉜다.

1단계는 정적 분석이다.
정규표현식 패턴, Python AST 분석, YARA 시그니처, 그리고 OSV.dev를 통한 실시간 CVE 조회로 구성된다.
OSV.dev 조회는 API 키가 필요 없으며, 외부 연결이 불가능할 때를 위한 오프라인 폴백도 제공한다.

2단계는 선택적 LLM 분석이다.
의미 기반 평가로 오탐을 걸러내고 탐지 결과에 대한 설명을 제공한다.
LLM 단계를 활성화하면 정밀도가 약 87%까지 향상된다.

도구는 어떤 단계에서도 스캔 대상 스킬을 실행하지 않으며 모든 분석은 파일 내용 기반으로 이뤄진다.

### 위험 점수 산정

탐지된 항목은 심각도에 따라 점수가 부여된다.

| 심각도 | 점수 |
|--------|------|
| CRITICAL | 50점 추가 |
| HIGH | 25점 추가 |
| MEDIUM | 10점 추가 |
| LOW | 5점 추가 |

실행 가능한 스크립트에 대해서는 점수에 1.3배 가중치가 곱해진다.
최종 점수는 0에서 100 사이로 산정되며, 점수 구간에 따라 권고가 달라진다.

| 점수 구간 | 등급 | 권고 |
|-----------|------|------|
| 0–20 | LOW | SAFE |
| 21–50 | MEDIUM | CAUTION |
| 51–80 | HIGH | DO NOT INSTALL |
| 81–100 | CRITICAL | DO NOT INSTALL |

## 설치와 사용법

### 설치

가장 빠른 방법은 uv를 사용하는 것이다.

```bash
uv tool install git+https://github.com/NVIDIA/skillspector.git
```

소스에서 직접 설치할 수도 있다.

```bash
git clone https://github.com/NVIDIA/skillspector.git
cd skillspector
uv venv .venv && source .venv/bin/activate
make install
```

Docker로도 실행할 수 있다.

```bash
make docker-build
docker run --rm -v "$PWD:/scan" skillspector scan ./my-skill/ --no-llm
```

실행에는 Python 3.12 이상이 필요하다.

### 스캔 실행

스캔 명령은 다양한 입력 형식을 받는다.

```bash
skillspector scan ./my-skill/
skillspector scan ./SKILL.md
skillspector scan https://github.com/user/my-skill
skillspector scan ./my-skill.zip
```

주요 CLI 옵션은 다음과 같다.

| 옵션 | 설명 |
|------|------|
| --format | terminal, json, markdown, sarif 중 선택 |
| --output | 결과 저장 경로 |
| --no-llm | 정적 분석만 수행 |
| --yara-rules-dir | YARA 규칙 디렉터리 지정 |
| --baseline | 베이스라인 파일 지정 |
| --show-suppressed | 억제된 항목 표시 |
| --verbose | 상세 출력 |

### 출력 형식과 종료 코드

출력 형식은 네 가지를 지원한다.
기본값은 터미널 형식이며, 나머지는 파일로 저장한다.

```bash
skillspector scan ./my-skill/ --format json --output report.json
skillspector scan ./my-skill/ --format markdown --output report.md
skillspector scan ./my-skill/ --format sarif --output report.sarif
```

JSON 출력의 최상위 키는 skill, risk_assessment, components, issues, metadata다.
risk_assessment 아래에는 score(0에서 100), severity(LOW, MEDIUM, HIGH, CRITICAL), recommendation(SAFE, CAUTION, DO_NOT_INSTALL) 필드가 있다.
metadata에는 llm_requested와 llm_available 정보가 담긴다.

종료 코드는 CI 파이프라인 연동에 유용하다.

| 종료 코드 | 의미 |
|-----------|------|
| 0 | 점수 50 이하 (SAFE 또는 CAUTION) |
| 1 | 점수 50 초과 (DO_NOT_INSTALL) |
| 2 | 오류 |

### 오탐 억제와 LLM 설정

베이스라인 기능으로 오탐을 억제할 수 있다.

```bash
skillspector baseline ./my-skill/ -o .skillspector-baseline.yaml
skillspector scan ./my-skill/ --baseline .skillspector-baseline.yaml
```

베이스라인은 글로브 규칙과 지문을 사용해 변경에 관대한 억제를 제공한다.

LLM 분석은 환경 변수로 공급자를 설정한다.
SKILLSPECTOR_PROVIDER 변수로 openai, anthropic, nv_build, anthropic_proxy 중 하나를 지정한다.
공급자별 키는 OPENAI_API_KEY, ANTHROPIC_API_KEY, NVIDIA_INFERENCE_KEY를 사용하며, SKILLSPECTOR_MODEL로 기본 모델을 덮어쓸 수 있다.

```bash
export SKILLSPECTOR_PROVIDER=anthropic
export ANTHROPIC_API_KEY=sk-ant-...
skillspector scan ./my-skill/
```

LLM 단계를 건너뛰려면 --no-llm 옵션으로 정적 분석만 수행한다.

### MCP 서버

SkillSpector는 MCP 서버로도 동작한다.

```bash
pip install "skillspector[mcp]"
skillspector mcp
skillspector mcp --transport http --host 127.0.0.1 --port 8000
```

MCP 서버는 scan_skill 도구를 노출하며, 인자로 target, use_llm, output_format을 받는다.

## 결론

SkillSpector는 AI 에이전트 스킬을 설치하기 전에 위협을 식별하는 정적 보안 스캐너다.
프롬프트 인젝션부터 공급망 위협, MCP 도구 포이즈닝까지 17개 그룹 68개 패턴을 점검한다.
정적 분석을 기본으로 하고 선택적 LLM 단계로 정밀도를 높이는 2단계 구조를 갖췄다.
스캔 대상을 실행하지 않으므로 안전하며, CI 파이프라인에 종료 코드로 연동할 수 있어 설치 전 자동 검증 워크플로에 통합하기 좋다.

## Reference

- [SkillSpector (GitHub)](https://github.com/NVIDIA/SkillSpector/)
