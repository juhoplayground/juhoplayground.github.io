---
layout: post
title: Airflow Variable 설정
author: 'Juho'
date: 2025-02-01 09:00:00 +0900
categories: [Airflow]
tags: [Airflow, Python]
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
1. [Airflow Variable](#celery-airflow-retry)
 - 1) [Variable 설정(UI)](#1-variable-설정ui)
 - 2) [Variable 설정(Code)](#2-variable-설정code)
 - 3) [Variable 사용](#3-variable-사용)

## Airflow Variable
Airflow에서 Variable을 설정하는 방법에 대해 알아보도록 하겠습니다.<br/>
[Variable](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html){:target="_blank"}은 전역적으로 사용할 수 있는 값을 미리 정의해 두고 사용할 수 있습니다.<br/>

#### 1) Variable 설정(UI)
1. 상단 메뉴에서 Admin > Variables를 클릭 <br/>
![Image](https://github.com/user-attachments/assets/317ff373-9bbe-49bd-92e9-33ba49b06a9f)
2. 파란색 + 버튼 클릭 (Add a new record) 클릭 <br/>
![Image](https://github.com/user-attachments/assets/5d246735-a010-49f2-8c40-2b95b0aef0c7)
3. Variable의 키, 값 입력 후 저장 <br/>
값에는 text, list, json 등 다양한 자료형의 값을 입력할 수 있습니다. <br/>
![Image](https://github.com/user-attachments/assets/fa20f81d-0051-4357-b7f6-c524e57c9b82)
4. 설정된 값 확인
![Image](https://github.com/user-attachments/assets/974fb3a9-61bb-4852-ad0c-7fbe14ae99ae)

<br/>

#### 2) Variable 설정(Code)
```python
from airflow.models import Variable


a = Variable.set("key2", "value2")
```
이러한 형태로 Variable을 설정할 수 있습니다. <br/>

#### 3) Variable 사용
```python
# 기본 호출 방법
value = Variable.get("key")

# 호출 시 default 값 설정
value = Variable.get("key", default_var="value")

# json 형태의 값 호출
value = Variable.get("json_key", deserialize_json=True)
```
json 형태의 값이 아닌 경우에는 deserialize_json 옵션을 넣으면 오류가 발생한다.<br/>


---

<br/>

다음 글로는 airflow Variables을 이용한 동적 DAG 생성 및 dependency 설정에 대해 알아보겠습니다.
