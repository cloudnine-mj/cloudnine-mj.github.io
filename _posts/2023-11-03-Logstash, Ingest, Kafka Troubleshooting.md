---
key: jekyll-text-theme
title: 'Logstash, Ingest, Kafka Troubleshooting'
excerpt: ' Logstash, Ingest, Kafka 자주 발생하는 문제 해결 😎'
tags: [Logstash, Ingest, Kafka, Troubleshooting]
---


# Logstash, Ingest, Kafka Troubleshooting

## **Logstash 죽었을 때**

1. `ps - ef | grep logstash` 로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : logstash-8.1.3_metric으로 들어가서 bin/logstash -f beat_metric.conf 로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup bin/logstash -f beat_metric.conf 2>&1 1>/dev/null &

3. log 확인 및 데이터 정상적으로 들어오는 지 확인할 것


## **Ingest 죽었을 때**


1. `ps - ef | grep ingest` 로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : logstash-8.1.3으로 들어가서 bin/logstash -f kafka.conf 로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup bin/logstash -f kafka.conf 2>&1 1>/dev/null &

3. log 확인 및 데이터 정상적으로 들어오는 지 확인할 것


## **Kafka 죽었을 때**  

### **Kafka 서버 죽었을 때**

1. `ps - ef | grep kafka`로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : kafka-2.12_2.6.0으로 들어가서 ./bin/kafka-server-start.sh ./config/server.properties로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup ./bin/kafka-server-start.sh ./config/server.properties 2>&1 1>/dev/null &

3. log 확인 및 kafka 정상적으로 실행되는 지 확인할 것


### **Zookeeper 서버 죽었을 때**

1. `ps - ef | grep zookeeper`로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : kafka-2.12_2.6.0으로 들어가서 ./bin/zookeeper-server-start.sh ./config/zookeeper.properties로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup ./bin/zookeeper-server-start.sh ./config/zookeeper.properties 2>&1 1>/dev/null &

3. log 확인 및 kafka 정상적으로 실행되는 지 확인할 것

### GCP 인스턴스 문제

* 문제 상황
	* GCP에 구축한 사내 파이프라인  Kafka에서 아래와 같은 [에러 로그] 가 발생. 사내 파이프라인에 구축된 Kafka 서버 3대에 들어가서 Kafka 서버 관련 로그 확인하려고 했으나 3대 중 1대에서 아래와 같이 connection 에러 발생 
	* 일단, 문제의 서버가 ping은 찍히는데 연결이 안되는 것 같아서 GCP 인스턴스 문제로 추정 

* 문제 해결
	* GCP 인스턴스를 재기동하고 Kafka 서버, Zookeeper 다시 껐다가 재기동하여 정상적으로 작동하는지 확인 및 log 점검

* 에러 로그

```
[2023-11-03T01:25:58,742][WARN][org.apache.kafka.clients.NetworkClient][main] [Producer clientId=producer-1] 1 partitions have leader brokers without a matching listener, including [****-******-topic-0]
```

```
[2023-11-03T07:12:34,349][WARN ][org.apache.kafka.clients.producer.internals.Sender][main] [Producer clientId=producer-1] Received invalid metadata error in produce request on partition tuba-meta-topic-0 due to org.apache.kafka.common.errors.NotLeaderForPartitionException: This server is not the leader for that topic-partition.. Going to request metadata update now
[2023-11-03T07:12:34,453][WARN ][org.apache.kafka.clients.producer.internals.Sender][main] [Producer clientId=producer-1] Got error produce response with correlation id 27970 on topic-partition tuba-meta-topic-0, retrying (2147483639 attempts left). Error: NOT_LEADER_FOR_PARTITION
[2023-11-03T07:12:34,453][WARN ][org.apache.kafka.clients.producer.internals.Sender][main] [Producer clientId=producer-1] Received invalid metadata error in produce request on partition tuba-meta-topic-0 due to org.apache.kafka.common.errors.NotLeaderForPartitionException: This server is not the leader for that topic-partition.. Going to request metadata update now
[2023-11-03T07:12:34,549][WARN ][org.apache.kafka.clients.NetworkClient][main] [Producer clientId=producer-1] 1 partitions have leader brokers without a matching listener, including [tuba-meta-topic-0]
```