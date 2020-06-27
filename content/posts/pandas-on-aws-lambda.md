---
title: "AWS Lambda에 Pandas 올리기"
date: 2020-06-27T21:45:40+09:00
tags: [AWS, Lambda, Pandas, Serverless]
draft: false
categories: AWS
comments: true
---

팀 내에서는 Lambda 안에 파이썬 코드를 올려서 쓰고 있지만, 혹시 Pandas와 같은 라이브러리를 Lambda에 올리려면 어떻게 해야 할 지 궁금해서 정리해 본다. 

이 예제에서는 Pandas를 Lambda Layer로 만들고, Layer를 Lambda 함수에 연결해서 사용해 보려고 한다. 

## AWS Lambda(Lambda Layer)의 제한

AWS Lambda에는 Lambda Layer라고 해서 의존성이 필요한 것들을 묶어서 별도의 계층으로 만들어 쓸 수 있도록 하고 있다. 

하지만 이런 기능도 제한이 있으니 한 번 확인해 보자. 

### 참고 문서

* [AWS Lambda 제한](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/gettingstarted-limits.html)
* [AWS Lambda 계층](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/configuration-layers.html)

### 주요 제한 사항

* 하나의 함수에서 사용할 수 있는 Layer 수: 5 개
* 함수와 Layer를 모두 합하여 250 MB를 초과할 수 없음

## Pandas Lambda Layer 만들기

Lambda Layer의 내용은 `/opt` 디렉터리에 들어가게 된다. 파이썬 코드의 경우 `/opt/python`이나 `/opt/python/lib/python(버전-예:3.8)/site-packages` 디렉터리에 풀릴 것이다. 

그래서, Layer를 압축 파일로 만들 때는 `python` 디렉터리 내에 필요한 라이브러리가 들어가도록 구성하면 된다. 동일한 라이브러리를 Lambda에서 지원하는 파이썬 버전에 따라 넣고 싶다면, `python/lib/python(버전)/site-packages`에 들어가도록 만들자. 

**예제: Lambda Layer 압축 파일의 구조** 

아래 내용은 Pillow 라이브러리를 Lambda Layer로 구성하는 경우이다. 다른 언어에 대한 내용은 참고 문서의 두번째 항목을 참고하면 된다. 

```
pillow.zip
│ python/PIL
└ python/Pillow-5.3.0.dist-info
```

이러한 설명을 바탕으로 Pandas를 Lambda Layer에 넣으면 되는데, Pandas는 C 코드를 포함하고 있기 때문에 OS나 CPU 종류에 따라 필요한 라이브러리를 빌드해서 첨부해야 한다. 

이를 위해 Docker를 사용하자. 다음 내용은 Docker에서 Amazon Linux 1 버전의 이미지를 실행해서 Pandas를 설치한다. 

Lambda 런타임이 어떤 OS를 사용하는지 알고 싶다면 [AWS Lambda 런타임](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/lambda-runtimes.html) 문서를 참고한다.

```shell script
mkdir layer_pandas
cd layer_pandas
mkdir python
docker run --rm -it -v $PWD/python:/layer amazonlinux:1 bash

# 여기서부터는 Docker 이미지에서 실행하는 내용이다. 
yum install -y python36                 # Python 3.7 버전은 Amazon Linux 1 버전에서 동작하지만, yum으로는 설치할 수 없으니 유의!
cd layer
python3 -m pip install pandas -t .      # Pandas와 의존성 있는 라이브러리를 /layer에 설치한다.
rm -r *.dist-info __pycache__           # 필요 없는 파일을 지운다.
exit                                    # 컨테이너를 종료한다. 

# Layer를 zip 파일로 만든다. (명령으로 하기 귀찮으면 그냥 Finder(macOS)나 Nautilus(우분투)를 써도 된다)
zip -r layer_pandas.zip .
# ls -l 명령으로 확인해 보면, 53MB 정도의 용량을 차지하고 있음을 볼 수 있다. 
```

이제 AWS CLI로 Layer를 생성해 보자.

```shell script
aws lambda publish-layer-version --layer-name layer-pandas --compatible-runtimes "python3.6" --zip-file fileb://$PWD/layer_pandas.zip
```

