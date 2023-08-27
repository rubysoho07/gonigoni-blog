---
title: "IP 주소로부터 여러 가지 정보를 가져오기"
date: 2023-08-27T18:57:25+09:00
tags: [IP 주소, DDoS]
draft: false
categories: [DevOps]
comments: true
---

## 들어가며

인터넷 서비스를 운영하다 보면 DDoS 공격을 받는 경우가 종종 있습니다. 해킹 목적으로 사이트를 공격한다거나, 경쟁사를 견제하기 위한 목적으로 DDoS 공격을 하는 사례가 있습니다. 그 외에도 실수로 API 호출을 너무 많이 하는 바람에 의도치 않게 상대 서비스를 다운시키는 경우도 있겠죠. 이러한 DDoS 공격을 막기 위해, 클라우드 벤더에서 제공하는 기능이나 여러 회사에서 제공하는 솔루션 제품들을 사용할 수 있습니다. 

이번 글에서는 공격에 사용된 IP 주소 목록이 있을 때, IP 주소의 세부 정보를 얻을 수 있는 방법에 대해 이야기 해 보고자 합니다. 

## IP 정보를 얻을 수 있는 API들 비교

IP 주소로 얻어낼 수 있는 정보는 많습니다. 예를 들어 [ipapi.co](https://ipapi.co) 라는 사이트에서 8.8.8.8 주소에 대해 알아본다고 가정하겠습니다. (아시다시피 이 주소는 구글의 DNS 서버입니다)

{{< figure src="/img/ip-location-api-1.png" >}}

위와 같이 도시, 지역, 국가, 우편번호, 언어, 시간대, 위치(경도, 위도), ISP 등을 알 수 있습니다. (8.8.8.8은 ISP도 구글로 표시되네요)

앞에서 말씀드린 ipapi.co 사이트에 접속하면 여러분이 접속한 지점의 공인 IP를 기준으로 정보가 표시됩니다. 이를 통해 국가나 ISP 정보 등을 정확히 알 수 있습니다. AWS와 같은 클라우드 벤더에서 제공하는 공인 IP는 ISP 이름이 Amazon으로 표시되기도 합니다. 다만 상세한 위치 정보는 다르게 표시될 때도 있습니다. 예를 들어 AWS 서울 리전에 속한 IP 주소인데 미국 시애틀로 찍힌다거나, 서울 지역의 IP 주소인데 옆 동네인 경기도 수원시로 찍히는 경우가 있습니다. 그렇기 때문에, 국가나 ISP 정보 정도만 참고하는 용도로 사용하시면 될 것 같습니다.

IP 주소의 상세 정보를 제공하는 서비스들이 있는데요. 일부 서비스는 HTTP로 호출 후 응답을 JSON과 같은 포맷으로 받을 수 있는 기능을 제공합니다. 이런 기능을 제공하는 서비스들을 기준으로 비교해 보겠습니다. 아래 내용은 무료 버전 기준으로 작성했습니다.

* ipapi.co : 하루에 1,000개까지 조회 가능
* [ipinfo.io](https://ipinfo.io) : 월 50,000개까지 조회 가능함. 국가, 지역, 도시, 우편번호, 시간대 정도의 데이터를 얻을 수 있음
* [ipgeolocation.io](https://ipgeolocation.io) : 하루에 1,000개까지 조회 가능하고, 월 30,000개까지 조회 가능
* [ip-api.com](https://ip-api.com) : 하나의 IP 주소 당 초당 45 개의 요청으로 제한됨. 지속적으로 접속 제한을 초과하는 경우, 1시간동안 접속이 제한됨

동일 주소를 여러 기관을 통해 한 번에 비교할 수 있는 [iplocation.net](https://iplocation.net) 과 같은 서비스도 있으니 참고해 보시면 좋겠습니다.

## Python 코드 예제 - 다량의 IP 주소 목록에서 상세 정보 가져오기

이번 예제는 여러분이 침입자의 IP 주소 목록을 받았을 때, 국가와 ISP 정보만 추출해서 CSV 파일로 저장하는 예제입니다. 이 예제에서는 ip-api.com 을 이용했는데요. 하루에 1,000 개까지 조회 가능하다고 해도, 연속해서 여러 번 조회하면 HTTP 429 응답을 받는 경우가 있었기 때문입니다. 

입력 파일은 다음과 같습니다. (`ip.csv` 파일이라 가정하겠습니다) 

```
1.1.1.1
2.2.2.2
3.3.3.3
4.4.4.4
5.5.5.5
6.6.6.6
7.7.7.7
8.8.8.8
9.9.9.9
```

본격적으로 코드를 작성하기 전에, `requests` 라이브러리가 설치되어 있다고 가정하고 진행하겠습니다. 아래 스크립트를 실행하다가 에러가 발생하면, 먼저 `python3 -m pip install requests` 명령으로 설치해 주세요. (상황에 따라 `python3 -m` 부분은 생략할 수 있습니다)

```python
import csv
import time

import requests

def get_ip_location(ip: str) -> list:
    r = requests.get(f"http://ip-api.com/json/{ip}")
    r_json = r.json()

    # 호출 제한을 피하기 위해 3초 쉬고 다음 사이트를 찾습니다.
    time.sleep(3)

    # IP, 국가, ISP 순으로 반환합니다.
    return [ip, r_json['country'], r_json['isp']]

with open('ip.csv') as r_file:
    reader = csv.reader(r_file)

    with open('result.csv', 'w') as w_file:
        writer = csv.writer(w_file)

        for row in reader:
            writer.writerow(get_ip_location(row[0]))
```

결과는 `result.csv` 파일로 저장합니다. 실행 후 결과를 볼까요?

```
1.1.1.1,Australia,"Cloudflare, Inc"
2.2.2.2,France,France Telecom Orange
3.3.3.3,United States,"Amazon.com, Inc."
4.4.4.4,United States,"Level 3 Communications, Inc."
5.5.5.5,Germany,Telefonica Germany GmbH & Co. OHG
6.6.6.6,United States,CONUS-YPG
7.7.7.7,United States,DoD Network Information Center
8.8.8.8,United States,Google LLC
9.9.9.9,United States,Quad9
```

* 1.1.1.1은 클라우드플레어에서 제공하는 DNS 서버입니다. 호주에 있다고 표시되네요. 
* 8.8.8.8은 앞에서도 말씀드렸듯 구글에서 제공하는 DNS 서버입니다. 
* 9.9.9.9는 Quad9이라고 해서, IBM과 여러 단체에서 제공하는 DNS 서버입니다. 

## 마무리

이렇게 IP 주소를 기반으로 여러 정보를 가져오는 방법을 알아 보았습니다. IP 주소 목록이 있고, 시간 여유가 있는데 금전적 여유가 없을 때 이 방법을 이용하는 것을 추천드립니다. 호출 제한이 있다 보니 시간 간격을 늘려서 호출할 수 밖에 없기 때문입니다.

읽어주셔서 감사합니다.