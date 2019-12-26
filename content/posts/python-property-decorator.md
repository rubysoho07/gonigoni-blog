---
title: "Python `@property` 데코레이터와 getter/setter"
categories: Python
tags: [Python, property, getter, setter]
draft: true
date: 2018-05-18T23:36:00+09:00
---

예전에 어떤 소스를 보다가 클래스 내부에 `@property` 데코레이터가 붙어 있는 경우를 본 적이 있다. 어떤 경우에 사용하는지 궁금해서 한 번 정리해 보려 한다.

### property() 함수

먼저 property() 함수에 대해 살펴보자.

```
class property(fget=None, fset=None, fdel=None, doc=None)
```

페이지 아래에 링크한 자료를 살펴보면, property 함수는 property 속성을 반환한다고 나와 있다. 그리고 fget 인자에는 속성 값에 대한 getter 함수, fset 인자에는 속성 값에 대한 setter 함수, fdel 인자에는 속성 값을 지우는 함수를 지정한다고 한다. 그리고 doc 인자는 속성에 대한 docstring을 넣어주면 된다.

예제를 살펴보자. 

```python
class C:
    def __init__(self):
        self._x = None

    def getx(self):
        return self._x

    def setx(self, value):
        self._x = value

    def delx(self):
        del self._x

    x = property(getx, setx, delx, "I'm the 'x' property.")
```

C 클래스의 인스턴스 c가 있다고 가정하자. `c.x`를 하면 getter인 getx 함수가 호출될 것이고, `c.x = (임의의 값)`을 수행하면 setter인 setx를 호출할 것이다. 그리고 del c.x를 수행하면 deleter인 delx가 호출될 것이다.

### `@property` decorator

아래에 있는 예제를 살펴보자. 

```python
class Parrot:
    def __init__(self):
        self._voltage = 100000

    @property
    def voltage(self):
        """Get the current voltage."""
        return self._voltage
```

Parrot 클래스의 voltage 부분을 살펴보자. `@property` 데코레이터가 붙어있음을 알 수 있다. 만약 Parrot 클래스의 인스턴스를 만들고, 아래와 같이 voltage 값을 조회하도록 하면 다음과 같은 결과를 볼 수 있을 것이다. 

```python
>>> p = Parrot()
>>> p.voltage
100000
```

그러면 voltage property에 새로운 값을 넣으려고 하면 어떻게 될까?

```python
>>> p.voltage = 1000000
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: can't set attribute
```

위와 같이 `can't set attribute`라는 오류 메시지를 볼 수 있다. 그렇다면, 새로운 값을 설정할 수 있게 만드려면 어떻게 해야 할까?

이 property에 setter를 지정해 주면 된다. 아래 예제에서 `@voltage.setter`가 붙은 부분에 주목하자.

```python
class Parrot:
    def __init__(self):
        self._voltage = 100000
    
    @property
    def voltage(self):
        """Get the current voltage."""
        return self._voltage
    
    @voltage.setter
    def voltage(self, value):
        """Set the new voltage."""
        self._voltage = value
```

이렇게 `@(property 이름).setter`를 지정하면 setter를 만들 수 있다. 원래의 property와 동일한 이름의 함수를 추가해야 함에 유의하자. (위의 예제에서는 `voltage`)

아래와 같이 테스트 해 보면 정상적으로 작동하는 것을 알 수 있다.

```python
>>> p.voltage = 1000000
>>> p.voltage
1000000
```

deleter 또한 `@(property 이름).deleter`로 지정할 수 있다.

```python
@voltage.deleter
def voltage(self):
    del self._voltage
```

### 참고자료

[https://docs.python.org/3/library/functions.html#property](https://docs.python.org/3/library/functions.html#property)
