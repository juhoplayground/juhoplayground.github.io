---
layout: post
title: Langchain PDF Chatbot 만들기 - 22 - OpenSouce Embedding 모델 찾기
author: 'Juho'
date: 2025-04-14 09:00:00 +0900
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
1. [모델 선정 기준](#모델-선정-기준)
2. [MTEB](#mteb)
 - [MTEB로 측정가능한 Task 목록](#mteb로-측정가능한-task-목록)
 - [MTEB 테스트 코드](#mteb-테스트-코드)
3. [테스트한 OpenSource Embedding 모델 List](#ragchecker-주요-metric-설명)

## 기존 사용 Embedding 모델
[OpenAI가 제공하는 임베딩 모델](https://platform.openai.com/docs/guides/embeddings){:target="_blank"}과  [Anthropic이 권장하는 임베딩 모델](https://docs.anthropic.com/ko/docs/build-with-claude/embeddings){:target="_blank"} 중에서 text-embedding-3-large를 사용하고 있었다.  

그런데 OpenSource 임베딩 모델을 확인해서 사용해야하는 상황이 되어버렸다.


## 모델 선정 기준
우선 HuggingFace에서 library가 sentence-transformers인 것 중에서 한국어를 지원하는 것  
그리고 Task가 sentence-similarity와 feature-extraction 둘 중에 하나인 것으로 우선순위를 정해서 테스트    
- sentence-similarity : 문장간 의미적 유사도를 평가하는데 최적화  
- feature-extraction : 입력 텍스트로부터 특징을 추출해 다양한 목적에 사용할 수 있는 임베딩 생성이 목적  

그 외에도 다른 사람들이 많이 사용하거나 추천하는 임베딩 모델도 테스트했음  

## MTEB
[MTEB](https://arxiv.org/abs/2210.07316){:target="_blank"}는 임베딩 모델을 평가하기 위한 [벤치마크 데이터](https://github.com/embeddings-benchmark/mteb){:target="_blank"}  


### MTEB로 측정가능한 Task 목록
[MTEB로 측정가능한 Task 목록](https://github.com/embeddings-benchmark/mteb/blob/main/docs/tasks.md){:target="_blank"}에서 Task를 선정하는 기준
- 한국어 데이터 셋  
- RAG에 필요한 평가 지표 Task  
  - STS(Semetic Textual Similarity) : 두 문장이 의미적으로 얼마나 유사한지 평가  
  - Retrieval : 주어진 질의를 바탕으로 관련 있는 문서를 찾아내는지 평가  
  - Reranking : 초기 검색 결과나 후보 리스트를 재정렬하여 관련성 높은 항목을 상위 배치하는지 평가  

### MTEB 테스트 코드
MTEB는 `pip install mteb`를 이용해서 설치  
```python
import mteb

tasks = mteb.get_tasks(languages=["kor"]).to_dataframe()
tasks = tasks[tasks['type'].isin(['STS', 'Retrieval', 'Reranking'])]
tasks = tasks['name'].tolist()
tasks = mteb.get_tasks(tasks=tasks_name)

model = mteb.get_model(model_name, device="cuda", trust_remote_code=True, token=HUGGINGFACE_API_KEY)

evaluation = mteb.MTEB(tasks=tasks)

results = evaluation.run(model, output_folder=f"embedding_test_results/{model_name}")
```

## 테스트한 OpenSource Embedding 모델 List
- [jinaai/jina-embeddings-v3](https://huggingface.co/jinaai/jina-embeddings-v3){:target="_blank"}
- [nomic-ai/nomic-embed-text-v2-moe](https://huggingface.co/nomic-ai/nomic-embed-text-v2-moe){:target="_blank"}
- [intfloat/e5-mistral-7b-instruct](https://huggingface.co/intfloat/e5-mistral-7b-instruct){:target="_blank"}
- [intfloat/multilingual-e5-large-instruct](https://huggingface.co/intfloat/multilingual-e5-large-instruct){:target="_blank"}
- [sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2](https://huggingface.co/sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2){:target="_blank"}
- [Alibaba-NLP/gte-multilingual-base](https://huggingface.co/Alibaba-NLP/gte-multilingual-base){:target="_blank"}
- [Alibaba-NLP/gte-Qwen2-7B-instruct](https://huggingface.co/Alibaba-NLP/gte-Qwen2-7B-instruct){:target="_blank"}
- [bespin-global/klue-sroberta-base-continue-learning-by-mnr](https://huggingface.co/bespin-global/klue-sroberta-base-continue-learning-by-mnr){:target="_blank"}
- [upskyy/kf-deberta-multitask](https://huggingface.co/upskyy/kf-deberta-multitask){:target="_blank"}
- [Salesforce/SFR-Embedding-2_R](https://huggingface.co/Salesforce/SFR-Embedding-2_R){:target="_blank"}
- [BAAI/bge-multilingual-gemma2](https://huggingface.co/BAAI/bge-multilingual-gemma2){:target="_blank"}
- [upskyy/bge-m3-korean](https://huggingface.co/upskyy/bge-m3-korean){:target="_blank"}
- [jhgan/ko-sroberta-multitask](https://huggingface.co/jhgan/ko-sroberta-multitask){:target="_blank"}
- [nlpai-lab/KURE-v1](https://huggingface.co/nlpai-lab/KURE-v1){:target="_blank"}
- [nlpai-lab/KoE5](https://huggingface.co/nlpai-lab/KoE5){:target="_blank"}
- [kakaocorp/kanana-nano-2.1b-embedding](https://huggingface.co/kakaocorp/kanana-nano-2.1b-embedding){:target="_blank"}
- [McGill-NLP/LLM2Vec-Meta-Llama-3-8B-Instruct-mntp-supervised](https://huggingface.co/McGill-NLP/LLM2Vec-Meta-Llama-3-8B-Instruct-mntp-supervised){:target="_blank"}
- [dragonkue/snowflake-arctic-embed-l-v2.0-ko](https://huggingface.co/dragonkue/snowflake-arctic-embed-l-v2.0-ko){:target="_blank"}
- [facebook/drama-1b](https://huggingface.co/facebook/drama-1b){:target="_blank"}
- [klue/bert-base](https://huggingface.co/klue/bert-base){:target="_blank"}

---  
