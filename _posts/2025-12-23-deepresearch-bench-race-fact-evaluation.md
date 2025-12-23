---
layout: post
title: DeepResearch Bench의 RACE와 FACT 평가 방법
author: 'Juho'
date: 2025-12-23 14:00:00 +0900
categories: [LLM]
tags: [llm, research-agent, evaluation, benchmark, deep-research, race, fact, citation]
pin: True
toc: True
---

## 들어가며

LLM 기반 연구 에이전트(Deep Research Agents, DRAs)의 성능을 어떻게 평가할 수 있을까요?  
단순히 텍스트 생성 품질만으로는 충분하지 않습니다.  
연구 에이전트는 웹 탐색, 정보 수집, 분석, 그리고 인용이 포함된 종합 보고서 작성까지 수행해야 합니다.  

DeepResearch Bench는 이러한 복잡한 연구 에이전트를 평가하기 위해 개발된 포괄적인 벤치마크입니다.  
물리학, 화학부터 금융, 마케팅, 엔터테인먼트까지 22개 학문 및 전문 분야에 걸쳐 박사 수준의 100개 연구 과제를 포함하고 있으며 96,147개의 실제 사용자 쿼리 분석을 바탕으로 제작되었습니다.  

이 글에서는 DeepResearch Bench의 핵심 평가 방법론인 RACE와 FACT에 대해 상세히 알아보겠습니다.

