---
layout: post
title: "루프란 무엇인가: Claude·GPT·Mira로 보는 AI 루프의 작동 원리와 비용"
author: 'Juho'
date: 2026-06-22 00:00:00 +0900
categories: [AI]
tags: [AI, Agent, Skill]
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
2. [프롬프트와 루프의 차이](#프롬프트와-루프의-차이)
   - [루프의 세 가지 핵심](#루프의-세-가지-핵심)
3. [루프가 정말 필요한가](#루프가-정말-필요한가)
4. [코드를 위한 루프와 다섯 가지 구성 요소](#코드를-위한-루프와-다섯-가지-구성-요소)
5. [아무도 말하지 않는 비용](#아무도-말하지-않는-비용)
6. [직접 만들어보는 기본 루프](#직접-만들어보는-기본-루프)
7. [실생활을 위한 루프, Mira](#실생활을-위한-루프-mira)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

AI는 이미 수년째 모두의 손안에 있지만, 매일 쓰는 사람조차 대부분 가장 느린 방식으로 사용한다.
요청을 입력하고, 기다리고, 고치고, 다시 묻는 일을 전부 손으로 반복하는 것이다.
이 글은 그 느린 방식의 대안인 루프(loop)가 무엇인지, 어떻게 작동하는지, 언제 가치가 있고 언제 함정이 되는지를 설명한다.
요청한 내용은 Anatoli Kopadze의 X 글 "Loops explained: Claude, GPT, Mira and what actually works"를 기반으로 정리했다.

## 프롬프트와 루프의 차이

한 번에 하나의 요청을 보내는 습관에서는 모든 단계가 사람을 거친다.
무엇을 물을지 정하고, 답을 판단하고, 다음에 무엇을 할지 결정하는 주체가 사람이다.
사람이 밀지 않으면 AI는 움직이지 않고, 멈추는 순간 작업도 멈춘다.
즉 사람이 엔진이고 AI는 손에 든 도구일 뿐이다.

루프는 다르다.
목표를 한 번만 주면 AI가 단계를 스스로 실행한다.
계획하고, 작업하고, 자기 결과를 검증하고, 약한 부분을 고치며, 목표가 충족될 때까지 반복한다.
프롬프트가 단일 명령이라면, 루프는 AI가 도달할 때까지 계속 일하는 목표다.
하나의 재귀적 목표(recursive goal)로 이해할 수 있으며, 다음 다섯 단계의 사이클을 스스로 돈다.

| 단계 | 의미 |
|------|------|
| DISCOVER | 무엇을 해야 하는지 파악 |
| PLAN | 어떻게 할지 결정 |
| EXECUTE | 실제 작업 수행 |
| VERIFY | 목표와 대조해 검증 |
| ITERATE | 아직 도달하지 못했다면 결과를 다시 넣고 반복 |

### 루프의 세 가지 핵심

다섯 단계 중 세 가지가 실제 작업의 전부를 담당하며, 사람들이 루프를 잘못 만드는 지점도 여기다.

검증(Verify)은 루프의 심장이다.
결과를 실제로 점검하지 않으면 그것은 루프가 아니라 에이전트가 스스로에게 동의하는 반복일 뿐이다.
점검은 하드 테스트("코드가 통과하는가"), 측정 가능한 조건("수치가 X 이상인가"), 또는 모델이 스스로 채점하는 루브릭일 수 있다.
게이트가 없으면 작업한 모델이 자기 숙제를 채점하게 되는데, 그 모델은 지나치게 후한 채점자다.

상태(State)는 루프가 학습하게 만든다.
매 반복마다 이미 시도한 것을 기억하지 못하면 같은 실수를 영원히 되풀이한다.
무엇이 완료됐고, 무엇이 실패했고, 다음이 무엇인지 작은 기록을 옆에 남겨야 다음 실행이 0에서 시작하지 않는다.

정지 조건(Stop condition)은 루프를 제정신으로 유지한다.
출구가 없는 루프는 성공하거나, 망가지거나, 계정 잔고를 다 쓸 때까지 돈다.
진지한 루프는 성공과 하드 리밋("8번 시도 후 멈추고 보고") 두 가지 종료 방식을 갖는다.

## 루프가 정말 필요한가

대부분의 글은 루프가 언제 실수인지 말하기 전에 루프를 판다.
진지한 사람들이 쓰는 테스트는, 다음 네 가지가 모두 참일 때만 루프를 만들 가치가 있다는 것이다.

| 조건 | 설명 |
|------|------|
| 작업이 반복된다 | 최소 주 1회 이상. 그보다 드물면 구축 비용을 회수하지 못한다 |
| 나쁜 출력을 자동으로 거부할 수 있다 | 테스트, 타입 체크, 빌드, 린터, 하드 룰 등 |
| 에이전트가 끝까지 스스로 처리할 수 있다 | 절반을 사람에게 되돌려주지 않는다 |
| 완료가 객관적이다 | 판단이 아닌 명확한 기준. 취향 문제라면 사람이 낫다 |

하나라도 충족하지 못하면 수동 프롬프트로 두는 편이 낫다.
루프 엔지니어링은 실재하지만, 대부분의 사람은 아직 무거운 버전이 필요하지 않다.

## 코드를 위한 루프와 다섯 가지 구성 요소

루프는 소프트웨어에서 먼저 떴는데, 코드는 검증이 가장 쉽기 때문이다.
테스트는 통과하거나 실패하므로 AI가 끝났는지 항상 알 수 있다.
코딩 루프는 목표와 엄격한 검증 방식을 함께 받는다.

```
▸ LOOP SPEC
GOAL: every test in /tests/auth passes, lint is clean, no type errors.

EACH ITERATION:
  1. run the test suite and read every failure
  2. pick the single highest-impact failure
  3. write the smallest change that fixes it
  4. re-run the tests, lint, and type checker

VERIFY: green tests + zero lint warnings + zero type errors
STOP WHEN: verify passes, OR 8 iterations reached
ON STOP: summarize what changed and what still fails
```

내부적으로 실제 루프는 다섯 개의 빌딩 블록으로 조립되며, Claude Code와 Codex가 이 다섯 가지를 모두 제공한다.

| 구성 요소 | 역할 |
|------|------|
| 자동화(heartbeat) | 스케줄에 따라 루프를 시작하는 트리거. Claude Code의 /loop, /goal, hooks, cron, GitHub Actions |
| 스킬(reusable instructions) | 규칙과 패턴, 건드리면 안 되는 목록을 파일로 저장해 매번 읽게 함 |
| 서브 에이전트 | 작업하는 에이전트와 검증하는 에이전트를 분리. 작성자는 빠르고 저렴하게, 검증자는 느리고 엄격하게 |
| 커넥터 | 제안에 그치지 않고 PR을 열고 티켓을 연결하는 등 실제 환경에서 행동하게 함 |
| 검증기(gate) | 나쁜 작업을 자동으로 거부하는 테스트·타입 체크·빌드. 루프를 진짜로 만드는 부분 |

이를 쌓으면 큰 팀이 운영하는 규모, 즉 수십에서 수천 개의 에이전트가 같은 작업을 도는 함대(fleet)가 된다.
한 엔지니어는 이런 루프로 전체 코드베이스를 다른 프로그래밍 언어로 약 6일 만에 다시 작성했는데, 손으로 하면 1년 가까이 걸릴 일이었다.

## 아무도 말하지 않는 비용

루프는 토큰으로 돌고, 토큰은 돈이다.
문제는 각 단계가 비용을 든다는 점이 아니라 비용이 복리로 불어난다는 점이다.
매 반복마다 에이전트는 목표, 코드, 마지막 결과, 실패 내역을 다시 읽으며, 그 더미는 매 패스마다 커진다.
10번 도는 루프는 프롬프트 10개가 아니라, 각각 점점 커지는 프롬프트 10개의 비용이다.
품질을 높이는 작성자-검증자 분리 기법은 두 모델이 작업을 읽으므로 청구액도 두 배가 된다.

| 항목 | 대략적 비용 |
|------|------|
| 단일 에이전트, 중간 난도 작업 1회 | 약 50,000 - 200,000 토큰 |
| 매 반복마다 재전송되는 컨텍스트 | 패스마다 증가 |
| 병렬로 도는 에이전트 함대 | 위 전부를 곱한 값 |

정작 중요하지만 거의 아무도 추적하지 않는 지표는 채택된 변경당 비용(cost per accepted change)이다.
루프가 결과 10개를 주는데 6개를 버린다면, 루프가 절약하려던 리뷰 작업을 사람이 하고 있는 셈이다.
채택률 50% 아래에서는 루프가 돌려주는 것보다 더 많이 든다.
또한 루프는 조용히 실패한다.
엔지니어 Geoffrey Huntley는 이를 "Ralph Wiggum 루프"라 부르는데, 에이전트가 너무 일찍 끝났다고 판단해 절반만 한 작업으로 빠져나가고도 루프는 계속 돌며 돈만 쓰는 현상이다.
작업을 실패시킬 수 있는 하드 게이트가 없으면 루프는 충돌하지 않고 조용히 청구한다.

## 직접 만들어보는 기본 루프

코딩 에이전트가 없어도 어떤 LLM에서든 프롬프트만으로 간단한 루프를 직접 돌려볼 수 있다.
요령은 모델에게 세 가지 루프 요소를 한 번에 주는 것이다.
목표, 엄격한 성공 기준, 그리고 멈추기 전에 스스로 점검하도록 강제하는 프로토콜이다.

```
▸ SELF-CHECKING LOOP  (paste into Claude or ChatGPT)
You will work in a loop until the task meets the bar.

TASK:
[describe exactly what you want produced]

SUCCESS CRITERIA (be strict, no soft passes):
- [criterion 1]
- [criterion 2]
- [criterion 3]

LOOP PROTOCOL, repeat every turn:
1. PLAN   - state the single next step.
2. DO     - produce or improve the work.
3. VERIFY - score the result 1-10 on each criterion.
            Be brutally honest. List exactly what is still weak.
4. DECIDE - if every criterion is 8+, print "FINAL" and stop.
            Otherwise print "ITERATING" and go again, fixing
            the weakest point first.

RULES:
- Never call it done until every criterion is 8 or higher.
- Each pass must fix the weakest score from the last VERIFY.
- Do not ask me questions. Make a sensible assumption, note it,
  and keep going.

Begin. Run the loop until FINAL.
```

모델은 초안을 쓰고, 기준에 맞춰 스스로 채점하고, 약한 지점을 찾아 다시 쓰기를 반복하며, 기준을 실제로 넘길 때까지 돈다.
다만 여기서 빠진 것이 핵심이다.
사람이 트리거다.
탭을 닫으면 사라지고, 스케줄도 없으며, 이메일이 도착했을 때 깨어나는 기능도 없다.
스스로 돌고 실제 이벤트로 트리거되며 사람이 지켜보지 않아도 되는 루프를 얻으려면, 보통 도구·호스팅·코드·게이트·청구서가 있는 무거운 세계로 들어가야 한다.

## 실생활을 위한 루프, Mira

코드와 비용을 걷어내면 남는 것은 단순하고 유용한 개념 하나다.
스케줄에 따라 또는 무언가 일어나는 순간 스스로 돌아가며, 사람이 기억하거나 그 자리에 있을 필요가 없는 작업이다.
글에서는 이를 위한 무료 옵션으로 Mira를 소개한다.
Mira는 Telegram 안에서 동작하며, 평범한 말로 설명하면 루프를 만들어주고, 그 루프를 Skill이라 부른다.

```
▸ SKILL
"Every weekday at 7am, check my Gmail and Google Calendar.
Send me a short brief: my 3 most important meetings, anything
urgent in the inbox, and one thing I said I'd follow up on but
haven't. Keep it under 120 words."
```

핵심은 Mira가 더 똑똑한 챗봇이 아니라는 점이다.
ChatGPT가 답한다면 Mira는 행동한다.
이메일을 쓰라고 하는 게 아니라 보내라고 하고, 티켓 초안이 아니라 담당자가 배정된 실제 Linear 티켓을 받는다.
Mira는 Composio를 통해 500개 이상의 앱(Notion, Gmail, Google Calendar, GitHub, Figma, Stripe 등)에 연결되고, 세션과 그룹 채팅을 넘어 유지되는 장기 기억을 가지며, 작업에 따라 GPT·Claude·Gemini를 골라 쓰는 모델 비종속(model-agnostic) 구조다.
업무 브리핑, 콘텐츠 생성, 음성 입력 처리, 운동 스트릭 관리나 항공권 가격 추적 같은 실생활 작업이 모두 한 메시지로 도는 루프가 된다.

## 결론

루프는 트렌드가 아니라 누가 일을 하는가의 전환이다.
AI가 모든 단계를 밀어주기를 기다리는 대신 작업 전체를 스스로 돌리기 시작한다.
다만 글의 결론은 신중하다.
무거운 루프는 반복되고, 자동 거부가 가능하며, 에이전트가 끝까지 처리할 수 있고, 완료가 객관적일 때만 가치가 있다.
대부분의 일상 작업에는 이미 있는 무료 도구로 충분하며, 그것이 부족하다고 실제로 느낄 때 비로소 무거운 버전을 고민하는 것이 합리적이다.

## Reference

- [Loops explained: Claude, GPT, Mira and what actually works](https://x.com/AnatoliKopadze/status/2068328135611822149)
