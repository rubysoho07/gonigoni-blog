---
title: "Amazon Q로 간단한 API 만들어 보기 (with Go)"
date: 2024-06-09T12:40:00+09:00
tags: [Amazon Q, AI, AWS]
draft: false
categories: [AWS]
comments: true
---

## 배경

인프라를 관리하다 보면, 수동으로 하는 일을 자동화 할 수 있는 방법을 고민할 것입니다. 그래서 인프라 구성을 코드로 관리하기도 하고, 간단한 스크립트나 프로그램을 만들기도 합니다. 그 중에서도 "스크립트로 제공하는 것들을 API로 만들어 보면 어떨까?" 라는 생각을 했는데요. DevOps나 SRE 분야에서 사용하는 도구들이 Go(Golang)으로 만든 경우가 많아서 Go를 선택하였습니다. 찾아 보니 Go를 이용한 웹 프레임워크로는 Gin과 같은 것들이 있더라구요. 이를 활용해서 간단한 API를 만들어 보려고 합니다. (에디터로는 Visual Studio Code를 이용합니다)

이 API의 요구사항을 정리해 보면 다음과 같습니다. 

* Golang으로 구성 (프레임워크로는 Gin을 이용)
* AWS API에 접근하는 기능이 필요함
* MySQL에서 CRUD 쿼리를 실행할 수 있어야 함
* Redis에 접속하여 여러 작업을 실행할 수 있어야 함

제가 커리어 초기에는 C로 개발을 했었고, 간단한 작업들은 Python으로 개발하는 경우가 많았는데요. 예전에 Go 강의를 들어본 적은 있긴 한데, Go에는 익숙하지 않아서 AI의 도움을 받는 것이 좋겠다고 판단했습니다. 어떤 것들을 쓰면 좋을지 고민하여, 아래와 같이 세 개의 툴을 후보로 선정했습니다. 

