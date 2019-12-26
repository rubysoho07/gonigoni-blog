---
title: "Python underscore 관련 naming convention"
categories: Python
tags: [Python, underscore, pep-8]
draft: true
date: 2018-05-26T10:32:00+09:00
---

> 파이썬에는 [PEP-8](https://www.python.org/dev/peps/pep-0008/)이라는 Python Code Style Guide가 있다. 모든 내용을 한 번에 정리하기 어려워서 그냥 코딩하다가 궁금한 것 위주로 정리해 보려고 한다.

이번 글에서는 '_'(Underscore)가 붙는 변수나 메서드에 관련한 내용을 정리하려고 한다.

### Trailing underscore

* `_single_leading_underscore`: 이런 경우 `from M import *`에서는 `_`로 시작하는 object를 import 하지 않는다.
* `single_trailing_underscore_`: 파이썬 언어 내에서 사용되는 키워드와의 충돌을 피하기 위해 작성한다. BeautifulSoup에서 HTML의 class를 기준으로 무언가를 찾을 때 `class_`와 같이 속성을 지정해 줄 때가 있다. 
* `__double_leading_underscore`: 클래스 속성의 이름을 정할 때 name mangling을 피한다. 아래에 그 예가 있으니 참고하면 될 것 같다.
* `__double_leading_and_trailing_underscore__`: 사용자가 컨트롤하는 네임스페이스에 있는 "magic" object나 속성이다. 예를 들어 `__init__`, `__import__`, `__file__`등이 있다. 이러한 이름을 새로 만들 일은 거의 없을 것 같다.

### Package and Module Names 

파이썬 모듈에서 C나 C++로 작성한 확장 모듈을 이용할 때, C나 C++ 모듈은 앞에 `_`를 하나 붙여준다. PEP-8 문서에서는 `_socket`을 예로 들고 있다.

### Global Variable Names

`from M import *`을 사용하여 모듈을 import 하는 경우, `__all__` 변수를 사용해서 특정한 global 변수를 외부에서 쓰지 못 하도록 막을 수 있다.

```python
# M.py
__all__ = [aaa, bbb]

aaa = 1
def bbb():
    return 'bbb'

def ccc():
    return 'ccc'
```

아래와 같이 M 모듈을 import 하고 ccc를 실행하면 예외가 발생한다.
```python
from M import *

print(aaa)
print(bbb)

# 여기서 예외가 발생한다.
print(ccc)
```

### Method Names and Instance Variables

* 앞에 1개의 `_`를 붙이는 경우: non-public 메소드/인스턴스 변수임을 의미한다.
* 앞에 2개의 `_`를 붙이는 경우: 상속된 클래스와 이름이 충돌하는 것을 막기 위해 사용한다.

> 참고: Foo 클래스가 `__a`라는 속성이 있으면, `Foo.__a`로 접근할 수 없다. 사용자는 `Foo._Foo__a`로만 접근할 수 있다. 상/하위 클래스 간 이름이 충돌하지 않도록 구현하는 경우에만 사용해야 한다.

### Designing for Inheritance (상속을 위한 디자인)

Java나 C++과 같은 언어에서는 `public`, `private`와 같이 속성이나 메소드의 접근 가능 범위를 지정해 주는 키워드가 존재한다. 하지만 파이썬에는 그런 키워드가 없다. 대신에 다음과 같은 코딩 스타일을 이용한다.

* Public attribute는 처음에 `_`을 붙이지 않는다.
* Public attribute가 예약된 키워드와 충돌하면 뒤에 `_`를 붙인다. 
* 클래스가 상속되고, 상속된 클래스에서 사용하지 않을 속성이 있다면 2개의 `_`로 시작하고 끝에는 `_`를 붙이지 않는다. 

### 기타 다른 이야기

Python에서 `_`를 쓰는 다른 용법이 있는데, 개인적으로 몰랐던 부분이 있어서 아래 링크한 글이 많은 도움이 되었다. 

* Interpreter에서 마지막 값을 저장
* 특정한 값을 무시 (반복문이나 여러 값을 반환하는 함수에서)
* Internationalization, Localization 함수로 사용
* 숫자를 천 단위로 분리해서 표현하고 싶을 때 (예: 1,000 -> 1_000)

[https://hackernoon.com/understanding-the-underscore-of-python-309d1a029edc](https://hackernoon.com/understanding-the-underscore-of-python-309d1a029edc)
