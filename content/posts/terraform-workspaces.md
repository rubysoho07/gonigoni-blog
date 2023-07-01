---
title: "Terraform Workspaces 기능 정리"
date: 2023-03-19T17:21:21+09:00
tags: [DevOps, IaC, Terraform]
draft: false
categories: [DevOps]
comments: true
---

Terraform에서 개발/운영 환경을 나누기 위해, 폴더/디렉터리를 이용할 때가 있습니다. 하지만 이런 상황도 있죠. 

1. 개발/운영 환경 내 staging/QA 환경이 필요한 경우
2. 테스트를 위해 임시 인프라 구성이 필요한 경우

위와 같은 상황에 대응하기 위해, Terraform에서는 Workspaces 라는 기능을 지원합니다. 이번에는 Terraform의 Workspaces 기능은 무엇인지, 어떻게 사용하면 되는지 알아보려고 합니다. 

## Workspaces를 사용하려면?

Terraform에서 관리하는 리소스 상태는 backend에 저장합니다. 이러한 데이터는 하나의 workspace에 속합니다. 기본적으로 이런 리소스 상태는 default workspace에 들어 있습니다. 몇몇 backend는 여러 개의 workspace를 지원하며, 하나의 설정과 연결된 여러 상태를 저장할 수 있습니다. 

* AzureRM
* Consul
* COS
* GCS
* Kubernetes
* Local
* OSS
* Postgres
* Remote
* S3

새로운 백엔드를 설정하거나 인증 정보를 변경하지 않아도, 여러 인스턴스를 배포할 수 있다는 것이 장점입니다. Hashicorp의 공식 문서에서는 workspaces 기능이 인증 정보나 권한이 분리된 상황에는 적절하지 않다고 이야기 하고 있습니다. 

## Workspace 기능 사용하기

Terraform은 default라는 이름의 기본 workspace로 시작합니다. 이 workspace는 삭제할 수 없습니다. 따로 workspace를 만들지 않았다면, default workspace를 사용하고 계신 겁니다. 

terraform plan 을 새로운 workspace에서 수행하면, Terraform은 다른 workspace에 있는 리소스에 접근하지 않습니다. 기존에 있던 리소스를 관리하려면 workspace를 전환하면 됩니다.

그러면 실제로 한 번 써 볼까요?

## CLI에서 Workspace 관리하기 

