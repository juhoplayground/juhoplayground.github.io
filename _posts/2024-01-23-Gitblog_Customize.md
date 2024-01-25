---
layout: post
title: Github blog 만들기 - 4 - GitHub Blog 커스터마이징
author: 'Juho'
date: 2024-01-23 09:00:00 +0900
categories: [GitHub blog, GitHub, Git]
tags: [GitHub blog, GitHub, Git]
pin: True
toc : True
---
<br/>


## 목차
0. [기본 설정 변경](#기본-설정-변경)
1. [Favicon 변경](#favicon-변경)
2. [프로필 사진 변경](#프로필-사진-변경)
3. [검색 결과 레이아웃 변경](#검색결과-레이아웃-변경)
4. [이전글, 다음글 설정 변경](#이전글-다음글-설정-변경)
5. [기타 설정](#기타-설정)


<br/>

## 기본 설정 변경
Chirpy를 적용하면 처음에 기본으로 적용된 설정이 있다.<br/>
Repository에 많은 폴더가 존재하는데 각 폴더와 yml 파일의 내용은 아래 표와 같다. <br/>

| 폴더/파일      | 설명                                                   |
|---------------|--------------------------------------------------------|
| _config.yml   | Jekyll 설정 파일로, 사이트의 기본 설정을 정의합니다.     |
| _layouts      | 레이아웃 파일을 포함하는 폴더로, 페이지의 구조를 정의합니다. |
| _includes     | 템플릿에서 재사용되는 부분을 포함하는 폴더입니다.         |
| _sass | scss 파일을 포함하는 폴더로, 스타일 시트를 정의합니다. |
| _posts        | 블로그 포스트가 담긴 폴더입니다.          |
| assets        | css, img, js 등과 같은 정적 자산을 포함한다. |
| tools        | GitHub에서 배포를 위한 코드가 들어 있다. 수정하지 않는것이 좋다. |

이제 나만의 블로그를 위해서 셋팅을 할 것 이다. <br/><br/>
1) `.travis.yml` 파일이 존재한다면 삭제한다. <br/>
2) `_posts` 폴더 하위의 파일들을 삭제한다.<br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; → 블로그 글의 예시가 있으므로 확인 후 삭제하는 것이 좋을 수 있다. <br/>
3) `docs` 폴더를 삭제한다. <br/>
4) `github>workflows>pages-deploy.yml.hook` 파일 제외하고 `.github` 폴더의 나머지 파일 삭제 <br/>
5) `github>workflows>pages-deploy.yml.hook`을 `github>workflows>pages-deploy.yml`로 변경  <br/>
6) _config.yml을 수정한다. <br/>

>  | 항목 | 값 | 설명 |
>  |----|----|----|
>  | lang | ko-KR | 언어를 한글로 설정 |
>  | timezone | Asia/Seoul | 서울 표준시로 설정 |
>  | title | 원하는 값 | 블로그 제목 |
>  | tagline | 원하는 값 | title 아래의 sub title |
>  | description | 원하는 값 | seo를 위한 키워드 입력 |
>  | url | https:{GitHub ID}.github.io | 블로그 URL |
>  | github | GitHub ID | 본인의 GitHub ID |
>  | twitter.username | twitter id | 트위터 ID |
>  | social.name | 이름 | 포스트에 표시할 이름 |
>  | social.email | 이메일 | 본인의 이메일 |
>  | social.links | SNS | 본인의 SNS의 홈 URL |
>  | google_site_verification | Google Search Console ID | Google Search Console |
>  | google_analytics | Google Analytics ID | Google Analytics |
>  | theme_mode | light or dark | 테마 스킨 |
>  | avatar | 이미지 경로 | 프로필 이미지 경로 |
>  | toc | true | 포스팅 된 글의 목차 표시 |
>  | disqus | https://disqus.com/ 에 가입 후 shortname | 댓글 기능 |
>  | paginate | 10 | 목록에 몇개의 글을 표시 |

