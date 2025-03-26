---
layout: post
title: GraphQL vs RESTful
author: 'Juho'
date: 2025-03-26 09:00:00 +0900
categories: [GraphQL]
tags: [GraphQL, FastAPI, strawberry, Python]
pin: True
toc : True
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
1. [GraphQL이란?](#graphql이란)
2. [GraphQL vs REST 비교](#graphql-vs-rest-비교)
3. [GraphQL이 더 적합한 경우](#graphql이-더-적합한-경우)
4. [RESTful API가 더 적합한 경우](#restful-api가-더-적합한-경우)
5. [간단하게 정리한 결론](#간단하게-정리한-결론)

## GraphQL이란?
[GraphQL](https://graphql.org/){:target="_blank"}은 페이스북의 내부 개발 과정에서 시작되어 점차 오픈소스 커뮤니티로 확산된 데이터 질의 (Query + Schema) 언어<br/>
클라이언트는 GraphQL 서버로 쿼리를 전송하고 서버는 해당 쿼리를 해석하고 데이터를 반환<br/>


## GraphQL vs REST 비교
<table>
  <thead>
    <tr>
      <th>항목</th>
      <th>GraphQL</th>
      <th>RESTful API</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>요청 단위</strong></td>
      <td>원하는 데이터 구조를 직접 명시</td>
      <td>리소스 단위 (ex. /users/1)</td>
    </tr>
    <tr>
      <td><strong>과/부족 요청</strong></td>
      <td>필요한 데이터만 요청 → 오버/언더페칭 문제 거의 없음</td>
      <td>오버/언더페칭 문제 발생 가능</td>
    </tr>
    <tr>
      <td><strong>엔드포인트 수</strong></td>
      <td>단일 엔드포인트 (/graphql) 사용</td>
      <td>리소스마다 엔드포인트 필요 (/users, /posts 등)</td>
    </tr>
    <tr>
      <td><strong>버전 관리</strong></td>
      <td>스키마 기반 → 필드 추가로도 하위 호환 유지 가능, 버전 관리 필요 없음</td>
      <td>URL 버전 방식 필요 (ex. /v1/users)</td>
    </tr>
    <tr>
      <td><strong>응답 최적화</strong></td>
      <td>클라이언트가 필요한 데이터만 요청 가능</td>
      <td>클라이언트가 응답 구조 조정 어려움</td>
    </tr>
    <tr>
      <td><strong>타입 시스템</strong></td>
      <td>강력한 스키마(Type System) 기반, 자동 문서화 지원</td>
      <td>명확한 타입 시스템 없음</td>
    </tr>
    <tr>
      <td><strong>프론트 개발 유연성</strong></td>
      <td>프론트가 필요한 데이터 정의 가능 → 유연성 높음</td>
      <td>프론트가 백엔드 변경에 의존적</td>
    </tr>
    <tr>
      <td><strong>개발자 도구</strong></td>
      <td>GraphiQL, Apollo Studio 등 도구 지원</td>
      <td>Swagger 등 도구 지원</td>
    </tr>
    <tr>
      <td><strong>캐싱 처리</strong></td>
      <td>POST 요청 및 유동적인 쿼리로 캐싱 복잡</td>
      <td>HTTP 캐시 및 CDN 활용 쉬움</td>
    </tr>
    <tr>
      <td><strong>학습 난이도</strong></td>
      <td>쿼리 언어, 스키마 설계 등 학습 곡선 있음</td>
      <td>직관적이며 학습이 쉬움</td>
    </tr>
    <tr>
      <td><strong>권한 관리</strong></td>
      <td>필드 단위 요청 → 권한 제어 복잡</td>
      <td>리소스 단위로 비교적 단순한 권한 처리 가능</td>
    </tr>
    <tr>
      <td><strong>특수 작업 지원</strong></td>
      <td>파일 업로드 등은 별도 구현 필요 (표준화 미흡)</td>
      <td>파일 업로드 등 일반적인 HTTP 기능 지원 용이</td>
    </tr>
    <tr>
      <td><strong>성능 문제</strong></td>
      <td>중첩 쿼리 및 N+1 문제 등 성능 최적화 필요</td>
      <td>예측 가능한 쿼리로 서버 성능 관리 쉬움</td>
    </tr>
    <tr>
      <td><strong>레거시 호환성</strong></td>
      <td>복잡한 환경에서는 통합 부담 있음</td>
      <td>오래된 시스템과의 연동 및 호환 쉬움</td>
    </tr>
  </tbody>
</table>


## GraphQL이 더 적합한 경우
<table>
  <thead>
    <tr>
      <th>조건</th>
      <th>이유</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>프론트엔드에서 다양한 분석 데이터를 자유롭게 선택/조합해서 조회해야 할 때</td>
      <td>오버페칭/언더페칭 문제 없이 필요한 데이터만 조회 가능</td>
    </tr>
    <tr>
      <td>복잡한 대시보드에서 여러 리소스를 한 요청으로 가져오고 싶은 경우</td>
      <td>GraphQL의 중첩 쿼리 및 관계 탐색 기능 유리</td>
    </tr>
    <tr>
      <td>리포트마다 필드 구성이 유동적으로 바뀌는 경우</td>
      <td>REST보다 유연한 요청 구조 가능</td>
    </tr>
    <tr>
      <td>프론트/백 간 협업에서 프론트 주도 개발이 필요한 경우</td>
      <td>프론트에서 직접 쿼리 정의 가능 → 백엔드 부담 감소</td>
    </tr>
  </tbody>
</table>


## RESTful API가 더 적합한 경우
<table>
  <thead>
    <tr>
      <th>조건</th>
      <th>이유</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>정해진 분석 API만 제공하고, 조회 구조가 고정적일 경우</td>
      <td>단순하게 구현 가능하며 캐싱/보안 관리 쉬움</td>
    </tr>
    <tr>
      <td>대량의 정적 리포트를 주기적으로 캐싱해두고 재사용할 때</td>
      <td>REST + HTTP 캐시, CDN으로 성능 극대화 가능</td>
    </tr>
    <tr>
      <td>시스템이 내부 전용이며 단순한 API 호출만 필요한 경우</td>
      <td>GraphQL의 복잡도가 오히려 불필요한 오버헤드</td>
    </tr>
    <tr>
      <td>보안과 접근 제어가 리소스 단위로 단순하게 구성될 수 있는 경우</td>
      <td>권한 관리가 쉬움</td>
    </tr>
  </tbody>
</table>


## 간단하게 정리한 결론
<table>
  <thead>
    <tr>
      <th>상황</th>
      <th>추천 방식</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>클라이언트가 유연하고 복합적인 데이터 요구를 할 수 있는 구조</td>
      <td>GraphQL</td>
    </tr>
    <tr>
      <td>API가 단순하고 정적이며 캐싱 중심 구조</td>
      <td>REST</td>
    </tr>
    <tr>
      <td>초기 구현 속도와 낮은 복잡성이 더 중요할 때</td>
      <td>REST</td>
    </tr>
    <tr>
      <td>프론트엔드 주도 대시보드, 필드 커스터마이징, 여러 리소스 합친 쿼리 필요</td>
      <td>GraphQL</td>
    </tr>
  </tbody>
</table>


<br/>

--- 

<br/>
다음 글에서는 FastAPI와 Strawberry를 이용해 GraphQL 기반 API를 구현하는 방법에 대해서 작성하도록 하겠습니다.<br/>
