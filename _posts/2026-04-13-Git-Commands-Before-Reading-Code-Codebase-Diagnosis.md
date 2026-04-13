---
layout: post
title: "코드를 읽기 전에 실행하는 5가지 Git 명령어"
author: 'Juho'
date: 2026-04-13 00:00:00 +0900
categories: [Dev]
tags: [Dev]
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
2. [핵심 내용](#핵심-내용)
   - [자주 변경되는 파일 파악](#자주-변경되는-파일-파악)
   - [팀 구성 파악](#팀-구성-파악)
   - [버그 집중 영역 파악](#버그-집중-영역-파악)
   - [프로젝트 동력 추적](#프로젝트-동력-추적)
   - [소방 패턴 감지](#소방-패턴-감지)
3. [결론](#결론)
4. [Reference](#reference)

## 개요

새로운 코드베이스를 분석할 때 코드를 열기 전에 Git 이력을 통해 프로젝트의 건강 상태를 빠르게 진단할 수 있다.
Ally Piechowski가 제시한 5가지 Git 명령어는 몇 분 만에 실행 가능하며, 문제 영역과 팀 동향, 프로젝트 모멘텀을 파악하는 체계적 분석 방향을 제시한다.

## 핵심 내용

### 자주 변경되는 파일 파악

```bash
git log --format=format: --name-only --since="1 year ago" | sort | uniq -c | sort -nr | head -20
```

최근 1년간 가장 많이 수정된 상위 20개 파일을 나열한다.
목록 상단의 파일은 거의 항상 사람들이 경고하는 파일이다.
높은 변경 빈도는 변경에 저항하는 코드를 의미한다.

2005년 Microsoft Research 연구에 따르면 변경 빈도 기반 메트릭이 복잡도 메트릭만으로는 예측하기 어려운 결함을 더 안정적으로 예측했다.
이 파일들을 버그 핫스팟과 교차 참조하면 가장 위험한 코드를 식별할 수 있다.

### 팀 구성 파악

```bash
git shortlog -sn --no-merges
```

커밋 수 기준으로 기여자 순위를 매긴다.
한 사람이 커밋의 60% 이상을 차지하면 버스 팩터 위험이 있다.
원래 설계자가 여전히 시스템을 유지보수하는지도 확인할 수 있다.

```bash
git shortlog -sn --no-merges --since="6 months ago"
```

최근 6개월 활동 수준을 추가로 확인하면 현재 유지보수 상태를 파악할 수 있다.

### 버그 집중 영역 파악

```bash
git log -i -E --grep="fix|bug|broken" --name-only --format='' | sort | uniq -c | sort -nr | head -20
```

fix, bug, broken 등의 키워드가 포함된 커밋에서 자주 등장하는 파일을 매핑한다.
변경 빈도가 높은 파일 목록과 버그 집중 파일 목록 모두에 나타나는 파일은 계속 고장나고 패치되지만 제대로 수정되지 않는 코드이다.
다만 이 방법은 커밋 메시지의 일관성에 의존한다는 한계가 있다.

### 프로젝트 동력 추적

```bash
git log --format='%ad' --date=format:'%Y-%m' | sort | uniq -c
```

월별 커밋 빈도를 프로젝트 전체 이력에 걸쳐 보여준다.
꾸준한 리듬은 안정성을 나타내고, 감소 곡선은 모멘텀 상실을 시사한다.
급증 후 조용한 기간은 지속적 배포가 아닌 배치 릴리스 워크플로우를 의미한다.

### 소방 패턴 감지

```bash
git log --oneline --since="1 year ago" | grep -iE 'revert|hotfix|emergency|rollback'
```

최근 1년간 revert과 hotfix 커밋 수를 측정한다.
빈번한 revert는 배포 프로세스 문제, 테스트 부족, 또는 스테이징 환경 미비를 나타낸다.
이러한 커밋의 존재 여부가 팀 안정성을 명확하게 보여준다.

## 결론

이 5가지 Git 명령어는 코드 한 줄을 열기 전에 코드베이스의 건강 상태를 체계적으로 진단하는 첫 단계이다.
고위험 코드를 향해 조사를 방향 짓고, 코드 검사만으로는 알 수 없는 팀 역학을 드러낸다.
새로운 프로젝트에 합류하거나 코드 감사를 시작할 때 가장 먼저 실행해볼 만한 명령어들이다.

## Reference

- [The Git Commands I Run Before Reading Any Code](https://piechowski.io/post/git-commands-before-reading-code/)
