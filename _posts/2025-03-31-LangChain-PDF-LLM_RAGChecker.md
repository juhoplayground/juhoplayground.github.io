---
layout: post
title: Langchain PDF Chatbot 만들기 - 21 - RAGChecker
author: 'Juho'
date: 2025-03-31 09:00:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, RAGChecker, Python]
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
1. [RAGChecker란?](#ragchecker란)
2. [RAGChecker 주요 특징](#ragchecker-주요-특징)
3. [RAGChecker 주요 Metric 설명](#ragchecker-주요-metric-설명)
4. [RAGChecker Code](#ragchecker-code)

## RAGChecker란?
[RAGChecker](https://arxiv.org/abs/2408.08067){:target="_blank"}는 [Amazon에서 만든 RAG 시스템의 성능을 평가하는 프레임워크](https://github.com/amazon-science/RAGChecker){:target="_blank"}입니다.<br/>
이틀 통해 개발자는 RAG 시스템의 성능 대한 포괄적인 지표를 확인하고 분석하여 개선할 수 있습니다.<br/>



## RAGChecker 주요 특징
1) 전체 평가<br/>
- RAGChecker는 전체 RAG 파이프라인 평가를 위한 `Overall Metrics`를 제공합니다.<br/>


2) 진단 지표<br/>
-  검색 컴포넌트를 분석하기 위한 `Diagnostic Retriever Metric`와 생성 컴포넌트를 평가하기 위한 `Diagnostic Generator Metrics`를 통해 목표 개선을 위한 인사이트를 제공합니다.<br/>


3) 세부 평가<br/>
- `claim-level entailment`의 연산을 활용하여 세부적인 평가를 수행합니다.<br/>


4) 벤치마크 데이터셋<br/>
- 10개 도메인 분야를 아우르는 4,000개 질문을 포함한 포괄적인 RAG 벤치마크 데이터셋을 제공합니다 (곧 출시 예정).<br/>


5) 메타 평가<br/>
- RAGChecker의 결과와 인간 판단 간의 상관관계를 평가하기 위한 인간 주석 기반 선호 데이터셋을 포함합니다.<br/>

## RAGChecker 주요 Metric 설명
1) Overall Metrics (RAG 시스템의 전반적인 성능을 평가)<br/>
- Precision<br/>
- Recall<br/>
- F1 Score<br/>

2) Diagnostic Retriever Metrics (검색 컴포넌트의 성능 평가)<br/>
- Claim Recall: 검색된 문맥이 얼마나 많은 관련 정보를 포함하는지 평가<br/>
- Context Precision: 검색된 문맥 중 실제로 정확환 정보의 비율<br/>

3) Diagnostic Generator Metrics (생성 컴포넌트의 성능 평가)<br/>
- Faithfulness: 생성된 응답이 제공된 문맥에 얼마나 충실한지 평가<br/>
- Noise Sensitivity: 생성기가 관련 정보와 노이즈를 어떻게 처리하는지 분석<br/>
- Hallucination: 문맥에 없는 정보를 생성하는 비율 평가<br/>
- Self-knowledge: 생성기가 자체 지식에 의존해 응답을 생성하는 비율<br/>
- Context Utilization: 검색된 문맥 정보를 얼마나 효과적으로 활용하는지 평가<br/>


## RAGChecker Code
1) 설치 <br/>
```
pip install ragchecker
python -m spacy download en_core_web_sm
```

2) CLI를 통한 실행<br/>
```
ragchecker-cli \
    --input_path=examples/checking_inputs.json \
    --output_path=examples/checking_outputs.json \
    --extractor_name=bedrock/meta.llama3-1-70b-instruct-v1:0 \
    --checker_name=bedrock/meta.llama3-1-70b-instruct-v1:0 \
    --batch_size_extractor=32 \
    --batch_size_checker=32 \
    --metrics all_metrics
```

3) Python 코드를 통한 실행<br/>
```python
from ragchecker import RAGResults, RAGChecker
from ragchecker.metrics import all_metrics


with open("examples/checking_inputs.json") as fp:
    rag_results = RAGResults.from_json(fp.read())

evaluator = RAGChecker(extractor_name="bedrock/meta.llama3-1-70b-instruct-v1:0",
                      checker_name="bedrock/meta.llama3-1-70b-instruct-v1:0",
                      batch_size_extractor=32,
                      batch_size_checker=32)

evaluator.evaluate(rag_results, all_metrics)
print(rag_results)
```

4) 결과값
```
"""Output
RAGResults(
  2 RAG results,
  Metrics:
  {
    "overall_metrics": {
      "precision": 76.4,
      "recall": 62.5,
      "f1": 68.3
    },
    "retriever_metrics": {
      "claim_recall": 61.4,
      "context_precision": 87.5
    },
    "generator_metrics": {
      "context_utilization": 87.5,
      "noise_sensitivity_in_relevant": 19.1,
      "noise_sensitivity_in_irrelevant": 0.0,
      "hallucination": 4.5,
      "self_knowledge": 27.3,
      "faithfulness": 68.2
    }
  }
)
"""
```
이러한 형태의 결과값을 확인할 수 있습니다.<br/>

--- 

<br/>

RAGAS보다 자세하고 많은 성능 지표를 제공해서 최근에는 RAGChecker를 사용하게 되는 것 같습니다.<br/>