---
title: "Python에서 logging으로 로그 만들기"
date: 2021-10-31T10:58:31+09:00
tags: [Python, logging, warning]
draft: false
categories: [Python]
comments: true
---

프로그램이 잘 동작하는 지 확인하고 싶다면, `print()`를 쓸 때가 많습니다. 하지만 프로그램이 복잡하게 바뀌면서 `print()`만으로는 감당하기 어려운 상황이 발생합니다. 

그럴 때 어떻게 로그를 만들고 관리할 지를 고민하게 되는데요. 이번 글에서는 Python에서 `logging` 모듈을 활용해서 어떻게 로그를 관리할 수 있는지 살펴보려고 합니다.

# 기본적인 로깅 만들기

`logging` 모듈을 이용해서 로그를 만드는 방법은 다음과 같습니다. 

```python
import logging

logging.critical('Critical log')
logging.error('Error log')
logging.warning('Warning log')
logging.info('Info log')
logging.debug('Debug log')
```

스크립트를 실행해 보면 다음과 같이 표시됩니다. 

```shell
CRITICAL:root:Critical log
ERROR:root:Error log
WARNING:root:Warning log
```

CRITICAL, ERROR, WARNING은 뭔지 알 것 같습니다. 그런데 갑자기 `root`는 왜 나오는 걸까요? 그리고 제가 출력하려고 한 INFO, DEBUG 로그는 왜 안 나오는 걸까요?

