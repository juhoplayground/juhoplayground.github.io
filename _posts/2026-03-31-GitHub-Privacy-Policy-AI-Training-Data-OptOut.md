---
layout: post
title: "GitHub 개인정보 정책 변경, 4월 24일부터 기본값으로 AI 학습 데이터 활용"
author: 'Juho'
date: 2026-03-31 00:00:00 +0900
categories: [Dev]
tags: [AI, Security, Dev]
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
2. [주요 변경 사항](#주요-변경-사항)
3. [적용 범위](#적용-범위)
4. [비활성화 방법](#비활성화-방법)
5. [의미와 시사점](#의미와-시사점)
6. [결론](#결론)
7. [Reference](#reference)

## 개요

GitHub가 개인정보 보호 정책과 서비스 약관을 업데이트하여, 2026년 4월 24일부터 별도 설정을 하지 않으면 사용자 데이터가 AI 모델 학습에 기본적으로 활용된다.
이번 정책 변경은 GitHub Copilot Free, Pro, Pro+ 전 플랜에 적용된다.

## 주요 변경 사항

GitHub는 "AI 모델을 개발, 훈련 및 개선하기 위해 입력(프롬프트, 코드 컨텍스트)과 출력(제안)을 수집"할 수 있는 라이선스를 확보하게 된다.

핵심 변경 내용은 다음과 같다.

- 기본값(opt-out 방식)으로 사용자 데이터를 AI 학습에 활용한다
- GitHub Copilot의 모든 유료/무료 플랜에 적용된다
- 프롬프트, 코드 컨텍스트, AI 제안 등이 수집 대상이다
- Private Repository의 코드 자체는 보호되지만, AI 기능 사용 시 해당 입출력 데이터는 수집 가능하다

## 적용 범위

| 항목 | 수집 여부 |
|------|----------|
| Copilot에 입력한 프롬프트 | 수집 대상 |
| 코드 컨텍스트 (열린 파일 등) | 수집 대상 |
| Copilot이 생성한 제안 | 수집 대상 |
| Private Repository 코드 (AI 미사용) | 보호됨 |
| Private Repository + AI 기능 사용 시 입출력 | 수집 대상 |

핵심은 Private Repository라도 Copilot을 사용하면 해당 상호작용 데이터가 AI 학습에 활용될 수 있다는 점이다.

## 비활성화 방법

AI 학습 데이터 수집을 원하지 않는 경우, GitHub 설정에서 직접 비활성화해야 한다.

1. GitHub 설정 페이지로 이동한다
2. GitHub Copilot Features 섹션을 찾는다
3. "Allow GitHub to use my data for AI model training" 항목을 Disabled로 변경한다

이 설정을 변경하지 않으면 4월 24일부터 자동으로 데이터 수집이 활성화된다.

## 의미와 시사점

이번 정책 변경은 여러 논란을 불러일으키고 있다.

첫째, opt-out 방식이라는 점이 문제다.
사용자가 적극적으로 설정을 변경하지 않으면 자동으로 동의한 것으로 간주된다.
많은 사용자가 이 변경을 인지하지 못할 가능성이 높다.

둘째, 공지 방식에 대한 비판이 있다.
사용자들은 GitHub가 이 중요한 변경을 눈에 띄지 않는 방식으로 공지했다고 지적하고 있다.

셋째, 기업 사용자에게 미치는 영향이 크다.
Private Repository에서 Copilot을 활용하는 기업은 코드 관련 상호작용이 AI 학습에 사용될 수 있다는 점을 인지해야 한다.

## 결론

GitHub의 이번 정책 변경은 AI 시대에서 코드 데이터의 활용 범위가 확장되고 있음을 보여준다.
4월 24일 이전에 설정을 확인하고, 필요한 경우 AI 학습 데이터 수집을 비활성화하는 것이 권장된다.
특히 기업 환경에서 Copilot을 사용하는 팀은 이 정책이 자사의 데이터 보호 정책과 부합하는지 검토할 필요가 있다.

## Reference

- [Updates to our privacy statement and terms of service: how we use your data](https://github.blog/changelog/2026-03-25-updates-to-our-privacy-statement-and-terms-of-service-how-we-use-your-data/)
