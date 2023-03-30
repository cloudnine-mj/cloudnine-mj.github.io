---
key: jekyll-text-theme
title: 'AWS OpenSearch'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---



# Analytics

## AWS OpenSearch 소개

* AWS OpenSearch는 완전 관리형 검색 및 분석 서비스입니다.

* Amazon ElasticSearch의 후속 서비스입니다. (라이선스 문제로 서비스명 변경)

## AWS OpenSearch 특징

- 부분적으로 일치하는 필드를 포함해 모든 필드를 검색 가능
- OpenSearch를 이용하여 검색 및 쿼리 분석 가능
- OpenSearch Dashboards로 OpenSearch 데이터를 시각화할 수 있음
- OpenSearch는 애플리케이션에서 검색 기능을 제공할 때 많이 사용됨
- OpenSearch를 생성하고 사용하려면 인스턴스의 클러스터를 생성해야 함 (서버리스 X)
- 자체 쿼리 언어가 있어서 SQL 지원 안함
  - Kinesis Data Firehose AWS IoT, CloudWatch Logs나 사용자 지정 애플리케이션의 데이터를 주입할 수 있음
- Cognito, IAM와 통합하여 제공되는 보안을 통해 저장 데이터 암호화와 전송 중 암호화가 가능
- 다양한 데이터 소스 지원 (Lambda, DynamoDB, Kinesis)



## AWS OpenSearch 패턴: DynamoDB

<img src ="https://user-images.githubusercontent.com/113915835/228440677-16b2a394-c5be-4860-a861-45b3f2cb33da.png" width = "80%">

### AWS OpenSearch 패턴: CloudWatch Logs

<img src = "https://user-images.githubusercontent.com/113915835/228440833-6861ba8d-8b9f-4962-97ef-3439222f6997.png" width = "80%">

### AWS OpenSearch 패턴: Kinesis Data Streams & Kinesis Data Firehose

<img src = "https://user-images.githubusercontent.com/113915835/228441006-a3fd3c4f-6a0c-4508-bfcf-2d84d70fa11e.png" width = "80%">

## 보안

- Cognito

- IAM



<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)