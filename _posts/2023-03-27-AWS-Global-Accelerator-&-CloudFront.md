---
key: jekyll-text-theme
title: 'AWS Global Accelerator & CloudFront'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study, Networking] 
---



# Networking



## AWS Global Accelerator 소개

* AWS Global Accelerator는 퍼블릭 애플리케이션의 가용성, 성능 및 보안을 개선하는 데 유용한 네트워킹 서비스
* Global Accelerator는 애플리케이션 엔드포인트로의 고정 진입점 역할을 하는 두 개의 글로벌 정적 공용 IP(예: Application Load Balancer, Network Load Balancer, Amazon Elastic Compute Cloud(EC2) 인스턴스 및 탄력적 IP)를 제공



## AWS Global Accelerator 특징

- 애니캐스트 IP -> 모든 서버가 동일한 IP 주소를 가지며 클라이언트는 가장 가까운 서버로 라우팅
- 사용자와 가장 가까운 엣지 로케이션으로 트래픽 직접 전송하여 ALB에 도달



## AWS Global Accelerator

<img src ="https://user-images.githubusercontent.com/113915835/228460185-08760725-e7c5-428c-85a7-1becb7523be5.png" width = "80%">

## AWS CloudFront 소개

* **Cloud Front = Content Delivery Network**
* 사용자로부터 가까운 엣지 로케이션(전송 지점; 216개) 을 사용하여 빠르고 안정적인 콘텐츠 전송하는 서비스

## AWS CloudFront 특징

- 웹사이트의 컨텐츠를 서로 다른 엣지 로케이션에 미리 캐싱하여 읽기 성능을 높이는 것
- DDos 공격에서 보호받을 수 있음

## AWS CloudFront

<img src = "https://user-images.githubusercontent.com/113915835/228462833-85b946c2-979c-46c4-aa19-8e8abcf800a5.png" width = "80%">



<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)