---
layout: post
title: Langchain PDF Chatbot 만들기 - 1 -  DataLoader
author: 'Juho'
date: 2025-02-13 09:00:00 +0900
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
1. [Loader](#loader)
 - 1) [Loader 종류](#1-loader-종류)
 - 2) [Loader 선택 과정](#2-loader-선택-과정)
 - 3) [Loader 테스트 코드](#3-loader-테스트-코드)
 - 4) [Loader 평가 및 결론](#4-loader-평가-및-결론)

## Loader
Load 단계가 중요한 이유는 쓰레기를 넣으면 쓰레기가 나온다(Garbage-In Garbage-Out)와 맞닿아 있습니다.<br/>
잘못된 데이터가 입력되면 아무리 후속 처리가 정교하더라도 부정확한 결과가 도출될 수 밖에 없습니다.<br/>
따라서 품질을 보장하기 위해서는 데이터를 정확하게 로드하는 과정이 필수입니다.<br/>

#### 1) Loader 종류
LangChain은 다양한 형태의 데이터를 처리할 수 있으며 대표적인 데이터 소스는 다음과 같이 분류할 수 있습니다. <br/>
1) 파일 기반 데이터: txt, pdf, doc, markdown 등<br/>
2) 웹 데이터: HTML 페이지 크롤링, 웹 검색 결과<br/>
3) 데이터베이스 데이터: MySQL, PostgreSQL, MongoDB 등의 정형 및 구조화된 데이터<br/>
4) API 응답 데이터: JSON, XML 등의 API 응답 데이터<br/>
5) 기타 구조화된 데이터: CSV, Excel, 로그 파일 등<br/>

PDF를 이용한 챗봇을 만들 것이기 때문에 PDF Loader를 확인해봐야 합니다. <br/>

#### 2) Loader 선택 과정
1) OCR 여부 확인 <br/>
PDF 문서를 로드할 때 OCR이 필요한지 여부를 고려해야 합니다. <br/>
이미지 기반 PDF, 손글씨 문서, 캡처된 웹페이지라면 OCR이 필요합니다.<br/>
하지만 제가 진행할 내용에서는 일반 텍스트 기반 PDF이므로 OCR이 필요하지 않습니다.<br/>

2) opensource 여부 확인 <br/>
LangChain에서는 다양한 PDF Loader를 제공합니다.<br/>
그중에서 유료 API가 필요한 Unstructured, Upstage Document Parse Loader, Amazon Textract, MathPix는 제외하였습니다.<br/>
그래서 테스트를 해볼 Loader는 다음과 같습니다.<br/>
`PyPDFLoader, PDFPlumberLoader, PyPDFium2Loader, PyMuPDFLoader, PDFMinerLoader, DoclingLoader` <br/>

#### 3) Loader 테스트 코드
```python
# 1. PyPDFLoader
def extract_text_pypdf(pdf_path):
   loader = PyPDFLoader(pdf_path,
                                 mode=mode,
                                 extraction_mode=extraction,
                                 extraction_kwargs={'layout_mode_space_vertically': layout_mode_space_vertically,
                                                    'layout_mode_scale_weight': layout_mode_scale_weight})      
   return documents

# 2. PDFPlumber
def extract_text_pdfplumber(pdf_path):
   loader = PDFPlumberLoader(pdf_path,
                             text_kwargs={"x_tolerance": x_tolerance,
                                          "y_tolerance": y_tolerance,
                                          "keep_linebreaks": keep_linebreaks,
                                          "keep_blank_chars": keep_blank_chars,
                                          "use_text_flow": use_text_flow,
                                          "sort_text": sort_text})
   return documents

# 3. PyPDFium2
def extract_text_pypdfium2(pdf_path):
   loader = PyPDFium2Loader(pdf_path)
   return documents

# 4. PyMuPDF
def extract_text_pymupdf(pdf_path):
   loader = PyMuPDFLoader(pdf_path,
                          extract_tables=extract_tables,
                          mode=mode)
   return documents

# 5. PDFMiner
def extract_text_pdfminer(pdf_path):
   loader = PDFMinerLoader(pdf_path,
                           mode=mode) 
   return documents

# 6. Docling
def extract_text_docling(pdf_path):
   pipeline_options = PdfPipelineOptions(do_table_structure=do_table_structure)
   pipeline_options.table_structure_options.mode = structure_options_mode
   converter = DocumentConverter(format_options={InputFormat.PDF: PdfFormatOption(pipeline_options=pipeline_options)}
   loader = DoclingLoader(pdf_path,
                          converter=converter),
                          export_type=export_type,
                          chunker=chunker) 
   return documents

# 실행 및 비교
extractors = {
   "PyPDF": extract_text_pypdf,
   "PDFPlumber": extract_text_pdfplumber,
   "PyPDFium2": extract_text_pypdfium2,
   "PyMuPDF": extract_text_pymupdf,
   "PDFMiner": extract_text_pdfminer,
   "Docling": extract_text_docling
}
for name, extractor in extractors.items():
   pdf_path = "./app/data/sample.pdf"
   print(f"\n{name} 결과:")
   docs = extractor(pdf_path)
   for index, doc in enumerate(docs, start=1):
       print('*'* 10, f'{index} 페이지 내용', '*'*10)
       print(doc)
   print('*'*20)
```

#### 4) Loader 평가 및 결론
제가 가진 PDF를 기준으로 주관적인 평가를 확인해보면 <br/>
PyPDF<br/>
- 적은 커스텀 설정만으로도 문맥이 잘 유지됨.<br/>

PDFPlumber<br/>
- 가장 커스텀 가능한 옵션이 많으며 문맥 유지가 뛰어남.<br/>

PyPDFium2<br/>
- 별도 커스텀이 불가능함.<br/>
- 테이블의 행/열을 인식하지 못해 줄 바꿈이 많음.<br/>

PyMuPDF<br/>
- 테이블 추출 기능이 있으나 정확도가 낮음.<br/>
- 테이블 구조를 제대로 인식하지 못해 줄 바꿈이 많음.<br/>

PDFMiner<br/>
- 별도 커스텀이 불가능함.<br/>
- 테이블 구조를 전혀 인식하지 못해 줄 바꿈이 많음<br/>

Docling<br/>
- 문장이 조각나서 단어를 제대로 인식하지 못함.<br/>
- 표 데이터가 완전히 깨짐.<br/>
- 처리 속도가 매우 느림.<br/>

<br/>

PDF 데이터를 효과적으로 불러오기 위해 여러 Loader를 다양한 파라미터로 테스트한 결과 PDFPlumberLoader가 가장 적합하다고 판단하였습니다.<br/>
1) 커스텀 가능성이 높음 → 다양한 파라미터 조정 가능<br/>
2) 문맥 유지가 우수함 → 다른 Loader보다 원본 형태를 더 잘 보존<br/>
3) 테이블 구조를 올바르게 인식 → 줄 바꿈이 적고 레이아웃 유지가 좋음<br/>

--- 
