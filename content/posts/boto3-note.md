---
title: "Boto3를 쓰면서 겪었던 일들 모음"
date: 2021-06-28T22:34:04+09:00
tags: [AWS, Python, Boto3, SDK]
draft: false
categories: AWS
comments: true
---

Python을 이용해서 AWS의 여러 기능을 자동화 할 때 Boto3를 많이들 쓰실 것 같은데요. 이번 달에는 Boto3를 쓰면서 궁금했던 것, 또는 자주 사용할 만한 것들을 정리해 보려고 합니다. 

# 현재 사용 중인 AWS 계정 ID 얻기

여기서 계정 ID라고 하면, 12자리의 숫자로 구성된 계정의 ID를 의미합니다. 

```python
import boto3

account_id = boto3.client('sts').get_caller_identity().get('Account')
```

출처: [Stack Overflow](https://stackoverflow.com/questions/33332050/getting-the-current-user-account-id-in-boto3)

# 에러 다루기

출처: [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/error-handling.html)

[Boto3의 코드](https://github.com/boto/boto3/blob/develop/boto3/exceptions.py)를 열어보면, 서비스에 따라 발생할 수 있는 모든 에러를 저장하고 있지 않습니다. Boto3와 AWS CLI의 기반이 되는 [Botocore 프로젝트의 코드](https://github.com/boto/botocore/blob/develop/botocore/exceptions.py)에서도 모든 종류의 예외를 다루지 않습니다. 

그래서 예외 처리를 하려면 다음과 같은 방법을 사용해야 합니다. 

## 클라이언트 에러 다루기

Boto3의 [Available Service](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/index.html) 문서를 보시면, 서비스 이름과 Client라는 내용이 있는데요. 

Client의 각 함수에 대한 문서를 확인해 보시면, 발생할 수 있는 Exception의 종류를 확인할 수 있습니다. 

이러한 Exception들은 `botocore.exceptions.ClientError`를 기준으로 예외 처리를 수행할 수 있습니다. 아래 예제를 보시죠. 

```python
import botocore
import boto3

client = boto3.client('aws_service_name')

try:
    client.some_api_call(SomeParam='param_value')
except botocore.exceptions.ClientError as err:
    """
    {
        'Error': {
            'Code': 'SomeServiceException',
            'Message': 'Details/context around the exception or error'
        },
        'ResponseMetadata': {
            'RequestId': '1234567890ABCDEF',
            'HostId': 'host ID data will appear here as a hash',
            'HTTPStatusCode': 400,
            'HTTPHeaders': {'header metadata key/values will appear here'},
            'RetryAttempts': 0
        }
    }
    """
    if err.response['Error']['Code'] == 'SomeServiceException':
        print('SomeServiceException raised')
    else:
        raise err
```

에러의 기본적인 구조는 `"""`으로 둘러싼 부분과 같습니다. Exception 객체의 response에서 발생한 예외의 종류와 메시지를 찾을 수 있습니다. 

## 리소스 에러 다루기

S3의 버킷이나 객체(Object)와 같은 리소스를 사용할 때도 예외 처리를 할 수 있습니다. 다만 위의 방법과는 조금 다른데요. 

클라이언트의 `meta` property에서 예외의 종류를 얻을 수 있습니다. `client.meta.client.exceptions.SomeServiceException` 와 같은 구조로 구성되어 있습니다. 

예제를 한 번 확인해 보시죠.

```python
import botocore
import boto3

client = boto3.resource('s3')

try:
    client.create_bucket(BucketName='myTestBucket')

except client.meta.client.exceptions.BucketAlreadyExists as err:
    print("Bucket {} already exists!".format(err.response['Error']['BucketName']))
    raise err
```

예제에서는 S3의 [서비스 리소스](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#service-resource)를 활용합니다. 하지만 서비스 리소스의 `create_bucket()` 함수에서는 어떤 예외가 발생할 수 있는지 확인하기 어려운데요. 이 때는 Client의 `create_bucket()` 함수에서 발생할 수 있는 예외를 확인해서 처리하면 됩니다. 

# Collection 자료 구조

Boto3에는 [Collection](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/collections.html#guide-collections)이라는 자료구조가 있습니다. Collection은 리소스들의 그룹에 대한 반복 가능한 인터페이스라고 하는데요. 이를 어떤 경우에 쓸 수 있는지 확인해 보시죠. 

먼저 S3에 대한 서비스 리소스를 생성합니다. 

```python
import boto3

s3 = boto3.resource('s3')
```

S3의 서비스 리소스에서는 [buckets](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.ServiceResource.buckets) 라는 콜렉션을 지원합니다. 

아래 코드를 이용하면 모든 버킷에 대한 정보를 리스트로 받아올 수 있습니다. 리스트의 내용은 [s3.Bucket 객체](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#bucket)로 구성되어 있습니다. 

```python
bucket_iterator = s3.buckets.all()

# 반복문을 통해 버킷의 이름을 출력합니다.
for bucket in s3.buckets.all():
    print(bucket.name)
```

예를 들어 len() 함수를 쓰고 싶을 때, list 타입으로 바꿔서 쓸 수 있습니다.

```python
buckets = list(s3.buckets.all())
```

이외에도 Filtering, Chainability, limiting result 등의 기능을 지원합니다. 특정한 컬렉션은 한 번에 여러 작업을 수행할 수도 있습니다. 자세한 내용은 위에 링크한 문서를 참고해 주세요. 

# 설정 값을 찾는 방법

* 출처: [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html)

리전 정보와 같은 설정을 찾는 순서는 다음과 같다고 합니다. 

* 클라이언트를 생성할 때 config 파라미터로 들어가는 Config 객체
* 환경 변수
* 사용자 홈 디렉터리의 `.aws/config` 파일

어떤 설정이 있는지는 출처 문서를 참고하세요.

# 자격 증명을 찾는 방법

* 출처: [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/credentials.html)

Access Key와 같은 자격 증명을 찾는 순서는 다음과 같습니다. 

* `boto3.client()` 함수의 파라미터로 전달
* Session Object를 생성할 때 파라미터로 전달
* 환경 변수
* 공유된 자격 증명 파일 (홈 디렉터리의 `.aws/credentials` 파일)
* AWS Config 파일 (홈 디렉터리의 `.aws/config` 파일)
* Assume Role 제공자
* Boto2 설정 파일
* IAM Role이 설정된 EC2 인스턴스 내 인스턴스 메타데이터 서비스

그 중에 몇 가지 방법만 정리해 보겠습니다. 

## 클라이언트 생성 시 파라미터로 전달

먼저 클라이언트를 만들 때 파라미터로 전달하는 방법입니다. 

```python
import boto3

client = boto3.client(
    's3',
    aws_access_key_id=ACCESS_KEY,
    aws_secret_access_key=SECRET_KEY,
    aws_session_token=SESSION_TOKEN
)
```

## 세션 객체를 만들 때 파라미터로 전달

두번째로는 세션 객체를 만들 때 파라미터로 전달하는 방법입니다. 

```python
import boto3 

session = boto3.Session(
    aws_access_key_id=ACCESS_KEY,
    aws_secret_access_key=SECRET_KEY,
    aws_session_token=SESSION_TOKEN
)

# 위에서 만든 세션에서 client()나 resource()를 호출하면 됩니다.
sqs = session.client('sqs')
s3 = session.resource('s3')
```