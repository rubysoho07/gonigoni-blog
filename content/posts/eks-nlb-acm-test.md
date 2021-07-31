---
title: "EKS 내 NGINX Ingress에서 NLB와 ACM 연동 방법 살펴보기"
date: 2021-07-31T20:08:14+09:00
tags: [AWS, EKS, Container, NLB, SSL, ACM]
draft: true
categories: AWS
comments: false
---

최근 팀 내에서 외부 서비스와 연동하는 서비스가 증가하면서, 고정된 IP 주소를 요청하는 경우가 많아졌습니다. 

한편, 대외 서비스가 늘어나다 보니 관리하는 서버의 수가 증가하였습니다. 그렇지만 생각보다 트래픽이 많지 않아 리소스가 낭비되는 경우가 많은데요. 

이러한 상황을 겪으면서 서버의 수를 줄이고, 서비스마다 고정된 IP 주소를 제공할 방법을 찾아보게 되었습니다. 

그러다가 Kubernetes와 Kubernetes의 Ingress를 적절히 활용하여 이러한 요건을 충족하는 방법을 찾아보았습니다. 

이번 글은 그 과정에서 겪었던 이슈를 정리하기 위해 작성하였습니다. 

# EKS와 NGINX Ingress Controller 사용

저희 팀의 웹 서비스들은 Elastic Beanstalk를 주로 이용하고 있습니다. 그렇지만 사용량이 생각했던 것보다는 많지 않아서 컨테이너를 기반으로 한 서비스로 변경해 보려고 합니다. 

AWS에서 컨테이너 오케스트레이션을 제공하는 서비스로는 ECS와 EKS가 있습니다. 저는 ECS를 사용해 본 적은 있지만, 장기적인 관점으로는 Kubernetes에 대한 경험이 필요하다고 생각하였습니다. 

그래서 이번에는 Kubernetes, 그리고 AWS에서 제공하는 Kubernetes Service인 EKS를 사용해 보려고 합니다.

Kubernetes는 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)라는 객체를 지원합니다. Ingress는 클러스터 내 HTTP/HTTPS로 접속하는 Service에 대해 외부 연결을 관리하는 객체라고 정의합니다. 

물론 [서비스 타입을 LoadBalancer로 설정](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)해도 되긴 하지만, 클러스터 외부에서 접속할 수 있는 URL을 제공하고, SSL/TLS 접속을 제공하고, URL 규칙을 기준으로 라우팅을 제공할 수 있기 때문에 Ingress를 선택했습니다. 

중요한 점으로, Ingress를 생성할 때는 Ingress와 Ingress Controller를 모두 생성해야 합니다.

Ingress Controller로는 여러 종류가 있지만, 저희 팀에서는 nginx를 웹 서버로 사용하고 있기 때문에 [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)를 사용하었습니다. 

## Ingress Object

