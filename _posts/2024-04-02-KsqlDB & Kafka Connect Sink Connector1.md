---
key: jekyll-text-theme
title: 'KsqlDB & Kafka Connect Sink Connector - 1'
excerpt: 'Opensearch Connect 설치/설정 😎'
tags: [KsqlDB, Kafka connect, 데이터 가공]
---

# KsqlDB & Kafka Connect Sink Connector - 1

* 현재 confluent에서는 opensearch connector를 지원하지않아, aiven의 opensearch connector 사용

* 현재 최신버전 기준 repository [https://github.com/Aiven-Open/opensearch-connector-for-apache-kafka/releases/tag/v3.1.1](https://github.com/Aiven-Open/opensearch-connector-for-apache-kafka/releases/tag/v3.1.1) Kafka Connect 가 설치되어 있어야 함.


* 참고 문서 : [https://aiven.io/docs/products/kafka/kafka-connect/howto/opensearch-sink](https://aiven.io/docs/products/kafka/kafka-connect/howto/opensearch-sink)


## jar 파일이 들어갈 pv, pvc 생성

* KsqlDB Connect Deploy 때 설치 했다면 생략해도 OK

```
pv-plugins.yml

##################################################################################
nfs 서버 설정
##################################################################################
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kafka-connect-dev-pv2
  labels:
    app: kafka-connect
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 10Gi
  nfs:
    server: 192.168.2.52 #10.0.0.199 #TODO: update with correct NFS server
    path: /root/km/ksqldb/connect-plugins
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-connect-dev-config-pvc2
  namespace: dataplatform
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage

##################################################################################
local storage 설정
##################################################################################
apiVersion: v1
kind: PersistentVolume
metadata:
  name: kafka-connect-plugins
  labels:
    app: kafka-connect
spec:
  storageClassName: local-redis-storage
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /root/kafka-connect-libs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-connect-plugins-config-pvc
  namespace: datatest
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-redis-storage     
```

## ksqldb-server.yml 수정

```
apiVersion: v1
kind: Pod
metadata:
  name: ksqldb-server-pod
  labels:
    app: ksqldb-server
spec:
  containers:
  - name: ksqldb-server
    image: confluentinc/ksqldb-server:latest
    ports:
    - containerPort: 8088
    env:
      - name: KSQL_BOOTSTRAP_SERVERS
        value: "kafka-cluster-kafka-bootstrap.datatest.svc.cluster.local:9092"
      #KSQLDB에서 connector를 관리하기 위해 추가
      - name: KSQL_KSQL_CONNECT_URL
        value: "http://10.111.72.75:8083"
```

## ksqldb-cli에서 명령어 확인

### Connector 생성

```
#ksqldb 접속
k exec -it ksqsldb-cli-pod bash
ksql ksql http://10.98.247.47:8088

#ksqldb에서 생성
CREATE SINK CONNECTOR test_opensearch_connector WITH(
"connector.class" = 'io.aiven.kafka.connect.opensearch.OpensearchSinkConnector',
"name" = 'testconnector',
"topics" = 'testOpensearchTopic',
"connection.url" = 'https://10.111.30.245:9200',
"connection.username" = 'admin',
"connection.password" = 'admin',
"type.name" = 'test',
"task.max" = '1',
"key.converter" = 'org.apache.kafka.connect.storage.StringConverter',
"value.converter.schemas.enable" = 'false',
"value.converter"= 'org.apache.kafka.connect.json.JsonConverter',
"schema.ignore" = 'true',
"key.ignore" = 'true'
);


#postman에서 생성
POST http://192.168.2.52:30099/connectors/testconnector/status
{
    "name": "testconnector",
    "config":{
        "connector.class": "io.aiven.kafka.connect.opensearch.OpensearchSinkConnector",
        "topics": "testOpensearchTopic",
        "connection.url": "https://192.168.2.52:30920",
        "connection.username": "admin",
        "connection.password": "admin",
        "type.name": "connectortestindex",
        "tasks.max":"1",
        "key.ignore": "true",
        "value.converter.schemas.enable": "false",
        "schema.ignore": "true",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    }
}
```

## Connector 조회

```
show connectors;
```

## 데이터 입력 확인

```
#kafka pod 접속
k exec -it kafka-cluster-kafka-0 bash
./bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic testOpensearchTopic
> {"test": "04"}
>{"test": "test02"}
>{"test": "test12345"}
```

* opensearch에서 topic 과 같은 이름의 인덱스가 생성되는 것을 확인할 수 있음.