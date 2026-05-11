---
layout: post
title: "Polaris MCFG: 라이선스 안전한 메트릭 호환 폰트 생성 CLI"
author: 'Juho'
date: 2026-05-11 00:00:00 +0900
categories: [Dev]
tags: [Python, Dev, Documentation]
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
3. [핵심 동작 원리](#핵심-동작-원리)
   - [5단계 파이프라인](#5단계-파이프라인)
   - [전송되는 메트릭 테이블](#전송되는-메트릭-테이블)
4. [CLI 사용법](#cli-사용법)
   - [주요 명령](#주요-명령)
   - [실전 예시](#실전-예시)
5. [라이브 데모 분석](#라이브-데모-분석)
6. [제약과 한계](#제약과-한계)
7. [라이선스와 의미](#라이선스와-의미)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Polaris MCFG(Metric-Compatible Font Generator)는 폴라리스오피스가 공개한 Python 기반 CLI 도구이다.
독점 폰트의 레이아웃 메트릭만 추출하여 자유 라이선스 폰트의 글리프 디자인에 적용한다.
글리프 외곽선(outline)은 절대 복사하지 않으면서 줄바꿈과 페이지네이션을 그대로 유지하는 합성 폰트를 만든다.
[GitHub 저장소](https://github.com/PolarisOffice/polaris_mcfg){:target="_blank"}와 [라이브 데모](https://polarisoffice.github.io/polaris_mcfg/demo/){:target="_blank"}가 공개되어 있다.

## 배경

기업 환경에서는 한컴 폰트 같은 독점 typeface를 표준으로 사용하는 경우가 많다.
이런 폰트는 특정 advance width, ascender/descender height, line gap을 가지고 있고, 문서의 줄바꿈과 페이지 분할이 이 메트릭에 의존한다.
재배포가 가능한 자유 라이선스 폰트(Noto Sans KR, Pretendard 등)로 단순 교체하면 동일한 텍스트라도 다른 위치에서 줄이 바뀌고 페이지가 어긋난다.

Polaris MCFG는 이 문제를 정공법으로 해결한다.
글리프 외곽선이 아닌 숫자 메트릭만 옮기기 때문에 법적 안전성을 유지한 채 레이아웃 호환성을 확보할 수 있다.
저장소 설명에 따르면 "extract layout metrics from a source font and apply them to a freely-licensed font's glyph design"이 핵심 미션이다.

## 핵심 동작 원리

### 5단계 파이프라인

도구는 다섯 단계로 구성된 파이프라인을 통해 동작한다.

| 단계 | 역할 |
|------|------|
| Extraction | 소스 폰트(TTF)에서 메트릭을 JSON 스펙으로 추출 |
| Comparison | 소스와 디자인 폰트의 메트릭 차이를 분석 |
| Generation | 디자인 폰트의 글리프와 소스 메트릭을 결합한 신규 폰트 합성 |
| Validation | 출력 폰트가 메트릭 요건을 만족하는지 검증 |
| Rendering regression | HarfBuzz 라인 폭 일관성 회귀 테스트 |

각 단계는 독립적인 CLI 서브커맨드로 노출되어 있어 자동화 파이프라인에 통합하기 쉽다.
M1부터 M7까지 7개 마일스톤이 모두 완료된 상태이며, HTML 리포팅과 프로덕션 패키징까지 포함된다.

### 전송되는 메트릭 테이블

표준으로 추출되는 메트릭은 advance width, ascender/descender height, line gap, global offsets이다.
M5 이후 옵션 메트릭으로 left side bearing(LSB), kerning pair, vertical metrics가 추가되었다.

데모 페이지의 기술 설명에 따르면 실제로 전송되는 OpenType 테이블은 `hmtx`, `OS/2`, `head`이다.
GSUB lookup table은 outline 라이선스가 필요하므로 의도적으로 제외된다.
이 결정 덕분에 도구는 메트릭 전송이라는 좁은 영역에 집중하면서 라이선스 침해 위험을 차단한다.

## CLI 사용법

### 주요 명령

| Command | Function |
|---------|----------|
| `mcfg extract` | 폰트 메트릭을 JSON으로 내보내기 |
| `mcfg compare` | 두 폰트의 메트릭 차이 분석(text/json/html) |
| `mcfg generate` | 새 폰트 합성 |
| `mcfg validate` | 메트릭 준수 여부 검증 |

생성 단계에서는 여러 옵션을 통해 동작을 제어할 수 있다.
`--apply global,advance`는 전송할 메트릭 범위를 지정한다.
`--scale-glyph none/fit/center`는 글리프 스케일링 전략을 결정하며, `center`는 글리프를 advance box 중심에 정렬해 레이아웃 동작을 보존한다.
`--family-name`, `--style-name`, `--license-text`, `--license-url`로 출력 폰트의 메타데이터를 지정한다.

### 실전 예시

다음은 NotoSansKR Bold의 메트릭을 추출하여 NotoSansKR Regular에 적용하는 예시이다.

```bash
# Bold 메트릭 추출
mcfg extract NotoSansKR-Bold.ttf --deterministic -o bold.json

# 새 폰트 생성
mcfg generate \
  --metrics bold.json \
  --design NotoSansKR-Regular.ttf \
  --output PolarisBoldMetrics-Regular.ttf \
  --apply global,advance

# 원본 대비 검증
mcfg validate PolarisBoldMetrics-Regular.ttf \
  --against NotoSansKR-Bold.ttf \
  --render-default
```

`--deterministic` 플래그는 추출 결과를 재현 가능하게 만들어 빌드 산출물 검증에 유용하다.
`--render-default`는 HarfBuzz 기반 렌더링 회귀 테스트를 함께 수행한다.

## 라이브 데모 분석

데모 페이지는 두 개의 메트릭 그룹을 제공하여 도구의 효과를 직관적으로 보여준다.
하나는 NotoSansKR 메트릭 기반이고, 다른 하나는 Pretendard 사양을 사용한다.
두 합성 폰트는 각각 Polaris NPM, Polaris PNM이라는 이름으로 SIL Open Font License 1.1 하에 배포된다.

데모는 다음 네 가지 시나리오를 검증한다.

첫째, 메트릭 정렬 렌더링이다.
같은 메트릭 그룹에 속한 폰트들은 시각적으로 다른 글리프를 사용하더라도 동일한 위치에서 줄을 바꾼다.

둘째, 다중 스크립트 지원이다.
한글, Latin, 숫자, 구두점 모두 두 메트릭 그룹에서 일관성을 보인다.

셋째, 언어 의존 동작이다.
`lang="ko"` 속성이 트리거하는 GSUB 치환이 공백에 영향을 준다.
NotoSansKR은 한국어 컨텍스트에서 공백을 약 25% 확장하지만 Pretendard는 이 동작이 없다.
이 차이는 동일 텍스트라도 메트릭 그룹별로 미묘한 시각 차이를 유발한다.

넷째, 시각적 일관성 테스트이다.
10–48px 사이즈 래더, 데이터 테이블, 단락 비교를 통해 합성 폰트가 다양한 스케일과 컨텍스트에서 원본 메트릭 특성을 유지하는지 확인한다.

데모에 사용된 폰트는 NotoSansKR과 Pretendard 모두 OFL 라이선스이며, WOFF2 형식으로 폰트당 약 50KB로 압축되어 있다.
NotoSansKR의 UPM은 1000, Pretendard는 2048이다.

## 제약과 한계

도구는 의도적으로 좁은 범위에 집중한다.
다음 항목은 현재 지원하지 않는다.

| 제약 | 설명 |
|------|------|
| 포맷 | TTF만 지원, WOFF2와 가변 폰트 미지원 |
| 스크립트 | RTL과 Indic 스크립트 미지원 |
| 마크 포지셔닝 | 다이어크리틱/액센트 메트릭 제외 |
| 외곽선 | 글리프 path는 절대 접근하거나 복사하지 않음 |

이 제약은 한국어 비즈니스 문서라는 주 사용 사례에는 영향이 거의 없지만, 다국어 타이포그래피 시스템에 도입할 때는 사전 평가가 필요하다.

프로젝트 구조는 `src/polaris_mcfg/` 코어 모듈, 62개의 단위 테스트, 8개의 아키텍처 설계 문서, 데모 스크립트 샘플로 구성된다.
Python 3.10 이상을 요구한다.

## 라이선스와 의미

도구 코드는 MIT License로 공개되어 있다.
생성된 폰트는 디자인 폰트의 라이선스(보통 OFL)를 그대로 상속한다.
중요한 점은 소스 폰트의 EULA가 메트릭 추출을 허용하는지 사용자가 직접 확인해야 한다는 것이다.

이 도구는 폰트 라이선스 컴플라이언스라는 좁고 까다로운 문제를 정확히 겨냥한다.
재배포 가능한 폰트로 강제 마이그레이션해야 하는 조직, 오피스 문서 호환성을 유지하려는 SaaS, 폰트 임베디드 PDF를 다루는 출판 워크플로에서 즉시 가치가 있다.
글리프 외곽선과 메트릭을 명확히 분리하고, 외곽선은 절대 만지지 않는다는 설계 원칙은 법무 검토 부담을 크게 낮춘다.

## 결론

Polaris MCFG는 "메트릭만 옮기고 외곽선은 절대 만지지 않는다"는 단순한 원칙으로 폰트 라이선스와 레이아웃 호환성 사이의 오랜 트레이드오프를 해소한다.
extract → compare → generate → validate → render로 이어지는 명확한 파이프라인과 CLI는 자동화하기 쉽고, 7개 마일스톤이 모두 완료되어 프로덕션 사용 준비가 되어 있다.
한국어 비즈니스 문서 호환성을 고민하는 팀이라면 도입 검토 가치가 충분한 도구이다.
공개된 GitHub 저장소와 라이브 데모를 통해 실제 동작을 직접 확인할 수 있다.

## Reference

- [Polaris MCFG GitHub Repository](https://github.com/PolarisOffice/polaris_mcfg/)
- [Polaris MCFG Live Demo](https://polarisoffice.github.io/polaris_mcfg/demo/)
