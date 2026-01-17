---
title: "AWS만 쓰던 사람이 Azure 써 보는 이야기: (4) 컨테이너"
date: 2026-01-17T11:16:22+09:00
tags: [Azure]
draft: false
categories: [Azure]
comments: true
---

AWS만 썼던 사람이 Azure 쓰는 이야기 마지막 편으로 컨테이너 관련 서비스를 살펴 보겠습니다. 

컨테이너가 나오기 전에는 서버에 필요한 것들을 세팅하고, 애플리케이션 소스나 실행 파일을 올리는 형태로 서비스를 제공했습니다. 관리해야 할 서버 개수가 늘어나면서 서버 관리 방법이 복잡해졌는데요. Ansible, Chef와 같은 구성 관리 툴 덕분에 많은 서버에 같은 설정을 쉽게 수행할 수 있게 되었습니다.

그러다가 Docker와 같은 컨테이너 빌드 툴이 나왔습니다. Dockerfile을 작성하고, 빌드만 하면 애플리케이션에 필요한 의존성과 애플리케이션을 컨테이너 이미지로 배포할 수 있게 되었습니다. 그리고 컨테이너 개수가 급격하게 많아지면서, 컨테이너 기반 애플리케이션을 어떻게 운영해야 할 지 고민하게 되었죠. 그동안 다수의 컨테이너를 운영하기 위한 많은 방법이 나왔지만, 결국 Kubernetes로 수렴하는 것 같습니다. 일단 AWS, Azure 외에도 많은 클라우드 서비스들이 Kubernetes는 기본적으로 지원하는 것만 봐도 알 수 있습니다.

이번 글에서는 Azure의 컨테이너 관련 서비스 중 Container Registry, Azure Kubernetes Service에 대해 살펴보겠습니다. 

## Azure Container Registry (ACR)

AWS에 ECR이 있다면, Azure에는 Container Registry가 있습니다. 줄여서 ACR이라고 하기도 합니다. 두 서비스를 한 번 살펴보겠습니다. 

AWS는 Private/Public 레지스트리가 각각 있고, 각 레지스트리에 리포지토리 단위로 컨테이너 이미지를 관리할 수 있습니다. (즉, 별도의 레지스트리를 생성할 수는 없음) 아래 스크린샷처럼 각각의 레지스트리를 나눠서 관리합니다.

{{< figure src="/img/starting-azure-4-1.png" width="30%" >}}

Azure는 여러 개의 레지스트리를 생성할 수 있습니다. 여러 개의 레지스트리를 관리할 수 있고, 하나의 레지스트리 안에는 여러 개의 리포지토리를 관리할 수 있습니다. 아래 스크린샷을 보면 왼쪽은 레지스트리를 관리하는 부분이고, 레지스트리를 클릭하면 메뉴 아래에 리포지토리를 관리하도록 설정할 수 있습니다. 

{{< figure src="/img/starting-azure-4-2.png" >}}

ECR Public처럼 운영할 일이 있다면 어떻게 해야 할까요? Azure에서는 ACR 레지스트리 단위로 로그인하지 않은 사용자에게 pull 액세스를 허용하는 방법이 있습니다. 주의해야 할 점은, 레지스트리 내 모든 리포지토리에 일괄적으로 해당 설정을 적용합니다. 해당 설정은 콘솔에서는 설정할 수 없지만 Azure CLI로 설정할 수 있습니다. 아래 링크한 문서를 참고하세요. 