먼저 [S3를 backend로 지정](https://developer.hashicorp.com/terraform/language/settings/backends/s3)해 보려고 합니다. 이를 위해 S3 버킷 하나와 DynamoDB 테이블 하나를 준비합니다. (AWS CLI를 설치 후, 설정까지 완료하셨다고 가정하겠습니다)

`(YOUR_BACKEND_BUCKET)`과 `(YOUR_BACKEND_LOCK_TABLE)`값은 적절하게 바꿉니다.

```shell
# S3 버킷 만들기
aws s3 mb s3://(YOUR_BACKEND_BUCKET)

# State Lock을 위한 DynamoDB Table 만들기
aws dynamodb create-table --table-name (YOUR_BACKEND_LOCK_TABLE) \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST
```

버킷과 DynamoDB 테이블은 테스트가 끝나면 정리할 예정입니다. 이 문서의 끝에 설명하였으니 참고해 주세요.

그리고 간략하게 Terraform 코드를 만들고, `main.tf` 라는 이름으로 저장합니다.

```terraform
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }

  backend "s3" {
    bucket = "(YOUR_BACKEND_BUCKET)"
    key = "main/main.tfstate"
    region = "ap-northeast-2"
    dynamodb_table = "(YOUR_BACKEND_LOCK_TABLE)"
  }
}

provider "aws" {
  region = "ap-northeast-2"
}
```

그리고 파일이 있는 폴더에서 터미널을 실행 후, `terraform init` 명령을 실행하여 초기화 합니다.

### Workspace 목록 확인하기

`$`은 터미널의 프롬프트를 의미하므로, 실제 명령 수행 시 제외하고 입력합니다.

```shell
$ terraform workspace list
# Output은 다음과 같습니다. 
# 초기 상태이니 default workspace만 있고, 해당 workspace가 선택되어 있습니다.
* default
```

### 새로운 Workspace 만들기 

`$`은 터미널의 프롬프트를 의미하므로, 실제 명령 수행 시 제외하고 입력합니다.

```shell
$ terraform workspace new test
# Output은 다음과 같습니다. 자동으로 새로 생성된 workspace로 전환됩니다.
Created and switched to workspace "test"!

You\'re now on a new, empty workspace. Workspaces isolate their state,
so if you run "terraform plan" Terraform will not see any existing state
for this configuration.

# Workspace 전환 여부를 확인하려면 다음과 같이 입력합니다.  
$ terraform workspace list
# Output은 다음과 같습니다. 새로 생성된 workspace로 전환되어 있음을 볼 수 있습니다.
  default
* test
```

### Workspace 전환하기 

`$`은 터미널의 프롬프트를 의미하므로, 실제 명령 수행 시 제외하고 입력합니다.

```shell
$ terraform workspace select default
# Output은 다음과 같습니다. 
Switched to workspace "default".

# 그럼 workspace 전환 여부를 확인해 보겠습니다. 
$ terraform workspace list
# Output은 다음과 같습니다. 
* default
  test
```

### Workspace 삭제하기

`$`은 터미널의 프롬프트를 의미하므로, 실제 명령 수행 시 제외하고 입력합니다.

```shell
# 현재 선택된 workspace는 삭제할 수 없습니다. 
$ terraform workspace delete default
Workspace "default" is your active workspace.

You cannot delete the currently active workspace. Please switch
to another workspace and try again.

# 하지만 다른 workspace는 삭제할 수 있어요.
$ terraform workspace delete test  
Deleted workspace "test"!
```

## 코드에서 Workspace 관리하기 

### 테스트용 S3 버킷 만들기 

앞서 만든 `main.tf` 파일 끝에 아래 내용을 추가합니다. 사용 중인 워크스페이스는 `${terraform.workspace}`와 같은 방법으로 참조할 수 있습니다.

```terraform
resource "aws_s3_bucket" "test" {
  bucket = "your-test-bucket-${terraform.workspace}"
}
```

저는 현재 default workspace에서 작업 중인데요. `terraform plan` 명령을 실행하면 다음과 같이 `your-test-bucket-default` 라는 버킷을 생성할 거라고 합니다. 

```shell
$ terraform plan 

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.test will be created
  + resource "aws_s3_bucket" "test" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "your-test-bucket-default"
      + bucket_domain_name          = (known after apply)
# (... 이하 생략)
```

`terraform apply` 명령을 이용해서 리소스를 생성하면 잘 생성되는 것을 확인할 수 있습니다. (실행할 것인지 물어보면 `yes` 라고 답하시면 됩니다)

### 다른 Workspace용으로 리소스 만들기

그럼 다른 workspace를 생성하고, 해당 workspace를 선택해서 리소스를 생성해 보겠습니다. 먼저 새로운 workspace를 생성합니다. 

(`$`은 터미널의 프롬프트를 의미하므로, 실제 명령 수행 시 제외하고 입력합니다.)

```shell
$ terraform workspace new staging
Created and switched to workspace "staging"!

You're now on a new, empty workspace. Workspaces isolate their state,
so if you run "terraform plan" Terraform will not see any existing state
for this configuration.
```

자동으로 staging workspace를 선택하였으니, **코드를 수정하지 않고** 바로 진행해 보겠습니다. 

```shell
$ terraform plan 

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.test will be created
  + resource "aws_s3_bucket" "test" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "your-test-bucket-staging"
      + bucket_domain_name          = (known after apply)
      + bucket_regional_domain_name = (known after apply)
      + force_destroy               = false
      + hosted_zone_id              = (known after apply)
      + id                          = (known after apply)
      + object_lock_enabled         = (known after apply)
      + policy                      = (known after apply)
      + region                      = (known after apply)
      + request_payer               = (known after apply)
      + tags_all                    = (known after apply)
      + website_domain              = (known after apply)
      + website_endpoint            = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

위의 결과를 확인해 보시면, 기존 버킷(`your-test-bucket-default`)은 체크하지 않고, `your-test-bucket-staging` 버킷을 생성할 예정이라고 합니다. 

이렇듯, workspace를 이용하면 인프라 복제가 필요할 때, 기존 인프라에 영향을 주지 않고 동일하게 구성할 수 있다는 장점이 있습니다.

`terraform apply`를 수행한 후, 진짜 두 개의 버킷이 생성되어 있는지 확인해 볼까요?

```shell
$ aws s3 ls | grep your-test

# 출력 결과
2023-03-20 00:07:42 your-test-bucket-default
2023-03-20 00:07:43 your-test-bucket-staging
```

역시 두 개의 버킷이 생성된 것을 보실 수 있습니다.

## State는 어디에 있을까?

AWS를 사용하시는 경우, 협업을 위해 S3를 backend로 지정하는 경우가 많으실 텐데요. Workspace를 분리하면 어떻게 저장되는지 살펴 보겠습니다. 

AWS CLI를 이용하여, 백엔드로 설정한 버킷을 조회해 봅니다. 

```shell
$ aws s3 ls --recursive s3://(YOUR_BACKEND_BUCKET)

# 출력 결과
2023-03-20 00:19:27       2663 env:/staging/main/main.tfstate
2023-03-20 00:17:26       2663 main/main.tfstate
```

* Default workspace의 경우, `main/main.tfstate`에 상태가 저장됩니다. 
* 앞에서 `staging` 이라는 workspace를 만들었는데, 이 workspace의 상태는 `env:/(workspace 이름)/main/main.tfstate` 파일에 저장되는 것을 볼 수 있습니다.

## 리소스 정리 

우선 `terraform apply`로 생성한 리소스부터 삭제합니다. 

```shell
# staging workspace의 리소스부터 삭제합니다. 
terraform workspace select staging

# `-auto-approve` 옵션 사용 시 확인하지 않고 리소스를 바로 삭제합니다. 
terraform destroy -auto-approve

# default workspace로 전환합니다. 
terraform workspace select default

# default workspace의 리소스를 삭제합니다. 
terraform destroy -auto-approve
```

Backend를 구축하기 위해 생성한 리소스도 삭제합니다. 

```shell
# DynamoDB 테이블을 삭제합니다. 
aws dynamodb delete-table --table-name (YOUR_BACKEND_LOCK_TABLE)

# S3 버킷을 삭제합니다. (버킷 안의 내용을 비워야 삭제 가능함에 유의)
aws s3 rm --recursive s3://(YOUR_BACKEND_BUCKET)/
aws s3 rb s3://(YOUR_BACKEND_BUCKET)
```

읽어주셔서 감사합니다.

## 참고자료

**Terraform 문서**
* [Workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)
* [Managing Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)
