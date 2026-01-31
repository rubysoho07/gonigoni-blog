---
title: "Playwright로 간단한 E2E 테스트 구현하기"
date: 2026-01-31T09:22:59+09:00
tags: [Playwright]
draft: false
categories: [Python]
comments: true
---

Headless 브라우저를 활용한 E2E 테스트를 구현해 볼 일이 있었습니다. 빠른 시간 내 만들어야 해서 익숙한 Python 언어를 선택했고, Python 기반으로 동작 가능한 툴을 찾아봤습니다. 이런 조건으로 아래와 같은 툴을 추천 받았어요. 

- [Selenium](https://www.selenium.dev/)
- [Playwright](https://playwright.dev/python/)
- [Pyppeter](https://github.com/pyppeteer/pyppeteer)

Selenium은 예전에 사이드 프로젝트에서 사용해 본 적이 있었는데요. 최근에는 Playwright를 추천하는 경우를 많이 봐서 Playwright를 사용해 보기로 했습니다. 

좀 더 찾아 보니 아래와 같은 특징이 있었습니다. 

- 여러 브라우저를 지원 (Chrome, Edge, Firefox, WebKit, …)
- TypeScript, JavaScript, Python, .NET, Java 지원
- `codegen` 명령을 사용해서 코드를 생성하도록 할 수 있음

이번 글에서는 Playwright를 사용해 보면서 경험했던 것들을 이야기 해 보려고 합니다. 

## 설치하기

[공식 문서](https://playwright.dev/python/docs/intro)에 따르면, `pytest-playwright`를 설치하는 것을 추천합니다. pytest의 테스트 케이스 작성 방법을 활용해서 스크립트를 작성하고, pytest만 실행하면 되기 때문에 간편하게 구축할 수 있습니다.

이번 글에서는 uv로 의존성을 관리한다고 가정하겠습니다. 혹시나 pip를 사용하시면 `uv add` 대신에 `pip install` 명령을 사용하시고, `uv run` 명령은 생략하시면 됩니다.

```shell
uv init playwright-example
cd playwright-example

# 라이브러리 설치
uv add pytest-playwright

# 브라우저 설치 (지원하는 브라우저는 `playwright help install` 명령 참조)
uv run playwright install

# 특정한 브라우저만 설치할 수 있습니다. (`--with-deps` 옵션을 추가하면 브라우저 실행에 필요한 의존성까지 설치함)
uv run playwright install --with-deps firefox
```

## 테스트 작성하기

[공식 문서에 있는 스크립트](https://playwright.dev/python/docs/intro#add-example-test)를 사용하겠습니다. 파일 이름은 `test_`로 시작해야 합니다. `test_playwright.py`로 저장합니다.

```python
import re
from playwright.sync_api import Page, expect

def test_has_title(page: Page):
    page.goto("https://playwright.dev/")

    # Expect a title "to contain" a substring.
    expect(page).to_have_title(re.compile("Playwright"))

def test_get_started_link(page: Page):
    page.goto("https://playwright.dev/")

    # Click the get started link.
    page.get_by_role("link", name="Get started").click()

    # Expects page to have a heading with the name of Installation.
    expect(page.get_by_role("heading", name="Installation")).to_be_visible()
```

그리고 `pytest` 명령을 실행합니다. (`pip`로 설치하셨다면 `uv run`은 빼고 실행합니다)

```shell
$ uv run pytest

# 아래와 같이 실행되면 성공입니다.
test_playwright.py ..                                                          [100%]

================================= 2 passed in 3.14s =================================
```

혹시 문제가 있다면 다음과 같은 옵션을 추가해서 확인할 수도 있습니다.

```shell
# 특정 브라우저를 지정해서 실행 (Firefox를 실행했다고 가정)
uv run pytest --browser firefox

# 디버깅을 위해 브라우저 화면을 띄워서 실행하기
uv run pytest --headed 
```

### 복잡한 테스트 케이스: codegen 활용

복잡한 테스트 케이스가 필요하다면 `codegen` 기능을 활용할 수 있습니다. 브라우저를 띄워서 테스트 코드를 작성하는 기능입니다. 

```shell
# 특정 브라우저를 지정하려면 `--browser firefox`와 같은 옵션을 추가합니다.
uv run playwright codegen demo.playwright.dev/todomvc 
```

명령어를 실행하면 아래 스크린샷과 같이 브라우저와 코드가 뜨는데요. 왼쪽에 있는 브라우저에서 여러 작업을 하면 오른쪽 창의 코드에 자동으로 반영됩니다. 이 코드를 바탕으로 테스트 케이스를 추가할 수 있습니다.

{{< figure src="/img/using-playwright-1.png">}}

## 컨테이너로 만들어서 실행하려면?

테스트 케이스가 많고, 주기적으로 테스트를 실행하려면 컨테이너로 만드는 방법도 있겠습니다. 다만 이 컨테이너를 클라우드 환경에서 실행할 때는 보안 관련 설정을 확인해야 합니다. 예를 들면 공격성 트래픽으로 오인하지 않도록 테스트 빈도를 조정하거나, 테스트 트래픽을 보내는 IP를 허용하는 것들이 있겠죠.

컨테이너 이미지 실행과 관련된 설명은 링크를 [참고](https://playwright.dev/python/docs/docker)하시기 바랍니다. 브라우저 및 브라우저 실행에 필요한 의존성 패키지가 같이 있어서 이미지 크기가 큰 편입니다.

Dockerfile은 다음과 같이 구성합니다. Base 컨테이너 image는 [링크](https://mcr.microsoft.com/en-us/artifact/mar/playwright/python/about)를 참고하세요.

> **주의사항**
> 설치한 Playwright 버전과 Base 컨테이너 이미지의 버전이 맞아야 정상 동작합니다. (브라우저 경로를 못 찾아서 실패합니다)
> - 설치한 Playwright 버전은 `uv.lock`이나 `requirements.txt` 파일에서 확인합니다.
> - Base 컨테이너의 태그에서 Playwright 버전을 지정할 수 있습니다. (예: v1.58.0)
> - 이 글을 작성하는 시점(2026-01-31) Playwright의 최신 버전은 1.58.0 입니다.

```Dockerfile
FROM mcr.microsoft.com/playwright/python:v1.58.0-noble

WORKDIR /app

COPY pyproject.toml .
RUN pip install .

COPY . .

CMD ["pytest"]
```

컨테이너 이미지를 빌드하고 실행해 봅시다. 

```shell
# 컨테이너 이미지 빌드
docker build -t playwright-example .

# 컨테이너 이미지 실행
docker run playwright-example

# 출력 내용: 아래와 같이 성공합니다.
============================= test session starts ==============================
platform linux -- Python 3.12.3, pytest-9.0.2, pluggy-1.6.0
rootdir: /app
configfile: pyproject.toml
plugins: playwright-0.7.2, base-url-2.1.0
collected 2 items

test_playwright.py ..                                                    [100%]

============================== 2 passed in 1.26s ===============================
```

## 마무리

Playwright를 써 보면서 개인적으로 경험한 것들을 몇 가지 말씀드리고 마치겠습니다.

이 작업을 하면서 AI에게 테스트 케이스를 만들어 달라고 한 적이 있는데요. 개인적인 경험으로는 AI에 맡기지 않고 직접 `codegen` 명령으로 브라우저를 보면서 테스트 코드를 생성했을 때 빠르게 작업할 수 있었어요. 근데 실제로 돌려보면 지연 때문에 실패할 때도 있었습니다. 이 때 `wait_for_url()` 함수([참고자료](https://playwright.dev/python/docs/api/class-frame#frame-wait-for-url))를 사용해서 잠깐 대기하는 작업을 추가하면 좋습니다.
    
두번째로 저는 macOS에서 테스트를 작성했는데, 인증된 바이너리를 사용하는 것이 아니라 OS 레벨에서 실행하지 못하게 막는 것 같았습니다. 개인적인 의견으로는 꼭 Chromium이나 Edge를 사용해야 하는 게 아니면 macOS에서는 Firefox를 이용하시는 것을 추천드리고, Chromium 계열의 브라우저를 사용해야 한다면 다른 OS에서 테스트 해 보시기 바랍니다. (컨테이너로 실행 시, 따로 설정하지 않으면 Chromium을 사용하기 때문에 컨테이너로 실행하는 것도 고려해 보면 좋겠습니다)

읽어주셔서 감사합니다.
