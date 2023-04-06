---
key: jekyll-text-theme
title: 'AWS Aurora'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---



# DataBase

## AWS Aurora 소개

* 클라우드에 최적화된 데이터베이스 
* RDS보다 더 높은 가용성, 트래픽 처리 가능 
* AWS 고유의 기술이며 오픈 소스 아님



## AWS Aurora 특징

- **MySQL, PostgresSQL** 와 호환
- RDS의 MySQL보다 5배 높은 성능, Postgres보다 3배 높음 

- 스토리지 용량 자동 확장 
  - 10 GB 시작 → 자동으로 128 GB 까지 확장

- Read replica 15개 (rds는 5개)
- 즉각적 장애 조치
- 높은 가용성
- 비용은 RDS에 비해 20% 정도 높음
- 세 AZ에 걸쳐 무언가를 기록할 때마다 6개의 사본 저장
- 쓰기는 6개 중 4개, 읽기는 6개 중 3개만 있어도 됨
- 자가 복구 과정 있음 (수백개 볼륨 사용)



## Aurora Replicas - Auto Scaling

<img src ="https://user-images.githubusercontent.com/113915835/228489706-a698cae2-49b6-4eda-b586-f261fc20ca7a.png" width = "80%">

* Read replica에 Auto scaling 설정 가능

### Aurora – Custom Endpoints

<img src = "https://user-images.githubusercontent.com/113915835/228490029-9cf4357a-7616-468a-aa0b-2da5c9fd890f.png" width = "80%">

- 쿼리가 subset으로만 향하도록 할 수 있음
- Custom Endpoint 사용 시 Reader Endpoint 사용 안함. 

### Aurora Serverless

<img src = "https://user-images.githubusercontent.com/113915835/228490300-5cb86536-0a8f-4eaa-a5c5-f540bf210111.png" width = "60%">

- 실제 사용량에 기반한 자동 데이터베이스 인스턴스화와 자동 스케일을 가능
  - 비정기적, 간헐적, 또는 예측 불허한 워크로드에 유용
- 용량 계획을 세울 필요가 전혀 없으며 각 Aurora 인스턴스에 대해 매 초당 비용을 지불
- 하위 proxy fleet과 소통, 서버리스 방식으로 인스턴스 생성

<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)