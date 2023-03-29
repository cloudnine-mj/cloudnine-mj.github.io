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

* IAM 정책에 따라 보안 강화 가능

* ECS Task Auto Scaling 가능

### EC2 Launch Type

<img src = "https://user-images.githubusercontent.com/113915835/228443511-ff452282-2669-49d6-bae4-e3696ebf9c55.png" width = "70%">

* EC2 Launch Type으로 EC2 클러스터를 사용할 때는 사용자가 인프라를 직접 프로비저닝, 유지해야 함.

- ECS 인스턴스는 각각 ECS 에이전트(Agent)를 실행해야 함.
  - ECS 에이전트가 각각의 EC2 인스턴스를 Amazon ECS 서비스와 지정된 ECS 클러스터에 등록
- 도커 컨테이너는 미리 프로비저닝한 ec2 인스턴스에 위치





<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)