우선 루트 로거(Logger)에 대해 한 번 생각해 보겠습니다. 로거는 [응용 프로그램 코드가 직접 사용하는 인터페이스를 노출한다](https://docs.python.org/ko/3/howto/logging.html)고 되어 있긴 한데요. 

루트 로거는 모든 로거의 상위 로거입니다. 로거 객체는 `logging.getLogger()`로 얻어올 수 있는데, 여러번 가져오더라도 동일한 객체를 참조합니다. 

또한 기본적으로 로거에 로그된 이벤트는 모듈에 지정한 로거에서 상위 계층의 로거 객체에 전달됩니다. 즉, 여러 모듈에서 로그가 발생하더라도 최종적으로 로그를 처리하는 건 루트 로거인 것이죠. (자세한 내용은 [문서](https://docs.python.org/ko/3/howto/logging.html#logging-flow) 참조)

어떤 의미인지 설명하기 애매해서, 예제를 가지고 설명해 볼게요. 

아래와 같이 mymodule.py 파일을 만듭니다. Formatter와 Handler에 대해서는 뒤에서 설명할 예정입니다. 

```python
import logging

logger = logging.getLogger(__name__)
formatter = logging.Formatter('%(name)s - %(message)s')
handler = logging.StreamHandler()
handler.setFormatter(formatter)
logger.addHandler(handler)

def my_function():
    logger.warning('My Module Message')
```

아래와 같이 logging_test.py 파일을 만듭니다. mymodule 모듈의 my_function 함수를 import 하여 사용하는 부분에 유의해 주세요.

```python
import logging

from mymodule import my_function

logger = logging.getLogger(__name__)
formatter = logging.Formatter('%(name)s - %(message)s')
handler = logging.StreamHandler()
handler.setFormatter(formatter)
logger.addHandler(handler)

logger.warning('Warning log')

my_function()
```

logging_test.py 파일을 실행해 보면 다음과 같은 메시지가 출력되는 것을 보실 수 있습니다.

```shell
__main__ - Warning log
mymodule - My Module Message
```

mymodule에서 생성한 로그는 mymodule의 로거에서 처리되고, 메인(logging_test.py) 프로그램의 로거로 전달되는 것으로 보이네요. 

이를 이용하면 여러 모듈에서 발생한 로그를 모아서 추적할 수 있을 것 같습니다.

# 로그 레벨

위의 예제에서 `logger.info()`로 호출한 메시지와 `logger.debug()`로 호출한 메시지가 출력되지 않는 것을 확인할 수 있었습니다. 

이는 로그 레벨, 또는 심각도(Severity)라고 하는 개념과 관련이 있는데요. 아래와 같이 각각의 수준과, 이에 대응하는 숫자로 구성됩니다.

|수준|숫자 값|
|-----|-----|
|CRITICAL|50|
|ERROR|40|
|WARNING|30|
|INFO|20|
|DEBUG|10|

기본 수준은 `WARNING`이며, 이보다 숫자 값이 높은 로그만 출력하게 됩니다. 

그런데 로그 수준을 다르게 설정하려면 어떻게 해야 할까요?

첫번째 방법은 logging 모듈 자체에서 설정하는 방법입니다. (`logging.basicConfig` 부분에 주목해 주세요)

```python
import logging

logging.basicConfig(level=logging.DEBUG)

logging.critical('Critical log')
logging.error('Error log')
logging.warning('Warning log')
logging.info('Info log')
logging.debug('Debug log')
```

이렇게 하면 모든 로그가 출력됩니다. 

```
CRITICAL:root:Critical log
ERROR:root:Error log
WARNING:root:Warning log
INFO:root:Info log
DEBUG:root:Debug log
```

그러면 로거마다 레벨을 다르게 지정하고 싶으면 어떻게 해야 할까요? 이 때는 로거와 핸들러에서 처리할 로그 레벨을 지정해 주어야 합니다. 

핸들러(Handler) 객체는 로거가 받은 로그를 여러 목적지로 보내주는 역할을 하는데요. 

로거 객체에만 DEBUG로 로그 레벨을 설정하더라도 적용되지 않는데, 핸들러 객체를 만들어서 핸들러에도 로그 레벨을 적용해 주면 적용되는 것을 볼 수 있습니다.

```python
import logging

# Logger 객체의 로그 레벨을 정해 줍니다.
logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

# Handler 객체를 만들고, 로그 레벨을 정해 줍니다. StreamHandler에 아무 옵션이 없다면 stderr로 로그를 내보냅니다.
handler = logging.StreamHandler()
handler.setLevel(logging.DEBUG)

# Handler 객체를 Logger 객체에 붙여 줍니다.
logger.addHandler(handler)

logger.critical('Critical log')
logger.error('Error log')
logger.warning('Warning log')
logger.info('Info log')
logger.debug('Debug log')

# 출력되는 결과는 다음과 같습니다.
# ------------------------
# Critical log
# Error log
# Warning log
# Info log
# Debug log
```

# 로그 메시지 양식 설정하기

루트 로거의 기본 로그 양식은 `<로그 레벨>:<로거 이름>:<메시지>` 입니다. 만약 로거를 지정한 경우라면 메시지만 출력됩니다. 

만약 로그 문자열의 포맷을 바꾸고 싶다면, 포매터(Formatter) 객체를 이용합니다.

포매터에서 포맷을 지정할 때 쓸 수 있는 속성은 [LogRecord 어트리뷰트](https://docs.python.org/ko/3/library/logging.html#logrecord-attributes) 문서를 참고하세요.

자주 사용할 만한 것들을 정리해 보면 다음과 같습니다.

* `%(asctime)s`: 사람이 읽을 수 있는 로그 레코드가 생성된 시간. YYYY-MM-DD hh:mm:ss,xxx 형식입니다. (xxx는 밀리초 단위)
* `%(levelname)s`: 로그 레벨 이름 (DEBUG, INFO, WARNING, ERROR, CRITICAL)
* `%(message)s`: 로그 메시지
* `%(name)s`: 로거 이름
* `%(filename)s`: 소스 파일의 전체 경로명 중, 파일 이름 부분
* `%(funcName)s`: 로깅 호출을 포함하는 함수의 이름

예를 들어서, 로거 이름, 로그 레벨 이름, 로그 레코드 생성 시간, 메시지 순으로 출력하려면 어떻게 해야 할까요?

아래 예제를 통해 확인해 보시죠.

```python
import logging

logger = logging.getLogger(__name__)

# Handler 객체를 설정합니다. 기본값은 위에서 말씀드렸듯이, stderr로 로그를 보냅니다.
handler = logging.StreamHandler()
# Formatter 객체를 설정합니다. (위의 내용을 참고하여 필요한 요소를 넣었습니다)
formatter = logging.Formatter('%(name)s - %(levelname)s - %(asctime)s - %(message)s')

# Formatter를 Handler에 설정합니다.
handler.setFormatter(formatter)
# Handler를 Logger에 추가합니다.
logger.addHandler(handler)

logger.critical('Critical log')
logger.error('Error log')
logger.warning('Warning log')
```

실행 결과는 다음과 같습니다. 스크립트를 만든 상태에서 실행했더니 로거 이름은 `__main__`으로 설정되네요.

```
__main__ - CRITICAL - 2021-10-31 15:57:30,550 - Critical log
__main__ - ERROR - 2021-10-31 15:57:30,550 - Error log
__main__ - WARNING - 2021-10-31 15:57:30,550 - Warning log
```

로그를 JSON 포맷의 문자열로 만들 때 포매터 객체를 적절히 만들어 주면 유용할 것 같습니다. 예를 들면 다음과 같은 방식으로 이용할 수 있겠죠?

```python
import logging

logger = logging.getLogger(__name__)
# JSON 문자열로 로그를 구성해 봅니다.
formatter = logging.Formatter('{"Name": "%(name)s", "Message": "%(message)s"}')
handler = logging.StreamHandler()
handler.setFormatter(formatter)
logger.addHandler(handler)

def my_function():
    logger.warning('My Module Message')

if __name__ == "__main__":
    my_function()
```

## 로그 메시지에 추가 정보를 같이 넣기

로그 메시지에 추가 정보를 같이 넣을 수도 있습니다. 우선 추가 정보를 딕셔너리 형태로 구성해 봅시다.

```python
d = {"src_ip": "192.168.0.1", "user": "test"}
```

그리고 포매터 객체를 다음과 같이 설정합니다.

```python
import logging

logger = logging.getLogger(__name__)
# 아래 줄에 src_ip, user 부분이 들어가 있는 점에 주목해 주세요.
formatter = logging.Formatter('%(levelname)s - %(message)s - %(src_ip)s - %(user)s')
handler = logging.StreamHandler()
handler.setFormatter(formatter)
logger.addHandler(handler)
```

로그를 찍을 때, `extra`에 앞에서 만든 딕셔너리로 지정해 줍니다.

```python
d = {"src_ip": "192.168.0.1", "user": "test"}
logger.warning('Warning log', extra=d)
```

그러면 다음과 같이 출력됩니다. 아래와 같이 `src_ip`, `user`에 해당하는 값을 볼 수 있습니다.

```
WARNING - Warning log - 192.168.0.1 - test
```

만약 `src_ip`나 `user`가 딕셔너리에 없는 경우, 예외가 발생하면서 로그가 출력되지 않습니다. 그래서 딕셔너리에 원하는 키가 없을 때 어떻게 처리할 것인지 고민해 보셔야 합니다.

# 로그 저장 위치 설정하기

핸들러를 사용하면 로그 저장 위치를 설정할 수 있습니다. 

앞에서 살펴본 내용 중 StreamHandler를 이용하면 sys.stdout이나 sys.stderr, 또는 파일과 비슷한 객체로 로그 출력을 보낼 수 있습니다. 

[logging.handlers](https://docs.python.org/ko/3/library/logging.handlers.html#module-logging.handlers) 문서를 살펴 보면 여러 종류의 핸들러 객체가 있음을 알 수 있습니다.

몇 가지를 살펴보면 다음과 같습니다.

* [StreamHandler](https://docs.python.org/ko/3/library/logging.handlers.html#streamhandler): stdout/stderr, 기타 파일류 객체로 로그를 내보냅니다. (기본값은 sys.stderr 입니다)
* [FileHandler](https://docs.python.org/ko/3/library/logging.handlers.html#filehandler): 로그 출력을 디스크 내 파일로 보냅니다. 파일명과 인코딩, 파일 쓰기 모드를 지정할 수 있습니다.
* [SocketHandler](https://docs.python.org/ko/3/library/logging.handlers.html#sockethandler): 로그 출력을 네트워크 소켓으로 보냅니다. 호스트와 포트를 지정할 수 있습니다. (기본적으로 TCP 소켓을 만듭니다)
* [SysLogHandler](https://docs.python.org/ko/3/library/logging.handlers.html#sysloghandler): syslog로 로그 메시지를 보냅니다.

## 로그 파일을 주기적으로 바꾸기

로그를 파일에 저장하는 경우, 로그 파일이 과도하게 커지는 상황을 방지하기 위해 파일의 최대 크기와 백업본 수를 지정할 수 있습니다. 

이를 위해서 로그 핸들러 중 [RotatingFileHandler](https://docs.python.org/ko/3/library/logging.handlers.html#rotatingfilehandler)를 사용할 수 있습니다. 

기본적인 사용 방법은 FileHandler와 같으나, 두 가지 옵션이 더 있습니다.

* `maxBytes`: 로그 파일의 최대 길이. 기본값은 0
* `backupCounts`: 최대 로그 파일 수. 기본값은 0

위의 두 옵션을 지정하지 않으면 로그 파일이 교체되지 않습니다. 

이외에도 주기적으로 로그 파일을 교체할 수 있는 [TimedRotatingFileHandler](https://docs.python.org/ko/3/library/logging.handlers.html#timedrotatingfilehandler)도 있으니, 필요할 때 사용하면 좋을 것 같네요.

# 참고자료

* [logging 모듈 문서](https://docs.python.org/ko/3/library/logging.html)
* [로깅 요리책](https://docs.python.org/ko/3/howto/logging-cookbook.html)
* [로깅 HOWTO](https://docs.python.org/ko/3/howto/logging.html)
