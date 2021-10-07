---
title: "Python Datetime으로 서로 다른 시간대 다루기"
categories: [Python]
tags: [Python, datetime, 시간대]
draft: false
date: 2018-05-17T22:06:00+09:00
comments: true
---

여러 국가에 걸쳐 시간과 관련된 데이터를 얻을 때 시간대가 가장 이슈가 될 것이다. 그리고 AWS와 같이 UTC를 기준으로 하는 서비스가 있기 때문에, 우리나라의 시간대를 기준으로 프로그램을 만들 때 고민이 되는 부분이 있다. 

시간대를 다루기 위해 pytz와 같은 라이브러리가 있지만, 이번에는 파이썬이 기본적으로 제공하는 datetime 모듈만 이용해서 시간대를 다루어 보고자 한다.

### 현재 시간 구하기 / UTC 기준 시간 구하기

현재 시간을 구하기 위해서는 `datetime.datetime.now()`를 이용한다.

```python
>>> import datetime
>>> datetime.datetime.now()
datetime.datetime(2018, 5, 17, 21, 43, 13, 554579)
```

한편, UTC 기준의 현재 시간을 구하기 위해서는 `datetime.datetime.utcnow()`를 이용한다.

```python
>>> datetime.datetime.utcnow()
datetime.datetime(2018, 5, 17, 12, 43, 19, 694917)
```

위의 값과 비교하면 UTC 기준 시간은 한국 시간보다 9시간 느림을 알 수 있다. 글 쓴 시점으로 보면 한국은 밤 10시를 향해 가지만 UTC 시간대에 있는 나라들은 오후 1시를 향해 간다.

### datetime 인스턴스에 시간대 적용시키기

이렇듯 datetime을 이용하여 날짜와 시간을 저장할 수 있다. 하지만 이렇게 만든 인스턴스는 시간대 정보를 가지고 있지 않다. (참고로 datetime 관련 글을 찾아보면 이런 경우에 `naive`라는 표현을 많이들 쓰는 것 같다. 반대로 시간대 정보가 있으면 `aware`라는 표현을 쓰는 경우가 많더라.) 

일단 글을 쓴 시점을 저장해 보자. 아래와 같이 시간대 정보가 저장되어 있지 않다.

```python
>>> test_datetime = datetime.datetime.now()
>>> test_datetime
datetime.datetime(2018, 5, 17, 21, 53, 0, 629629)
```

시간대 정보를 적용하기 위해 `replace()` 함수를 이용한다. `tzinfo`에 시간대를 지정해 준다. 한국 시간은 UTC와 9시간 차이가 나므로, 아래와 같이 시간대를 지정해 준다.

```python
>>> test_datetime_kst = test_datetime.replace(tzinfo=datetime.timezone(datetime.timedelta(hours=9)))
>>> test_datetime_kst
datetime.datetime(2018, 5, 17, 21, 53, 0, 629629, tzinfo=datetime.timezone(datetime.timedelta(0, 32400)))
```

표시되는 시간은 그대로이고, tzinfo 안에 timezone 정보가 들어가 있음을 알 수 있다.

### 한국 시간대가 적용된 datetime 인스턴스를 UTC로 바꾸기

앞에서 만들어 본 datetime 인스턴스는 한국 시간대가 적용되어 있다. 이를 UTC 기준으로 변경하려면 어떻게 해야 할까? datetime의 astimezone() 함수를 이용한다. 인자로는 UTC를 의미하는 `datetime.timezone.utc`를 넣어준다.

```python
>>> test_datetime_utc = test_datetime_kst.astimezone(datetime.timezone.utc)
>>> test_datetime_utc
datetime.datetime(2018, 5, 17, 12, 53, 0, 629629, tzinfo=datetime.timezone.utc)
```

### 참고자료

[https://docs.python.org/3/library/datetime.html](https://docs.python.org/3/library/datetime.html)