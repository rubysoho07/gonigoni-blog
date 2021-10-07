---
title: "AWS Session Manager & Run Command로 외부에서 쉘 스크립트 실행하기"
date: 2021-09-30T14:38:05+09:00
tags: [AWS, Systems Manager, Session Manager]
draft: false
categories: [AWS]
comments: true
---

AWS의 Systems Manager에는 [Run Command](https://aws.amazon.com/ko/blogs/aws/new-ec2-run-command-remote-instance-management-at-scale/)와 [Session Manager](https://aws.amazon.com/ko/blogs/korea/new-session-manager/)라는 기능이 있습니다.

SSH를 이용하여 EC2 인스턴스에 접근할 때는 SSH 키 페어가 필요하고, SSH 접속을 위한 보안 그룹을 열어야 하는데요. 위 기능을 이용하면 이러한 과정을 생략하고 원격에서 EC2 인스턴스에 접근하여 명령을 실행할 수 있습니다.

먼저, Session Manager로 웹 브라우저에서 쉘을 실행하는 방법부터 설명드리겠습니다.

# Session Manager로 웹 브라우저에서 쉘 실행하기

Session Manager로 웹 브라우저에서 쉘을 실행하려면, EC2 인스턴스가 Systems Manager와 관련된 작업을 실행할 수 있는 권한을 부여해야 합니다. 

먼저 EC2 서비스에 대한 신뢰 정책을 생성합니다. 아래 내용을 복사하여 임의의 파일 이름으로 저장합니다. (저는 `assume-role.json`이라는 파일 이름으로 저장했습니다)

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
}
```

저장한 파일 이름을 이용해서 다음 명령으로 IAM Role을 생성합시다. 그리고 SSM을 EC2에서 사용할 수 있도록 하기 위해 AmazonEC2RoleforSSM Policy를 연결해 줍니다.

```shell
aws iam create-role --role-name AWSSSMTestRole --assume-role-policy-document file://./assume-role.json
aws iam attach-role-policy --role-name AWSSSMTestRole \
    --policy-arn "arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM"
```

CLI로 작업하는 경우, 인스턴스 프로파일에 생성한 Role을 추가해야 합니다. (콘솔로 작업하는 경우, IAM Role 생성 시 자동으로 수행됩니다.)

```shell
aws iam create-instance-profile --instance-profile-name AWS_SSM_Instance_Profile
aws iam add-role-to-instance-profile --instance-profile-name AWS_SSM_Instance_Profile --role-name AWSSSMTestRole
```

콘솔로 작업하시려면 [링크](https://aws.amazon.com/ko/getting-started/hands-on/remotely-run-commands-ec2-instance-systems-manager/)한 문서의 1단계를 실행해 주세요.

이제 EC2 인스턴스를 생성합니다. 이미지 ID는 x64 버전의 Amazon Linux 2 이미지 ID입니다. 키 이름은 생성하신 키 페어 이름을 적으면 됩니다. 없으시다면 [이 문서](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/ec2-key-pairs.html)를 참고해서 만들어 주세요. 

```shell
aws ec2 run-instances --iam-instance-profile Name=AWS_SSM_Instance_Profile \ 
     --instance-type t3.nano \ 
     --key-name <your-key-pair-name> \ 
     --image-id ami-08c64544f5cfcddd0
```

조금 기다리면 EC2 인스턴스가 생성되어 있을 겁니다. 그리고 Systems Manager 콘솔로 이동합니다. 왼쪽 아래를 보면 Session Manager라는 메뉴가 있는데요. 이 메뉴를 클릭합니다.

{{< figure src="/img/aws-session-manager-run-command-1.png" >}}

Session Manager 메뉴를 클릭하면 우측에 Start Session 버튼이 있습니다. (위의 스크린샷 참조) 이 버튼을 클릭합니다.

방금 생성한 인스턴스를 선택하고, 아래의 Start Session 버튼을 클릭합니다.

{{< figure src="/img/aws-session-manager-run-command-2.png" >}}

그러면 아래 스크린샷과 같이 쉘 화면이 실행된 것을 볼 수 있습니다. 종료하려면 우측 상단의 **Terminate** 버튼을 클릭하거나, `exit` 명령을 치면 됩니다. 

{{< figure src="/img/aws-session-manager-run-command-3.png" >}}

참고로, Session Manager를 사용한 경우 실행 중인 사용자는 `ssm-user`, 로케일 설정은 `C.UTF-8`으로 되어 있는 것을 확인할 수 있습니다. (위의 스크린샷 참조)

# Run Command로 원격으로 쉘 스크립트 실행하기

아직 EC2 인스턴스를 끝내지 않으셨다면, 인스턴스 ID(`i-...`으로 시작하는 값)를 메모해 주시기 바랍니다. 

이번에는 원격으로 쉘 스크립트를 실행하는 작업을 해 보겠습니다. 

Boto3가 설치되어 있다면, 아래와 같이 스크립트를 작성해 봅니다. 

```python
import boto3

