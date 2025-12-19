---
layout: post
title: Jenkins 21 업그레이드 후 Gradle 빌드 오류 해결 (Unsupported class file major version 65)
author: 'Juho'
date: 2025-12-20 00:00:00 +0900
categories: [Jenkins]
tags: [Jenkins, Gradle, Java, CI/CD, 트러블슈팅]
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
1. [문제 상황](#문제-상황)
2. [오류 메시지 분석](#오류-메시지-분석)
3. [원인 파악](#원인-파악)
4. [해결 방법](#해결-방법)
5. [참고: Java와 Gradle 버전 호환성](#참고-java와-gradle-버전-호환성)

## 문제 상황

Jenkins를 17 버전에서 21 버전으로 업그레이드 한 후, 정상적으로 작동하던 Gradle 빌드가 갑자기 실패하기 시작했습니다.  

```bash
A problem occurred configuring root project '{prjoect name}'.
> Could not open cp_proj generic class cache for build file '/var/jenkins_home/workspace/{project name}/build.gradle' (/var/jenkins_home/.gradle/caches/8.0.2/scripts/7lak1b0qt6lt9k956q878p4nr).
   > BUG! exception in phase 'semantic analysis' in source unit '_BuildScript_' Unsupported class file major version 65
```

## 오류 메시지 분석

핵심 오류 메시지는 `Unsupported class file major version 65` 입니다.  

class file major version이란?
- Java 컴파일러가 생성하는 `.class` 파일의 버전을 나타내는 숫자입니다.  
- 각 Java 버전마다 고유한 major version 번호를 가집니다.  

| Java 버전 | Class File Major Version |
|-----------|-------------------------|
| Java 8    | 52                      |
| Java 11   | 55                      |
| Java 17   | 61                      |
| Java 21 | 65                  |

즉, major version 65는 Java 21로 컴파일된 클래스 파일을 의미합니다.  

## 원인 파악

Jenkins 21 LTS 버전은 Java 21을 기반으로 동작합니다.  

1. Jenkins가 Java 21로 실행되면서 Gradle wrapper나 빌드 스크립트를 Java 21로 컴파일합니다.  
2. 하지만 프로젝트에서 사용 중인 Gradle 버전이 Java 21을 지원하지 않으면 이 오류가 발생합니다.  

Gradle과 Java 호환성:  
- Gradle 7.3 ~ 7.6: Java 19까지 지원  
- Gradle 8.5 이상: Java 21 지원  

즉, 기존에 Gradle 8.0.2 버전을 사용하고 있었다면 Java 21과의 호환성 문제로 인해 빌드가 실패하게 됩니다.  

## 해결 방법

### 방법 1: Gradle 버전 업그레이드 (권장)

Jenkins 21과 함께 사용하려면 Gradle 8.5 이상으로 업그레이드해야 합니다.  

#### 1) Gradle Wrapper 사용 시

프로젝트 루트에서 `gradle/wrapper/gradle-wrapper.properties` 파일을 수정합니다:

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

그리고 wrapper를 재생성합니다:

```bash
./gradlew wrapper --gradle-version 8.5
```

#### 2) 로컬 Gradle 설치 시

```bash
# Gradle 8.5 이상 버전 다운로드 및 설치
wget https://services.gradle.org/distributions/gradle-8.5-bin.zip
unzip gradle-8.5-bin.zip
sudo mv gradle-8.5 /opt/gradle

# 환경 변수 설정
export GRADLE_HOME=/opt/gradle
export PATH=$GRADLE_HOME/bin:$PATH

# 버전 확인
gradle -v
```

#### 3) Jenkins Pipeline에서 Gradle 버전 지정

Jenkins Pipeline을 사용하는 경우, Jenkinsfile에서 Gradle 버전을 명시적으로 지정할 수 있습니다:

```groovy
pipeline {
    agent any

    tools {
        gradle 'gradle-8.5'  // Jenkins Global Tool Configuration에서 설정한 이름
    }

    stages {
        stage('Build') {
            steps {
                sh './gradlew clean build'
            }
        }
    }
}
```

### 방법 2: Jenkins에서 Java 17 사용 (임시 방편)

Gradle 버전을 바로 업그레이드하기 어려운 상황이라면, Jenkins를 Java 17 기반으로 실행할 수 있습니다:

```bash
# Jenkins Docker 컨테이너를 Java 17 버전으로 실행
docker pull jenkins/jenkins:lts-jdk17
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk17
```

하지만 Jenkins는 Java 17에 대한 지원은 2026년 3월 31일 이후에 종료될 예정입니다.  
그러므로 이 방법은 임시 방편일 뿐이며 장기적으로는 Gradle 8.5 이상으로 업그레이드하는 것이 권장됩니다.  

## 참고: Java와 Gradle 버전 호환성

Java 버전별 필요한 최소 Gradle 버전:

| Java 버전 | 최소 Gradle 버전 |
|-----------|-----------------|
| Java 8    | 2.0             |
| Java 11   | 5.0             |
| Java 17   | 7.3             |
| Java 21 | 8.5         |

자세한 호환성 정보는 [Gradle 공식 문서](https://docs.gradle.org/current/userguide/compatibility.html){:target="_blank"}에서 확인할 수 있습니다.  

## 마무리

Jenkins 업그레이드 시에는 Java 버전 변경에 따른 빌드 도구 호환성도 함께 고려해야 합니다.    

특히:
- Jenkins 17 → Jenkins 21: Java 17 → Java 21로 변경  
- Java 21 사용 시 Gradle 8.5 이상 필요  
- Maven, Ant 등 다른 빌드 도구도 유사한 호환성 체크 필요  

업그레이드 전에 버전 호환성을 미리 확인하면 이런 문제를 예방할 수 있습니다.   

---

## 참고 자료
- [Gradle Compatibility Matrix (docs.gradle.org)](https://docs.gradle.org/current/userguide/compatibility.html)  
- [Jenkins LTS Changelog (jenkins.io)](https://www.jenkins.io/changelog-stable/)  
