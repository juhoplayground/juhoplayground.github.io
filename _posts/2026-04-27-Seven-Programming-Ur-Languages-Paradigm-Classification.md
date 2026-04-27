---
layout: post
title: "일곱 가지 프로그래밍 원형 언어: 파라다임으로 언어를 분류하기"
author: 'Juho'
date: 2026-04-27 00:00:00 +0900
categories: [Dev]
tags: [Dev, Compiler]
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
2. [일곱 원형 언어](#일곱-원형-언어)
   - [ALGOL 계열](#algol-계열)
   - [Lisp 계열](#lisp-계열)
   - [ML 계열](#ml-계열)
   - [Self 계열](#self-계열)
   - [Forth 계열](#forth-계열)
   - [APL 계열](#apl-계열)
   - [Prolog 계열](#prolog-계열)
3. [학습 로드맵](#학습-로드맵)
4. [결론](#결론)
5. [Reference](#reference)

## 개요

프로그래밍 언어는 겉보기에 수백 개로 다양해 보이지만 실제로는 일곱 개의 원형(ur-language) 주변에 클러스터링된다는 주장이 madhadron 블로그에서 제기됐다.
저자는 같은 원형 계열 안에서는 언어 간 이동이 쉽지만 다른 원형으로 넘어가려면 새로운 사고 회로를 형성해야 한다고 말한다.
즉 언어 문법을 익히는 것보다 각 파라다임이 요구하는 신경 회로(neural pathway)를 발달시키는 것이 학습의 본질이라는 것이다.

## 일곱 원형 언어

저자가 제시하는 일곱 원형은 ALGOL, Lisp, ML, Self, Forth, APL, Prolog이다.
각 원형은 반복, 재귀, 조합 방식에서 근본적으로 다른 접근을 취한다.

### ALGOL 계열

대입, 조건문, 루프를 순차적으로 구성하고 이를 함수로 묶는 명령형 프로그래밍의 기본형이다.
C, Python, Java, JavaScript, Fortran이 모두 여기에 속하며 현대 프로그래밍에서 가장 광범위하게 쓰이는 계열이다.
대부분의 개발자가 첫 언어로 접하는 원형이기도 하다.

### Lisp 계열

괄호와 전위 표기법(prefix notation)을 사용하며 강력한 매크로 시스템을 통해 언어 자체의 의미를 재정의할 수 있다.
코드가 리스트 자료구조와 동일한 형태로 존재한다는 점(코드-데이터 동형성)이 핵심이다.
Common Lisp, Scheme, Clojure가 대표적이다.

### ML 계열

일급 함수와 Hindley-Milner 타입 시스템을 갖춘 함수형 언어군이다.
모든 반복은 재귀 또는 고차 함수로 표현되며 타입 추론이 프로그래밍 경험의 중심이 된다.
Haskell, OCaml, Standard ML이 이 계열에 속한다.

### Self 계열

객체 간 메시지 전달로 프로그램을 구성하는 객체 지향 원형이다.
클래스 없이 기존 객체로부터 새 객체를 파생시키는 프로토타입 기반 방식이 특징이다.
Smalltalk와 JavaScript가 대표적 예시다.

### Forth 계열

스택 기반의 역폴란드 표기법(reverse Polish notation)을 사용한다.
프로그래머가 파서 자체를 가로채고 교체할 수 있어 도메인 특화 언어(DSL)를 쉽게 만들 수 있다.
PostScript, Factor가 이 계열에 해당한다.

### APL 계열

배열 중심 언어로 기호 연산자가 고수준 연산을 수행한다.
저자의 표현처럼 "기호의 시퀀스 자체가 연산의 이름"이 되는 독특한 문법을 가진다.
J, K가 대표적이다.

### Prolog 계열

사실(fact)과 규칙(rule) 기반의 논리 프로그래밍 원형이다.
프로그램은 기반 사실과 유도 규칙으로 구성되고 실행은 질의(query)의 해답을 탐색하는 과정이 된다.
흥미롭게도 SQL이 이 계열의 실용적 대표로 꼽힌다.

## 학습 로드맵

저자는 다음 순서로 학습을 권장한다.

| 단계 | 권장 학습 |
|------|----------|
| 1 | ALGOL 계열 언어 1개 숙달 |
| 2 | SQL로 관계형/논리 프로그래밍 체험 |
| 3 | 매년 다른 원형으로 확장 |

구체적 언어 선택보다는 서로 다른 파라다임을 경험하는 것이 지적 성장에 더 중요하다는 것이 저자의 핵심 메시지다.
Hacker News 토론에서는 Ruby와 Python이 진정한 ALGOL 계열인지, 특정 언어를 원형으로 분류하는 기준이 무엇인지에 대한 논쟁이 이어졌다.

## 결론

일곱 원형 언어 분류는 단순한 학술적 분류가 아니라 개발자의 사고 확장을 위한 실용적 지도다.
한 원형에 갇혀 있으면 문제를 바라보는 관점이 제한되며, 다른 원형을 경험할 때 새로운 해법이 보인다.
SQL을 배우는 것만으로도 Prolog의 세계를 엿볼 수 있다는 점은 특히 주목할 만하다.

## Reference

- [The Seven Programming Ur-Languages](https://madhadron.com/programming/seven_ur_languages.html/)
