---
title: "파이썬 문자열 다루기 - str.format()"
categories: Python
tags: [문자열, format]
draft: false
date: 2018-12-01T20:54:00+09:00
---

문자열을 다루다 보면 `.format()`을 사용하는데, .format()으로 어떻게 문자열 중간을 채워볼 지에 대해 알아보겠다. 이 글은 필요할 때 매번 찾아보다가 귀찮아서 정리해 본 것이다.

### 들어가기 전에...

혹시 C를 배워본 적이 있는지? C에는 `printf()`라는 함수가 있다. 변수의 내용을 표시하기 위해 아래 코드와 같이 `%d`, `%f`등의 문자열을 사용한 적이 있을 것이다. 

```c
int a = 0;
printf('%d', a);
```

파이썬 또한 이런 방법을 지원한다. 하지만 튜플이나 딕셔너리와 같은 자료형은 제대로 표시되지 않는 문제가 있다고 한다. [자세한 내용은 링크를 참조하자.](https://docs.python.org/3/library/stdtypes.html#old-string-formatting)

그래서, 파이썬에서는 다른 문자열 포매팅 방법을 지원한다. 바로 `str.format()`이나 `Template Strings`라는 것인데, 여기서는 `str.format()`에 대해 알아보려고 한다. 이 기능이 만들어진 배경에 대해서는 [PEP-3101](https://www.python.org/dev/peps/pep-3101/#id7)을 함께 보면 좋을 것 같다.

### 기본적인 사용법

* 순서 없이 표현하기

```python
>>> "{} {}".format("Hello", "Python")
'Hello Python'
```
기본적으로 `format()` 안에 들어간 순서대로 문자열에 들어간다.

* 순서를 지정해서 표현하기

```python
>>> "{1} {0}".format("Hello", "Python")
'Python Hello'
```
`{0}` 자리에는 처음에 넣은 `Hello`가 들어간다.
`{1}` 자리에는 두번째로 넣은 `Python`이 들어간다.

여기서 순서는 **0부터 시작함**에 유의하자. 두 개를 넣었는데 1부터 시작해서 2까지 쓰는 경우, 다음과 같은 에러를 만날 수 있다.

```python
>>> "{2} {1}".format("Hello", "Python")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: tuple index out of range
```

* 이름을 지정해서 표현하기

```python
>>> "{greeting} {name}".format(greeting="Hello", name="Python")
'Hello Python'
```
greeting에는 `Hello`, name에는 `Python`을 지정해서 전달한다.

주의해야 할 점은, 문자열에 있는 내용을 format()안에 전달하지 않으면 다음 오류를 볼 수 있다.

```python
>>> "{greeting} {name}".format(greeting="Hello")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'name'
```

### 특정 포맷에 맞게 출력하기

* 정수를 n개의 자리로 표현 (0 포함)
```python
>>> "{:03d}".format(1)
'001'
```

* 정수를 n개의 자리로 표현 (0 제외)

```python
>>> "{:3d}".format(1)
'  1'
```

* 소수 n개의 자리까지 표현하기
```python
>>> "{:.2f}".format(1.333)
'1.33'
```

* 문자열을 특정 길이까지 표현하기 (20자의 자리로 표현함) - 왼쪽 정렬 
```python
>>> "{:20}".format("test")
'test                '
```

(참고) `{:20}`대신에 `{:<20}`을 써도 똑같다.

* 문자열을 특정 길이까지 표현하기 (20자의 자리로 표현함) - 오른쪽 정렬 
```python
>>> "{:>20}".format("test")
'                test'
```

* datetime 객체에서 연, 월, 일만 가져오기

```python
>>> import datetime
>>> now = datetime.datetime.now()
>>> "Y: {} M: {} D: {}".format(now.year, now.month, now.day)
'Y: 2018 M: 12 D: 1'
```

* datetime 객체에서 시, 분, 초만 가져오기

```python
>>> import datetime
>>> now = datetime.datetime.now()
>>> "H: {} M: {} S: {}".format(now.hour, now.minute, now.second)
'H: 16 M: 23 S: 52'
```

이외에도 더 많은 내용은 [여기](https://docs.python.org/3/library/string.html#format-examples)를 참고하자.

### 참고자료

* [PEP-3101](https://www.python.org/dev/peps/pep-3101/#id7)
* [printf-style String Formatting](https://docs.python.org/3/library/stdtypes.html#old-string-formatting)
* [str.format() - Python Documentation](https://docs.python.org/3/library/stdtypes.html#str.format)
* [Format String Syntax - Python Documentation](https://docs.python.org/3/library/string.html#format-string-syntax)