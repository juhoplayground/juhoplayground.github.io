---
layout: post
title: Jenkins Docker 업그레이드 하는 방법
author: 'Juho'
date: 2025-11-09 09:00:00 +0900
categories: [Jenkins]
tags: [Jenkins, CI, CD, CI/CD, Docker]
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
1. [권장하지 않는 방법](#권장하지-않는-방법)  
2. [권장하는 방법](#권장하는-방법)



## 권장하지 않는 방법  
```bash
# jenkins container에 root로 진입
docker exec -u 0 -it ${jenkins_container} /bin/bash  

# 업데이트할 버전의 jenkins.war 파일 다운로드
wget http://updates.jenkins-ci.org/download/war/{jenkins_version}/jenkins.war

## 소유권 변경
chown jenkins:jenkins /usr/share/jenkins/jenkins.war

## jenkins conatiner 재시작
docker restart ${jenkins_container}
```
이렇게 하면 안되는 이유는  
1. 도커 이미지 불변성(immutable) 원칙 위반  
- 이미지는 특정 시점의 환경으로 빌드된 것이고 jenkins.war 교체는 그 이미지를 수정된 상태로 만든다.  
- 컨테이너를 재생성하면 이 변경은 사라지게 된다.  
2. 빌드 환경 불일치
- 기존 이미지는 그 버전에 맞춰서 OS 플러그인, JAR 구조가 구성되어 있다.  
- 그런데 단순히 jenkins.war만 교체하면 내부 Java 라이브러리나 dependency mismatch가 생길 수 있다.  
3. 재현 불가
- 누가 어떤 war를 교체했는지 기록이 없고 이미지 태그도 깨지게 된다.  
- 나중에 같은 환경을 다른 서버에 옮기거나 백업 복구하려면 정확히 동일한 상태를 재현 불가능하게 된다. 
4. 보안 취약점 노출 가능성 
- 기존 이미지가 n개월 전 빌드일 수 있고 OS 레벨 보안 패치가 오래된 상태일 수 있다.  
- 단순히 jenkins.war만 교체한다면 OS 취약점은 그대로 남게 된다.  

## 권장하는 방법  
우선 JENKINS_HOME : host의 `/home/ec2-user/jenkins_home_backup` → 컨테이너 `/var/jenkins_home`로 bind mount로 되어 있다.    
```bash
## 백업 복원용
mkdir -p ~/jenkins_backup
tar -C ~/jenkins_backup -xzf ~/jenkins_home_YYYY-MM-DD.tgz
# 기본 jenkins UID는 1000(이미지에 따라 다름) -> 권한 맞추기
sudo chown -R 1000:1000 ~/jenkins_backup

## 기존 서버 중지 및 제거
docker stop jenkins_bind
docker rm jenkins_bind

## 새버전의 Jenkins 이미지 다운로드
docker pull jenkins/jenkins:lts-jdk21
docker run -d --name jenkins_bind \
 -p 80:8080 -p 50000:50000 \
 -v ~/jenkins_backup:/var/jenkins_home \
jenkins/jenkins:lts-jdk21

## 부팅 중 플러그인 마이그레이션 메시지 또는 오류 확인
docker logs -f jenkins_bind
```

---  
  





