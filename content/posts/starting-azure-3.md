---
title: "AWS만 쓰던 사람이 Azure 써 보는 이야기: (3) 스토리지"
date: 2026-01-14T22:50:00+09:00
tags: [Azure]
draft: false
categories: [Azure]
comments: true
---

[1편]({{< ref "starting-azure-1.md" >}}), [2편]({{< ref "starting-azure-2.md" >}})에 이어서 AWS만 써 온 사람이 Azure를 사용해 보는 이야기를 이어가겠습니다. 저는 인프라의 가장 중요한 구성요소는 네트워크, 컴퓨팅 리소스, 스토리지라고 생각하는데요. 이번 글에서는 인프라의 가장 중요한 요소 중 하나인 스토리지에 대한 이야기를 해 보겠습니다. 

## Azure의 스토리지 서비스 비교하기

파일을 저장할 곳을 생각한다면 다음과 같은 기준으로 선택을 하지 않을까 싶습니다. 

- 서버 내 파일 시스템에 마운트 하는 HDD/SSD → 블록 스토리지
- 서버에 마운트하지 않고, 자유롭게 업로드/다운로드 하는 스토리지 → 객체 스토리지
- 여러 서버에서 동시에 연결할 수 있는 스토리지 → 네트워크 스토리지 (NFS/SMB 프로토콜 기반)

AWS와 Azure에서 각각의 용도에 맞는 서비스를 비교해 보면 다음과 같습니다. 하나씩 살펴보시죠. 

| 종류               | AWS 서비스         | Azure 서비스          |
|------------------|------------------|---------------------|
| 블록 스토리지        | EBS              | Azure Disk          |
| 객체 스토리지        | S3            | Blob Storage               |
| 네트워크 스토리지 | EFS, FSx      | 파일 공유 |

## 블록 스토리지 

AWS에는 EBS가 있고, Azure에서는 동일한 서비스로 Azure Disk가 있습니다. 어떤 문서에서는 Managed Disk로 설명할 때도 있네요. 

