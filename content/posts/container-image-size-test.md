---
title: "Base image에 따른 컨테이너 이미지 크기 테스트"
date: 2025-07-13T10:23:00+09:00
tags: [컨테이너, Alpine, Distroless, Docker]
draft: false
categories: [DevOps]
comments: true
---

컨테이너 이미지의 크기는 기본적으로 이미지를 저장하는 비용과 관련 있습니다. AWS의 ECR은 private repository에 저장된 이미지에 월 GB 당 0.1 USD를 부과합니다. ([출처](https://aws.amazon.com/ko/ecr/pricing/)) 한편 컨테이너 이미지를 실행하는 환경에서도 컨테이너 이미지 크기가 중요할 수 있는데요. 예를 들어 EC2와 같은 서버에서 컨테이너 이미지를 실행한다고 했을 때, 이미지의 크기가 작을수록 다운로드에 걸리는 시간이 줄어들 것이고, 이는 서버에서 컨테이너를 처음으로 실행하는 데 걸리는 시간에도 영향을 줄 수 있습니다. 그리고 ECR의 경우 동일 리전에서는 전송 비용을 받지 않지만, 리전이 달라지면 전송 비용을 받기도 하니 네트워크 비용도 고려하면 좋지 않을까 싶습니다.

그래서 이번 글에서는 컨테이너 이미지의 크기를 어떻게 하면 줄일 수 있을지에 대해 실험해 보려고 합니다. 이번 테스트는 실제 애플리케이션을 실행하는 환경과 다를 수 있으니, 업무에 적용하실 때는 충분한 테스트를 거쳐 도입해 보시기 바랍니다.

## 테스트 방법

Go의 Gin 프레임워크를 이용한 간단한 프로그램을 만들었습니다. `main.go` 파일은 다음과 같습니다. 

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.GET("/", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "Hello goni!",
			"status":  "success",
		})
	})

	r.Run(":8080")
}
```

위의 코드에서 보는 것과 같이 8080 포트가 열려 있고, `/` 엔드포인트만 열려 있는 애플리케이션입니다. 모듈 세팅 방법은 따로 다루지 않겠습니다. 필요하실 때 한 번 찾아보시기 바랍니다.

프로젝트 폴더 안에는 `go.mod`, `go.sum`, `main.go` 파일만 있는 상태로 보시면 되겠습니다. 

저는 다음과 같은 방식으로 컨테이너 이미지를 빌드해 보겠습니다. 제 개인 맥북에서 테스트 했고, 컨테이너 이미지 빌드에는 Docker를 이용합니다. 

* Debian 기반 `golang:1.23` base image 이용 (싱글 스테이지 빌드이며, 이후에는 모두 멀티 스테이지로 빌드합니다)
* Debian 기반 `golang:1.23` base image 로 빌드, 실행은 `debian:bookworm` 이미지 기반
* Debian 기반 `golang:1.23` base image 로 빌드, 실행은 `gcr.io/distroless/static-debian12` 이미지 기반
* Debian 기반 `golang:1.23` base image 로 빌드, 실행은 `scratch` 이미지 기반
* Alpine 기반 `golang:1.23-alpine` base image 로 빌드, 실행은 `alpine:3.22` 이미지 기반

그러면 결과를 한 번 확인해 보시죠.

### Case 1: `golang:1.23` 기반 싱글 스테이지 빌드

가장 단순한 형태입니다. `golang:1.23` base image는 Debian bookworm(12) 버전을 기반으로 합니다.

빌드 시 다음과 같은 공통 옵션을 주었습니다. 
* `CGO_ENABLED=0`: 빌드 시 cgo를 비활성화 합니다. [cgo](https://pkg.go.dev/cmd/cgo)는 C 언어 기반 코드를 호출할 수 있도록 하는 기능으로, 다른 라이브러리에 의존하지 않는 정적 실행 파일을 만들 때 사용하기도 합니다. ([관련 글](http://news.ycombinator.com/item?id=7677926))

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app
# 의존성 패키지를 가져옵니다.
COPY go.mod go.sum ./
RUN go mod download

# 소스를 가져온 후 빌드합니다.
COPY . .
RUN CGO_ENABLED=0 go build -o main .

EXPOSE 8080

CMD ["./main"]
```

### Case 2: `golang:1.23` + `debian:bookworm` 기반 멀티 스테이지 빌드

