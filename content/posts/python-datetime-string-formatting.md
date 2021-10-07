---
title: "Python 날짜/시간을 문자열로 만들기 위한 규칙 정리"
categories: [Python]
tags: [Python, 문자열]
draft: false
date: 2018-05-14T23:24:00+09:00
comments: true
---

파이썬을 쓰다 보면 문자열을 다루는 경우가 많은데, 문자열을 특정한 양식에 맞추어야 하는 경우가 종종 있었다. 이 경우 [PyFormat](http://pyformat.info)과 같은 사이트를 참조한다. 하지만 날짜나 시간을 formatting 할 때는 상세한 규칙이 정리되어 있지 않아서 이 참에 정리를 해 보려고 한다. 

문자열을 넣거나 숫자를 다루는 경우, PyFormat에 자세한 내용이 있어서 생략한다.

이 글에서는 현재 시간을 datetime 객체로 받고, 이를 가지고 날짜와 시간을 문자열로 formatting 해 볼 것이다. (아래 코드 참조)

```python
import datetime
now = datetime.datetime.now()
```

이 글을 쓰는 시점에서 now의 내용은 다음과 같다.

```python
>>> now
datetime.datetime(2018, 5, 14, 21, 58, 48, 136538)
```

### 날짜를 formatting 할 때 규칙

**`%Y`: 연도를 4자리로 나타내고 싶을 때 사용한다.**

방법 1) `now.strftime("%Y")`

방법 2) `"{:%Y}".format(now)`

결과는 모두 `'2018'`이다.

**`%y`: 연도를 2자리로 나타내고 싶을 때 사용한다.**

방법 1) `now.strftime("%y")`

방법 2) `"{:%y}".format(now)`

결과는 모두 `'18'`이다.

**`%m`: 월을 2자리로 나타내고 싶을 때 사용한다. 1자리 수인 경우, 앞에 0이 붙는다.**

방법 1) `now.strftime("%m")`

방법 2) `"{:%m}".format(now)`

결과는 모두 `'05'`이다.

**`%d`: 일을 2자리로 나타내고 싶을 때 사용한다. 위와 마찬가지로 1자리 수인 경우, 앞에 0이 붙는다.**

방법 1) `now.strftime("%d")`

방법 2) `"{:%d}".format(now)`

결과는 모두 `'14'`이다.

**숫자 앞에 0을 붙이고 싶지 않다면, `format` 함수를 쓰자.**

이 방법은 일, 시, 분, 초 모두의 경우에 쓸 수 있다. `strftime` 함수에서 zero-padding을 안 붙이고 쓰는 방법이 없기 때문에 이러한 방법을 쓴다. 아래 예시는 월 항목에 padding을 붙이고 싶지 않을 때 사용한 방법이다.

방법) `"{:d}".format(now.month)`

결과) `'5'`

**`%a`, `%A`: 요일을 이름으로 표시한다. (월요일, 화요일, ...)**

세 글자로 요약) `now.strftime("%a")` 또는 `"{:%a}".format(now)`

결과) `'Mon'`

전체 텍스트를 표시) `now.strftime("%A")` 또는 `"{:%A}".format(now)`

결과) `'Monday'`

**`%b`, `%B`: 월을 이름으로 표시한다. (January, February, ...)**

세 글자로 요약) `now.strftime("%b")` 또는 `"{:%b}".format(now)`

결과) `'May'`

전체 텍스트를 표시) `now.strftime("%B")` 또는 `"{:%B}".format(now)`

결과) `'May'` 

(이 글을 쓴 시점이 5월이라 'May'로 똑같이 표시되지만, 다음 코드를 실행해보면 다름을 알 수 있다.)

```python
>>> after_1_month = now + datetime.timedelta(days=30)
>>> "{:%B}".format(after_1_month)
'June'
>>> "{:%b}".format(after_1_month)
'Jun'
```

**한글 locale을 이용해서 표시하기**

우리는 한국 사람이므로 한글로 날짜/시간을 표시할 때가 있다. 이러한 경우에 대처하기 위해 locale 설정을 해 주자. 

```python
>>> import locale
>>> locale.setlocale(locale.LC_TIME,'ko_KR.UTF-8')
```

`locale.LC_TIME`을 설정한 경우, 시간 설정에 대해서만 해당 locale이 적용된다. 그리고 뒤의 `'ko_KR.UTF-8'`은 locale 설정을 한국으로 했다면 비워도 상관 없다. (빈 문자열은 사용자의 기본 설정을 따르도록 함)

그리고 위의 코드들을 실행해 보면 다음과 같은 결과들을 볼 수 있다.

```python
>>> now.strftime("%b")
' 5월'
>>> now.strftime("%B")
'5월'
>>> now.strftime("%a")
'월'
>>> now.strftime("%A")
'월요일'
```

### 시간을 formatting 할 때 규칙

**`%H`: 시간을 2자리로 나타내고 싶을 때 사용한다. 1자리 수인 경우, 앞에 0이 붙는다.** 

방법 1) `now.strftime("%H")`

방법 2) `"{:%H}".format(now)`

결과는 모두 `'21'`이다.

**`%M`: 분을 2자리로 나타내고 싶을 때 사용한다. 1자리 수인 경우, 앞에 0이 붙는다.**

방법 1) `now.strftime("%M")`

방법 2) `"{:%M}".format(now)`

결과는 모두 `'58'`이다.

**`%S`: 초를 2자리로 나타내고 싶을 때 사용한다. 1자리 수인 경우, 앞에 0이 붙는다.**

방법 1) `now.strftime("%S")`

방법 2) `"{:%S}".format(now)`

결과는 모두 `'48'`이다.

**`%f`: 마이크로초 단위를 숫자로 나타내고 싶을 때 사용한다. 기본적으로 6자리이며, 앞에 0이 붙는다.**

방법 1) `now.strftime("%f")`

방법 2) `"{:%f}".format(now)`

결과는 모두 `'136538'`이다.

(번외) **마이크로초 단위의 자리수를 바꾸고 싶다면? 그냥 문자열 슬라이싱을 이용하자.**

방법) `now.strftime("%f")[:3]`

결과는 `'136'`이다. 

### 참고 자료

* [https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior](https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior)
* [https://docs.python.org/3.6/library/locale.html#locale.setlocale](https://docs.python.org/3.6/library/locale.html#locale.setlocale)