---
layout: post
title: Github blog 만들기 - 5 - GitHub Blog 댓글 설정하기
author: 'Juho'
date: 2024-01-24 19:00:00 +0900
categories: [GitHub blog, GitHub, Git]
tags: [GitHub blog, GitHub, Git]
pin: True
toc : True
---
<br/>

Chirpy는 기본적으로 Disqus를 지원합니다. <br/>
`_config.yml`에서 `disqus`로 검색하면 바로 확인할 수 있습니다. <br/>
그런데 UI가 내 취향이 아니여서 다른 것을 찾아보다가 <br/>
[Utterances](https://utteranc.es/)과 [Giscus](https://giscus.app/ko)를 알게 되었다.<br/>
그 중 `Giscus`의 UI가 조금 더 내 취향에 가깝고<br/>
커스텀 테마나 다양한 설정이 가능한 것이 마음에 들어 선택했다. <br/>


## 목차
0. [Giscus란?](#giscus란)
1. [Giscus 적용방법](#giscus-적용방법)
  - 1) [Giscus App 설치](#1-giscus-app-설치)
  - 2) [Discussions 활성화](#2-discussions-활성화)
  - 3) [Giscus 추가](#3-giscus-추가)


<br/>

## Giscus란?
Giscus는 GitHub Rpository에 댓글 기능을 추가하기 위한 오픈 소스 도구다. <br/>
Utterances에 많은 영감을 받았다고 한다. <br/>
Giscus을 사용하면 GitHub 프로젝트에 대한 댓글을 사용자들이 달 수 있게 되어 의사소통을 간편하게 할 수 있다. <br/>

Giscus는 [GitHub Discussions 검색 API](https://docs.github.com/en/graphql/guides/using-the-graphql-api-for-discussions#search)를 사용하여<br/>
선택된 매핑 방법에 따라 페이지와 연관된 Discussion을 찾는다. <br/>
일치하는 Discussion이 없으면 Giscus봇이 누군가 처음으로 댓글이나<br/>
반응을 남길 때 자동으로 Discussion을 생성한다. <br/>

댓글을 남기게 위해서 사용자는 GitHub Oauth를 이용하여 Giscus APP이 대신 게시할 수 있도록 권한을 부여한다. <br/>

## Giscus 적용방법
#### 1) Giscus App 설치
[Giscus APP](https://github.com/apps/giscus)을 댓글을 저장할 Repository에 설치하면 된다. <br/>
`Read more about this app on the Marketplace.`를 클릭한 다음 <br/>
녹색 `Set up a plan` 버튼을 클릭한 다음 <br/>
우측의 `Install it for free` 버튼을 클릭하여 설치한다. <br/>

#### 2) Discussions 활성화
`Settings > General > Features`에서 Discussions를 체크해서 활성화한다. <br/>

#### 3) Giscus를 GitHub Blog에 추가
> ① [Giscus](https://giscus.app/ko)에서 설정 부분으로 이동한다. <br/>
> ② `저장소` 섹션을 찾아서 GitHub Blog Repository를 입력합니다. <br/>
> - 위의 단계를 정상적으로 하면 아래와 같이 확인됩니다. <br/>
> - 만약 통과하지 못했다면, 위의 내용을 다시 확인해서 시도하면 됩니다. <br/>

> ③ GitHub Blog 페이지와 Discussions을 연동할 방법을 선택합니다. <br/>
> ④ Discussion 카테고리를 설정합니다. <br/>
> - Announcements 유형의 카테고리를 권장하므로 Announcements로 선택합니다. <br/>

> ⑤ 특정 기능 추가 <br/>
> ⑥ 테마 선택  <br/>
> ⑦ 생성된 Script 태그를 웹 페이지에 추가합니다. <br/>

---
다음에는 구글 검색 가능하게 해보도록 하겠습니다.