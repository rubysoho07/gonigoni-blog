---
title: "파이썬에서 테스트 코드를 만들면서 겪은 일들"
categories: Python
tags: [Python, 테스트]
draft: false
date: 2018-05-15T22:35:00+09:00
---

이번 글에서는 파이썬에서 테스트 코드를 만들면서 겪었던 시행착오에 대해 소개하고자 한다. 여기서는 파이썬에 내장된 `unittest`를 이용한다.

### 테스트를 수행하는 기준은?

간단한 코드를 만들어 보자. 파일 이름은 `aaa.py`로 저장한다.

```python
import unittest

class TestExample(unittest.TestCase):
    def test_a(self):
        self.assertEqual(1 == 2, False)
```

그리고 쉘에서 `python -m unittest`를 입력하면, 아무 테스트도 실행하지 않는다.

```
$ python -m unittest

----------------------------------------------------------------------
Ran 0 tests in 0.000s

OK
```

왜 그럴까? `unittest` 라이브러리는 기본적으로 실행하는 top-level 디렉터리에서 `test`로 시작하는 파일을 찾아서 테스트를 수행한다. [(참고)](https://docs.python.org/3/library/unittest.html#test-discovery) 일단 파일 이름을 test로 시작하도록 바꾼 뒤 아래와 같이 테스트해 보자. 

```
$ mv aaa.py test.py
$ python -m unittest
.
----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
```

위와 같이 테스트가 수행된 것을 볼 수 있다.

개인적으로는 Java의 Maven을 사용할 때와 같이, 소스 파일과 테스트 파일을 별도의 디렉터리에 저장하는 방식을 선호한다. 

### PyCharm에서 테스트 실행하기

나는 파이썬으로 개발할 때 주로 PyCharm을 이용한다. PyCharm에서 테스트 코드를 실행할 때 설정한 방법에 대해 정리해 보려 한다.

아래 스크린샷과 같은 화면은 메뉴에서 Run -> Edit Configurations 메뉴를 클릭하면 확인할 수 있다. 

아래와 같은 화면이 나오지 않는 경우, 왼쪽 위의 `+`버튼을 누르고, `Python Tests` -> `Unittests`를 클릭해서 추가해 준다.
<figure>
    <img src="{{ "media/img/python-making-test-module-screenshot-1.png" | absolute_url }}">
    <figcaption>PyCharm 테스트 설정</figcaption>
</figure>

* Target: Script Path를 선택하고 테스트 코드가 있는 곳을 지정해 준다.
* Pattern: 테스트 파일의 패턴을 정해주는데, 파일 이름이 `test`로 시작한다면 굳이 적을 필요는 없다.
* Environment Variables: 테스트에 필요한 환경변수를 적는다. `...`을 누르면 별도의 창이 뜨는데, Key와 Value를 적어주면 된다.
* Python Interpreter: 사용하는 파이썬 버전과 가상환경에 맞게 설정하면 된다.
* Working Directory: 테스트를 시작할 때 어느 디렉터리에서 실행할 지를 적는다. 다른 설정보다 이 부분 때문에 삽질한 적이 좀 있었다.

### 참고자료

[https://docs.python.org/3/library/unittest.html](https://docs.python.org/3/library/unittest.html)
