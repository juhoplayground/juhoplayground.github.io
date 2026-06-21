---
layout: post
title: "Open Code Review : 알리바바의 오픈소스 AI 코드 리뷰 CLI"
author: 'Juho'
date: 2026-06-18 00:00:00 +0900
categories: [Dev]
tags: [Agent, AI, Security]
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
2. [아키텍처](#아키텍처)
   - [결정론적 엔지니어링](#결정론적-엔지니어링)
   - [에이전트 계층](#에이전트-계층)
3. [성능 벤치마크](#성능-벤치마크)
4. [환경 설정](#환경-설정)
   - [설치](#설치)
   - [LLM 설정과 리뷰 실행](#llm-설정과-리뷰-실행)
5. [리뷰 규칙과 에이전트 통합](#리뷰-규칙과-에이전트-통합)
   - [리뷰 규칙 파일](#리뷰-규칙-파일)
   - [에이전트 통합과 CI/CD](#에이전트-통합과-cicd)
6. [주의사항](#주의사항)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

Open Code Review는 알리바바 그룹이 개발한 AI 기반 코드 리뷰 CLI 도구다.
프로젝트는 "Open-source & free — Battle-tested at Alibaba's scale"를 표방한다.
2년간 알리바바 내부에서 수만 명의 개발자가 사용했으며, 수백만 개의 코드 결함을 식별한 실전 경험을 바탕으로 한다.

핵심 동작은 단순하다.
Git diff를 읽어 설정 가능한 LLM으로 전송하고, 라인 레벨 정밀도의 구조화된 리뷰 주석을 생성한다.
단순히 diff만 보는 것이 아니라 파일 콘텐츠를 열람하고, 코드베이스를 검색하며, 변경된 다른 파일을 검사해 표면 수준이 아닌 심층 리뷰를 수행한다.

내장 규칙셋은 실무에서 자주 문제가 되는 결함을 직접 겨냥한다.
NPE(Null Pointer Exception), 스레드 안전성, XSS, SQL 인젝션 등을 검증한다.

라이선스는 Apache-2.0이며 GitHub 기준 8.2k stars, 508 forks를 기록하고 있다.
구현 언어 비중은 Go가 74.7%, TypeScript가 14.5%, JavaScript가 3.4%다.

## 아키텍처

Open Code Review의 설계는 결정론적 엔지니어링과 에이전트의 하이브리드 구조다.
규칙으로 강제할 부분은 코드로 고정하고, 판단이 필요한 부분만 LLM에게 맡긴다.

### 결정론적 엔지니어링

결정론적 엔지니어링 계층은 하드 제약(Hard Constraints)을 담당한다.
이 계층은 다음과 같은 역할을 수행한다.

| 구성 요소 | 역할 |
|-----------|------|
| 정확한 파일 선택 | 리뷰 대상 파일 누락 방지 |
| 스마트 파일 번들링 | 관련 파일을 그룹화하여 함께 분석 |
| 세분화된 규칙 매칭 | 변경 내용에 맞는 규칙 정밀 적용 |
| 댓글 위치 조정/반영 모듈 | 독립적으로 주석 위치를 조정하고 반영 |

이 계층 덕분에 어떤 파일을 검토할지, 어떤 규칙을 적용할지가 LLM의 변덕에 좌우되지 않는다.

### 에이전트 계층

에이전트 계층은 동적 의사결정을 담당한다.
시나리오별로 최적화된 프롬프트 템플릿을 사용하며, 대규모 프로덕션 데이터 분석을 기반으로 구성한 도구 집합을 활용한다.

즉, 결정론적 계층이 "무엇을, 어떤 규칙으로" 검토할지를 고정하면 에이전트 계층이 그 안에서 "어떻게" 판단할지를 담당하는 구조다.

지원하는 모델과 플랫폼은 폭넓다.

| 구분 | 지원 항목 |
|------|-----------|
| 모델/플랫폼 | OpenAI(GPT), Anthropic(Claude), DashScope, DeepSeek, Z-AI |
| 엔드포인트 | 커스텀 엔드포인트 지원 |
| 프로토콜 | Anthropic, OpenAI |

## 성능 벤치마크

성능 검증은 50개 오픈소스 저장소, 200개 실제 PR, 10개 언어를 대상으로 진행되었다.
비교 대상은 Claude Code다.

| 항목 | 결과 |
|------|------|
| F1 점수 | Claude Code 대비 크게 향상 |
| 정확도(Precision) | 유의미하게 증가 |
| 토큰 소비 | 약 1/9 수준 |
| 처리 속도 | 더 빠름 |
| 재현율(Recall) | 일반 에이전트보다 낮음 |

주목할 트레이드오프는 재현율이다.
일반 에이전트보다 재현율은 낮지만, 이는 "정확성 우선" 설계의 의도된 결과다.
잘못된 지적을 줄이는 대신 일부 결함을 놓칠 수 있다는 방향이며, 토큰 소비가 약 1/9 수준이라는 점은 대규모 적용 시 비용 측면에서 강한 이점이 된다.

## 환경 설정

### 설치

설치 방법은 세 가지가 제공된다.

| 방법 | 명령/설명 |
|------|-----------|
| NPM(권장) | npm install -g @alibaba-group/open-code-review |
| GitHub Release | macOS/Linux/Windows 바이너리 다운로드 |
| 소스 빌드 | git clone 후 make build |

NPM 설치 시 전역 ocr 명령이 등록된다.

```bash
npm install -g @alibaba-group/open-code-review
```

소스에서 직접 빌드하려면 다음과 같이 진행한다.

```bash
git clone https://github.com/alibaba/open-code-review.git
cd open-code-review
make build
```

### LLM 설정과 리뷰 실행

먼저 사용할 LLM을 설정한다.
대화형 방식과 수동 방식, 환경변수 방식을 모두 지원한다.

대화형 설정은 다음과 같다.

```bash
ocr config provider
ocr config model
```

수동 설정은 항목별로 직접 값을 지정한다.

```bash
ocr config set llm.url https://api.anthropic.com/v1/messages
ocr config set llm.auth_token your-api-key
ocr config set llm.model claude-opus-4-6
ocr config set llm.use_anthropic true
```

환경변수는 가장 높은 우선순위를 가진다.

| 환경변수 | 용도 |
|----------|------|
| OCR_LLM_URL | LLM 엔드포인트 URL |
| OCR_LLM_TOKEN | 인증 토큰 |
| OCR_LLM_MODEL | 사용할 모델 |
| OCR_USE_ANTHROPIC | Anthropic 프로토콜 사용 여부 |

설정을 마쳤다면 연결을 테스트한 뒤 리뷰를 실행한다.

```bash
ocr llm test
ocr review
ocr review --from main --to feature-branch
ocr review --commit abc123
```

review 명령은 다양한 플래그를 지원한다.

| 플래그 | 설명 |
|--------|------|
| --from / --to | 비교 기준 브랜치 또는 참조 지정 |
| --commit (-c) | 특정 커밋 리뷰 |
| --preview (-p) | LLM 없이 파일 미리보기, 기본 false |
| --format (-f) | text 또는 json, 기본 text |
| --concurrency | 동시 처리 수, 기본 8 |
| --audience | human 또는 agent, 기본 human |
| --model | 사용할 모델 지정 |

주요 설정 항목으로는 provider, llm.url, llm.auth_token, llm.model, language(기본 English), telemetry.enabled가 있다.

## 리뷰 규칙과 에이전트 통합

### 리뷰 규칙 파일

리뷰 규칙은 4계층 우선순위로 적용된다.
상위 항목이 하위 항목을 덮어쓴다.

| 우선순위 | 출처 |
|----------|------|
| 1 | --rule 플래그 |
| 2 | 프로젝트 .opencodereview/rule.json |
| 3 | 사용자 ~/.opencodereview/rule.json |
| 4 | 시스템 기본값 |

규칙 파일은 JSON 형식이며 rules 배열과 include, exclude 패턴으로 구성된다.
rules 배열의 각 항목은 path와 rule을 가진다.

```json
{
  "rules": [
    { "path": "src/**", "rule": "스레드 안전성과 NPE를 중점 검토" }
  ],
  "include": ["**/*.go"],
  "exclude": ["**/vendor/**"]
}
```

규칙 파일의 유효성은 별도 명령으로 확인할 수 있다.

```bash
ocr rules check <file>
```

### 에이전트 통합과 CI/CD

다른 에이전트 환경에도 통합할 수 있다.

Skill 형태로 추가하려면 다음과 같이 한다.

```bash
npx skills add alibaba/open-code-review --skill open-code-review
```

Claude Code 플러그인으로 설치하려면 마켓플레이스를 추가하고 설치한다.

```bash
/plugin marketplace add alibaba/open-code-review
/plugin install open-code-review@open-code-review
```

이 외에도 Codex 플러그인(로컬) 방식과 커맨드 파일을 직접 복사하는 방식을 지원한다.

CI/CD 파이프라인에서는 json 포맷으로 결과를 받아 후속 처리에 연결한다.

```bash
ocr review --from "origin/main" --to "<commit_sha>" --format json
```

GitHub Actions와 GitLab CI 예제는 저장소의 examples/ 디렉토리에서 확인할 수 있다.

관찰성은 OpenTelemetry 통합으로 제공되며 기본적으로 비활성화되어 있다.
필요 시 다음과 같이 활성화한다.

```bash
ocr config set telemetry.enabled true
ocr config set telemetry.exporter otlp
ocr config set telemetry.otlp_endpoint localhost:4317
```

그 밖에 자주 쓰는 명령은 다음과 같다.

| 명령 | 설명 |
|------|------|
| ocr config provider | provider 설정 |
| ocr llm providers | 사용 가능한 provider 목록 |
| ocr viewer | WebUI 세션 뷰어, localhost:5483 |
| ocr version | 버전 확인 |

## 주의사항

벤치마크에서 확인되듯 이 도구는 재현율보다 정확도를 우선한다.
따라서 모든 결함을 빠짐없이 잡아내는 것을 목표로 한다면, 일부 결함이 누락될 수 있다는 점을 전제로 운용해야 한다.

리뷰 품질은 설정한 LLM과 규칙 파일에 크게 의존한다.
프로젝트 특성에 맞는 .opencodereview/rule.json을 작성하면 NPE, 스레드 안전성, XSS, SQL 인젝션 등 내장 규칙셋 외에도 팀의 컨벤션을 검토 기준에 반영할 수 있다.

환경변수가 수동 설정보다 우선순위가 높으므로, CI 환경에서 의도치 않은 값이 주입되지 않도록 우선순위를 명확히 관리해야 한다.

## 결론

Open Code Review는 알리바바가 2년간 사내 대규모 환경에서 검증한 AI 코드 리뷰 도구를 오픈소스로 공개한 결과물이다.
결정론적 엔지니어링으로 파일 선택과 규칙 적용을 고정하고, 에이전트로 판단을 맡기는 하이브리드 구조가 핵심이다.

Claude Code 대비 F1과 정확도를 높이면서 토큰 소비를 약 1/9 수준으로 줄였다는 점은 대규모 적용 시 비용과 품질을 동시에 고려하는 팀에게 매력적인 선택지가 된다.
NPM 한 줄로 설치되는 ocr 명령, 4계층 규칙 우선순위, Claude Code 플러그인과 CI/CD 통합까지 갖춰 실무 도입 부담도 낮은 편이다.

## Reference

- [Open Code Review (GitHub)](https://github.com/alibaba/open-code-review/)
