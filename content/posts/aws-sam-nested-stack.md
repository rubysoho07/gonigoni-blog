---
title: "AWS SAM에서 중첩된 스택 배포 시 유의해야 할 것들"
date: 2020-06-29T20:58:17+09:00
tags: [AWS, SAM, Serverless, Lambda, CloudFormation]
draft: false
categories: [AWS]
comments: true
---

팀에서 AWS SAM을 적극적으로 사용하고 있는데, SAM을 쓰면서 느낀 점들을 예전에 [글로 남긴 적이 있었다]({{< ref "retrospect-aws-sam.md" >}}). 그런데 SAM은 CloudFormation 스택으로 리소스를 생성하다 보니, CloudFormation의 제약 사항을 그대로 가지고 있다. 예를 들어 [하나의 CloudFormation 템플릿에서 선언할 수 있는 리소스 수는 200개를 넘지 않아야 한다](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)는 것이 대표적일 것이다. 이러한 문제를 겪으면서, 많은 리소스로 구성되어 있는 애플리케이션을 여러 스택으로 나누는 작업을 해야 했다. 이 글에서는 하나의 서버리스 애플리케이션을 여러 스택으로 나누는 문제를 해결하면서 겪었던 일들을 기록해 보려고 한다.

## CloudFormation의 Output 참조 vs. Export

처음에 CloudFormation의 Nested Stack을 보다가 이해를 못 했던 것이 Output 참조와 Export 부분이었다. 정리해 보면 다음과 같다. 

### Output 참조

여러 중첩 스택(nested stack)을 사용하면서, 중첩 스택의 값을 참조하고 싶을 때 사용한다. 

먼저 CloudFormation의 Outputs 영역에 스택 밖에서 사용할 수 있는 값을 지정한다. 

```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: '파일 경로'
      Role: 'IAM Role ARN'
      # ...

Outputs:
  FunctionArn:
    Value: !Ref MyFunction.Alias
    Description: 설명을 아무거나 적는다.
```

이 값을 참조하는 스택에서는 먼저 참조할 스택을 가져오고, Outputs 값에 있는 내용을 가져오려면 다음과 같이 참조하면 된다. 

```yaml
Resources:
  NestedApplication:
    Type: AWS::Serverless::Application
      Properties:
        Location: nested/template.yaml
  
  TestStateMachine:
    Type: AWS::Serverless::StateMachine
    Properties:
      Role: 'IAM Role ARN'
      DefinitionUri: statemachine/definition.json
      DefinitionSubstitutions:
        NestedMyFunction: !GetAtt NestedApplication.Outputs.FunctionArn
```

### Export

Export는 내가 만든 스택에서 내보낸 값을 아예 다른 스택에서도 사용할 수 있도록 하는 개념이다. 예를 들어 네트워크 설정을 하나의 스택으로 구성하고, VPC나 서브넷 설정을 export 했다고 가정하자. 동일한 계정 및 리전에 있는 다른 스택이 네트워크 설정을 갖고 있는 스택에서 내보낸 값들을 가져다가 쓸 수 있는 것이다. 

값을 내보낼 때는 다음과 같이 한다. 

```yaml
Outputs:
  FunctionArn:
    Value: !Ref MyFunction.Alias
    Description: 설명을 아무거나 적는다.
    Export:
      Name: SharedValueToImport     # 이 이름을 기준으로 다른 스택에서 가져올 수 있다.
```

Export 된 값을 가져다 쓰려면 다음과 같이 하면 된다. 

```yaml
Fn::ImportValue: SharedValueToImport

# 또는 다음과 같이 단축해서 쓸 수도 있다. 
!ImportValue: SharedValueToImport
```

당연한 말이지만, 다른 스택에서 내보낸 값이 등록되어 있지 않은 상태에서 ImportValue를 하면 스택 배포에 실패한다. 

## SAM에서 Nested Application 설정하기 vs. CloudFormation에서 Nested Stack 설정하기

### CloudFormation에서 Nested Stack 설정하기

다음과 같이 설정할 수 있다. ([AWS 문서 참조](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/aws-properties-stack.html))

```yaml
Type: AWS::CloudFormation::Stack
Properties: 
  NotificationARNs: 
    - String
  Parameters: 
    Key : Value
  Tags: 
    - Tag
  TemplateURL: String
  TimeoutInMinutes: Integer
```

참고로 TemplateURL은 S3 버킷에 있는 템플릿을 가리켜야 한다. 템플릿을 수정할 때마다 올리려면 귀찮으니 `aws cloudformation package` 명령 실행 후 생성된 파일을 이용해 스택을 배포하면 번거로운 일을 줄일 수 있을 것이다.

### SAM에서 Nested Application 설정하기

SAM은 Application이라는 단위로 서버리스 애플리케이션을 관리한다. 이 문서를 참조하여 생성해 보자. 

```yaml
Type: AWS::Serverless::Application
Properties:
  Location: String | ApplicationLocationObject
  NotificationARNs: List
  Parameters: Map
  Tags: Map
  TimeoutInMinutes: Integer
```

여기서 Location 속성의 값은 CloudFormation/SAM 템플릿을 포함하고 있는 S3 내의 템플릿 URL, 로컬 파일 경로 등을 사용할 수 있다.