하지만 다음과 같은 오류가 발생할 것이다. 참고로 69,905,067 바이트면 69.9 MB이고, Layer 자체의 용량은 54 MB 정도이지만 다른 방법을 찾아봐야 한다.

```
An error occurred (RequestEntityTooLargeException) when calling the PublishLayerVersion operation: Request must be smaller than 69905067 bytes for the PublishLayerVersion operation
```

그래서 S3에 Layer를 업로드 후 이 파일을 바탕으로 Layer를 만들어 보자. 

```shell script
# S3에 버킷 만들고 파일 업로드 하기
aws s3 mb s3://(S3 버킷 이름)
aws s3 cp layer_pandas.zip s3://(S3 버킷 이름)
# 그러면 (S3 버킷 이름)/layer_pandas.zip으로 올라가게 된다. 

# Lambda Layer 만들기
aws lambda publish-layer-version --layer-name layer-pandas --compatible-runtimes "python3.6" --content S3Bucket=(S3 버킷 이름),S3Key=layer_pandas.zip

# 이 명령의 실행 결과는 다음과 같다.
{
    "Content": {
        "Location": "https://awslambda-ap-ne-2-layers.s3.(Region 정보).amazonaws.com/snapshots/...",
        "CodeSha256": "(생략)",
        "CodeSize": 56268526
    },
    "LayerArn": "arn:aws:lambda:(Region 정보):(Account ID):layer:layer-pandas",
    "LayerVersionArn": "arn:aws:lambda:(Region 정보):(Account ID):layer:layer-pandas:1",
    "Description": "",
    "CreatedDate": "2020-06-27T13:56:18.637+0000",
    "Version": 1,
    "CompatibleRuntimes": [
        "python3.6"
    ]
}
```

이 결과에서 `LayerVersionArn` 정보를 잘 메모해 두자.

## Lambda Layer를 적용하기

먼저 Lambda 콘솔에서 다음과 같은 설정으로 Lambda 함수를 만들자. 방금 만든 Layer의 버전이 3.6 기준이므로, 함수의 런타임 버전은 Python 3.6으로 설정해야 한다. 

{{< figure src="/img/pandas-on-aws-lambda-1.png">}}

이제 함수가 생성되었으면, 하단의 Layers를 클릭 후 `[Add a layer]` 버튼을 누른다. 그리고 방금 만든 Layer 이름과 버전을 지정 후 추가 버튼을 누른다. (처음 만들었다면 버전 번호는 1일 것이다)

{{< figure src="/img/pandas-on-aws-lambda-2.png">}}

마지막으로 Pandas를 제대로 사용할 수 있는지 테스트 해 보자. 

Lambda 함수의 코드를 다음과 같이 입력해 보자. 아래 코드를 그대로 복사해도 된다. 

```python
import json

import pandas as pd

def lambda_handler(event, context):
    
    my_data = [10, 20, 30]
    
    s = pd.Series(data=my_data)
    
    print(s)
    
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

함수를 저장하고, 테스트 이벤트 구성으로 테스트용 이벤트를 만든 뒤 **테스트** 버튼을 눌러서 테스트 해 보자. 약간의 시간이 걸릴 수도 있지만 호출에 성공하고 다음과 같은 로그를 볼 수 있을 것이다. 

이 예제에서는 Pandas의 Series 타입의 데이터를 만들었기 때문에 0 10, 1 20, 2 30... 과 같이 출력된다.

```
START RequestId: 5dddcd14-8325-4771-ae70-69cb3f20c514 Version: $LATEST
0    10
1    20
2    30
dtype: int64
END RequestId: 5dddcd14-8325-4771-ae70-69cb3f20c514
REPORT RequestId: 5dddcd14-8325-4771-ae70-69cb3f20c514	Duration: 1.49 ms	Billed Duration: 100 ms	Memory Size: 128 MB	Max Memory Used: 111 MB	Init Duration: 802.38 ms	
```

모든 것들을 다 테스트 했다면, 리소스를 정리하는 것을 잊어서는 안 된다. 다음 리소스는 모두 삭제하자. 

* Lambda 함수
* S3 버킷
* Lambda Layer