ssm = boto3.client('ssm')

response = ssm.send_command(
    InstanceIds=['i-your_instance_id'],
    DocumentName='AWS-RunShellScript',
    Parameters={
        "commands": [
            "echo 'Run command as ...'",
            "whoami",
            "echo 'Current locale ...'",
            "locale"
        ]
    },
    OutputS3BucketName='your-bucket-name'
)
```

Run Command의 결과는 S3나 CloudWatch Logs로 내보낼 수 있습니다. 물론 필수가 아니기 때문에 생략할 수도 있는데요. 자세한 것은 [Boto3 문서](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ssm.html#SSM.Client.send_command)를 참고해 주세요.

* S3로 output을 내보내는 경우: `send_command()` 호출 시 OutputS3BucketName, OutputS3KeyPrefix 파라미터를 사용
* CloudWatch Logs로 output을 내보내는 경우: `send_command()` 호출 시 CloudWatchOutputConfig 파라미터를 사용

이 스크립트를 실행하면, S3에 stdout 내용이 올라와 있는 것을 보실 수 있습니다. 내용은 다음과 같습니다. 

```
Run command as ...
root
Current locale ...
LANG=
LC_CTYPE="POSIX"
LC_NUMERIC="POSIX"
LC_TIME="POSIX"
LC_COLLATE="POSIX"
LC_MONETARY="POSIX"
LC_MESSAGES="POSIX"
LC_PAPER="POSIX"
LC_NAME="POSIX"
LC_ADDRESS="POSIX"
LC_TELEPHONE="POSIX"
LC_MEASUREMENT="POSIX"
LC_IDENTIFICATION="POSIX"
LC_ALL=
```

즉, `send_command()`를 사용하면 `root` 계정으로 스크립트의 내용을 실행하고, 로케일 설정은 기본값으로 주어집니다. 이 경우, 실행할 명령에 한글이 섞여 있을 때 에러가 발생할 수 있으니 유의하시기 바랍니다. 그리고 [SSM Agent는 root 계정으로 실행하기 때문에](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-restrict-root-level-commands.html), 해당 작업에 대한 권한 부여는 최대한 제한하는 게 좋겠네요.

# 리소스 정리

테스트가 끝나셨다면 인스턴스를 종료해 주셔야 과금되지 않습니다. 

```shell
aws ec2 terminate-instances --instance-ids i-your_instance_id
```

추가로, 과금되는 건 아니지만 IAM Role도 삭제해 보겠습니다. 먼저 인스턴스 프로파일에서 IAM Role 연결을 끊고, 인스턴스 프로파일을 삭제합니다.

```shell
aws iam remove-role-from-instance-profile --instance-profile-name AWS_SSM_Instance_Profile --role-name AWSSSMTestRole
aws iam delete-instance-profile --instance-profile-name AWS_SSM_Instance_Profile
```

마무리로 IAM Role과 Policy 간 연결을 해제하고 Role을 삭제합니다.

```shell
aws iam detach-role-policy --role-name AWSSSMTestRole \ 
    --policy-arn "arn:aws:iam::aws:policy/service-role/AmazonEC2RoleforSSM"
aws iam delete-role --role-name AWSSSMTestRole
```

# 참고자료

* [New EC2 Run Command – Remote Instance Management at Scale](https://aws.amazon.com/ko/blogs/aws/new-ec2-run-command-remote-instance-management-at-scale/) (영어)
* [AWS Systems Manager Session Manager, EC2 인스턴스 쉘 접근을 위한 신규 기능](https://aws.amazon.com/ko/blogs/korea/new-session-manager/)
* [EC2 인스턴스에서 원격으로 명령 실행](https://aws.amazon.com/ko/getting-started/hands-on/remotely-run-commands-ec2-instance-systems-manager/)