이미지 크기를 줄이기 위해 사용하는 방법 중 하나가 [멀티 스테이지 빌드](https://docs.docker.com/build/building/multi-stage/)입니다. 빌드 단계와 실행 단계를 나눌 수 있고, 빌드 단계에서 생성된 파일을 실행 단계에 복사하여 사용할 수 있습니다. `golang:1.23` base image는 Debian Bookworm 기반인데, 실행 환경인 `debian:bookworm`과는 기본 이미지 크기에 차이가 있습니다. (`golang:1.23` - 846MB, `debian:bookworm` - 139MB) 아래와 같은 방법으로 이미지 크기를 줄여 보겠습니다. 

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o main .

# 실행을 위한 스테이지를 추가합니다.
FROM debian:bookworm-slim AS runner
WORKDIR /app
# 이전 builder 단계에서 컴파일한 실행 파일을 복사합니다.
COPY --from=builder /app/main .

EXPOSE 8080
CMD ["./main"]
```

### Case 3: `golang:1.23` + Distroless 기반 멀티 스테이지 빌드

[Distroless](https://github.com/GoogleContainerTools/distroless)는 애플리케이션 실행을 위한 최소한의 요소로 만든 컨테이너 이미지입니다. 설명에 따르면, (`apt`, `yum`, `apk`와 같은) 패키지 매니저, 쉘, 그 외 보통의 리눅스 배포판에 들어 있는 구성 요소를 삭제했다고 합니다. 애플리케이션이 공격 받을 수 있는 범위를 줄일 수 있다는 점, 그리고 Alpine이나 Debian과 같은 다른 base image보다 용량이 작은 점이 장점입니다. (`gcr.io/distroless/static-debian12` 기준으로 이미지 사이즈가 2MB라고 하네요) 

일반 Debian 기반 이미지 외에도 Python 3, Java 17, 21 버전, Node.js 20, 22, 24 버전을 따로 지원합니다.

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o main .

# 실행 환경을 Distroless 기반으로 변경합니다.
FROM gcr.io/distroless/static-debian12 AS runner
COPY --from=builder /app/main /
EXPOSE 8080

CMD ["/main"]
```

### Case 4: `golang:1.23` + Scratch 기반 멀티 스테이지 빌드

Docker에서 제공하는 [scratch](https://hub.docker.com/_/scratch) 이미지는 base image를 만들 때, 또는 아주 작은 용량의 이미지를 만들 때 사용하는 base image입니다. Docker로 아주 작은 컨테이너 이미지를 만드는 예제를 살펴보면, 빌드는 따로 하고 scratch 이미지에 실행 파일만 복사하는 예제를 볼 수 있습니다. ([참고](https://docs.docker.com/build/building/base-images/#create-a-minimal-base-image-using-scratch)) 다만, `glibc`와 같은 라이브러리가 따로 없기 때문에, scratch 이미지에 들어가는 실행 파일은 동적으로 연결된 라이브러리가 없어도 실행할 수 있어야 합니다. 그렇기 때문에 Go 애플리케이션을 빌드할 때 `CGO_ENABLED=0` 옵션을 추가하였습니다.

만약 Python이나 Node.js, Java와 같은 언어로 만든 애플리케이션이라면 Scratch 이미지는 사용하기 어려울 것입니다. 차라리 Distroless나 Alpine 기반 이미지를 사용하시는 것이 좀 더 편할 것 같네요.

```dockerfile
FROM golang:1.23 AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o main .

# 실행 환경을 scratch로 변경합니다.
FROM scratch AS runner

COPY --from=builder /app/main /
EXPOSE 8080

CMD ["/main"]
```

### Case 5: `golang:1.23-alpine` + `alpine:3.22` 기반 멀티 스테이지 빌드

[Alpine Linux](https://www.alpinelinux.org/)는 경량 리눅스 배포판입니다. 보통 다른 리눅스 배포판에서 사용하는 glibc를 대신하여 musl이라는 라이브러리를 이용하고, 여러 리눅스 실행 파일을 하나의 실행 파일로 축소한 BusyBox를 이용해서 이미지 용량을 줄였습니다. `docker pull` 명령으로 Alpine 이미지를 받아보시면 10MB를 넘어가지 않는 것을 볼 수 있습니다. 

앞에서 말씀드렸듯 Alpine에서는 glibc를 musl로 대체했기 때문에 Debian이나 CentOS과 같은 리눅스 배포판 기반 이미지를 이용해서 빌드하면 Alpine 기반 이미지에서 실행되지 않습니다. 정적 바이너리로 빌드했더라도 혹시 모를 문제를 방지하기 위해 빌드 환경과 실행 환경 모두 Alpine 기반으로 변경해서 이미지를 빌드해 보겠습니다. 

```dockerfile
FROM golang:1.23-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0  go build -o main .

FROM alpine:3.22 AS runner

WORKDIR /app

COPY --from=builder /app/main .
EXPOSE 8080

CMD ["./main"]
```

## 결과

결과는 다음과 같습니다. 

{{< figure src="/img/container-image-size-test-1.png" >}}

* Scratch 기반: 10.8 MB
* Distroless 기반: 12.8 MB
* Alpine 기반: 19.3 MB
* Debian 기반 멀티 스테이지 빌드: 108 MB
* Golang 이미지 기반 싱글 스테이지 빌드: 1.22 GB

역시나 Scratch 이미지를 기반으로 한 이미지가 가장 크기가 작았는데요. 그런데 Alpine 기반 이미지보다 Distroless 기반 이미지가 크기가 더 작은 것이 신기했습니다. 크게 문제가 없다면 Distroless 기반으로 빌드해 보는 것도 괜찮을 것 같습니다. 

다만, 앞에서도 말씀드렸듯 본인이 사용하는 환경에 맞는지 충분하게 테스트 한 후에 적절한 환경을 찾아 보시기를 바랍니다. 

읽어주셔서 감사합니다. 
