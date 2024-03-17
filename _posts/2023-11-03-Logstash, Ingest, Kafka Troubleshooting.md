---
key: jekyll-text-theme
title: 'Logstash, Ingest, Kafka Troubleshooting'
excerpt: ' Logstash, Ingest, Kafka 자주 발생하는 문제 해결 😎'
tags: [Logstash, Ingest, Kafka, Troubleshooting]
---


# Logstash, Ingest, Kafka Troubleshooting

## **Logstash 죽었을 때**

1. ps - ef | grep 로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : logstash-8.1.3_metric으로 들어가서 bin/logstash -f beat_metric.conf 로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup bin/logstash -f beat_metric.conf 2>&1 1>/dev/null &

3. log 확인 및 데이터 정상적으로 들어오는 지 확인할 것


## **Ingest 죽었을 때**


1. ps - ef | grep 로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : logstash-8.1.3으로 들어가서 bin/logstash -f kafka.conf 로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup bin/logstash -f kafka.conf 2>&1 1>/dev/null &

3. log 확인 및 데이터 정상적으로 들어오는 지 확인할 것
 

## **Kafka 죽었을 때**  

### **Kafka 서버 죽었을 때**

1. ps - ef | grep 로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : kafka-2.12_2.6.0으로 들어가서 ./bin/kafka-server-start.sh ./config/server.properties로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup ./bin/kafka-server-start.sh ./config/server.properties 2>&1 1>/dev/null &

3. log 확인 및 kafka 정상적으로 실행되는 지 확인할 것


## **Zookeeper 서버 죽었을 때**

1. ps - ef | grep 로 pid 숫자 확인

2. 재기동 
- 포그라운드 확인 : kafka-2.12_2.6.0으로 들어가서 ./bin/zookeeper-server-start.sh ./config/zookeeper.properties로 다시 재기동 (포그라운드로 일단 되는 지 확인)
- 백그라운드 모드 실행 :  nohup ./bin/zookeeper-server-start.sh ./config/zookeeper.properties 2>&1 1>/dev/null &

3. log 확인 및 kafka 정상적으로 실행되는 지 확인할 것