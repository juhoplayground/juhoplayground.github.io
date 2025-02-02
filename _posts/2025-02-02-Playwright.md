---
layout: post
title: Playwright를 이용한 웹 크롤링
author: 'Juho'
date: 2025-02-01 09:00:00 +0900
categories: [Playwright]
tags: [Playwright, Python]
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
1. [Playwright?](#celery-airflow-retry)
 - 1) [Variable 설정(UI)](#1-variable-설정ui)
 - 2) [Variable 설정(Code)](#2-variable-설정code)
 - 3) [Variable 사용](#3-variable-사용)

## Playwright?
[Playwrigh](https://playwright.dev/python/){:target="_blank"}는 Microsoft에서 만든 웹 브라우저 테스트 및 웹 스크래핑을 위한 오픈 소스 자동화 라이브러리이다.<br/>
개발실의 다른 인원이 Selenium으로 만든 웹 크롤링하는 프로젝트에서 크롤링이 막혀서 도움을 요청했다.<br/>
해당 인원이 여러 방법을 사용해보았지만 크롤링이 자꾸 막혀서 어떤 방법으로 해보면 좋겠는지 고민을 해달라는 내용이였다.<br/>
<br/>
내가 생각하는 크롤링이 자꾸 막히는 이유는 Selenium의 동작 방식에 있다고 생각을 했다.<br/>
Selenium은 WebDriver를 사용하여 브라우저를 제어하는데 이 과정에서 브라우저와 별도로 실행되는 외부 프로세스로 동작한다.<br/>
이러한 특성으로 인해 Google과 같은 검색 엔진에서는 Selenium의 사용 여부를 쉽게 감지할 수 있으며 reCAPTCHA를 요구하거나 특정 요청을 차단하는 방식으로 대응하고 있다.<br/>
Selenium을 탐지하는 방법은 계속 발전하고 있으며 대응하는 우회 기법들도 등장하고 있다.<br/>
하지만 reCAPTCHA를 매번 해결하는 것은 현실적으로 어렵고 지속적인 우회 방법을 적용하는 것도 장기적인 해결책이 되기 어렵다고 생각했다.<br/>
<br/>
이 문제를 근본적으로 해결하기 위해서는 WebDriver를 사용하지 않고 브라우저를 제어하는 방식이 필요했다.
이를 위해 Microsoft에서 개발한 Playwright를 고려했다.<br/>
Playwright는 DevTools Protocol을 기반으로 동작한다.
Selenium처럼 브라우저를 직접 제어하는 것이 아니라 브라우저의 내부 API를 활용하여 자동화를 수행한다.<br/>
이러한 방식은 Selenium보다 탐지가 어렵고 상대적으로 안정적인 자동화 환경을 구축할 수 있는 장점이 있다.<br/>

#### 1) 설치 방법
```
sudo apt-get install libnss3 libnspr4  libasound2
pip install playwright
playwright install
```


#### 2) 간단한 구현 테스트
```python
from playwright.async_api import async_playwright
from playwright_stealth import stealth_async
import urllib.parse
import asyncio


async def set_extra_http_headers(page):
    await page.set_extra_http_headers({
    "accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
    "accept-encoding": "gzip, deflate, br, zstd",
    "accept-language": "ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7",
    "cache-control": "max-age=0",
    "downlink": "10",
    "priority": "u=0, i",
    "rtt": "100",
    "sec-ch-prefers-color-scheme": "light",
    "sec-ch-ua": "\"Not A(Brand\";v=\"8\", \"Chromium\";v=\"132\", \"Google Chrome\";v=\"132\"",
    "sec-ch-ua-arch": "x86",
    "sec-ch-ua-bitness": "64",
    "sec-ch-ua-form-factors": "Desktop",
    "sec-ch-ua-full-version": "132.0.6834.160",
    "sec-ch-ua-full-version-list": "\"Not A(Brand\";v=\"8.0.0.0\", \"Chromium\";v=\"132.0.6834.160\", \"Google Chrome\";v=\"132.0.6834.160\"",
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-model": "\"\"",
    "sec-ch-ua-platform": "Windows",
    "sec-ch-ua-platform-version": "19.0.0",
    "sec-ch-ua-wow64": "?0",
    "sec-fetch-dest": "document",
    "sec-fetch-mode": "navigate",
    "sec-fetch-site": "same-origin",
    "sec-fetch-user": "?1",
    "upgrade-insecure-requests": "1",
    "user-agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36"})
    return page

async def do_test(context, keyword):
    term = urllib.parse.quote_plus(keyword)
    url = f"https://www.google.com/search?q={term}"

    page = await browser.new_page()
    await stealth_async(page)
    page = await set_extra_http_headers(page)
    await page.evaluate("""Object.defineProperty(navigator, 'webdriver', {get: () => undefined});""")

    await page.goto(url, wait_until="domcontentloaded")

    screenshot_path = f"{keyword}_test.png"
    await page.screenshot(path=screenshot_path, full_page=True)
    await page.close()


async def main():
    keyword_list = ["Python", "Playwright"]
    async with async_playwright() as pw:
        browser = await pw.chromium.launch(headless=True,
                                           executable_path="/usr/bin/google-chrome-stable",
                                           args=["--disable-blink-features=AutomationControlled", "--no-sandbox"])
        
        tasks = [do_test(browser, keyword) for keyword in keyword_list]
        await asyncio.gather(*tasks)
        await browser.close()


if __name__ == "__main__":
    asyncio.run(main())
```


---

<br/>

정상적으로 조회가 잘 되는 것을 확인할 수 있었다.<br/>
