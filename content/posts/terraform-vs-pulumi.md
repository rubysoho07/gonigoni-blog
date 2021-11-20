---
title: "Terraform vs Pulumi"
date: 2021-11-20T10:30:24+09:00
tags: [DevOps, IaC, Terraform, Pulumi]
draft: false
categories: [DevOps]
comments: true
---

클라우드 환경 내 인프라 구성이 복잡할수록 IaC(Infrastructure as Code) 툴을 고민하게 되는데요. 물론 AWS의 CloudFormation과 같이 클라우드 벤더가 제공하는 서비스가 있지만, 앞으로는 특정 벤더에 종속되는 것을 피하고 싶다는 생각이 들었습니다. 

그러다가 알아본 툴 중에 Terraform과 Pulumi가 있는데요. Terraform은 저도 개인적으로 사용해 본 적이 있었고, Pulumi는 익숙한 개발 언어(Python, TypeScript, Golang, ...)로 인프라 구성을 구축할 수 있다고 해서 관심을 갖게 되었습니다. 

이번 글에서는 AWS에서 EC2, S3 Bucket, IAM Policy, Role 정도를 만드는 정도로 테스트해 보겠습니다. 그림으로 표현하면 다음과 같습니다.

{{< figure src="/img/terraform-vs-pulumi-1.png" >}}

각 툴의 설치 방법은 다음 글이나 튜토리얼을 참고해 주세요. 

