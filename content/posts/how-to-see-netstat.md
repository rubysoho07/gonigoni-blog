---
title: "Netstat으로 리눅스 서버에서 클라이언트 접속 확인하기"
categories: Linux
tags: [netstat, 모니터링, 네트워크]
draft: false
date: 2018-12-25T17:15:00+09:00
---

회사에서 개발한 웹 서비스의 내부 테스트 버전을 배포하고, 다음과 같은 질문을 받았다. 

> 박대리님, 서버에서 클라이언트가 몇 명이 접속 중인지 알 수 있는 방법이 있을까요?

사실 여러 사람을 대상으로 하는 웹 서비스를 개발하고 배포하는 건 처음이라, 이걸 어떻게 알 수 있는지 궁금하긴 했다. 

그래서, `how to get the number of connections from web server`라는 키워드로 구글 검색을 해 보았다. 

{{< figure src="/img/netstat-1.png" >}}

여기 나오는 문서들 중에서 가장 많이 언급되는 유틸리티가 `netstat`인데, 여기서 연결 개수를 어떻게 체크해 볼 수 있는지 알아보자.

netstat의 사용법은, 모든 리눅스 명령어가 마찬가지겠지만 man page에서 확인할 수 있다. (`man netstat`)

## 80번 포트로 접속한 친구들을 확인해 봅시다.

터미널에서 `netstat -ant | grep ESTABLISHED | grep :80`이라고 입력하면 다음과 같은 화면을 볼 수 있다. (내가 갖고 있는 EC2 인스턴스에서 실험한 내용이다.)

{{< figure src="/img/netstat-2.png" >}}

(명령을 실행한 시점에 따라 연결된 클라이언트의 정보가 달라질 수도 있다.)

여기서 볼 수 있는 정보는 다음과 같다. (grep을 하지 않고 보면 헤더가 들어가므로 이 부분을 참고해 보자.)

* 맨 처음에 나타나는 tcp는 프로토콜을 의미한다. TCP인지, UDP인지 등을 볼 수 있다.
* 두번째와 세번째는 Recv-Q와 Send-Q라는 항목이다. 상세한 내용은 man page를 참고하자.
* Local Address: 로컬 주소와 포트 번호이다. 로컬에서 80번 포트(HTTP)로 연결된 클라이언트만 확인하기 위해 `grep :80`을 사용하였다.
* Foreign Address: 연결된 곳의 주소와 포트 번호이다. 
* State: 소켓의 상태를 나타낸다. Raw 소켓이나 UDP에는 상태가 없기 때문에, TCP 상태로 이해하면 되지 않을까 싶다. 연결이 맺어진 경우는 ESTABLISHED 상태이며, 이런 상태만 뽑아내기 위해 `grep ESTABLISHED`를 사용하였다. (TCP에 대해서는 따로 공부가 필요할 것 같다.) 

### netstat의 옵션에 대한 설명

그러면 앞에서 쓴 명령 중, netstat에 사용한 옵션은 무엇을 의미하는지 알아보자.

* `-a`: Listening 상태이거나 Non-Listening 상태인 소켓을 모두 보여준다.
* `-n`: 호스트 정보에 숫자로 된 IP 주소를 넣어서 보여준다.
* `-t`: TCP 연결인 경우에 한정해서 보여준다. (UDP인 경우 `-u` 옵션을 사용한다.)

그리고 다른 옵션도 한 번 살펴보자.

* `-o`: 네트워킹 타이머와 관련된 정보를 포함한다. (keepalive, timewait, off...)
* `-p`: 소켓을 사용하는 프로세스의 PID 및 프로그램 정보를 보여준다.

### 그냥 간단하게 개수만 알 수는 없나요?

다 필요 없고, 위에서 설명한 명령 끝에 `| wc -l`만 붙이자. (`|`를 빼먹지 말자!)

wc는 파일에서 줄바꿈 개수나 파일의 바이트 수, 문자 수를 알려주는 명령이다. 

`-l`은 줄바꿈 개수를 알려주는 옵션으로, 해당 조건에 해당하는 내용이 몇 줄인지를 출력해 준다.

## 덤: ss는 또 뭐야? ip route는 또 뭐지?

netstat의 man page에 보면 다음과 같은 내용을 확인할 수 있다. 

```
NOTES
       This  program  is  mostly  obsolete.   Replacement  for  netstat is ss.
       Replacement for netstat -r is ip route.  Replacement for netstat -i  is
       ip -s link.  Replacement for netstat -g is ip maddr.
```

여기서 netstat은 mostly obsolete, 즉, 거의 시대에 뒤처진 프로그램이라고 설명하고 있다. `netstat`의 대체재로 `ss`를, `netstat -r`의 대체재로 `ip route`를, 그리고 `netstat -i`의 대체재로 `ip -s link`를 사용하라고 한다. 또한 `netstat -g` 대신 `ip maddr`을 쓰라고 하고 있다. `ss`는 어떻게 쓸 수 있는지 확인하기 전에, 나머지 명령어가 어떤 역할을 하는지를 알아보자. 옵션의 의미는 모두 man page에서 찾은 내용이다.

* `netstat -r` -> `ip route`: 커널 내의 라우팅 테이블을 표시한다. 
* `netstat -i` -> `ip -s link`: 모든 네트워크 인터페이스의 테이블을 표시한다.
* `netstat -g` -> `ip maddr`: IPv4나 IPv6에 대한 멀티캐스트 그룹 멤버십 정보를 표시한다. 

### ss의 사용법

터미널에서 `ss`를 입력하면 다음과 같은 화면을 볼 수 있을 것이다. (이번에도 역시 내가 가지고 있는 EC2 인스턴스에 SSH로 접속해서 확인한 내용이다.)

{{< figure src="/img/netstat-3.png" >}}

여기서 우리가 주목해 볼 만한 것은 다음과 같다.
* Netid - TCP 연결인지? UDP 연결인지? HTTP 프로토콜로 연결했다면, TCP로 연결했을 것이다. 
* State - 연결 상태를 확인할 수 있다. Listening 상태인 연결과 non-listening 상태인 연결을 모두 보려면, ss를 실행할 때 `-a`나 `--all` 옵션을 붙이면 된다.
* Local Address:Port - 로컬 시스템에서는 어떤 주소, 어떤 포트로 연결되어 있는 지 확인할 수 있다.
* Peer Address:Port - 상대방은 어떤 주소인지, 어떤 포트로 연결하여 있는 지 확인할 수 있다.

특정한 경우만 필터링해서 보고 싶다면 다음 옵션을 사용하면 된다. 물론 여러 옵션을 같이 쓸 수도 있다.
* TCP 연결만 보기: `ss -t` 또는 `ss --tcp`
* UDP 연결만 보기: `ss -u` 또는 `ss --udp`
* IPv4 연결만 보기: `ss -4` 또는 `ss --ipv4`
* IPv6 연결만 보기: `ss -6` 또는 `ss --ipv6`
