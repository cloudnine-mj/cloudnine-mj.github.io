---
key: jekyll-text-theme
title: 'AWS ECS & Fargate'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---


# Containers

## AWS ECS

* **ECS = Elastic Container Service**
* Docker 컨테이너를 쉽게 배포 및 관리할 수 있는 오케스트레이션 서비스 

## AWS ECS 특징

* 두 가지의 Launch Type이 있음

  * EC2 Launch Type
  * Fargate Launch Type

### EC2 Launch Type

<img src = "https://user-images.githubusercontent.com/113915835/228443511-ff452282-2669-49d6-bae4-e3696ebf9c55.png" width = "70%">

* EC2 Launch Type으로 EC2 클러스터를 사용할 때는 사용자가 인프라를 직접 프로비저닝, 유지해야 함.

- ECS 인스턴스는 각각 ECS 에이전트(Agent)를 실행해야 함.
  - ECS 에이전트가 각각의 EC2 인스턴스를 Amazon ECS 서비스와 지정된 ECS 클러스터에 등록
- 도커 컨테이너는 미리 프로비저닝한 ec2 인스턴스에 위치



### Fargate Launch Type

<img src = "https://user-images.githubusercontent.com/113915835/228445041-a4a182fa-997d-4443-bbc2-d7e3dd27617f.png" width="70%">

* Fargate 유형은 ECS 클러스터가 있을 때, ECS 태스크 정의만 생성하면 필요한 CPU나 RAM에 따라 ECS 태스크를 AWS가 대신 실행
  * 새 도커 컨테이너를 실행하면 어디서 실행되는지 알리지 않고 그냥 실행됨
  * 작업을 위해 백엔드에 EC2 인스턴스가 생성될 필요 없음

> **참고**
>
>  AWS에 도커 컨테이너를 실행하는데인프라를 프로비저닝하지 않아 관리할 EC2 인스턴스가 없음(서버리스)
>
> but, 서버를 관리하지 않아 서버리스라 부르는데 서버가 없는 건 아님





<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)