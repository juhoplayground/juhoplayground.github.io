---
layout: post
title: "Hermes Agent Self-Evolution: DSPy+GEPA로 에이전트 스킬·프롬프트를 자동 진화시키는 파이프라인"
author: 'Juho'
date: 2026-06-14 00:00:00 +0900
categories: [AI]
tags: [Agent, AI, LLM]
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
2. [핵심 엔진](#핵심-엔진)
   - [DSPy + GEPA](#dspy--gepa)
   - [Darwinian Evolver](#darwinian-evolver)
   - [DSPy MIPROv2](#dspy-miprov2)
3. [최적화 대상과 단계](#최적화-대상과-단계)
   - [Phase 1: 스킬 파일 진화](#phase-1-스킬-파일-진화)
   - [Phase 2: 툴 설명 최적화](#phase-2-툴-설명-최적화)
   - [Phase 3: 시스템 프롬프트 진화](#phase-3-시스템-프롬프트-진화)
   - [Phase 4: 코드 진화](#phase-4-코드-진화)
   - [Phase 5: 지속적 자기개선 루프](#phase-5-지속적-자기개선-루프)
4. [아키텍처](#아키텍처)
   - [최적화 루프](#최적화-루프)
   - [평가 데이터 전략](#평가-데이터-전략)
5. [가드레일과 제약 조건](#가드레일과-제약-조건)
6. [사용 방법](#사용-방법)
7. [결론](#결론)
8. [Reference](#reference)

## 개요

[Hermes Agent Self-Evolution](https://github.com/NousResearch/hermes-agent-self-evolution){:target="_blank"}은 Nous Research가 공개한 독립 최적화 파이프라인으로, [Hermes Agent](https://github.com/NousResearch/hermes-agent){:target="_blank"}의 스킬 파일·툴 설명·시스템 프롬프트·구현 코드를 자동으로 진화시켜 성능을 향상시킨다.
핵심 특징은 GPU 훈련 없이 API 호출만으로 동작한다는 점이다.
모델 가중치를 변경하는 대신 프롬프트 텍스트, 지시문, few-shot 예시 같은 문자열을 변이·평가·선택하는 방식을 사용한다.
1회 최적화 실행 비용은 약 $2~10 수준이다.
이 저장소는 hermes-agent 내부에 포함되지 않고, hermes-agent 레포지터리 위에서 동작하는 외부 파이프라인으로 설계되었다.

## 핵심 엔진

세 가지 엔진이 최적화 대상과 성격에 따라 역할을 분담한다.

### DSPy + GEPA

GEPA(Genetic-Pareto Prompt Evolution)는 DSPy에 통합된 반사적 프롬프트 진화 엔진이다.
단순히 실패 여부를 기록하는 것이 아니라 실행 트레이스를 읽어 실패 원인을 파악한 뒤 목표한 수정안을 제안한다.
3개 이상의 예시만 있어도 동작하며, RL 및 이전 DSPy 옵티마이저 대비 우수한 성능을 보인다.
ICLR 2026 Oral 논문 기반의 엔진이다.
스킬 파일(SKILL.md), 툴 설명, 시스템 프롬프트 섹션에 적용되는 1차 엔진이다.
라이선스는 MIT이다.

### Darwinian Evolver

Git 기반 유기체(GitBasedOrganism) 개념으로 코드 파일 자체를 진화시키는 엔진이다.
pytest와 벤치마크 스코어를 피트니스 함수로 사용하며, 코드 변이 결과는 모두 git 브랜치에 커밋되어 추적 가능하다.
라이선스가 AGPL v3이므로 Python 임포트 대신 외부 CLI 방식으로만 호출한다.
Phase 4의 코드 진화에 사용된다.

### DSPy MIPROv2

베이지안 최적화 기반의 few-shot 예시 및 지시문 텍스트 최적화 엔진이다.
GEPA가 주 엔진이고, MIPROv2는 대체(fallback) 옵티마이저로 활용된다.
라이선스는 MIT이다.

## 최적화 대상과 단계

최적화는 5개 단계로 순차 진행된다.
각 단계 완료 후 벤치마크 게이트를 통과해야 다음 단계로 넘어간다.

### Phase 1: 스킬 파일 진화

SKILL.md 파일은 에이전트가 따르는 절차 지시문이다.
스킬 텍스트를 DSPy 모듈로 래핑하고, 테스트 태스크에서 실행한 결과를 기반으로 GEPA가 개선 변형을 제안한다.
대상 스킬 예시로는 `github-code-review`, `systematic-debugging`, `arxiv` 등이 있다.
평가 데이터셋은 스킬당 15~30개 예시로 구성한다.
1개 이상의 스킬에서 평가 점수 10% 이상 향상, 벤치마크 회귀 없음(TBLite 2% 이내)이 Phase 1 완료 조건이다.

### Phase 2: 툴 설명 최적화

툴 스키마의 `description` 필드 — 에이전트가 어떤 툴을 선택할지 결정할 때 참조하는 텍스트를 최적화한다.
툴 선택은 분류 문제이므로 DSPy 최적화에 적합하다.
예를 들어 `search_files`와 `terminal(grep)` 중 적절한 툴을 더 정확히 선택하도록 설명 문구를 개선하는 것이 목표다.
툴 설명은 API 호출마다 전송되므로 글자 수 제한이 엄격하다(툴당 최대 500자).
한 툴의 설명 개선이 다른 툴의 선택률을 저하시키지 않도록 모든 툴 설명을 함께 평가하는 크로스-툴 평가 방식을 사용한다.

### Phase 3: 시스템 프롬프트 진화

`agent/prompt_builder.py`가 조립하는 시스템 프롬프트의 섹션 단위를 최적화한다.
진화 가능한 섹션은 `DEFAULT_AGENT_IDENTITY`, `MEMORY_GUIDANCE`, `SESSION_SEARCH_GUIDANCE`, `SKILLS_GUIDANCE`, `PLATFORM_HINTS` 다섯 가지다.
사용자 실제 기억(memory block), 자동 생성 스킬 인덱스, 프로젝트별 컨텍스트 파일(AGENTS.md 등)은 최적화 대상에서 제외된다.
시스템 프롬프트 변경은 영향 범위가 넓으므로 가장 엄격한 벤치마크 검증(TBLite + YC-Bench)이 요구된다.
각 섹션의 현재 크기 대비 20% 초과 확장이 금지되며, 프롬프트 캐싱 경계를 유지해야 한다.

### Phase 4: 코드 진화

`tools/*.py` 실제 파이썬 소스 코드를 Darwinian Evolver로 진화시킨다.
GitHub 이슈로 보고된 알려진 버그에 대한 재현 스크립트를 생성하고, 진화를 통해 수정안을 탐색한다.
피트니스 함수는 pytest 결과, 벤치마크 점수, 버그 재현 해소 여부의 복합 점수로 구성된다.
함수 시그니처와 `registry.register()` 호출은 변경이 금지된다.
2550개 이상의 전체 테스트 슈트 통과가 필수 조건이며, 모든 코드 변경에는 인간 리뷰가 필요하다.

### Phase 5: 지속적 자기개선 루프

Phase 1~4의 수동 최적화가 검증된 뒤, 이를 자동화하는 단계다.
성능 모니터가 스킬별 성공률, 툴 선택 정확도, 벤치마크 점수를 실시간 추적한다.
자동 트리아지 로직이 개선 잠재력과 사용 빈도를 곱한 값으로 최적화 대상의 우선순위를 결정한다.
Hermes cron 스케줄러를 통해 주간 벤치마크 실행과 임계 초과 시 GEPA 자동 트리거가 설정된다.
자동화는 감지와 최적화 단계만 대상이며, 배포(PR 병합)는 여전히 인간이 승인한다.

## 아키텍처

### 최적화 루프

최적화는 6단계 루프로 진행된다.

1단계: 최적화 대상 선정 — 스킬, 프롬프트 섹션, 툴 중 하나를 선택하고 현재 버전을 기준선으로 로드한다.
2단계: 평가 데이터셋 구축 — SessionDB에서 실제 사용 예시를 마이닝하거나 수작업 테스트 케이스를 준비하고, 훈련/검증/테스트 세트로 분할한다.
3단계: DSPy 모듈 래핑 — 스킬 텍스트는 `dspy.Signature`로, 에이전트 워크플로는 `dspy.ReAct`로, 툴 선택은 `dspy.Predict`로 각각 변환한다.
4단계: 옵티마이저 실행 — GEPA(1차), MIPROv2(대체), Darwinian Evolver(코드) 중 적합한 엔진을 선택해 최적화를 수행한다.
5단계: 평가 및 비교 — 홀드아웃 테스트 세트에서 최적화 버전의 정확도·비용·지연 시간을 기준선과 비교한다.
6단계: 배포(승인 후) — 개선된 버전을 git 커밋하고 PR을 생성하며, 필요 시 롤백은 `git revert`로 수행한다.

### 평가 데이터 전략

평가 데이터는 네 가지 소스에서 수집된다.

소스 A는 합성 생성 데이터다.
강력한 모델(예: Claude Opus)이 스킬 파일을 읽고 15~30개의 현실적인 (입력 태스크, 기대 동작) 쌍을 생성한다.
기대 동작은 정확한 텍스트가 아닌 루브릭(rubric) 형태로 표현된다.

소스 B는 SessionDB 마이닝이다.
실제 대화에서 스킬이 로드된 세션을 추출하고, LLM-as-judge로 각 (태스크, 응답) 쌍을 채점한다.
고득점 쌍은 긍정 예시로, 저득점 쌍은 GEPA의 반사적 분석을 위한 실패 케이스로 활용된다.

소스 C는 수작업 황금 데이터셋이다.
중요 스킬에 대해 수동으로 작성된 테스트 케이스를 JSONL 파일로 저장한다.
품질이 가장 높지만 수작업 비용이 크므로 핵심 스킬에만 적용한다.

소스 D는 스킬별 자동 평가다.
`systematic-debugging`은 버그를 심어 수정 여부를 확인하고, `arxiv`는 알려진 논문을 검색하여 결과를 검증하는 방식처럼, 자동 평가가 가능한 스킬에 선택적으로 적용한다.

## 가드레일과 제약 조건

모든 진화 변형은 다섯 가지 제약 조건을 모두 통과해야 유효하다.

첫째, 전체 테스트 슈트 통과다.
`pytest tests/ -q` 가 100% 통과해야 하며, 하나라도 실패하면 즉시 거부된다.

둘째, 텍스트 크기 제한이다.
스킬 파일은 기본 15KB 이하, 툴 설명은 500자 이하, 시스템 프롬프트 섹션은 현재 크기 대비 20% 이내로 유지해야 한다.
피트니스 함수에 길이 페널티가 포함되어 장황한 변형이 선택될 가능성을 억제한다.

셋째, 프롬프트 캐싱 호환성이다.
진화된 콘텐츠는 활성 대화에 핫스왑되지 않는다.
모든 변경은 다음 새 세션에서만 적용된다.

넷째, 의미 보존이다.
'GitHub 코드 리뷰' 스킬은 여전히 코드 리뷰를 수행해야 한다.
피트니스 함수에 의미 유사도 검사가 포함되어 목적 이탈을 방지한다.

다섯째, PR을 통한 배포다.
모든 진화 결과는 `evolve/<대상>-<타임스탬프>` 브랜치에 커밋되고 PR로 제출된다.
PR 본문에는 훈련·검증·홀드아웃 세트의 전후 점수, 변경 diff, 최적화 비용, 거부된 변형의 제약 위반 내역이 포함된다.
직접 커밋은 어떤 경우에도 허용되지 않는다.

## 사용 방법

저장소 클론 및 설치 후 `HERMES_AGENT_REPO` 환경 변수로 hermes-agent 경로를 지정한다.

```bash
git clone https://github.com/NousResearch/hermes-agent-self-evolution.git
cd hermes-agent-self-evolution
pip install -e ".[dev]"

export HERMES_AGENT_REPO=~/.hermes/hermes-agent
```

스킬 진화는 `--eval-source` 옵션으로 평가 데이터 소스를 선택한다.
`synthetic`은 합성 데이터, `sessiondb`는 실제 세션 기록을 사용한다.

```bash
# 합성 평가 데이터로 스킬 진화
python -m evolution.skills.evolve_skill \
    --skill github-code-review \
    --iterations 10 \
    --eval-source synthetic

# 실제 세션 기록 기반 스킬 진화
python -m evolution.skills.evolve_skill \
    --skill github-code-review \
    --iterations 10 \
    --eval-source sessiondb
```

툴 설명, 시스템 프롬프트, 코드 진화는 각각 별도 모듈 명령으로 실행한다.

```bash
# 툴 설명 진화 (Phase 2)
python -m evolution.tools.evolve_tool_descriptions \
    --iterations 5 \
    --benchmark-gate tblite-fast

# 시스템 프롬프트 섹션 진화 (Phase 3)
python -m evolution.prompts.evolve_prompt_section \
    --section MEMORY_GUIDANCE \
    --iterations 5

# 코드 진화 (Phase 4)
python -m evolution.code.evolve_tool_code \
    --tool file_tools \
    --bug-issue 742 \
    --iterations 10
```

모든 명령은 hermes-agent 레포지터리에 대한 PR 브랜치와 요약 결과를 출력한다.
병합은 인간 리뷰어가 결정한다.

저장소 디렉토리 구조는 다음과 같다.

```
hermes-agent-self-evolution/
├── PLAN.md
├── README.md
├── pyproject.toml
└── evolution/
    ├── core/           # 공유 인프라
    │   ├── dataset_builder.py
    │   ├── fitness.py
    │   ├── constraints.py
    │   ├── benchmark_gate.py
    │   └── pr_builder.py
    ├── skills/         # Phase 1
    ├── tools/          # Phase 2
    ├── prompts/        # Phase 3
    ├── code/           # Phase 4
    └── monitor/        # Phase 5
```

## 결론

Hermes Agent Self-Evolution은 에이전트 스킬·툴 설명·시스템 프롬프트·구현 코드를 GPU 없이 API 호출만으로 자동 진화시키는 파이프라인이다.
GEPA의 반사적 트레이스 분석, 다층 가드레일, PR 기반 인간 검토 체계를 결합해 안전하게 성능을 개선하는 구조를 취한다.
5개 단계는 순차적으로 검증되며, 각 단계의 벤치마크 게이트를 통과해야 다음 단계가 시작된다.
Phase 1의 스킬 진화는 이미 구현 완료 상태이고, 나머지 단계는 로드맵에 따라 순차 진행 예정이다.
자기진화 에이전트 시스템에 관심 있는 개발자라면 저장소의 PLAN.md를 함께 확인하면 전체 설계를 파악하는 데 도움이 된다.

## Reference

- [NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution/)
- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent/)
- [DSPy](https://github.com/stanfordnlp/dspy/)
- [GEPA](https://github.com/gepa-ai/gepa/)
- [Darwinian Evolver](https://github.com/imbue-ai/darwinian_evolver/)
