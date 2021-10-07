---
title: "CloudFormation과 SAM을 쓰면서 겪었던 일들 모음"
date: 2021-08-28T15:51:33+09:00
tags: [CloudFormation, SAM, Serverless, IaC]
draft: false
categories: [AWS]
comments: true
---

최근 인프라 구성을 CloudFormation을 이용해서 조정해 보았습니다. 그 과정에서 여러 Lambda 함수를 쓸 때는 SAM으로, 그 외의 경우는 CloudFormation을 사용했는데요.

이번 작업으로 여러 CloudFormation 스택에 흩어져 있던 리소스를 하나로 모으고, 템플릿의 관리 방식도 좀 더 관리하게 편하도록 설정할 수 있게 되었습니다.

저희 팀이 여러 IaC(Infrastructure as Code) 툴 중에 왜 SAM과 CloudFormation을 사용하는 이유는 [이 문서]({{< ref "retrospect-aws-sam.md" >}})를 참고해 주세요. 

이번 글은 CloudFormation과 SAM을 쓰면서 겪었던 일들을 정리해 보려고 합니다. 

## SAM에서 API Gateway 정의를 SAM Template에 넣기

SAM에서 제공하는 `AWS::Serverless::Api` 리소스는 AWS Gateway의 REST API를 생성해 주는 기능입니다. API Gateway에는 [OpenAPI 규격으로 작성된 API 스펙을 가지고 올 수 있는 기능](https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/api-gateway-import-api.html)이 있는데요. SAM에서도 마찬가지로 해당 기능을 지원합니다. 

