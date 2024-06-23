---
title: "EKS에서 ASCP로 Parameter Store/Secrets Manager 이용하기"
date: 2024-06-23T20:51:32+09:00
tags: [AWS, EKS, Secrets Manager, Parameter Store]
draft: false
categories: [DevOps]
comments: true
---

## 배경

어떤 애플리케이션이든 민감한 정보를 관리할 필요가 있습니다. DB에 연결하기 위한 사용자 정보 및 비밀번호가 필요할 것이고, 외부 API를 호출하기 위해 API 키가 필요할 수도 있죠. 이러한 정보를 관리하는 방법에는 여러 가지가 있는데요. 이번에는 EKS 클러스터가 있다고 가정하고, AWS의 Parameter Store, Secrets Manager에 있는 값을 어떻게 Pod에 넣을 수 있을지 살펴 보겠습니다.

## 준비하기 

* AWS CLI 설치
* `kubectl`, `helm` 설치
* EKS 클러스터: 콘솔에서 생성하거나 `eksctl`과 같은 툴로 생성해 주세요. (**테스트 후 비용 문제로 클러스터는 꼭 삭제해 주시기 바랍니다.**)

## Secrets Store CSI Driver & ASCP(Amazon Secrets Manager and Config Provider) 설치

Kubernetes에서는 [Secret Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver)라는 것이 있습니다. 민감한 정보를 CSI Volume으로 관리할 수 있도록 하는 도구입니다. 즉, 민감한 정보를 특정 볼륨에 마운트 하는 방식으로 관리하는 것입니다. 이 도구는 여러가지 외부 Secret store provider를 지원하는데요. Azure, GCP, Vault 뿐만 아니라 AWS도 지원합니다. 

