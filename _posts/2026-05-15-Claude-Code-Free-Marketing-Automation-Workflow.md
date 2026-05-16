---
layout: post
title: "Claude Code 기반 무료 마케팅 자동화 워크플로우"
author: 'Juho'
date: 2026-05-15 00:00:00 +0900
categories: [MCP]
tags: [MCP, Skill, AI]
pin: True
toc: True
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
1. [개요](#개요)
2. [배경](#배경)
3. [연결 가능한 마케팅 플랫폼](#연결-가능한-마케팅-플랫폼)
4. [무료 도구 카탈로그](#무료-도구-카탈로그)
   - [SEO 자동화 - Toprank](#seo-자동화---toprank)
   - [광고 종합 감사 - Claude Ads](#광고-종합-감사---claude-ads)
   - [멀티플랫폼 광고 실행 - amekala/ads-mcp](#멀티플랫폼-광고-실행---amekalaads-mcp)
   - [Microsoft/Bing Ads 실행](#microsoftbing-ads-실행)
   - [Apple Search Ads 실행](#apple-search-ads-실행)
   - [Google과 YouTube Ads 실행](#google과-youtube-ads-실행)
   - [LinkedIn과 TikTok 단독 옵션](#linkedin과-tiktok-단독-옵션)
5. [통합 워크플로우 구성](#통합-워크플로우-구성)
   - [셋업](#셋업)
   - [월간 - 전략과 감사](#월간---전략과-감사)
   - [주간 - 실행](#주간---실행)
   - [일간 - 모니터링](#일간---모니터링)
6. [이 구성의 장점](#이-구성의-장점)
7. [주의점](#주의점)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Claude Code는 단일 IDE 환경에서 SEO와 7개 광고 플랫폼을 한꺼번에 운영할 수 있는 마케팅 자동화 허브로 진화하고 있다.
플러그인과 MCP(Model Context Protocol) 서버 생태계가 두꺼워지면서, 대시보드 클릭 대신 자연어 명령으로 키워드를 조정하고, 캠페인 예산을 변경하고, 메타태그를 배포하는 흐름이 가능해졌다.
주목할 점은 이 모든 구성을 무료 오픈소스 도구만으로 끝낼 수 있다는 것이다.
Toprank, Claude Ads, amekala/ads-mcp, Microsoft Ads MCP, Apple Ads MCP, Google Ads MCP를 셀프호스팅으로 묶으면, 외부 SaaS 비용 없이 Google Ads, Meta, YouTube, LinkedIn, TikTok, Microsoft, Apple Search Ads와 SEO까지 단일 워크플로우로 통합된다.

## 배경

전통적으로 마케팅 운영은 분절되어 있었다.
Google Ads UI에서 키워드를 손보고, Search Console에서 랭킹을 확인하고, Meta Ads Manager에서 크리에이티브 성과를 보고, LinkedIn Campaign Manager에서 ABM 캠페인을 점검하는 식으로 도구마다 컨텍스트가 분리되었다.
플랫폼 간 데이터를 비교하려면 별도 BI 도구나 스프레드시트로 합쳐야 했고, 의사결정 사이클이 길어질 수밖에 없었다.

MCP 표준이 등장하면서 외부 API와 LLM의 연결이 일관된 형태로 표준화되었다.
각 광고 플랫폼의 API를 MCP 서버로 감싸면, Claude Code가 곧바로 그 도구들을 호출할 수 있다.
그 위에 Toprank처럼 도메인 특화 플러그인이 쌓이고, Claude Ads처럼 감사 전용 스킬이 추가되면서, 마케팅 운영 전체가 단일 IDE 안에서 완결된다.
이 글은 그 결과 만들어지는 무료 풀스택 구성을 정리한다.

## 연결 가능한 마케팅 플랫폼

Claude Code 위에서 무료로 연결 가능한 플랫폼은 다음과 같다.

| 영역 | 플랫폼 | 자동 실행 가능 | 감사 가능 |
|------|--------|----------------|-----------|
| SEO | Google Search Console | O (Toprank) | O |
| SEO | WordPress, Strapi, Contentful, Ghost | O (Toprank CMS 커넥터) | - |
| 광고 | Google Ads | O | O |
| 광고 | YouTube Ads | O (Google Ads MCP 하위) | O |
| 광고 | Meta (Facebook, Instagram) | O | O |
| 광고 | LinkedIn Ads | O | O |
| 광고 | TikTok Ads | O | O |
| 광고 | Microsoft/Bing Ads | O | O |
| 광고 | Apple Search Ads | O | O |

7개 광고 플랫폼과 SEO 영역이 모두 자연어로 운영 가능하며, 모든 도구가 GitHub에 공개된 오픈소스다.

## 무료 도구 카탈로그

### SEO 자동화 - Toprank

NotFair에서 개발한 오픈소스 Claude Code 플러그인으로, SEO 영역에서 가장 풍부한 8개 스킬을 제공한다.

| 스킬 | 기능 |
|------|------|
| SEO Audit | Google Search Console 데이터 기반 종합 감사 |
| Content | E-E-A-T 가이드라인을 따르는 콘텐츠 작성 |
| Keyword Research | 키워드 리서치와 토픽 클러스터링 |
| Meta | A/B 변형이 있는 메타 태그 최적화 |
| Schema | JSON-LD 구조화 데이터 생성 |
| Page Analysis | 단일 페이지 심층 분석 |
| Broken Links | 깨진 링크 감지 |
| GEO | AI 검색 엔진을 위한 Generative Engine Optimization |

WordPress, Strapi, Contentful, Ghost 같은 CMS와 네이티브로 연결되어, SEO 권고가 곧바로 콘텐츠 변경으로 이어진다.
Gemini 통합을 통해 중요한 의사결정 시 다른 모델의 검증을 받는 두 번째 의견 구조도 제공한다.
설치는 다음 두 줄로 끝난다.

```bash
/plugin marketplace add nowork-studio/toprank
/plugin install toprank@nowork-studio
```

### 광고 종합 감사 - Claude Ads

7개 광고 플랫폼을 대상으로 250개 이상의 감사 체크를 수행하는 오픈소스 스킬이다.

| 플랫폼 | 체크 항목 |
|--------|-----------|
| Google Ads | 80+ |
| Meta | 50+ |
| YouTube | 포함 |
| LinkedIn | 포함 |
| TikTok | 포함 |
| Microsoft/Bing | 포함 |
| Apple Ads | 포함 |

가장 큰 차별점은 광고 계정에 자동 로그인하거나 API를 직접 연결하지 않는다는 점이다.
사용자가 내보낸 데이터, 스크린샷, 붙여넣은 지표만으로 분석을 수행하므로 로컬 환경에서 완결적으로 동작한다.
0~100 스케일의 Ads Health Score를 산출하고, 11종 산업 템플릿(SaaS, 이커머스, 로컬 서비스, B2B 엔터프라이즈, 모바일 앱, 부동산, 헬스케어, 금융 등)을 통해 산업별 적정 벤치마크를 자동 로딩한다.
부동산, 신용, 금융 같은 Special Ad Category 산업에는 컴플라이언스 체크가 자동 활성화된다.

```bash
/plugin marketplace add AgriciDaniel/claude-ads
/plugin install claude-ads@agricidaniel-claude-ads
```

### 멀티플랫폼 광고 실행 - amekala/ads-mcp

Google Ads, Meta Ads, LinkedIn Ads, TikTok Ads 4개 플랫폼을 100+ 도구로 통합한 오픈소스 MCP 서버다.
캠페인 생성, 성과 분석, 키워드 리서치, 예산 최적화를 단일 MCP에서 처리한다.
Claude Code, Cursor, ChatGPT, Gemini CLI, Codex, Windsurf 모두와 호환되며, 셀프호스팅하면 외부 SaaS 비용 없이 4개 플랫폼이 한 번에 해결된다.

### Microsoft/Bing Ads 실행

Duartemartins/microsoft-ads-mcp-server는 MIT 라이선스로 공개된 Microsoft Advertising MCP 서버다.
Bing Ads와 DuckDuckGo Ads를 대상으로 캠페인 생성, 관리, 리포트까지 CRUD 지원이 포함되어 있다.
OAuth로 인증한 뒤 라이브 계정에 직접 명령을 실행할 수 있다.

### Apple Search Ads 실행

ppcprophet/apple-ads-mcp는 Apple Search Ads API를 감싼 오픈소스 MCP다.
캠페인, 키워드, 입찰 관리가 가능하며, 인증서(P8 키) 기반 OAuth를 사용한다.
주간 성과 요약, Discovery 캠페인 검색어 자동 분석, 임계값 초과 시 키워드 자동 추가 같은 자동화도 구성할 수 있다.

### Google과 YouTube Ads 실행

google-marketing-solutions/google_ads_mcp는 Google이 직접 공개한 공식 MCP 서버다.
YouTube Ads는 Google Ads API의 하위 영역이므로 이 MCP 하나로 검색·디스플레이·PMax·YouTube까지 모두 커버된다.
googleads/google-ads-mcp 역시 Google 공식 저장소다.

### LinkedIn과 TikTok 단독 옵션

amekala/ads-mcp 대신 단독 MCP를 쓰고 싶다면 다음 옵션이 있다.

| 플랫폼 | 저장소 | 특징 |
|--------|--------|------|
| LinkedIn Ads | ZLeventer/linkedin-campaign-manager-mcp | 19개 도구, 캠페인, 예산, Lead Gen Form, 전환, 타겟팅 |
| LinkedIn Ads | danielpopamd/linkedin-ads-mcp | 분석과 인사이트 중심 |
| TikTok Ads | AdsMCP/tiktok-ads-mcp-server | 캠페인과 광고그룹 CRUD |
| TikTok Ads | ysntony/tiktok-ads-mcp | TikTok Business API, 보고서 |

단독 MCP는 도구 노출 범위가 좁아 정밀 제어에 유리하지만, OAuth 관리 부담이 늘어난다.

## 통합 워크플로우 구성

### 셋업

5개 저장소를 한 번에 셋업하면 풀스택 환경이 완성된다.

```bash
/plugin marketplace add nowork-studio/toprank
/plugin install toprank@nowork-studio

/plugin marketplace add AgriciDaniel/claude-ads
/plugin install claude-ads@agricidaniel-claude-ads
```

이후 Claude Code의 `~/.claude/settings.json` 또는 `.mcp.json`에 다음 MCP 서버들을 추가한다.

- amekala/ads-mcp (LinkedIn, TikTok 보강)
- Duartemartins/microsoft-ads-mcp-server (Bing)
- ppcprophet/apple-ads-mcp (Apple Search Ads)
- google-marketing-solutions/google_ads_mcp (Google, YouTube)

각 MCP는 OAuth 토큰 또는 API 키를 .env에 저장하고, OS 키체인이나 시크릿 매니저로 보호한다.
Claude Code 안에서 `/mcp` 명령으로 모든 서버가 정상 연결되었는지 확인한다.

### 월간 - 전략과 감사

목적은 어디에 문제가 있는가를 정량화하는 것이다.

| 순서 | 도구 | 명령 | 산출물 |
|------|------|------|--------|
| 1 | 수동 | 광고 데이터 export | CSV와 스크린샷 |
| 2 | Claude Ads | /ads audit | 7개 플랫폼 종합 Health Score |
| 3 | Claude Ads | /ads math, /ads test | CPA, ROAS, A/B 설계 |
| 4 | Claude Ads | /ads report | 클라이언트용 PDF |
| 5 | Toprank | /toprank:seo audit | GSC 기반 SEO 감사 |

Health Score와 SEO 감사 결과를 다음 달 예산과 콘텐츠 우선순위로 변환한다.

### 주간 - 실행

감사로 도출된 액션을 실제 계정에 반영한다.

화요일은 광고 운영이다.
Toprank로 Google과 Meta를 처리하고, amekala/ads-mcp로 LinkedIn과 TikTok을, Microsoft Ads MCP로 Bing을, Apple Ads MCP로 Apple Search Ads를 처리한다.
모두 자연어 명령으로 입찰, 예산, 부정 키워드, 광고세트 조정이 가능하다.

수요일은 크리에이티브다.
/toprank:google-ads copy로 RSA 카피를 생성하고, /ads creative로 크로스플랫폼 품질 평가를 거친다.
통과한 안만 각 플랫폼 MCP로 게재한다.

목요일은 랜딩과 SEO 연계다.
/toprank:google-ads landing으로 광고와 랜딩 일치도를 평가하고, /toprank:seo meta로 메타태그 A/B 변형을 CMS에 직접 반영한다.
/toprank:seo content는 E-E-A-T 가이드라인을 따르는 콘텐츠를 생성한다.

### 일간 - 모니터링

매일 5분에서 10분의 상태 점검 루틴이다.

| 항목 | 도구 | 명령 |
|------|------|------|
| 깨진 링크 | Toprank | /toprank:seo broken-links |
| 캠페인 이상 (CPA 3배 초과 등) | Claude Ads | /ads google 또는 /ads meta |
| 자동 SEO 작업 | Toprank openclaw | 크론 자동 실행 |

Claude Ads의 강제 게이트는 데일리 알람 역할을 한다.
타겟 CPA 3배 초과 캠페인을 즉시 검토 대상으로 플래그하고, Smart Bidding 미사용 시 Broad Match 추천을 차단한다.

## 이 구성의 장점

첫째, 비용이 0이다.
모든 도구가 오픈소스이고 셀프호스팅이므로, Adspirer, Pipeboard, Blend MCP, Synter 같은 호스팅 SaaS의 월 구독료가 발생하지 않는다.
키 관리와 서버 운영만 자체 부담이다.

둘째, 데이터가 외부로 나가지 않는다.
Claude Ads는 사용자가 내보낸 데이터로만 동작하고, 다른 MCP는 본인 OAuth 토큰으로 직접 API를 호출한다.
중간 SaaS를 경유하지 않으므로 광고 계정 정보, 매출 지표, 키워드 전략이 외부에 캐싱되지 않는다.

셋째, 자연어 단일 인터페이스다.
7개 광고 대시보드와 GSC, CMS를 오갈 필요 없이, Claude Code 한 곳에서 모든 작업이 끝난다.
플랫폼별 UI 학습 비용이 사라지고, 신규 마케터의 온보딩이 빨라진다.

넷째, 감사와 실행이 분리되어 있다.
Claude Ads의 정량 감사가 우선 동작하고, 실행은 별도 도구가 처리하는 구조다.
실행 자동화에 따른 사고 위험이 분리되며, 변경 전 후 비교가 명확해진다.

다섯째, 컴플라이언스가 자동으로 적용된다.
Claude Ads는 부동산, 신용, 금융 같은 Special Ad Category에서 컴플라이언스 체크를 자동 활성화한다.
Smart Bidding 없이 Broad Match 추천을 차단하고, 타겟 CPA 3배 초과 캠페인을 강제 플래그하는 등의 게이트가 기본 내장되어 있다.

여섯째, 도구 종속을 피할 수 있다.
Toprank는 `~~category` 형태의 플레이스홀더를 사용해 다른 MCP 구현으로 갈아끼울 수 있도록 설계되어 있다.
NotFair MCP 서버에 묶이지 않고도 동일 스킬을 활용할 수 있고, 멀티플랫폼 MCP인 amekala/ads-mcp 대신 단독 MCP로 교체하는 것도 가능하다.

일곱째, 점진적 누적이 가능하다.
스킬들은 비즈니스 프로필을 저장해 다른 스킬이 이전 감사 결과를 참조할 수 있게 한다.
단발성 명령이 아닌 점진적으로 누적되는 마케팅 의사결정 구조가 만들어진다.

## 주의점

OAuth 토큰을 .env에 저장하는 방식이 일반적이므로, 키 노출 방지 정책을 먼저 정해두는 게 좋다.
Apple Search Ads는 인증서(P8 키) 발급이 필요하다.

도구 간 데이터 시점이 어긋날 수 있다.
Toprank나 ads-mcp가 변경 중인 캠페인을 동시에 Claude Ads로 분석하면 결과가 일관되지 않으므로, 감사는 변경 전 데이터로 고정하는 것이 안전하다.

자동 실행 범위에는 의도적 제약이 있다.
Toprank의 Meta는 캠페인 생성을 제외하고, Claude Ads는 자동 로그인 자체를 하지 않는다.
신규 캠페인 생성 같은 위험도 높은 작업은 사람이 직접 처리하도록 분리되어 있다.

호스팅형 SaaS와 혼동하지 않도록 한다.
Adspirer, Pipeboard, Blend MCP, Synter, Windsor, appleadsmcp.com, Two Minute Reports는 자체 호스팅 무료 옵션이 아니라 매니지드 SaaS다.
무료 풀스택 구성에서는 위에서 정리한 GitHub 오픈소스 저장소만 사용해야 한다.

## 결론

Claude Code 기반 무료 마케팅 자동화는 다섯 개의 GitHub 저장소만 셀프호스팅하면 완성된다.
SEO와 7개 광고 플랫폼이 단일 IDE 안에서 자연어 명령으로 운영되며, 호스팅 SaaS 비용이 발생하지 않는다.
감사는 Claude Ads, SEO와 Google/Meta 실행은 Toprank, 나머지 플랫폼은 멀티플랫폼 또는 단독 MCP로 채우는 구조가 핵심이다.
컴플라이언스 자동화, 데이터 외부 유출 방지, 도구 종속 회피, 점진적 누적 같은 설계 원칙이 모두 기본 내장되어 있어, 다른 도메인의 Claude Code 자동화에도 그대로 응용 가능한 레퍼런스다.

## Reference

- [nowork-studio/toprank - GitHub](https://github.com/nowork-studio/toprank/)
- [AgriciDaniel/claude-ads - GitHub](https://github.com/AgriciDaniel/claude-ads)
- [amekala/ads-mcp - GitHub](https://github.com/amekala/ads-mcp)
- [ZLeventer/linkedin-campaign-manager-mcp - GitHub](https://github.com/ZLeventer/linkedin-campaign-manager-mcp)
- [danielpopamd/linkedin-ads-mcp - GitHub](https://github.com/danielpopamd/linkedin-ads-mcp)
- [AdsMCP/tiktok-ads-mcp-server - GitHub](https://github.com/AdsMCP/tiktok-ads-mcp-server)
- [ysntony/tiktok-ads-mcp - GitHub](https://github.com/ysntony/tiktok-ads-mcp)
- [Duartemartins/microsoft-ads-mcp-server - LobeHub](https://lobehub.com/mcp/duartemartins-microsoft-ads-mcp-server)
- [ppcprophet/apple-ads-mcp - GitHub](https://github.com/ppcprophet/apple-ads-mcp)
- [google-marketing-solutions/google_ads_mcp - GitHub](https://github.com/google-marketing-solutions/google_ads_mcp)
- [googleads/google-ads-mcp - GitHub](https://github.com/googleads/google-ads-mcp)
