---
layout: post
title: Spring Data JPA는 어떻게 쿼리를 자동으로 만들어낼까?
author: 'Juho'
date: 2025-10-19 09:00:00 +0900
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
0. [글을 쓰게 된 사소한 계기](#글을-쓰게-된-사소한-계기)
1. [Spring Data JPA는 어떻게 쿼리를 자동으로 만들어 실행할까?](#spring-data-jpa는-어떻게-쿼리를-자동으로-만들어-실행할까)  
2. [Spring Data JPA란?](#spring-data-jpa란)
3. [Repository 인터페이스가 자동으로 구현되는 이유](#repository-인터페이스가-자동으로-구현되는-이유)
4. [메소드 이름으로 쿼리를 만드는 규칙](#메소드-이름으로-쿼리를-만드는-규칙)
5. [직접 쿼리 지정도 가능 — @Query](#직접-쿼리-지정도-가능--query)  
6. [EntityManager와의 관계](#entitymanager와의-관계)
7. [동작 구조 요약](#동작-구조-요약)


## 글을 쓰게 된 사소한 계기  
지난 주 쯤이였나 신규 기능에 대한 PR을 올렸는데 팀원 중 한명이 내 자리로 와서  
'실제 쿼리 작성을 안했는데 어떻게 쿼리가 동작하는거예요?'라고 물어봐서 설명해주었다.  
설명을 다 듣고 '신기하네요'라고 하시고 자리로 돌아가는 팀원분을 보고 든 생각은  
1. 연차가 있으신데 이것도 모르시는건 문제가 있지 않나?  
2. 요즘 ChatGPT나 Claude 같은 코딩 잘하는 AI도 있는데 핑거 프린세스이신가?  
3. 그래도 모르는걸 직접 물어보러 오시다니 용기 있으시다.  
이렇게 순차적으로 생각이 들었던 것 같다.  

## Spring Data JPA는 어떻게 쿼리를 자동으로 만들어 실행할까?
Spring Boot로 백엔드를 개발하다 보면, 이런 코드를 자주 보게 된다.
```java
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByName(String name);
}
```

구현체가 없는데도 실제로 쿼리가 실행이 된다.  
어떻게 구현체가 없는데도 실제로 쿼리가 실행이 되는것인지 대략적으로 정리해보았다.  


## Spring Data JPA란?
Spring Data JPA는 JPA(EntityManager)를 더 편하게 사용할 수 있도록 추상화해주는 라이브러리다.  
원래 JPA를 직접 쓸 때는 EntityManager를 이렇게 사용해야한다.  
```java
TypedQuery<User> query = em.createQuery("SELECT u FROM User u WHERE u.name = :name", User.class);
query.setParameter("name", "홍길동");
List<User> result = query.getResultList();
```

하지만 Spring Data JPA를 사용하면 이렇게 바뀐다.  
```java
List<User> users = userRepository.findByName("홍길동");
```

그런데 내부적으로는 동일한 JPQL → SQL 쿼리가 만들어져 실행되게 된다.  

## Repository 인터페이스가 자동으로 구현되는 이유  
핵심은 프록시(Proxy) 기반의 동적 구현(Dynamic Implementation)입니다.  
Spring Boot가 실행되면 @EnableJpaRepositories 혹은 @SpringBootApplication 내부 설정 덕분에 UserRepository 같은 인터페이스를 스캔합니다.  
그 후 Spring Data JPA 내부의 다음 컴포넌트들이 동작한다.    
1. JpaRepositoryFactoryBean  
- UserRepository 인터페이스를 감지하고 Factory 생성  
2. JpaRepositoryFactory  
- 인터페이스의 타입 정보(User, Long)를 바탕으로  
- 프록시 객체를 생성 (동적으로 구현체를 만듦)  
3. SimpleJpaRepository  
- 기본 CRUD 메소드(save, findById, delete 등)를 지원하는 실제 클래스  
- 대부분의 JpaRepository 기능은 여기서 처리됨  
4. QueryLookupStrategy  
- findByName, countByEmail, existsById 같은 메서드명을 분석해 JPQL 쿼리를 생성하거나  
- 이미 선언된 @Query를 찾아 연결  

## 메소드 이름으로 쿼리를 만드는 규칙  
Spring Data JPA는 메서드 이름을 파싱해서 쿼리를 자동으로 생성한다.    
```java
List<User> findByName(String name);
List<User> findByAgeGreaterThan(int age);
List<User> findByEmailContaining(String keyword);
User findTopByOrderByIdDesc();
boolean existsByEmail(String email);
```

예를 들어 위와 같은 메소드가 있다고 한다면 각각 아래와 같이 변환이 된다.  
  
| 메서드 이름 | 생성되는 JPQL |  
|--------------|----------------|  
| `findByName` | `SELECT u FROM User u WHERE u.name = :name` |  
| `findByAgeGreaterThan` | `SELECT u FROM User u WHERE u.age > :age` |  
| `findByEmailContaining` | `SELECT u FROM User u WHERE u.email LIKE %:keyword%` |  
| `findTopByOrderByIdDesc` | `SELECT u FROM User u ORDER BY u.id DESC LIMIT 1` |  
| `existsByEmail` | `SELECT COUNT(u) > 0 FROM User u WHERE u.email = :email` |  

이 규칙은 QueryLookupStrategy가 내부적으로 처리한다.  

## 직접 쿼리 지정도 가능 — @Query  
자동 생성 규칙으로는 표현하기 어려운 경우도 있다.  
이런 경우에는 @Query를 사용하면 된다.  

```java
@Query("SELECT u FROM User u WHERE u.email LIKE %:email% AND u.age > :age")
List<User> searchUser(@Param("email") String email, @Param("age") int age);
```

또는 네이티브 SQL도 가능하다.  
```java
@Query(value = "SELECT * FROM users WHERE email LIKE %:email%", nativeQuery = true)
List<User> searchByEmailNative(@Param("email") String email);
```

## EntityManager와의 관계
Spring Data JPA는 내부적으로 EntityManager를 감싸고 있다.  
즉 호출하는 userRepository.save() 나 findByName() 같은 메서드는 다음과 같은 기능을 호출한 것 이다.  

| Repository 메서드 | 내부 EntityManager 동작      |  
| -------------- | ------------------------ |  
| `save()`       | `persist()` 또는 `merge()` |  
| `findById()`   | `find()`                 |  
| `delete()`     | `remove()`               |  
| `findAll()`    | JPQL `SELECT` 실행         |  

## 동작 구조 요약  
```
┌───────────────────────────┐
│     UserRepository        │ ← 우리가 만든 인터페이스
│   (extends JpaRepository) │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│  Proxy (동적 구현체)      │ ← 런타임에 생성됨
│  - findByName() 호출시    │
│  - Query 생성 및 실행     │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│   SimpleJpaRepository     │
│   → EntityManager 사용    │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│     EntityManager         │
│     → JPQL → SQL 변환     │
│     → DB Query 실행       │
└───────────────────────────┘
```

---  

Spring Data JPA는 JPA의 복잡한 반복 코드를 추상화하고 Repository 패턴을 통해 데이터 접근 로직을 깔끔하게 분리해준다.  
덕분에 개발자는 비즈니스 로직에만 집중할 수 있고 쿼리 작성 대신 도메인 모델 설계에 더 많은 시간을 쓸 수 있다.  
결국 중요한 것은 `자동 쿼리 생성` 그 자체가 아니라 그 내부의 동작 원리를 이해하고 상황에 맞게 활용하는 것이라고 생각한다.  
프레임워크를 깊이 이해할수록 우리는 더 단순하고 명확한 코드를 작성할 수 있다.  
  

다음 글에서는 이 자동 쿼리 방식의 한계를 보완해주는 QueryDSL 연동도 함께 다뤄볼 예정이다.  
  
참고 문서  
- https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html
