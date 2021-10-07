---
title: "Python의 map() filter() reduce() 사용 방법 정리"
date: 2020-11-28T13:13:35+09:00
tags: [python, map, filter, reduce, 내장 함수]
draft: false
categories: [Python]
comments: true
---

가끔씩 Python의 `map()`, `filter()`, `reduce()`에 대해 헷갈리는 것들이 있어서 정리해 본다.

# map(function, iterable, ...)

* Reference: [Python 문서 참조](https://docs.python.org/3/library/functions.html#map)

`iterable`에 있는 모든 요소에 `function`을 적용하여 그 결과를 반환한다. `function`은 여러 인자를 받을 수 있어야 하고, 모든 iterable의 아이템에 동시에 적용되도록 해야 한다.

```python
>>> l1 = [1, 2, 3, 4]
>>> map(lambda x: x * 2, l1)
<map object at 0x1006d1040>
```

map()을 수행한 결과는 map object로 반환되므로, 이를 list나 tuple로 바꾸는 작업이 필요하다.

```python
>>> list(map(lambda x: x * 2, l1))
[2, 4, 6, 8]
```

함수 정의를 보면, `iterable` 뒤에 `...`이 붙는 것을 볼 수 있다. 여러 iterable을 붙이면 어떻게 될까?

```python
>>> l1 = [1, 2, 3, 4]
>>> l2 = [5, 6, 7, 8]
>>> list(map(lambda x, y: (x*2, y*2), l1, l2))
[(2, 10), (4, 12), (6, 14), (8, 16)]
```

위와 같이 l1, l2를 인자로 넘겨주게 되면, l1과 l2에서 각각 하나씩 가져와서 처리하게 된다. 공식 문서에서는 여러 개의 iterable이 들어오면 가장 짧은 iterable의 내용만큼만 적용한다고 되어 있다. 

```python
>>> l3 = [10, 11, 12]
>>> list(map(lambda x, y: (x * 2, y * 2), l1, l3))
[(2, 20), (4, 22), (6, 24)]
```

l3이 l1보다 길이가 짧기 때문에, l3에 있는 element 개수만큼 map이 실행된다.

# filter(function, iterable)

* Reference: [Python 문서 참조](https://docs.python.org/3/library/functions.html#filter)

`iterable`의 각 element에 대해 `function`이 True를 반환하는 element의 iterator라고 한다. 설명만 보면 감이 안 잡히니, 코드로 한 번 살펴보자.

```python
>>> l5 = [1, 2, 3, 4, 5, 6]
>>> filter(lambda x: x % 2 == 0, l5)
<filter object at 0x1006cd460>
```

filter() 함수도 역시 filter object를 반환한다. 그렇다면 리스트로 바꿔보자.

```python
>>> list(filter(lambda x: x % 2 == 0, l5))
[2, 4, 6]
```

정리해 보면 다음과 같다. 

* 이 예제에서는 function을 x 값이 짝수인지 판단하는 lambda 식으로 지정했다. 
* `l5 = [1, 2, 3, 4, 5, 6]` 중에서 짝수인 2, 4, 6을 담아서 filter object에 반환한다.  

그럼 function 부분에 `None`을 넣으면 어떻게 될까?

```python
>>> list(filter(None, l5))
[1, 2, 3, 4, 5, 6]
```

원래의 리스트를 그대로 반환한다. 감이 안 잡히니 다른 예제로 테스트 해 보자.

```python
>>> l6 = [True, False, True, False]
>>> list(filter(None, l6))
[True, True]
```

즉, function을 지정해 주지 않더라도 iterable의 각 element 중 False인 것은 제외하고 반환한다.

# reduce(function, iterable[, initializer])

Python 3으로 오면서 reduce가 내장 함수에서 빠졌다고 한다. 대신 Python에 내장된 functools 라이브러리에서 import 해서 사용할 수 있다. 

* Reference: [functools 문서 참조](https://docs.python.org/3/library/functools.html#functools.reduce)

기본적인 동작은 다음과 같다. 
* `iterable`의 각 element를 왼쪽부터 오른쪽 방향으로 function을 적용하여 하나의 값으로 합친다.
* 예를 들어 `reduce(lambda x, y: x + y, [1, 2, 3, 4, 5])`라고 한다면, `((((1+2)+3)+4)+5)`의 값을 반환한다. 
* function에서 왼쪽 인자는 누적된 값, 오른쪽 인자는 iterable로부터 업데이트에 사용할 값이라고 한다. 설명하면 다음과 같다. 
    * 처음에 전달될 값은 1, 2고 결과는 3이다.
    * 이제 3, 3이 전달되고, 결과는 6이다.
    * 다음으로는 6, 4가 전달되고, 결과는 10이다.
    * 마지막으로 10, 5가 전달되고, 결과는 15다. 
    * 이제 왼쪽부터 오른쪽까지 다 돌았으니 15를 반환한다. 

진짜 그런지 알아보자.

```python
>>> from functools import reduce
>>> reduce(lambda x, y: x+y, [1,2,3,4,5])
15
```

앞에서 설명했던 것처럼 15를 반환한다. 

근데 뒤에 붙은 initializer는 뭘까? 문서에서는 계산할 때 iterable의 item의 왼쪽에 위치하여, iterable이 빈 값일 때 기본값으로 제공된다고 한다. 

그리고 만약 initializer가 주어지지 않고, iterable이 하나의 아이템만 갖고 있으면 첫번째 아이템이 반환된다고 한다. 

먼저 initializer가 없을 때는 다음과 같이 동작한다. 리스트에 1밖에 없기 때문에 1을 반환한다.

```python
>>> reduce(lambda x, y: x+y, [1])
1
```

리스트에 1개의 값이 있는데, initializer도 있으면 다음과 같이 동작한다. initializer 값이 1이기 때문에 이 값과 리스트에 있던 값을 더해서 2를 반환한다.

```python
>>> reduce(lambda x, y: x+y, [1], 1)
2
```

그리고 iterable에 None을 넣거나 빈 리스트를 넣으면 다음과 같이 동작한다.

```python
>>> reduce(lambda x, y: x+y, None, 1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: reduce() arg 2 must support iteration
>>> reduce(lambda x, y: x+y, [], 1)
1
>>> reduce(lambda x, y: x+y, [])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: reduce() of empty sequence with no initial value
```

NoneType은 iteration이 불가능하기 때문에 TypeError가 발생하고, 빈 리스트를 iterable에 전달하면 initializer 값인 1이 반환된다. 또한 빈 리스트를 iterable에 전달했는데 initializer 값이 없으면 TypeError가 발생하게 된다. 
