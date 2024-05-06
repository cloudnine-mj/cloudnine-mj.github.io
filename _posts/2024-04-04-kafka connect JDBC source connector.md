---
key: jekyll-text-theme
title: 'Kafka connect JDBC source connector'
excerpt: 'JDBC Connect 설치/설정 😎'
tags: [KsqlDB, JDBC connect, 데이터 가공]
---

# Kafka connect JDBC source connector

:star: JDBC source connector는 JDBC기반 DB에서 kafka로 메시지를 보낼 때 사용된다.

* Kafka connect 실행 -> Database, table 실행 -> Kafka Connector 실행


## kafka-connect.yml 수정

### kafka-connect.yml

```
#jar가 위치할 디렉토리 확인! 아래는 path의 예를 돕기위해 적어놓은 부분, 
#자신의 설정에 맞게 따로 적어야함
    - name: CONNECT_PLUGIN_PATH
      value: /usr/share/java/libs,/etc/kafka/connect,/usr/share/java/libs/confluentinc-kafka-connect-jdbc-10.7.4, 
             /usr/share/java/libs/opensearch-connector-for-apache-kafka-3.1.1
```

* connector는 해당 DB에 table이 있어야지만 생성이 가능함.

### 필요한 플러그인, connector 설치

* confluent JDBC connector 10.7.4  

* mysql-connector-java-8.0.28  

* mariadb-java-client-2.7.4.jar   

* 받은 플러그인을 class path안에 전부 넣어준다.

* source connector 생성 

```
#connector의 주소
http://192.168.2.52:30099/connectors

{
    "name": "mariadbSourceTest", #커넥터의 이름
    "config": {
        #사용할 플러그인, source, sink 설정 예제에서는 source
        "connector.class" : "io.confluent.connect.jdbc.JdbcSourceConnector", 
        
        #사용할 JDBC DB종류, 도메인, database 설정
        "connection.url" : "jdbc:mariadb://mariadb:3306/connectTest",
        
        #계정 정보
        "connection.user" : "root",
        "connection.password" : "secret",
        
        #토픽이 생성될 시 앞에 붙을 스트링 
        "topic.prefix" : "mariadb-connector-",
        
        #실행하고 있는 동안 커넥터는 jdbc를 통해 rdb를 폴링한다. 
        #변경이 있으면 카프카에 전달하고 변경 감지는 incrementing 방법으로 진행한다. 
        #mode는 incrementing외에도 bulk, timestamp 등이 있다. 
        #incrementing.column.name 을 통해 변경을 감지한다.
        "mode" : "incrementing",
        "incrementing.column.name" : "id",
        "poll.interval.ms" : 1000,
        
        #load 할 대상 테이블
        "table.whitelist": "sourcetest",
        
        #생성될 토픽의 레플리카, 파티션 설정
        "topic.creation.default.replication.factor" : 2,
        "topic.creation.default.partitions" : 3
        
        #인풋될 데이터 converter
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "true",
        "key.converter.schemas.enable": "true",
        
        #최대 task 개수
        "max.tasks": 1
    }
}


#생성 완료 확인
http://192.168.2.52:30099/connectors/mariadbSourceTest/status
{
    "name": "mariadbSourceTest",
    "connector": {
        "state": "RUNNING",
        "worker_id": "10.244.4.222:8083"
    },
    "tasks": [
        {
            "id": 0,
            "state": "RUNNING",
            "worker_id": "10.244.4.222:8083"
        }
    ],
    "type": "source"
}
```

## source connect 연결 확인

* Database 에서 데이터 insert

* topic 생성 확인

* 토픽 consumer 확인

```
#connector 확인
[kafka@kafka-cluster-kafka-0 bin]$ ./kafka-topics.sh --list --bootstrap-server localhost:9092
__consumer_offsets
__strimzi-topic-operator-kstreams-topic-store-changelog
__strimzi_store_topic
_confluent-ksql-default__command_topic
_schemas
connect-configs
kafka-connect-offsets
kafka-connect-status
#prefix 대로 topic 생성
mariadb-connector-sourcetest
testOpensearchTopic

#데이터 insert 후 cosumer 확인
[kafka@kafka-cluster-kafka-0 bin]$ ./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic mariadb-connector-sourcetest -from-beginning
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":true,"field":"user_id"},{"type":"string","optional":true,"field":"pwd"},{"type":"string","optional":true,"field":"name"}],"optional":false,"name":"users"},"payload":{"id":9,"user_id":"test12123","pwd":"test2","name":"tes411322"}}
```