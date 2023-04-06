---
key: jekyll-text-theme
title: 'AWS ECR & EKS'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study, Containers] 
---



# Containers

## AWS ECR 소개

* **ECR = Elastic Container Registry**

* Docker 이미지를 저장하고 관리하기 위한 완전 관리형 Docker 이미지 저장소

## AWS ECR 특징
* Amazon ECS와 완전히 통합되어 있고, 이미지는 백그라운드에서 Amazon S3에 저장
* 이미지 취약점 스캐닝, 버저닝 태크, 수명 주기(이미지 lifecycle) 관리 기능 제공
* Public, Private repository 존재
* IAM 정책을 통해 컨트롤 가능

### AWS ECR

<img src ="https://user-images.githubusercontent.com/113915835/228447822-8b78fd42-63e1-4a34-8f8c-dbace96c3a31.png" width = "30%">

## AWS EKS 소개

* **EKS = Elastic K8s Service**

* AWS에 관리형 Kubernetes 클러스터를 실행할 수 있는 서비스

  > Kubernetes는 오픈 소스 시스템으로 Docker로 컨테이너화한 애플리케이션의 자동 배포, 확장, 관리를 지원함

## AWS EKS 특징

- 컨테이너를 실행한다는 목적은 ECS와 비슷하나 사용하는 API가 다름 (ECS는 오픈소스가 아님)
- EKS 두 가지 실행 모드가 존재
  - EC2 모드:  EC2 인스턴스에서처럼 작업자 모드를 배포할 때 사용
  - Fargate 모드:  EKS 클러스터에 서버리스 컨테이너를 배포할 때 사용
- ECS와 유사하게 EKS 서비스나 Kubernetes 서비스를 노출할 때는 프라이빗 로드 밸런서나 퍼블릭 로드 밸런서를 설정해 웹에 연결해야 함

### AWS EKS

<img src = "https://user-images.githubusercontent.com/113915835/228448413-44b3ab45-6990-404f-b96e-3efa76dcaa93.png" width = "80%">

### 노드 Types

- **관리형 노드 그룹**: EKS에서 관리하는 노드
  - AWS로 노드, 즉 EC2 인스턴스를 생성하고 관리
- **Self-managed Node**: 직접 EKS에 등록한 노드
  - 사용자 지정 사항이 많고 제어 대상이 많은 경우 사용자가 직접 노드를 생성하고 EKS 클러스터에 등록한 다음, ASG의 일부로 관리해야 함.
  - 사전 빌드된 AMI인 Amazon EKS 최적화 AMI를 사용하면 시간을 절약할 수 있음.
- **AWS fargate**: 노드 없이 EKS위에 컨테이너 띄움, 유지 및 관리 필요 없음 (노드를 원하지 않는 경우)

## AWS EKS 사용 사례

* 회사가 온프레미스나 클라우드에서 Kubernetes나 Kubernetes API를 사용중일 때 Kubernetes 클러스터를 관리하기 위해 Amazon EKS를 사용
* Kubernetes는 클라우드 애그노스틱으로 Azure, Google Cloud 등 모든 클라우드에서 지원됨

> **애그노스틱 기술(agnostic technology)**
>
> ‘애그노스틱 (Agnostic)’은 '구애받지 않는' 이라는 사전적 의미를 가지고 있음. 천 가지가 넘는 변수에 구애받지 않으며 클라우드의 복잡한 인프라 특성을 해결할 수 있는 기술
>
> 참조: [https://blog.naver.com/n_cloudplatform/222988261555](https://blog.naver.com/n_cloudplatform/222988261555)

<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)
