---
title: "AWS만 쓰던 사람이 Azure 써 보는 이야기: (2) 서버"
date: 2025-12-28T08:04:10+09:00
tags: [Azure]
draft: false
categories: [Azure]
comments: true
---

지난 글에 이어서 AWS만 써 온 사람이 Azure를 사용해 보는 이야기를 계속하겠습니다. 이번 글에서는 AWS와 비교해서 Azure에서 가상 머신을 관리하는 방법을 이야기 해 보려고 합니다. 이번 글에서는 Azure CLI와 Terraform을 사용하는 경우가 많아서 이러한 툴이 설치되어 있으면 좋습니다. 

글이 길어서 이번 글에서 다루는 내용을 간단하게 요약하겠습니다. 아래 내용을 참고해 주세요.

* EC2 vs 가상 머신
    * 서버 크기 선택
    * OS 이미지 선택
    * 네트워크 설정
    * SSH 키 페어 관리
* 오토 스케일링 그룹 vs VMSS(가상 머신 확장 집합)

## 가상 머신

AWS의 EC2와 같은 서비스로 Azure의 가상 머신(Virtual Machines) 서비스가 있습니다. 포털에서 가상 머신을 만드는 방법을 전부 알려드리지 않고, 차이점 위주로 설명해 보겠습니다.

### 서버 크기 선택

