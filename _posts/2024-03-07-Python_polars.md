---
layout: post
title: Python - Polars
author: 'Juho'
date: 2024-03-07 09:00:00 +0900
categories: [Polars]
tags: [Polars, python]
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
1. [Polars를 찾게 된 이유](#polars-찾게-된-이유)
2. [Polars란?](#polars란)
3. [Polars 기초 문법](#polars-기초-문법)

## Polars 찾게 된 이유
기존에 데이터 처리 및 분석에 Pandas를 사용하고 있었다.<br/>
하지만 대용량 데이터 처리에 대한 처리 속도와 메모리 이슈 등의 한계로 다른 방법을 찾게 되었다.<br/>
[H20의 벤치마크 결과](https://h2oai.github.io/db-benchmark/){:target="_blank"}를 보면 대부분의 경우에 Pandas보다 Polars가 빠른 속도를 보인다.<br/>


## Polars란?
[Polars](https://docs.pola.rs/py-polars/html/reference/){:target="_blank"}는 Rust로 구현된 데이터 처리 및 분석 도구로 Pandas와 비슷한 API를 제공한다.<br/>
차이점으로는 Polars는 Dataframe에 인덱스를 사용하지 않아 데이토 조작이 간편하다.<br/>
내부 데이터 표현에 Apache Arrow 배열을 사용하여 로드 시간, 메모리 사용량 및 계산 효율성이 높다.<br/>
병렬 처리를 제공하여 대용량 데이터 처리를 더욱 빠르게 할 수 있다.<br/>
Pandas의 eager evaluation과 달리 lazy evaluation을 지원하여 필요에 따라 쿼리를 최적화하고 메모리 사용량을 최소화한다.<br/>

## Polars 기초 문법
설치
```
pip install polars
```

1) read_csv
```python
df = pl.read_csv('data.csv')
```

2) select
```python
# Dataframe에서 특정 열을 선택
df = pl.DataFrame(
    {
        "foo": [1, 2, 3],
        "bar": [6, 7, 8],
        "ham": ["a", "b", "c"],
    }
)
df.select("foo")

# 여러 열은 열 이름의 리스트를 사용
df.select(["foo", "bar"])

# 리스트 대신 위치 인자를 사용하여 여러 열을 선택할 수 있으며 표현식도 허용됨
df.select(pl.col("foo"), pl.col("bar") + 1)

# 표현식 입력을 쉽게 지정하려면 키워드 인자를 사용하면 됨
df.select(threshold=pl.when(pl.col("foo") > 2).then(10).otherwise(0))

# 다중 출력을 가진 표현식은 `Config.set_auto_structify(True)`설정을 활성화하여 자동으로 Struct로 생성 가능함
with pl.Config(auto_structify=True):
    df.select(is_odd=(pl.col(pl.INTEGER_DTYPES) % 2).name.suffix("_is_odd"))
```

3) with_columns
```python
# Dataframe에 신규 열을 추가
# 동일한 이름을 가진 기존 열은 새로 추가 된 열로 대체됨
# 이 방법으로 새 DataFrame을 생성하면 기존 데이터의 새 복사본이 생성되지 않음

# 신규 열로 추가할 표현식을 입력
df = pl.DataFrame(
    {
        "a": [1, 2, 3, 4],
        "b": [0.5, 4, 10, 13],
        "c": [True, True, False, True],
    }
)
df.with_columns((pl.col("a") ** 2).alias("a^2"))

# 여러 열은 표현식의 리스트를 전달하여 추가할 수 있음
df.with_columns(
    [
        (pl.col("a") ** 2).alias("a^2"),
        (pl.col("b") / 2).alias("b/2"),
        (pl.col("c").not_()).alias("not c"),
    ]
)

# 리스트 대신 위치 인자를 사용하여도 여러 열을 추가할 수 있음
df.with_columns(
    (pl.col("a") ** 2).alias("a^2"),
    (pl.col("b") / 2).alias("b/2"),
    (pl.col("c").not_()).alias("not c"),
)

# 표현식 입력을 쉽게 지정하려면 키워드 인자를 사용하면 됨
df.with_columns(
    ab=pl.col("a") * pl.col("b"),
    not_c=pl.col("c").not_(),
)

# 다중 출력을 가진 표현식은 `Config.set_auto_structify(True)`설정을 활성화하여 자동으로 Struct로 생성 가능함
with pl.Config(auto_structify=True):
    df.drop("c").with_columns(
        diffs=pl.col(["a", "b"]).diff().name.suffix("_diff"))
```

4) filter
```python
# 하나 이상의 조건식을 기반으로 DataFrame의 행을 필터링
# 남은 행의 원래 순서가 보존
df = pl.DataFrame(
    {
        "foo": [1, 2, 3],
        "bar": [6, 7, 8],
        "ham": ["a", "b", "c"],
    }
)

# 한 가지 조건으로 필터링
df.filter(pl.col("foo") > 1)

# and/or 연산자와 결합하여 여러 조건으로 필터링
df.filter((pl.col("foo") < 3) & (pl.col("ham") == "a"))
df.filter((pl.col("foo") == 1) | (pl.col("ham") == "c"))

# *args를 사용하여 여러 조건으로 필터링
df.filter(
    pl.col("foo") <= 2,
    ~pl.col("ham").is_in(["b", "c"]),
)

# **kwargs를 사용하여 여러 조건으로 필터링
df.filter(foo=2, ham="b")
```

5) Group By
```python
# 하나의 열을 기준으로 group by하고 agg를 호출하여 다른 열의 group by된 합계를 계산
df = pl.DataFrame(
    {
        "a": ["a", "b", "a", "b", "c"],
        "b": [1, 2, 1, 3, 3],
        "c": [5, 4, 3, 2, 1],
    }
)
df.group_by("a").agg(pl.col("b").sum())

# maintain_order=True로 설정하면 group 된 결과의 순서가 입력과 동일하게 유지됨
df.group_by("a", maintain_order=True).agg(pl.col("c"))

# 여러 열 기준으로 group by하려면 열 이름의 리스트를 사용
df.group_by(["a", "b"]).agg(pl.max("c"))

# 동일한 방식으로 여러 열을 기준으로 group by하려면 위치 인자를 사용하거나 표현식을 사용
df.group_by("a", pl.col("b") // 2).agg(pl.col("c").mean())

# 이 방법으로 반환된 GroupBy 객체는 반복 가능하며, 각 group의 이름과 데이터를 반환
for name, data in df.group_by("a"):  
    print(name)
    print(data)
```

