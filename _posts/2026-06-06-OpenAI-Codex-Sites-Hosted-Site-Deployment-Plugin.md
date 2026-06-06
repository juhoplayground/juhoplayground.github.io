---
layout: post
title: "Codex Sites: 프롬프트에서 호스팅 사이트까지, OpenAI의 배포 플러그인"
author: 'Juho'
date: 2026-06-06 00:00:00 +0900
categories: [Dev]
tags: [AI, Agent, OpenAI, VibeCoding]
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
2. [시작하기](#시작하기)
   - [활성화와 플러그인 추가](#활성화와-플러그인-추가)
   - [Sites 작업 시작하기](#sites-작업-시작하기)
3. [프로젝트·버전·배포 이해하기](#프로젝트버전배포-이해하기)
   - [저장과 배포의 두 단계](#저장과-배포의-두-단계)
   - [지원되는 사이트 형태 선택](#지원되는-사이트-형태-선택)
4. [접근 제어와 시크릿 관리](#접근-제어와-시크릿-관리)
5. [공유 전 검토 항목](#공유-전-검토-항목)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

Sites는 Codex가 OpenAI가 호스팅하는 웹사이트, 웹 앱, 게임을 생성·저장·배포·검사할 수 있게 하는 플러그인이다.

별도의 배포 워크플로우를 구성하지 않고도 프롬프트나 호환되는 기존 프로젝트를 호스팅 사이트로 전환하고 싶을 때 사용한다.
중요한 점은, 모든 Sites 배포 URL이 프로덕션 배포라는 것이다.
빌드를 라이브로 전환하기 전에 검토하고 싶다면 Codex에게 배포 없이 버전만 저장하도록 요청해야 한다.

## 시작하기

Sites는 프리뷰 단계이며 현재 ChatGPT Business와 Enterprise 워크스페이스에서 사용 가능하다(추후 더 많은 플랜으로 확대 예정).

### 활성화와 플러그인 추가

ChatGPT Enterprise를 사용한다면 워크스페이스 관리자가 RBAC(역할 기반 접근 제어) 설정에서 해당 역할에 Sites를 켜야 한다.
ChatGPT Business 워크스페이스는 Sites가 기본 활성화되어 있어 이 단계를 건너뛸 수 있다.

Sites가 아직 보이지 않으면 Codex 앱의 Plugins에서 Sites를 찾아 추가한다.
플러그인 설치 후에는 새 스레드를 시작해야 한다.

### Sites 작업 시작하기

스레드에서 만들거나 게시하고 싶은 사이트를 설명한다.
작업이 호스팅 배포로 끝나야 할 때는 `@Sites`로 플러그인을 명시적으로 호출할 수 있다.

새 웹사이트, 대시보드, 내부 도구를 만들 때는 대상 사용자, 핵심 경험, 필요한 데이터를 포함해 요청한다.

```text
@Sites Build a project request dashboard for my operations team. Let team
members submit requests, see who owns each one, update the status, and filter
the list. Require people to sign in with their workspace account, and keep the
request data saved between visits.
```

기존 프로젝트를 게시할 때는 다음과 같이 요청한다.

```text
@Sites Deploy this project. Check whether it is compatible with Sites, make any
required changes, and give me the deployment URL.
```

배포 후에는 앱 사이드바의 Sites에서 프로젝트로 돌아갈 수 있고, 저장된 버전 검사, 배포 상태 확인, 접근 권한 변경 등을 요청할 수 있다.

## 프로젝트·버전·배포 이해하기

Sites 프로젝트는 로컬 소스 프로젝트를 Sites가 관리하는 호스팅에 연결한다.

Codex는 이 연결과 선택적 스토리지 바인딩 이름을 `.openai/hosting.json`에 저장한다.
새로 만든 로컬 스타터는 `project_id` 없이 시작할 수 있으며, Sites가 호스팅 프로젝트를 프로비저닝한 뒤 추가한다.

관계형 데이터베이스 바인딩을 사용하고 파일 스토리지는 없는 프로비저닝된 사이트의 예시는 다음과 같다.

```json
{
  "project_id": "<project-id>",
  "d1": "DB",
  "r2": null
}
```

### 저장과 배포의 두 단계

Sites 게시는 분리된 두 단계로 이루어진다.

| 단계 | 설명 |
|------|------|
| 버전 저장 | Codex가 배포 가능한 사이트를 빌드하고 빌드에 사용된 소스 Git 커밋과 연결. 검토용 배포 후보를 원할 때 사용 |
| 버전 배포 | 저장된 버전을 게시하고 성공 시 프로덕션 URL 보고. 선택된 대상이 사이트에 접근하기를 의도할 때만 사용 |

이전 배포 후보를 식별해야 할 때는 Codex에게 저장된 버전을 나열하거나 검사하도록 요청할 수 있다.

### 지원되는 사이트 형태 선택

Sites는 Cloudflare Worker 호환 출력을 ES 모듈로 빌드하는 프로젝트를 호스팅한다.

필요한 제품 동작을 Codex에게 알려주면 적절한 사이트 형태를 선택해준다.

| 사이트 요구 | Sites에 요청할 것 |
|------------|------------------|
| 콘텐츠 중심 웹사이트나 랜딩 페이지 | 경험상 필요하지 않다면 지속 상태가 없는 사이트 |
| 저장된 레코드, 사용자 진행, 게임 점수 | D1, 내구성 있는 구조적 데이터를 위한 관계형 DB |
| 이미지, 문서, 오디오, 비디오 등 업로드 | R2, 파일을 위한 오브젝트 스토리지 |
| 검색 가능한 메타데이터가 있는 업로드 파일 | 메타데이터는 D1, 파일 내용은 R2 |
| 현재 워크스페이스 사용자 신원이 필요한 내부 사이트 | 워크스페이스 인증 사용자 신원 |
| 공개 로그인이나 외부 신원 공급자 | 인증이 활성화된 Sites 프로젝트 |

테마 선택이나 닫은 배너처럼 일시적 표현 상태에는 지속 스토리지를 요청하지 않는다.
사용자가 사이트가 기억하기를 기대하는 제품 데이터에만 스토리지를 요청한다.

## 접근 제어와 시크릿 관리

배포 URL을 공유하기 전에 대상을 설정해야 한다.

새 사이트는 콘텐츠, 데이터 처리, 예상 대상을 검토할 때까지 소유자와 워크스페이스 관리자로 접근을 제한하는 것이 좋다.

| 접근 모드 | 접근 가능한 대상 |
|-----------|-----------------|
| 소유자와 관리자 (admins_only) | 사이트 소유자와 워크스페이스 관리자 |
| 워크스페이스 (workspace_all) | 워크스페이스의 모든 활성 사용자 |
| 커스텀 (custom) | 직접 선택한 특정 활성 사용자나 그룹(소유자는 항상 허용) |

런타임 환경 값과 시크릿은 앱 사이드바의 Sites 패널에서 추가·수정·제거한다.
이 값들을 `.openai/hosting.json`에 저장해서는 안 된다.
로컬 `.env`와 `.env.example` 파일은 로컬 개발에 필요한 키와 정렬하되 시크릿 값은 커밋하지 않는다.

## 공유 전 검토 항목

배포하거나 접근을 넓히기 전에 다음을 확인한다.

- Codex 리뷰 창에서 소스 변경과 DB 마이그레이션 검토
- 빌드 성공 여부와 게시하려는 저장 버전이 맞는지 확인
- 의도한 대상만 사이트에 접근 가능한지 확인
- 런타임 시크릿을 Sites로 설정했고 소스 파일에 커밋하지 않았는지 확인
- 배포 후 Codex에게 배포 상태와 프로덕션 URL을 확인한 뒤 공유

## 결론

Codex Sites는 프롬프트나 기존 프로젝트를 별도 배포 파이프라인 없이 OpenAI 호스팅 사이트로 전환한다.

저장과 배포를 분리한 두 단계 워크플로우, D1/R2 스토리지와 다양한 인증 모드, 세밀한 접근 제어와 시크릿 관리를 제공한다.
모든 배포가 프로덕션이라는 점을 유념하고, 라이브 전환 전 저장·검토 단계를 활용하는 것이 안전하다.

## Reference

- [Codex Sites](https://developers.openai.com/codex/sites){:target="_blank"}
