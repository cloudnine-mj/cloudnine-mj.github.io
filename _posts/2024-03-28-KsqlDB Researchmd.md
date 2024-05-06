---
key: jekyll-text-theme
title: 'KsqlDB Research'
excerpt: 'KsqlDB 파헤치기 😎'
tags: [KsqlDB, Research]
---

# KsqlDB Research

## 특징

* 유사 sql 구문을 통해 관계형 데이터베이스에 유사하게 접근하는 방식과 비슷하게 실시간 스트리밍 처리 가능

* Fault-torerant, scale 이 가능하도록 설계

* Kafka Connect 관리기능 제공

* 데이터 필터링, 변환, 집계, 조인, 윈도우 및 세션화를 포함, 광범위한 스트리밍 작업을 위해 여러 함수 지원

* Ksal 사용자 정의 함수(Rambda 함수도 지원)도 구현 가능하도록 지원함.


## 기본 개념

:star: 참고 자료 : [https://developer.confluent.io/courses/inside-ksqldb/stateless-operations/](https://developer.confluent.io/courses/inside-ksqldb/stateless-operations/)

### KStreams

* 순차적으로 들어오는 시퀀스

### KTable 

* 현재 상태 기반의 최신값으로 업데이트되어 보관된 데이터

### Stateless
	
* 상태 기반으로 Processing을 하는 것
* Stateless는 상태에 영향을 주지 않는 연산(ex group by, count ) stream에서 key값 외의 value에 사칙 연산 등을 하는 것은 state에 영향을 주지 않기 때문에 stateless

```
CREATE STREAM readings (
    sensor VARCHAR KEY,
    reading DOUBLE,
    location VARCHAR
) WITH (
    kafka_topic='readings',
    value_format='json',
    partitions=3
);

INSERT INTO readings (sensor, reading, location)
VALUES ('sensor-1', 45, 'wheel');  

INSERT INTO readings (sensor, reading, location)
VALUES ('sensor-2', 41, 'motor');  
```

### stateful
	
:star:  참고 자료 : [https://developer.confluent.io/courses/inside-ksqldb/stateful-operations/](https://developer.confluent.io/courses/inside-ksqldb/stateful-operations/) 

* Table이 연산이 되고 난 이후 상태 기반으로 값이 update되는 것

### join, aggregation

* stateful한 stream처리를 위해선 연관된 모든 stream이 반드시 동일한 파티션 정책을 가져가야함, 동일한 파티션 정책을 가져가려면 파티션 수 와 KEY가 반드시 동일해야 함.

* co-partition, re-partition: 기본적으로 정책을 잘못 적어도 내부에서 re-partition을 적용해 정상적으로  stateful stream처리가 가능하지만, 내부적으로 불필요한 re-partition이 발생할 수 있으므로 정책을 신경써야 함.

* re-partition이 불가능한 경우

	* KStream과 Ktable join에서 Stream은 re-partition이 가능하지만, Table은 불가능

	* KStremam과 KStream, KStream과 KTable join에서 Stream의 Source가  window기반으로 생성되는 stream이면 불가능

	* KTable의 join key column이 2개 이상인 경우 불가능

```
CREATE TABLE avg_readings AS
    SELECT sensor, 
         AVG(reading) as avg 
    FROM readings
    GROUP BY sensor
    EMIT CHANGES;
```

### Push Query

* 들어온 내용을 확인하며 아래에 업데이트 된 내용이 있다면 계속해서 밀어 넣어주며 알려주게 됨
* Emit Changes, Emit Final 구문을 마지막에 붙여줘야함, EMIT CHANGES 를 사용하면 Stream 의 변경 사항들을 연속적으로 반환하는 것이고, EMIT FINAL 은 아래에서 설명할 ‘windowed aggregation’ 에서만 사용할 수 있음. 이를 사용하면 ‘마지막’ 윈도우의 결과를 반환

### Pull Query

* 그때 당시의 데이터를 한번 조회하고 query 종료

### KsqlDB Windowing

* Streaming Process에서는 데이터가 무한하다고 가정.
* 데이터가 들어와 aggregation을 한다고 하면 Partition이 없기 때문에 aggregation을 이론상 진행 할 수 없음, 그 때 필요한것이 Windowing이며 현재 Tubling, Hopping, Session 사용 가능

### Windowed Agreggation

* tumbling : 매 n {timeUnit} 마다 ‘고정된’ 윈도우를 생성해서 윈도우 내부에 존재하는 이벤트들을 aggregation 함.

* Hopping:  고정된 윈도우 크기를 가지지만, ‘hop’ interval 기준으로 윈도우를 생성, hop interval 의 설정에 따라 윈도우 간 겹치는 이벤트가 발생할 수 있어 주의해서 설정해야 함.

* Session:  n {timeUnit} 의 고정된 윈도우 크기를 가지지만, ‘inactivity gap’ interval 기준으로 윈도우가 존재하지 않는 구간을 추가해 윈도우를 생성함.

```
CREATE TABLE VIEW_ORDER AS
SELECT o.userId, COUNT(*) FROM STREAM_ORDER o
WINDOW TUMBLING (
    SIZE 1 HOUR,
    RETENTION 1 DAYS,
    GRACE PERIOD 10 MINUTES
)
GROUP BY o.userId
```

* size : Window의 크기 및 timeUnit 설정

* RETENTION : Window의 남겨둘 시간

* Grace Period : 네트워크 상의 delay에 의해 늦게 들어온 Stream을 얼마나 기다려서 join을 해줄지 결정

### Windowed Join

* KsqlDB 는 특정 window 내에 속한 이벤트 Stream 을 join 할 수 있음. 각 이벤트를 기준으로 ‘n분 이전’ 부터 ‘m분 이후’ 까지와 같은 설정을 할 수 있음.

```
SELECT o.orderId, o.itemProducts, p.paymentId, p.isSucess
FROM STREAM_ORDER o
LEFT JOIN STREAM_PAYMENT p
WITHIN 1 HOURS [WITHIN 20 MINUTES, 40 MINUTES]
ON o.orderId = p.orderId
EMIT CHANGES;
```

* WITHIN 구문을 사용해 join 에서 사용할 윈도우의 크기를 설정할 수 있으며, 윈도우는 ‘n {timeUnit} 이전’ 부터 ‘m {timeUnit} 이후’ 형태로 생성이 됨

* n 과 m 값을 별도로 설정하지 않은 WITHIN 절을 사용한다면 ksqlDB 내부에서는 ‘n {timeUnit} 이전’ 부터 ‘n {timeUnit} 이후’ 까지 윈도우를 생성, 만약 n 과 m 값을 다르게 설정하고 싶다면 위의 SQL 구문의 대괄호 부분처럼 다른 값을 직접 설정해야 함.



## KsqlDB Architecture

:star: 참고 자료 : [https://docs.ksqldb.io/en/latest/operate-and-deploy/how-it-works/](https://docs.ksqldb.io/en/latest/operate-and-deploy/how-it-works/)