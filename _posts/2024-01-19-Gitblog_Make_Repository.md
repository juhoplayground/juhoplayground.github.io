---
layout: post
title: Github blog 만들기 - 1 - Repository 만들기
author: 'Juho'
date: 2024-01-19 16:12:00 +0900
categories: [GitHub blog, GitHub, Git]
tags: [GitHub blog, GitHub, Git]
pin: True
toc : True
---
<br/>
내가 생각하는 GitHub Blog의 장점은 커스터마이징이 자유롭다는것이고 <br/>
단점은 깃헙이 익숙하지 않거나, 마크다운 작성법이 익숙하지 않으면 불편하다는것이였다. <br/>
Git 사용은 익숙한데, 마크다운은 ReadMe.md 파일이나 Notion 작성할 때만 사용해서 고민이 되었는데<br/> 시간과 노력을 들이면 해결될 것이라고 생각해서 Velog가 아닌 GitHub Blog로 결정했다.

## 목차
1. [GitHub Blog란?](#github-blog란)
2. [Repository 생성](#repository-생성)
3. [Local로 git clone](#local로-git-clone)

<br/>


## GitHub Blog란?
Github Repository에 저장된 html 파일과 같은 정적 웹 문서들을<br/>
Github에서 무료로 웹에서 볼 수 있도록 호스팅 서비스를 제공해주는 것이다.<br/>
Github 사용자들은 Repository에 정적 파일 업로드를 업로드하여<br/>
한 계정당 1개의 고유한 정적 웹 사이트를 가질 수 있다.

<br/>
**GitHub 계정이 없는 경우 GitHub에 가입하고, 계정이 있다면 로그인한다.**
<br/>

## Repository 생성
1) [깃허브](https://github.com)에 들어가서 왼쪽 상단의 녹색의 `Create Repository` 버튼을 클릭하여 레파지토리를 생성한다.

2) 버튼을 클릭하면 Repository 생성 페이지로 이동된다.

3) Repository name을 본인의 Github ID인 [<span style="color:red">**Github ID**</span>].github.io로 설정해야한다.<br/>
Description은 생략해도 된다.<br/>
공개 범위는 초기 상태처럼 Public으로 유지한다.

4) 그리고 생성을 마무리하면 아래와 같이 Repository가 정상적으로 생성된 것을 확인할 수 있다.

5) 그 다음 웹 주소에 생성한 [Github ID].github.io을 입력하면 블로그가 생성된 것을 확인할 수 있다.<br/>
하지만 현재는 아무것도 작성한 것이 없기 때문에 404 Not Found 페이지를 확인할 수 있다.

## Local로 git clone
PC에서 작업할 수 있도록 git clone을 해야한다.<br/>
빨간색 상자 영역의 Repository의 주소를 복사한다.<br/>

Repository를 다운로드 받을 빈 폴더 하나를 생성한다.<br/>
지금은 폴더명을 `gitblog`로 정했다. <br/>

폴더 경로창을 클릭해서 전부 지운 뒤 cmd를 입력하면 명령 프롬프트 창이 실행된다.<br/>
그리고 git clone 명령어를 통해 Repository를 가져온다. <br/>

```
$ git clone [복사한 Repository 주소]
```

<br/>


---
다음에는 Jekyll 테마를 적용해보도록 하자.
