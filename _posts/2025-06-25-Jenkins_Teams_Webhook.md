---
layout: post
title: Jenkins Pipeline 동작 후 Teams 알림 받기
author: 'Juho'
date: 2025-06-25 09:01:00 +0900
categories: [Jenkins]
tags: [Jenkins, CI, CD, CI/CD, GitLab]
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
1. [Temas Webhook 설정하기](#temas-webhook-설정하기)
2. [Office 365 Connector 플러그인 설치 ](#office-365-connector-플러그인-설치)
3. [Jenkins Pipeline 수정하기](#jenkins-pipeline-수정하기)

## Temas Webhook 설정하기
`채널 > 채널 관리 > 커넥터`에서 편집 버튼 클릭한다.  
이후 `incoming Webhook`을 검색하여 추가한 다음 connection의 이름을 입력하고 만들기 버튼 누르면 하단에 채널방 URL이 생성된다.  
해당 URL이 나중에 필요하니까 따로 메모해둔다.  

## Office 365 Connector 플러그인 설치  
`Jenkins 관리 > Plugins > Available plugins`에서 `Office 365 Connector / Power Automate workflows`을 설치한다.  

## Jenkins Pipeline 수정하기
특정 Pipelin Job을 선택하고 `Configuration > General > Office 365 Connector / Power Automate workflows`을 찾아 `Add Webhook`버튼을 클릭한다.  
URL 입력란에는 이전에 만들었던 채널방 URL을 입력하고 Name에는 원하는 값을 입력한다.  
고급 설정에서는 알람이 전송될 빌드 상태를 설정할 수 있다.  
```
[ ] Notify Build Start
[ ] Notify Aborted
[ ] Notify Failure
[ ] Notify Not Built
[ ] Notify Success
[ ] Notify Unstable
[ ] Notify Back To Normal
[ ] Notify Repeated Failure
```

원하는 상황에 맞춰서 상태를 체크하면 된다.  
  
    
---  

Slack과 같은 다른 메신저도 Webhook 방식을 이용해서 알림을 받아볼 수 있다.  
