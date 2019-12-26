---
title: "몰라서 찾아 봤던 파이썬 내장 함수 모음"
categories: Python
tags: [Python, 내장 함수]
draft: false
date: 2018-05-16T23:27:00+09:00
---

파이썬을 쓰다가 필요한 기능이 있어서 찾아 보면, 언어 자체에서 지원하는 기능인 경우가 종종 있었다. 지금까지 파이썬을 다루면서 찾아봤던 내장 함수들을 정리해 본다.

아래에서 계속 iterable이라는 말을 많이 쓸 것이다. iterable에는 sequence type인 list, str, tuple 뿐만 아니라 non-sequence type인 dict, file object, 그리고 Sequence semantics를 구현하기 위해 `__iter__()`나 `__getitem__()` method가 정의되어 있는 클래스의 object 등이 있다. [(참고)](https://docs.python.org/3/glossary.html)

### sum(): 각각의 item을 합치기

참고: [https://docs.python.org/3/library/functions.html#sum](https://docs.python.org/3/library/functions.html#sum)

```
sum(iterable[, start])
```

iterable 객체를 받아서 각 요소들의 합을 구한다. start는 적어주지 않으면 0으로 정해진다. 다음과 같이 사용할 수 있다.

```python
>>> test = [1, 2, 3, 4]
>>> sum(test)
10
```

### update(): 2개의 dictionary를 합치고 싶을 때

참고: [https://docs.python.org/3/library/stdtypes.html#dict.update](https://docs.python.org/3/library/stdtypes.html#dict.update)

```
update([other])
```

두 개의 dictionary를 합치고 싶을 때 이용한다. 원본 dictionary에 동일한 key가 있으면 other에 있는 값으로 덮어 쓴다. 다른 dictionary 뿐만 아니라 key/value 쌍의 iterable을 넣을 수 있다. 

```python
>>> d = {'a': 1, 'b': 2}
>>> d.update({'b': 3, 'c': 4})
>>> d
{'a': 1, 'b': 3, 'c': 4}
>>> d.update(c=5, d=6)
>>> d
{'a': 1, 'b': 3, 'c': 5, 'd': 6}
```

### sorted(): iterable 객체로부터 정렬된 리스트를 얻기

참고: [https://docs.python.org/3.6/library/functions.html#sorted](https://docs.python.org/3.6/library/functions.html#sorted)

```
sorted(iterable, *, key=None, reverse=False)
```

iterable에는 iterable 객체가 들어갈 수 있다. key에는 각각의 요소에서 비교를 위한 키를 추출하기 위해 사용되는 하나의 argument를 받는 함수를 지정한다. 기본값은 None이다. reverse는 boolean 값으로, True인 경우 반대 순서로 정렬하게 된다. 

```python
>>> test_list = [4, 7, 1, 9, 2, 3]
>>> sorted(test_list)
[1, 2, 3, 4, 7, 9]
>>> sorted(test_list, reverse=True)
[9, 7, 4, 3, 2, 1]
```

### type(): 객체의 type 확인하기

참고: [https://docs.python.org/3.6/library/functions.html#type](https://docs.python.org/3.6/library/functions.html#type)

```
type(object)
```

말 그대로 객체의 type을 확인할 수 있는 함수이다. 

```python
>>> type(1)
<class 'int'>
>>> type('a')
<class 'str'>
>>> type([1,2,3])
<class 'list'>
>>> type({'a': 1, 'b': 2})
<class 'dict'>
```

### any(), all(): 여러 조건 한 번에 확인하기

참고
* [https://docs.python.org/3/library/functions.html#any](https://docs.python.org/3/library/functions.html#any)
* [https://docs.python.org/3/library/functions.html#all](https://docs.python.org/3/library/functions.html#all)

두 함수 모두 iterable을 받는다. 차이점은 다음과 같다.

any(): iterable 객체 안의 하나라도 True이면 True를 반환한다. 빈 객체이면 False를 반환한다.
all(): iterable 객체 안의 모든 요소가 True이면 True를 반환한다. 빈 객체인 경우에도 True를 반환한다.

```python
>>> any([True, False, True])
True
>>> all([True, False, True])
False
>>> any([])
False
>>> all([])
True
```

### zip(): 여러 iterable들의 요소를 합쳐서 iterator 만들기

참고: [https://docs.python.org/3/library/functions.html#zip](https://docs.python.org/3/library/functions.html#zip)

설명만 들어서는 어떨 때 써야 할 지 애매할 수도 있다. 예제를 통해 알아보자.
a라는 리스트와 b라는 리스트가 있다고 가정하자. 그런데 두 리스트에 있는 요소를 하나씩 꺼내서 뭔가를 하고 싶은 경우가 있다. zip() 함수는 이러한 경우에 사용할 수 있다.

```python
>>> a_list = [1,2,3,4]
>>> b_list = ['a','b','c']
>>> for a, b in zip(a_list, b_list):
...     print(a, b)
...
1 a
2 b
3 c
```

참고로 길이가 짧은 iterable을 기준으로 하므로, 이에 유의하자. 길이가 긴 iterable을 기준으로 하고 싶은 경우, itertools 모듈의 zip_longest() 함수를 이용한다.

```python
>>> import itertools
>>> for a, b in itertools.zip_longest(a_list, b_list):
...     print(a, b)
...
1 a
2 b
3 c
4 None
```

(참고) 이 글은 앞으로도 계속 업데이트 할 예정이다. 