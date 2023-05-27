---
title: "EKS에 올린 Spring Boot 애플리케이션에 IAM Role 연결하기"
date: 2023-05-27T22:10:18+09:00
tags: [Spring Boot, Java, EKS, IAM, Kubernetes]
draft: false
categories: [DevOps]
comments: true
---

최근 EKS에 Spring Boot 애플리케이션을 올렸는데, ServiceAccount에 IAM Role을 연결했음에도 AWS API를 호출하지 못하는 문제가 있었습니다. 이 문제를 해결하기 위해 테스트 한 과정을 남겨보려고 합니다.

우선, [AWS의 Java 버전 SDK는 부여된 권한을 어떻게 찾는지](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html) 알아보겠습니다. 따로 설정을 하지 않았다면, 아래와 같은 순서로 찾습니다.

* Java의 시스템 properties
* 환경 변수
* **AWS의 STS(Security Token Service)로 부터 얻은 Web Identity Token** 
* 공유된 `credentials`, `config` 파일: 별도의 설정이 없으면 `default` 프로필을 이용
* ECS 컨테이너 credentials
* EC2 인스턴스에 부여된 IAM Role

그리고 EKS의 IAM Roles for Service Accounts(이후 IRSA로 표기) 기능은 STS(Security Token Service)의 `AssumeRoleWithWebIdentity` API 호출을 통해 권한을 얻습니다. ([참고](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)) 그렇기 때문에 STS을 사용할 수 있어야 합니다.

> In 2014, AWS Identity and Access Management added support for federated identities using OpenID Connect (OIDC). This feature allows you to authenticate AWS API calls with supported identity providers and receive a valid OIDC JSON web token (JWT). You can pass this token to the AWS STS `AssumeRoleWithWebIdentity` API operation and receive IAM temporary role credentials. You can use these credentials to interact with any AWS service, including Amazon S3 and DynamoDB. 

순서 상 Web Identity Token을 먼저 받아서 사용하도록 되어 있는데, IRSA를 사용하지 못한다면 STS를 사용할 수 있는지 확인해 보시기 바랍니다. 자세한 설명을 듣고 싶으시다면 계속 읽어 주세요. 

# Spring Boot 애플리케이션 생성하기

간단하게 S3 버킷 목록을 읽는 Spring Boot 애플리케이션을 만들어 보겠습니다. Spring의 [Quickstart Guide](https://spring.io/quickstart)를 이용해서 애플리케이션을 생성해 주세요. 저는 다음과 같이 설정했습니다. 

* Gradle Project
* Language: Java
* Dependencies: Spring Web

생성한 애플리케이션은 IntelliJ와 같은 IDE나 Visual Studio Code와 같은 에디터로 열어 봅니다.

AWS SDK를 사용하기 위해 `build.gradle` 파일을 수정합니다. 

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'

    // 아래 내용을 추가합니다. 
    implementation platform('software.amazon.awssdk:bom:2.20.74')
    implementation 'software.amazon.awssdk:s3'
    implementation 'software.amazon.awssdk:sso'     // 기본적인 방법으로 권한을 얻으려면 필요합니다.

    // IRSA로 권한을 얻으려면 STS가 필요하기 때문에 추가해 줍니다.
    implementation 'software.amazon.awssdk:sts'
}
```

그리고 생성된 애플리케이션 내에 S3 버킷 목록을 얻어오기 위한 기능을 추가해 줍니다. 애플리케이션의 클래스 이름은 생성하신 애플리케이션에 따라 다를 수 있습니다.

```java
package kr.gonigoni.eksiamrolepoc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.Bucket;
import software.amazon.awssdk.services.s3.model.ListBucketsResponse;

import java.util.ArrayList;

@SpringBootApplication
@RestController
public class EksIamRolePocApplication {
    /* 클라이언트를 클래스 멤버로 뺀 이유는 계속해서 연결을 유지할 수 있도록 하기 위함입니다.
       이에 클래스 생성자에서 클라이언트를 생성하도록 했습니다. 자세한 내용은 아래 페이지를 참고하세요.

       https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/using.html#creating-clients
    */
    private final S3Client client;

    public EksIamRolePocApplication() {
        // 클라이언트를 설정합니다. 별다른 설정을 하지 않았기 때문에, 위에서 설명한 순서대로 credential을 찾습니다.
        Region region = Region.AP_NORTHEAST_2;
        client = S3Client.builder().region(region).build();
    }

