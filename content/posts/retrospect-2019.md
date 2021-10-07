---
title: "Retrospect 2019"
date: 2020-01-04T15:11:05+09:00
tags: [회고, 2019년]
draft: false
categories: [Everything]
comments: true
---

2020년이 된 지 4일이 되었지만, 지난 1년 간에 있었던 일들을 정리해 보려 합니다. 

## 발표

### AWS Summit Seoul 2019

커뮤니티 세션에서 저희 부서의 데이터 저장 기반 구축 사례를 소개했습니다. 다음 영상에서 확인하실 수 있습니다. ([슬라이드는 여기를 눌러주세요](https://www.slideshare.net/awskorea/aws-aws-summit-seoul-2019-141290094))

{{< youtube wxScotB5lXk >}}

PyCon Korea 외의 다른 행사에서 발표하는 건 처음이었는데요. 정말 많은 분들이 오셔서 긴장했는데, 어떻게든 잘 넘어갔던 것 같아요. 발표한 내용은 재작년(2018) 말 기준으로 구성했던 내용이고, 현재 저희 팀의 AWS 인프라는 많은 부분이 바뀐 상태입니다. 그리고 작년 AWS Summit을 기점으로 AWSKRUG 내 소모임도 가끔씩 참석하게 되었습니다.

### SSAFY (삼성 청년 SW 아카데미)

봄에 멀티캠퍼스에 계신 분으로부터 메일을 받았습니다. 삼성에서 진행하는 ‘[삼성 청년 SW 아카데미](https://www.ssafy.com/ksp/jsp/swp/swpMain.jsp)’라는 교육 과정 중에, 파이썬 개발자로서의 경험을 공유해 주실 수 있느냐는 제안을 받았습니다. 

개인적으로는 지금까지 겪어온 과정과 시행착오를 정리해 보고 싶었는데요. 좋은 경험이 될 것 같아서 수락했습니다. 원래는 발표 날 전에 혼자 홍콩 여행을 갔다 오려고 계획하고 있었는데요. 결국 여행 때는 여기저기 돌아다니다가, 저녁 먹고 쉴 때는 발표 연습을 했었습니다.

서울에서는 현장에서 강의(?)를 진행하고, 나머지 장소에서는 녹화해서 보여주신다고 했는데요. 강의를 들어보셨던 분들은 어떻게 받아들이셨는지 모르겠습니다. 사실 직접 겪기 전에는 개발자로서 커리어를 지속하는 데, 특히 좀 더 나은 환경에서 일하기 위해서 많은 노력과 시행착오가 필요한 지 몰랐거든요. 비전공자든, 저와 같은 반전공자든 좋은 결과가 있으셨으면 좋겠습니다.

[내용은 다음 슬라이드에서 확인하실 수 있습니다.](https://www.slideshare.net/secret/GJrlfZ2Eh9ybAX)

## 4번의 미국 출장

작년에는 출장으로 미국을 4번 다녀 왔습니다. 세 번은 IMS Global의 Quarterly Meeting, 한 번은 re:Invent를 다녀왔습니다. 이제 한 번만 더 다녀오면 모닝캄도 가능한데... [어라..?](https://www.koreanair.com/korea/ko/promotions/new_skypass/)

### IMS Global Quarterly Meeting

[IMS Global](https://www.imsglobal.org)은 교육 관련 기관(학교도 포함), 교육 관련 서비스 벤더 등이 모여 설립한 단체입니다. 저희 회사도 [Contributing Member](https://www.imsglobal.org/membersandaffiliates.html)로 등록되어 있습니다. 이 단체에서는 교육 관련 데이터나 도구 간 상호운용성에 대한 표준을 만들고 있습니다. 

IMS Global은 분기별로 행사를 진행하는데요. 저는 2, 8, 11월의 Quarterly Meeting에 참석했습니다. (5월은 Learning Impact라는 행사가 있는데요. 교육 관련된 서비스에 대한 시상식 같은 개념으로 보시면 됩니다)

매 분기의 Quarterly Meeting은 다음과 같은 주제가 있습니다. 
* 2월: Digital Badge Summit (Open Badge라는 디지털 인증 표준과 도입 사례)
* 8월: Technical Congress (이 단체에서 관리하는 교육 관련 각종 표준과 도입 사례)
* 11월: Learning Analytics Summit (학습 분석과 관련된 사례, 학습 데이터 구축 및 운영 사례)

개인적으로 감명깊게 봤던 것들을 2개 정도 요약하면 다음과 같습니다. (모두 11월 학습 분석 서밋에서 들은 내용입니다)

* Georgia State University: 저소득층 학생의 중퇴율이나 미등록을 줄이기 위해 데이터를 이용
* University of Buffalo: 캠퍼스 간 버스 탑승 데이터 분석해서 신규 노선 추가, 특정 수업의 Student Success 예측

> **사진 설명** 왼쪽: Georgia State University의 사례 / 오른쪽: University of Buffalo의 사례 중 수업 관련 내용 분석

{{< figure src="/img/retrospect-2019-1.png" >}}

### AWS re:Invent

갑자기 re:Invent 출장이 잡혀서 급하게 준비하게 되었습니다. 이미 많은 세션들이 이미 예약이 꽉 차 있었던 관계로, 듣고 싶었던 것 중에 못 들은 것도 있었습니다. 그리고 회사 돈으로 가는 거라, 준비부터 정산, 보고까지 회사 내부의 프로세스를 따라야 했습니다. 다행히 10월 부터 국외 출장에 대한 사내 프로세스가 간소화되어 그나마 좀 더 편하게 다녀올 수 있었습니다. 다만 좀 더 여행을 못 해본 건 아쉽네요. 

그리고 한국 밖에서 생일을 보낸 건 처음이었어요. (목요일이 생일이었어서 re:Play에 잠깐 다녀와서 술도 먹고, 먹을 것도 먹고, 음악도 좀 들어보고, ... ㅋㅋ)

개인적으로는 신규 서비스보다는 DevOps나 개발 문화와 관련된 세션에 많이 들어가 봤습니다. 팀이 좀 더 커지다 보니 그런 내용에 더 관심이 생겨서요. 그리고 데이터베이스나 컨테이너 관련 세션 중에서 들어갈 수 있는 곳은 들어가 봤습니다.

제가 들은 세션을 정리하면 다음과 같습니다. 참고로 하루에 3개 이상 세션을 듣는 건 좀 무리라 생각합니다. 물론 한 건물에 계속 있어도 3개 이상은 지칠 가능성이 높습니다. 키노트는 수요일 키노트(Global Partner Summit)를 제외하고는 다 들었네요.

* [SEC205: The fundamentals of AWS cloud security](https://youtu.be/QMBkq6MrT2w)
* [ARC210: Microservices decomposition for SaaS environments](https://youtu.be/AOfuZN5yo38)
* [DAT202: What's new in Amazon Aurora](https://youtu.be/QJB1vUlkmWQ)
* [SEC340: Using AWS KMS for data protection, access control, and audit](https://youtu.be/hxWvbNvj2lg)
* [CON330: Running Kubernetes clusters at scale: Bird](https://youtu.be/WVbyQvPa5O8)
* [CON217: Roadmaps for containers, application networking & Amazon Linux at AWS](https://youtu.be/IQdUeq9dkKQ)
* [DAT321: Deep Dive on Amazon Aurora with MySQL Compatibility](https://youtu.be/GwEtiRZR4g4)
* [DOP308-S: Building a culture of obserability](https://youtu.be/qM5-v8tRLRY)
* [DOP404: Amazon's approach to high-availability deployment](https://youtu.be/bCgD2bX1LI4)

얻어 온 SWAG은 제가 필요한 것만 챙기고 나머지는 주변 분들께 모두 나눠드렸어요. 그리고 엑스포 첫 날에는 사람이 너무 많아 구경하기 불편했습니다. 다른 날에 가시는 걸 추천드려요.

## 기타 등등

Udemy에서 다음과 같은 강좌를 들었습니다. 
* Docker & Kubernetes
* Elasticsearch, Logstash, Kibana

작년에 제가 배웠던 것을 언제 써 볼 수 있을지 모르겠지만, 아마도 올해 저희 팀 내에서 적극적으로 도입하게 될 것 같네요. (다만 K8s(EKS) 대신에 ECS를 이용할 것 같습니다)

그리고 집 계약이 끝나기 전에 대출을 받아 전세로 이사했습니다. 월세로 살다가 전세집으로 이사가려고 알아보니 준비할 것들이 많았네요. 이것도 준비 과정을 적으려면 글 하나는 나올 것 같긴 한데, 블로그를 검색하다 보면 후기를 흔히 보실 수 있을 거라서 생략하겠습니다.

읽어주셔서 감사합니다. 