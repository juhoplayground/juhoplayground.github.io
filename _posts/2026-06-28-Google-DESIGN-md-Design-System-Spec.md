---
layout: post
title: "DESIGN.md: AI 에이전트를 위한 디자인 시스템 명세 포맷"
author: 'Juho'
date: 2026-06-28 00:00:00 +0900
categories: [AI]
tags: [AI, Documentation, Dev]
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
2. [DESIGN.md 포맷 구조](#designmd-포맷-구조)
   - [YAML 토큰 레이어](#yaml-토큰-레이어)
   - [마크다운 본문 레이어](#마크다운-본문-레이어)
3. [CLI 사용법](#cli-사용법)
   - [설치](#설치)
   - [lint diff export spec](#lint-diff-export-spec)
4. [linting 규칙과 소비자 동작](#linting-규칙과-소비자-동작)
5. [주의사항](#주의사항)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

DESIGN.md는 시각적 아이덴티티를 코딩 에이전트에게 전달하기 위한 포맷 명세다.
공식 설명에 따르면 DESIGN.md는 에이전트에게 디자인 시스템에 대한 지속적이고 구조화된 이해를 제공한다.
즉 색상, 타이포그래피, 간격 같은 디자인 토큰을 기계가 읽을 수 있는 형태로 정의하고, 동시에 사람이 읽을 수 있는 디자인 근거를 함께 담는 단일 파일이다.

이 포맷은 google-labs-code에서 공개했으며 라이선스는 Apache-2.0이다.
포맷 버전은 현재 alpha 단계이며 명세, 스키마, CLI가 활발히 개발되고 있다.
이 글에서는 DESIGN.md 파일의 구조, npx 기반 CLI 명령, 그리고 9개의 linting 규칙을 정리한다.

## DESIGN.md 포맷 구조

DESIGN.md 파일은 두 개의 레이어를 결합한다.
첫 번째는 기계가 읽는 YAML front matter로, 디자인 토큰을 담는다.
두 번째는 사람이 읽는 마크다운 본문으로, `##` 섹션을 사용해 디자인 근거를 설명한다.

### YAML 토큰 레이어

YAML front matter는 파일 상단의 구분자 사이에 위치하며 토큰 스키마를 따른다.
스키마의 전체 형태는 다음과 같다.

```yaml
version: <string>
name: <string>
description: <string>
colors:
  <token-name>: <Color>
typography:
  <token-name>: <Typography>
rounded:
  <scale-level>: <Dimension>
spacing:
  <scale-level>: <Dimension | number>
components:
  <component-name>:
    <token-name>: <string | token reference>
```

토큰 값은 몇 가지 타입으로 나뉜다.
각 타입과 형식은 아래 표와 같다.

| 타입 | 형식 | 예시 |
|------|------|------|
| Color | CSS 색상 형식 | 1A1C1E 형태의 hex, oklch 함수 표기 |
| Dimension | 숫자와 단위 | 48px, -0.02em |
| Token Reference | 점 경로 참조 | colors.primary 참조 |
| Typography | 폰트 속성 객체 | fontFamily, fontSize, fontWeight 등 |

Token Reference는 다른 토큰을 가리키는 참조로, 중괄호 안에 점 경로를 적는다.
예를 들어 색상 primary를 참조할 때는 {% raw %}{colors.primary}{% endraw %} 형태로 작성한다.
Typography 타입은 fontFamily, fontSize, fontWeight, lineHeight, letterSpacing, fontFeature, fontVariation 같은 속성을 갖는 객체다.

실제 토큰 블록의 예시는 다음과 같다.

```yaml
---
name: Heritage
colors:
  primary: "#1A1C1E"
  secondary: "#6C7278"
  tertiary: "#B8422E"
  neutral: "#F7F5F2"
typography:
  h1:
    fontFamily: Public Sans
    fontSize: 3rem
  body-md:
    fontFamily: Public Sans
    fontSize: 1rem
  label-caps:
    fontFamily: Space Grotesk
    fontSize: 0.75rem
rounded:
  sm: 4px
  md: 8px
spacing:
  sm: 8px
  md: 16px
---
```

components 항목은 컴포넌트 이름을 하위 토큰 속성 그룹에 매핑한다.
값에는 직접 값이나 토큰 참조를 사용할 수 있다.

```yaml
components:
  button-primary:
    backgroundColor: "{colors.tertiary}"
    textColor: "{colors.on-tertiary}"
    rounded: "{rounded.sm}"
    padding: 12px
  button-primary-hover:
    backgroundColor: "{colors.tertiary-container}"
```

컴포넌트에 사용할 수 있는 유효한 속성은 backgroundColor, textColor, typography, rounded, padding, size, height, width다.
hover, active, pressed 같은 변형(variant)은 관련된 키 이름을 가진 별도의 컴포넌트 항목으로 정의한다.

### 마크다운 본문 레이어

마크다운 본문은 `##` 헤딩을 사용한 섹션으로 구성되며 정해진 순서를 따라야 한다.
선택적 섹션은 생략할 수 있지만, 존재할 경우 아래 표의 순서를 지켜야 한다.

| 순서 | 섹션 | 별칭 |
|------|------|------|
| 1 | Overview | Brand & Style |
| 2 | Colors | 없음 |
| 3 | Typography | 없음 |
| 4 | Layout | Layout & Spacing |
| 5 | Elevation & Depth | Elevation |
| 6 | Shapes | 없음 |
| 7 | Components | 없음 |
| 8 | Do's and Don'ts | 없음 |

이 순서는 canonical order로 불리며, 어긋나면 linting에서 경고가 발생한다.

## CLI 사용법

DESIGN.md는 파일을 검증하고 비교하고 내보내는 CLI 도구를 제공한다.
모든 명령은 npx를 통해 실행할 수 있다.

### 설치

npm으로 패키지를 설치한 뒤 lint 명령을 실행할 수 있다.

```bash
npm install @google/design.md
npx @google/design.md lint DESIGN.md
```

Windows PowerShell 환경에서는 다음과 같은 대체 형식을 사용한다.

```bash
npx -p @google/design.md designmd lint DESIGN.md
```

### lint diff export spec

lint 명령은 구조적 정확성을 검증한다.
파일 경로를 인자로 받거나 stdin으로 입력을 받을 수 있으며, 오류가 있으면 종료 코드 1을 반환한다.

```bash
npx @google/design.md lint DESIGN.md
npx @google/design.md lint --format json DESIGN.md
cat DESIGN.md | npx @google/design.md lint -
```

lint의 출력 예시는 아래와 같으며, 심각도와 경로, 메시지를 포함한다.

```json
{
  "findings": [
    {
      "severity": "warning",
      "path": "components.button-primary",
      "message": "textColor (#ffffff) on backgroundColor (#1A1C1E) has contrast ratio 15.42:1 — passes WCAG AA."
    }
  ],
  "summary": { "errors": 0, "warnings": 1, "info": 1 }
}
```

diff 명령은 두 개의 DESIGN.md 파일을 비교해 토큰 수준의 변경을 찾는다.
before와 after 두 파일 경로를 받으며, 회귀(regression)가 감지되면 종료 코드 1을 반환한다.

```bash
npx @google/design.md diff DESIGN.md DESIGN-v2.md
```

diff의 출력은 섹션별로 추가, 제거, 수정된 토큰 목록과 회귀 여부를 담는다.

```json
{
  "tokens": {
    "colors": { "added": ["accent"], "removed": [], "modified": ["tertiary"] },
    "typography": { "added": [], "removed": [], "modified": [] }
  },
  "regression": false
}
```

export 명령은 토큰을 다른 포맷으로 내보낸다.
지원하는 포맷은 json-tailwind(또는 tailwind), css-tailwind, dtcg다.

```bash
npx @google/design.md export --format json-tailwind DESIGN.md > tailwind.theme.json
npx @google/design.md export --format css-tailwind DESIGN.md > theme.css
npx @google/design.md export --format dtcg DESIGN.md > tokens.json
```

각 export 포맷의 의미는 아래 표와 같다.

| 포맷 | 설명 |
|------|------|
| json-tailwind | Tailwind v3의 theme.extend JSON |
| css-tailwind | 커스텀 속성을 사용하는 Tailwind v4 CSS |
| dtcg | W3C Design Tokens Format Module JSON |

spec 명령은 포맷 명세 자체를 출력한다.
옵션을 통해 linting 규칙 표를 함께 출력하거나 규칙만 출력할 수 있다.

```bash
npx @google/design.md spec
npx @google/design.md spec --rules
npx @google/design.md spec --rules-only --format json
```

CLI 외에 프로그래밍 방식 API도 제공된다.
linter 모듈의 lint 함수에 마크다운 문자열을 넘기면 findings, summary, 파싱된 designSystem을 담은 리포트를 반환한다.

```javascript
import { lint } from '@google/design.md/linter';

const report = lint(markdownString);

console.log(report.findings);       // Finding[]
console.log(report.summary);        // { errors, warnings, info }
console.log(report.designSystem);   // Parsed DesignSystemState
```

## linting 규칙과 소비자 동작

DESIGN.md 파일은 9개의 linting 규칙으로 검증된다.
각 규칙은 error, warning, info 중 하나의 심각도를 가진다.

| 규칙 | 심각도 | 목적 |
|------|--------|------|
| broken-ref | error | 해석되지 않는 토큰 참조 |
| missing-primary | warning | 색상은 정의됐으나 primary 색상이 없음 |
| contrast-ratio | warning | 컴포넌트 텍스트와 배경 대비가 WCAG AA(4.5대1) 미만 |
| orphaned-tokens | warning | 컴포넌트가 참조하지 않는 색상 토큰 |
| token-summary | info | 섹션별 정의된 토큰 요약 |
| missing-sections | info | 토큰이 있는데 선택적 섹션이 없음 |
| missing-typography | warning | 색상은 있으나 타이포그래피 토큰이 없음 |
| section-order | warning | 섹션이 canonical order를 벗어남 |
| unknown-key | warning | 오타로 보이는 최상위 YAML 키(예: colours) |

규칙 중 broken-ref만 error 심각도이며, 나머지는 warning이나 info다.
contrast-ratio 규칙은 WCAG AA 기준인 4.5대1 대비를 검사한다.

파일을 소비하는 측의 동작 규칙도 정의되어 있다.
알 수 없는 항목을 어떻게 처리하는지는 아래 표와 같다.

| 상황 | 동작 |
|------|------|
| 알 수 없는 섹션 헤딩 | 오류 없이 보존 |
| 알 수 없는 색상 토큰 이름 | 값이 유효하면 허용 |
| 알 수 없는 타이포그래피 토큰 이름 | 유효한 것으로 허용 |
| 알 수 없는 컴포넌트 속성 | 경고와 함께 허용 |
| 중복된 섹션 헤딩 | 오류 처리하고 파일 거부 |

이처럼 DESIGN.md는 확장에 관대하면서도, 중복 섹션처럼 구조를 깨뜨리는 경우에는 파일을 거부하는 방식으로 설계되어 있다.

## 주의사항

DESIGN.md 포맷은 현재 alpha 버전이다.
명세, 스키마, CLI가 활발히 개발 중이므로 토큰 스키마나 규칙이 변경될 수 있다.
프로덕션에서 사용할 때는 버전 변화에 유의해야 한다.

또한 lint와 diff 명령은 오류나 회귀가 감지되면 종료 코드 1을 반환한다.
CI 파이프라인에 통합할 때 이 종료 코드를 활용하면 디자인 토큰의 회귀를 자동으로 차단할 수 있다.

## 결론

DESIGN.md는 디자인 시스템을 코딩 에이전트가 이해할 수 있는 단일 파일로 표현하는 포맷이다.
YAML front matter로 색상, 타이포그래피, 간격, 컴포넌트 토큰을 정의하고, 마크다운 본문으로 디자인 근거를 설명한다.
lint, diff, export, spec 네 가지 CLI 명령과 9개의 linting 규칙을 통해 파일의 구조적 정확성과 접근성을 검증할 수 있다.
디자인 토큰을 Tailwind나 W3C DTCG 형식으로 내보낼 수 있어 기존 도구 체인과의 연동도 가능하다.

## Reference

- [google-labs-code/design.md](https://github.com/google-labs-code/design.md)
