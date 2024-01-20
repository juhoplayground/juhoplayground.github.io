---
layout: post
title: Github blog 만들기 - 2 - Jekyll 테마 적용하기
author: 'Juho'
date: 2024-01-20 16:30:00 +0900
categories: [GitHub blog, GitHub, Git]
tags: [GitHub blog, GitHub, Git]
pin: True
toc : True
---
<br/>

## 목차
1. [Jekyll이란?](#jekyll이란)
2. [Jekyll이란 테마 다운로드](#jekyll테마-다운로드)
3. [Jekyll Bundle](#jekyll-bundle)
4. [Ruby 다운로드](#ruby-다운로드)
5. [Jekyll Bundler 다운로드](#jekyll-bundler-다운로드)

<br/>


## Jekyll이란?
Jeykll은 정적 웹 사이트 생성기로, Jekyll 테마는 웹 사이트 다지인 및 레이아웃을 정의하는 템플릿과 스타일의 모음이다.<br/>
Jeykll 테마를 사용하여 웹 사이트를 쉽게 만들고 관리할 수 있도록 도와준다.<br/>
Jekyll 테마의 Repository를 clone 하고, 필요에 따라 조정하여 GitHub 블로그에 적용할 수 있다. <br/>

## Jekyll테마 다운로드
[http://jekyllthemes.org](http://jekyllthemes.org/)에 접속한다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/09f036c9-ca4f-4f4d-a889-59d6048ae4c0) <br/>

원하는 테마를 하나 선택해서 클릭한다.
지금 이 블로그에 적용된 테마는 [Chirpy](http://jekyllthemes.org/themes/jekyll-theme-chirpy/)다. <br/>

![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/2f25d2c3-0c74-4829-99a2-b5db0e5e3c37)
Download 버튼을 클릭한다. <br/>
Download 받은 zip 파일을 압축 해제하고, 모든 파일을 local에 git clone 받은 폴더로 복사 붙여넣기 한다. <br/>

## Jekyll Bundle
변경 사항을 commit하고 push해서 확인을 매번 하려면 번거롭기 때문에 Local에서 확인할 수 있도록 Bundle을 설치하려고 합니다.<br/>
Jeykll이 Ruby로 만들어서 Ruby를 설치하도록 하겠습니다.<br/>

## Ruby 다운로드
[Ruby 공식 설치 사이트](https://rubyinstaller.org/downloads/)로 이동합니다. <br/>
WITH DEVKIT에서 굵게 표기된 버전으로 다운로드 받습니다. <br/>
굵게 표기된 버전의 경우 업데이트에 따라 달라질 수 있기 때문에 상황에 맞춰서 다운로드 받으면 됩니다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/fdd443be-f35b-488e-8354-83928ad11c06) <br/>

exe를 다운로드 받고 실행하여 Ruby를 설치합니다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/bc1a3468-73ec-4a1d-82a3-2141ba89df83)
Next를 클릭하여 설치를 완료하고 cmd(명령 프롬프트)를 실행하여
```
$ ruby -v
```
를 입력하여 설치가 정상적으로 되었는지 확인합니다.<br/>
설치가 정상적으로 되었다면 위의 명령어를 실행했을 때
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/a2de0bc3-367c-46b9-83e2-4e0a470c3a1d)
설치한 Ruby의 버전이 표시 됩니다.


## Jekyll Bundle 다운로드
이제 Jekyll 및 Bundle을 설치해보겠습니다.

먼저
```
gem install jekyll
```
명령어를 실행하여 Jeykll을 설치합니다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/b692fea2-b8fd-4783-a629-868a86998f15) <br/>

이어서
```
gem install bundler
```
명령어를 입력하여 Bundle을 설치합니다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/d8f02868-e7ae-4d58-b385-01fda6034389)

마지막으로 Ruby 웹 서버를 설치합니다.
```
gem install webrick
```
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/e1b1d330-bcc2-4927-95bb-ab3df52b087a)

그 다음
```
bundle install
```
을 실행하여 Gemfile에 명시된 종속성들을 설치합니다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/72503f9c-1a39-403a-a531-6b154a7d0c8c)

이재 Jeykll 서버를 동작시켜보겠습니다.
```
bundle exec jekyll serve
```
명령어를 실행해봅니다.
에러 없이 정상적으로 모두 실행되었다면 cmd의 하단에 아래와 같이 표기됩니다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/2cdcc8d1-122b-479f-93db-73ea91c532ed) <br/>

브라우저 주소창에 [http://127.0.0.1:4000/](http://127.0.0.1:4000/) 또는 [ http://localhost:4000/]( http://localhost:4000/)를 입력하여 접속해봅니다.

그럼 다음과 같이 Jeykll 테마가 적용된 화면을 볼 수 있습니다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/5ef8e628-ce83-4bac-830d-0055c92135d0)

이제 변경사항을 git에 push하면 됩니다.

