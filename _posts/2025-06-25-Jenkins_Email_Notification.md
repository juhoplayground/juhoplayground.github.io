---
layout: post
title: Jenkins Pipeline 동작 후 이메일 알림 받기
author: 'Juho'
date: 2025-06-25 09:00:00 +0900
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
1. [Email Extension 설치하기](#email-extension-설치하기)
2. [Email Notification 설정하기](#email-notification-설정하기)
3. [Jenkins Pipeline 수정하기](#jenkins-pipeline-수정하기)

## Email Extension 설치하기  
`Jenkins 관리 > Plugins > Available plugins`에서 `Email Extension`을 설치한다.  

## Email Notification 설정하기
`Jenkins 관리 > System`에서 아래로 내리면 `Extended E-mail Notification`이라는 항목이 보인다.  
`STMP Server`에는 Outlook을 사용하기 때문에 `outlook.office365.com`을 입력한다.  
`Use SMTP Authentication`에는 메일 발송에 필요한 계정 정보를 입력한다.  
그리고 `USE SSL`은 체크를 해제했고 `USE TLS`는 체크를 하였다.  
`SMTP Port`는 `587`로 입력하고 Reply-To Address에는 적절한 메일 주소를 입력한다.  
이후 아래에 `Test configuration by sending test e-mail`을 체크하면 설정 값이 정상적으로 동작하는지 확인할 수 있다.  
테스트 메일을 받을 메일 주소를 입력하고 `Test configuration`을 클릭한다.  
정상적으로 설정되었다면 `Email was sucessfully sent`라고 출력되는 것을 확인할 수 있다.  


## Jenkins Pipeline 수정하기
`Email Extension` 플러그인을 설치했기 때문에 `mail` 기능을 Jenkins에서 사용할 수 있게 되었다.  
```groovy
pipeline {
    agent any

    stages {
        stage('DoSmth') {
            steps {
                
            }
        }
    }

    post {
      success {
        // pipeline 성공 시 메일 발송 
        mail(
          to : "abc@def.com",
          subject : "[Success] Deployment Completed - ${env.JOB_NAME} #${env.BUILD_NUMBER}"
          body: """
          Job: ${env.JOB_NAME}
          Build Number: ${env.BUILD_NUMBER}
          Build URL: ${env.BUILD_URL}
          Timestamp: ${new Date()}
          """
        )
      }
      failure {
        // pipeline 실패 시 메일 발송
        script {
          def failureReason = "Unknown failure"
          def failedStage = "Unknown stage"

          if (env.STAGE_NAME) {
            failedStage = env.STAGE_NAME
            }
          
          try {
            def buildLog = currentBuild.rawBuild.getLog(50)
            if (buildLog.any { it.contains("test1") }) {
              failureReason = "Fail Reason 1"
            } elif (buildLog.any { it.contains("test2") }) {
              failureReason = "Fail Reason 2"
            }
          } catch (Exception e) {
            failureReason = "Could not determine failure reason"
          }

          mail (
            to : "abc@def.com",
            subject : "[FAILURE] Deployment Completed - ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            body: """
            Job: ${env.JOB_NAME}
            Build Number: ${env.BUILD_NUMBER}
            Build URL: ${env.BUILD_URL}
            Console Log: ${env.BUILD_URL}console
            Failed Stage: ${failedStage}
            Failure Reason: ${failureReason}
            Build Status: ${currentBuild.currentResult}
            Timestamp: ${new Date()}
            """
          )
        }
      }
    }
}
```
  
이러한 형태를 사용하면 pipeline 이후 가능하다.  
혹은 success, failure 위치에 always를 사용해서 성공, 실패와 상관없이 항상 메일을 발송할 수도 있다.  
   
---  
