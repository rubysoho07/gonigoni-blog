---
title: "AWS만 쓰던 사람이 Azure 써 보는 이야기: (1) 네트워크"
date: 2025-12-20T13:33:23+09:00
tags: [Azure]
draft: false
categories: [Azure]
comments: true
---

제 커리어를 되돌아 보니, 저는 인프라를 구축하고 운영할 때 AWS만 사용해 왔습니다. AWS가 국내 뿐만 아니라 전세계에서 가장 시장 점유율이 높은 서비스임이 분명하지만, 실제로 인프라를 운영하는 입장에서는 다른 클라우드를 사용하는 경우도 많습니다. 게다가 많은 기업들은 여전히 데이터 센터에 서비스 인프라를 구축하기도 합니다. 현재 재직 중인 회사에서 개발한 솔루션은 다양한 인프라 환경에 배포하는 것을 전제로 하고 있는데요. 그러다가 최근 회사에서 Azure를 사용해 볼 수 있는 기회가 생겨서 Azure를 사용해 보았습니다. 만약 다른 클라우드 환경을 사용하게 된다면, 어떤 부분에 집중해서 인프라를 구성할 지에 대해 기록해 보려고 합니다.

이 시리즈는 여러 파트로 나눠서 작성하려고 합니다. 아마도 네트워크, 가상 머신 생성, 스토리지, Kubernetes에 대해 이야기 한다면 처음 Azure를 접하는 분께는 적절하지 않을까 싶습니다.

이번 글에서는 Azure 환경에서 네트워크를 구성하면서 경험한 것들을 이야기 해 보겠습니다. AWS에 익숙한 분들을 위해 최대한 AWS 서비스와 비교해서 설명하겠습니다. 

## Azure에 로그인