* [CloudFormation의 API Gateway REST API 중 Body 부분](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-restapi.html#cfn-apigateway-restapi-body)
* [Serverless Application Model의 AWS::Serverless::Api 중 DefinitionBody 부분](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-api.html#sam-api-definitionbody)

즉, 다음과 같이 규격을 정의할 수 있습니다. 

* SAM의 AWS::Serverless::Api 리소스의 DefinitionBody / CloudFormation의 경우 AWS::ApiGateway::RestApi 리소스의 Body 속성: JSON이나 YAML 형식으로 API 정의를 생성
* SAM의 AWS::Serverless::Api 리소스의 DefinitionUri / CloudFormation의 경우 AWS::ApiGateway::RestApi 리소스의 BodyS3Location 속성: S3에 올린 OpenAPI 정의 파일로 API 정의를 생성 (JSON 또는 YAML 포맷)

기본적인 OpenAPI의 구조는 [OpenAPI Basic Structure](https://swagger.io/docs/specification/basic-structure/) 문서를 참고해 보시면 될 것 같습니다. 만약 Lambda 함수를 연결하여야 하는 경우라면, [이 예제](https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/api-as-lambda-proxy-export-swagger-with-extensions.html)를 참고해 보시면 좋을 것 같네요.

## SAM에서 API Gateway 연동 시 OpenAPI Definition에 `${AWS::Region}` 값을 얻어올 수 없는 이슈 

참고자료: [Stack Overflow](https://stackoverflow.com/questions/53582820/unable-to-parse-api-definition-because-of-a-malformed-integration-at-path-pric)

예를 들어, 다음과 같이 API 정의를 넣었다고 가정해 보겠습니다.

```yaml
x-amazon-apigateway-integration:
  httpMethod: post
  type: aws
  uri:
    !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${MyFunction.Arn}/invocations"
  responses:
    default:
      statusCode: '200'
```

위와 같은 API 정의를 넣고 배포하려고 하면, `Unable to parse API definition because of a malformed integration at path /...`와 같은 에러가 발생하는데요. 

원인은 API Gateway의 제약 사항 때문이라고 합니다. (SAM의 [GitHub 이슈](https://github.com/aws/serverless-application-model/issues/79) 참조)

그래서 함수 이름만(위 예제의 `${MyFunction.Arn}`으로 표시한 부분) `!Sub`로 대체하는 것으로 하고, 리전 정보는 하드코딩 하여 우회했습니다.

## SAM에서 Lambda Layer ARN을 파라미터 스토어에서 가져오려고 할 때 에러 발생 이슈

Lambda Layer를 이용하면 공통 코드나 라이브러리를 쉽게 관리할 수 있는데요. Lambda Layer의 ARN을 Parameter Store에 저장해 두고, 이 값을 참조하려고 하면 에러가 발생합니다. ([GitHub 이슈](https://github.com/aws/aws-sam-cli/issues/1069) 참조)

다음 방법으로 우회할 수 있습니다. 

1. 파라미터 값을 쉘의 변수로 저장: `LAYER_ARN=$(aws ssm get-parameter --name '/path/to/layer/arn' --query 'Parameter.Value' --output text)`
2. 1에서 얻어온 파라미터 값을 SAM Template의 파라미터로 지정하여 빌드: `sam build --parameter-overrides LambdaArnParam=${LAYER_ARN}`
3. 1에서 얻어온 파라미터 값을 SAM Template의 파라미터로 지정하여 배포: `sam deploy --parameter-overrides LambdaArnParam=${LAYER_ARN}`

SAM Template에 파라미터를 지정하는 방법은 CloudFormation Template의 [Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html) 섹션 내용을 참고하세요.

## SAM CLI로 파이썬 런타임을 사용하는 함수를 빌드할 때, `“Error: PythonPipBuilder:ResolveDependencies”` 오류 발생하는 경우

제가 사용하는 Python 런타임 버전은 3.8인데, Python 3.7 버전을 사용하는 Lambda 함수를 빌드해야 할 때가 있습니다. 그러다 보면 라이브러리 버전이 안 맞아서 위와 같은 에러가 발생할 때가 있는데요. 

이 때는 `sam build` 명령을 사용할 때 `--use-container` 옵션을 추가하여 빌드합니다. 다만 Docker를 설치한 뒤, 실행하고 있어야 정상적으로 동작합니다.

## SAM으로 이벤트 스케줄을 정할 때 `!If [Condition, true, false]` 조건이 동작하지 않는 문제

SAM에는 다음과 같이 특정 시간이나 일정한 주기로 Lambda 함수를 수행할 수 있도록 설정하는 기능이 있습니다.

```yaml
Resources:
  MyLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      # 중간 생략
      Events:
        MyScheduleEvent:
          Type: Schedule
          Properties:
            Schedule: rate(5 minutes)
            Name: my-schedule
            Description: Example schedule
            Enabled: True
```

그런데, `Schedule` 부분에 `!If [Condition, True, False]`와 같은 방식으로 조건을 주면 배포가 되지 않는 이슈가 있습니다. (GitHub 이슈 참조 - [#1360](https://github.com/aws/serverless-application-model/issues/1360), [#1329](https://github.com/aws/serverless-application-model/issues/1329))

이러한 경우, 수동으로 CloudFormation의 `AWS::Events::Rule` 리소스를 생성하고, Lambda 함수를 타겟으로 지정하는 방법으로 우회할 수 있습니다. ([#1329 이슈의 댓글](https://github.com/aws/serverless-application-model/issues/1329#issuecomment-768659579) 참조)

이에 덧붙여, EventBridge(CloudWatch Events)가 Lambda 함수를 실행할 수 있도록 `AWS::Lambda::Permission` 리소스를 추가로 생성해 주어야 합니다. ([#1329 이슈의 댓글](https://github.com/aws/serverless-application-model/issues/1329#issuecomment-851545027) 참조)

## CloudFormation의 Conditions

CloudFormation Template를 구성하는 요소 중 하나인 [Conditions 섹션](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html)은 다음과 같은 경우에 이용할 수 있습니다. 

* 개발/운영 스택 중 특정 스테이지의 스택에만 리소스를 생성하고 싶을 때
* 조건에 따라 값을 다르게 구성하고 싶을 때 

기본적인 구성은 다음과 같습니다. 

```yaml
Conditions:
  Logical ID:
    Intrinsic function
```

그리고 예제를 한 번 보시죠.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Parameters:
  EnvType:
    Description: Environment type.
    Default: test
    Type: String
    AllowedValues:
      - prod
      - test
    ConstraintDescription: must specify prod or test.
Conditions:
  CreateProdResources: !Equals 
    - !Ref EnvType
    - prod
```

위 템플릿으로 스택을 만들 때 `EnvType`이라는 파라미터가 `prod`면, `CreateProdResources` Condition이 true가 됩니다. 

이런 조건을 어떻게 사용할 지는 다음 예제를 소개해 드리겠습니다. 

```yaml
Resources:
  EC2Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      ImageId: ami-0ff8a91507f77f867
  MountPoint:
    Type: 'AWS::EC2::VolumeAttachment'
    Condition: CreateProdResources
    Properties:
      InstanceId: !Ref EC2Instance
      VolumeId: !Ref NewVolume
      Device: /dev/sdh
  NewVolume:
    Type: 'AWS::EC2::Volume'
    Condition: CreateProdResources
    Properties:
      Size: 100
      AvailabilityZone: !GetAtt 
        - EC2Instance
        - AvailabilityZone
```

위 예제의 경우, 각 리소스에 `Condition` 속성이 있습니다. 만약 `CreateProdResources` 속성이 true면 MountPoint와 NewVolume 리소스가 함께 생성될 것입니다. 

그리고 조건에 따라 다른 값을 넣도록 하려면, CloudFormation이 제공하는 내장 함수를 이용할 수 있습니다. [Condition Functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-conditions.html) 문서를 한 번 참고해 보세요.

예를 들어 아래 예시와 같이, 개발/운영 인프라에 따라 버킷 이름을 바꾸는 방법이 있겠습니다.

```yaml
Resources:
  MyBucket:
    Type: 'AWS::S3::Bucket'
    BucketName: !If [CreateProdResources, 'my-dev-bucket', 'my-ops-bucket']
```

## CloudFormation의 `Fn::If` 함수의 결과를 Array로 할당하려면?

YAML로 작성하는 경우, 다음과 같이 작성하면 Array로 들어갑니다. ([참고자료](https://stackoverflow.com/questions/56970457/how-to-use-fnif-with-array-values-in-cloud-formation-templates))

```yaml
Value:
  Fn::If:
    - MyCondition
      - - Value1
        - Value2
      - - Value3
        - Value4
```

## `DELETE_FAILED` 상태로 멈춰 있는 CloudFormation 스택을 삭제하려면?

여러 이유로 스택 삭제에 실패할 때가 있습니다. [이 문서](https://aws.amazon.com/ko/premiumsupport/knowledge-center/cloudformation-stack-delete-failed/)를 참고하세요.

## CloudFormation 스택 간 리소스 이동하기

CloudFormation은 템플릿에 정의한 리소스들을 스택이라는 이름으로 관리합니다. 만약 스택을 병합하고 싶거나 다른 스택으로 리소스를 이동하려면 [다음 문서](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/refactor-stacks.html)를 참고해 보세요. 

그 전에, AWS의 모든 리소스를 이동할 수 있는 건 아니라는 점을 명심하셔야 합니다. 그래서 여러 스택을 하나로 합치는 데 실패하기도 했습니다. Template 간 이동을 지원하는 리소스는 [이 문서](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-supported-resources.html)를 참고하세요.

기본적인 순서는 다음과 같습니다. 

1. 원본 스택에서 기존 리소스가 삭제되지 않도록 각 리소스에 `DeletionPolicy: Retain` 옵션 추가
2. 원본 스택 업데이트: `aws cloudformation update-stack --stack-name <STACK_NAME> --template-body file://<TEMPLATE_PATH> --capabilities <CAPABILITY_OPTIONS>`
    * `aws cloudformation deploy` 명령을 사용하는 경우, 리소스의 `DeletionPolicy` 옵션이 적용되지 않습니다. 그래서 위와 같이 `aws cloudformation update-stack` 명령을 사용합니다. 
    * `--capabilities` 옵션은 필요 시 사용하면 됩니다. 
3. 리소스를 이동할 스택에 리소스 추가: `DeletionPolicy`를 포함하여 붙여 넣습니다.
4. 원본 스택에서 리소스 삭제 후 재배포: `aws cloudformation deploy --template-file <TEMPLATE_PATH> --stack-name <STACK_NAME>`
5. 리소스를 이동할 스택에 새로운 리소스 Import: [다음 문서](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/refactor-stacks.html) 참조

## CloudFormation 템플릿을 점검할 때 `Template format error: unsupported structure` 에러가 발생하는 경우

`aws cloudformation validate-template` 명령을 수행할 때, 위와 같은 에러가 발생할 수 있습니다.

다음 [Stack Overflow](https://stackoverflow.com/questions/41992569/template-format-error-unsupported-structure-seen-in-aws-cloudformation) 질문을 참고하면, `--template-body` 속성에는 File URI(`file://...`) 값이 들어가야 함을 알 수 있습니다.

이외에도 템플릿 검증이나 형식 오류가 발생한 경우 [AWS의 문서](https://aws.amazon.com/ko/premiumsupport/knowledge-center/cloudformation-template-validation/)를 참고하세요. 