---
layout: post
title: Spring Boot에서 JPA의 N+1 문제 
author: 'Juho'
date: 2025-09-14 09:00:00 +0900
categories: [SpringBoot]
tags: [SpringBoot, JPA]
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
1. [Spring Boot에서 JPA의 N+1 문제](#spring-boot에서-jpa의-n1-문제)  
2. [N+1 문제 해결 방법](#n1-문제-해결-방법)  
  - [Fetch Join](#fetch-join)  
  - [EntityGraph](#entitygraph)  
  - [Batch Size](#batch-size)  
3. [BatchSize 설정 방법]  
  - [BatchSize 어노테이션](#batchsize-어노테이션)  
  - [hibernate.default_batch_fetch_size](#hibernatedefault_batch_fetch_size)  
  - [우선순위](#우선순위)
4. [성능 측정 및 모니터링 방법]

## Spring Boot에서 JPA의 N+1 문제
Spring Boot에서 JPA의 N+1 문제는 대표적인 성능 이슈 중 하나다.  

```java
@Entity
public class Parent {
    @Id @GeneratedValue
    private Long id;
    
    @OneToMany(mappedBy = "parent", fetch = FetchType.LAZY)
    private List<Child> children = new ArrayList<>();
}

// N+1 문제가 발생하는 코드
List<Parent> parents = parentRepository.findAll();
for (Parent parent : parents) {
    System.out.println(parent.getChildren().size()); // 여기서 N번의 추가 쿼리 발생
}
```

```sql
SELECT * FROM parent; -- 1번
SELECT * FROM child WHERE parent_id = 1;
SELECT * FROM child WHERE parent_id = 2;
...
SELECT * FROM child WHERE parent_id = N;
```

## N+1 문제 해결 방법
보통은 아래의 방식으로해결할 수 있다.  

### Fetch Join
```java
@Query("SELECT p FROM Parent p JOIN FETCH p.children")
List<Parent> findAllWithChildren();
```
  
### EntityGraph
```java
@EntityGraph(attributePaths = {"children"})
@Query("SELECT p FROM Parent p")
List<Parent> findAllWithEntityGraph();
```

### Batch Size
```yml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100
```

- Batch Fetch 적용 시
```
SELECT * FROM parent;
SELECT * FROM child WHERE parent_id IN (1,2,3,...,100);
SELECT * FROM child WHERE parent_id IN (101,102,...,200);
```

정리해보면 다음과 같다  
  

| 방법 | 장점 | 단점 | 사용 시기 |  
|------|------|------|-----------|  
| Fetch Join | 한 번의 쿼리로 해결 | 페이징 불가, 카테시안 곱 | 데이터 양이 적을 때 |  
| EntityGraph | 선택적 fetch 가능 | 복잡한 관계에서 제약 | 중간 복잡도 |  
| Batch Size | 페이징과 호환 | 여러 쿼리 실행 | 대용량 데이터 |   


## Batch Size 설정 방법
### BatchSize 어노테이션
```java
@Entity
public class Parent {
    @Id @GeneratedValue
    private Long id;
    
    @BatchSize(size = 10)  // 특정 연관관계에만 적용
    @OneToMany(mappedBy = "parent")
    private List<Child> children;
    
    @BatchSize(size = 5)   // 다른 연관관계에는 다른 크기 적용 가능
    @OneToMany(mappedBy = "parent") 
    private List<Order> orders;
}
```
  
적용 범위: 클래스(엔티티), 컬렉션, 개별 연관관계 단위  
설정 방식: 어노테이션으로 직접 지정  
특징  
- 특정 연관관계/엔티티에만 선택적으로 batch size 적용 가능  
- 프로젝트 내에서 관계마다 다른 batch 크기를 줄 수 있음  
- 코드에 박히므로 상황에 따라 일괄 변경이 어려움  

### hibernate.default_batch_fetch_size
```yml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 100  # 모든 연관관계에 기본적으로 적용
```
적용 범위: 전역 설정 (모든 LAZY 연관관계, 컬렉션, ToOne 관계)  
설정 방식: application.yml 또는 application.properties  
특징    
- 기본적으로 모든 지연 로딩에 동일한 batch size 적용  
- 설정 한 줄이면 전체 적용 → 운영 시 빠르게 튜닝 가능  
- 세밀한 제어는 어렵고, 특정 관계에만 다르게 적용하려면 @BatchSize 병행 필요  

### 우선순위
@BatchSize 어노테이션이 hibernate.default_batch_fetch_size보다 우선순위가 높음  
특정 연관관계에 @BatchSize가 설정되어 있으면 전역 설정을 무시하고 어노테이션 값 사용  


## 성능 측정 및 모니터링 방법
```yml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.SQL.myUnit: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
    org.hibernate.engine.internal.StatisticalLoggingSessionEventListener: DEBUG

spring:
  jpa:
    show-sql: true
    properties:
      hibernate.format_sql: true
      hibernate.generate_statistics: true
```
- show-sql=true: 실행된 SQL 출력  
- BasicBinder=TRACE: 파라미터 값 출력  
- generate_statistics=true: 실행 횟수, 캐시 히트율, 엔티티 로딩 수 확인 가능  

---  
  
Batch 크기 너무 크면  또는 네트워크 부담될 수 있음 그래서 모니터링 하면저 트래픽, DB 상황에 맞춰 조정해야한다.  
일부 DB는 IN 절 파라미터나 패킷 크기 제한이 있음 → 너무 크게 잡지 않아야한다.  
