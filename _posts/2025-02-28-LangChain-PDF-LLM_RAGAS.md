---
layout: post
title: Langchain PDF Chatbot 만들기 - 14 - RAGAS
author: 'Juho'
date: 2025-02-28 09:00:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, RAGAS, Python]
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
1. [RAGAS란?](#ragas란)
2. [RAGAS Code](#ragas-code)
3. [지표 결과](#지표-결과)

## RAGAS란?
RAG 파이프라인의 성능을 측정하기 위해서 Rule Base 지표들을 자동으로 게산하기 위한 프레임워크인 RAGAS를 사용합니다.<br/>
[RAGAS 공식 사이트](https://docs.ragas.io/en/stable/index.html){:target="_blank"}<br/>
[RAGAS Github](https://github.com/explodinggradients/ragas){:target="_blank"}<br/>

RAGAS 주요 성능 메트릭을 살펴보면 크게 Retrieval, Generation 각 카테고리 별 측면에서 메트릭을 정의할 수 있습니다.<br/>
Retrieval 은 정확하고 일관성 있는 답변 생성을 위해 정확성, 정밀성, 관련성 측면에서 좋은 품질의 Context 정보 검색 여부를 평가합니다.<br/>
Generation은 검색한 Context 정보를 기반으로 LLM이 생성한 답변의 정확성 그리고 질문에 대한 명확한 이해를 통해 답변 생성 여부에 대한 성능을 평가합니다.<br/>

#### 1) Retrieval
Context recall<br/>
검색된 문맥중에서 실제로 유용한 문맥의 비율을 의미한다.<br/>
<br/>
Context precision<br/>
질문을 답하는 데 필요한 전체 문맥 중에서 실제 검색된 문맥의 비율을 의미한다.<br/>

#### 2) Generation
Faithfulness<br/>
생성된 답변이 검색된 문맥과 일치하는 정도를 의미한다.<br/>
<br/>
Answer relevancy<br/>
생성된 답변이 사용자 질문과 얼마나 관련이 있는지를 측정하는 지표이다.<br/>

#### 3) End-to-End Evaluation
End-To-End 파이프라인의 성능을 평가하는 것도 사용자 경험에 직접적인 영향을 미치기 때문에 매우 중요합니다.<br/>
Ragas는 파이프라인의 전반적인 성능을 평가하는 데 사용할 수 있는 메트릭을 제공하여 종합적인 평가를 보장합니다.<br/>
<br/>
Answer semantic similarity<br/>
생성된 답변이 실제로 올바른지를 측정하는 지표이다.<br/>
<br/>
Answer correctness<br/>
생성된 답변과 기대되는 정답간의 의미적 유사도를 측정하는 지표이다.<br/>

## RAGAS Code
`pip install -U  ragas datasets`로 필요한 라이브러리를 설치합니다.<br/>
```python
test_questions, test_references = [], []
for ex in exmaple_jsons:
    test_questions.append(ex['human'])
    test_references.append(ex['ai'])

contexts = []
for query in test_questions:
    result = await retriever.invoke(query)
    contexts.append([doc.page_content for doc in result])

    result = chat_responses(query=query)
    answers.append(result)

dataset = Dataset.from_dict({"question": test_questions,
                            "answer": answers,
                            "contexts": contexts,
                            "reference" : test_references,
                            "ground_truths": test_references})

evaluate_result = evaluate(dataset = dataset, 
                          metrics=[context_precision, context_recall, faithfulness, answer_relevancy, answer_correctness, answer_similarity],
                          raise_exceptions=False)

result_df = evaluate_result.to_pandas()
```

## 지표 결과
이론적으로 완벽한 검색과 답변 생성이 이루어진다면 모든 지표가 1이 될 수 있음.<br/>
하지만 현실적으로는 일부 지표 간에 트레이드오프가 발생하기 때문에 완벽한 1을 달성하는 것은 거의 불가능한 것 같습니다.<br/>

**Context Precision vs. Context Recall**<br/>
Precision을 높이려면 검색된 문맥이 질문과 정확히 일치해야 함<br/>
→ 하지만 이 경우 더 많은 문맥을 검색하는 것이 어려워져 Recall이 낮아질 수 있음.<br/>
반대로 Recall을 높이려면 더 많은 문맥을 검색해야 하지만 이 경우 Precision이 떨어질 가능성이 있음.<br/>
<br/>

**Faithfulness vs. Answer Correctness**<br/>
Faithfulness를 1로 만들려면 LLM이 검색된 문맥 내에서만 답을 생성해야 함<br/>
→ 하지만 문맥이 불완전할 경우 정확한 답을 못 만들 수도 있음.<br/>
Answer Correctness를 1로 만들려면 경우에 따라 검색된 문맥 이외의 정보를 활용해야 하지만 이 과정에서 Faithfulness가 떨어질 수 있음.<br/>

**Answer Relevancy vs. Faithfulness**<br/>
Answer Relevancy를 높이려면 질문과 유사한 답변을 생성해야 하지만<br/>
Faithfulness를 1로 만들기 위해 문맥 내에서만 답변을 만들 경우 질문과 자연스럽게 연결되지 않을 수도 있음.<br/>

<br/>

--- 

<br/>
모든 지표를 1로 만들기는 어렵지만 균형을 맞추면서 높은 성능을 달성하기 위한 방법을 다음 글 부터 이야기해보도록 하겠습니다.<br/>