    public static void main(String[] args) {
        SpringApplication.run(EksIamRolePocApplication.class, args);
    }

    /* 버킷 이름을 얻어와서 ArrayList로 반환합니다. */
    private ArrayList<String> getS3BucketList() {
        ArrayList<String> result = new ArrayList<>();
        ListBucketsResponse response = client.listBuckets();

        for (Bucket bucket: response.buckets()) {
            // Append bucket name to ArrayList
            result.add(bucket.name());
        }

        return result;
    }

    /* /bucket_list 페이지로 이동하면 계정 내 있는 버킷 이름을 표시합니다. */
    @GetMapping("/bucket_list")
    public String getBucketList() {
        return String.join("<br/>", getS3BucketList());
    }
}

```

로컬에서 테스트하시는 경우, 로컬 환경에 AWS Access Key를 설정하셨는지 확인해 보시기 바랍니다.

* [AWS CLI로 설정하기](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-methods)
* [환경 변수로 설정하기](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)

빌드는 다음과 같이 하시면 됩니다. 

```shell
gradle build

# build/libs에 jar 파일이 생성된 것을 볼 수 있습니다.
```

# 컨테이너 이미지 생성하기 

Gradle로 빌드하면 jar 파일이 나오는데, 이 파일을 가지고 컨테이너 이미지를 생성해 보겠습니다. 아래는 이 프로젝트에서 사용할 Dockerfile 입니다. 아래 내용을 복사해서 Dockerfile로 저장해 주세요.

```dockerfile
FROM amazoncorretto:17

EXPOSE 8080

WORKDIR /app

COPY build/libs/eks-iam-role-poc-0.0.1-SNAPSHOT.jar /app

CMD ["java", "-jar", "eks-iam-role-poc-0.0.1-SNAPSHOT.jar"]
```

저는 우선 Docker Hub에 이미지를 올려서 테스트 해 보겠습니다. 프로젝트의 루트 디렉터리에서 실행해 주세요. 

```shell
docker login
docker build -t eks-iam-role-poc .
docker tag eks-iam-role-poc YOUR_DOCKER_HUB_USER_NAME/eks-iam-role-poc                   
docker push YOUR_DOCKER_HUB_USER_NAME/eks-iam-role-poc
```

# EKS 클러스터 생성하기 

eksctl을 이용해서 EKS 클러스터를 생성해 보겠습니다. 이 예제에서는 간단하게 테스트 하기 위해 Fargate를 이용해 보겠습니다. 

```shell
eksctl create cluster --name eks-iam-role-poc --region ap-northeast-2 --fargate

# ... 

# 아래와 같이 나오면 성공입니다. 
2023-05-27 21:14:19 [✔]  EKS cluster "eks-iam-role-poc" in "ap-northeast-2" region is ready
```

10분 넘게 걸리니 여유있게 기다립니다. kubectl을 사용하고 계시다면, `kubectl config get-contexts` 명령을 이용해서 생성된 클러스터를 가리키고 있는지 확인해 보세요. 생성한 클러스터 왼쪽에 `*`가 있다면 해당 클러스터를 기준으로 동작하고 있는 것입니다.

## EKS 클러스터와 연결할 IAM Role 생성하기 

먼저 클러스터에 대한 IAM OIDC Provider를 생성합니다. 콘솔에서도 할 수 있는 작업이니, [매뉴얼](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)을 참고해 주세요. 

IAM Role을 생성할 때 AssumeRolePolicy는 다음과 같이 생성합니다. (`eksctl`을 이용하면 좀 더 쉽게 생성 가능합니다)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:default:my-service-account",
                    "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

여기서 다음 부분을 설정합니다. 
* Principal → Federated 부분에서
    * 111122223333: Account ID를 입력합니다. 
    * region-code: AWS Region 코드입니다. 서울 리전이라면 `ap-northeast-2`로 입력합니다.
    * EXAMPLED539D4633E53DE1B71EXAMPLE: EKS 클러스터의 OpenID Connect Provider URL의 `/id/` 뒷부분과 동일하게 입력합니다.
* `default`는 namespace 이름, `my-service-account` 부분은 여러분이 설정하실 ServiceAccount 이름으로 설정합니다.

저는 namespace 설정은 안 해서 `default`로, Service account 이름은 뒤에 있는 manifest의 `metadata.name` 부분에 설정한 것과 같이 `eks-iam-role-poc`로 설정했습니다.

Policy 내용은 다음과 같습니다. 말 그대로 `ListAllMyBuckets` 권한만 준 상태입니다. 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```