어떤 종류의 디스크가 있는지는 [Azure 관리 디스크 형식](https://learn.microsoft.com/ko-kr/azure/virtual-machines/disks-types) 문서에서 확인해 볼 수 있습니다.

디스크를 생성하는 방법과 Linux 서버에 연결하는 방법은 생략하겠습니다. 아래 참고자료를 한 번 살펴보시면 좋을 것 같네요. 

[포털을 사용하여 Linux VM에 데이터 디스크 연결](https://learn.microsoft.com/ko-kr/azure/virtual-machines/linux/attach-disk-portal)

## 객체 스토리지

AWS의 S3은 이름처럼 간단하게 사용할 수 있습니다. 버킷을 만들고, boto3와 같은 AWS SDK를 이용하면 쉽게 파일을 업로드 하거나 다운로드 할 수 있기 때문이죠. 그리고 용량 제한도 없습니다. 

Azure에서도 S3와 비슷한 객체 스토리지가 있습니다. Blob Storage라는 이름의 서비스인데요. 다만 S3와 동작 방식에 차이가 있습니다. 

Azure의 Blob Storage는 아래와 같은 방식으로 구성됩니다. 

{{< figure src="https://learn.microsoft.com/ko-kr/azure/storage/blobs/media/storage-blobs-introduction/blob1.png" caption="출처: https://learn.microsoft.com/ko-kr/azure/storage/blobs/storage-blobs-introduction">}}

- 스토리지 계정: 하나의 S3 버킷과 비슷한 개념입니다. `gonigoni`라는 스토리지 계정이 있다면 `https://gonigoni.blob.core.windows.net`이라는 엔드포인트로 접근합니다.
- 컨테이너: 스토리지 계정 내 디렉토리로 생각하시면 됩니다. `images`라는 컨테이너가 있다면, `스토리지 계정 엔드포인트/images` 와 같은 방법으로 URL이 생성됩니다.
- Blob: S3 버킷에 저장된 파일이 하나의 Blob입니다. `myfile.txt`라는 파일이 있다면, `스토리지 계정 엔드포인트/컨테이너 이름/myfile.txt`과 같은 방식으로 URL이 등록됩니다. 

AWS S3는 (엄밀히 말하면) 디렉토리 개념이 없고, 버킷과 키만 이용해서 파일을 관리합니다. Azure Blob은 S3와 다르게 스토리지 계정, 컨테이너, Blob의 구조로 파일을 관리하는 것이 차이점입니다. 

AWS boto3와 마찬가지로, Azure에서도 Blob 스토리지에 데이터를 업로드/다운로드 할 수 있는 SDK를 제공합니다. 

- Python: [azure-storage-blob](https://github.com/Azure/azure-sdk-for-python/tree/azure-storage-blob_12.28.0/sdk/storage/azure-storage-blob)
- JavaScript: [Azure Storage Blob Client Library for JavaScript](https://github.com/Azure/azure-sdk-for-js/tree/main/sdk/storage/storage-blob)

## 네트워크 스토리지

여러 서버가 하나의 스토리지에 접속해야 하는 경우가 있습니다. 이럴 때 NFS나 SMB와 같은 프로토콜로 파일을 공유할 수 있는데요. 개인적인 느낌이지만, SMB로 파일을 공유하면 Azure가 접근성이 좋았고, NFS로 파일을 공유한다면 AWS가 좀 더 접근성이 좋다는 생각이 들었습니다.

**SMB 프로토콜로 파일 공유하기**

Windows Server를 사용할 일이 거의 없어서 SMB에 대해서는 잘 모릅니다. 다만 찾아 보니 AWS에서는 FSx for Windows File Server라는 서비스가 있네요. Azure에서는 스토리지 계정이 있다면, **파일 공유** 기능으로 SMB로 파일을 공유할 수 있습니다. 하지만 같은 스토리지 계정을 사용한다고 해서 Blob Storage에 있는 파일에 접근 가능한 게 아닐까? 라고 생각했는데 그건 아니었습니다. 파일 공유 기능을 사용하면 Blob 스토리지와는 별개의 공간을 만든다고 생각하셔야 합니다. 

스토리지 계정에서 [파일 공유를 생성](https://learn.microsoft.com/ko-kr/azure/storage/files/create-classic-file-share?tabs=azure-portal)하면, Windows, Linux, macOS 모두 SMB 프로토콜로 연결할 수 있습니다. 아래 스크린샷은 Linux에 SMB 파일 공유를 연결하는 방법입니다. 

{{< figure src="/img/starting-azure-3-1.png" >}}

1. 파일 공유에 진입 후 **연결**을 클릭합니다. 
2. Linux 서버에 연결할 거라 Linux를 클릭합니다. 
3. 스크립트 표시를 클릭하면, 스크립트가 나옵니다. 디렉토리 생성, 사용자 및 비밀번호 정보, `/etc/fstab` 파일 수정(서버 재부팅 후에도 마운트하도록 하기 위함), 마운트 명령을 확인할 수 있습니다. 그대로 입력하면 SMB 파일 공유가 마운트 된 것을 확인할 수 있습니다.

**NFS 프로토콜로 파일 공유하기**

AWS에서는 EFS 파일 시스템과 Access Point를 생성한 다음 Linux 서버에 마운트 하여 사용할 수 있습니다. Azure에서는 Blob Storage를 NFS v3으로 연결할 수 있습니다.

NFS를 사용하려면 기존의 스토리지 계정을 이용할 수 없기 때문에, 스토리지 계정을 따로 만들어야 합니다. 스토리지 계정을 만들 때, 다음 내용을 확인해서 만들어 주세요. 포털 기준으로 설명하면 다음과 같습니다.

(기본 탭)
- 중복도: LRS / ZRS 중 선택

(고급)
- REST API 작업을 위한 보안 전송 필요: 선택 해제
- 계층 구조 네임스페이스 사용: 선택 
- 네트워크 파일 시스템 v3 사용: 선택

(네트워킹)
- 공용 네트워크 액세스: 사용 안 함
- 프라이빗 엔드포인트 추가
    - 이름: 임의로 지정
    - 스토리지 하위 리소스: blob 또는 file로 선택
    - 가상 네트워크 및 서브넷 선택

스토리지 계정을 만든 후, 컨테이너를 만듭니다. 

Terraform으로 구성하는 경우, [링크](https://gist.github.com/rubysoho07/f1befc56ca10aad80d8c66c64ac3987c)를 참고해 주세요. 

서버에 마운트할 때는 AZNFS Mount Helper라는 도구를 사용해야 합니다. 아래 링크한 내용을 참고해서 마운트 해 보시면 됩니다.

- [AZNFS 설치 방법](https://learn.microsoft.com/ko-kr/azure/storage/blobs/network-file-system-protocol-support-how-to#step-5-install-the-aznfs-mount-helper-package)
- [컨테이너 탑재 방법](https://learn.microsoft.com/ko-kr/azure/storage/blobs/network-file-system-protocol-support-how-to#step-6-mount-the-container)

최근 [NFS v4.1 버전을 지원하는 파일 공유](https://learn.microsoft.com/ko-kr/azure/storage/files/files-nfs-protocol)가 있다고 하는데, 지원하는 리전이 제한적이어서 테스트 해 보지는 못했습니다. (참고로 Korea Central(서울)리전은 선택 불가능했는데, Korea South(부산)리전은 선택 가능했습니다)

## 마무리

지금까지 AWS의 스토리지와 비교하여 Azure의 여러 스토리지 서비스를 알아봤습니다. 최대한 간단하게 설명하려고 노력은 했는데, 도움이 되실지는 잘 모르겠습니다. 

다음 글을 마지막으로 AWS 사용자가 Azure를 사용해 본 이야기를 마무리 하려고 합니다. 다음 글에서는 컨테이너 관련 서비스에 대해 설명해 볼 예정입니다. 아마 Kubernetes 위주로 설명을 드리지 않을까 싶습니다. 

읽어주셔서 감사합니다.

# 참고자료

* (AWS) [Cloud Storage on AWS](https://aws.amazon.com/ko/products/storage/)
* (AWS) [NFS와 SMB의 차이점은 무엇인가요?](https://aws.amazon.com/ko/compare/the-difference-between-nfs-smb/)
* (Azure) [Azure Storage 소개](https://learn.microsoft.com/ko-kr/azure/storage/common/storage-introduction)