## 목차
1. [DeepResearch Bench의 특징](#deepresearch-bench의-특징)
2. [RACE: 참조 기반 적응형 기준 평가](#race-참조-기반-적응형-기준-평가)
3. [FACT: 사실성과 인용 신뢰도 평가](#fact-사실성과-인용-신뢰도-평가)
4. [RACE + FACT: 종합 평가](#race--fact-종합-평가)
5. [벤치마크 결과 및 분석](#벤치마크-결과-및-분석)
6. [실전 팁: 높은 RACE/FACT 점수 받기](#실전-팁-높은-racefact-점수-받기)
7. [한계점](#한계점)
8. [결론](#결론)

## DeepResearch Bench의 특징

### 벤치마크 구성

DeepResearch Bench는 실제 사용자의 연구 니즈를 반영하기 위해 철저한 데이터 분석을 거쳐 제작되었습니다:

데이터 수집 및 큐레이션:
- 96,147개의 실제 사용자 쿼리 분석
- 연구 목적의 복잡한 질문 필터링
- 도메인 전문가의 검토를 통한 과제 정제
- 최종 100개의 PhD 수준 연구 과제 선정

언어 분포:
- 중국어 과제: 50개
- 영어 과제: 50개
- 균형 잡힌 주제 분포 유지
- 다국어 연구 에이전트 평가 지원

학문 분야 분포 (22개 분야):

```
과학 분야:
- 물리학 (Physics)
- 화학 (Chemistry)
- 생물학 (Biology)
- 수학 (Mathematics)
- 컴퓨터 과학 (Computer Science)

공학 분야:
- 전기공학 (Electrical Engineering)
- 기계공학 (Mechanical Engineering)
- 재료공학 (Materials Science)

사회과학:
- 경제학 (Economics)
- 심리학 (Psychology)
- 사회학 (Sociology)
- 정치학 (Political Science)

비즈니스 및 실무:
- 금융 (Finance)
- 마케팅 (Marketing)
- 경영학 (Management)

인문학 및 예술:
- 역사 (History)
- 문학 (Literature)
- 철학 (Philosophy)

기타 전문 분야:
- 의학 (Medicine)
- 법학 (Law)
- 환경과학 (Environmental Science)
- 엔터테인먼트 (Entertainment)
```

### 품질 보증

도메인 전문가 검증:
- 각 분야 전문가가 과제 설계 및 검토
- 참조 답변 작성 (RACE 평가의 기준)
- 난이도와 진정성(authenticity) 검증

과제 특성:
- 단순 검색으로 답할 수 없는 복잡성
- 다단계 추론 및 정보 종합 요구
- 실제 연구 시나리오 반영
- 명확한 평가 기준 설정 가능성

## RACE: 참조 기반 적응형 기준 평가

### RACE란?

RACE (Reference-based Adaptive Criteria-driven Evaluation)는 생성된 연구 보고서의 품질을 체계적으로 평가하는 방법론입니다.  정적인 평가 기준을 사용하는 대신, 각 연구 과제의 특성에 맞춘 동적 기준을 생성하여 평가합니다.  

### 핵심 원리

RACE의 핵심은 세 가지 핵심 메커니즘으로 구성됩니다:

#### 1. 동적 기준 생성 (Dynamic Criteria Generation)

RACE는 각 연구 과제에 대해 자동으로 평가 기준을 생성합니다. 이는 다음과 같은 장점을 제공합니다:

- 과제별 맞춤화: 물리학 연구와 마케팅 분석은 서로 다른 평가 기준이 필요합니다  
- 유연성: 정적 루브릭(rubric)에 얽매이지 않고 과제의 특성을 반영합니다  
- 공정성: 모든 과제가 자신의 맥락에서 평가받습니다  

예를 들어, "양자 컴퓨팅의 최신 동향"에 대한 연구는 기술적 정확성과 최신성이 중요하지만  
"소셜 미디어 마케팅 전략"에 대한 연구는 실용성과 창의성이 더 중요할 수 있습니다.  

#### 2. 4가지 평가 차원

RACE는 모든 연구 보고서를 다음 4가지 차원에서 평가합니다:

1) Comprehensiveness (포괄성)

주제에 대한 폭넓고 깊이 있는 커버리지를 측정합니다:
- 핵심 개념이 모두 다뤄졌는가?
- 관련된 하위 주제들이 충분히 탐구되었는가?
- 다양한 관점과 접근법이 포함되었는가?

```
예시: "인공지능 윤리"에 대한 연구라면
✓ 편향성, 프라이버시, 투명성, 책임성 등 주요 주제 포함
✓ 각 주제별 구체적 사례와 논의
✗ 한두 가지 주제만 다루고 다른 중요 영역 누락
```

2) Insight/Depth (통찰력/깊이)

분석의 질적 수준과 지적 엄밀성을 평가합니다:
- 단순한 사실 나열을 넘어 심층 분석이 있는가?
- 새로운 통찰이나 관점을 제시하는가?
- 비판적 사고와 종합이 드러나는가?

```
표면적 분석: "AI는 많은 산업에서 사용되고 있다"
깊이 있는 분석: "AI 도입은 제조업에서 30% 효율 증대를 가져왔으나,
                노동력 재배치라는 사회적 과제를 동반한다.
                이는 기술-사회 공진화 이론의 전형적 사례다"
```

3) Instruction-Following (지시사항 준수)

주어진 연구 과제의 요구사항과 제약조건을 얼마나 충실히 따랐는지 평가합니다:
- 명시된 연구 질문에 답했는가?
- 요청된 형식과 구조를 따랐는가?
- 특정 제약사항(예: 특정 기간, 특정 지역)을 준수했는가?

```
과제: "2020-2024년 한국의 전기차 시장 동향을 5000자 이내로 분석"

준수: ✓ 해당 기간 데이터 사용
      ✓ 한국 시장에 집중
      ✓ 4800자로 작성

미준수: ✗ 글로벌 시장 포함
        ✗ 7000자로 작성
```

4) Readability (가독성)

보고서의 명확성, 논리적 구성, 효과적 전달을 평가합니다:
- 논리적 흐름과 구조가 명확한가?
- 전문 용어가 적절히 설명되었는가?
- 문장이 명확하고 이해하기 쉬운가?
- 시각 자료나 구조화가 효과적인가?

```
좋은 가독성:
1. 명확한 섹션 구분
2. 단계별 논리 전개
3. 핵심 개념 정의
4. 예시와 설명 병행

나쁜 가독성:
- 긴 문단의 나열
- 갑작스러운 주제 전환
- 설명 없는 전문 용어 남발
```

#### 3. 참조 기반 점수 매기기 (Reference-Based Scoring)

RACE의 가장 혁신적인 부분은 고품질 참조 보고서와의 비교를 통한 평가입니다:

- 벤치마크 설정: 각 과제에 대해 도메인 전문가가 작성한 고품질 참조 보고서 사용
- 상대적 평가: 절대적 기준이 아닌 참조 보고서와의 비교를 통한 차별화
- 일관성: 모든 평가가 동일한 기준점을 참조하여 일관성 확보

```python
# 개념적 평가 프로세스 (의사코드)
def race_evaluate(generated_report, reference_report, task):
    # 1. 동적 기준 생성
    criteria = generate_adaptive_criteria(task)

    # 2. 4개 차원 평가 (참조 보고서와 비교)
    scores = {
        'comprehensiveness': compare_coverage(generated, reference, criteria),
        'insight_depth': compare_depth(generated, reference, criteria),
        'instruction_following': check_requirements(generated, task),
        'readability': assess_clarity(generated, criteria)
    }

    # 3. 가중치 적용
    weights = calculate_task_weights(task)
    final_score = weighted_average(scores, weights)

    return final_score
```

#### 4. 가중 평가 (Weighted Assessment)

모든 차원이 동등하게 중요한 것은 아닙니다. RACE는 과제의 특성에 따라 가중치를 조정합니다:

```
학술 논문 리뷰 과제:
- Comprehensiveness: 30%
- Insight/Depth: 40% (높은 가중치)
- Instruction-Following: 20%
- Readability: 10%

실무 가이드 작성 과제:
- Comprehensiveness: 25%
- Insight/Depth: 20%
- Instruction-Following: 25%
- Readability: 30% (높은 가중치)
```

### RACE의 장점

1. 단일 메트릭 최적화 방지: 4개 차원 평가로 편향된 최적화 방지
2. 과제별 맞춤 평가: 동적 기준으로 다양한 연구 유형 공정 평가
3. 고품질 벤치마크: 참조 보고서 사용으로 명확한 품질 기준 제시
4. 해석 가능성: 각 차원별 점수로 구체적 피드백 제공

### RACE의 기술적 구현

DeepResearch Bench는 LLM 기반 평가를 사용하여 RACE 점수를 계산합니다:

평가 모델:
- 기본 모델: Google Gemini API
- 대안 모델: 사용자 정의 가능 (AIClient 클래스 수정)
- 0-1 정규화 점수 체계

평가 프롬프트 구조:
```python
evaluation_prompt = f"""
과제: {task_description}

참조 보고서 (전문가 작성):
{reference_report}

생성된 보고서 (평가 대상):
{generated_report}

다음 기준에 따라 0-1 점수를 부여하세요:

1. Comprehensiveness (포괄성):
   - 참조 보고서와 비교하여 주제 커버리지 평가
   - 핵심 개념과 하위 주제 포함 여부
   - 점수: 0.0 - 1.0

2. Insight/Depth (통찰력):
   - 분석의 깊이와 새로운 통찰 제시
   - 비판적 사고와 종합 능력
   - 점수: 0.0 - 1.0

3. Instruction-Following (지시사항 준수):
   - 과제 요구사항 충족도
   - 형식 및 제약조건 준수
   - 점수: 0.0 - 1.0

4. Readability (가독성):
   - 논리적 구조와 명확성
   - 전달 효과성
   - 점수: 0.0 - 1.0

가중치: {task_weights}
최종 점수: 가중 평균 계산
"""
```

평가 프로세스:
1. 과제별 동적 가중치 생성
2. LLM을 사용한 4개 차원 독립 평가
3. 가중 평균으로 최종 점수 산출
4. 통계적 유의성 검증

## FACT: 사실성과 인용 신뢰도 평가

### FACT란?

FACT (Framework for Factual Abundance and Citation Trustworthiness)는 연구 보고서의 정보 수집 능력과 인용의 신뢰성을 평가하는 프레임워크입니다.  

연구 보고서는 단순히 잘 쓰여진 것만으로는 부족합니다.
제시된 사실이 정확해야 하고, 인용된 출처가 실제로 그 주장을 뒷받침해야 합니다.  
FACT는 바로 이 부분을 검증합니다.  

### FACT의 핵심 프로세스

#### 1. 자동 주장 추출 (Automatic Claim Extraction)

FACT는 생성된 보고서에서 사실적 주장(factual claims)과 해당 출처를 자동으로 추출합니다:

```markdown
보고서 예시:
"2023년 글로벌 AI 시장 규모는 1,500억 달러에 달했으며[1],
2030년까지 연평균 37% 성장할 것으로 예상된다[2]."

추출 결과:
Claim 1: "2023년 글로벌 AI 시장 규모는 1,500억 달러"
Source 1: [1] https://example.com/ai-market-report-2023

Claim 2: "2030년까지 연평균 37% 성장 예상"
Source 2: [2] https://example.com/ai-growth-forecast
```

중복 제거 과정:
- 동일한 주장-URL 쌍 제거
- 실질적으로 다른 주장에 집중
- 효율적인 검증을 위한 최적화

```python
# 개념적 추출 프로세스 (의사코드)
def extract_claims_and_sources(report):
    claims = []

    # LLM을 사용한 주장 추출
    raw_claims = llm_extract_factual_statements(report)

    # 인용 매칭
    for claim in raw_claims:
        citation = extract_citation(claim)
        url = resolve_citation_to_url(citation, report)
        claims.append({
            'statement': claim,
            'citation': citation,
            'url': url
        })

    # 중복 제거
    unique_claims = deduplicate(claims)

    return unique_claims
```

#### 2. 인용 검증 (Citation Verification)

추출된 각 주장에 대해 FACT는 실제로 인용된 출처가 그 주장을 뒷받침하는지 검증합니다:

검증 프로세스:

1. 웹 스크래핑: 인용된 URL의 콘텐츠 가져오기
2. 관련성 판단: LLM을 사용하여 출처 내용이 주장을 지지하는지 판단
3. 검증 결과: 지지함/지지하지 않음/판단 불가

```python
# 개념적 검증 프로세스 (의사코드)
def verify_citation(claim, url):
    # 1. 웹 스크래핑
    source_content = scrape_web_content(url)

    if source_content is None:
        return "VERIFICATION_FAILED"

    # 2. LLM 기반 판단
    prompt = f"""
    주장: {claim['statement']}
    출처 내용: {source_content}

    위 출처가 주장을 뒷받침합니까?
    답변: 예/아니오/불명확
    """

    judgment = llm_judge(prompt)

    if judgment == "예":
        return "SUPPORTED"
    elif judgment == "아니오":
        return "NOT_SUPPORTED"
    else:
        return "UNCLEAR"
```

검증 예시:

```
주장: "Python은 2023년 가장 인기 있는 프로그래밍 언어였다"
출처: https://www.tiobe.com/tiobe-index/

검증 과정:
1. 웹페이지 스크래핑
2. 내용 확인: "Python이 2023년 TIOBE Index 1위"
3. 판단: SUPPORTED ✓

---

주장: "Python은 1995년에 처음 출시되었다"
출처: https://www.python.org/downloads/

검증 과정:
1. 웹페이지 스크래핑
2. 내용 확인: 다운로드 페이지에는 릴리스 연도 정보 없음
3. 판단: NOT_SUPPORTED ✗
   (실제로는 Python은 1991년 출시되었으며, 잘못된 출처 인용)
```

#### 3. 메트릭 계산 (Metrics Calculation)

FACT는 두 가지 핵심 메트릭을 계산합니다:

1) Citation Accuracy (인용 정확도)

전체 인용 중 실제로 출처가 주장을 뒷받침하는 비율입니다:

```
공식:
Citation Accuracy = (검증된 인용 수) / (전체 인용 수) × 100%

예시:
총 인용: 20개
검증된 인용: 16개
인용 정확도 = 16/20 × 100% = 80%
```

2) Effective Citations (효과적 인용 수)

과제당 실제로 검증된 인용의 평균 개수입니다:

```
공식:
Effective Citations = Σ(각 과제의 검증된 인용 수) / 총 과제 수

예시:
과제 1: 검증된 인용 12개
과제 2: 검증된 인용 8개
과제 3: 검증된 인용 15개

Effective Citations = (12 + 8 + 15) / 3 = 11.67
```

### FACT 평가의 실제 적용

```python
# 전체 FACT 평가 파이프라인 (의사코드)
def fact_evaluate(report):
    # 1. 주장 및 출처 추출
    claims = extract_claims_and_sources(report)
    print(f"총 {len(claims)}개의 주장-인용 쌍 추출됨")

    # 2. 각 인용 검증
    verification_results = []
    for claim in claims:
        result = verify_citation(claim, claim['url'])
        verification_results.append({
            'claim': claim,
            'verification': result
        })

    # 3. 메트릭 계산
    total_citations = len(claims)
    supported_citations = sum(
        1 for r in verification_results
        if r['verification'] == 'SUPPORTED'
    )

    citation_accuracy = (supported_citations / total_citations) * 100
    effective_citations = supported_citations

    return {
        'citation_accuracy': citation_accuracy,
        'effective_citations': effective_citations,
        'total_citations': total_citations,
        'detailed_results': verification_results
    }

# 사용 예시
report = generate_research_report("AI의 윤리적 과제")
fact_scores = fact_evaluate(report)

print(f"인용 정확도: {fact_scores['citation_accuracy']:.2f}%")
print(f"효과적 인용 수: {fact_scores['effective_citations']}")
print(f"전체 인용 수: {fact_scores['total_citations']}")
```

### FACT의 중요성

#### 1. 환각(Hallucination) 방지

LLM은 그럴듯하지만 거짓인 정보를 생성할 수 있습니다:

```
나쁜 예:
"노벨 물리학상 수상자 Albert Einstein은 1905년 MIT에서 박사학위를 받았다[1]"
[1] https://www.nobelprize.org/

문제:
- Einstein은 취리히 대학교에서 박사학위를 받았음 (MIT 아님)
- 인용된 출처에는 이 정보가 없음
- FACT는 이를 NOT_SUPPORTED로 판정
```

#### 2. 인용 품질 보장

단순히 많은 인용이 아닌, 정확한 인용이 중요합니다:

```
케이스 A: 50개 인용, 인용 정확도 60% → 효과적 인용 30개
케이스 B: 35개 인용, 인용 정확도 95% → 효과적 인용 33개

FACT 평가: 케이스 B가 더 우수
(더 적은 인용이지만 높은 신뢰성)
```

#### 3. 연구 신뢰성 향상

학술 및 전문 연구에서 인용의 정확성은 필수적입니다:

- 학술 연구: 잘못된 인용은 연구 전체의 신뢰성 손상
- 정책 보고서: 부정확한 데이터로 잘못된 결정 초래 가능
- 기술 문서: 잘못된 출처는 구현 오류로 이어질 수 있음

### FACT의 기술적 구현

DeepResearch Bench는 자동화된 인용 검증 파이프라인을 사용합니다:

웹 스크래핑:
- 기본 도구: Jina AI Reader API
- URL에서 텍스트 콘텐츠 추출
- JavaScript 렌더링 지원
- 클린한 마크다운 형식으로 변환

```python
# Jina API를 사용한 웹 스크래핑
def scrape_with_jina(url):
    jina_url = f"https://r.jina.ai/{url}"
    headers = {
        "Authorization": f"Bearer {JINA_API_KEY}"
    }
    response = requests.get(jina_url, headers=headers)
    return response.text  # 마크다운 형식의 콘텐츠
```

LLM 기반 검증:
- 평가 모델: Google Gemini API
- 주장과 출처 내용 비교
- 3단계 판정: 지지함 / 지지하지 않음 / 불명확

```python
verification_prompt = f"""
주장: {claim}

출처 URL: {source_url}

출처 내용:
{scraped_content}

위 출처가 주장을 실질적으로 뒷받침합니까?

판정 기준:
- "지지함": 출처가 주장을 직접적으로 또는 명확히 뒷받침
- "지지하지 않음": 출처가 주장과 무관하거나 모순
- "불명확": 출처 내용이 불완전하거나 간접적

답변: [지지함/지지하지 않음/불명확]
이유: [간단한 설명]
"""
```

메트릭 집계:
```python
# Citation Accuracy 계산
citation_accuracy = (supported_count / total_citations) * 100

# Effective Citations 계산
effective_citations = supported_count / num_tasks

# 태스크별 상세 결과
results = {
    "task_id": task_id,
    "total_citations": total,
    "supported_citations": supported,
    "accuracy": accuracy,
    "failed_verifications": failed_urls
}
```

### FACT의 한계와 고려사항

1. 웹 스크래핑의 어려움
- 페이월(paywall) 뒤의 콘텐츠
- 동적으로 로드되는 JavaScript 콘텐츠
- 접근 불가능한 URL
- 로봇 차단 (robots.txt) 정책

해결 방안:
- Jina AI Reader는 대부분의 JavaScript 콘텐츠 처리 가능
- 접근 실패한 URL은 "검증 실패"로 분류 (점수에서 제외 가능)

2. LLM 판단의 불확실성
- 미묘한 맥락 차이 해석
- 전문 분야 지식 요구
- 간접적 지지의 판단

해결 방안:
- 명확한 판정 기준 프롬프트
- 다수의 샘플에서 일관성 검증
- 필요시 인간 평가자와의 합의도 측정


## RACE + FACT: 종합 평가

DeepResearch Bench는 RACE와 FACT를 결합하여 연구 에이전트를 종합적으로 평가합니다:

### 평가 파이프라인

```
1. 쿼리 처리
   ↓
   100개 벤치마크 쿼리 (JSONL)

2. 보고서 생성
   ↓
   연구 에이전트가 인용 포함 보고서 생성

3. 독립 평가
   ↓
   RACE 평가 (품질)    FACT 평가 (사실성)
   - Comprehensiveness   - Citation Accuracy
   - Insight/Depth       - Effective Citations
   - Instruction
   - Readability

4. 결과 집계
   ↓
   리더보드 순위
```

### 출력 형식

연구 에이전트는 다음 JSONL 형식으로 결과를 제출해야 합니다:

```json
{
  "id": "task_001",
  "prompt": "2020-2024년 양자 컴퓨팅 발전 동향 분석",
  "article": "# 양자 컴퓨팅 발전 동향\n\n2020년 구글의 양자 우월성 달성 이후[1]...\n\n[1] https://www.nature.com/articles/quantum-supremacy"
}
```

### 실행 방법

```bash
# 1. API 키 설정
export GEMINI_API_KEY="your-key"
export JINA_API_KEY="your-key"

# 2. 에이전트 실행 및 결과 저장
python run_agent.py --output data/test_data/raw_data/my_agent.jsonl

# 3. 자동 평가 실행
bash run_benchmark.sh

# 4. 결과 확인
cat results/my_agent_scores.json
```

### 리더보드 메트릭

최종 평가는 다음 메트릭들을 포함합니다:

```json
{
  "model_name": "MyResearchAgent",
  "race_scores": {
    "overall": 85.2,
    "comprehensiveness": 88.5,
    "insight_depth": 82.3,
    "instruction_following": 90.1,
    "readability": 80.0
  },
  "fact_scores": {
    "citation_accuracy": 87.5,
    "effective_citations": 14.2,
    "total_citations": 16.3
  },
  "combined_score": 86.1
}
```

## 벤치마크 결과 및 분석

### 베이스라인 모델 평가

DeepResearch Bench는 다양한 연구 에이전트 시스템을 평가했습니다:

평가된 시스템 유형:
1. 순수 LLM 기반: GPT-4, Claude, Gemini 등을 백엔드로 사용
2. RAG 기반 에이전트: 검색 증강 생성 시스템
3. 다중 단계 에이전트: 계획-실행-검증 파이프라인
4. 특화된 연구 에이전트: 학술 연구에 최적화된 시스템


### 주요 발견사항

1. 품질과 정확성의 트레이드오프

일부 시스템은 RACE 점수는 높지만 FACT 점수가 낮은 경향:
```
시스템 A: RACE 88, Citation Accuracy 68%
→ 잘 쓰여졌으나 인용 신뢰성 낮음

시스템 B: RACE 82, Citation Accuracy 92%
→ 더 신뢰할 수 있는 연구 결과
```

2. 도메인별 성능 차이

- 과학/공학: 높은 FACT 점수 (구체적 데이터 선호)
- 인문학/사회과학: 높은 Insight/Depth 점수 필요
- 비즈니스/실무: Readability와 Instruction-Following 중요

3. 언어별 성능

- 영어 과제: 전반적으로 더 높은 점수
- 중국어 과제: 리소스 제약으로 약간 낮은 점수
- 다국어 지원 시스템의 중요성

### 기존 벤치마크와의 비교

DeepResearch Bench vs 전통적 벤치마크:

```
전통적 Q&A 벤치마크 (MMLU, TriviaQA):
✓ 단일 답변 정확도 측정
✗ 복잡한 연구 능력 평가 불가
✗ 인용 검증 없음

DeepResearch Bench:
✓ 다단계 연구 프로세스 평가
✓ 인용 신뢰성 검증
✓ 4개 차원 종합 평가
✓ 실제 연구 시나리오 반영
```

LLM 벤치마크 생태계에서의 위치:
```
단순 ← --------------------------------- → 복잡

[GLUE/SuperGLUE] → [MMLU] → [BigBench] → [DeepResearch Bench]
  NLU 기본 능력      지식 평가    복합 추론     심층 연구 능력
```

## 실전 팁: 높은 RACE/FACT 점수 받기

### RACE 점수 향상 전략

1. Comprehensiveness 높이기
- 주제의 모든 주요 측면 다루기
- 최신 동향과 역사적 맥락 모두 포함
- 다양한 관점과 반대 의견도 제시

2. Insight/Depth 강화
- "왜"와 "어떻게"에 답하기
- 단순 나열이 아닌 분석과 종합
- 비판적 평가와 함의 도출

3. Instruction-Following 완벽히
- 요구사항 체크리스트 작성
- 명시된 제약사항 철저히 준수
- 형식 요구사항 확인

4. Readability 개선
- 명확한 구조와 섹션 분리
- 핵심 용어 정의
- 시각 자료 활용 (표, 그래프)
- 논리적 흐름 유지

### FACT 점수 향상 전략

1. 고품질 출처 사용
- 신뢰할 수 있는 학술 저널, 공식 문서
- 최신 자료 우선
- 1차 자료 선호

2. 정확한 인용
- 주장과 출처의 정확한 매칭
- 과도한 일반화 피하기
- 출처가 실제로 해당 내용 포함하는지 확인

3. 검증 가능한 주장
```
좋은 예:
"2023년 3분기 Tesla의 전기차 판매량은 43만 5천 대였다[1]"
→ 구체적, 검증 가능

나쁜 예:
"Tesla는 전기차 시장을 지배하고 있다[1]"
→ 모호함, "지배"의 정의 불명확
```

4. 인용 밀도 최적화
- 너무 적음: 신뢰성 저하
- 너무 많음: 정확도 저하 위험
- 적정선: 주요 주장마다 1-2개 인용

## 한계점

1. 평가 비용
- LLM API 호출 비용 (Gemini + Jina)
- 100개 과제 전체 평가 시 상당한 비용 발생
- 대규모 실험에는 제약

2. 평가자 일관성
- LLM 기반 평가의 변동성
- 동일 입력에 대한 재현성 이슈
- 여러 번 실행 시 미세한 점수 차이

3. 도메인 커버리지
- 22개 분야이지만 세부 전문 영역은 제한적
- 신흥 분야 (AI 안전성, 양자 기술 등) 과소 대표
- 과제당 1-2개 정도로 통계적 대표성 제한

4. 다중 모달리티 부족
- 현재는 텍스트 중심 평가
- 이미지, 표, 그래프 생성 평가 미포함
- 실제 연구는 시각 자료 포함하는 경우 많음

## 결론

DeepResearch Bench의 RACE와 FACT 평가 방법론은 LLM 기반 연구 에이전트를 포괄적으로 평가하는 혁신적인 프레임워크입니다:

- RACE: 연구 보고서의 품질을 4개 차원(포괄성, 통찰력, 지시사항 준수, 가독성)에서 동적 기준과 참조 기반으로 평가  
- FACT: 인용의 정확성과 신뢰성을 자동 검증하여 환각을 방지하고 연구 신뢰성 보장  

이 두 방법론은 상호 보완적으로 작동하여:
- 품질과 정확성의 균형 추구
- 단일 메트릭 최적화 방지
- 실제 연구 품질 반영


## 참고 자료

- [DeepResearch Bench GitHub Repository](https://github.com/Ayanami0730/deep_research_bench)
- [DeepResearch Bench: A Comprehensive Benchmark for Deep Research Agents (arXiv:2506.11763)](https://arxiv.org/abs/2506.11763)
- Authors: Mingxuan Du, Benfeng Xu, Chiwei Zhu, Xiaorui Wang, Zhendong Mao