* [ChatGPT](https://chatgpt.com)
* [GitHub Copilot](https://github.com/features/copilot)
* [Amazon Q Developer](https://aws.amazon.com/ko/q/developer/)

ChatGPT는 한국어로 질문/답변이 가능하다는 점에서는 좋다고 생각했습니다. 다만 웹 브라우저를 왔다갔다 해야 한다는 점이 불편했습니다. (물론 ChatGPT를 지원하는 VSCode Extension이 있지만, OpenAI에서 공식적으로 지원하는 건 아닙니다) 

GitHub Copilot은 유료로 이용해야 해서 일단 배제했습니다. 대충 찾아보니 코드 자동완성은 되는데, 한국어로 질문하는 건 지원하지 않는 듯 했습니다.

Amazon Q Developer는 GitHub Copilot과 같이 영어로 질문해야 하는 단점은 있습니다. 다만 [Free tier를 제공하고](https://aws.amazon.com/ko/q/developer/pricing/), 코드 자동 완성도 가능하다는 점을 장점으로 보았습니다.

## Visual Studio Code Extension 이용하기

Amazon Q는 JetBrains의 IDE, Visual Studio Code를 지원합니다. 이번 글에서는 Visual Studio Code 기준으로 설명합니다. 

우선 Extensions에서 Amazon Q를 검색하여 설치합니다. (빨간색으로 표시한 1️⃣ → 2️⃣ → 3️⃣ 순으로 클릭합니다)

{{< figure src="/img/create-api-with-amazon-q-1.png" >}}

Extension을 설치하면 왼쪽에 Amazon Q 아이콘이 뜹니다. (빨간색으로 표시) 이 아이콘을 누르면 로그인을 하라는 메시지를 볼 수 있습니다. Free tier의 경우는 AWS 계정이 없어도 가입 및 로그인하여 사용할 수 있습니다. 'Use For Free'를 클릭 후, Continue를 클릭합니다.

{{< figure src="/img/create-api-with-amazon-q-2.png" >}}

AWS Builder ID를 만들거나, 이미 있는 AWS Builder ID로 로그인하면 아래와 같은 화면이 뜹니다. Allow Access를 클릭합니다. 

{{< figure src="/img/create-api-with-amazon-q-3.png" >}}

이제 Visual Studio Code에서 Amazon Q를 사용할 수 있게 되었습니다.

## Amazon Q에게 물어 보기 

Go 프로젝트를 만들기 위해 Amazon Q에 질문해 보겠습니다. VSCode 왼쪽의 Amazon Q 아이콘을 누르고, 궁금한 것을 입력하면 질문을 할 수 있습니다.

{{< figure src="/img/create-api-with-amazon-q-4.png" >}}

일단 안내하는 대로 해 보겠습니다. 프로젝트 초기화까지는 그럭저럭 맞는 설명을 하는 듯 합니다. 다만 `go mod init` 명령 뒷 부분은 `github.com/~` 말고 임의의 값을 넣어도 됩니다. 저는 `go mod init gin-test` 명령을 실행했습니다.

다만 실제로 테스트를 해 보려고 하니 코드가 좀 이상하게 표시되네요. 복사/붙여넣기를 해 보니 정상적으로 동작할 것 같지는 않습니다. ChatGPT에게 물어보니 아래와 같은 코드를 추천해서 이걸로 사용하겠습니다.

```golang
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

`go run main.go`를 실행하면 정상적으로 실행되는 것을 볼 수 있습니다.

그러면 Gin을 프로젝트에 추가하려면 어떻게 해야 하는지 물어보겠습니다. 문법에 안 맞게 대충 질문했지만, 잘 대답해 주네요. 

{{< figure src="/img/create-api-with-amazon-q-5.png" >}}

이번에는 설명대로 진행하니 잘 되는 것 같습니다. `main.go` 파일은 다음과 같이 작성해 주면 됩니다. 

```golang
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	// Add your routes here
	r.GET("/hello", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, World!",
		})
	})

	r.Run() // listen and serve on 0.0.0.0:8080
}
```

웹 브라우저에서 `localhost:8080/hello`로 접속하면 JSON 형식의 데이터를 반환합니다. 

## 코드 자동 완성 활용하기 

AWS API를 이용하여 S3 버킷 목록을 반환하는 코드를 작성해 보겠습니다. 우선 AWS의 Go SDK를 프로젝트에 추가하는 방법을 물어보겠습니다. 

{{< figure src="/img/create-api-with-amazon-q-6.png" >}}

일단 `go get` 명령으로 필요한 패키지를 설치해 줍니다. 실제로는 config 패키지 및 사용하고자 하는 AWS 서비스에 필요한 패키지들도 설치해 주어야 합니다. ([AWS 문서](https://aws.github.io/aws-sdk-go-v2/docs/getting-started/) 참조)

Amazon Q가 답변한 내용을 바탕으로 필요한 패키지를 import 하고, Config 설정도 가져왔습니다.

이제 본격적으로 API를 만들어 보겠습니다. 주석으로 작성하고 싶은 코드에 대한 설명을 붙이면, 아래와 같이 자동으로 코드를 생성해 줍니다. 처음에는 몇 줄만 생성해 주는데, Tab 키를 누르고 Enter 키를 누르다 보면 나머지 부분도 자동으로 생성해 줍니다. 

{{< figure src="/img/create-api-with-amazon-q-7.png" >}}

`go run main.go` 명령을 실행하고, `localhost:8080/s3`로 들어가서 확인해 봅니다. AWS 계정에 있는 버킷 목록을 JSON으로 반환해서 보여주고 있습니다. (버킷 이름은 가렸습니다)

{{< figure src="/img/create-api-with-amazon-q-8.png" >}}

이와 같이 AWS Go SDK를 Gin 프레임워크에서 이용하는 방법을 알 수 있었습니다. 저는 이런 방법을 이용해서 Redis, MySQL도 어떻게 사용할 수 있을지 Amazon Q로 물어보고 코드를 작성했습니다. 다만 실제로 코드를 작성하다 보면 답변한 내용을 그대로 쓰기는 어려웠습니다. 에러가 발생하면 왜 에러가 발생하는지 찾고, 의도대로 동작하지 않으면 왜 그런지 찾고 해결하는 과정이 필요했습니다.

여기까지 Amazon Q를 활용한 프로젝트는 [gin-test](https://github.com/rubysoho07/gin-test) GitHub 저장소에 올렸습니다. (초기 작업은 Amazon Q를 사용하지 않았고, 중간에 시행착오들이 있어서 이 글의 설명과 다를 수도 있습니다)

## 후기

직접 써 보고 느낀 점은 다음과 같습니다. 

* AI 기반 개발 도구들은 개발자를 보조하는 용도로 사용하는 게 좋습니다. 특히 반복적인 작업을 수행할 때, 코드 자동 완성이 효율성을 높일 수 있었습니다.
* 개발자는 AI가 작성한 코드를 직접 검증할 수 있어야 합니다. 자동 완성된 코드를 실행했을 때, 제가 의도한 동작이 아닌 경우가 종종 있었는데요. 자동으로 완성되는 코드는 여러 자료를 찾아가면서 크로스 체크해야 했습니다. "코딩을 몰라도 프로그램을 만들 수 있다"는 건 아직 조금 먼 일이 아닌가 싶습니다.
* 효율성은 확실히 올라가는 느낌이었습니다. 처음에 어떻게 시작해야 할 지 모를 때 빠르게 실행할 수 있는 장점이 있었습니다. 

특히 Amazon Q는 여러 언어도 지원하는데([참고자료](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/q-language-ide-support.html)), Terraform 코드 생성도 지원해서 좋았습니다. Terraform으로 비슷하지만 조금 다른 AWS 리소스를 생성할 때 좋았습니다.

이번에 Amazon Q를 사용해 보면서 기술의 발전이 빠르다는 생각이 들었습니다. 이런 도구들의 도움을 받다 보면 업무의 효율성은 올라갈 것 같습니다. 다만 앞으로 인간이 소프트웨어 개발에 어떤 역할을 할 수 있을지 생각해 보는 기회가 되었네요. 

읽어주셔서 감사합니다. 