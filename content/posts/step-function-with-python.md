---
title: "Python으로 Step Functions 활동 만들기"
categories: AWS
tags: [AWS, Step Functions, EC2]
draft: false
date: 2018-10-04T23:21:00+09:00
---

AWS에는 [Step Functions](https://aws.amazon.com/ko/step-functions/)라는 서비스가 있다. 여러 개의 활동(activity)를 조합해서 순서대로 또는 반복적으로 원하는 작업을 실행할 수 있도록 해 주는 서비스이다. 

일반적으로는 여러 개의 Lambda 함수를 연결해서 사용하는 경우가 많다. 하지만 Lambda 함수의 실행 시간이 5분을 넘어가면, 다른 방법을 고려해야 한다. 이럴 때 활동을 생성하고 이를 수행하는 코드를 작성하면, 오래 걸리는 작업도 Step Functions로 이용할 수 있다. 

## 활동(Activity) 만들기

1. Step Functions 콘솔의 왼쪽 메뉴에서 `활동`을 클릭한다. 
2. 화면이 바뀌면 우측의 `활동 생성`을 클릭하여 새로운 활동을 만든다.
3. 활동 이름 입력 창에서 임의의 활동 이름을 입력한다.
4. 여기서 나오는 ARN을 메모한다. 활동 이름은 알고 있어도 되지만 몰라도 상관은 없다.

## 상태 머신(State Machine) 만들기

1. 콘솔의 왼쪽 메뉴에서 `상태 머신`을 클릭한다. 
2. 우측 상단의 `상태 머신 생성`을 클릭한다.
3. 상태 머신의 이름을 입력하고, 필요한 경우 IAM 역할을 생성한다. <br/> (`나를 위한 역할 생성`에 체크, `AWS Step Functions이 Lambda 함수에 대한...` 옵션에 체크)
4. 그리고 `상태 머신 정의` 칸에는 다음과 같이 입력한다. (여기가 앞에서 메모한 ARN이 사용되는 부분이다. ARN은 뒤에서도 이용하니 메모를 버리지 말자.)

```json
{
  "StartAt": "TestActivity",
  "States": {
    "TestActivity": {
      "Type": "Task",
      "Resource": "앞에서 메모한 ARN 주소",
      "End": true
    }
  }
}
```

간단하게 설명을 하자면, 

1. 이 상태 머신은 TestActivity부터 동작을 시작한다.
2. TestActivity의 작동이 끝나면 이 상태 머신의 작동도 끝난다는 의미이다. 

## 파이썬으로 작동할 내용 코딩하기

먼저 작업을 실행할 수 있도록 해 보자. 

```python
import boto3
import json

# Step Functions에 대한 Client를 생성한다. 
client = boto3.client('stepfunctions')

# 실행 요청이 들어올 때까지 대기한다.
response = client.get_activity_task(
    activityArn='앞에서 메모한 ARN 주소',
    workerName='TestWorker'
)
```

`get_activity_task`에서 `activityArn`은 앞에서 메모한 ARN을 입력하면 된다. `workerName`은 따로 적어 주지 않아도 된다. 원래 용도는 작업이 할당된 worker를 구분하기 위한 용도이다. 

그리고 `import json`이 들어간 이유는 나중에 확인할 수 있다.

이 결과로 반환되는 값은 **dictionary**다. 반환되는 값과 의미는 다음과 같다.

* **taskToken**: 작업을 구분하기 위한 Token이다. 작업을 수행 중인지를 알려주거나(HeartBeat) 성공, 실패 여부를 전달할 때 사용한다.
* **input**: State Machine이 시작될 때나 앞에서 수행한 내용에서 입력된 값이다. JSON 포맷의 문자열로 전달된다.

각 값은 아래와 같이 가져올 수 있다.

```python
task_token = response['taskToken']
activity_input = json.loads(response['input'])
```

입력 값을 가져올 때, JSON 포맷의 문자열을 dictionary로 변환하기 위해 `json.loads()`를 이용하였다. `import json`이 앞에 있는 이유는 여기 있었다. 

그 다음으로는 가져온 값으로 처리할 내용을 자유롭게 적어준다. 사실 가져온 값과 상관 없이 수행하고 싶은 내용을 적어도 무방하다. ([전체 코드](https://gist.github.com/rubysoho07/4892af6d47e364ebb976b715738e3163))

이제 성공과 실패를 갈라보자.

```python
if activity_input['in'] == 'SUCCESS':
    # 작업 성공 시, Task가 성공했음을 알린다.
    response = client.send_task_success(
        taskToken=task_token,
        output=json.dumps({ 'result': activity_input })
    )
else:
    # 작업 실패 시, Task가 실패했음을 알린다.
    response = client.send_task_failure(
        taskToken=task_token,
        error='NO_SUCCESS_INPUT',
        cause='Input is not "SUCCESS"'
    )
```

* **작업이 성공했을 때는 `send_task_success`를 호출한다.** 앞에서 가져온 Task Token과 결과 값을 JSON 형식의 문자열로 전달해야 한다. 앞의 코드에서 `import json`을 쓴 이유가 여기에도 있다.
* **작업이 실패했을 때는 `send_task_failure`를 호출한다.** Task Token은 당연히 들어가야 하고, `error`에는 임의의 오류 코드, `cause`에는 에러가 발생한 이유를 적는다. 

## 테스트 해 보기

앞에서 만들었던 스크립트를 EC2에서 실행하면, 활동을 수행하기 위해 기다릴 것이다. (혹시 스크립트를 실행할 수 없다면, AWS CLI로 사용자 설정을 해야 한다. 또한 이 사용자는 Step Functions를 실행할 권한을 갖고 있어야 한다.)

그리고 콘솔에서 앞에서 만들었던 상태 머신으로 들어가면, 주황색의 `실행 시작`이라는 버튼을 볼 수 있다. 

실행 ID는 임의로 입력하고, 입력 내용은 다음과 같이 입력한다.

```json
{
    "in": "SUCCESS"
}
```

이 경우는 성공한다. 실행 결과는 다음과 같다. 

```json
{
  "output": {
    "result": {
      "in": "SUCCESS"
    }
  }
}
```

실패하는 경우를 만드려면, `"in"` 부분에 임의의 값을 입력한다.

```json
{
    "in": "RANDOM"
}
```

실패 시에는 실행 이벤트 내역에서 ActivityFailed 부분을 참고하면 오류 내용을 파악할 수 있다. 

```json
{
  "error": "NO_SUCCESS_INPUT",
  "cause": "Input is not \"SUCCESS\""
}
```

## 참고자료

[Boto 3 Documentation - Step Functions(SFN)](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/stepfunctions.html)