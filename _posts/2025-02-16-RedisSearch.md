---
layout: post
title: RediSearch 설치하기
author: 'Juho'
date: 2025-02-16 09:00:00 +0900
categories: [Redis]
tags: [Redis, RediSearch, Python]
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
1. [RediSearch란?](#redisearch란)
2. [RediSearch 설치](#redisearch-설치)
 - 1) [Docker 설치](#1-docker-설치)
 - 2) [source로 build해서 설치](#2-source로-build해서-설치)

## RediSearch란?
Redis 데이터베이스의 기능을 확장하는 오픈 소스 모듈로 전체 텍스트 검색과 세컨더리 인덱싱 기능을 제공한다.<br/>
이를 통해 Redis에 저장된 데이터를 신속하게 검색하고, 복잡한 쿼리를 수행할 수 있다.<br/>
[RediSearch](https://github.com/RediSearch/RediSearch){:target="_blank"} 관련 내용은 여기서 확인할 수 있다.<br/>


## RediSearch 설치
#### 1) Docker 설치
[Redis Stack Docker image](https://hub.docker.com/r/redis/redis-stack-server/){:target="_blank"} 도커로 설치하면 매우 편리하다.<br/>
`docker run -p 6379:6379 redis/redis-stack-server:latest`로 실행하면 된다.<br/>

#### 2) source로 build해서 설치
1) 서브 모듈을 포함해서 저장소를 클론 <br/>
`git clone --recursive https://github.com/RediSearch/RediSearch.git`를 한다.<br/>
이미 클론한 경우에는 `git submodule update --init --recursive`를 하면 된다.<br/>
<br/>

2) 최신 릴리즈 태크를 체크아웃 (혹은 안정적인 릴리즈 태그)<br/>
`git checkout v2.10.12`를 한다.(작성일 기준 가장 최신 Releases 버전)<br/>
<br/>

3) 빌드하기<br/>
```
make setup
make
```
<br/>

4) redisearch.so 모듈 로드할 수 있게 설정하기<br/>
```
sudo mkdir -p /etc/redis/modules
sudo cp /home/{user}/RediSearch/bin/linux-x64-release/search/redisearch.so /etc/redis/modules/
sudo chown redis:redis /etc/redis/modules/redisearch.so
sudo chmod 755 /etc/redis/modules/redisearch.so
```
<br/>

5) redis.conf 수정하기 <br/>
`enable-module-command` yes 변경<br/>
`loadmodule /etc/redis/modules/redisearch.so` 추가<br/>
<br/>

6) redis 재실행 <br/>
`sudo systemctl restart redis-server.service` <br/>
<br/>

7) RediSearch 확인 <br/>
`redis-cli`를 통해서 redis로 접속한 다음 `module list` 명령어를 입력한다.<br/>
`"/etc/redis/modules/redisearch.so"` 내용이 보인다면 설치가 정상적으로 된 것이다.<br/>


---


<br/>
RediSearch를 docker가 아닌 source로 빌드해서 설치하는 방법을 알아보았다.<br/>
