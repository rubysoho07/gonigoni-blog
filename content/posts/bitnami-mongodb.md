---
title: "EC2에 Bitnami MongoDB 이미지 올리기"
categories: [AWS]
tags: [AWS, EC2, MongoDB]
draft: false
date: 2019-02-16T22:30:00+09:00
comments: true
---

최근에 AWS에서 MongoDB와 호환되는 [DocumentDB](https://aws.amazon.com/ko/documentdb/)를 출시했지만, ~~아직 서울 리전에서는 사용할 수 없다.~~

(2019년 5월에 서울 리전에 출시되었습니다. [링크한 글을 확인해 주세요~](https://aws.amazon.com/ko/blogs/korea/amazon-documentdb-and-kinesis-analytics-seoul-region/))

그렇지만 필요에 따라 MongoDB를 쓸 일이 있을 것이다. AWS에서 Bitnami의 이미지를 활용해서 EC2에 MongoDB를 올려보고, 시험 삼아 데이터를 넣어보자. 

### EC2 설정

EC2 인스턴스를 생성하기 위해 AWS의 EC2 콘솔로 들어간다. 아래 스크린샷과 같은 화면이 나오면, '인스턴스 시작'을 누른다.

{{< figure src="/img/bitnami-mongodb-1.png" >}}

그리고 검색어에서 MongoDB를 입력하고, 왼쪽에서 AWS Marketplace를 누른다.

{{< figure src="/img/bitnami-mongodb-2.png" >}}

스크롤을 아래로 내리다 보면, 'MongoDB Certified by Bitnami'가 있다. 오른쪽의 선택 버튼을 누른다.

{{< figure src="/img/bitnami-mongodb-3.png" >}}

라이선스에 대한 설명이 나오면 오른쪽 위의 '요금 내역' 부분을 확인한다. 만약 유료 소프트웨어를 쓴다면 EC2 사용 요금 외의 별도 요금이 청구되는데, 여기서 선택한 인스턴스는 별도 요금이 청구되지 않는다.

확인했으면 Continue를 누른다.

{{< figure src="/img/bitnami-mongodb-4.png" >}}

그 다음으로는 인스턴스 타입을 설정하는 단계이다. 테스트 용도라면 굳이 비싼 인스턴스를 이용할 필요가 없을것이다. 여기서는 t2.micro를 선택하고, 검토 및 시작을 누른다. 

{{< figure src="/img/bitnami-mongodb-5.png" >}}

내용을 확인한 뒤 '시작하기'를 누른다. 

{{< figure src="/img/bitnami-mongodb-6.png" >}}

그러면 pem 키를 설정하는 창이 뜰 것이다. 이미 키가 있다면 기존의 키를 사용하고, 키가 없다면 새로 생성한다. 여기서는 테스트용 키를 새로 생성하였다.

키를 생성하는 경우, 키를 다운로드 해야 인스턴스를 시작할 수 있다. 

{{< figure src="/img/bitnami-mongodb-7.png" >}}

### MongoDB 접속 설정

조금 기다리면 인스턴스를 사용할 수 있다. 먼저 MongoDB를 사용하기 위한 계정 정보를 가져와야 한다. 

생성한 인스턴스를 선택 후, 작업 - 인스턴스 설정 - 시스템 로그 가져오기를 클릭한다.

{{< figure src="/img/bitnami-mongodb-8.png" >}}

그리고 스크롤을 내리다 보면, 아래와 같은 내용을 발견할 수 있다. 
비밀번호를 메모하자. (계정 이름은 root이다)

{{< figure src="/img/bitnami-mongodb-9.png" >}}

필요한 경우 ssh를 이용해서 접속할 수도 있겠지만, 여기서는 Robo 3T를 이용해서 인스턴스에 접속해 보겠다. 

먼저 연결할 주소는 콘솔에서 인스턴스를 선택하면 IPv4 퍼블릭 IP를 찾을 수 있다. 

이를 복사해서 넣어준다. 

{{< figure src="/img/bitnami-mongodb-10.png" >}}

그리고 Authentication을 눌러 다음과 같이 방금 메모한 계정 정보와 비밀번호를 넣어준다. 모든 설정이 끝나면 Save를 눌러 빠져나간다.

{{< figure src="/img/bitnami-mongodb-11.png" >}}

만약 접속을 시도하면 잘 안 될 것이다. 이 경우 보안 그룹에서 27017 포트를 열어주어야 한다. 

AWS 콘솔에서 인스턴스를 클릭한 뒤, 보안 그룹 이름을 클릭하면 보안 그룹 설정 페이지로 이동할 것이다. 

이후에는 다음과 같이 진행한다.
* 인바운드를 누른다.
* 편집을 누른다.
* 규칙 추가를 누른 뒤, 아래 스크린샷과 같은 규칙을 추가하고 저장한다.

{{< figure src="/img/bitnami-mongodb-12.png" >}}

이제 Robo 3T에서 데이터베이스와 콜렉션을 추가한다. 아래 스크린샷과 같이 연결 위에서 오른쪽 버튼을 눌러 데이터베이스를 추가하고, 데이터베이스 아래의 Collections 위에서 오른쪽 버튼을 눌러 콜렉션을 추가한다.

(이번 예제에서는 데이터베이스, 콜렉션 모두 test로 지정하였다)

{{< figure src="/img/bitnami-mongodb-13.png" >}}

### 데이터 삽입/조회 테스트

이제 만든 콜렉션을 더블클릭하면 아무것도 없을 것이다. 이 때, 다음 명령을 위의 입력 창에 입력하고 Ctrl+Enter를 누른다. 

```
db.getCollection('test').insert({"name": "value"});
```

그러면 아래 창에 다음과 같은 메시지를 볼 수 있다.

```
Inserted 1 record(s) in 21ms
```

{{< figure src="/img/bitnami-mongodb-14.png" >}}

이제 조회를 해 보자. 위의 입력창에서 다음과 같이 입력한 뒤 Ctrl+Enter를 누르면, 방금 삽입한 레코드를 조회할 수 있다. 

```
db.getCollection('test').find();
```

{{< figure src="/img/bitnami-mongodb-15.png" >}}

### 마무리

* 테스트가 끝나면, 인스턴스를 삭제해 주자.
* 이번에 살펴본 것들은 간단한 테스트 용도로 쓸 수 있는 방법이다. 실제 서비스에서는 인스턴스에 접속할 수 있는 네트워크/호스트를 제한하거나, 별도의 사용자를 만들어서 관리해야 할 것이다. 