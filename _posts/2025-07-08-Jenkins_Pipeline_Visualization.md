---
layout: post
title: Jenkins Pipeline 시각화 - (BlueOcean, Stage View, Pipeline Graph View)
author: 'Juho'
date: 2025-07-08 09:00:00 +0900
categories: [Jenkins]
tags: [Jenkins, CI, CD, CI/CD]
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
1. [Blue Ocean](#blue-ocean)  
2. [Pipeline: Stage View](#pipeline-stage-view)
3. [Pipeline Graph View](#pipeline-graph-view)



## Blue Ocean
기존에 Jenkins에서는 groovy를 이용해서 pipeline을 만들어여했는데 이게 익숙하지 않으면 어려운 부분이 있다.  
[Jenkins Blue Ocean](https://www.jenkins.io/doc/book/blueocean/){:target="_blank"}은 그 단점을 보완하기 위한 플러그인이다.  
![Activity View](https://www.jenkins.io/doc/book/resources/blueocean/activity/overview.png)  
설치를 하게 되면 pipeline의 각 stage와 step을 시각화해서 볼 수 있고 빌드 상태를 확인하기가 용이해진다.  
각 stage를 재실행도 가능하다. 
![Detail View](https://www.jenkins.io/doc/book/resources/blueocean/pipeline-run-details/overview.png)  
또한 시각적인 가이드를 통해서 Jenkins 파일을 생성, 수정할수 있어서 groovy가 익숙하지 않은 사용자도 쉽게 pipeline 정의해서 사용할 수 있다.  
![Stage configuration](https://www.jenkins.io/doc/book/resources/blueocean/editor/stage-configuration.png)  
하지만 Blue Ocean은 기능 추가 없이 오류가 있을 경우에만 업데이트가 이루어지고 있는 단점이 있다.  
  
## Pipeline: Stage View
[Pipeline: Stage View](https://plugins.jenkins.io/pipeline-stage-view/)은 설치만 하면 다른 설정 없이 pipeline 시각화를 볼 수 있다.  
![Pipeline: Stage View](https://cdn.jsdelivr.net/gh/jenkinsci/pipeline-stage-view-plugin@master/docs/images/who-broke-it.png)    
테이블 상단 부분에서 보이는 Build, Deploy, Test, Promote는 사용자가 지정한 stage를 표시해준다.  
각 stage가 몇 초 소요했는지, 실패했는지 확인할 수 있다.  
그리고 각 stage 영역에 마우스 오버하면 해당 Stage의 logs를 확인할 수 있다.  


## Pipeline Graph View  
[Pipeline Graph View](https://plugins.jenkins.io/pipeline-graph-view/)를 설치하고
`Jenkins 관리 > Appearance`에서 `Pipeline Stages`와 `Pipeline Graph`를 체크해서 활성화하면 pipeline 시각화를 볼 수 있다.  
job 페이지가 아닌 실제 run한 기록에서 `Pipeline Overview`가면 Graph를 확인할 수 있다.  
![Pipeline Graph View](https://cdn.jsdelivr.net/gh/jenkinsci/pipeline-graph-view-plugin@main/docs/images/preview.png)  

---  
  
목적이나 상황에 맞게 선택하면 되는 취향 차이의 영역으로 개인적인 생각은 아래와 같다.  
- groovy 사용이 익숙하지 않아서 pipeline 편집 기능까지 필요하다면 : Blue Ocean  
- 간단한 실행 현황 모니터링만 필요하다 : Pipeline: Stage View
- 구조적·인터랙티브한 그래프 조회 : Pipeline Graph View




