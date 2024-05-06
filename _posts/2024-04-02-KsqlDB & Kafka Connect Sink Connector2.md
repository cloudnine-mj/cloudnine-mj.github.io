---
key: jekyll-text-theme
title: 'KsqlDB & Kafka Connect Sink Connector - 2'
excerpt: 'Opensearch Connect 설치/설정 😎'
tags: [KsqlDB, Kafka connect, 데이터 가공]
---

# KsqlDB & Kafka Connect Sink Connector - 2


##  KsqlDB-CLI 를 이용한 connecor 생성, 조회, data input 

* Sink connector는 kafka에서 외부로 데이터가 나가는 connector
* 아래는 KsqlDB를 이용해 connector생성, stream생성, stream에 data insert 후 생성되는 opensearch index 확인하는 것!


### connector 생성 & 확인

```
ksql> CREATE SINK CONNECTOR `kslqdb-opensearch-sink-connector` WITH (
>"connector.class" = 'io.aiven.kafka.connect.opensearch.OpensearchSinkConnector',
>"topics" = 'ksqldbConnectorTest',
>"connection.url" = 'https://192.168.2.52:30920',
>"connection.username" = 'admin',
>"connection.password" = 'admin',
>"type.name" ='connectTest',
>"tasks.max" = '1',
>"key.ignore" = 'true',
>"value.converter.schemas.enable" = 'false',
>"schema.ignore" = 'true',
>"key.converter" = 'org.apache.kafka.connect.storage.StringConverter',
>"value.converter" = 'org.apache.kafka.connect.json.JsonConverter');


ksql> show connectors;

 Connector Name                   | Type   | Class                                                     | Status                      
-----------------------------------------------------------------------------------------------------------------------
 kslqdb-opensearch-sink-connector | SINK   | io.aiven.kafka.connect.opensearch.OpensearchSinkConnector | RUNNING (1/1 tasks RUNNING) 
-----------------------------------------------------------------------------------------------------------------------
```

### Stream 생성 & 확인

```
ksql> CREATE STREAM ksqldb_opensearch_connector_test(
>test1 varchar,
>test2 varchar) WITH (
>kafka_topic = 'ksqldbConnectorTest',
>value_format='json');
 Message        
----------------
 Stream created 
----------------


ksql> show streams;
 Stream Name                      | Kafka Topic         | Key Format | Value Format | Windowed 
-----------------------------------------------------------------------------------------------
 KSQLDB_OPENSEARCH_CONNECTOR_TEST | ksqldbConnectorTest | KAFKA      | JSON         | false    
-----------------------------------------------------------------------------------------------
```

### opensearch index 생성 확인

* Steam에 data insert후 kafka consumer에서 확인 후 opensearch index 생성 확인

```
#ksqldb에 insert
ksql> INSERT INTO ksqldb_opensearch_connector_test(test1, test2) values ('test1', 'test2');
ksql> INSERT INTO ksqldb_opensearch_connector_test(test1, test2) values ('test2', 'test3');

#kafka consumer에서 massage 확인
[kafka@kafka-cluster-kafka-0 bin]$ ./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ksqldbConnectorTest  --from-beginning
{"TEST1":"test1","TEST2":"test2"}
{"TEST1":"test2","TEST2":"test3"}


#opensearch에서 index 생성 & doc 확인
GET delta-topic/_search
{
  "took": 253,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 3,
      "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "ksqldbconnectortest",
        "_id": "ksqldbConnectorTest+2+0",
        "_score": 1,
        "_source": {
          "TEST2": "test2",
          "TEST1": "test1"
        }
      },
      {
        "_index": "ksqldbconnectortest",
        "_id": "ksqldbConnectorTest+0+0",
        "_score": 1,
        "_source": {
          "TEST2": "test3",
          "TEST1": "test2"
        }
      }
    ]
  }
}
```
