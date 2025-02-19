---
layout: post
title: Langchain PDF Chatbot 만들기 - 2 -  Splitter
author: 'Juho'
date: 2025-02-19 09:00:00 +0900
categories: [LangChain]
tags: [LangChain, PDF, Chatbot, Python]
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
1. [Splitter](#splitter)
 - 1) [Splitter 종류](#1-splitter-종류)
 - 2) [Splitter 테스트](#2-splitter-테스트)
 - 3) [Splitter 평가 및 결론](#3-splitter-평가-및-결론)

## Splitter
문서를 Load 한 것을 바로 사용하지 않고 Split 단계를 거쳐 작은 단위인 chunk로 나누는 이유는 크게 2가지입니다.<br/>
- LLM 모델 입력 토큰의 수가 정해져 있기 때문에 허용 한도를 넘지 않아야 합니다.<br/>
- 문서가 너무 긴 경우 중요 정보를 제외한 불필요한 정보가 포함될 수 있어 품질의 정확도가 떨어지게 됩니다.<br/>
→ 핵심 정보가 유지될 수 있는 적절한 크키의 chunk로 나누는 것이 중요합니다.<br/>
chunk size, overlap을 결정하기 어렵다면 Greg Kamradt가 만든 [Chunk Visualization ](https://chunkviz.up.railway.app/){:target="_blank"} 사이트를 확인해보면서 결정하는것도 좋을 것 같습니다. <br/>

#### 1) Splitter 종류
LangChain은 다양한 Splitter를 제공하고 있습니다. <br/>
그 중에서 PDF의 split과 관련이 없는 Code(개발 언어), MarkdownHeader, HTMLHeader, HTMLSection, RecursiveJson은 제외하고 알아보도록 하겠습니다. <br/>
CharacterTextSplitter, RecursiveCharacterTextSplitter, TokenTextSplitter, SemanticChunker를 하나씩 확인해보겠습니다. <br/>
`pip install -U langchain_text_splitters langchain_experimental tiktoken`로 필요한 라이브러리를 설치합니다.<br/>


#### 2) Splitter 테스트
1) CharacterTextSplitter <br/>
가장 기본적인 CharacterTextSplitter부터 알아보면 주어진 텍스트를 문자 단위로 분할합니다.<br/>
간단한 문서나 구조가 복잡하지 않은 경우 빠르고 직관적으로 사용할 수 있지만 <br/>
문장의 중간이나 의미의 경계를 무시하고 단순히 문자 수로 분할되므로 문맥이 끊길 수 있는 단점이 있습니다.<br/>
```python
text_splitter = CharacterTextSplitter(separator="\n\n",
                                     chunk_size=1000,
                                     chunk_overlap=100,
                                     length_function=len,
                                     is_separator_regex=False)
```

2) RecursiveCharacterTextSplitter <br/>
여러 단계의 구분자를 활용하여 텍스트를 재귀적으로 분할합니다.<br/>
텍스트의 자연스러운 경계를 고려하여 분할하기 때문에 의미의 흐름을 보다 잘 보존할 수 있습니다.<br/>
문장이 중간에 끊기지 않도록 하여 문맥을 유지하는 데 유리합니다.<br/>
```python
text_splitter = RecursiveCharacterTextSplitter(separators=["\n\n", "\n", " ", ""],
                                              chunk_size=1000,
                                              chunk_overlap=100,
                                              length_function=len,
                                              is_separator_regex=False)
```

3) TokenTextSplitter<br/>
LLM 모델이 내부적으로 토큰화를 사용하기 때문에 모델의 토큰 수와 직접적으로 연동하여 분할합니다.<br/>
문자 단위보다는 모델의 실제 입력 단위인 토큰 기준으로 분할하므로 모델이 처리 가능한 범위 내에서 정확히 분할합니다.<br/>
토큰 기준 분할은 의미 단위와 일치하지 않을 수 있어 텍스트의 자연스러운 문맥이 끊길 가능성이 있습니다.<br/>
```python
text_splitter = TokenTextSplitter(model_name="gpt-4",
                                  chunk_size=1000,
                                  chunk_overlap=100)
```

4) SemanticChunker<br/>
단순히 문자나 토큰의 수로 분할하는 대신 텍스트의 의미(semantic)를 고려하여 관련 내용끼리 묶는 방식입니다. <br/>
내부적으로 임베딩과 유사도 계산을 활용하여 의미적으로 일관된 단위로 텍스트를 분할합니다.<br/>
문서에 다양한 주제가 혼재된 경우 문서 내에서 서로 관련 있는 내용들이 하나의 청크로 묶이게 되어 각 청크가 하나의 주제로 집중되어 더 나은 이해도를 제공합니다.<br/>
```python
text_splitter = SemanticChunker(embeddings = OpenAIEmbeddings(model=OPENAI_API_EMBEDDING, openai_api_key=OPENAI_API_KEY),
                               breakpoint_threshold_type="percentile",
                               breakpoint_threshold_amount=95)
```

#### 3) Splitter 평가 및 결론
문맥상의 연결이 자연스러운 것이 저는 중요하기 때문에 RecursiveCharacterTextSplitter,  SemanticChunker을 비교 진행했습니다.<br/>
우선 SemanticChunker를 이용해서 breakpoint_threshold_type을 percentile, standard_deviation, interquartile, gradient 로 변경하면서<br/>
각 type에 맞는 breakpoint_threshold_amount을 수정하면서 테스트를 반복했지만 제가 원하는 chunk 결과물이 나오지 않았습니다.<br/>
그래서 RecursiveCharacterTextSplitter과 정규식을 이용해서 처리 하기로 결정하였습니다.<br/>
<br/>

--- 
