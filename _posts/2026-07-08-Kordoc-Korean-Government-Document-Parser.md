---
layout: post
title: "Kordoc: 한국 공문서 지옥을 파싱해버리는 범용 파서"
author: 'Juho'
date: 2026-07-08 00:00:00 +0900
categories: [Dev]
tags: [Documentation, PDF, MCP]
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
2. [핵심 기능](#핵심-기능)
   - [문서 변환과 고급 기능](#문서-변환과-고급-기능)
   - [기술 스택과 아키텍처](#기술-스택과-아키텍처)
3. [사용 방법](#사용-방법)
   - [설치와 CLI](#설치와-cli)
   - [주요 API](#주요-api)
4. [성능과 보안](#성능과-보안)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

Kordoc은 "대한민국에서 둘째가라면 서러울 문서지옥"을 겨냥한 한국 공문서 범용 파서다.
작성자는 광진구청에서 7년간 한글 파일을 다룬 지방공무원으로, 실제 관공서 문서 수천 건을 검증하며 개발했다.
HWP3, HWP5, HWPX, HWPML, PDF, XLS, XLSX, DOCX까지 8가지 포맷을 통일된 마크다운으로 변환하는 것이 핵심이다.

## 핵심 기능

Kordoc의 목표는 단순 명료하다.
"모두 파싱해버리겠다"는 것이다.

### 문서 변환과 고급 기능

8가지 포맷을 마크다운으로 추출하는 것을 넘어, 공문서 업무에 필요한 고급 기능을 함께 제공한다.

| 기능 | 설명 |
|------|------|
| 문서 변환 | 8가지 포맷을 통일된 마크다운으로 추출 |
| 신구대조 생성 | 두 문서 간 차이를 분석해 변화 지점 가시화 |
| 양식 자동 채우기 | 공문서 템플릿 빈칸을 규칙 기반으로 채움 |
| 서식 보존 패치 | 편집된 마크다운을 원본에 반영하되 형식 유지 |
| 역변환 생성 | 마크다운을 HWPX 공문서로 생성 (프리셋 포함) |
| 레이아웃 렌더링 | HWPX를 SVG로 렌더링해 미리보기 제공 |
| MCP 통합 | Claude Desktop, Cursor 등 AI 클라이언트와 직접 연동 |

### 기술 스택과 아키텍처

TypeScript와 Node.js 18 이상 환경에서 동작한다.
주요 의존성으로 PDF 처리에 pdfjs-dist, ZIP 기반 포맷에 JSZip, OLE2 컨테이너 파싱에 cfb를 사용한다.
HWP5 복호화를 위한 rhwp, 테이블 감지 알고리즘을 위한 OpenDataLoader PDF 등 오픈소스 기여도 포함되어 있다.

내부 처리 흐름은 입력 포맷을 포맷별 파서로 처리한 뒤, 통일된 중간 표현(IR)으로 변환하는 구조다.

```
입력 (8가지 포맷)
    ↓
포맷별 파서 (parseHwpx, parseHwp, parsePdf 등)
    ↓
통일된 IR 구조 (IRBlock[], DocumentMetadata)
    ↓
마크다운 + 메타데이터 + 블록 정보
    ↓
고급 기능 (비교, 필드 추출, 패치, 생성)
```

IRBlock은 heading, paragraph, table, list, image, separator 타입으로 구성되며, 각각 bbox, style, pageNumber 정보를 담는다.

## 사용 방법

### 설치와 CLI

AI 에이전트 연동을 위한 가장 빠른 방법은 setup 마법사다.

```bash
npx -y kordoc setup
```

대화형 마법사가 설치된 AI 클라이언트를 감지하고 설정을 자동으로 패치한다.
npm 설치 후 CLI로 직접 다양한 작업을 수행할 수도 있다.

```bash
npx kordoc 사업계획서.hwpx                     # 터미널 출력
npx kordoc 보고서.hwp -o 보고서.md             # 파일 저장
npx kordoc fill 신청서.hwpx -f '성명=홍길동'   # 양식 채우기
npx kordoc generate 보고서.md -o 보고서.hwpx   # 마크다운을 HWPX로
npx kordoc render 결재문서.hwpx -o 미리보기.svg # 레이아웃 렌더
```

### 주요 API

코드에서는 포맷 자동 감지 파싱부터 서식 보존 패치까지 API로 사용할 수 있다.

```typescript
// 파싱 (포맷 자동 감지)
const result = await parse(buffer)
if (result.success) {
  console.log(result.markdown)   // 마크다운 텍스트
  console.log(result.blocks)     // 구조화 데이터
}

// 양식 채우기 (원본 서식 유지)
const filled = await fillForm(template,
  { 성명: "홍길동", 주소: "서울" },
  { format: "hwpx-preserve" }
)

// 문서 비교 (크로스 포맷 지원)
const diff = await compare(oldDoc, newDoc)

// 서식 보존 패치 (변경 텍스트만 반영)
const patched = await patchHwpx(원본, 편집된마크다운)
```

MCP를 통해서는 parse_document, compare_documents, fill_form, generate_document, place_seal 등 11개 도구가 자동 활성화되어 AI 클라이언트에서 직접 호출된다.

## 성능과 보안

v3.0 기준 정확도 벤치마크는 상당히 높다.

| 지표 | 정확도 |
|------|------|
| HWPX 텍스트 재현율 | 99.998% |
| 표 구조 정확도 | 100% (1,421개 표 기준) |
| PDF 커버리지 | 99.16% |
| HWP5-HWPX 유사도 | 99.94% |

이미지를 대량 참조하는 문서에서는 메모리 사용량을 17GB에서 445MB로 개선했다.
보안 측면에서도 ZIP bomb 방지, XXE/Billion Laughs 공격 방어, 경로 순회 차단, 500MB 파일 크기 제한, MCP 에러 정제 등 프로덕션급 강화가 적용되어 있다.

## 결론

Kordoc은 한국 공공부문 특유의 복잡한 문서 생태계를 실무자의 경험에서 출발해 자동화한 프로젝트다.
HWP 계열부터 PDF, Office 포맷까지 하나의 마크다운 파이프라인으로 묶고, 신구대조와 양식 채우기 같은 실제 업무 기능을 MCP로 AI 에이전트에 연결한 점이 인상적이다.
관공서 문서를 다루는 개발자나 자동화가 필요한 실무자에게 실질적인 도구가 될 만하다.

## Reference

- [chrisryugj/kordoc](https://github.com/chrisryugj/kordoc/)
