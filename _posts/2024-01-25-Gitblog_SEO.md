---
layout: post
title: Github blog 만들기 - 6 - GitHub Blog 검색엔진에 등록하기
author: 'Juho'
date: 2024-01-25 11:30:00 +0900
categories: [GitHub blog, GitHub, Git]
tags: [GitHub blog, GitHub, Git]
pin: True
toc : True
---
<br/>




## 목차
0. [검색엔진 등록 준비](#검색엔진-등록-준비)
  - 1) [_config.yml의 description 수정하기](#1-_configyml의-description-수정하기)
  - 2) [sitemap.xml 생성하기](#2-sitemapxml-생성하기)
  - 3) [robots.txt 생성하기](#3-robotstxt-생성하기)
1. [구글 등록](#구글-등록)
2. [네이버 등록](#네이버-등록)
3. [다음 등록](#다음-등록)
4. [Google Analytics 연동](#google-analytics4-연동)

## 검색엔진 등록 준비
#### 1) _config.yml의 description 수정하기
`description`은 사이트의 메타데이터 중 하나로 일반적으로 사이트의 간단한 설명이나 소개를 나타냅니다.<br/>
`description`은 웹 검색 결과나 소셜 미디어에서 링크를 공유할 때, 페이지 미리보기에서 표시될 수 있습니다. <br/>
검색 엔진은 웹 페이지의 메타 데이터를 사용하여 페이지의 컨텐츠와 관련된 정보를 이해하고 검색 결과에 표시합니다.<br/>
따라서 잘 작성된 `description`은 방문자가 검색 결과에서 해당 페이지를 클릭하도록 유도할 수 있습니다. <br/>
일반적인 SEO 가이드라인에 따르면, `description`은 페이지의 내용을 간결하고 명확하게 설명하는데 사용되어야 합니다.
물론, SEO에는 다양한 요소가 영향을 미치기 때문에 `description`만으로 SEO를 최적화하는 것은 아닙니다. 

#### 2) sitemap.xml 생성하기
`sitemap.xml`은 웹사이트의 검색 엔진에게 사이트의 페이지 구조를 알려주는 XML 형식의 파일입니다.<br/>
이 파일은 검색 엔진이 웹사이트를 크롤링할 때 어떤 페이지들이 있는지, 각 페이지의 중요도는 어떤지 등을 알려주는데 사용됩니다.<br/>
특히 SEO를 개선하고 검색 결과에서 더 잘 나타나도록 하는 데 도움이 됩니다.<br/>

게시물을 업로드할 때마다 매번 `sitemap.xml`을 업데이트해야하는 것은 번거로운 일이기 때문에, 자동화해주는 스크립트를 통해서 `sitemap.xml`을 생성할 것 입니다.<br/>

[sitemap.xml](https://github.com/juhoplayground/juhoplayground.github.io/blob/main/sitemap.xml)의 코드를 복사해서 `sitemap.xml` 파일을 생성하고 복사붙여넣기 하여 루트 디렉토리에 추가하면 된다.<br/>

Git에 푸시한 후 `https://{GitHub Id}.github.io/sitemap.xml`에 접속하면 GitHub Blog의 모든 글과 주소를 출력하는지 확인해보면 된다.<br/>

#### 3) robots.txt 생성하기
`robots.txt` 파일은 웹사이트의 루트 디렉토리에 위치하는 텍스트 파일로, 검색 엔진 크롤러에게 웹 페이지를 어떻게 크롤링하거나 색인화해야 하는지에 대한 지침을 제공합니다.<br/>
이 파일은 웹사이트의 특정 부분이나 파일이 검색 결과에서 제외되도록 하는 데 사용됩니다.<br/>

[robots.txt](https://github.com/juhoplayground/juhoplayground.github.io/blob/main/robots.txt)의 코드를 복사해서 `robots.txt` 파일을 생성하고 복사붙여넣기 하여 루트 디렉토리에 추가하면 된다.<br/>

#### 4) feed.xml 생성하기
`feed.xml` 파일은 주로 웹사이트의 최신 컨텐츠를 정형화된 형식으로 제공하는 역할을 합니다.<br/>
일반적으로 Atom 또는 RSS 피드 형식을 따르며, 웹사이트의 최신 글이나 뉴스를 구독자에게 효과적으로 전달하기 위해 사용됩니다.<br/>

검색 엔진은 피드를 통해 웹사이트의 구조와 최신 업데이트를 이해할 수 있습니다.<br/>
피드를 제공함으로써 검색 엔진이 사이트를 더 효과적으로 색인화하고 검색 결과에 포함시킬 수 있습니다.<br/>
SEO의 향상을 위해서 `feed.xml`도 생성합니다.

[feed.xml](https://github.com/juhoplayground/juhoplayground.github.io/blob/main/feed.xml)의 코드를 복사해서 `feed.xml` 파일을 생성하고 복사붙여넣기 하여 루트 디렉토리에 추가하면 된다.<br/>

## 구글 등록
[Google Search Console](https://search.google.com/search-console/welcome?utm_source=about-page)에 접속합니다.<br/>
좌측은 구매한 도메인이 있는 경우, 오른쪽은 구매한 도메인이 없는 경우입니다.<br/>
저는 구매한 도메인이 없기 때문에 오른쪽을 선택하도록 하겠습니다.<br/>
URL에 블로그 전체 주소를 작성해줍니다.

해당 URL의 소유권을 확인합니다. <br/>
아래의 화면에서 보이는 것 처럼 `google~~~.html` 파일을 다운로드 받은 뒤, 루트 디렉토리에 추가하도록 합니다.<br/>

Git에 푸시하고 Build & Deploy가 정상적으로 되었다면 `확인` 버튼을 클릭하여 계속 진행합니다 <br/>
소유권이 확인되면 아래의 결과를 확인할 수 있습니다.<br/>


그리고 제출된 화면을 볼 수 있습니다.<br/>

만약 며칠이 지나도 구글에서 검색이 되지 않는다면 왼쪽 탭에서 `URL 검사`를 클릭해서 `색인 생성 요청`을 클릭해보면 된다.<br/>

색인 생성이 되었다면 아래와 같이 화면을 볼 수 있다.<br/>

## 네이버 등록
[네이버 서치 어드바이저](https://searchadvisor.naver.com/)에 접속합니다.<br/>
1) 오른쪽 상단의 `웹 마스터 도구`를 클릭합니다.<br/>
2) `이곳에 URL을 입력해주세요.~~` 이 부분에 GitHub Blog 주소를 입력한 다음 오른쪽의 버튼을 클릭합니다.<br/>
3) HTML 파일을 다운로드 하고 루트 디렉토리에 추가한 다음 Git에 푸시합니다.<br/>
4) 아래의 `소유 확인` 버튼을 클릭합니다.<br/>
5) 자동 등록 방지 문자를 입력합니다.<br/>
6) 사이트를 클릭하여 사이트 관리 페이지로 이동합니다.<br/>
7) 왼쪽 탭의 `요청`을 클릭한 다음 `RSS 제출`을 클릭합니다.<br/>
8) 위에서 생성한 `feed.xml` URL을 입력해주고 자동 등록 방지 문자를 입력합니다.<br/>
9) 그 다음은 `사이트 맵` 제출을 클릭합니다.<br/>
10) 마찬가지로 위에서 생성한 `sitemap.xml`의 URL을 입력해주고 자동 동륵 방지 문자를 입력합니다.<br/>
11) 그 다음은 `검증`을 클릭한 다음 `robots.txt`를 클릭합니다.<br/>
12) `수집 요청` 버튼을 클릭하고 `robots.txt`가 제대로 호출된 것을 확인합니다.<br/>
13) `robots.txt 검증` 부분에 본인의 GitHub Blog URL을 작성하고 `확인` 버튼을 클릭합니다.<br/>
14) 수집이 가능하다는 팝업이 발생하는데, 발생하지 않는다면 `robots.txt`를 제대로 생성하고 다시 시도합니다.<br/>
15) 그 다음은 `검증`을 클릭한 다음 `URL 검사`를 클릭합니다.<br/>
16) 자신의 GitHub Blog URL을 입력하고 `확인` 버튼을 클릭합니다.<br/>
17) 그럼 `미수집 문서입니다. 페이지가 검색될 수 있게 수집 요청해보세요.` 라고 안내가 발생한다.<br/>
18) `수집 요청` 버튼을 클릭한다.<br/>
19) `요청`탭의 `웹 페이지 수집`으로 이동하게 되는데 `수집 요청 URL`에 본인의 GitHub Blog URL을 입력하고 `확인` 버튼을 클릭한다.<br/>
20) 그 다음 상단의 `간단 체크`를 클릭하고 본인의 GitHub Blog URL을 입력하고 자동 등록 방지 문자를 입력합니다.<br/>
21) 본인의 블로그에 어떤 설정이 잘못되었는지 체크하고 부족한 부분을 수정한다.<br/>

