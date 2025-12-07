---
layout: post
title: Google Lighthouse - 웹페이지 품질 개선 도구
author: 'Juho'
date: 2025-12-07 00:00:00 +0900
categories: [Lighthouse]
tags: [Lighthouse, SEO, Performance, Web Optimization]
pin: False
toc: True
---

## 목차
1. [Lighthouse란 무엇인가?](#lighthouse란-무엇인가)
2. [Lighthouse의 핵심 평가 영역](#lighthouse의-핵심-평가-영역)
3. [설치 및 사용 방법](#설치-및-사용-방법)
4. [SEO 감사 기능](#seo-감사-기능)
5. [성능 최적화 방법](#성능-최적화-방법)

---

## Lighthouse란 무엇인가?

Google Lighthouse는 웹페이지 품질 개선을 위해 오픈소스로 제공되는 자동화 도구입니다.  
Chrome 브라우저의 개발자 도구에 내장되어 있으며 웹사이트의 성능, 접근성, SEO, 모범 사례 등을 종합적으로 평가하여 사이트 품질을 개선할 수 있는 방법을 알려줍니다.  

### 핵심 개념

Lighthouse는 웹페이지를 자동으로 분석하여 다음과 같은 측면에서 점수를 제공합니다:  
- 성능 (Performance): 페이지 로딩 속도 및 사용자 경험  
- 접근성 (Accessibility): 장애인을 포함한 모든 사용자의 접근 가능성  
- 모범 사례 (Best Practices): 웹 개발 표준 준수 여부  
- SEO: 검색 엔진 최적화 상태  
- PWA (Progressive Web App): 프로그레시브 웹 앱 기준 충족 여부  

### 개발 및 라이선스
- 개발사: Google  
- 라이선스: Apache 2.0 (오픈소스)  
- GitHub: [GoogleChrome/lighthouse](https://github.com/GoogleChrome/lighthouse)  

---

## Lighthouse의 핵심 평가 영역  

### 1. Performance (성능)  

웹사이트의 로딩 속도와 상호작용 성능을 측정  

#### 주요 측정 지표  
- FCP (First Contentful Paint): 첫 콘텐츠가 화면에 표시되는 시간  
- LCP (Largest Contentful Paint): 가장 큰 콘텐츠 요소가 로드되는 시간  
- TBT (Total Blocking Time): 메인 스레드가 차단된 총 시간  
- CLS (Cumulative Layout Shift): 레이아웃 이동 누적 점수  
- Speed Index: 페이지 콘텐츠가 시각적으로 표시되는 속도  

### 2. Accessibility (접근성)  

모든 사용자가 웹사이트를 사용할 수 있는지 평가  

#### 주요 검사 항목  
- 이미지 대체 텍스트 (alt 속성)  
- 색상 대비  
- ARIA 속성 적절성  
- 키보드 탐색 가능성  
- 스크린 리더 호환성  

### 3. Best Practices (모범 사례)  

웹 개발의 현대적 표준을 준수하는지 확인  

#### 검사 항목
- HTTPS 사용  
- 콘솔 오류 없음  
- 사용되지 않는 API 미사용  
- 보안 취약점 점검  
- 브라우저 호환성  

### 4. SEO (검색 엔진 최적화)  

검색 엔진이 페이지를 잘 이해하고 색인할 수 있는지 평가  

#### 주요 검사 항목  
- meta description 존재 여부  
- title 태그 적절성  
- 모바일 친화성  
- robots.txt 적절성  
- 크롤링 가능한 링크  
- 구조화된 데이터  

### 5. PWA (Progressive Web App)  

프로그레시브 웹 앱 기준을 충족하는지 평가  

#### 검사 항목  
- Service Worker 등록  
- HTTPS 사용  
- 앱 매니페스트 파일  
- 오프라인 작동 여부  
- 홈 화면 추가 가능성  

---

## 설치 및 사용 방법  

### 방법 1: Chrome DevTools (권장)  

가장 간단하고 일반적인 방법  

#### STEP 1: Chrome 개발자 도구 열기  

1. 분석하려는 웹페이지를 Chrome에서 엽니다  
2. 다음 중 하나의 방법으로 개발자 도구를 엽니다:  
   - `F12` 키 누르기
   - `Ctrl + Shift + I` (Windows/Linux)
   - `Cmd + Option + I` (Mac)
   - 마우스 우클릭 → "검사" 선택

#### STEP 2: Lighthouse 탭 선택  

개발자 도구 상단 메뉴에서 "Lighthouse" 탭을 클릭  

#### STEP 3: 평가 항목 선택  

분석하고자 하는 카테고리를 선택합니다:  
- Performance  
- Accessibility  
- Best Practices  
- SEO  
- Progressive Web App  

디바이스 모드 선택:  
- Mobile: 모바일 환경 시뮬레이션  
- Desktop: 데스크톱 환경  

#### STEP 4: 분석 실행  

"Analyze page load" 또는 "리포트 생성" 버튼을 클릭  

분석이 완료되면 각 항목별 점수(0-100)와 개선 제안 사항이 표시  

### 방법 2: Chrome 확장 프로그램  

독립적인 확장 프로그램으로 사용할 수 있습니다.  

#### 설치 방법

1. [Chrome 웹 스토어](https://chrome.google.com/webstore)에서 "Lighthouse" 검색  
2. "Chrome에 추가" 클릭  
3. 확장 프로그램 아이콘이 브라우저 툴바에 추가

#### 사용 방법  

1. 분석하려는 페이지에서 Lighthouse 아이콘 클릭  
2. "Generate report" 클릭  
3. 새 탭에서 상세 리포트 확인  

### 방법 3: Node CLI  

프로그래밍 방식으로 Lighthouse를 실행할 수 있습니다.  

#### 설치

```bash
# npm 사용
npm install -g lighthouse

# yarn 사용
yarn global add lighthouse
```

#### 기본 사용법

```bash
# 기본 분석
lighthouse https://example.com

# 특정 카테고리만 분석
lighthouse https://example.com --only-categories=performance,seo

# HTML 리포트 생성
lighthouse https://example.com --output=html --output-path=./report.html

# JSON 리포트 생성
lighthouse https://example.com --output=json --output-path=./report.json

# 모바일 모드로 실행
lighthouse https://example.com --preset=mobile

# 데스크톱 모드로 실행
lighthouse https://example.com --preset=desktop
```

#### 고급 옵션

```bash
# 헤드리스 모드 (화면 없이 실행)
lighthouse https://example.com --chrome-flags="--headless"

# 네트워크 쓰로틀링 조정
lighthouse https://example.com --throttling-method=devtools

# 여러 URL 분석 (bash 스크립트 사용)
for url in https://example.com https://example.org; do
  lighthouse "$url" --output=json --output-path="./report-$(echo $url | sed 's/https\?:\/\///g' | tr '/' '-').json"
done
```

### 방법 4: 프로그래밍 방식 (Node.js)

Node.js 애플리케이션에서 Lighthouse를 직접 호출할 수 있습니다.

```javascript
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

async function runLighthouse(url) {
  // Chrome 실행
  const chrome = await chromeLauncher.launch({
    chromeFlags: ['--headless']
  });

  // Lighthouse 실행
  const options = {
    logLevel: 'info',
    output: 'html',
    onlyCategories: ['performance', 'seo'],
    port: chrome.port
  };

  const runnerResult = await lighthouse(url, options);

  // 리포트 출력
  console.log('Report is done for', runnerResult.lhr.finalUrl);
  console.log('Performance score was', runnerResult.lhr.categories.performance.score * 100);

  // Chrome 종료
  await chrome.kill();
}

runLighthouse('https://example.com');
```

#### CI/CD 통합 예시

```javascript
// GitHub Actions에서 사용 예시
const fs = require('fs');
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

async function runCI() {
  const chrome = await chromeLauncher.launch({
    chromeFlags: ['--headless', '--no-sandbox']
  });

  const options = {
    logLevel: 'info',
    output: 'json',
    port: chrome.port
  };

  const runnerResult = await lighthouse('https://example.com', options);
  const score = runnerResult.lhr.categories.performance.score * 100;

  // 점수가 90 미만이면 빌드 실패
  if (score < 90) {
    console.error(`Performance score ${score} is below threshold`);
    process.exit(1);
  }

  // 리포트 저장
  fs.writeFileSync('lighthouse-report.json', JSON.stringify(runnerResult.lhr));

  await chrome.kill();
}

runCI();
```

---

## SEO 감사 기능

2018년 2월, Google은 Lighthouse Chrome 확장 프로그램에 SEO 감사 카테고리를 추가했습니다.  
이는 웹마스터들이 검색 엔진 최적화 상태를 쉽게 확인할 수 있도록 도와줍니다.  

### 주요 SEO 검사 항목

#### 1. 문서 메타데이터

```html
<!-- Good Example -->
<head>
  <title>명확하고 설명적인 제목 (50-60자)</title>
  <meta name="description" content="페이지의 내용을 정확히 설명하는 메타 설명 (150-160자)">
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
```

검사 항목:
- `<title>` 태그 존재 여부
- meta description 존재 여부
- viewport 설정 적절성

#### 2. 모바일 친화성

```html
<!-- Viewport 설정 -->
<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
```

검사 항목:
- viewport meta 태그 설정
- 텍스트 크기가 충분한지
- 터치 요소 간격이 적절한지
- 콘텐츠가 화면 크기에 맞는지

#### 3. 크롤링 가능성

```html
<!-- robots.txt -->
User-agent: *
Allow: /

<!-- 페이지 내 robots 메타 태그 -->
<meta name="robots" content="index, follow">
```

검사 항목:
- robots.txt가 유효한지
- 페이지가 차단되지 않았는지
- noindex가 부적절하게 사용되지 않았는지

#### 4. 링크 구조

```html
<!-- Good Example: 크롤 가능한 링크 -->
<a href="/about">About Us</a>

<!-- Bad Example: JavaScript 전용 링크 -->
<div onclick="navigate('/about')">About Us</div>
```

검사 항목:
- 모든 링크가 크롤 가능한지
- `<a>` 태그의 href 속성 존재
- 링크 텍스트가 설명적인지

#### 5. 구조화된 데이터

```html
<!-- JSON-LD 구조화된 데이터 예시 -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "기사 제목",
  "datePublished": "2025-12-07",
  "author": {
    "@type": "Person",
    "name": "작성자 이름"
  }
}
</script>
```

검사 항목:
- 유효한 구조화된 데이터
- Schema.org 마크업
- JSON-LD 형식 권장

#### 6. HTTP 상태 코드

검사 항목:
- 페이지가 200 OK를 반환하는지
- 불필요한 리다이렉트가 없는지
- 404 에러가 없는지

#### 7. 캐노니컬 URL

```html
<!-- 중복 콘텐츠 방지 -->
<link rel="canonical" href="https://example.com/original-page">
```

검사 항목:
- canonical 태그 적절성
- 자기 참조 캐노니컬
- HTTPS vs HTTP 일관성

### SEO 점수 해석

Lighthouse SEO 점수는 0-100점으로 표시됩니다:

- 90-100: 우수 (초록색)
- 50-89: 개선 필요 (주황색)
- 0-49: 불량 (빨간색)

### SEO 개선 전략

#### 1. 메타데이터 최적화

```html
<head>
  <!-- 제목 최적화: 주요 키워드 포함, 50-60자 -->
  <title>Lighthouse SEO 가이드 - 웹페이지 검색 최적화</title>

  <!-- 메타 설명: 설득력 있고 정확한 요약, 150-160자 -->
  <meta name="description" content="Google Lighthouse를 활용한 SEO 감사 및 최적화 방법을 상세히 알아봅니다. 검색 순위를 높이는 실전 가이드.">

  <!-- Open Graph (소셜 미디어 공유) -->
  <meta property="og:title" content="Lighthouse SEO 가이드">
  <meta property="og:description" content="웹페이지 검색 최적화 완벽 가이드">
  <meta property="og:image" content="https://example.com/image.jpg">

  <!-- Twitter Card -->
  <meta name="twitter:card" content="summary_large_image">
</head>
```

#### 2. 모바일 최적화

```css
/* 반응형 디자인 */
@media (max-width: 768px) {
  body {
    font-size: 16px; /* 최소 폰트 크기 */
  }

  button, a {
    min-height: 48px; /* 터치 영역 최소 크기 */
    min-width: 48px;
  }
}
```

#### 3. 페이지 속도 개선

SEO와 성능은 밀접한 관련이 있습니다. Core Web Vitals는 Google 검색 순위 요소입니다.

```html
<!-- 이미지 최적화 -->
<img src="image.webp" alt="설명적인 대체 텍스트"
     width="800" height="600" loading="lazy">

<!-- 중요 리소스 우선 로드 -->
<link rel="preload" href="/critical.css" as="style">
<link rel="preconnect" href="https://fonts.googleapis.com">
```

---

## 성능 최적화 방법

Lighthouse 성능 점수를 개선하는 실전 방법들입니다.

### 1. 이미지 최적화

#### WebP 포맷 사용

```html
<!-- picture 태그로 WebP 지원 -->
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg" type="image/jpeg">
  <img src="image.jpg" alt="설명">
</picture>
```

#### 적절한 크기 이미지 제공

```html
<!-- srcset으로 반응형 이미지 -->
<img srcset="small.jpg 480w, medium.jpg 800w, large.jpg 1200w"
     sizes="(max-width: 600px) 480px, (max-width: 1000px) 800px, 1200px"
     src="medium.jpg" alt="설명">
```

#### Lazy Loading

```html
<!-- 네이티브 지연 로딩 -->
<img src="image.jpg" loading="lazy" alt="설명">
```

### 2. 코드 스플리팅

#### React에서 동적 import

```javascript
// Before: 모든 컴포넌트를 한번에 로드
import HeavyComponent from './HeavyComponent';

// After: 필요할 때만 로드
const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

function App() {
  return (
    <React.Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </React.Suspense>
  );
}
```

#### Next.js 동적 import

```javascript
import dynamic from 'next/dynamic';

const DynamicComponent = dynamic(() => import('../components/Heavy'), {
  loading: () => <p>Loading...</p>,
  ssr: false // 클라이언트에서만 로드
});
```

### 3. 캐싱 전략

#### 브라우저 캐싱 설정

```nginx
# Nginx 설정
location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
  expires 1y;
  add_header Cache-Control "public, immutable";
}
```

#### Service Worker 캐싱

```javascript
// service-worker.js
const CACHE_NAME = 'v1';
const urlsToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
  );
});
```

### 4. CSS 최적화

#### Critical CSS 인라인

```html
<head>
  <!-- 중요 CSS를 인라인으로 포함 -->
  <style>
    /* Above-the-fold 콘텐츠에 필요한 CSS만 */
    body { margin: 0; font-family: system-ui; }
    .header { background: #333; color: white; }
  </style>

  <!-- 나머지 CSS는 비동기 로드 -->
  <link rel="preload" href="styles.css" as="style"
        onload="this.onload=null;this.rel='stylesheet'">
  <noscript><link rel="stylesheet" href="styles.css"></noscript>
</head>
```

#### 사용하지 않는 CSS 제거

```bash
# PurgeCSS 사용
npm install --save-dev @fullhuman/postcss-purgecss

# postcss.config.js
module.exports = {
  plugins: [
    require('@fullhuman/postcss-purgecss')({
      content: ['.//*.html', './/*.js'],
      defaultExtractor: content => content.match(/[\w-/:]+(?<!:)/g) || []
    })
  ]
}
```

### 5. JavaScript 최적화

#### 번들 크기 분석

```bash
# webpack-bundle-analyzer 설치
npm install --save-dev webpack-bundle-analyzer

# webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin()
  ]
};
```

#### Tree Shaking

```javascript
// package.json
{
  "sideEffects": false  // 모든 모듈에 부작용 없음
}

// 또는 특정 파일만 제외
{
  "sideEffects": ["*.css", "*.scss"]
}
```

#### 코드 압축

```javascript
// webpack.config.js
const TerserPlugin = require('terser-webpack-plugin');

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [new TerserPlugin({
      terserOptions: {
        compress: {
          drop_console: true, // console.log 제거
        },
      },
    })],
  },
};
```

### 6. 폰트 최적화

```html
<head>
  <!-- 폰트 파일 미리 연결 -->
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>

  <!-- display=swap으로 FOIT 방지 -->
  <link href="https://fonts.googleapis.com/css2?family=Roboto:wght@400;700&display=swap"
        rel="stylesheet">
</head>

<style>
  /* 로컬 폰트 사용 시 */
  @font-face {
    font-family: 'Custom';
    src: url('/fonts/custom.woff2') format('woff2');
    font-display: swap; /* 폰트 로딩 중에도 텍스트 표시 */
  }
</style>
```

### 7. CDN 활용

```html
<!-- 정적 리소스를 CDN에서 제공 -->
<script src="https://cdn.example.com/library.min.js"></script>
<img src="https://cdn.example.com/images/photo.jpg" alt="Photo">
```

### 8. 서버 성능 개선

#### GZIP 압축

```nginx
# Nginx
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css text/xml text/javascript
           application/x-javascript application/xml+rss
           application/javascript application/json;
```

#### HTTP/2 사용

```nginx
# Nginx
listen 443 ssl http2;
```

### 9. Third-party 스크립트 최적화

```html
<!-- 비동기 로딩 -->
<script async src="https://example.com/analytics.js"></script>

<!-- defer로 지연 실행 -->
<script defer src="https://example.com/widget.js"></script>

<!-- Facade 패턴: 사용자 상호작용 후 로드 -->
<div id="youtube-facade" onclick="loadYouTube()">
  <img src="thumbnail.jpg" alt="Video">
  <button>Play Video</button>
</div>

<script>
function loadYouTube() {
  const iframe = document.createElement('iframe');
  iframe.src = 'https://www.youtube.com/embed/VIDEO_ID';
  document.getElementById('youtube-facade').replaceWith(iframe);
}
</script>
```

### 10. 정기 점검 전략

웹사이트의 규모와 업데이트 빈도에 따라 점검 주기를 설정합니다:

#### 권장 점검 주기
- 주요 페이지: 월 1회
- 전체 사이트: 분기 1회
- 대규모 업데이트 후: 즉시
- CI/CD 파이프라인: 모든 배포마다

#### 자동화 스크립트 예시

```javascript
// lighthouse-ci.js
const fs = require('fs');
const lighthouse = require('lighthouse');
const chromeLauncher = require('chrome-launcher');

const urls = [
  'https://example.com',
  'https://example.com/about',
  'https://example.com/products'
];

async function auditSite() {
  const results = [];

  for (const url of urls) {
    const chrome = await chromeLauncher.launch({chromeFlags: ['--headless']});
    const options = {port: chrome.port};
    const runnerResult = await lighthouse(url, options);

    results.push({
      url,
      performance: runnerResult.lhr.categories.performance.score * 100,
      seo: runnerResult.lhr.categories.seo.score * 100,
      accessibility: runnerResult.lhr.categories.accessibility.score * 100
    });

    await chrome.kill();
  }

  // 결과를 JSON 파일로 저장
  fs.writeFileSync('lighthouse-results.json', JSON.stringify(results, null, 2));

  // 임계값 체크
  const failed = results.filter(r =>
    r.performance < 90 || r.seo < 90 || r.accessibility < 90
  );

  if (failed.length > 0) {
    console.error('Performance threshold not met:', failed);
    process.exit(1);
  }
}

auditSite();
```

---

## 결론

Google Lighthouse는 웹사이트의 성능, SEO, 접근성을 종합적으로 평가하고 개선할 수 있는 강력한 도구입니다.

### 핵심 포인트
- Chrome DevTools에 내장되어 있어 쉽게 사용 가능
- 성능, SEO, 접근성, PWA 등 5가지 영역을 자동으로 평가
- 구체적인 개선 제안과 점수를 제공
- CLI 및 프로그래밍 방식으로 자동화 가능
- CI/CD 파이프라인에 통합하여 지속적인 품질 관리 가능

### 2021년 이후 중요성

2021년 이후 Google은 Core Web Vitals를 검색 순위 요소로 공식 발표했습니다.  
따라서 Lighthouse를 통한 성능 측정과 개선은 SEO 전략의 필수 요소가 되었습니다.  

### 시작하기

1. Chrome 브라우저에서 F12를 눌러 개발자 도구를 엽니다
2. Lighthouse 탭으로 이동합니다
3. 평가 항목을 선택하고 분석을 실행합니다
4. 제안 사항을 하나씩 적용하여 점수를 개선합니다

정기적인 Lighthouse 점검을 통해 웹사이트의 품질을 지속적으로 개선하고 사용자 경험과 검색 순위를 향상시킬 수 있습니다.

---

## 참고 자료

- [Google Developers - Lighthouse](https://developers.google.com/web/tools/lighthouse)
- [Lighthouse GitHub Repository](https://github.com/GoogleChrome/lighthouse)
- [Web.dev - Lighthouse Performance Scoring](https://web.dev/performance-scoring/)
- [Core Web Vitals](https://web.dev/vitals/)