IAM Role ARN은 복사해서 manifest 파일에 적용하시면 됩니다.

# Kubernetes Manifest 생성하기

간단하게 다음 리소스만 활용해서 구성해 보겠습니다. 

* ServiceAccount: IAM Role을 연결해 줍니다. 
* Deployment: 컨테이너를 설정하고, ServiceAccount와 연결해 줍니다.
* Service: 앞의 Deployment를 연결합니다.

아래 파일을 `manifest.yaml`로 저장합니다. 단, 아래 내용은 여러분의 상황에 따라 적절히 바꿔줍니다. 

* `YOUR_DOCKER_HUB_USER_NAME`: Docker Hub username 입력 (혹시나 이미지 이름이 다르거나 다른 저장소에 올리셨다면 `image` 뒤의 전체 내용을 바꿔 주세요)
* `YOUR_AWS_ACCOUNT_ID`: AWS Account ID 입력 (IAM Role 이름이 다르다면, 콘솔에서 ARN을 조회하여 붙여 주세요)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eks-iam-role-poc
  labels:
    app.kubernetes.io/name: eks-iam-role-poc
    app.kubernetes.io/instance: poc
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: eks-iam-role-poc
      app.kubernetes.io/instance: poc
  template:
    metadata:
      labels:
        app.kubernetes.io/name: eks-iam-role-poc
        app.kubernetes.io/instance: poc
    spec:
      serviceAccountName: eks-iam-role-poc
      containers:
        - name: eks-iam-role-poc
          image: YOUR_DOCKER_HUB_USER_NAME/eks-iam-role-poc
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: eks-iam-role-poc
  labels:
    app.kubernetes.io/name: eks-iam-role-poc
    app.kubernetes.io/instance: poc
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: eks-iam-role-poc
    app.kubernetes.io/instance: poc
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-iam-role-poc
  labels:
    app.kubernetes.io/name: eks-iam-role-poc
    app.kubernetes.io/instance: poc
  annotations:
    # 이 부분을 이용해서 IAM Role을 연결해 줍니다.
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/eks-iam-role-poc
```

# 테스트

앞에서 만든 manifest 파일을 배포합니다. 

```shell
kubectl apply -f manifest.yaml        
# 출력 내용
deployment.apps/eks-iam-role-poc created
service/eks-iam-role-poc created
serviceaccount/eks-iam-role-poc created
```

Pod이 정상적으로 실행되고 있는지 확인해 봅니다. Running 상태여야 합니다. 

```shell
kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
eks-iam-role-poc-7945d8bdf6-c5sdq   1/1     Running   0          74m
```

그렇다면 포트 포워딩으로 한 번 확인해 봅니다. Manifest 파일에서, 서비스에 연결된 포트를 80으로 설정해 두었기 때문에, 80을 Target Port로 설정합니다. (임의의 포트로 포워딩 됩니다)

```shell
kubectl port-forward service/eks-iam-role-poc :80
Forwarding from 127.0.0.1:61968 -> 8080
Forwarding from [::1]:61968 -> 8080
```

웹 브라우저에서 `localhost:61968/bucket_list` 로 접속하면, 버킷 목록이 표시되는 것을 볼 수 있습니다. (61968 번 포트는 상황에 따라 바뀔 수 있으니 출력 내용을 확인하세요)

확인했다면 Ctrl + c를 눌러서 실행 중인 포트 포워딩을 종료합니다.

# 리소스 정리 

먼저 EKS 클러스터 내 리소스를 정리합니다. 

```shell
kubectl delete -f manifest.yaml
deployment.apps "eks-iam-role-poc" deleted
service "eks-iam-role-poc" deleted
serviceaccount "eks-iam-role-poc" deleted
```

그리고 EKS 클러스터를 삭제해 줍니다. 

```shell
eksctl delete cluster --name eks-iam-role-poc
```

# 참고자료 

* [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)