* Terraform: [Get Started - AWS](https://learn.hashicorp.com/collections/terraform/aws-get-started)
* Pulumi: [Get Started with AWS](https://www.pulumi.com/docs/get-started/aws/)

그리고 AWS Access Key 설정은 이미 진행했다고 가정하고 진행 하겠습니다. 환경 변수 또는 CLI를 통해서 설정하는 방법은 링크를 참고해 주세요.

* [환경변수로 설정하는 경우](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-envvars.html)
* [CLI를 이용하여 구성 및 자격 증명 파일 설정](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html)

# 인프라 구성 방법

## Terraform

인프라 구성 시 필요한 것들은 아래 문서를 참고하였습니다. 

* [Build Infrastructure](https://learn.hashicorp.com/tutorials/terraform/aws-build?in=terraform/aws-get-started)
* [AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

작성한 코드가 생각보다 길어서, 테스트 해 본 코드는 [링크](https://gist.github.com/rubysoho07/71ee30b353f51dae7b133297b7ad20e3)를 참고해 주세요.

링크한 코드는 다음과 같이 구성되어 있습니다.

* `terraform`: Terraform이 인프라를 배포할 때 사용하는 provider는 무엇인지, provider와 Terraform의 버전은 어떤 것을 이용할 것인지에 대한 설정입니다.
* `provider`: AWS Provider를 이용했습니다. Provider에서 사용할 수 있는 설정은 [링크](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#argument-reference)를 참고하세요.
* `resource`로 시작하는 것들: 각각의 리소스라고 보시면 됩니다.
    * EC2 인스턴스
    * EC2에서 사용할 보안 그룹
    * EC2 인스턴스 프로파일: IAM Role을 연결하기 위해 필요합니다.
    * S3 버킷
    * IAM Role, Policy: 여기서는 Inline Policy로 생성해 봤습니다. Role은 EC2가 작업을 수행할 수 있도록 AssumeRole 권한을 부여했습니다.

## Pulumi

디렉터리를 만들고, 새로운 프로젝트를 생성합니다. 저는 AWS와 Python을 사용할 예정이므로 `pulumi new aws-python`이라고 입력했습니다.

```shell
mkdir pulumi-test && cd pulumi-test
pulumi new aws-python
Manage your Pulumi stacks by logging in.
Run `pulumi login --help` for alternative login options.
Enter your access token from https://app.pulumi.com/account/tokens
    or hit <ENTER> to log in using your browser                   : 
```

그냥 엔터 키를 누르면 웹 브라우저가 실행되면서 로그인 창이 뜹니다. 저는 GitHub 계정으로 연결했습니다. 참고로 여러 명이 사용하면 무료가 아니니 이 점은 조심해 주세요.

로그인이 끝나면 프로젝트 이름, 스택 이름, 어느 리전을 이용할 것인지 물어봅니다. 대충 대답해 줍시다. (저는 서울 리전을 쓸 예정이라, `ap-northeast-2`를 입력했습니다)

여기서 프로젝트와 스택의 차이는 다음과 같습니다.

* 프로젝트: GitHub repository와 유사합니다. 코드로 구성된 단일한 저장소로 보면 됩니다. 
* 스택: 분리된 설정을 가진 코드

즉, Project Foo는 개발, 테스트, 운영 단계의 스택으로 나눌 수 있습니다. 또는 여러 리전에 배포해야 할 때와 같이 다른 설정으로 분리할 수도 있다고 합니다.

```shell
project name: (pulumi-test) 
project description: (A minimal AWS Python Pulumi program) 
Created project 'pulumi-test'

Please enter your desired stack name.
To create a stack in an organization, use the format <org-name>/<stack-name> (e.g. `acmecorp/dev`).
stack name: (dev) yungon-iac-test
Created stack 'yungon-iac-test'

aws:region: The AWS region to deploy into: (us-east-1) ap-northeast-2
```

이후에는 dependency를 받고(`pip`를 이용하는 것 같네요), 다음과 같은 메시지가 출력되면 성공입니다.

```shell
Finished installing dependencies
Your new project is ready to go! ✨

To perform an initial deployment, run 'pulumi up'
```

그리고 디렉토리를 살펴보면 다음과 같이 구성되어 있는 것을 확인할 수 있습니다. 

{{< figure src="/img/terraform-vs-pulumi-2.png" >}}

가상 환경이 `venv` 디렉토리에 구성되고, `__main__.py` 파일이 생성되어 있음을 볼 수 있습니다. 그 외에도 여러 파일이 있네요.

* Pulumi.yaml: 프로젝트 정의 파일
* Pulumi.(스택 이름).yaml: 스택에 대한 설정 파일

지금부터는 제가 구성하려는 인프라를 구성해 보겠습니다. `__main__.py` 파일을 수정해 보죠. 

파일을 열어보면 S3 버킷이 만들어져 있습니다. 이를 적절히 수정하고, 여러 인프라를 구성해 보았습니다.

전체 코드는 [링크](https://gist.github.com/rubysoho07/a270ed1e77fe4ca693b641165840d367)를 참고해 주세요.

제가 참고한 문서는 다음과 같습니다.

* [AWS: API Docs](https://www.pulumi.com/registry/packages/aws/api-docs/)
* [Inputs and Outputs](https://www.pulumi.com/docs/intro/concepts/inputs-outputs/)

Pulumi로 동일한 인프라를 구성하면서 달랐던 점은 다음과 같습니다. 예를 들어 보겠습니다.

```python
import json

import pulumi_aws as aws

bucket = aws.s3.Bucket('yungon-iac-test-bucket',
    tags={
        "Name": "yungon-iac-test-bucket",
        "CreatedBy": "yungon"
    }
)

test_policy = aws.iam.RolePolicy("yungon-test-role-policy",
    role=test_role.id,
    policy=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Action": ["s3:*"],
            "Effect": "Allow",
            "Resource": [
                bucket.arn,             
                f"{bucket.arn}/*"       # For objects of the bucket
            ]
        }]
    })
)
```

이렇게 Bucket의 ARN을 참조해서 리소스를 생성하려고 했으나, 아래와 같이 에러가 발생합니다. 

```
      File "/Users/.../workspace/test/pulumi-test/./__main__.py", line 47, in <module>
        policy=json.dumps({
      File "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/lib/python3.9/json/__init__.py", line 231, in dumps
        return _default_encoder.encode(obj)
      File "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/lib/python3.9/json/encoder.py", line 199, in encode
        chunks = self.iterencode(o, _one_shot=True)
      File "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/lib/python3.9/json/encoder.py", line 257, in iterencode
        return _iterencode(o, 0)
      File "/usr/local/Cellar/python@3.9/3.9.8/Frameworks/Python.framework/Versions/3.9/lib/python3.9/json/encoder.py", line 179, in default
        raise TypeError(f'Object of type {o.__class__.__name__} '
    TypeError: Object of type Output is not JSON serializable
    error: an unhandled error occurred: Program exited with non-zero exit code: 1
```

왜 그런지 확인해 보니, Pulumi에서 S3 버킷의 ARN이 해당 리소스의 output으로 등록되어 있지 않은 것이 원인이었습니다. ([S3 Bucket 문서](https://www.pulumi.com/registry/packages/aws/api-docs/s3/bucket/#outputs) 참조)

또한 Output 값을 그대로 사용할 수 없어서, 이를 가공하기 위해 `.apply(function)` 함수를 사용해야 하는데요. 이 부분 때문에 생각보다 테스트에 많은 시간이 소요되었습니다. 

결국 최종 코드는 다음과 같이 바뀌었습니다.

```python
def make_bucket_policy(bucket_name: str):
    return json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Action": ["s3:*"],
            "Effect": "Allow",
            "Resource": [
                f"arn:aws:s3:::{bucket_name}",             
                f"arn:aws:s3:::{bucket_name}/*"       # For objects of the bucket
            ]
        }]
    })

test_policy = aws.iam.RolePolicy("yungon-test-role-policy",
    name="yungon-iac-test-policy",
    role=test_role.id,
    policy=bucket.id.apply(make_bucket_policy)      # bucket.id -> Bucket name
)
```

# 인프라 배포 방법

## Terraform

다음 명령으로 배포합니다. init 작업으로 Provider를 가져온 뒤, apply 명령으로 인프라를 배포합니다. apply 명령으로 배포할 때, 진짜 배포할 거냐고 물어보면 `yes`를 정확하게 입력합니다. 

```shell
$ terraform init
$ terraform apply
```

삭제할 때는 다음 명령을 입력합니다. 진짜 삭제할 거냐고 물어보면 `yes`를 정확히 입력합니다.

```shell
$ terraform destroy
```

## Pulumi

인프라를 배포할 때는 `pulumi up` 명령을 사용합니다. 스택을 선택하고, 생성할 것인지 물어보면 방향키를 이용하여 yes를 선택한 뒤, 엔터를 눌러줍니다.

```shell
pulumi up     
Please choose a stack, or create a new one: yungon-iac-test
Previewing update (yungon-iac-test)

View Live: https://app.pulumi.com/...

     Type                        Name                              Plan       
 +   pulumi:pulumi:Stack         pulumi-test-yungon-iac-test       create     
 +   ├─ aws:iam:Role             yungon-iac-test-role              create     
 +   ├─ aws:s3:Bucket            yungon-iac-test-bucket            create     
 +   ├─ aws:ec2:SecurityGroup    yungon_test_security_group        create     
 +   ├─ aws:iam:RolePolicy       yungon-test-role-policy           create     
 +   ├─ aws:iam:InstanceProfile  yungon-iac-test-instance-profile  create     
 +   └─ aws:ec2:Instance         yungon-iac-test-ec2               create     
 
Resources:
    + 7 to create

Do you want to perform this update? yes
Updating (yungon-iac-test)

View Live: https://app.pulumi.com/...

     Type                        Name                              Status      
 +   pulumi:pulumi:Stack         pulumi-test-yungon-iac-test       created     
 +   ├─ aws:ec2:SecurityGroup    yungon_test_security_group        created     
 +   ├─ aws:iam:Role             yungon-iac-test-role              created     
 +   ├─ aws:s3:Bucket            yungon-iac-test-bucket            created     
 +   ├─ aws:iam:RolePolicy       yungon-test-role-policy           created     
 +   ├─ aws:iam:InstanceProfile  yungon-iac-test-instance-profile  created     
 +   └─ aws:ec2:Instance         yungon-iac-test-ec2               created     
 
Outputs:
    bucket_name: "..."
    server_ip  : "..."

Resources:
    + 7 created

Duration: 36s
```

참고로 `View Live`에 표시되는 URL로 접근하면, 배포 중인 상태를 확인할 수 있습니다. 아래와 같은 화면을 보실 수 있을 겁니다.

{{< figure src="/img/terraform-vs-pulumi-3.png" >}}

삭제할 때는 다음 명령을 이용합니다. 생성한 스택을 선택 후 엔터 키를 누르고, 방향키로 yes를 선택 후 엔터 키를 누르면 리소스를 삭제합니다.

리소스를 삭제하는 과정도 표시되는 링크를 따라 들어가면 확인할 수 있습니다.

```shell
$ pulumi destroy
Please choose a stack: yungon-iac-test
Previewing destroy (yungon-iac-test)

View Live: https://app.pulumi.com/.....

     Type                        Name                              Plan       
 -   pulumi:pulumi:Stack         pulumi-test-yungon-iac-test       delete     
 -   ├─ aws:ec2:Instance         yungon-iac-test-ec2               delete     
 -   ├─ aws:iam:InstanceProfile  yungon-iac-test-instance-profile  delete     
 -   ├─ aws:iam:Role             yungon-iac-test-role              delete     
 -   ├─ aws:s3:Bucket            yungon-iac-test-bucket            delete     
 -   └─ aws:ec2:SecurityGroup    yungon_test_security_group        delete     
 
Resources:
    - 6 to delete
```

아예 스택 자체를 없애고 싶으면, `pulumi stack rm <스택 이름>` 명령을 실행합니다.

# 상태 관리 방법

## Terraform

`terraform init` 명령을 입력하면, 작업 중인 디렉터리에 다음과 같은 파일과 디렉터리가 생성됩니다. 

* `.terraform` 디렉터리: Terraform이 캐시된 프로바이더 플러그인과 모듈을 관리하기 위해 사용합니다.
* `.terraform.lock.hcl` 파일: 특정 버전의 Provider를 사용하는 경우, dependency lock file로 생성합니다. Provider의 의존성을 관리하기 위해 사용한다고 하네요.

그리고 `terraform apply` 명령을 입력하면 다음과 같은 파일이 생성됩니다. 

* `terraform.tfstate`: 상태 데이터 파일이며, 기본적으로는 로컬 백엔드를 이용합니다.
* `terraform.tfstate.backup`: 이전 상태의 데이터를 저장합니다.

상태 파일은 원격으로 관리할 수도 있습니다. Terraform Cloud, Consul, S3 등으로 관리할 수 있다고 하네요. ([참조](https://www.terraform.io/docs/language/state/remote.html))

S3으로 관리하는 경우, state locking 및 일관성 체크를 DynamoDB를 통해서 수행한다고 합니다. ([참조](https://www.terraform.io/docs/language/settings/backends/s3.html))

### 참고자료

* [Initializing Working Directories](https://www.terraform.io/docs/cli/init/index.html)
* [Dependency Lock File](https://www.terraform.io/docs/language/dependency-lock.html)
* [State](https://www.terraform.io/docs/language/state/index.html)

## Pulumi

Pulumi로 인프라를 배포할 때, 상태 관리를 위한 파일이 로컬 환경에 남지 않는다는 것이 인상적이었습니다. 

기본적으로 Pulumi에서 상태를 관리한다고 생각하면 될 것 같네요. 다만, 원한다면 로컬 환경이나 S3 등의 서비스에서 상태 파일을 관리하도록 설정할 수 있습니다. (심지어는 자체 서버로 구축하는 방법도 있습니다)

자세한 내용은 Pulumi의 [State and Backends](https://www.pulumi.com/docs/intro/concepts/state/) 문서를 참고해 주세요. 

# 번외: 기존 리소스를 가져오기

아마 IaC 툴을 이용하신다면, 이미 클라우드 계정 내에 여러 리소스가 있을 것 같은데요. 번외로 기존에 이미 있던 리소스를 가져오는 방법을 찾아 보았습니다.

## Terraform

`import` 기능을 이용하면 될 것 같은데, 여러 제약사항이 있어 보입니다. ([참조](https://www.terraform.io/docs/cli/import/index.html)) 필요할 때 나중에 자세히 봐야겠네요.

## Pulumi

[Importing Infrastructure](https://www.pulumi.com/docs/guides/adopting/import/) 문서에 설명되어 있습니다. CLI나 코드를 이용할 수 있고, Terraform에서 마이그레이션하는 방법도 제공하고 있는 것으로 보입니다.

# 고려해야 할 점

이상으로 Terraform과 Pulumi를 사용해 보았습니다. 이번 글에서 다루었던 Terraform, Pulumi 모두 좋은 툴이라는 생각이 듭니다. 다만 팀의 상황이나, 협업 방식, 발생할 수 있는 비용 등을 고려하여 적절한 도구를 선택해야 할 것 같네요. 

그리고 코드를 보면 아시겠지만, 리소스를 생성할 때 입력하는 속성이 비슷해서 둘 중 어떤 도구를 선택하셔도 괜찮을 것 같습니다.

마지막으로 각 도구별 장단점에 대해 말씀드리며 이번 글을 마치겠습니다. 

**제가 생각한 Terraform의 장단점은 다음과 같았습니다.**

* (장점) 사용자가 많아 레퍼런스 찾기가 쉬웠다. 
* (단점) 구조가 복잡하지는 않지만, 언어 하나를 새로 배운다고 생각해야 한다. 은근히 문서가 친절하지 않다. (예를 들어, 리소스의 어떤 속성이 어떤 타입의 데이터인지...)

**그리고 제가 생각한 Pulumi의 장단점은 다음과 같았습니다.**

* (장점) 내가 아는 언어로 인프라 구축이 가능하며, 해당 언어의 문법을 개발에 그대로 이용할 수 있다.
* (단점) 모르는 게 있을 때 레퍼런스 찾기가 어렵다. 리소스 종류에 따라 다르겠지만, 리소스의 Output을 다루는 방식이 직관적이지 않다.

읽어주셔서 감사합니다.