<br/>
제가 설정한 [_config.yml](https://github.com/juhoplayground/juhoplayground.github.io/blob/main/_config.yml) 입니다. <br/>
수정하실 때 어렵다면 참고하시면 됩니다. <br/>
`_config.yml`을 수정하면 항상 jekyll을 재구동 해줘야합니다. <br/>

## Favicon 변경
파비콘이란 웹 브라우저 상단에 사이트 제목과 함깨 뜨는 작은 아이콘입니다.  <br/>
주소 표시줄에서 웹 사이트를 식별하는데 사용됩니다.  <br/>
파비콘은 주로 16X16 픽셀 크기의 이미지로 제작되며 ICO 형식으로 제공되거나 웹페이지의 HTML 코드에서 링크를 통해 지정할 수 있습니다. <br/>
[Real Favicon Generator](https://realfavicongenerator.net/)에 접속합니다. <br/>
파란색 `Select your Favicon image` 버튼을 클릭 후 파비콘으로 사용하려는 이미지를 업로드 합니다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/8a94a544-5366-4718-9364-ceb1a6f4da75)
하단의 `Generate your Favicons and HTML code` 버튼을 클릭해줍니다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/1899e59f-7df1-44b2-a727-619e75ba02e7)
`Favicon packag` 버튼을 클릭하면 `.zip` 파일이 다운로드 됩니다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/59265d22-3256-476b-bdec-c252493ed053)
해당 `.zip` 파일을 압축 해제한 다음 `browserconfig.xml`과 `site.webmanifest` 파일을 삭제해줍니다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/20a3ebf5-9834-4bf8-915e-39cfc0aea16b)
나머지 파일들은 `assets/img/favicons` 폴더에 넣어줍니다 <br/>
그 다음 `_includes/favicons.html` 파일을 열어서 `href` 부분을 아래와 같이 수정해줍니다 <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/049db1e1-5f39-4737-b774-6d4b95ed08e0)
그렇게 되면 Favicon이 적용된 것을 확인할 수 있습니다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/5f2b15d4-c71c-46bc-8818-7df112545c29)

## 프로필 사진 변경
좌측 사이드바의 프로필 사진을 변경하려면 `assets/img` 폴더 아래에 원하시는 이미지를 추가합니다. <br/>
저는 `profile.jpg` 이라는 이미지를 추가하였습니다. <br/>
그리고 `_inclues/sidebar.html`을 수정하면 됩니다. <br/>
`<img src="{​{- avatar_url -}​}" width="112" height="112" alt="avatar" onerror="this.style.display='none'">`를 `<img src="/assets/img/profile.jpg" width="112" height="112" alt="avatar" onerror="this.style.display='none'">`로 변경하면 됩니다. <br/>
혹은 src 부분에 원하는 이미지의 URL을 입력하면 됩니다.

## 검색결과 레이아웃 변경
Chirpy 테마에서 검색을 하면 결과물에 제목, 카테고리, 태그, 내용이 포함되고 <br/>
한 줄에 2개의 결과씩 출력이 되는 것을 확인할 수 있다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/492cc048-33d0-4573-922e-d5cdfbe0064a)
<br/>

우선 검색 결과물에 내용이 출력되지 않도록 하고 싶고<br/>
언제 작성했는지 작성일자를 추가하고 싶다.<br/>

1) 검색 결과물에 내용이 출력되지 않는 법 <br/>
`_includes/search-loader.html`파일에서 `<p>{snippet}</p>`이 부분을 주석처리한다. <br/>
 
2) 검색 결과물에 작성 날짜 출력하는 방법 <br/>
`assets/js/data/search.json` 파일에서 `"date": "{​{ post.date }​}"`을 <br/>
`"date": "{​{ post.date | date: "%Y-%m-%d"}​}",` 이렇게 변경한다. <br/>
그리고 `_includes/search-loader.html` 파일에서 주석으로 처리한 `<p>{snippet}</p>` 윗 라인에 <br/>
`{date}`를 추가해준다. <br/>


그 다음에는 한 줄에 1개의 검색 결과만 출력되게 하고싶다. <br/>
`_includes/search-results.html`파일의 `<div id="search-results" class="d-flex flex-wrap justify-content-center text-muted mt-3"></div>` 부분을 <br/>
`<div id="search-results" class="d-flex flex-column justify-content-center text-muted mt-3"></div>` 이렇게 변경한다. <br/>
그리고 `_saas/addon/commons.scss`파일에서  `#search-results > article `이 부분을 찾아서 주석처리해준다. <br/>