EC2에 다양한 인스턴스 타입이 있듯, Azure에서도 여러가지 크기의 가상 머신을 지원합니다. 처음에 가상 머신의 크기를 선택할 때 B, D와 같은 타입이 어떤 의미인지 잘 몰라서 Azure의 [가상 머신 크기](https://learn.microsoft.com/ko-kr/azure/virtual-machines/sizes/overview), [Azure Virtual Machine 크기 명명 규칙](https://learn.microsoft.com/ko-kr/azure/virtual-machines/vm-naming-conventions) 문서를 참고하여 작성했습니다.

서버 크기에는 어떤 규칙이 있는지 알아 보겠습니다. 기본적으로는 `[패밀리], [vCPU 수], [추가기능], [버전]`으로 구성됩니다.

- 패밀리: 범용, 컴퓨팅 최적화, 메모리 최적화, 스토리지 최적화, GPU 가속과 같은 특성에 따라 나뉩니다. 뒤에 알파벳 대문자가 추가로 붙는 경우가 있는데, 여러 용도에 따라 세분화하여 구분하기 위해 사용됩니다.
- vCPU 수: 서버의 CPU 코어 수를 의미합니다.
- 추가기능: AMD나 ARM 기반 CPU 관련 옵션, 메모리 또는 스토리지 관련 옵션이 붙습니다. 소문자로 표기합니다.
- 버전: EC2의 `t` 타입 인스턴스도 `t2, t3, t4`와 같이 세대 번호가 붙는데, 이와 비슷하다고 보시면 될 것 같습니다. 보통 인스턴스 크기 이름 끝에 `v2`, `v3`과 같은 방식으로 표기합니다.

예를 들어서 `D2as_v6` 크기의 서버를 만든다고 하면, 이런 특성이 있는 서버라고 유추할 수 있습니다. 
* D: 범용 서버입니다. EC2의 `m` 타입 서버와 용도가 비슷하다고 보면 되겠습니다. 
* 2: vCPU 개수가 2개입니다. 
* a: AMD CPU를 사용합니다.
* s: 프리미엄 SSD 형식과 호환됩니다. 
* v6: 6세대 버전의 서버입니다.

EC2의 인스턴스 패밀리와 어떻게 매칭할 수 있을지 정리해 보면 다음과 같습니다. [EC2 인스턴스 유형](https://aws.amazon.com/ko/ec2/instance-types/) 문서도 같이 확인해 보세요. 

| Azure 패밀리 | AWS 인스턴스 패밀리 | 설명 |
| --- | --- | --- |
| B | t | 버스트 가능한 범용 인스턴스 |
| D | m | 범용 인스턴스 |
| E | r | 메모리 최적화 인스턴스 |
| F | c | 컴퓨팅 최적화 인스턴스 |
| L | i | 스토리지 최적화 인스턴스 |
| N | p, g | GPU 가속 인스턴스 |

### 가상 머신 이미지 선택

AWS에서는 EC2 인스턴스를 시작할 때 Amazon Linux, Ubuntu, Windows Server, Red Hat, SUSE, Debian 등을 선택할 수 있습니다. 이러한 운영 체제는 AMI(Amazon Machine Image)라는 이름의 가상 머신 이미지로 제공됩니다. Azure에서도 비슷합니다. Ubuntu, Debian, Red Hat, SUSE, Windows Server와 같은 이미지를 선택할 수 있습니다. 

**(1) 선택할 수 있는 가상 머신 이미지 정보**

Azure에서 CLI나 Terraform으로 가상 머신 생성을 자동화하려고 하는 경우, 가상 머신 이미지를 지정할 때 입력해야 하는 값들이 있습니다. 

* Publisher: 이미지를 만든 조직 (Canonical, RedHat, SUSE, Microsoft, ...)
* Offer: Publisher가 만든 이미지 그룹의 이름 (ubuntu-24_04-lts, RHEL, sles-15-sp5, WindowsServer, ...)
* SKU: 배포 릴리즈
* Version: SKU의 버전 번호

이러한 값들은 Azure CLI에서 조회 가능합니다. Azure CLI를 이용하신다면, `az vm image list --output table` 명령으로 조회할 수 있습니다. 위에서 말씀드린 네가지 요소들은 이 명령으로 확인할 수 있습니다. 참고로 `Urn` 항목에 있는 값은 `Publisher:Offer:SKU:Version` 순으로 표시됩니다.

Terraform으로 Ubuntu 24.04 LTS 서버를 구성한다면 다음과 같이 지정하면 됩니다. 

```hcl
resource "azurerm_linux_virtual_machine" "gonigoni_vm" {
  # 중간 생략
  source_image_reference {
    publisher = "Canonical"
    offer     = "ubuntu-24_04-lts"
    sku       = "server"
    version   = "latest"
  }
}
```

**(2) Marketplace 이미지 사용**

Rocky Linux와 같은 리눅스 배포판을 사용하려면 AWS나 Azure 모두 Marketplace에서 구독하셔야 합니다. 추가 비용이 없는 Rocky Linux로 가상 머신을 구성해 보겠습니다. 

포털에서 Marketplace로 진입하여 Rocky Linux를 검색하면 공식 이미지가 표시됩니다. 이번에는 AMD64 이미지를 사용해 보겠습니다. AMD64 이미지를 클릭한 뒤, 오른쪽 화면에서 **"사용 정보 + 지원"** 탭을 클릭합니다. 아래와 같은 화면을 보실 수 있습니다.

{{< figure src="/img/starting-azure-2-1.png" caption="Azure Marketplace에서 Rocky Linux 이미지 검색" >}}

Azure CLI로 사용가능한 이미지를 아래와 같이 검색합니다. `--publisher resf` 옵션으로 Publisher를 지정하고, `--offer rockylinux-x86_64` 옵션으로 Offer를 지정합니다. 아래와 같이 표시됩니다. 서버를 만들 때 버전은 `latest`로 지정해도 됩니다.

```bash
az vm image list --publisher resf --offer rockylinux-x86_64 --all --output table

# Output
Architecture    Offer              Publisher    Sku     Urn                                         Version
--------------  -----------------  -----------  ------  ------------------------------------------  ------------
# ...
x64             rockylinux-x86_64  resf         9-base  resf:rockylinux-x86_64:9-base:9.6.20250531  9.6.20250531
# ...
```

다만, **Marketplace 이미지를 이용할 때는 Plan 정보를 지정해야 합니다.** Plan 정보를 확인하려면, Azure CLI로 다음 명령을 입력합니다. Rocky Linux 9 버전(SKU가 `9-base`인 최신 버전)을 이용한다고 가정하겠습니다. Azure CLI로 아래 명령을 입력하세요. 

```bash
az vm image show --urn resf:rockylinux-x86_64:9-base:latest

# Output
{
  "accepted": false,
  # 중간 생략
  "name": "9-base",
  "plan": "9-base",
  "product": "rockylinux-x86_64",
  "publisher": "resf",
  # 중간 생략
}
```

출력된 결과 중 `accepted: false`인 경우 아래와 같이 사용 조건 약관에 동의해야 합니다. Azure CLI로 아래와 같이 사용 조건 약관에 동의할 수 있습니다.

```bash
az vm image terms accept --urn resf:rockylinux-x86_64:9-base:latest

# Output
{
  "accepted": true,
  # 중간 생략
  "name": "9-base",
  "plan": "9-base",
  "publisher": "resf",
  "product": "rockylinux-x86_64",
  # 중간 생략
}
```

`accepted: true` 로 변경된 것을 확인했습니다. 이제 Terraform 코드에서 다음과 같이 입력합니다. 

```hcl
resource "azurerm_linux_virtual_machine" "gonigoni_vm" {
  # 나머지는 생략 

  # 소스 이미지 지정
  source_image_reference {
    publisher = "resf"
    offer     = "rockylinux-x86_64"
    sku       = "9-base"
    version   = "latest"
  }

  # Plan 지정: `az vm image terms show` 명령의 결과를 참조하여 작성합니다.
  plan {
    publisher = "resf"
    product   = "rockylinux-x86_64"
    name      = "9-base"
  }
}
```

`terraform apply` 명령으로 서버 생성 후 접속합니다. `/etc/os-release` 파일을 확인하면 Rocky Linux 이미지를 사용하는 것을 볼 수 있습니다. 

```bash
$ ssh -i your_private_key.pem youradminuser@your_vm_public_ip
[youradminuser@gonigoni_vm ~]$ cat /etc/os-release
NAME="Rocky Linux"
VERSION="9.6 (Blue Onyx)"
```

### 네트워크 관련 설정

포털에서 서버를 생성할 때는 놓칠 수도 있는데, Terraform에서 서버를 생성해 보면서 두가지 내용을 알게 되었습니다.

첫번째로는 Terraform으로 가상 머신을 생성할 때, **매번 서버에 연결할 네트워크 인터페이스를 따로 생성하고 연결해야 한다**는 점이 AWS와 다릅니다. AWS와 비교하면 ENI와 EC2 인스턴스를 각각 정의 후 연결하는 것과 같다고 보시면 되겠습니다. (물론 EC2 인스턴스도 그렇게 만들 수는 있습니다. [Terraform AWS Provider 문서](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#network-and-credit-specification-example)를 참고해 주세요.) 

두번째로, 네트워크 인터페이스에는 네트워크 보안 그룹을 연결해야 합니다. 네트워크 보안 그룹을 설정하지 않으면 외부에서 접근이 불가능하기 때문입니다. 이 부분은 EC2에 보안 그룹을 연결하는 것과 큰 차이는 없어 보이네요.

그래서 Terraform으로 인스턴스를 구성하려면 아래와 같이 작성해야 합니다. (공인 IP가 붙은 서버를 생성한다고 가정합니다)

```hcl
# 가상 머신에서 사용할 공인 IP
resource "azurerm_public_ip" "gonigoni_vm_public_ip" {
  name                = "gonigoni-vm-public-ip"
  # 리소스 그룹, 위치 설정은 생략
  allocation_method = "Static"
}

# 네트워크 인터페이스
resource "azurerm_network_interface" "gonigoni_vm_nic" {
  name                = "gonigoni-vm-nic"
  # 리소스 그룹, 위치 설정은 생략

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.gonigoni_public_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.gonigoni_vm_public_ip.id
  }
}

# 네트워크 보안 그룹 (외부에서 SSH 접근 허용)
resource "azurerm_network_security_group" "gonigoni_vm_nsg" {
  name                = "gonigoni-vm-nsg"
  # 리소스 그룹, 위치 설정은 생략

  security_rule {
    name                       = "SSH"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# 네트워크 인터페이스와 네트워크 보안 그룹 연결
resource "azurerm_network_interface_security_group_association" "gonigoni_vm_nic_nsg_assoc" {
  network_interface_id      = azurerm_network_interface.gonigoni_vm_nic.id
  network_security_group_id = azurerm_network_security_group.gonigoni_vm_nsg.id
}

# 가상 머신
resource "azurerm_linux_virtual_machine" "gonigoni_vm" {
  name                  = "gonigoni-vm"
  # 리소스 그룹, 위치 설정은 생략
  size                  = "Standard_B2as_V2"
  admin_username        = "youradminuser"
  network_interface_ids = [azurerm_network_interface.gonigoni_vm_nic.id]

  # 중간 생략
}
```

### SSH 키 페어 관리

리눅스 기반 가상 머신에 접근할 때, 비밀번호 대신 SSH 키 페어를 이용해서 접근하는 경우가 많습니다. AWS에서 했던 것처럼 Azure도 포털에서 개인 키(private key. `.pem` 파일)를 다운로드 받아서 사용할 수 있습니다.

그런데, AWS와 Azure에서 SSH 키를 사용할 때 아래와 같은 차이가 있습니다. 
* EC2 인스턴스: 어떤 키 페어를 이용할 지 지정
* Azure 가상 머신: SSH 키의 공개 키(public key)를 지정해야 함

이런 차이를 이해하고 [Azure의 공식 문서](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-terraform)를 참고하여 Terraform으로 가상 머신을 생성하려고 했는데, 공식 문서에서는 개인 키를 받는 방법에 대한 설명이 없어서 삽질을 했던 경험이 있습니다. 이 문제를 어떻게 해결할 수 있는지 설명해 드릴게요. 

**(1) Azure 포털에서 키 페어를 발급받은 경우**

포털에서 키 페어를 발급했다면 개인 키 `.pem` 파일은 있으시겠죠? 다만 가상 머신을 생성할 때 공개 키를 지정해야 하기 때문에 아래와 같이 `data` 소스로 참조하여 가상 머신에 지정하면 됩니다. 

```hcl
data "azurerm_ssh_public_key" "test" {
  name                = "your_test_public_key"
  resource_group_name = "your_resource_group"
}

resource "azurerm_linux_virtual_machine" "gonigoni_vm" {
  # 중간 생략

  admin_ssh_key {
    username   = "youradminuser"
    # 아래와 같이 공개 키를 지정해 줍니다.
    public_key = data.azurerm_ssh_public_key.test.public_key
  }

  # 중간 생략
}
```

갖고 계신 개인 키로 서버에 접속하시면 됩니다.

**(2) `azapi` Terraform Provider로 키 페어를 생성한 경우**

위에 링크한 Azure의 공식 문서에서는 `azapi` Terraform Provider를 사용해서 키 페어를 생성합니다. 근데 이 문서에서는 개인 키에 대한 설명이 없어서 어려움이 있었습니다. 개인 키를 어떻게 받을 수 있을지 정리해 보겠습니다. 아래와 같이 Terraform 코드를 작성했다고 가정합니다.

```hcl
resource "azapi_resource_action" "ssh_public_key_gen" {
  type        = "Microsoft.Compute/sshPublicKeys@2022-11-01"
  resource_id = azapi_resource.ssh_public_key.id
  action      = "generateKeyPair"
  method      = "POST"

  # 아래 설정에 주목하세요!
  response_export_values = ["publicKey", "privateKey"]
}

resource "azapi_resource" "ssh_public_key" {
  type      = "Microsoft.Compute/sshPublicKeys@2022-11-01"
  name      = "my_ssh_key"
  # 리소스 그룹과 위치 정보는 생략
}

resource "azurerm_linux_virtual_machine" "gonigoni_vm" {
  # 중간 생략

  admin_ssh_key {
    username   = "gonigoni"
    # 생성한 공개 키를 참조합니다.
    public_key = azapi_resource_action.ssh_public_key_gen.output.publicKey
  }

  # 중간 생략
}

# 개인 키를 output으로 등록합니다.
output "ssh_private_key" {
  value = azapi_resource_action.ssh_public_key_gen.output.privateKey
  # plan이나 apply 시 개인 키를 노출하지 않기 위해 설정합니다.
  sensitive = true
}
```

Terraform에서 apply 명령을 수행하면 공개 키와 개인 키를 생성합니다. Output을 이용하여 터미널에서 아래 명령으로 개인 키를 생성할 수 있습니다.

```bash
terraform output -raw ssh_private_key > my_private_key.pem
chmod 400 my_private_key.pem
```

Output에 개인 키를 지정하는 게 불안하게 느껴질 수 있습니다. 이럴 때는 따로 output을 만들지 마시고 다음과 같이 하시면 됩니다. 

```bash
terraform console
# `>` 뒤의 내용을 입력합니다.
> azapi_resource_action.ssh_public_key_gen.output.privateKey
# EOT 사이에 출력된 내용을 복사해서 별도의 파일로 저장합니다. `chmod`로 400으로 권한을 변경해야 정상 실행 가능합니다.
```

이러한 방법으로 새로 생성한 리눅스 가상 머신에 접근할 수 있습니다.

## 오토 스케일링 그룹과 VMSS(가상 머신 확장 집합)

AWS에서 서버 개수를 자동으로 관리하기 위해 Auto Scaling Group(이하 ASG)을 이용하는데요. ASG를 생성하려면, Launch Template을 생성하고 ASG에 연결해야 합니다. 보통 ASG에서는 서브넷, 서버 대수 및 스팟 인스턴스 사용 여부와 같은 내용을 설정하고, Launch Template에서는 AMI, 인스턴스 타입, 스토리지와 같이 EC2 인스턴스와 관련된 내용을 설정합니다.

이와 비슷하게, Azure는 가상 머신 확장 집합(VMSS; Virtual Machine Scale Set)이라는 기능이 있습니다. **VMSS는 ASG + Launch Template과 같다고 생각하시면 됩니다.** AWS와 다르게 필요한 속성을 하나의 리소스에서 설정할 수 있습니다.

Terraform으로 2대의 Ubuntu 24.04 서버가 있는 VMSS를 배포해 보겠습니다. SSH 키와 네트워크 보안 그룹은 따로 생성했고, 로드 밸런서는 연결하지 않는 것으로 가정하겠습니다.

(2026-01-14 수정: 최근 권장되는 방식은 Flexible Orchestration 입니다. ([링크 참조](https://learn.microsoft.com/ko-kr/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes)) 이를 바탕으로 Terraform 코드를 수정했습니다.)

```hcl
resource "azurerm_orchestrated_virtual_machine_scale_set" "gonigoni_test" {
  name                        = "gonigoni-vmss"
  resource_group_name         = azurerm_resource_group.gonigoni.name
  location                    = azurerm_resource_group.gonigoni.location
  platform_fault_domain_count = 1
  instances                   = 2                  # 서버 대수

  sku_name = "Mix"
  sku_profile {
    # 서버 타입: 최대 5개까지 설정 가능
    vm_sizes = ["Standard_D2as_v6"]

    # allocation_strategy에 대한 설명: https://learn.microsoft.com/ko-kr/azure/virtual-machine-scale-sets/instance-mix-overview
    # LowestPrice가 기본값이며, 안정성이 중요하면 CapacityOptimized를 사용
    allocation_strategy = "CapacityOptimized"
  }

  os_profile {
    linux_configuration {
      admin_username = "azureuser"

      # SSH Key
      admin_ssh_key {
        username   = "azureuser"
        public_key = data.azurerm_ssh_public_key.gonigoni_v2.public_key
      }
    }
  }

  # Ubuntu 24.04 LTS
  source_image_reference {
    publisher = "Canonical"
    offer     = "ubuntu-24_04-lts"
    sku       = "server"
    version   = "latest"
  }

  # OS Disk
  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }

  # 네트워크 인터페이스
  network_interface {
    name    = "gonigoni-vmss-nic"
    primary = true

    ip_configuration {
      name      = "internal"
      primary   = true
      subnet_id = azurerm_subnet.gonigoni_private_subnet.id
    }

    # 네트워크 보안 그룹과 연결
    network_security_group_id = azurerm_network_security_group.gonigoni_vmss_nsg.id
  }
}
```

VMSS를 생성한 후 포털에 들어가 보니, 서버도 안 보이고 네트워크 인터페이스도 안 보입니다. 뭔가 잘못된 것이 아닌가 싶었는데, 포털에서 **VMSS에 속한 서버와 네트워크 인터페이스는 생성한 VMSS에 진입 후 확인 가능**합니다. 아래의 스크린샷과 같이 진입해서 확인하면 됩니다. AWS ASG가 관리하는 서버도 EC2 인스턴스 항목에서 확인 가능한 것과는 차이가 있네요.

{{< figure src="/img/starting-azure-2-2.png" caption="VMSS가 관리하는 인스턴스 확인" >}}

Private 서브넷에 서버를 생성하여 서버에 접속할 수는 없지만, 일단 2대의 서버를 배포할 수 있었습니다. 

## 마무리

생각보다 정리해야 할 내용들이 많아서 글이 길어졌습니다. 긴 글이지만 읽어주셔서 감사합니다. 틀린 부분이 있다면 댓글로 말씀해 주세요.