[AWS 서비스를 지원하기 위한 Provider](https://github.com/aws/secrets-store-csi-driver-provider-aws)는 Amazon Secrets Manager and Config Provider라는 긴 이름을 갖고 있는데요. 이름이 너무 길어서 다음부터는 ASCP라고 지칭하겠습니다. ASCP는 Secrets Manager와 Parameter Store를 지원합니다. Secret이 JSON 형식이라면 여러 값들 중 하나만 마운트 할 수도 있습니다. 그리고 Secret Manager의 rotation 기능도 지원합니다.

다만 제약사항이 있는데요. EC2 Node Group에서 실행하는 EKS 1.17 버전 이상이 필요하다고 합니다. 즉, Fargate Node Group은 지원하지 않습니다. 

이제 기본적인 설정 방법에 대해 알아 보겠습니다.

**(1) Kubernetes Secrets Store CSI Driver 설치 (Helm 이용)**

```shell
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver
```

**(2) ASCP 설치**

```shell
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

**(3) IAM Policy 생성하기**

ASCP를 이용하려면 Service Account, Service Account에 연결된 IAM Role & Policy가 있어야 합니다. Policy에는 다음과 같은 권한이 필요합니다.

* Parameter Store와 연동: `ssm:GetParameters`
* Secrets Manager와 연동: `secretsmanager:GetSecretValue`, `secretsmanager:DescribeSecret`

아래 내용을 `policy.json` 파일로 저장합니다. 프로덕션 용도라면 `Resource` 부분은 사용하는 Parameter나 Secret의 ARN을 넣어주세요. 

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": ["ssm:GetParameters", "secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
        "Resource": "*"
    }]
}
```

그리고 IAM Policy를 생성합니다. 출력되는 내용 중 Policy의 ARN을 메모해 주세요. IRSA를 연결할 때 필요합니다.

```shell
aws iam create-policy --policy-name ascp-test-policy --policy-document file://./policy.json
```

**(4) IRSA에 연결하기**

IRSA는 OIDC Provider를 필요로 하기 때문에, 클러스터에 OIDC Provider를 설정해야 합니다. [문서](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)를 참고하여 설정해 주세요.

그 후에는 IAM Role과 연결된 Service Account를 생성합니다. [문서](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html)를 참고하여 설정해 주세요.

## Parameter Store에서 Parameter 값을 가져오도록 설정

**(1) Parameter 만들기**

테스트용 Parameter를 만들어 보겠습니다.

```shell
aws ssm put-parameter --name /test/parameter --value gonitest --type SecureString 
```

**(2) Pod에 적용하기**

아래 내용을 `mypod_parameter_store.yaml` 파일로 저장합니다. 설명은 하단에 있으니 참고해 주세요.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-parameter-store
spec:
  provider: aws
  parameters:
    # `objectName` 부분은 Parameter의 이름으로 구성
    objects: |
      - objectName: "/test/parameter"   
        objectType: "ssmparameter"
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-parameter-store
spec:
  # ServiceAccountName은 앞에서 만들었던 Service Account 이름으로 작성해 줍니다.
  serviceAccountName: ascp-test
  # SecretProviderClass와 Volume 연결
  volumes:
    - name: parameter-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets-parameter-store
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    # 위에서 구성한 volume mount
    volumeMounts:
      - mountPath: "/var/secrets"
        name: parameter-store
```

`---` 위의 내용은 SecretProviderClass 리소스에 대한 내용입니다. 

* objectType: `ssmparameter`로 작성합니다.
* objectName: Parameter의 이름을 적어줍니다. (ARN은 사용 불가)

`---` 아래의 내용은 Pod에 설정해야 하는 내용입니다. (상황에 따라 Deployment에 설정하셔도 됩니다)

* `spec.serviceAccountName`을 지정해야 합니다. 저는 `ascp-test`라는 ServiceAccount를 만들어서 이걸로 설정했습니다.
* `spec.volume`은 위의 `SecretProviderClass`에 작성한 내용과 연결해 줍니다.
* `spec.containers[*].volumeMounts`에는 위에 연결했던 Volume과 연결합니다. (이 Pod의 설정에서는 `/var/secrets` 디렉토리에 마운트 됩니다)

**(3) 확인하기**

실제로 EKS 클러스터에 적용해 봅니다. 

```shell
kubectl apply -f mypod_parameter_store.yaml

# Output
secretproviderclass.secrets-store.csi.x-k8s.io/aws-secrets-parameter-store created
pod/nginx-pod-parameter-store created
```

그리고 Pod의 터미널로 들어가서 확인해 보겠습니다.

```shell
kubectl exec -it nginx-pod-parameter-store -- sh

## 여기서부터는 Pod 안에서 실행합니다.
cd /var/secrets/
ls
_test_parameter
cat _test_parameter
gonitest
```

Parameter 이름의 `/`는 `_`로 대체된 것을 알 수 있습니다. 파일을 열어보면 제가 Parameter Store에 넣은 값과 동일하네요.

## Secrets Manager에서 Parameter 값을 가져오도록 설정

**(1) Secret 만들기**

JSON 포맷으로 간단하게 Secret을 생성해 보겠습니다.

```shell
aws secretsmanager create-secret --name test/secret --secret-string "{\"username\":\"example\",\"password\":\"example1234\"}"
```

**(2) Pod에 적용하기**

아래 내용을 `mypod_secrets_manager.yaml` 파일로 저장합니다. 

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-secrets-manager
spec:
  provider: aws
  parameters:
    # `objectName` 부분은 Secret의 이름이나 ARN으로 구성
    objects: |
      - objectName: "test/secret"   
        objectType: "secretsmanager"
        jmesPath:
          - path: "username"
            objectAlias: "SECRET_USERNAME"
          - path: "password"
            objectAlias: "SECRET_PASSWORD"
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-secrets-manager
spec:
  # ServiceAccountName은 앞에서 만들었던 Service Account 이름으로 작성해 줍니다.
  serviceAccountName: ascp-test
  # SecretProviderClass와 Volume 연결
  volumes:
    - name: secrets-manager
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: aws-secrets-secrets-manager
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
      requests:
        memory: "64Mi"
        cpu: "250m"
    # 위에서 구성한 volume mount
    volumeMounts:
      - mountPath: "/var/secrets"
        name: secrets-manager
```

`---` 위의 내용은 SecretProviderClass 리소스에 대한 내용입니다. 

* objectType: `secretsmanager`로 작성합니다.
* objectName: Secret의 이름을 적거나, Secret의 ARN을 지정할 수 있습니다.
* `jmesPath`: JMESPath를 이용하여 JSON 포맷 secret에서 가져올 특정한 값을 지정합니다. (이 글에서는 사용자 이름은 `SECRET_USERNAME`, 비밀번호는 `SECRET_PASSWORD`라는 이름으로 가져오는 것을 가정하였습니다.)

Pod에 설정한 내용은 위의 내용과 동일하여 생략하겠습니다.

**(3) 확인하기**

이것도 실제로 EKS 클러스터에 적용해 봅니다. 

```shell
kubectl apply -f mypod_secrets_manager.yaml

# Output
secretproviderclass.secrets-store.csi.x-k8s.io/aws-secrets-secrets-manager created
pod/nginx-pod-secrets-manager created
```

그리고 Pod의 터미널로 들어가서 확인해 보면 다음과 같이 표시될 것입니다. 

```shell
kubectl exec -it nginx-pod-secrets-manager -- sh

## 여기서부터는 Pod 안에서 실행합니다.
cd /var/secrets
ls
SECRET_PASSWORD  SECRET_USERNAME  test_secret
cat test_secret
{"username":"example","password":"example1234"}
cat SECRET_USERNAME
example
cat SECRET_PASSWORD
example1234
```

역시나 Secret 이름의 `/`는 `_`로 대체되었습니다. Secret 이름과 같은 파일을 열어보면 Secret 값이 JSON 형식으로 들어있습니다. Secrets Manager 콘솔에서 표시되는 값과 동일함을 알 수 있습니다. 

그리고 `SecretProviderClass`의 `jmesPath`항목에 설정한 것들은 manifest의 `objectAlias` 항목에 지정한 이름과 동일한 파일이 만들어진 것을 볼 수 있습니다.

## 참고자료 

* [Use AWS Secrets Manager secrets in Amazon Elastic Kubernetes Service](https://docs.aws.amazon.com/ko_kr/secretsmanager/latest/userguide/integrating_csi_driver.html)
* [Use Parameter Store parameters in Amazon Elastic Kubernetes Service](https://docs.aws.amazon.com/ko_kr/systems-manager/latest/userguide/integrating_csi_driver.html)
* [AWS Secrets Manager and Config Provider for Secret Store CSI Driver](https://github.com/aws/secrets-store-csi-driver-provider-aws)