위의 내용을 모두 반영하면 아래와 같이 한 줄에 1개의 검색 결과만 출력되게 변경된다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/083551a1-a2e1-49ad-99e1-3e4bd81f9a38)


## 이전글, 다음글 설정 변경
다른 분들의 GitHub 블로그를 구경할때 현재 글의 주제와 상관없는 글로 다음글 설정이 되어있으면 <br/>
글을 읽을 때 불편하다고 생각해서 카테고리별로 이전글, 다음글이 설정되었으면 좋겠다고 생각했다. <br/>

`_includes/post-nav.html` 파일에서 이전, 다음글 설정을 할 수 있는데 <br/>
`<nav>` 태그 하위의 내용을 아래와 같이 변경한다.<br/>

```
  ​{​% assign cat = page.categories[0] ​%​}​
    ​{​% assign cat_list = site.categories[cat] ​%​}​
    ​{​% for post in cat_list ​%​}​
        ​{​% if post.url == page.url ​%​}​
            ​{​% assign prevIndex = forloop.index0 | minus : 1 ​%​}​
            ​{​% assign nextIndex = forloop.index0 | plus : 1 ​%​}​
            ​{​% if forloop.first == false ​%​}​
                ​{​% assign next_post = cat_list[prevIndex] ​%​}​
            ​{​% endif ​%​}​
            ​{​% if forloop.last == false ​%​}​
                ​{​% assign prev_post = cat_list[nextIndex] ​%​}​
            ​{​% endif ​%​}​
            ​{​% break ​%​}​
        ​{​% endif ​%​}​
    ​{​% endfor ​%​}​
    ​{​% if prev_post ​%​}​
        <span class="previous-post">Previous :
            <a href="{{ prev_post.url }}" class="pagination--pager">{{ site.data.ui-text[site.locale].pagination_previous}}{{ prev_post.title }}</a>
        </span>
    ​{​% else ​%​}​
        <span class="previous-post">
            <a href="#" class="pagination--pager disabled">{{ site.data.ui-text[site.locale].pagination_previous}}</a>
        </span>
    ​{​% endif ​%​}​

    ​{​% if next_post ​%​}​
        <span class="next-post">Next :
            <a href="{{ next_post.url }}" class="pagination--pager">{{ site.data.ui-text[site.locale].pagination_next}}{{ next_post.title }}</a>
        </span>
    ​{​% else ​%​}​
        <span class="next-post">
            <a href="#" class="pagination--pager disabled">{{ site.data.ui-text[site.locale].pagination_next}}</a>
        </span>
    ​{​% endif ​%​}​
```
카테고리별로 이전글, 다음글 설정이 되면서 카테고리의 첫 번째 글에서는 Previous가 표기되지 않고 
마지막글에서는 Next가 표기되지 않는다. <br/>

카테고리의 글 순서는 `_posts` 폴더에 있는 md 파일의 YAML 프론트매터를 사용하여 메타데이터를 지정한다. <br/>
메타데이터 중 `date` 필더를 기반으로 포스트가 정렬됩니다. <br/>

만약 날짜 이외의 다른 기준으로 포스트를 정렬하고 싶다면 `_config.yml` 파일에서<br/>
```
collections:
  tabs:
    output: true
    sort_by: order
```
이 부분을
```
collections:
  posts:
    output: true
    order:
      - field: 원하는 기준
        direction: asc or desc
```
이렇게 설정을 변경하면 됩니다.  <br/>

## 기타 설정
위의 내용을 다 따라하기 보다는 본인이 원하는 부분을 원하는 방식으로 변경하면 된다. <br/>
`localhost:4000`으로 블로그를 띄워놓고 본인이 수정하기 원하는 부분을 개발자 모드를 이용해서 찾으면 된다.<br/>
수정하고 싶은 영역에서 `마우스 오른쪽 버튼 클릭 > 검사`를 클릭해서 태그나 css 내용등을 찾은 뒤 <br/>
IDE로 돌아와서 검색 기능을 통해서 태그 정보나 css 정보를 입력해서 설정이 존재하는 파일을 찾는다 <br/>
그리고 본인이 원하는 내용으로 수정하면 됩니다. <br/>

---
다음에는 블로그에 댓글 기능을 추가하는 방법을 알아보도록 하겠다.