---
layout: post
title: "Codex의 SQLite 로그가 연간 640TB를 기록해 SSD를 갉아먹은 사건"
author: 'Juho'
date: 2026-06-24 00:00:00 +0900
categories: [AI]
tags: [AI, Database, Dev]
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
2. [무슨 일이 일어났나](#무슨-일이-일어났나)
   - [피해 규모](#피해-규모)
3. [증거와 근본 원인](#증거와-근본-원인)
   - [로그 구성 분석](#로그-구성-분석)
   - [쓰기 증폭](#쓰기-증폭)
4. [해결책과 결과](#해결책과-결과)
5. [결론](#결론)
6. [Reference](#reference)

## 개요

OpenAI Codex의 GitHub 이슈 #28224는, AI 코딩 도구가 로컬 SQLite 피드백 로그에 과도한 데이터를 기록해 SSD 수명을 급격히 단축시킨 문제를 다룬다.
작성자는 @1996fanrui이며, CLI, bug, performance 라벨이 붙었다.
요지는 간단하다.
Codex가 백그라운드에서 너무 많은 로그를 쉼 없이 기록한 탓에, SSD가 1년 안에 내구성 한계에 도달할 수 있다는 것이다.

## 무슨 일이 일어났나

영향을 받은 파일은 다음 세 가지다.

```bash
~/.codex/logs_2.sqlite
~/.codex/logs_2.sqlite-wal
~/.codex/logs_2.sqlite-shm
```

### 피해 규모

21일 운영 기간 동안 메인 SSD에 약 37TB가 기록되었다.

| 항목 | 값 |
|------|------|
| 21일간 기록량 | 약 37TB |
| 연간 추정 기록량 | 약 640TB |
| 1TB SSD 기준 | 연간 약 640회 전체 용량 기록 |
| 일반 SSD 내구성 | 약 600TBW |

일반적인 SSD 내구성 수준이 약 600TBW인데, 연간 640TB를 기록하면 1년 이내에 내구성 한계에 도달할 수 있다.
즉 AI 코딩 도구를 켜두기만 했는데 SSD가 빠르게 소모되는 셈이다.

## 증거와 근본 원인

작성자는 정황이 아니라 데이터로 문제를 입증했다.
logs_2.sqlite 파일 크기는 1.2 GiB였는데, 현재 보유 행 수는 506,149개인 반면 총 할당된 행 ID는 약 55억 4천만 개였다.
보유 행과 누적 ID 사이에 약 1만 배 차이가 나는데, 이는 10TB 이상 규모의 과거 로그가 삽입과 삭제를 반복했다는 의미다.

### 로그 구성 분석

로그 수준별로 보면 대부분이 TRACE 레벨이었다.

| 로그 수준 | 추정 크기 | 비율 |
|------|------|------|
| TRACE | 732.5 MiB | 70.7% |
| INFO | 266.5 MiB | 25.7% |
| DEBUG | 30.6 MiB | 3.0% |
| WARN | 5.9 MiB | 0.6% |

가장 큰 원본은 WebSocket 응답을 기록하는 `codex_api::endpoint::responses_websocket`(TRACE)으로 527.4 MiB를 차지했다.
그 뒤를 OpenTelemetry 로그인 `codex_otel.log_only`와 `codex_otel.trace_safe`가 이었다.
TRACE와 OpenTelemetry 로그만으로 전체의 약 96%를 차지했다.

### 쓰기 증폭

15초 샘플링 결과가 문제를 명확히 보여준다.
보유 행 수는 681,774개로 변화가 없었는데, 최대 행 ID는 약 3만 6천 개나 증가했다.
즉 새 행을 계속 삽입하면서 오래된 행을 즉시 삭제하는 패턴이 반복되어, 보유량은 그대로인데 디스크 쓰기만 폭증한 것이다.
근본 원인은 SQLite 피드백 로그 싱크가 전역 TRACE를 기본값으로 설정한 데 있었다.

```bash
Targets::new().with_default(Level::TRACE)
```

이 때문에 의존성 로그, 내부 로그, 대용량 프로토콜 페이로드가 전부 보관되었다.

## 해결책과 결과

작성자는 다음과 같은 개선을 제안했다.

1. SQLite 피드백 로그 싱크에서 전역 TRACE 사용 중단
2. 저가치 의존성 노이즈 필터링
3. 원본 WebSocket 페이로드 대신 요약만 저장(이벤트 종류, 소요 시간, 성공 여부, 토큰 사용량 등)
4. 중복 OpenTelemetry 로그 제거
5. 전역 로그 DB 크기와 쓰기 제한 설정

이 이슈는 2026년 6월 22일 두 개의 PR이 병합되며 종료되었다.

| PR | 내용 |
|------|------|
| PR #29432 | 모든 Responses WebSocket 이벤트 로깅 중단 |
| PR #29457 | 영구 로그에서 시끄러운 대상 필터링 |

작성자는 이 두 PR로 문제가 약 85% 감소했음을 확인하고 이슈를 닫았다.

## 결론

이 사건은 AI 도구의 관찰 가능성(observability) 설계가 사용자 하드웨어에 직접적인 비용을 전가할 수 있음을 보여준다.
디버깅을 위해 전역 TRACE를 기본값으로 두면 편하지만, 대용량 프로토콜 페이로드까지 영구 저장하면 삽입-삭제 증폭으로 SSD 쓰기가 폭증한다.
로그 레벨 기본값, 페이로드 요약, 노이즈 필터링, 로그 DB 크기 제한은 단순한 디테일이 아니라 사용자 환경을 지키는 안전장치다.
유사한 I/O 및 로그 문제 이슈가 다수 연결되어 있다는 점도, 이 문제가 한 번의 실수가 아니라 구조적으로 주의해야 할 영역임을 시사한다.

## Reference

- [openai/codex Issue #28224](https://github.com/openai/codex/issues/28224)
