---
layout: post
title: "Python의 global과 nonlocal 키워드: 차이, 동작 원리, 안티패턴과 대안"
author: 'Juho'
date: 2026-06-24 00:00:00 +0900
categories: [Python]
tags: [Python, Function, Dev]
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
2. [스코프 모델과 이름 해석 순서](#스코프-모델과-이름-해석-순서)
   - [할당이 변수를 지역으로 만든다](#할당이-변수를-지역으로-만든다)
   - [LEGB 해석 순서](#legb-해석-순서)
3. [global과 nonlocal의 정의와 차이](#global과-nonlocal의-정의와-차이)
   - [global 문](#global-문)
   - [nonlocal 문](#nonlocal-문)
   - [비교 테이블](#비교-테이블)
4. [에러 조건: SyntaxError와 UnboundLocalError](#에러-조건-syntaxerror와-unboundlocalerror)
5. [실무 패턴: 클로저, 카운터, 데코레이터](#실무-패턴-클로저-카운터-데코레이터)
6. [안티패턴 논쟁과 대안](#안티패턴-논쟁과-대안)
7. [도입 배경: PEP 3104와 Python 3.0](#도입-배경-pep-3104와-python-30)
8. [결론](#결론)
9. [Reference](#reference)

## 개요

Python에서 함수 안의 변수에 값을 할당하면, 그 변수는 자동으로 지역(local) 변수로 간주된다.
이 규칙 때문에 함수 바깥의 변수를 "수정"하려는 시도가 의도와 다르게 동작하거나 `UnboundLocalError`로 이어지는 경우가 흔하다.
`global`과 `nonlocal`은 바로 이 지역화 규칙을 명시적으로 깨고, "지금 바깥 스코프의 값을 바꾸는 것이지 새 지역 변수를 만드는 게 아니다"라는 의도를 선언하는 장치다.

두 키워드는 가리키는 스코프가 다르다.
`global`은 식별자를 모듈 레벨(전역) 네임스페이스의 이름으로 해석시키고, `nonlocal`은 가장 가까운 인접 외부(enclosing) 함수 스코프의 이름을 가리키게 한다.
이 글에서는 Python 공식 언어 레퍼런스와 PEP 3104를 기준으로 두 키워드의 동작 원리, 에러 발생 조건, 실무 패턴, 그리고 커뮤니티에서 반복되는 안티패턴 논쟁과 대안을 정리한다.

## 스코프 모델과 이름 해석 순서

### 할당이 변수를 지역으로 만든다

Python의 실행 모델은 "이름을 바인딩하는 연산이 블록 내 어디에든 존재하면 그 이름은 해당 블록의 지역 변수가 된다"고 규정한다.
공식 문서의 원문은 다음과 같다.

> "If a name binding operation occurs anywhere within a code block, all uses of the name within the block are treated as references to the current block. This can lead to errors when a name is used within a block before it is bound."

여기서 "바인딩 연산"에는 단순 대입뿐 아니라 함수의 형식 매개변수, 함수/클래스 정의, `for` 루프 헤더, `with ... as`, `except ... as`, `import` 문, `del` 타깃 등이 포함된다.
즉, 함수 본문 어딘가에 `x = ...`가 있으면 그 함수 전체에서 `x`는 지역 변수로 취급되며, 동명의 외부 변수를 가린다(shadowing).

반대로 단순 참조만 하는 변수는 free variable로 취급되어 바깥 스코프의 값을 읽을 수 있다.
free variable의 이름 해석은 컴파일 타임이 아니라 런타임에 일어난다.

```python
i = 10
def f():
    print(i)
i = 42
f()   # 42 출력
```

`f`가 정의된 시점이 아니라 호출되는 시점의 전역 `i` 값(42)을 읽는다.

### LEGB 해석 순서

이름이 사용될 때 Python은 가장 가까운 enclosing 스코프부터 차례로 해석한다.
공식 문서는 "LEGB"라는 약어를 직접 사용하지 않지만, 다음 규칙으로 동일한 순서를 정의한다.

> "When a name is used in a code block, it is resolved using the nearest enclosing scope."

> "The global namespace is searched first. If the names are not found there, the builtins namespace is searched next."

정리하면 Local(현재 블록) → Enclosing(중첩된 바깥 함수들) → Global(모듈 네임스페이스) → Built-in(`builtins` 모듈) 순서다.
전역 네임스페이스를 빌트인보다 먼저 검색한다.
이 순서를 흔히 LEGB라고 부르지만, 이는 커뮤니티 통용 약어이며 공식 문서상으로는 "nearest enclosing scope" 및 "global namespace first, then builtins" 표현으로 규정된다는 점을 함께 알아두면 정확하다.

## global과 nonlocal의 정의와 차이

### global 문

문법은 다음과 같다.

```
global_stmt: "global" identifier ("," identifier)*
```

공식 문서의 정의 원문은 다음과 같다.

> "The global statement causes the listed identifiers to be interpreted as globals. It would be impossible to assign to a global variable without global, although free variables may refer to globals without being declared global."

즉 `global` 없이도 전역 변수를 읽는 것(free variable 참조)은 가능하지만, 함수 내부에서 전역 변수에 대입(재바인딩)하려면 반드시 `global` 선언이 필요하다.
`global` 문은 현재 스코프 전체(모듈, 함수 본문, 클래스 정의 전체)에 적용된다.

전역에 해당 변수가 미리 존재할 필요는 없다.
함수 안에서 `global x` 후 처음으로 `x = ...` 하면 전역 변수가 새로 생성된다.

```python
var1 = 10
def fun():
    global var1
    var1 = var1 + 20
    print('var1 is', var1)
fun()  # var1 is 30
```

`global` 없이 `var1 = 20`만 쓰면 지역 변수만 만들어지고 전역은 10으로 그대로 남는다.

### nonlocal 문

문법은 다음과 같다.

```
nonlocal_stmt: "nonlocal" identifier ("," identifier)*
```

공식 문서의 정의 원문은 다음과 같다.

> "When the definition of a function or class is nested (enclosed) within the definitions of other functions, its nonlocal scopes are the local scopes of the enclosing functions. The nonlocal statement causes the listed identifiers to refer to names previously bound in nonlocal scopes. It allows encapsulated code to rebind such nonlocal identifiers."

즉 중첩된 함수에서 바깥쪽(enclosing) 함수의 지역 스코프에 이미 바인딩된 이름을 가리키게 하고, 그 이름을 재바인딩할 수 있게 한다.
이름이 여러 nonlocal 스코프에 바인딩돼 있으면 가장 가까운(nearest) 바인딩이 사용된다.

`nonlocal`은 전역 변수에는 접근할 수 없다.
오직 비전역 외부 스코프, 즉 중첩 함수의 바깥 함수 지역 변수에만 동작한다.
또한 대상 이름이 바깥 함수 스코프에 이미 존재해야 한다.

```python
def fun():
    var1 = 10
    def gun():
        nonlocal var1
        var1 = var1 + 10
        print(var1)
    gun()
fun()  # 20 출력
```

`nonlocal` 없이 `var1 = 20`만 쓰면 내부 함수가 자기 지역 변수를 만들어 바깥값은 10으로 유지된다(shadowing).

### 비교 테이블

두 키워드의 차이를 공식 언어 레퍼런스 기준으로 정리하면 다음과 같다.

| 항목 | global | nonlocal |
| --- | --- | --- |
| 가리키는 대상 | 모듈(전역) 네임스페이스 | 가장 가까운 enclosing 함수 스코프 |
| 적용 범위 | 모듈/함수/클래스 본문 전체 | 함수/클래스 본문 전체(중첩 함수 안) |
| 사전 바인딩 필요 | 불필요(전역 신규 생성 가능) | 필요(외부에 이미 존재해야 함) |
| 대상 바인딩 부재 시 | 에러 아님(전역 신규 생성/참조) | SyntaxError(컴파일 타임) |
| 모듈 레벨 사용 | 효과 없음(이미 전역) | 사용 불가, SyntaxError |
| 전역 변수 접근 | 가능 | 불가 |
| 선언 전 사용/대입 | SyntaxError | SyntaxError |
| 도입 버전 | 이전부터 존재 | Python 3.0, PEP 3104 |

## 에러 조건: SyntaxError와 UnboundLocalError

`global`과 `nonlocal`은 모두 파서 디렉티브(parser directive)다.
같은 시점에 파싱되는 코드에만 적용되며, `exec()`, `eval()`, `compile()`에 넘긴 문자열 코드에는 영향을 주지 않는다.
또한 두 키워드 모두 선언보다 먼저 그 변수를 사용하거나 대입하면 `SyntaxError`가 발생한다.

`nonlocal`의 제약이 가장 까다롭다.
바깥 함수 스코프에 대상 이름이 바인딩돼 있지 않거나, nonlocal 스코프 자체가 없으면 컴파일 타임에 `SyntaxError`가 발생한다.

> "If a name is not bound in any nonlocal scope, or if there is no nonlocal scope, a SyntaxError is raised."

따라서 모듈 레벨이나 중첩되지 않은 함수에서는 `nonlocal`을 쓸 수 없으며, 전역만 존재하고 enclosing 함수에는 없는 이름에도 쓸 수 없다.
PEP 3104는 추가로 형식 매개변수와 이름이 충돌하면 `SyntaxError`이며, 좌변에는 식별자만 허용되어 `x[0]` 같은 식은 불가하다고 명시한다.

반면 가장 흔하게 마주치는 런타임 에러는 `UnboundLocalError`다.
외부에 `x = 10`이 있어도 함수 안에서 `print(x)` 다음에 `x += 1`처럼 할당이 있으면, 컴파일러가 `x`를 지역으로 판정한다.
그 결과 `print(x)` 시점에 아직 값이 없는 지역 변수를 참조하게 되어 `UnboundLocalError: cannot access local variable 'x' where it is not associated with a value`가 발생한다.
특히 증강 대입(`x += 1`)은 `x = x + 1`이라 읽기가 먼저인데 이미 지역으로 확정되었기 때문에 초보자가 가장 자주 겪는 함정이다.

```python
x = 10
def foo():
    print(x)   # UnboundLocalError 발생
    x += 1

def foobar():
    global x
    print(x)   # 10
    x += 1     # 전역 x 갱신 -> 11

def foo_nested():
    x = 10
    def bar():
        nonlocal x
        print(x)   # 10
        x += 1     # 외부 함수 x 갱신 -> 11
    bar()
    print(x)       # 11
```

공식 FAQ는 이 동작이 의도된 설계라고 설명한다.
모든 전역 참조에 `global`을 요구하면 내장 함수나 임포트한 모듈마다 써야 해서 선언의 의미가 사라진다.
그래서 "할당이 일어나는 변수"에만 명시를 요구해 의도치 않은 부수효과를 막는다는 것이다.

## 실무 패턴: 클로저, 카운터, 데코레이터

`nonlocal`의 대표적인 정당한 용도는 클로저로 상태를 캡슐화하는 것이다.
전역을 오염시키지 않으면서 함수 내부에 상태를 보관할 수 있어 카운터, 메모이제이션, 데코레이터에서 흔히 쓰인다.

```python
def make_counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    return increment

c = make_counter()
c()  # 1
c()  # 2
```

`make_counter()`가 `count`를 초기화하고 내부 `increment()`가 `nonlocal`로 그 값을 갱신·캡슐화한다.
오직 `increment`만 `count`를 바꿀 수 있으므로 상태가 함수 경계 안에 안전하게 갇힌다.

함수형 데코레이터는 함수 객체를 인자로 받아 기능을 확장한 또 다른 함수 객체(클로저)를 반환한다.
호출 횟수 카운팅처럼 상태를 유지하는 데코레이터에서 `nonlocal`로 카운터를 갱신하는 패턴이 자주 사용된다.
로깅, 접근 제어, 타이밍 측정, 메모이제이션이 대표적인 용도다.

여기서 주의할 점이 하나 있다.
`nonlocal`은 이름을 재대입할 때만 필요하다는 사실이다.
바깥 변수를 읽기만 한다면 키워드 없이도 enclosing 변수에 접근할 수 있으므로, 입문자가 `nonlocal`을 과도하게 붙이는 것은 흔한 오해다.
실제로 `nonlocal`이 없던 Python 2에서는 가변 컨테이너(리스트나 딕셔너리)에 상태를 담아 "재대입이 아닌 내부 변경"으로 우회했다.
다만 리스트라도 `a += b`는 `__iadd__`로 재대입이 일어나므로 여전히 `nonlocal`이나 `a.extend(b)`가 필요하다는 미묘한 지점이 있다.

## 안티패턴 논쟁과 대안

커뮤니티에서는 두 키워드 자체를 "나쁜 기능"으로 보지 않는다.
오히려 "함수 안에서 대입하면 그 이름은 지역이 된다"는 규칙을 명시적으로 깨는 정직한 장치라는 평가가 많다.
다만 `global`에 대해서는 "전역 상태는 안티패턴"이라는 정서가 지배적이다.

`global`이 비판받는 이유는 다음과 같다.

| 문제 | 설명 |
| --- | --- |
| 숨은 의존성 | 전역 변수는 함수 시그니처에 드러나지 않아 의존 관계가 코드에 숨는다 |
| 타이밍 버그 | 한 함수가 값을 세팅하기 전에 다른 함수가 읽어버리는 경합이 발생한다 |
| 추적 비용 | 변수가 어디서 수정되는지 여러 함수와 모듈을 뒤져야 해 가독성을 해친다 |
| 테스트 곤란 | 전역 상태를 테스트마다 관리·리셋해야 하고 결합도가 높아진다 |

권장되는 대안은 크게 세 가지로 모인다.
첫째, 인자로 받고 값을 반환하는 순수 함수 스타일로 작성한다.
둘째, 관련 전역들을 클래스의 멤버로 보관해 객체로 캡슐화한다.
셋째, 중첩 함수라면 클로저와 `nonlocal`로 상태를 좁은 스코프에 가둔다.
이 외에도 의존성 주입, 설정 객체(config object), 컨텍스트 매니저(`with`)로 전역 상태 의존을 제거하는 방식이 거론된다.

다만 균형론도 존재한다.
환경에 진짜 단 하나만 존재하는 엔티티, 예를 들어 보안 정책이나 표준 출력(stdout) 같은 단일 자원에는 전역이 자연스럽다는 옹호다.
"global이 무조건 나쁘다는 건 단순화된 의견"이라는 시각과 함께, "전역 변수를 피할 수 있다면 그것은 피하는 게 낫다는 신호"라는 절충된 합의가 자주 인용된다.

`nonlocal`도 무제한 허용되는 것은 아니다.
자주 쓰게 된다면 코드 구조를 개선할 수 있다는 신호이며, 중첩이 깊어지면 가독성이 떨어지고 숨은 상태가 늘어 차라리 클래스로 리팩터링하라는 경고가 반복된다.

## 도입 배경: PEP 3104와 Python 3.0

`nonlocal`은 Python 3.0에서 PEP 3104("Access to Names in Outer Scopes")로 도입되었다.
`global`은 Python 1.x부터 존재해 왔다.

PEP 3104가 해결한 문제는 다음과 같다.

> "code can refer to a name in any enclosing scope, but it can only rebind names in two scopes: the local scope (by simple assignment) or the module-global scope (using a global declaration)."

PEP 227(Python 2.1)이 바깥 스코프의 이름을 읽는 것은 가능하게 했으나, 재바인딩은 지역 스코프와 모듈 전역 스코프 두 곳에서만 가능했다.
PEP 3104는 그 사이의 중첩 함수 스코프를 재바인딩할 수 없던 공백을 메운다.

키워드 이름으로 `nonlocal`이 선택된 근거도 흥미롭다.
후보였던 `outer`는 표준 라이브러리에 147회 등장해 충돌 위험이 컸던 반면, `nonlocal`은 0회 등장했고 "declares a name not local"이라는 의미를 정확히 표현했다.
이는 `global`과 대칭되는 "내부 스코프에서의 scope override declaration"으로 설계되었으며, JavaScript, Perl, Ruby, Scheme, Smalltalk 등 중첩 스코프 재바인딩을 지원하는 언어들과의 정합성 확보가 동기였다.

## 결론

`global`과 `nonlocal`은 Python의 "할당하면 지역화" 규칙을 명시적으로 깨기 위한 선언이다.
`global`은 모듈 전역 네임스페이스를 가리키며 대상이 없어도 새로 만들 수 있는 반면, `nonlocal`은 가장 가까운 enclosing 함수 스코프를 가리키며 대상이 미리 존재하지 않으면 컴파일 타임에 `SyntaxError`를 낸다.
가장 흔한 함정인 `UnboundLocalError`는 함수 안의 할당이 변수를 지역으로 확정시키기 때문에 발생하며, 적절한 선언이나 인자 전달로 해결한다.

실무에서는 `nonlocal`을 카운터, 데코레이터, 메모이제이션 같은 클로저 상태 캡슐화에 활용하되, 재대입이 필요한 경우에만 쓰는 것이 권장된다.
`global`은 단일 자원에 한해 정당할 수 있으나, 숨은 의존성과 타이밍 버그를 피하기 위해 가능하면 인자/반환값, 클래스 캡슐화, 클로저로 대체하는 것이 커뮤니티의 합의다.

## Reference

- [The global statement / The nonlocal statement (Python Language Reference)](https://docs.python.org/3/reference/simple_stmts.html#the-global-statement)
- [Execution model — Naming and binding (Python Language Reference)](https://docs.python.org/3/reference/executionmodel.html)
- [Programming FAQ (Python Documentation)](https://docs.python.org/3/faq/programming.html)
- [PEP 3104 — Access to Names in Outer Scopes](https://peps.python.org/pep-3104/)
- [Use of nonlocal vs global keyword in Python (GeeksforGeeks)](https://www.geeksforgeeks.org/python/use-of-nonlocal-vs-use-of-global-keyword-in-python/)
- [Using the global statement — Python Anti-Patterns](https://docs.quantifiedcode.com/python-anti-patterns/maintainability/using_the_global_statement.html)
- [Python Closures (Real Python)](https://realpython.com/python-closure/)
- [Understanding UnboundLocalError in Python (Eli Bendersky)](https://eli.thegreenplace.net/2011/05/15/understanding-unboundlocalerror-in-python/)
- [Lists and Nonlocal (Kavi Gupta)](https://kavigupta.org/2018/10/03/Lists-And-Nonlocal/)
