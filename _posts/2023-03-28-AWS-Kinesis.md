---
key: jekyll-text-theme
title: 'AWS Kinesis'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study, Analytics] 
---



# Analytics

## AWS Kinesis 소개

* AWS의 스트리밍 데이터 서비스입니다.

* Kinesis를 활용하면 실시간 스트리밍 데이터를 손쉽게 수집하고 처리하여 분석할 수 있습니다.

  

## AWS Kinesis 서비스 특징

- **Kinesis Data Streams**: 데이터 스트림을 수집하여 처리하고 저장합니다.

- **Kinesis Data Firehose**: 데이터 스트림을 AWS 내부나 외부의 데이터 저장소로 읽어 들입니다.

- **Kinesis Data Analytics**: SQL 언어나 Apache Flink를 활용하여 데이터 스트림을 분석합니다.

- **Kinesis Video Streams**: 비디오 스트림을 수집하고 처리하여 저장합니다.

### AWS Kinesis Data Stream

<img src = "https://user-images.githubusercontent.com/113915835/228427619-db906892-0781-42b6-8e26-ff83a6978365.png" width = "80%">



- 시스템에서 큰 규모의 데이터 흐름을 다루는 서비스 

  - 실시간(Real Time)

- 여러 개의 Shard(샤드)로 구성(1~N까지 번호 매겨지며, 사전에 사용자가 프로비저닝 해야 함)

- 데이터는 모든 샤드에 분배됨

  > 샤드는 데이터 수집률이나 소비율 측면에서 스트림의 용량을 결정함

- Producers가 Kinesis data stream에 레코드 전달하고 데이터는 잠시 거기 머물면서 여러 Consumer에게 읽힘

- 보존기간은 1~365일

- 기본적으로 데이터 다시 처리와 확인 가능

- 데이터가 일단 들어오면 삭제할 수 없음(불변성)

- 데이터 스트림으로 메시지를 전송하면 파티션 키가 추가되고, 파티션 키가 같은 메시지들은 같은 샤드로 들어가게 되어 키를 기반으로 데이터 정렬 가능

- 두 가지 용량 유형

  - **프로비저닝 유형**

    - 프로비저닝할 샤드 수 정함 (수동으로)
    - 일반적인 소비 유형이나 Fan Out 방식에 적용
    - 샤드를 프로비저닝할 때 마다 시간당 비용이 부과됨.

  - **온디맨드**

    - 프로비저닝 하거나 용량을 관리할 필요 없음.
    - 시간에 따라 언제든 용량 조정 (Auto Scaling) / 기본적으로는 초당 4MB 또는 초당 4천 개의 레코드를 처리함.
    - 30일동안 관측한 최대 처리량에 기반하여 자동으로 조정. 시간당 스트림 송수신 데이터양(GB 단위)에 따라 비용 부과

```
사전에 사용량을 예측할 수 없다면 온디맨드 유형으로!
사전에 사용량을 계획할 수 있다면 프로비저닝 유형으로!
```

- 보안

  - IAM 정책 사용하여 샤드를 생성하거나 샤드에서 읽어 들이는 접근 권한을 제어할 수 있음

  - HTTPS로 전송 중 데이터를 암호화 할 수 있으며, CMS로 암호화 가능

  - Kinesis 에서 VPC 엔드포인트 사용 

    - Kinesis에 인터넷을 거치지 않고 프라이빗 서브넷의 인스턴스에서 직접 손쉽게 접근할 수 있음
  - KMS를 이용하여 저장 암호화 
  - CloudTrails를 이용하여 API 모니터링 



### Kinesis Data Firehose

<img src ="https://user-images.githubusercontent.com/113915835/228429944-2cca56b0-25d5-4254-972c-b131940c7e8b.png" width="80%">

- 스트리밍 데이터를 수집하고 S3, Amazon Redshift, Amazon Elasticsearch와 같은 AWS 서비스로 쉽게 전달할 수 있는 완전 관리형 서비스 
  - 완전 실시간은 아니지만 거의 실시간: Kinesis Data Firehose에서 수신처로 데이터를 배치로 쓰기 때문
  - 자동으로 용량의 크기가 조정되고, 서버리스이므로 관리할 서버가 없음
- Firehose를 통과하는 데이터양에 따라 비용 지불
- Producer에서 데이터 가져올 수 있음
- 주로 Kinesis Data stream에서 데이터 가져옴
- 데이터 변형 지원
- 수신처는 세 종류
  - AWS Destinations
  - 3rd-party
  - custom

### Kinesis Data Stream vs SQS FIFO

* SQS FIFO queues: 순서가 중요한 어플리케이션에 적합 

* Kinesis: 대규모 정보 실시간 처리와 분석이 필요할때 사용 적합 

### Kinesis Data Streams vs Firehose

<img src = "https://user-images.githubusercontent.com/113915835/228431407-1ab8557d-3afe-4ea9-98ff-176a2afcb8e1.png" width = "80%">

### SQS vs SNS vs Kinesis

<img src ="https://user-images.githubusercontent.com/113915835/228432222-b308a9f9-3970-4922-9b16-5e840999db38.png" width = "80%">

 

<br/>



> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)