---
title: "RDS MySQL에서 일반/느린 쿼리 로그 찍기"
categories: AWS
tags: [RDS, MySQL, AWS]
draft: false
date: 2018-10-03T14:24:00+09:00
---

RDS MySQL을 이용하면, 아래와 같이 CloudWatch에 일반/감사/느린 쿼리 로그를 찍도록 설정할 수 있다. 

<figure>
    <img src="{{ "media/img/rds-logging-1.png" | absolute_url }}">
    <figcaption>로그 찍기 설정(RDS 인스턴스 생성 시)</figcaption>
</figure>

그리고 RDS 콘솔에 들어가면 로그 파일을 볼 수 있는데, 일반 로그나 느린 쿼리 로그를 찾을 수 없었다. 그래서 CloudWatch Logs를 찾아봤지만, 역시 로그가 없었다.

<figure>
    <img src="{{ "media/img/rds-logging-2.png" | absolute_url }}">
    <figcaption>로그 파일이 없다(-_-...)</figcaption>
</figure>

그 이유를 찾아 보니, 파라미터 그룹에 로그 관련 설정을 하지 않은 것이 원인이었다. 

다음과 같이 설정하면 된다.

먼저, RDS 콘솔에서 `파라미터 그룹` 메뉴를 클릭한다. 쓰던 파라미터 그룹이 있다면, 그 파라미터 그룹을 클릭하고, 새로 생성해야 한다면 `파라미터 그룹 생성`을 클릭해서 파라미터 그룹을 만든다. (RDS에서 기본으로 제공하는 파라미터 그룹은 편집이 불가능하니 파라미터 그룹을 만들어야 한다.)

필요에 따라 다음 설정을 수정 후 저장한다.

* 일반 로그: `general_log`를 1로 변경 (로그를 테이블 대신에 파일에 쓰는 설정을 추가로 해야 할 수도 있다.)
* 느린 쿼리 로그: `slow_query_log`를 1로 변경

<figure>
    <img src="{{ "media/img/rds-logging-3.png" | absolute_url }}">
    <figcaption>일반 로그(general log) 설정 변경</figcaption>
</figure>

참고로, 감사 로그는 파라미터 그룹에서 따로 설정하는 부분이 없다. 대신 Aurora RDS에서는 고급 감사 기능을 이용할 수 있다고 한다. ([링크 참조](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/AuroraMySQL.Auditing.html#AuroraMySQL.Auditing.View))

파라미터 그룹을 저장했다면, 인스턴스에 변경된 파라미터 그룹을 지정하고 저장한다. 파라미터 그룹 상태에 `(재시작 보류중)`이라고 뜨면 인스턴스를 재부팅한다. 파라미터 그룹 상태가 `(동기화)`가 될 때까지 기다리면, 아래와 같이 로그 파일이 생성됨을 확인할 수 있다.

<figure>
    <img src="{{ "media/img/rds-logging-4.png" | absolute_url }}">
    <figcaption>일반 로그와 느린 쿼리 로그 파일이 생겼다!</figcaption>
</figure>

### 참고자료

[https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/USER_LogAccess.Concepts.MySQL.html](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/USER_LogAccess.Concepts.MySQL.html)