중첩된 SAM 애플리케이션의 배포 방법은 아래 항목에서 다룰 예정이다.

## CloudFormation의 AWS::StepFunctions::StateMachine vs. SAM의 AWS::Serverless::StateMachine

올해 5월에 [SAM에서 자체적으로 Step Functions를 지원](https://aws.amazon.com/ko/about-aws/whats-new/2020/05/aws-sam-adds-support-for-aws-step-functions/)하게 되었다. 이전과 같으면 `AWS::StepFunctions::StateMachine` 리소스를 가져와야 했지만, 이제는 `AWS::Serverless::StateMachine`이라는 리소스를 가져와서 작업하면 된다. 다만 이 기능은 SAM CLI 기준으로 0.52.0 버전부터 지원하니 이 점은 유의해야 한다. 이 변화로 Step Functions의 State Machine 정의를 SAM Template 밖에 선언할 수 있게 되어서 좀 더 좋아졌다고 생각한다. 

[AWS SAM의 문서](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-statemachine.html)를 참고해서 만들어 보면 다음과 같이 만들 수 있을 것이다. 

```yaml
Type: AWS::Serverless::StateMachine
Properties:
  Definition: Map
  DefinitionSubstitutions: Map
  DefinitionUri: String | S3Location
  Events: EventSource
  Logging: LoggingConfiguration
  Name: String
  Policies: String | List | Map
  Role: String
  Tags: Map
  Type: String
```

* State Machine의 정의가 단순하다면, 그냥 `Definition` 속성에다 JSON 형태로 넣으면 된다. 
* 로컬에 있는 State Machine의 정의를 참조하려면 `DefinitionUri` 속성에 경로를 적어 넣으면 된다.
* 상황에 따라 바뀔 수 있는 것들은 `DefinitionSubstitutions`에 넣으면 된다. (예를 들어 State Machine 정의에 `${속성 이름}` 값이 있으면, `DefinitionSubstitutions` 속성 아래에 `속성 이름: 들어가야 하는 값`과 같이 넣으면 된다.)
* `Role`이나 `Policies` 속성 중 하나는 필수로 들어가야 한다.

## SAM에서 중첩된 애플리케이션 빌드/패키징

SAM CLI로 중첩된 애플리케이션을 한 번에 배포하고 싶을 때 recursive 하게 빌드하지 않는다. ([GitHub 이슈 참조](https://github.com/awslabs/aws-sam-cli/issues/1213)) 많은 사람들이 이러한 기능을 원하고 있지만 아직까지는 소식이 없으니, 다음과 같이 해 보자. 

먼저 다음과 같은 디렉터리 구조를 갖고 있다고 가정하자. 

```
project/
⎿ functions/
  ⎿ function1/
  ⎿ function2/
⎿ template.yaml
⎿ template.nested.yaml
```

각각의 템플릿은 다음과 같이 구성되어 있다고 가정하자.

* template.yaml: `functions/functions1` 함수와 `template.nested.yaml` 파일에 선언된 애플리케이션을 참조한다.
* template.nested.yaml: `functions/functions2` 함수를 가지고 있다.

먼저 nested application을 빌드한다. 

```shell script
sam build --template template.nested.yaml --build-dir target_dir/
```

그러면 프로젝트 루트 디렉터리 밑에 target_dir 디렉터리가 생성될 것이고, 그 안에 template.yaml 파일이 생성될 것이다. 프로젝트 루트 폴더에 있는 template.yaml 파일에서 template.nested.yaml 파일을 참조하는 부분을 다음과 같이 바꾸어야 한다. 

```yaml
DashboardLambdaStack:
  Type: AWS::Serverless::Application
    Properties:
      Location: ./target_dir/template.yaml
```

그리고 루트 애플리케이션을 빌드한다.

```shell script
sam build --template template.yaml
```

그러면 기본적으로 `.aws-sam` 디렉터리가 만들어지고 이 안에 template.yaml 파일이 생성될 것이다. 

이제 `aws sam package` 명령으로 함수 내용과 template 모두를 업로드 해 보자. (`your-s3-bucket-name`은 본인이 사용할 S3 버킷 이름으로 바꾼다)

```shell script
sam package --output-template-file packaged.yaml --s3-bucket your-s3-bucket-name
```

그러면 프로젝트의 루트 디렉터리에는 packaged.yaml이 생성될 것이고, 함수 내용과 중첩된 스택 템플릿이 S3에 업로드 될 것이다. 

이제 이를 바탕으로 deploy를 수행하면 된다. (`your-stack-name`은 본인이 원하는 대로 바꾼다)

```shell script
sam deploy --template-file packaged.yaml --stack-name your-stack-name --capabilities CAPABILTIY_IAM
```

참고로 InsufficientCapabilities 오류가 발생하면 `--capabilities` 옵션에 CAPABILITY_NAMED_IAM을 추가해야 할 수도 있다. 

## 참고자료

* [AWS CloudFormation 한도](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)
* [AWS::CloudFormation::Stack](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/aws-properties-stack.html)
* [AWS CloudFormation 템플릿 코드 조각](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/quickref-cloudformation.html)
* [(AWS SAM Documentation) Using Nested Applications](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-template-nested-applications.html)