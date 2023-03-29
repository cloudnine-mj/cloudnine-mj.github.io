---
key: jekyll-text-theme
title: 'AWS Athena'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---



:fire: 서버리스 SQL 엔진을 사용한 Amazon S3 데이터 분석이 나오면 Athena를 떠올리기!! (TIP)



# Analytics

## AWS Athena 소개

* Athena는 Amazon S3 버킷에 저장된 데이터 분석에 사용하는 서버리스 쿼리 서비스입니다.

  

## AWS Athena 특징

* SQL 언어를 사용하는 Presto 엔진에 빌드됩니다.

* 서버리스 SQL을 이용하여, S3에 저장된 비정형, 반정형 및 정형 데이터를 분석합니다.

  * Athena는 서버리스로 S3 버킷의 데이터를 바로 분석할 수 있습니다.

* - CSV, JSON, ORC, Avro, Parquet 지원합니다.
  - 전체 서비스가 서버리스여서 데이터베이스를 프로비저닝 할 필요가 없습니다.
  - Athena는 Amazon QuickSight 라는 도구와 함께 사용하는 일이 많습니다.

- 표준 SQL을 이용한 간편한 쿼리

- 인프라 및 클러스터 관리 필요없이 쿼리를 실행 가능

- 실행한 쿼리에 대해서 비용 지불 (스캔된 데이터의 TB당 고정 가격을 지불)

## AWS Athena

<img src = "https://user-images.githubusercontent.com/113915835/228435350-e470c7fb-488b-4bdc-901d-53fa01d08773.png" width ="30%">

###  Performance Improvement

- 데이터를 적게 스캔할 유형의 데이터 사용
  -  비용을 지불할 때 스캔된 데이터의 TB당 가격을 지불하기 때문
- Columnar data(열 기반 데이터)를 사용
  - 열 기반 데이터 유형을 사용하면 필요한 열만 스캔하므로 비용을 절감할 수 있음.
  - Apache Parquet, ORC 형식 권장 
- 데이터를 압축해 더 적게 검색
- 특정 열을 항상 쿼리할 경우, 데이터 set을 분할하기
  - S3 버킷에 있는 전체 경로를 슬래시로 분할
  - 각 슬래시에 다른 열 이름을 붙여 열별로 특정 값을 저장
- 큰 파일을 사용해서 오버헤드를 최소화
  - Athena는 큰 파일이 있을 때보다 작은 파일이 너무 많을 경우 성능이 저하됨.
  - 128 MB 이상의 파일을 사용해야 함.



### Federated Query

<img src ="https://user-images.githubusercontent.com/113915835/228436155-e9a38796-2bbd-4c03-844b-3ec518086faf.png" width = "50%">

- 관계형 데이터베이스나 비관계형 데이터베이스 객체, 사용자 지정 데이터 원본 모두 쿼리 가능합니다.
- 쿼리 결과는 사후 분석을 위해 Amazon S3 버킷에 저장할 수 있습니다.





<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)