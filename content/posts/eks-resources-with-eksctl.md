---
title: "eksctl로 생성한 EKS 클러스터의 리소스 살펴 보기"
date: 2022-07-17T21:45:26+09:00
tags: [AWS, EKS, Kubernetes]
draft: false
categories: [DevOps]
comments: true
---

AWS에서 Kubernetes를 사용하려면 여러 방법이 있겠지만, 가장 편한 방법은 EKS 서비스를 이용하는 것이죠. EKS 클러스터를 생성하는 방법은 여러 가지가 있는데요. 이번 글에서는 eksctl이라는 프로그램을 이용해서 EKS 클러스터를 생성하는 방법을 알아보고, 어떤 리소스를 생성하는지 알아보려고 합니다. 

먼저 Kubernetes 클러스터의 구조를 알아보고, EKS에서는 어떤 차이점이 있는지 알아 보겠습니다. 

# Kubernetes 클러스터의 구조와 EKS 

Kubernetes 클러스터는 다음과 같이 구성되어 있습니다. 

{{< figure src="https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg" caption="출처: [kubernetes.io](https://kubernetes.io/ko/docs/concepts/overview/components/)" >}}

* [노드(Node)](https://kubernetes.io/ko/docs/concepts/architecture/nodes/): 컨테이너 내의 애플리케이션을 실행하는 서버들의 집합으로, 파드(Pod)는 워커 노드에서 동작합니다. 앞으로 노드 또는 워커 노드라는 말이 계속 등장할텐데, 똑같은 것으로 이해해 주시면 됩니다.
* **컨트롤 플레인(Control Plane)**: 워커 노드와 클러스터 내의 파드를 관리합니다. 프로덕션 환경에서는 컨트롤 플레인을 여러 머신에 배포하여 내결함성/고가용성을 확보합니다. 

컨트롤 플레인은 다음과 같은 구성요소들을 가지고 있습니다. 

* kube-apiserver: Kubernetes API를 외부에 노출하는 역할을 합니다. 
* etcd: 클러스터 데이터를 담는 키-값 저장소입니다.
* kube-scheduler: 노드가 배정되지 않은 새로 생성된 파드를 감지하고, 실행할 노드를 선택합니다. 
* kube-controller-manager: [컨트롤러](https://kubernetes.io/ko/docs/concepts/architecture/controller/) 프로세스를 실행하는 컴포넌트입니다. 노드 컨트롤러, 레플리케이션 컨트롤러, 엔드포인트 컨트롤러, 서비스 어카운트 & 토큰 컨트롤러를 포함합니다. 
* cloud-controller-manager: 클러스터를 클라우드 공급자의 API에 연결하고, 해당 클라우드 플랫폼과 상호 작용하는 컴포넌트와 클러스터와만 상호 작용하는 컴포넌트를 구분할 수 있게 해 줍니다. 

노드는 다음과 같은 구성요소들을 가지고 있습니다. 

* kubelet: 클러스터의 각 노드에서 실행되는 에이전트로, 파드에서 컨테이너가 확실히 동작하도록 관리합니다. 
* kube-proxy: 클러스터의 각 노드에서 실행되는 네트워크 프록시입니다. 노드의 네트워크 규칙을 관리하며, 규칙에 따라 내부 네트워크 또는 클러스터 바깥에서 파드로 네트워크 통신을 가능하게 해 줍니다. 
* 컨테이너 런타임: 컨테이너 실행을 담당하는 소프트웨어입니다. 자세한 내용은 [링크](https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/)를 참고하세요. 

마지막으로 애드온은 Kubernetes 리소스를 이용하여 클러스터 기능을 구현합니다. 애드온과 관련된 내용은 [링크](https://kubernetes.io/ko/docs/concepts/cluster-administration/addons/)를 참고해 주세요. 

## EKS는 어떤 차이가 있나요?

EKS 또한 컨트롤 플레인과 노드로 구성되어 있습니다. 

* **컨트롤 플레인**은 AWS에서 관리하는 계정에서 실행되고, Kubernetes API는 클러스터와 연결된 EKS 엔드포인트를 통해 노출됩니다.
* **노드**는 사용자의 AWS 계정에서 실행되고, 사용자는 API 서버 엔드포인트를 통해 클러스터의 컨트롤 플레인과 연결합니다. 

즉, **컨트롤 플레인은 AWS가 관리해 주고, 노드는 사용자가 관리한다고 보시면 되겠습니다.** 

EKS 클러스터의 노드는 자체 관리형 노드, EKS 관리형 노드 그룹, Fargate로 구성할 수 있습니다. 각각의 차이는 다음과 같습니다. 

* EKS 관리형 노드 그룹: EKS 클러스터의 노드 프로비저닝과 수명 주기 관리를 자동화합니다. 사용자가 직접 EC2 인스턴스를 생성하고 EKS 클러스터에 등록할 필요가 없다는 의미입니다.
* 자체 관리형 노드: 사용자의 AWS 계정에서 동작하는 EC2를 자체적으로 노드 그룹으로 배포할 수 있습니다. 노드 그룹 내 모든 인스턴스는 동일한 인스턴스 타입, 동일한 AMI 실행, 동일한 EKS 노드 IAM 역할을 사용하여야 한다는 조건이 있습니다. `kubernetes.io/cluster/<your_cluster_name>` 태그의 값을 `owned`로 지정하면 자체 관리형 노드를 추가할 수 있습니다. 
* AWS Fargate: 가상 머신을 생성할 필요 없이 컨테이너에 필요한 컴퓨팅 자원을 제공하는 기술입니다.

# eksctl로 EKS 클러스터 생성하기 

eksctl은 Weaveworks라는 회사에서 만든 EKS 관리 도구입니다. AWS 공식 문서에서도 eksctl으로 클러스터 생성 방법을 설명하는 것으로 보아, 거의 공식 툴처럼 취급받는 것 같습니다. (GitHub의 [eksctl GitHub 저장소](https://github.com/weaveworks/eksctl)에서도 'The official CLI for Amazon EKS'라고 하는 것을 보면 공식 툴이 맞는 것 같습니다 😅)

제가 테스트에 사용한 eksctl 버전은 0.105.0 이며, 다음과 같은 조건으로 클러스터를 생성하겠습니다. 

* 제가 지정한 이름으로 클러스터를 생성합니다. 
* 서울 리전에 클러스터를 생성하고, Default VPC에 생성한 Private Subnet을 이용합니다. 
* 노드 생성은 기본 옵션을 이용합니다. eksctl은 `m5.large` 타입의 EC2 인스턴스 두 대를 생성할 것입니다. 
* 나머지 옵션은 모두 기본값을 이용합니다. (예를 들어 AMI라던가, ...)

eksctl을 설치하고 터미널에서 다음과 같이 입력합니다. 

```shell
eksctl create cluster --name <your-cluster-name> --region ap-northeast-2 \
                      --vpc-private-subnets subnet-<your-subnet-id-1>,subnet-<your-subnet-id-2> \
                      --node-private-networking

# 작업을 완료하려면 10분 이상 걸립니다. 컨트롤 플레인을 구성하는 데 가장 많은 시간이 걸렸습니다. 
(위의 내용은 생략)
2022-07-19 18:19:16 [ℹ]  building cluster stack "eksctl-<your-cluster-name>-cluster"
2022-07-19 18:19:16 [ℹ]  deploying stack "eksctl-<your-cluster-name>-cluster"
2022-07-19 18:19:46 [ℹ]  waiting for CloudFormation stack "eksctl-<your-cluster-name>-cluster"
(중략)
2022-07-19 18:30:18 [ℹ]  waiting for CloudFormation stack "eksctl-<your-cluster-name>-cluster"

# 바로 다음에는 EKS가 관리하는 노드 그룹을 생성합니다. 생각했던 것보다 시간이 많이 걸리네요.
2022-07-19 18:32:19 [ℹ]  building managed nodegroup stack "eksctl-<your-cluster-name>-nodegroup-ng-<node-group-id>"
2022-07-19 18:32:20 [ℹ]  deploying stack "eksctl-<your-cluster-name>-nodegroup-ng-<node-group-id>"
(중략)
2022-07-19 18:35:59 [ℹ]  waiting for CloudFormation stack "eksctl-<your-cluster-name>-nodegroup-ng-<node-group-id>"
2022-07-19 18:35:59 [ℹ]  waiting for the control plane availability...

# 컨트롤 플레인과 연결하기 위한 설정인 kubeconfig를 저장합니다. 
2022-07-19 18:35:59 [✔]  saved kubeconfig as "/Users/<user_name>/.kube/config"
2022-07-19 18:35:59 [ℹ]  no tasks
2022-07-19 18:35:59 [✔]  all EKS cluster resources for "<your-cluster-name>" have been created

# 노드 그룹 생성이 완료되면 노드가 생성됩니다. 
2022-07-19 18:36:00 [ℹ]  nodegroup "ng-<node-group-id>" has 2 node(s)
2022-07-19 18:36:00 [ℹ]  node "ip-172-31-57-103.ap-northeast-2.compute.internal" is ready
2022-07-19 18:36:00 [ℹ]  node "ip-172-31-90-121.ap-northeast-2.compute.internal" is ready
2022-07-19 18:36:00 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-<node-group-id>"

# 아래와 같은 메시지가 나오면 클러스터 생성이 완료된 것입니다. 
2022-07-19 18:36:10 [ℹ]  kubectl command should work with "/Users/<user_name>/.kube/config", try 'kubectl get nodes'
2022-07-19 18:36:10 [✔]  EKS cluster "<your-cluster-name>" in "ap-northeast-2" region is ready
```

참고로 `--node-private-networking` 옵션을 추가한 이유는, 프라이빗 서브넷에만 노드를 생성할 때 필요한 옵션이어서 그렇습니다. 

이 옵션을 추가하지 않으면 CloudFormation이 노드 그룹을 생성할 때 `No export named eksctl-<your-cluster-name>-cluster::SubnetsPublic found. Rollback requested by user.` 에러가 발생하면서 노드 그룹이 생성되지 않습니다. 

# eksctl로 생성한 리소스 확인하기 

## CloudFormation Stack 확인하기

eksctl은 EKS 클러스터를 만들 때, `eksctl-<your-cluster-name>-cluster` 라는 이름의 CloudFormation 스택을 생성합니다. 이 스택은 다음과 같은 리소스로 구성되어 있습니다. (가린 부분은 클러스터 이름입니다)

아래 설명은 Logical ID 기준입니다.

{{< figure src="/img/eks-resources-with-eksctl-1.png" caption="EKS 클러스터 CloudFormation 스택" >}}

* **ControlPlane**: 말 그대로 컨트롤 플레인입니다. 이 부분은 AWS에서 관리합니다. 
* 보안 그룹
    * **ClusterSharedNodeSecurityGroup**: `eksctl-<cluster_name>-cluster-ClusterSharedNodeSecurityGroup-<임의의 문자/숫자>`이라는 이름의 보안 그룹으로, **"클러스터 내 모든 노드 간 통신"** 이라는 설명을 보실 수 있습니다. 노드들이 서로 통신할 수 있도록 모든 트래픽을 허용하는 인바운드 규칙이 있고, 모든 아웃바운드 트래픽을 허용합니다.
    * **ControlPlaneSecurityGroup**: `eksctl-<cluster_name>-cluster-ControlPlaneSecurityGroup-<임의의 문자/숫자>`라는 이름의 보안 그룹으로, **"컨트롤 플레인과 워커 노드 그룹 간 통신"** 이라는 설명을 보실 수 있습니다. 인바운드 규칙은 없고, 모든 트래픽을 허용하는 아웃바운드 규칙만 있습니다. 콘솔에서는 *Additional security groups*라는 이름으로 등록되어 있습니다.
* 보안 그룹의 Ingress 규칙
    * **IngressDefaultClusterToNodeSG**: 이 스택에는 없지만, 보안 그룹을 조회하면 `eks-cluster-sg-<cluster-name>-<임의의_숫자>`라는 이름의 보안 그룹이 있습니다. 보안 그룹의 설명을 보면, **EKS가 컨트롤 플레인의 마스터 노드에 붙은 ENI에 적용되는 보안 그룹을 만들었다**는 내용이 있습니다. EKS 콘솔에서 클러스터 정보를 조회하면 *Cluster security group*으로 등록되어 있는데요. 클러스터 보안 그룹에 붙은 규칙으로, ClusterSharedNodeSecurityGroup으로부터 온 모든 트래픽을 허용합니다.
    * **IngressInterNodeGroupSG**: ClusterSharedNodeSecurityGroup으로부터 온 모든 트래픽을 허용하는 규칙으로, ClusterSharedNodeSecurityGroup 보안 그룹에 붙어 있습니다.
    * **IngressNodeToDefaultClusterSG**: ControlPlane의 보안 그룹으로부터 온 모든 트래픽을 허용하는 규칙으로, ClusterSharedNodeSecurityGroup 보안 그룹에 붙어 있습니다.
* **ServiceRole**: EKS 서비스가 여러 작업들을 수행할 수 있도록 하는 서비스 역할입니다. 아래와 같은 정책을 포함합니다.
    * _(AWS 관리 정책)_ **AmazonEKSClusterPolicy**: Kubernetes가 사용자를 대신해서 리소스를 관리하는 권한입니다. Auto scaling, EC2, ELB, KMS의 서비스를 호출하고, 서비스와 연결된 역할을 생성하는 권한이 있습니다. 
    * _(AWS 관리 정책)_ **AmazonEKSVPCResourceController**: 워커 노드의 ENI와 IP를 관리하는 권한입니다. ENI 관련 권한이 붙어 있는 것을 볼 수 있습니다.  
    * **PolicyCloudWatchMetrics**: CloudWatch Metric에 지표 정보를 올리는 권한입니다. 
    * **PolicyELBPermissions**: 계정 속성 확인(`ec2:DescribeAccountAttributes`), 주소 확인(`ec2:DescribeAddresses`), 인터넷 게이트웨이 확인(`ec2:DescribeInternetGateways`) 권한이 있습니다. 

EKS 클러스터 CloudFormation 스택을 생성한 후에는 관리형 노드 그룹을 CloudFormation 스택으로 생성합니다. `eksctl-(클러스터 이름)-nodegroup-ng-(임의의 숫자/알파벳)`이라는 이름을 보실 수 있을 것입니다. 이 스택은 다음과 같은 리소스로 구성되어 있습니다. (가린 부분은 클러스터 이름 및 노드 그룹의 임의의 숫자/알파벳 입니다)

아래 설명은 Logical ID 기준입니다.

{{< figure src="/img/eks-resources-with-eksctl-2.png" caption="EKS 클러스터 노드 그룹의 CloudFormation 스택" >}}

* **LaunchTemplate**: 워커 노드의 Launch Template 입니다. 노드 그룹에 대한 태그, gp3 타입의 EBS 80 GB, 컨트롤 플레인의 보안 그룹을 사용하는 것으로 설정되어 있습니다. (컨트롤 플레인의 보안 그룹 ID는 클러스터 스택의 Output에서 가져옵니다)
* **ManagedNodeGroup**: EKS 클러스터에 연결하는 노드 그룹입니다. 위의 LaunchTemplate에 따라 노드를 생성합니다. 각 노드에는 아래의 NodeInstanceRole이 붙어 있고, 기본값으로는 2대를 생성합니다. (서브넷 설정은 클러스터 스택의 Output에서 가져옵니다)
* **NodeInstanceRole**: 워커 노드는 EC2 인스턴스이므로, 사용자를 대신해서 EC2 노드가 수행할 역할입니다. 다음과 같은 AWS 관리 정책이 붙어 있습니다. 
    * **AmazonEKSWorkerNodePolicy**: EKS 워커 노드가 EKS 클러스터에 연결하는 것을 허용합니다.
    * **AmazonEC2ContainerRegistryReadOnly**: ECR 저장소에 읽기 전용으로 액세스 할 수 있습니다. 
    * **AmazonSSMManagedInstanceCore**: Systems Manager 서비스의 핵심 기능을 활성화 합니다. ([참고자료](https://aws.amazon.com/ko/blogs/mt/applying-managed-instance-policy-best-practices/))
    * **AmazonEKS_CNI_Policy**: Amazon VPC CNI 플러그인(amazon-vpc-cni-k8s)이 EKS 워커 노드의 IP 주소 설정을 수정하기 위한 권한을 얻습니다. ENI와 관련된 권한을 사용하는 것을 볼 수 있습니다. 

이 CloudFormation 스택에서는 보이지 않지만, 노드 그룹을 생성하면 Auto Scaling Group을 생성합니다. 노드를 추가하거나 종료할 때는 이 CloudFormation 스택의 Launch Template을 이용합니다. 자세한 내용은 [링크](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/managed-node-groups.html)를 참고해 주세요. 

## 클러스터 내 리소스 알아보기 

아무것도 생성하지 않은 EKS 클러스터에는 어떤 리소스들이 있을까요? eksctl을 이용해서 클러스터를 생성했다면 클러스터 설정까지 반영되어 있을 것입니다. 바로 kubectl을 이용해서 리소스를 살펴보시죠. 

먼저 노드부터 한 번 살펴 보겠습니다. 

```shell
$ kubectl get nodes
NAME                                               STATUS   ROLES    AGE     VERSION
ip-172-31-57-103.ap-northeast-2.compute.internal   Ready    <none>   3m31s   v1.22.9-eks-810597c
ip-172-31-90-121.ap-northeast-2.compute.internal   Ready    <none>   3m30s   v1.22.9-eks-810597c
```

그 다음으로는 어떤 Pod, Service, DaemonSet이 돌아가고 있는지 보겠습니다. StatefulSet은 돌아가고 있지 않기 때문에 생략하겠습니다.

```shell
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-95869             1/1     Running   0          6m15s
kube-system   aws-node-spg99             1/1     Running   0          6m14s
kube-system   coredns-556f6dffc4-khv7d   1/1     Running   0          15m
kube-system   coredns-556f6dffc4-m2jr9   1/1     Running   0          15m
kube-system   kube-proxy-682pk           1/1     Running   0          6m15s
kube-system   kube-proxy-kc72t           1/1     Running   0          6m14s

$ kubectl get svc --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
default       kubernetes   ClusterIP   10.100.0.1    <none>        443/TCP         85m
kube-system   kube-dns     ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP   85m

$ kubectl get daemonsets --all-namespaces
NAMESPACE     NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   aws-node     2         2         2       2            2           <none>          85m
kube-system   kube-proxy   2         2         2       2            2           <none>          85m
```

`kube-proxy` 서비스에 대해서는 앞에서 언급했으니 생략하겠습니다. 실행 중인 다른 서비스나 Pod에 대해서는 좀 더 설명이 필요해 보이네요. 

* `coredns`: Kubernetes 클러스터 DNS로 사용할 수 있는 DNS 서버입니다. EKS 클러스터를 시작하면 노드 수에 관계 없이 CoreDNS의 복제본 2개가 기본적으로 배포됩니다. 이는 `kube-dns` 서비스로 연결됩니다. ([참고자료1](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/managing-coredns.html), [참고자료2](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-dns-failure/))
* `aws-node`: EKS 클러스터의 Pod 네트워킹을 위한 플러그인입니다. Kubernetes 노드에 VPC IP 주소를 할당하고, 각 노드의 Pod에 필수 네트워크를 구성하는 역할을 합니다. ([참고자료](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/cni-iam-role.html))

이외에도 여러 많은 리소스들이 있습니다. ConfigMap, Secret, ServiceAccount, ClusterRole, ClusterRoleBinding, Role, RoleBinding과 같은 리소스들이 있습니다. 자세한 내용은 EKS 콘솔이나 kubectl을 통해 확인해 보시기 바랍니다. 

# Fargate를 이용하는 경우는 어떤 차이가 있을까?

터미널에서 다음과 같이 입력합니다. 뒤에 `--fargate` 옵션이 들어간 것을 빼면 동일합니다. 

```shell
$ eksctl create cluster --name <your-cluster-name> --region ap-northeast-2 \
                        --vpc-private-subnets subnet-<your-subnet-id-1>,subnet-<your-subnet-id-2> \ 
                        --node-private-networking --fargate

# 역시나 컨트롤 플레인을 만드는 데 많은 시간이 소요됩니다. 
2022-07-20 19:31:42 [ℹ]  building cluster stack "eksctl-<your-cluster-name>-cluster"
2022-07-20 19:31:43 [ℹ]  deploying stack "eksctl-<your-cluster-name>-cluster"
(생략)
2022-07-20 19:42:44 [ℹ]  waiting for CloudFormation stack "eksctl-<your-cluster-name>-cluster"

# Fargate Profile을 만드는 과정도 오랜 시간이 걸립니다. 
2022-07-20 19:44:45 [ℹ]  creating Fargate profile "fp-default" on EKS cluster "<your-cluster-name>"
2022-07-20 19:49:04 [ℹ]  created Fargate profile "fp-default" on EKS cluster "<your-cluster-name>"
(생략)
2022-07-20 19:49:34 [ℹ]  "coredns" is now schedulable onto Fargate
2022-07-20 19:50:37 [ℹ]  "coredns" is now scheduled onto Fargate
2022-07-20 19:50:37 [ℹ]  "coredns" pods are now scheduled onto Fargate
2022-07-20 19:50:37 [ℹ]  waiting for the control plane availability...
2022-07-20 19:50:37 [✔]  saved kubeconfig as "/Users/<user_name>/.kube/config"
2022-07-20 19:50:37 [ℹ]  no tasks
2022-07-20 19:50:37 [✔]  all EKS cluster resources for "<your-cluster-name>" have been created
2022-07-20 19:50:38 [ℹ]  kubectl command should work with "/Users/<user_name>/.kube/config", try 'kubectl get nodes'
2022-07-20 19:50:38 [✔]  EKS cluster "<your-cluster-name>" in "ap-northeast-2" region is ready
```

역시나 컨트롤 플레인을 만드는 데 많은 시간이 소요되는데, 이후에 Fargate Profile을 만드는 과정도 시간이 걸립니다. 

관리자는 Fargate Profile을 사용해서 어떤 파드를 Fargate에서 실행할 지 선언할 수 있습니다. Profile의 selector에 네임스페이스를 정의하고, 키-값으로 레이블 필드를 정의하면 됩니다. 레이블을 정의하지 않으면 선택한 네임스페이스의 모든 파드를 Fargate로 실행합니다. 자세한 내용은 [AWS의 문서](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/fargate-profile.html)를 참고해 주세요. 

그러면 어떤 리소스들이 생성되었는지 한 번 살펴볼까요? (검정색으로 가린 부분은 클러스터 이름입니다)

{{< figure src="/img/eks-resources-with-eksctl-3.png" caption="Fargate를 사용하는 EKS 클러스터 CloudFormation 스택" >}}

다른 리소스는 모두 동일한데, **FargatePodExecutionRole**이라는 IAM Role을 볼 수 있습니다. 이 역할에는 **AmazonEKSFargatePodExecutionRolePolicy** 라는 AWS에서 관리하는 정책이 적용되어 있고, 이는 Fargate에서 EKS 파드를 실행할 때 필요한 서비스에 대한 권한을 받습니다. 실제로 살펴보면 ECR에 대한 권한이 부여되어 있습니다. 

노드 그룹은 따로 생성하지 않기 때문에, 바로 EKS 클러스터 안에 어떤 리소스들이 있는지 살펴 보겠습니다. 

```shell
$ kubectl get nodes
NAME                                                       STATUS   ROLES    AGE   VERSION
fargate-ip-172-31-55-228.ap-northeast-2.compute.internal   Ready    <none>   18m   v1.22.6-eks-14c7a48
fargate-ip-172-31-80-204.ap-northeast-2.compute.internal   Ready    <none>   18m   v1.22.6-eks-14c7a48

# 파드는 coredns만 실행 중입니다. 
$ kubectl get pods --all-namespaces 
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-6dbfd5fd98-dbbsp   1/1     Running   0          19m
kube-system   coredns-6dbfd5fd98-dn9b5   1/1     Running   0          19m

# 서비스는 EC2 노드 그룹을 사용했을 때와 동일한 것 같네요.
$ kubectl get svc --all-namespaces 
NAMESPACE     NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
default       kubernetes   ClusterIP   10.100.0.1    <none>        443/TCP         30m
kube-system   kube-dns     ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP   30m

# Daemonset은 aws-node, kube-proxy 등이 있긴 한데, 실행 중인 컨테이너는 없습니다.
$ kubectl get daemonsets --all-namespaces
NAMESPACE     NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   aws-node     0         0         0       0            0           <none>          30m
kube-system   kube-proxy   0         0         0       0            0           <none>          30m
```

* `kube-proxy`의 경우, Fargate 노드에는 배포되지 않는다고 하네요. [링크](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/managing-kube-proxy.html)를 참고해 주세요. 
* `aws-node` Daemonset 또한 Fargate 클러스터에서는 실행하지 않는데, 네트워크와 관련된 설정을 Fargate에서 담당하기 때문에 그런 것으로 보입니다. 

# 마무리

지금까지 eksctl로 EKS 클러스터를 생성할 때, 어떤 리소스들을 만드는 지 확인해 보았습니다. EC2 노드를 사용하는 것을 기준으로 요약 해 보면, 아래 그림과 같은 구조입니다.

{{< figure src="/img/eks-resources-with-eksctl-4.png" >}}

개인적으로는 ClusterSharedNodeSecurityGroup이 굳이 필요한가...? 라는 생각이 들었습니다. 이 보안 그룹이 필요한 이유가 궁금해서 찾아보긴 했는데, 명쾌하게 설명되어 있는 글을 찾기가 어렵더라구요.

글 읽어주셔서 감사드리며, 혹시나 잘못된 부분을 발견하시면 댓글 부탁드리겠습니다. 감사합니다.

# 참고자료

## Kubernetes 문서

* [쿠버네티스 컴포넌트](https://kubernetes.io/ko/docs/concepts/overview/components/)

## EKS 문서

* [Amazon EKS 클러스터](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/clusters.html)
* [Amazon EKS 클러스터와 노드 그룹의 eksctl 문제를 해결하려면 어떻게 해야 하나요?](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-troubleshoot-eksctl-cluster-node/)
* [Amazon EKS 노드](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/eks-compute.html)
* [Amazon EKS 보안 그룹 요구 사항 및 고려 사항](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/sec-group-reqs.html)
* [AWS Fargate 프로필](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/fargate-profile.html)
