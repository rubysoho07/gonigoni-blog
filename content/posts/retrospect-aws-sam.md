---
title: "AWS SAM을 사용하면서 느꼈던 것들"
categories: [AWS]
tags: [AWS, SAM, Lambda, CloudFormation]
draft: false
date: 2019-10-22T21:02:00+09:00
comments: true
---

## 왜 SAM을 사용하게 되었나?

시스템 내부에서 관리하는 Lambda 함수들이 늘어나면서, 이를 관리할 방법을 찾아야 했다. 기존에는 [Apex](https://apex.run)를 Lambda 함수 배포에 이용했지만, 뭔가 자동화된 방법을 찾고 싶었다. 

그래서 Serverless Framework, Terraform, SAM과 같은 도구들을 검토해 봤다. 그러다가 SAM을 최종적으로 선택했는데, 이유는 다음과 같다.

* Serverless Framework는 다양한 클라우드 벤더를 지원하지만, 다른 AWS 서비스를 연동하는 데 제약이 있지 않을까? 하는 막연한 생각이 들었다. (잘 찾아보니 내가 원하는 것들은 구현 가능할 것 같더라. 내가 잘못 생각했던 것 같다)
* Serverless Framework의 경우 Node.js 기반으로 동작한다. 인프라 관리를 위한 툴이나 언어는 최대한 간단하게 구성하고 싶었기 때문에 후보에서 제외했다. (Serverless Framework를 제외한 데는 이게 가장 컸다)

* Terraform 또한 다양한 클라우드 벤더를 지원한다. 그리고 사용 사례들도 찾기 쉬워서 괜찮은 툴이라 생각한다.
* 하지만 Terraform 사용 방법을 배우는 시간이 필요한데, 개발 일정을 맞추기에는 좀 부족할 것 같았다.

* SAM은 AWS에서 직접 만든 것이라, 갑자기 이걸 접는다고 하지 않는 이상은 계속 쓸 수 있을 것 같았다. 
* Lambda 함수 개발에 주로 Python을 이용하고 있는데, 또 다른 언어를 설치하지 않고 테스트나 배포에 이용하기 쉬울 것이라 생각했다. (물론 잘 쓰려면 Docker를 설치해야 한다)
* 필요한 리소스인데 SAM이 지원하지 않는 것들은 CloudFormation Template 구성 방법을 찾아 설정할 수 있었다. 자동 배포를 할 때는 CloudFormation으로 배포하면 된다는 점도 괜찮았다. 

## 개발하면서 겪었던 것들과 해결 방법

커밋 로그를 보니 5월 하순부터 SAM을 써 보기 시작한 것 같다. 그동안 겪었던 문제와, 이를 해결했던 방법을 정리해 보았다. 

### 템플릿에 한글을 사용할 수 없어요

SAM Template에 한글을 사용하면 읽지 못하는 이슈가 있다. CloudFormation도 마찬가지인 것 같다. 주석이 필요하면 어쩔 수 없이 모두 영어로 적어야 한다.

### AutoPublishAlias 옵션이 이전의 Alias를 삭제하는 이슈

참고자료: [GitHub 이슈](https://github.com/awslabs/serverless-application-model/issues/304)

SAM Template에는 `AutoPublishAlias`라는 속성으로 Lambda 함수에 붙을 alias를 지정해 줄 수 있다. 그런데 이 속성의 값을 바꾼 뒤 다시 배포하면, 이전의 alias가 삭제되는 문제점이 있다. 

위의 이슈를 잘 읽어보면 개발용 스택과 운영용 스택을 따로 구성하는 방법을 제시하고 있다. 이 이슈를 빠른 시일 내에 해결할 것 같지는 않아서, 일단은 별도의 스택으로 구성해야 한다.

### 외부 Swagger definition에 Lambda 함수의 ARN을 넣을 수 없어요

참고자료: [GitHub 이슈](https://github.com/awslabs/serverless-application-model/issues/839)

SAM에서는 Swagger 정의를 별도의 파일로 생성한 경우, 이 파일을 읽을 수 있는 방법을 지원한다. 그런데 이 경우, 내가 만든 Lambda의 ARN을 전달할 수 없다. 

이 때 해결할 수 있는 방법은 다음과 같다. 
* Lambda ARN을 외부 Swagger 정의에 넣는다. (근데 이건 Lambda 함수 이름이 바뀌면 곤란한 경우가 생길 수 있다)
* Swagger 정의를 SAM Template에 넣는다. (`AWS::Serverless::Api` 리소스의 `DefinitionBody` 속성에 JSON이나 YAML로 넣으면 됨)

### Circular Dependency between resources

참고자료: [GitHub 이슈](https://github.com/awslabs/serverless-application-model/issues/138)

S3 버킷을 만들고, 거기에 Lambda Trigger를 연동하고, 필요한 IAM Role을 생성했을 때 이런 이슈가 발생했었다. 이러면 IAM Role이 S3와 Lambda를 참조하고, Lambda가 S3와 IAM Role을 참조하는 상황이어서 그런 것 같았다. 

이 경우 아래와 같이 S3 버킷 이름을 Parameters에 빼고, Lambda, S3 버킷, IAM Role이 참조하도록 만들었더니 해결되었다. 

```yaml
# 윗부분 생략 
Parameters:
  MyBucketName:
    Type: String
    Default: yungon-test-bucket

# 중간 생략

Resources:
  # 함수 부분
  MyFunction:
    Type: AWS::Serverless::Function
    # ...
    Events:
      MyS3Event:
        Type: S3
        Properties:
          Bucket: !Ref MyBucketName
          Events: s3:ObjectCreated:*
          # ...
  
  # 버킷 생성 부분
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref MyBucketName
  
  # IAM Role 만드는 부분은 생략
```

### API Gateway와 Cognito 연동하기

참고자료: [AWS 문서 - Controlling Access to API Gateway APIs](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-controlling-access-to-apis.html)

보통은 API 리소스의 Auth 속성에 설정해 주면 된다.
```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      # 잠깐 생략
      Auth:
        DefaultAuthorizer: MyCognitoAuthorizer
        Authorizers:
          MyCognitoAuthorizer:
            # Resources 속성 안에 MyCognitoUserPool이 있다고 가정한다.
            UserPoolArn: !GetAtt MyCognitoUserPool.Arn   
```

API 리소스 안에 DefinitionBody 속성을 설정했다면, DefinitionBody 속성에도 이런 내용을 넣어야 할 것이다. (물론 Function 리소스의 Events 속성에 API를 연결했다면 없어도 무방하다)

```yaml
paths: 
  /test-path: 
    post: 
      # 중간 과정 생략
      security: 
        - MyCognitoAuthorizer: 
          - "scope" 
```

상세한 내용은 위의 참고자료를 확인하자.

### S3 트리거가 왜 동작하지 않을까?

참고자료: [관련 GitHub 이슈](https://github.com/awslabs/serverless-application-model/issues/300)

S3의 특정 조건을 기준으로 Events 속성을 지정했는데, 배포 후 살펴보니 Lambda 함수에 트리거 정보가 없었다. 

이 경우 아래와 같이 Lambda의 Permission을 Resources에 추가하면 해결 가능하더라.

```yaml
Resources:
  # 함수(TestLambdaFunction)나 버킷(TestBucket) 관련 정보가 있다고 가정합니다.
  MyFunctionPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:invokeFunction
      SourceAccount: !Sub ${AWS::AccountId}
      FunctionName: !Ref TestLambdaFunction.Alias
      Principal: s3.amazonaws.com
      SourceArn: !GetAtt TestBucket.Arn
```

## 결론?

SAM을 쓰기 시작한 지 5개월 정도 된 것 같다. Python과 Docker를 사용하는 환경이라면, 또는 팀이 이들 기술을 기반으로 한다면, Lambda 함수들을 관리할 때 SAM을 쓰는 것도 괜찮은 선택인 것 같다. (물론 AWS를 쓰는 환경이 아니라면 어쩔 수 없다) 다만 대부분의 문서들이 한글로 번역되어 있지 않다는 점이 단점이라 할 수 있을 것 같다. 하지만 CloudFormation 문서들을 한글로도 볼 수 있으니 상관 없을 수도 있겠다.
