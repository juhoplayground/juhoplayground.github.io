---
layout: post
title: Jenkins GitLab 연동하고 pipeline 만들기
author: 'Juho'
date: 2025-06-17 09:00:00 +0900
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
1. [Jenkins Gitlab Plugin 설치](#jenkins-gitlab-plugin-설치)
2. [GitLab Token 발급받기](#gitlab-token-발급받기)
3. [발급한 GitLab Token을 Jenkins에 등록하기](#발급한-gitlab-token을-jenkins에-등록하기)
4. [Gitlab 연동 테스트](#gitlab-연동-테스트)
5. [Pipeline에서 사용할 Credentials 생성하기](#pipeline에서-사용할-credentials-생성하기)
6. [Pipeline 만들기](#pipeline-만들기)
7. [GitLab Webhook 만들기](#gitlab-webhook-만들기)
8. [SCM 방식 사용하기](#scm-방식-사용하기)
  

## Jenkins Gitlab Plugin 설치
GitLab을 연동하기 위해서는 Plugin 설치가 필요하다.  
`http://xxx.xxx.xxx.xxx:port/manage/pluginManager/`로 접속하면 플러그인을 관리할 수 있다.  
`Available plugins`를 클릭한 다음 `GitLab`, `Generic Webhook Trigger` 플러그인을 설치했다.  
플러그인 설치 후 Jenkins를 재실행 해주어야 플러그인이 적용된다.  

## GitLab Token 발급받기  
`User Setting > Access tokens > Personal access tokens` 부분에서 Add new token 버튼을 통해 GitLab Token을 발급합니다.  
token scope의 경우 `api, read_user, read_repository, write_repository`정도만 먼저 부여하였습니다.  
필요시 다른 scope를 추가하면 됩니다.  

## 발급한 GitLab Token을 Jenkins에 등록하기  
`Dashboard > Jenkins 관리 > Credentials` 부분에서 Stores scoped to Jenkins에서 Domains에 있는 (global)을 클릭합니다.  
`Global credentials (unrestricted)` 페이지로 이동이 되는데 여기서 Add credentials 버튼을 클릭합니다.  
`Kind` 영역을 `GitLab API token`으로 변경한 다음 `Scope`는 따로 수정하지 않고 `API token`에 발급 받은 토큰을 입력해줍니다.  

## Gitlab 연동 테스트  
`Dashboard > Jenkins 관리 > System` 부분에서 아래로 드래그해서 GitLab 영역을 찾습니다.  
connection name과 Gitlab host URL을 입력한 다음 Credentials에서 방금 설정한 Credentials을 선택합니다.  
이후에 오른쪽 하단에 있는 Test Connection 버튼을 클릭합니다.  
왼쪽 하단에 Success라고 표기가 된다면 정상적으로 연동이 된 것 입니다.  

## Pipeline에서 사용할 Credentials 생성하기  
이번에는 `Kind`을 `Username with password`로 변경한 다음 Username에 GitLab ID 혹은 GitLab Email을 입력하고  
Password에는 발급 받은 GitLab Token을 입력해줍니다.  

## Pipeline 만들기  
`Dashboard > + 새로운 Item`을 클릭하고 Pileline을 생성합니다.  
간단하게 Clone 하는 예시를 생성해보겠습니다.  
```bash
pipeline {
    agent any
    
    stages {
        stage('Clone') {
            steps {
                git branch: '${branch}',
                    credentialsId: '${생성한 credentialId}',
                    url: '${repository 주소}.git'
                echo 'Hello World'
            }
        }
    }
}
```
그 다음 Save 버튼을 클릭하여 저장합니다.  
이 후 `지금 빌드` 버튼을 통해 해당 스크립트를 실행할 수 있습니다.  
상세 내용은 Console OutPut에서 확인할 수 있습니다.  
```bash
[Pipeline] echo
Hello World
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

## GitLab Webhook 만들기  
프로젝트에서 특정 이벤트가 발생했을 때 pipeline이 실행되도록 하려면 `Configure > Triggers`에서  
`Build when a change is pushed to GitLab. GitLab webhook URL: `을 체크하면 된다.  
먼저 Jenkins에서 생성한 해당 pipeline의 Triggers 설정 부분의 `Build when a change is pushed to GitLab. GitLab webhook URL:` 부분을 확인하고 활성화합니다.  
이후에 `고급`을 선택해서 Secret token을 생성해줍니다.  
그 다음 GitLab으로 이동한 다음 프로젝트의 Settings에서 Webhook을 부분에서 `Add nwe webhook` 버튼을 클릭합니다.  
URL에 `Build when a change is pushed to GitLab. GitLab webhook URL:` 뒷 부분에 있던 URL을 입력하고  
Secret token에는 방금 생성한 Secret token을 입력하고 `Add webhook` 버튼을 클릭하여 생성을 완료합니다.  


## SCM 방식 사용하기
이번에는 `Pipeline script from SCM`을 사용해보겠다.  
`Pipeline script`는 Jenkins UI 내에서 직접 pipeline 스크립트를 작성하는 방식인데  
`Pipeline script from SCM` 방식은 Git과 같은 SCM(Source Code Management)에서 pipeline 스크립트를 가져오는 방식이다.  
프로젝트 코드와 함깨 Jenkinsfile을 소스 코드 저장소에서 관리해서 버전 관리가 가능해지므로 개인적으로 더 나은 방식이라고 생각한다.  
  
프로젝트에 `Jenkinsfile` 파일을 만들고 아까 생성한 pipeline 스크립트를 복사 붙여넣는다.  
그리고 Jenkins UI에서 기존에는 Defintion을 `Pipeline script`로 하였는데 `Pipeline script from SCM`로 변경한다.
SCM을 `Git`으로 변경하고, Repositories에 `https://gitlab.com/user/project.git` 처럼 입력한다.  
Credentials은 이전에 만들어놨던 Credentials을 사용하면 된다.
Branches to build의 경우 기본으로 `*/master`로 되어 있다.
원하는 branch로 변경하면 된다.  
Script Path에 아까 만든 `Jenkinsfile` 파일의 경로를 넣어주면 된다.  

---  
