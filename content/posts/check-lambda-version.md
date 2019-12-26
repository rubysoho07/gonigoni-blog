---
title: "AWS Lambda에서 별칭(alias)으로 함수 버전 구분하기"
categories: AWS
tags: [AWS, Lambda]
draft: false
date: 2018-10-02T23:16:00+09:00
comments: true
---

test라는 함수가 있고, dev, release라는 별칭(alias)이 존재한다고 하자. 그리고 모든 별칭은 동일한 버전을 가리킨다고 하자.

이 경우 context 객체의 invoked_function_arn은 어떻게 달라지는지 보자. 테스트를 위해, 다음과 같이 파이썬으로 함수를 작성하였다.

```python
import json

def lambda_handler(event, context): 
    print(context.invoked_function_arn)
    return "Success"
```

그리고 CloudWatch에 기록된 로그를 보자.

dev인 경우

```
START RequestId: 2dce0b80-5fb4-11e8-b5b7-41e11422a67a Version: 1
arn:aws:lambda:ap-northeast-2:256724228018:function:test:dev
END RequestId: 2dce0b80-5fb4-11e8-b5b7-41e11422a67a
REPORT RequestId: 2dce0b80-5fb4-11e8-b5b7-41e11422a67a  Duration: 1.56 ms       Billed Duration: 100 ms         Memory Size: 128 MB     Max Memory Used: 22 MB
```

release인 경우

```
START RequestId: d6b91b43-5fb3-11e8-88fd-494e3ecde1f0 Version: 1
arn:aws:lambda:ap-northeast-2:256724228018:function:test:release
END RequestId: d6b91b43-5fb3-11e8-88fd-494e3ecde1f0
REPORT RequestId: d6b91b43-5fb3-11e8-88fd-494e3ecde1f0  Duration: 0.32 ms       Billed Duration: 100 ms         Memory Size: 128 MB     Max Memory Used: 22 MB
```

`arn:aws:lambda:ap-northeast-2:256724228018:function:test:dev`와 같이 맨 끝에 별칭이 붙는다. ":"로 구분하면 될 것 같다.

참고로, function_version은 별칭에 상관 없이 1로 출력된다. context 객체 내의 다른 속성들은 아래 참고자료를 참조하자.

### 참고자료

[https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/python-context-object.html](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/python-context-object.html)