---
title: "VPC Reachability Analyzer로 AWS 네트워크 문제 확인하기"
date: 2024-12-25T21:49:00+09:00
tags: [VPC, Reachability Analyzer]
draft: true
categories: [AWS]
comments: false
---

인프라 구성을 하다 보면, 네트워크 설정 때문에 문제를 겪는 일이 많습니다. "Connection timeout"과 같은 네트워크 관련 오류가 갑자기 발생하면, 어디서부터 원인을 찾아야 할 지 막막한데요. AWS를 사용하신다면 네트워크 문제를 겪을 때 VPC Reachability Analyzer라는 서비스를 이용하여 원인을 찾을 수 있습니다. 출발하는 지점부터 도착지까지 어떤 과정으로 트래픽이 도달하는 지 상세하게 확인할 수 있는 서비스입니다. 네트워크 문제가 발생한다면 한 번 사용해 보시기 바랍니다.

## VPC Reachability Analyzer

[VPC Reachability Analyzer](https://aws.amazon.com/ko/blogs/korea/new-vpc-insights-analyzes-reachability-and-visibility-in-vpcs/)는 2020년에 소개된 기능으로, 소스와 타겟 리소스를 지정하여 내가 의도한 대로 네트워크를 구성하였는지 확인할 수 있습니다. 다만 AWS에 구성한 네트워크 기준으로만 확인할 수 있기 때문에, 애플리케이션 내부에서 트래픽을 차단하는 경우는 감지하지 못합니다. 애플리케이션에서 특정 포트를 차단하는 경우가 있다면 서버 내의 설정을 확인하거나, 애플리케이션 코드를 확인해 보아야 합니다.

### 가능한 소스와 타겟

**Source**
* EC2 인스턴스
* 인터넷 게이트웨이
* 네트워크 인터페이스 (ENI)
* Transit Gateway
* Transit Gateway Attachments
* VPC Endpoints
* VPC Endpoint Services
* VPC Peering Connection
* VPN Gateways

**Target**
* EC2 인스턴스
* 인터넷 게이트웨이
* IP 주소
* 네트워크 인터페이스 (ENI)
* Transit Gateway
* Transit Gateway Attachments
* VPC Endpoints
* VPC Endpoint Services
* VPC Peering Connection
* VPN Gateways

## 실제로 분석해 보기

### 테스트 서버 구성

간단하게 서버 두 대를 만들어 보겠습니다. Terraform이 설치되어 있고, AWS Access Key와 Secret Access Key가 설정되어 있다고 가정해 볼게요.

{{< gist rubysoho07 1915bc95eaf255409300a37da01ab32c >}}

우선 위 코드를 복사해서 임의의 파일로 저장합니다.

서버의 구성은 다음과 같습니다. 

{{< figure src="/img/vpc-reachability-analyzer-1.svg" >}}

* Source EC2: 어디에서든 80 포트로 진입이 가능하며, 모든 나가는 트래픽을 허용합니다.
* Target Available EC2: 어디에서든 80 포트로 진입이 가능하며, 모든 나가는 트래픽을 허용합니다. 
* Target Unavailable EC2: **__트래픽이 들어올 수 있는 방법은 없지만__**, 모든 나가는 트래픽을 허용합니다.

그리고 Terraform으로 서버를 생성해 줍니다.

```shell
terraform init
# (출력 결과 생략)
terraform apply -auto-approve
# (출력 결과 생략)
```

콘솔에 들어가 보면, 아래 스크린샷과 같이 서버 3 대가 생성된 것을 확인할 수 있습니다.

{{< figure src="/img/vpc-reachability-analyzer-2.png" >}}

생성된 각 인스턴스의 ID를 메모하는 것을 권장합니다.

### 테스트 해 보기 

콘솔에서 'Reachability Analyzer'를 검색해서 실행합니다. 

{{< figure src="/img/vpc-reachability-analyzer-3.png" >}}

영어 콘솔 기준으로 주황색의 'Create and Analyze Path' 버튼을 누릅니다. 

이제 하나씩 테스트 해 보겠습니다. 

#### 첫번째 테스트: 성공하는 경우

우선 다음과 같이 입력합니다.

**Path Source**
* Source type: Instances (EC2 인스턴스를 의미)
* Source: EC2 인스턴스 목록에서 `source` 표시가 있는 인스턴스를 선택
* 'Additional packet header configurations at source' 항목을 펼쳐서 Destination port에 80 입력

**Path Destination**
* Destination type: Instances (EC2 인스턴스를 의미)
* Destination: EC2 인스턴스 목록에서 `target_available` 표시가 있는 인스턴스를 선택

Protocol: TCP 선택 (기본값)

그리고 아래로 내려가서 Create and analyze path 버튼을 클릭합니다. 

화면이 바뀐 후에는 Pending 상태가 되는데, 잠깐 기다리면 아래와 같이 분석이 끝납니다. 

{{< figure src="/img/vpc-reachability-analyzer-4.png" >}}

조금 더 내려가면 Path details 항목이 나오는데, 어떤 과정을 거쳐서 목적지에 접근 가능한 지 알 수 있습니다.

{{< figure src="/img/vpc-reachability-analyzer-5.png" >}}

과정을 정리해 보면, EC2 인스턴스에 연결된 네트워크 인터페이스(ENI) → source 인스턴스의 보안 그룹 → target_available 인스턴스의 보안 그룹 → 타겟 인스턴스에 연결된 네트워크 인터페이스를 거쳐서 타겟 인스턴스와 통신하였음을 알 수 있습니다.

#### 두번째 테스트: 실패하는 경우

이번에는 실패하는 경우를 테스트 해 보도록 하겠습니다. 다시 Reachability Analyzer로 돌아가서 Create and analyze path를 클릭합니다. 

다음과 같이 입력합니다. 달라져야 하는 부분은 굵은 글씨로 표시했습니다. 

**Path Source**
* Source type: Instances (EC2 인스턴스를 의미)
* Source: EC2 인스턴스 목록에서 `source` 표시가 있는 인스턴스를 선택
* 'Additional packet header configurations at source' 항목을 펼쳐서 Destination port에 80 입력

**Path Destination**
* Destination type: Instances (EC2 인스턴스를 의미)
* Destination: **EC2 인스턴스 목록에서 `target_unavailable` 표시가 있는 인스턴스를 선택**

Protocol: TCP 선택 (기본값)

그리고 아래로 내려가서 Create and analyze path 버튼을 클릭합니다. 

조금 기다리면, 아래 스크린샷과 같이 **Not reachable**로 표시되는 것을 볼 수 있습니다.

{{< figure src="/img/vpc-reachability-analyzer-6.png" >}}

어느 지점에서 문제가 발생했는지 알아보고 싶으시다면 아래로 스크롤을 내려봅니다. 

{{< figure src="/img/vpc-reachability-analyzer-7.png" >}}

위의 스크린샷을 보시면 **None of the ingress rules in the following security groups apply** 라는 문구를 볼 수 있습니다. 이 인스턴스와 연결된 보안 그룹에는 ingress 규칙을 설정하지 않았는데요. 그렇기 때문에 접속에 실패하는 것이 당연합니다.

### 리소스 정리하기

모든 테스트가 끝났다면 사용한 리소스를 정리합니다. 

```shell
terraform destroy -auto-approve
# (출력 결과 생략)
# 아래와 같은 결과가 나왔는지 꼭 확인하세요!
Destroy complete! Resources: 6 destroyed. 
```

## 마무리하며

Reachability Analyzer는 정말 편한 기능이지만, 아쉽게도 무료 서비스는 아닙니다. 서울 리전 기준으로 한번 분석을 수행할 때마다 $ 0.1 이 부과됩니다. 앞에서도 말씀드렸지만, 인스턴스 내부에 있는 애플리케이션이 트래픽을 차단하는 경우에는 Reachability Analyzer로 감지하기 어렵다는 단점도 있습니다. 이 부분을 고려하셔서 네트워크 문제를 해결해 보시기를 바라겠습니다. 

읽어주셔서 감사합니다. 
