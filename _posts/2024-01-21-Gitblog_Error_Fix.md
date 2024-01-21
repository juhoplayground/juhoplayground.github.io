---
layout: post
title: Github blog 만들기 - 3 - GitHub Blog 오류 수정
author: 'Juho'
date: 2024-01-21 21:00:00 +0900
categories: [GitHub blog, GitHub, Git]
tags: [GitHub blog, GitHub, Git]
pin: True
toc : True
---
<br/>

## 목차
1. [오류 사항1](#오류-사항1)
2. [오류 사항2](#오류-사항2)
3. [오류 사항3](#오류-사항3)


<br/>


## 오류 사항1
Git Push를 하고 [GitHub Id].github.io로 접속해보면 Local에서 확인한 것 처럼 Jekyll 테마가 적용된 블로그가 보여야하는데<br>

![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/1874f972-dee5-4489-91ff-e7b3c1cd6306)

이렇게 오류가 발생한 페이지가 보인다. <br/>
1. GitHub > Settings > Pages에서 Build and deployment를 `Deploy from a branch`에서 `GitHub Actions`로 변경한다. <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/86cefa95-72fe-4516-a6df-5b4fd5fe1a84)

2. Configure를 클릭한다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/bad8aa7c-3128-4d4c-8eb8-5c3e3ff071d0)

3. workflows/jekyll.yml 파일이 생성되는데, 수정하지 않고 우측 상단의 `Commit changes...` 버튼을 클릭한다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/02d59334-8671-4095-8ef1-e087c55e0e3f)
그리고 우측하단의 `Commit changes` 버튼을 클릭한다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/8b906e5a-3cc9-4529-9669-9f31a218f637)

<br/>

## 오류 사항2
빌드가 제대로 되었는지 확인하기 위해서 Actions를 클릭해서 확인해보자.<br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/82594cfc-dd9f-475d-82ba-a7cc8eaccbb2)
그럼 아까 수정한 내용에 오류가 발생한 것을 확인할 수 있다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/d26bcc27-00e7-4e86-b72f-c94b62f712c3)
해당 오류를 클릭해서 확인해보면
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/78fd35fc-cabd-4385-90e9-4888df23c205)
빌드할 때 Setup Ruby 단계에서 오류가 발생한 것을 확인할 수 있다. <br/>
gemfile.lock의 문제로
cmd에
```
bundle lock -add-platform x86_64-linux
```
명령어를 사용하여 해결하면 된다. <br/>
명령어를 실행하면 Gemfile.lcok 파일이 수정되는데 이것을 Git에 push를 한다.
<br/>

## 오류 사항3
위의 내용을 반영하고 다시 Actions를 확인해보면, 오류가 또 발생한 것을 확인할 수 있다.

![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/d068e503-fee3-46a0-97b0-83d5ad450b9d)

`assets/js/dist/xxx.min.js` 파일이 존재하지 않는다고 한다.

이 문제를 해결하기 위해서는 [Node.js](https://nodejs.org/en/download) 를 설치해야한다. <br/>
Node.js를 설치하면 npm(node package manager)도 같이 설치가 된다.

1. node 버전 확인
```
node -v
```
2. npm 버전 확인
```
npm -v
```

![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/370c56c6-f262-4b44-98b6-ba9e88f79eaa)

그 다음
```
npm install && npm run build
```
명령어를 실행해준다. <br/>
그러면 `'NODE_EN'은(는) 내부 또는 외부 명령,실행할 수 있는 프로그램, 또는 배치 파일이 아닙니다`라는 오류가 발생한다.


그럼
```
npm install -g win-node-env
```
명령어를 실행하고 <br/>
다시 위의 
```
npm install && npm run build
```
명령어를 실행한다.

![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/01354a62-efee-4230-8c99-dce8842aafe8)
그럼 `assets/js/dist/xxx.min.js` 파일들이 생성된 것을 확인할 수 있다. <br/>
이제 이 내용을 Git에 푸시를 해야하는데 .gitignore에 
`assets/js/dist`가 추가가 되어 있다. <br/>
그 부분을 주석처리한다.
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/f4ee7116-bbec-4341-bbe1-ef89be0956d0) <br/>
그리고 Git에 푸시하고 Actions을 확인해보면, 이전과 달리 정상적으로 빌드된 것을 확인할 수 있다.<br/>
다시 [GitHub Id].github.io로 접속해보면 <br/>
![image](https://github.com/juhoplayground/juhoplayground.github.io/assets/156918118/e0d95506-9cc4-4dd8-b368-756bf1299f55)
드디어 정상적으로 블로그를 확인할 수 있다.

---
다음에는 블로그 커스터마이징을 해보도록하겠다.