## 다음 등록
[다음 검색등록](https://register.search.daum.net/index.daum)에 접속합니다.<br/>
1) `URL` 부분에 본인의 GitHub Blog URL을 입력하고 `확인` 버튼을 클릭합니다.<br/>
2) 신규 등록 관련 서비스 약관 동의를 작성합니다.
3) 공통정보 내용을 `_config.yml` 기반으로 작성합니다.<br/>

## Google Analytics4 연동
[Google Aanlytics](https://analytics.google.com/analytics/web/#/)에 접속합니다.<br/>
대부분의 블로그가 UA기준으로 작성되었는데, [UA는 2023년 7월 1일부터 더 이상 신규 데이터를 처리하지 않고, 2024년 7월 1일부터 UA 인터페이스 및 API에 엑세스 할 수 없다.](https://support.google.com/analytics/answer/11583528?hl=ko)<br/>

아래의 단계를 따라서 진행하면 GA4연동을 진행할 수 있다.<br/>

1) 계정 설정 단계에서는 계정 이름만 작성하면 됩니다.<br/>
    계정 이름은 본인이 원하는 아무거나 입력하면 됩니다.<br/>
    체크를 하시거나, 해제하는 것 없이 다음 버튼 클릭하면 됩니다.<br/>

2) 속성 만들기에서 `속성 이름`은 본인의 GitHub Blog 주소를 `https://` 포함해서 작성합니다.<br/>
    `보고 시간대`및 `통화`는 `대한민국`으로 설정하시면 됩니다.<br/>
    그리고 그 아래의 `고급 옵션 보기`가 있는데 클릭합니다.<br/>
    이것을 설정을 해야Blog와 Google Analytics 연결이 가능합니다.<br/>

3) 그 이후 비즈니스 관련은 대충 입력하고 다음으로 넘어가면 됩니다.<br/>

4) 이후 서비스 약관 계약이 나오는데, 동의하고 다음으로 진행하면 됩니다.<br/>

5) 아래와 같이 측정ID를 `G-~~~`형식으로 알려줍니다.<br/>

6) 그 ID를 복사하여 _config.yml의 `google_analytics`에 붙여넣으면 된다.<br/>

7) 그리고 GitHub에 반영하면 아래와 같이 `데이터 수집이 활성화되었습니다.`라고 안내해준다.<br/>

8) 연동 후 24시간 정도가 지나면 아래와 같이 데이터가 잘 수집되는 것을 확인할 수 있다.<br/>


---
## 마무리
이제까지 GitHub Blog 관련해서 설정할 수 있는 것들을 해보았습니다.<br/>
나중에 `Google Adsense`를 등록하게 된다면 GitHub Blog 관련해서 추가 글을 작성해보록 하겠습니다.<br/>
긴 글 읽어주셔서 감사합니다. <br/>