6) when
```python
# Python의 if-else 문과 유사한 표현식
# 항상 pl.when(<condition>).then(<value if condition>).로 시작
# 하나 이상의 .when(<condition>).then(<value>) 문을 연결할 수 있음
# 조건이 하나도 참이 아닌 경우, 옵션으로 .otherwise(<value if all statements are false>)를 끝에 추가할 수 있음
# 추가되지 않고 조건이 하나도 참이 아닌 경우 None이 반환

df = pl.DataFrame({"foo": [1, 3, 4], "bar": [3, 4, 0]})

# when-then-otherwise
df.with_columns(pl.when(pl.col("foo") > 2).then(1).otherwise(-1).alias("val"))

# 여러 when-then
df.with_columns(
    pl.when(pl.col("foo") > 2)
    .then(1)
    .when(pl.col("bar") > 2)
    .then(4)
    .otherwise(-1)
    .alias("val")
)

# otherwise가 없는 경우 
df.with_columns(pl.when(pl.col("foo") > 2).then(1).alias("val"))
```

7) join
```python
# SQL과 유사한 방식

df = pl.DataFrame(
    {
        "foo": [1, 2, 3],
        "bar": [6.0, 7.0, 8.0],
        "ham": ["a", "b", "c"],
    }
)
other_df = pl.DataFrame(
    {
        "apple": ["x", "y", "z"],
        "ham": ["a", "b", "d"],
    }
)

# how를 지정하지 않은 경우 (inner join과 같음)
df.join(other_df, on="ham")
df.join(other_df, on="ham", how="inner")

# left
df.join(other_df, on="ham", how="left")

# outer 
df.join(other_df, on="ham", how="outer")
df.join(other_df, on="ham", how="outer_coalesce") #중복된 키를 병합함

# cross (카타시안 곱)
df.join(other_df, on="ham", how="cross")

# semi (오른쪽 테이블에서 일치하는 행을 필터링)
df.join(other_df, on="ham", how="semi")

# anti(오른쪽 테이블에서 일치하지 않는 행을 필터링)
df.join(other_df, on="ham", how="anti")

# validate 옵션을 추가할 경우 join의 유형별로 validate하고
# validation하지 않으면 에러를 반환

# join_nulls 옵션을 추가할 경우 null 값에 대한 조인을 수행
# 기본적으로 null 값은 join하지 않음
```


<br/>

---
프로젝트에서 큰 데이터를 처리해야해서 사용을 해보게 되었는데 Pandas에 비해 좋은 것 같다.<br/>
다만 Polars에 적응을 해야하는 부분도 많고, 모르는 것이 많은데 개발 중 막히게 되면 자료가 많지 않아 스스로 해결해야하는 경우가 생긴다.<br/>
그래도 공식 docs가 잘 정리되어 있고, 예시도 꽤 잘 만들어져있는 편이라 학습 난이도가 그렇게 높지는 않은 것 같다.<br/>