저는 다음과 같이 Ingress Object를 생성했습니다. `(my-domain)` 부분은 사용하는 도메인으로 적절히 변경합니다. (전체 내용은 [링크](https://github.com/rubysoho07/eks-nlb-test/blob/main/k8s/cluster_config.yaml)를 참고해 주세요)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-backend
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: (my-domain)
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: backend-service
              port: 
                number: 80
```

## Ingress Controller

NGINX Ingress Controller를 설치하는 방법으로는 [Installation Guide](https://kubernetes.github.io/ingress-nginx/deploy/) 문서를 참고하였습니다. 

기본적으로 AWS에 Ingress Controller를 배포하면 Network Load Balancer가 생성되는데요. 

저는 NLB 뿐만 아니라 HTTPS(TLS) 통신이 필요하기 때문에 `TLS Termination In AWS Load Balancer (ELB)` 항목을 참고했습니다. 

다만, 고정 IP를 제공하기 위해 NLB(Network Load Balancer)를 사용하려는 경우, 약간의 수정이 필요합니다. 

전체 과정을 정리하면 다음과 같습니다. 

* [링크](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/aws/deploy-tls-termination.yaml)로부터 `deploy-tls-termination.yaml` 파일을 받습니다.
* `proxy-real-ip-cidr`을 EKS 클러스터가 있는 VPC CIDR로 바꿉니다. (예: xxx.xxx.xxx.xxx/xx)
* `service.beta.kubernetes.io/aws-load-balancer-ssl-cert`: ACM 인증서의 ARN을 입력합니다. 
* `service.beta.kubernetes.io/aws-load-balancer-type`: `elb`로 입력되어 있는데, 수정하지 않으면 Classic Load Balancer가 생성됩니다. NLB를 사용하려면 `nlb`로 변경합니다. 
* `use-forwarded-headers`: 해당 값은 `false`로 변경해 줍니다. `true`로 지정하여도 동작은 하지만, NLB를 사용한다면 굳이 필요 없는 설정입니다. ([참조](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-forwarded-headers))

참고로, NLB에 SSL Termination을 사용하려면 Kubernetes 1.15 버전 이상을 사용해야 합니다. ([출처](https://aws.amazon.com/ko/premiumsupport/knowledge-center/terminate-https-traffic-eks-acm/))

# NLB 연동 시 하나의 Node만 Healthy인 이유는?

NLB를 올리고 Target Group 설정을 확인해 보면, Target Group의 Health Status가 하나만 Healthy로 표시되는 경우가 있습니다. 

GitHub에 올라온 [이슈](https://github.com/kubernetes/ingress-nginx/issues/5592)를 참고하면 비슷한 사례를 확인할 수 있는데요. 

이 현상이 발생한 원인은 Ingress Controller가 `externalTrafficPolicy: Local`로 설정된 것 때문이라고 합니다. 그래서 Ingress Controller가 동작 중인 노드만 `Healthy`로 표시된다고 하네요.

해당 이슈의 댓글을 보면 아시겠지만, 이러한 설정은 개발자가 의도한 것이라고 합니다.

이러한 이슈를 해결할 수 있는 여러 방법이 있는데요. 

* `externalTrafficPolicy` 옵션을 `Cluster`로 설정: `Local`로 설정하면 Client IP주소가 Pod까지 전달되지만, 트래픽이 균등하게 분배되지 않을 수도 있다고 합니다. ([참조](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip))
* DaemonSet을 사용하거나, PodAntiAffinity를 사용: DaemonSet으로 모든 노드에 Controller가 설치되도록 하거나, PodAntiAffinity 설정으로 하나의 노드에 여러 Pod을 배치하지 않도록 하는 방법

일단 저는 첫번째 방법을 사용해서 테스트 해 보니, 다음과 같이 정상적으로 동작하는 것을 확인할 수 있었습니다.

{{< figure src="/img/eks-nlb-acm-test-1.png" >}}

(참고로 기존에 `Local`로 설정되어 있던 것을 `Cluster`로 변경하면 잘 안 되서, 아예 처음부터 `Cluster`로 변경 후 배포하니 정상적으로 동작했던 경험이 있습니다)

# 마무리

우선 제 입장에서 가장 필요했던 요건을 충족할 수 있는 시스템을 구축할 수 있을 것 같아서 다행이었습니다.

- 고정된 IP 주소를 제공하면 좋겠다
- 갖고 있던 ACM 인증서를 그대로 사용할 수 있으면 좋겠다

다만, 새로운 버전의 배포나 애플리케이션 모니터링 측면에서 어떤 것들이 더 필요할 지는 고민해 봐야 할 문제인 것 같네요. 이 부분은 좀 더 확인해 보고 적절한 방법을 찾아야 하겠습니다.

전체 테스트 코드는 [GitHub 저장소](https://github.com/rubysoho07/eks-nlb-test)에서 확인하실 수 있습니다. 

테스트 코드는 개인적인 관심으로 생성한 것이며, 회사 업무와는 무관하게 생성한 것임을 알려드립니다.