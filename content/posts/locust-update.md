---
title: "Locust 1.0 이후 달라진 점들 요약"
date: 2021-03-30T21:53:01+09:00
tags: [Locust, 부하 테스트]
draft: false
categories: Python
comments: true
---

작년 이맘때에 AWSKRUG에서 [ECS+Locust로 부하 테스트 진행하기](https://www.slideshare.net/YungonPark/ecslocust-232571377)라는 주제로 발표를 한 적이 있었습니다. 

그러다가 5월에 Locust 1.0 버전이 나왔는데요. 최신 버전의 Locust로 부하 테스트를 진행하다 보니 바뀐 부분이 많아서, 새 버전으로 테스트를 수행하기 위해 필요한 내용 위주로 다시 정리해 보려고 합니다.

이 글은 위에 링크한 슬라이드 중 11~19 페이지의 내용을 대체합니다. 지금부터 한 번 살펴보겠습니다.

## Locust -> User 클래스 이름 변경

한 명의 사용자를 표현하는 Locust 클래스가 'User'로 이름을 바꾸었습니다. HTTP를 이용하는 클라이언트는 HttpLocust에서 HttpUser로 클래스 이름을 바꾸었습니다. (GitHub [이슈](https://github.com/locustio/locust/issues/1283) 참조)

하지만 각각의 사용자가 실행할 작업을 의미하는 TaskSet 클래스는 이름이 바뀌지 않았습니다. 

그리고 User/HttpUser 클래스에서 실행할 작업을 지정할 때, `task_set` 속성 대신에 `tasks` 속성을 지정해 주어야 합니다. 

`tasks` 속성에 들어갈 수 있는 내용은 [Python callable](https://docs.python.org/3/reference/datamodel.html#emulating-callable-objects)이나 TaskSet 클래스의 리스트입니다. 

Locust의 공식 문서에서는 tasks 속성을 지정하기 위해 다음과 같은 방법을 소개하고 있습니다.

### User 클래스 안에 Task 지정하기

가장 기본적인 방법입니다.

```python
from locust import User, task, constant

class MyUser(User):
    wait_time = constant(1)

    @task
    def my_task(self):
        print("User instance (%r) executing my_task" % self)
```

이렇게 하면 task1 함수를 1초에 한 번씩 실행하겠네요.

그리고 여러 개의 @task 데코레이터로 실행하는 경우, 비율을 지정할 수도 있습니다. 

* `@task(1)`
* `@task(2)`

가 있다면, `@task(2)`로 지정한 작업을 두 배 더 실행할 것입니다.

### tasks 속성이 list라면?

```python
from locust import User, constant

def my_task(user):
    pass

class MyUser(User):
    tasks = [my_task]
    wait_time = constant(1)
```

tasks 속성을 리스트로 구성했다면, task를 수행해야 할 때 tasks 리스트에 있는 것들 중에 무작위로 골라서 실행합니다. 

### tasks 속성이 dict라면?

tasks 속성을 dict 타입으로 지정할 수도 있는데요. 이 때는 callable을 key로, int 타입의 숫자를 value로 구성해야 한다고 합니다. 

이 때 각각의 callable에 지정한 value가 실행 비율이라고 보시면 되겠습니다. 예제는 아래와 같습니다.

```
{my_task: 3, another_task: 1}
```

그러면 my_tasks는 another_task에 비해 3배 더 실행됩니다.

## 테스트 시작/종료 시 필요한 작업들

GitHub에 올라온 [이슈](https://github.com/locustio/locust/issues/1284)를 보면, Locust 클래스에서 `setup()`, `teardown()` hook을 삭제했다고 합니다. 

그러면 이를 대체하려면 어떻게 해야 할까요?

최근 버전에 도입된 개념 중 Event 라는 것을 볼 수 있습니다. 이런 경우에 사용할 수 있다고 합니다. 

* 부하 테스트의 시작 또는 중지 시 코드를 실행하고 싶을 때: `test_start`, `test_stop` 이벤트를 사용
    * Locustfile의 모듈 레벨에 이벤트 listener를 지정할 수 있음
    * Locust를 분산 환경에서 실행하는 경우, 이들 이벤트는 master 노드에서만 실행함

test_start, test_stop 이벤트에 리스너를 지정하는 예제는 다음과 같습니다. 

```python
from locust import events

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    print("A new test is starting")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    print("A new test is ending")
```

* 각각의 Locust 프로세스의 시작 시 호출하는 코드가 필요할 때: `init` 이벤트를 사용
    * Locust를 분산 환경에서 실행할 때, 각각의 Worker 프로세스가 이 이벤트에 연결된 코드를 실행함

init 이벤트에 리스너를 지정하는 예제는 다음과 같습니다. 

```python
from locust import events
from locust.runners import MasterRunner

@events.init.add_listener
def on_locust_init(environment, **kwargs):
    if isinstance(environment.runner, MasterRunner):
        print("I'm on master node")
    else:
        print("I'm on a worker or standalone node")
```

추가로 참고할 만한 문서를 링크해 드립니다. 필요할 때 참고하시면 될 것 같아요.

* [사용 가능한 이벤트의 종류와 필요한 parameter](https://docs.locust.io/en/stable/api.html#event-hooks)
* [이벤트 Hook을 확장하는 방법](https://docs.locust.io/en/stable/extending-locust.html#extending-locust)

TaskSet 클래스의 경우, `on_start()`, `on_stop()` 메소드가 남아있기 때문에 이를 그대로 이용하면 됩니다. 설명은 아래와 같습니다.

* `on_start()`: User가 TaskSet을 실행할 때 호출됨
* `on_stop()`: User가 실행 중인 TaskSet을 중지할 때 호출됨 (`interrupt()`가 호출되거나 User가 종료될 때)

## 용어의 변경

예전에 제가 분산 환경으로 구축하는 방법을 설명했을 때, `master / slave` 구성으로 설명을 드렸는데요. GitHub에 올라온 [이슈](https://github.com/locustio/locust/issues/220)에 따르면, `master / worker`로 용어가 변경되었습니다. 최근에 개발 분야에서 여러 용어를 중립적으로 변경하는 추세에 따르는 것 같습니다. 제가 이전에 `slave`라고 설명드렸던 부분은 `worker`라는 이름으로 바꾸어서 생각하시면 될 것 같습니다. 나중에 여유가 생기면 GitHub에 샘플 코드로 올렸던 내용 중 소스 코드나 설명을 수정할 예정입니다.

## 참고자료

* [Locust Quick Start](https://docs.locust.io/en/stable/quickstart.html)
* [Writing a locustfile](https://docs.locust.io/en/stable/writing-a-locustfile.html)
* [Locust 1.0 Release Note](https://github.com/locustio/locust/releases/tag/1.0)