---
title: "S3 버킷의 객체가 1,000개를 넘을 때 객체 목록 조회하기"
categories: AWS
tags: [AWS, S3, Boto3, Paginator]
draft: false
date: 2018-11-29T00:38:00+09:00
---

S3 버킷에는 여러 파일들을 저장할 수 있다. 그런데, 버킷에 저장된 파일의 목록을 보고 싶은 경우가 있을 것이다. 하지만, AWS의 Python SDK인 Boto3에서 `list_objects()`나 `list_objects_v2()` 함수를 이용하면 최대 1,000개까지의 object만 가져올 수 있다. [\[참고\]](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.Client.list_objects_v2) (근본적으로는 AWS의 API가 최대 1,000개까지의 object만 가져오도록 구현되어 있다. - [\[참고\]](https://docs.aws.amazon.com/AmazonS3/latest/API/v2-RESTBucketGET.html))

이런 문제를 해결하기 위해, 다음과 같이 Paginator를 이용해 보자.

### Paginator 이용하기

#### get_paginator()로 Paginator 가져오기

기본적으로 S3를 담당할 클라이언트를 생성한 뒤, get_paginator()로 Paginator를 가져온다. 여기서는 하나의 버킷에서 object들을 가져오기 위해 `list_objects_v2`를 이용한다. (Boto3 문서에서는 `list_objects()`보다 `list_objects_v2()`를 이용하는 것을 권장하니 이를 이용하자.)

```python
import boto3

client = boto3.client('s3')
paginator = client.get_paginator('list_objects_v2')
```

#### Paginator를 이용해서 버킷의 파일 목록 가져오기

Paginator의 `paginate()` 메소드를 이용하면, 지정한 paginator에서 응답을 페이지 단위로 저장하는 iterator를 만든다. 게시판에서 페이지 단위로 글 목록을 만드는 것과 비슷하다고 보면 된다. 

```python
response_iterator = paginator.paginate(
    Bucket='string',
    Prefix='string'
)
```

일단 자주 쓸만한 옵션에 대해 설명해 보겠다. 자세한 내용은 아래 참고자료에 링크한 문서를 확인해 보자.

* Bucket: 버킷 이름을 지정한다. **(필수 값이다.)**
* Prefix: S3 버킷에 올라간 파일은 키를 갖고 있다. `bucket 이름/111/222/333.txt`와 같이 파일이 저장되어 있는데, `111/222` 폴더 안에 있는 파일 목록을 얻으려면 `111/222`와 같이 적어주면 된다. 

이렇게 생성된 iterator를 이용해 파일 이름만 출력해 보자. 여기서 파일 이름은 앞에서 지정한 `Prefix` 옵션을 포함해서 출력된다. 

```python
for page in response_iterator:
    for content in page['Contents']:
        print(content['Key'])
```

위의 코드에서 `content['Key']` 부분에서 `'Key'` 대신 다른 정보도 가져올 수 있다.

* `LastModified`: 마지막으로 수정된 날짜/시간이다. (DateTime 형)
* `Size`: 정수(int)로 되어 있고, 바이트 단위이다.
* `StorageClass`: 저장된 파일의 스토리지 클래스이다. 따로 스토리지 클래스를 지정하지 않았다면 `STANDARD`라는 문자열을 볼 수 있다.

### 여담

Paginator는 위에서 언급한 것처럼 파일 목록을 가져오는 경우 외에도 사용할 수 있다. 또한 S3 외의 서비스에서도 Paginator를 사용할 수 있다. 예를 들어 AWS Lambda에서 사용 중인 함수의 목록을 관리할 때, `list_functions()`를 이용할 때가 있다. 하지만 `list_functions()`가 반환하는 최대 함수 수는 50개이다. 이 경우에도 Paginator를 이용해 50 개가 넘는 함수 목록을 얻을 수 있다.

### 참고자료

[Boto3 - S3.Paginator.ListObjectsV2](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html#S3.Paginator.ListObjectsV2)