[인증되지 않은 익명 끌어오기 액세스](https://learn.microsoft.com/ko-kr/azure/container-registry/anonymous-pull-access)

## Azure Kubernetes Service (AKS)

AWS의 EKS에 대응하는 관리형 Kubernetes 서비스로 AKS(Azure Kubernetes Service)가 있습니다. Kubernetes의 컨트롤 플레인은 AWS나 Azure가 관리하고, 데이터 플레인은 사용자가 관리하는 방식은 동일합니다. 클러스터를 구성하는 방법에 큰 차이는 없지만 몇 가지 다른 점이 있어서 말씀 드릴게요. 

### 클러스터 구성

AWS에서 EKS 클러스터를 구성하는 것과 같이 Azure에서도 AKS 클러스터를 생성할 수 있습니다. 클러스터 생성 방법은 [링크](https://learn.microsoft.com/ko-kr/azure/aks/learn/quick-kubernetes-deploy-cli)를 참조하세요. (페이지 왼쪽을 보시면 Terrafrom과 같은 다른 방법으로 구성하는 방법도 나와 있습니다)

kubeconfig 파일을 받을 때는 Azure CLI를 이용하여 아래 명령을 입력하시면 됩니다. 

```shell
# '-f FILENAME' 옵션은 기본 kubeconfig($HOME/.kube/config) 대신 다른 이름으로 저장할 때 사용합니다. 
az aks get-credentials --resource-group (리소스 그룹 이름) --name (클러스터 이름) --admin -f $HOME/.kube/aks.config
```

Terraform으로 간단한 클러스터를 생성해 봤는데 3분~4분 정도 소요되었습니다.

**노드 풀 구성 시 주의할 점**

워커 노드를 생성하려면 노드 그룹을 생성해야 합니다. Azure의 AKS에서도 노드 그룹에 대응되는 노드 풀을 생성합니다. 노드 풀에 Label 및 Taint 설정도 가능합니다. 다만 AKS의 노드 풀을 구성할 때는 다음과 같은 점에 유의해야 합니다.

- 클러스터를 구성할 때 기본 노드 풀(시스템 노드 풀)을 생성해야 합니다. ([참고자료](https://learn.microsoft.com/ko-kr/azure/aks/use-system-pools))
    - 2개 이상의 노드를 포함해야 합니다. 
    - 운영체제는 Linux여야 합니다. 
    - Spot 서버로 구성할 수 없습니다. 
    - B 시리즈 VM(EC2의 t3, t4g 같은 인스턴스 타입)은 이용 불가능합니다.
- 시스템 노드 풀 외에 추가로 사용자가 노드 풀을 추가할 수 있습니다. 사용자가 생성한 노드 풀에는 시스템 노드 풀과 같은 제약이 없습니다. 노드 풀 내 서버 개수도 0으로 지정할 수 있습니다. 

**ACR에서 이미지를 가져올 수 있는 권한 추가**

EKS에서는 각각의 워커 노드에 ECR에 대한 읽기 전용 권한을 부여하면 Private 이미지를 가져올 수 있습니다. AKS에서는 클러스터 단위로 권한을 설정할 수 있습니다. ([참고자료](https://learn.microsoft.com/ko-kr/azure/aks/cluster-container-registry-integration?tabs=azure-cli))

(1) Azure CLI를 이용하는 방법

```shell
# 새 클러스터를 만들 때 연결하기
az aks create --name <cluster-name> --resource-group <resource-group-name> --generate-ssh-keys --attach-acr <acr-name>

# 기존에 있던 클러스터에 연결하기
az aks update --name <cluster-name> --resource-group <resource-group-name> --attach-acr <acr-name>
```

(2) Terraform으로 구성하는 방법

```hcl
# azurerm_container_registry 리소스 및 azurerm_kubernetes_cluster 리소스가 있다고 가정합니다. 
resource "azurerm_role_assignment" "acr_role" {
  principal_id                     = azurerm_kubernetes_cluster.your_cluster.kubelet_identity[0].object_id
  role_definition_name             = "AcrPull"
  scope                            = azurerm_container_registry.your_registry.id
  skip_service_principal_aad_check = true
}
```

### 애플리케이션 라우팅

애플리케이션을 외부에서 접근 가능하도록 설정할 때, Ingress를 기반으로 설정하는 경우가 많습니다. (다만 요즘은 Gateway API를 이용하는 것을 권장합니다) Ingress를 설정하려면 Ingress Controller를 따로 설치해야 하는데요. ingress-nginx와 같은 애플리케이션을 설치합니다. AWS에서는 AWS Load Balancer Controller를 설치해서 ALB/NLB를 쉽게 연동할 수 있습니다.

AKS에서 로드 밸런서와 연결하려면 여러가지 방법이 있는데, 
- AWS의 NLB에 대응하는 Load Balancer와 연결: Service에 `type: LoadBalancer`만 설정해도 가능합니다. (아래 참조)
- AWS의 ALB에 대응하는 Application Gateway와 연결: [컨테이너용 Application Gateway](https://learn.microsoft.com/ko-kr/azure/application-gateway/for-containers/overview)를 설치 후 연결

근데 컨테이너용 Application Gateway를 연결하려고 하다 보니 이래저래 고려해야 할 점이 많더라구요. 더 쉬운 방법이 있을지 찾아보다가 [Application Routing](https://learn.microsoft.com/ko-kr/azure/aks/app-routing) 기능을 발견했습니다. 이 기능을 이용하면 ingress-nginx를 설치하지 않아도 바로 Ingress를 연결할 수 있습니다.

AKS 클러스터에 애플리케이션 라우팅을 활성화 해 봅시다. 

(1) Azure CLI

```shell
az aks approuting enable --resource-group <ResourceGroupName> --name <ClusterName>
```

(2) Terraform

```hcl
resource "azurerm_kubernetes_cluster" "my_aks_cluster" {
  # (중간 생략)

  # 아래 내용을 추가해 주세요.
  web_app_routing {
    dns_zone_ids = []
  }

  # (중간 생략)
}
```

이제 Ingress를 다음과 같이 설정해서 배포합니다. (`spec.ingressClassName: webapprouting.kubernetes.azure.com` 부분에 주목!)

```yaml
# Service, Deployment는 생략했습니다. (다음 파트의 YAML 파일 내용 중, Service의 spec.type만 ClusterIP로 바꿔서 넣어도 됩니다)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: default
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

`kubectl get ingress` 명령으로 조회해 보면 아래와 같이 나옵니다. `ADDRESS` 항목이 비어 있으면 조금 기다렸다가 다시 확인해 주세요.

```shell
$ kubectl get ingress
NAME            CLASS                                HOSTS   ADDRESS        PORTS   AGE
nginx-ingress   webapprouting.kubernetes.azure.com   *       (임의의 IP 주소)  80      73s
```

`ADDRESS` 항목에 있는 IP를 복사해서 웹 브라우저에 접속해 보면 페이지가 나오는 것을 볼 수 있습니다.

### Service에 `type: LoadBalancer`를 지정하면?

아래와 같은 k8s manifest를 작성 후 `test.yaml` 파일로 저장합니다. 그리고 `kubectl apply -f test.yaml`을 실행해 보겠습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Ingress Controller를 설치하지 않았을 때, Service의 `spec.type` 항목의 Load Balancer를 지정하면 어떻게 동작할까요?

**EKS에서 동작**: CLB와 CLB에 연결된 보안 그룹을 생성합니다. 워커 노드를 타겟 인스턴스로 지정합니다. `kubectl`에서 조회해 보면 아래와 같이 표시됩니다. 

```shell
$ kubectl get svc
NAME            TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
kubernetes      ClusterIP      10.100.0.1       <none>           443/TCP        62m
nginx-service   LoadBalancer   10.100.195.137   (CLB DNS name)   80:30464/TCP   30m
```

**AKS에서 동작**: AKS가 관리하는 노드 풀에 Load Balancer가 있습니다. (AWS의 NLB와 같다고 생각하시면 됩니다) Load Balancer에 공인 IP를 추가하고 서비스와 연결합니다. `kubectl`로 조회해 보면 아래와 같습니다.

```shell
$ kubectl get svc                              
NAME            TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP      10.1.0.1      <none>        443/TCP        71m
nginx-service   LoadBalancer   10.1.67.147   (Public IP)   80:31821/TCP   88s
```

`EXTERNAL-IP` 항목에 표시된 DNS Name이나 IP 주소로 접속하면 페이지를 확인할 수 있습니다. 

## 마무리

다른 클라우드 환경을 배우는 건 외국어를 배우는 것 같다는 생각이 들었습니다. 한국어로 말하려고 했던 걸 영어로는 어떻게 해야 할 지 고민하는 것과 비슷한데요. AWS에서 제가 써 보려는 기능이 있다면, Azure에서 이와 비슷한 기능은 어떤 것이 있는지, 그리고 어디서 설정해야 하는 지 찾는 일이 많았습니다. 그러면서 시행착오를 아주 많이 경험했네요. 다만 뒤돌아 보면 '아, AWS에서는 이렇게 했던 걸 Azure에서는 저렇게 하는 거였구나'라고 깨달은 순간이 종종 있었습니다. 

다만 디테일을 좀 더 생각해 보면, 같은 개념이어도 미묘하게 다른 부분이 있었습니다. 예를 들어, 네트워크를 다룬 1편에서 보안 그룹 이야기를 했었는데 AWS의 보안 그룹과 Azure의 네트워크 보안 그룹, 애플리케이션 보안 그룹은 설정 방법이 다릅니다. AWS의 보안 그룹은 내가 허용할 Inbound/Outbound를 Stateful하게 설정하는 기능이지만, Azure의 네트워크 보안 그룹 설정 방법은 오히려 AWS VPC의 Network ACL과 비슷하죠. 애플리케이션 보안 그룹은 서버에 어떤 레이블을 추가한다는 느낌으로 보시면 됩니다. 이런 과정을 겪으며 자세히 들여봐야 하는 차이를 이해하고 내가 필요한 상황에 적용하는 것이 중요하다고 느꼈습니다. 

Azure 관련 글을 쓸 때도 있겠지만 이제는 다른 주제로 찾아 뵙겠습니다. 감사합니다.