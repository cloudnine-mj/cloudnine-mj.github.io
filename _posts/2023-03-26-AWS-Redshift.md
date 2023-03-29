---
key: jekyll-text-theme
title: 'AWS Redshift'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---



# DataBase

## AWS Redshift 소개

* AWS 관리형 데이터 웨어 하우스 



## AWS Redshift 특징

- Redshift의 모든 노드들은 같은 가용영역(AZ)에 있어야 함

- PostgreSQL 기술 기반이지만 OLTP(Online Transaction Processing)에 사용되지는 않음

  > OLTP (Online Transaction Processing)
  >
  > 터미널에서 받은 메시지를 따라 호스트가 처리를 하고, 그 결과를 다시 터미널에 되돌려주는 방법

- OLAP (Online Analytical Processing)

- 다른 데이터 웨어 하우징보다 성능이 10배 좋고, 데이터가 PB 규모로 확장

- 대용량 / 복잡한 쿼리 수행할 때 적합

- 열(column) 기반 데이터 스토리지 사용 (병렬 쿼리 엔진)

- 프로비저닝한 인스턴스에 대한 비용만 지불

- 쿼리 수행 시 SQL문 사용 가능

- AWS Quicksight 같은 BI 도구나 Tableau 같은 도구도 Redshift와 통합 가능



## AWS Redshift 기능

* 백업 및 복원

* 외부 데이터 처리

* 데이터 로드

  * AWS Kinesis  Data Firehose
  * S3 using COPY command
  * EC2 Instance JDBC driver

  <img src ="https://user-images.githubusercontent.com/113915835/228483669-3a364d20-7b5f-454f-afa2-6d44b8872e98.png" width = "80%">

## AWS Redshift 스펙트럼

<img src = "https://user-images.githubusercontent.com/113915835/228484068-8ba36561-b5f9-4598-a84d-c4e6852af83e.png" width = "50%">

* AWS S3에서 AWS Redshift로 데이터를 로드하지 않아도 OK.
  *  클러스터에서 프로비저닝한 것보다 더 많은 처리 능력을 활용

<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)