AWS에서는 콘솔이라고 하지만, Azure에서는 포털이라는 곳에서 리소스를 관리할 수 있습니다. 포털 주소는 [portal.azure.com](https://portal.azure.com) 입니다. AWS의 IAM User와 같이 특정 조직에 속한 사용자라면, **로그인 옵션** > **조직에 로그인** 을 선택 후, 조직의 도메인을 입력합니다. 조직의 도메인은 항상 `onmicrosoft.com` 으로 끝난다고 합니다. 이후에는 각자의 이메일 주소를 입력해서 로그인 하면 됩니다. 

## 한 눈에 비교하기 

AWS 계정을 하나 만들었다고 가정하겠습니다. 콘솔에 로그인 해서 각 리전의 VPC 서비스에 들어가면, 기본 VPC, 퍼블릭 서브넷, Internet Gateway, 라우트 테이블, Network ACL, 기본 보안 그룹이 있을 것입니다. Azure 포털에 로그인하면, 아무것도 없습니다. 기본적으로 **구독** 이 필요하고, 구독 아래에 **리소스 그룹**을 생성해야 합니다. 그 아래에 **가상 네트워크**를 구축하고, **서브넷**을 구축합니다. 라우팅 테이블(Azure에서는 **경로 테이블** 이라고 번역했네요)이나 보안 그룹에 대해서는 조금 있다가 설명하겠습니다. 

AWS와 Azure의 각 기능을 비교해 보면 다음과 같습니다. 

| 기능               | AWS 서비스         | Azure 서비스          |
|------------------|------------------|---------------------|
| 가상 네트워크        | VPC              | 가상 네트워크          |
| 서브넷             | 서브넷            | 서브넷               |
| 인터넷 연결         | 인터넷 게이트웨이      | _(따로 설정하는 부분이 없음)_         |
| 라우팅 테이블        | 라우팅 테이블        | 경로 테이블       |
| 네트워크 ACL      | 네트워크 ACL      | 네트워크 보안 그룹      |
| 보안 그룹          | 보안 그룹          | 네트워크 보안 그룹      |

구축할 내용을 하나의 그림으로 보여드리면 다음과 같습니다. (라우팅 테이블과 보안 그룹 관련 내용은 생략했습니다)

{{< figure src="/img/starting-azure-1-1.svg" caption="AWS와 Azure 네트워크 비교 (왼쪽: AWS / 오른쪽: Azure)" >}}

이후 설명은 Azure에 구독과 리소스 그룹이 설정되어 있음을 가정하겠습니다. 

## 가상 네트워크와 서브넷 구성

AWS의 VPC와 같은 기능인 가상 네트워크를 구성해 보겠습니다. VPC든 가상 네트워크든 [RFC 1918](https://datatracker.ietf.org/doc/html/rfc1918)에 정의된 Private IP 영역에 따라 구성하는 것이 좋습니다. Private 주소 영역은 다음과 같습니다. 

* 10.0.0.0        -   10.255.255.255  (10/8 prefix)
* 172.16.0.0      -   172.31.255.255  (172.16/12 prefix)
* 192.168.0.0     -   192.168.255.255 (192.168/16 prefix)

다만 위에 있는 모든 prefix를 사용할 수 있는 건 아니고 제약사항이 있습니다. 

* AWS VPC에서는 `/16` 블록까지만 사용할 수 있습니다. ([출처](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-cidr-blocks.html)) 그 외에도 제약사항이 있기 때문에, VPC를 설계할 때 출처 문서를 꼭 읽어보세요. 
* Azure 가상 네트워크를 설정할 때 `/2` 블록까지 설정할 수 있긴 합니다. 실제로도 그렇게 설정할 수 있을지는 모르겠네요. 저는 일단 `/16` 블록으로 설정했습니다.

AWS든 Azure든, VPC와 가상 네트워크 내부에 서브넷을 설정하여 네트워크를 구분할 수 있습니다. 가상 네트워크를 만든 후, 일정한 기준에 따라 서브넷을 생성하여 영역을 나눕니다. 가상 네트워크와 서브넷을 생성하는 방법은 생략하겠습니다. Terraform으로 구성한다면 다음과 같이 하시면 됩니다. (리소스 그룹을 정의하는 부분은 생략)

```hcl
resource "azurerm_virtual_network" "gonigoni_vnet" {
  name                = "gonigoni-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.gonigoni.location
  resource_group_name = azurerm_resource_group.gonigoni.name
}

resource "azurerm_subnet" "gonigoni_public_subnet" {
  name                 = "gonigoni-public-subnet"
  resource_group_name  = azurerm_resource_group.gonigoni.name
  virtual_network_name = azurerm_virtual_network.gonigoni_vnet.name
  address_prefixes     = ["10.0.0.0/22"]
}

resource "azurerm_subnet" "gonigoni_private_subnet" {
  name                 = "gonigoni-private-subnet"
  resource_group_name  = azurerm_resource_group.gonigoni.name
  virtual_network_name = azurerm_virtual_network.gonigoni_vnet.name
  address_prefixes     = ["10.0.16.0/20"]

  # 프라이빗 서브넷으로 만들기 위해 아웃바운드 액세스 비활성화
  default_outbound_access_enabled = false
}
```

가상 네트워크와 서브넷을 생성하면서 제가 느낀 AWS와 차이점은 다음과 같습니다. 

1. AWS와 다르게, Azure에서는 **서브넷을 생성할 때 가용 영역을 설정하지 않음**
2. Private 서브넷 설정 시 차이
    * AWS에서는 라우팅 테이블에서 `0.0.0.0/0`(외부 인터넷)에 대한 경로 설정을 Internet Gateway 대신 NAT 장비로 바꾸면 Private 서브넷이 됨 (아니면 `0.0.0.0/0` 관련 라우팅 항목을 삭제하는 방법도 있죠)
    * Azure에서는 서브넷을 생성하거나 설정을 수정할 때 **"프라이빗 서브넷 사용(기본 아웃바운드 액세스 없음)"** 옵션에 체크하면 Private 서브넷이 됨 (2026년 3월 말부터는 프라이빗 서브넷이 기본값임)
3. Azure에서 가상 네트워크를 만들 때, AWS의 Internet Gateway와 같이 외부와 통신하기 위한 장비를 확인할 수 없음. 외부 네트워크와 통신할 장비를 따로 설정하지 않아도 되나 봄

이런 차이점을 고려해서 네트워크를 구성하시면 될 듯 합니다. 

## 라우팅 테이블

그러면 라우팅 테이블을 따로 지정하지 않았다면, 기본적으로 트래픽이 어떻게 흐를까요? 이 부분은 [Azure 문서](https://learn.microsoft.com/ko-kr/azure/virtual-network/virtual-networks-udr-overview#system-routes)를 참조하여 알아보겠습니다. 참고로, Azure에서의 라우팅 테이블은 **경로 테이블**이라는 이름으로 번역되어 있습니다.

위 링크에서 **기본 시스템 경로** 항목을 보시면 다음과 같이 설정되어 있습니다. 별도의 라우팅 테이블을 설정하지 않았다면 아래와 같은 방식으로 경로를 설정합니다. AWS에서 기본 라우팅 테이블을 확인할 수 있는 것과 다르게, 이 라우팅 테이블은 Azure 포털에서 확인할 수 없습니다.

* 가상 네트워크 prefix: 해당 가상 네트워크 내부에서 통신
* `0.0.0.0/0`: 인터넷
* 그 외 통신하지 못하는 영역이 있는데, 자세한 내용은 위의 링크를 참고해 주세요. 

Azure의 Private 서브넷은 외부 인터넷으로 가는 경로(`0.0.0.0/0`)가 차단되었을 것입니다. 만약 프라이빗 서브넷에서 외부 통신을 위해 NAT이 필요하면 어떻게 해야 할까요?

* NAT Gateway를 따로 구성했다면 서브넷 설정에서 NAT Gateway를 지정하는 방법이 있습니다. **라우팅 테이블을 따로 설정하지 않아도 됩니다.**
* NAT 인스턴스를 구성하여 연결하려면, 라우팅 테이블에서 경로를 지정합니다. 그리고 서브넷과 라우팅 테이블을 연결해야 합니다. 자세한 내용은 [링크](https://learn.microsoft.com/ko-kr/azure/virtual-network/tutorial-create-route-table?tabs=portal)를 참고하세요. 

AWS에서 프라이빗 서브넷을 생성했다면 프라이빗 서브넷을 위한 라우팅 테이블을 따로 생성하고 서브넷을 연결해야 합니다. 반면 Azure에서는 NAT Gateway를 사용한다면 프라이빗 서브넷에 라우팅 테이블을 따로 세팅할 필요가 없다는 점이 달랐습니다.

## 네트워크 보안 그룹과 애플리케이션 보안 그룹

Azure의 [네트워크 보안 그룹](https://learn.microsoft.com/ko-kr/azure/virtual-network/network-security-groups-overview)은 AWS의 Network ACL과 비슷해 보입니다. 우선 순위, 포트, 프로토콜, 소스와 대상, 허용/차단 여부를 설정할 수 있는 것이 AWS의 Network ACL과 비슷하다는 생각이 들었습니다. 네트워크 보안 그룹에서는 우선순위가 높은 규칙을 기준으로 트래픽의 허용/차단 여부를 결정합니다. (아래 스크린샷을 보시면 어떤 느낌인지 이해하실 수 있을거예요. 아래 스크린샷은 개인 계정에서 캡처했습니다)

{{< figure src="/img/starting-azure-1-2.png" caption="AWS와 Azure 네트워크 비교 (왼쪽: AWS / 오른쪽: Azure)" >}}

네트워크 보안 그룹은 Network ACL과 마찬가지로 서브넷 단위로 연결할 수 있습니다. 하지만, Azure의 네트워크 보안 그룹은 Network ACL과 다르게 네트워크 인터페이스에도 연결할 수 있습니다. AWS의 ENI에 Network ACL을 연결할 수 있다고 생각하시면 되겠습니다. 허용 규칙만 정할 수 있는 AWS의 보안 그룹과 비교하면, 트래픽 허용 및 차단 규칙도 정할 수 있다는 점이 달랐습니다. 가상 서버를 만들 때 사용자가 설정한 네트워크 보안 그룹을 지정할 수 있습니다. 

네트워크 보안 그룹은 다음과 같이 소스와 대상을 설정할 수 있습니다.
* Any: 전체 네트워크에 대한 설정
* IP Addresses: CIDR 또는 특정 IP 주소에 대한 설정
* Service Tag: IP 주소 범위를 특정한 이름으로 나타낸 것으로 보시면 됩니다. ([참고](https://learn.microsoft.com/ko-kr/azure/virtual-network/service-tags-overview)) AWS의 Managed Prefix List나 VPC Endpoint 같은 느낌이 드네요.
* Application Security Group: 애플리케이션 보안 그룹이라는 것이 있는데, 이건 아래에서 조금 더 설명하겠습니다. 

[애플리케이션 보안 그룹](https://learn.microsoft.com/ko-kr/azure/virtual-network/application-security-groups)은 비슷한 역할을 하는 네트워크 인터페이스를 묶어서 관리할 수 있도록 하는 기능입니다. 예를 들어 설명해 보겠습니다.

* 웹 서버가 여러 대 있다면 각각의 서버에는 네트워크 인터페이스가 붙어 있을 것입니다. 웹 서버에 붙은 네트워크 인터페이스들을 애플리케이션 보안 그룹에 등록합니다. 
* 로드 밸런서에 연결된 네트워크 보안 그룹의 아웃바운드 규칙에, 타겟을 애플리케이션 보안 그룹으로 설정합니다. 그 외 트래픽은 차단합니다. 

이런 방식으로 트래픽을 관리하기 위해 애플리케이션 보안 그룹을 사용할 수 있습니다. AWS에서 보안 그룹을 관리할 때, 다른 보안 그룹을 소스나 타겟으로 지정하는 것과 비슷하다고 보시면 될 것 같습니다. 근데 네트워크 인터페이스를 수동으로 추가한다면, 관리할 때는 어떻게 자동화해야 할 지 궁금하네요.

## 마무리

당연한 말이지만 콘솔과 포털의 UI가 서로 다르고, Terraform으로 인프라를 구성할 때에도 다른 점이 많아서 어려움이 있었습니다. 한편, AWS에 비하면 국내 자료가 부족하고, Microsoft의 공식 문서가 그렇게 친절한 느낌은 아닌 것 같다는 생각이 들었어요. 다음 글에서는 가상 머신을 올리는 과정을 비교하면서 AWS와 Azure가 어떤 점이 다른지 알아 보겠습니다.

틀린 내용이 있다면 댓글로 말씀해 주세요. 읽어 주셔서 감사합니다. 

## 참고자료

- [가상 네트워크를 구성하기](https://learn.microsoft.com/ko-kr/training/modules/configure-virtual-networks/)
    - [Azure Virtual Network란?](https://learn.microsoft.com/ko-kr/azure/virtual-network/virtual-networks-overview)
    - [네트워크 엔지니어용 Azure](https://learn.microsoft.com/ko-kr/azure/networking/azure-for-network-engineers)
    - [Azure Virtual Network에 대한 아키텍처 모범 사례](https://learn.microsoft.com/ko-kr/azure/well-architected/service-guides/virtual-network)