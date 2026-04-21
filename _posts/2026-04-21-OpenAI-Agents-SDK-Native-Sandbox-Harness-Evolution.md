---
layout: post
title: "OpenAI Agents SDK 진화 - 네이티브 샌드박스와 모델 네이티브 하네스"
author: 'Juho'
date: 2026-04-21 00:00:00 +0900
categories: [AI]
tags: [Agent, OpenAI, AI]
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
2. [새로운 하네스의 역할](#새로운-하네스의-역할)
   - [에이전트 루프와 프리미티브](#에이전트-루프와-프리미티브)
   - [코드 예시](#코드-예시)
3. [네이티브 샌드박스 실행](#네이티브-샌드박스-실행)
   - [지원 프로바이더](#지원-프로바이더)
   - [Manifest 추상화](#manifest-추상화)
4. [하네스와 컴퓨트의 분리](#하네스와-컴퓨트의-분리)
5. [가격과 가용성](#가격과-가용성)
6. [의미와 시사점](#의미와-시사점)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

OpenAI가 Agents SDK에 새로운 기능을 도입했다.
표준화된 인프라를 제공해 개발자가 쉽게 시작할 수 있으면서도 OpenAI 모델에 올바르게 최적화된 구성을 목표로 한다.

핵심은 두 가지다.
첫째, 파일과 도구를 넘나들며 작업할 수 있는 모델 네이티브 하네스.
둘째, 그 작업을 안전하게 실행하기 위한 네이티브 샌드박스 실행 환경이다.

기존 옵션은 각자 트레이드오프가 명확했다.
모델 중립 프레임워크는 유연하지만 프론티어 모델의 역량을 충분히 활용하지 못했다.
모델 제공자 SDK는 모델에 가깝지만 하네스 가시성이 부족했다.
매니지드 에이전트 API는 배포를 단순화하지만 실행 위치와 민감 데이터 접근을 제약했다.

## 새로운 하네스의 역할

### 에이전트 루프와 프리미티브

업데이트된 하네스는 문서, 파일, 시스템을 다루는 에이전트에 더 강한 역량을 제공한다.
설정 가능한 메모리, 샌드박스 인식 오케스트레이션, Codex 유사 파일시스템 도구, 프론티어 에이전트 시스템에서 공통화되는 프리미티브 통합을 갖춘다.

프리미티브에는 다음이 포함된다.
MCP를 통한 도구 사용, 스킬을 통한 점진적 공개, AGENTS.md를 통한 커스텀 지침, 셸 도구를 통한 코드 실행, apply patch 도구를 통한 파일 편집이다.
하네스는 시간이 지나며 새로운 에이전틱 패턴과 프리미티브를 계속 통합할 예정이다.

### 코드 예시

개발자는 에이전트에 통제된 워크스페이스, 명시적 지침, 필요한 도구를 부여할 수 있다.

```python
# pip install "openai-agents>=0.14.0"

import asyncio
import tempfile
from pathlib import Path

from agents import Runner
from agents.run import RunConfig
from agents.sandbox import Manifest, SandboxAgent, SandboxRunConfig
from agents.sandbox.entries import LocalDir
from agents.sandbox.sandboxes import UnixLocalSandboxClient


async def main() -> None:
    with tempfile.TemporaryDirectory() as tmp:
        dataroom = Path(tmp) / "dataroom"
        dataroom.mkdir()
        (dataroom / "metrics.md").write_text(
            """# Annual metrics

| Year | Revenue | Operating income | Operating cash flow |
| --- | ---: | ---: | ---: |
| FY2025 | $124.3M | $18.6M | $24.1M |
| FY2024 | $98.7M | $12.4M | $17.9M |
""",
            encoding="utf-8",
        )

        agent = SandboxAgent(
            name="Dataroom Analyst",
            model="gpt-5.4",
            instructions="Answer using only files in data/. Cite source filenames.",
            default_manifest=Manifest(entries={"data": LocalDir(src=dataroom)}),
        )

        result = await Runner.run(
            agent,
            "Compare FY2025 revenue, operating income, and operating cash flow with FY2024.",
            run_config=RunConfig(
                sandbox=SandboxRunConfig(client=UnixLocalSandboxClient()),
            ),
        )
        print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

`SandboxAgent`는 모델, 지침, 기본 manifest를 받는다.
`Runner.run`은 `SandboxRunConfig`와 함께 호출되어 샌드박스 안에서 파일을 읽고 작업을 수행한다.

## 네이티브 샌드박스 실행

### 지원 프로바이더

업데이트된 Agents SDK는 샌드박스 실행을 네이티브로 지원한다.
에이전트가 필요한 파일, 도구, 의존성을 갖춘 통제된 컴퓨터 환경에서 실행될 수 있다.

자체 샌드박스를 가져올 수 있고, 빌트인 지원도 제공된다.

| Provider | 비고 |
|----------|------|
| Blaxel | 빌트인 |
| Cloudflare | 빌트인 |
| Daytona | 빌트인 |
| E2B | 빌트인 |
| Modal | 빌트인 |
| Runloop | 빌트인 |
| Vercel | 빌트인 |

### Manifest 추상화

프로바이더 간 이식성을 위해 SDK는 Manifest 추상화를 도입한다.
Manifest는 에이전트의 워크스페이스를 기술하는 단위다.

로컬 파일 마운트, 출력 디렉터리 정의, 스토리지 프로바이더로부터의 데이터 반입이 가능하다.
지원되는 스토리지는 AWS S3, Google Cloud Storage, Azure Blob Storage, Cloudflare R2다.

로컬 프로토타입에서 프로덕션 배포까지 에이전트 환경을 일관된 방식으로 구성할 수 있다.
모델 입장에서도 입력 위치, 출력 위치, 장시간 태스크에서의 작업 정리 방식이 예측 가능해진다.

## 하네스와 컴퓨트의 분리

에이전트 시스템은 프롬프트 인젝션과 데이터 탈취 시도를 전제로 설계되어야 한다.
하네스와 컴퓨트를 분리하면 모델 생성 코드가 실행되는 환경에 크리덴셜이 노출되지 않는다.

두 번째 이점은 내구성 있는 실행이다.
에이전트 상태가 외부화되어 있으면 샌드박스 컨테이너를 잃어도 실행이 사라지지 않는다.
빌트인 스냅샷과 리하이드레이션을 통해 원래 환경이 실패하거나 만료되더라도 새 컨테이너에서 마지막 체크포인트부터 이어갈 수 있다.

세 번째 이점은 확장성이다.
에이전트 실행은 단일 샌드박스 또는 다수 샌드박스를 사용하거나, 필요 시에만 샌드박스를 호출하거나, 서브에이전트를 격리 환경으로 라우팅하거나, 컨테이너 간 병렬화로 실행을 가속할 수 있다.

## 가격과 가용성

새로운 Agents SDK 기능은 모든 고객에게 API를 통해 일반 제공된다.
요금은 표준 API 가격 정책을 따르며 토큰과 도구 사용량 기반이다.

새 하네스와 샌드박스 기능은 Python으로 먼저 출시되며, TypeScript 지원은 추후 릴리즈 예정이다.
코드 모드와 서브에이전트를 포함한 추가 역량도 Python과 TypeScript 양쪽으로 확장 중이다.

## 의미와 시사점

이번 업데이트는 "좋은 모델"만으로는 프로덕션 에이전트가 완성되지 않는다는 업계의 공감대를 제도화한 릴리즈다.
Oscar Health는 이 SDK로 임상 기록 워크플로우 자동화를 프로덕션 가능 수준으로 끌어올렸다고 밝혔다.
Staff Engineer Rachael Burns에 따르면 차이는 메타데이터 추출 정확도만이 아니라 "길고 복잡한 기록에서 각 encounter의 경계를 올바르게 이해하는 것"이었다.

하네스/컴퓨트 분리, Manifest 기반 워크스페이스 표준화, 다중 샌드박스 프로바이더 지원은 동일한 방향성을 가리킨다.
에이전트가 프로토타입에서 프로덕션으로 넘어갈 때 필요한 인프라가 SDK 레벨에서 표준화되고 있다.

## 결론

OpenAI Agents SDK는 모델 네이티브 하네스와 네이티브 샌드박스 실행을 도입하며 프로덕션 에이전트 인프라의 표준을 제시한다.
Manifest 추상화로 로컬-프로덕션 이식성을 확보하고, 하네스/컴퓨트 분리로 보안·내구성·확장성을 동시에 얻는다.
Python 우선 지원에서 출발해 TypeScript, 코드 모드, 서브에이전트 등으로 확장될 예정이므로 에이전트 시스템 설계자는 로드맵을 주시할 필요가 있다.

## Reference

- [The next evolution of the Agents SDK](https://openai.com/index/the-next-evolution-of-the-agents-sdk/)
