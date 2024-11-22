---
title: "VPC Reachability Analyzer로 AWS 네트워크 문제 확인하기"
date: 2024-11-21T21:54:21+09:00
tags: [VPC, Reachability Analyzer]
draft: true
categories: [AWS]
comments: false
---

인프라 구성을 하다 보면, 네트워크 설정 때문에 문제를 겪는 일이 많습니다. "Connection timeout"과 같은 네트워크 관련 오류가 갑자기 발생하면, 어디서부터 원인을 찾아야 할 지 막막한데요. AWS를 사용하신다면 네트워크 문제를 겪을 때 VPC Reachability Analyzer라는 서비스를 이용하여 원인을 찾을 수 있습니다. 출발하는 지점부터 도착지까지 어떤 과정으로 트래픽이 도달하는 지 상세하게 확인할 수 있는 서비스입니다. 네트워크 문제가 발생한다면 한 번 사용해 보시기 바랍니다.

# VPC Reachability Analyzer

[VPC Reachability Analyzer](https://aws.amazon.com/ko/blogs/korea/new-vpc-insights-analyzes-reachability-and-visibility-in-vpcs/)는 2020년에 소개된 기능으로, 네트워크 구성이 의도한 대로 되었는지 확인할 수 있는 기능입니다. 

## 가능한 소스와 타겟

Source
* EC2 인스턴스
* 인터넷 게이트웨이
* 네트워크 인터페이스 (ENI)
* Transit Gateway
* Transit Gateway Attachments
* VPC Endpoints
* VPC Endpoint Services
* VPC Peering Connection
* VPN Gateways

Target
* EC2 인스턴스
* 인터넷 게이트웨이
* IP 주소
* 네트워크 인터페이스 (ENI)
* Transit Gateway
* Transit Gateway Attachments
* VPC Endpoints
* VPC Endpoint Services
* VPC Peering Connection
* VPN Gateways

# 실제로 분석해 보기

## 테스트 서버 구성


## 테스트 해 보기 


## 결과 확인 


## 실패하는 사례로 분석하면?


## 리소스 정리하기


# 고려해야 할 것들

## 비용


## VPC Peering으로 서로 다른 계정에 연결되어 있는 경우


