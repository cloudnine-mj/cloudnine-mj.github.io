---
key: jekyll-text-theme
title: 'Logstash input - Kafka'
excerpt: 'Logstash research 😎'
tags: [Logstash, Kafka, Research]
---

# Logstash input - Kafka


## consumer_threads

- consumer를 여러 개 만들 수 있는 옵션
- partition에 따라 쓰레드를 정해야 하며, 파티션보다 쓰레드가 많으면 낭비가 될 수 있음.

## group_id

- cosumer group은 kafka에서 한 토픽에 여러 consumer가 붙을 때 정합성이나 분배를 위해 만들어진 개념이다. 같은 group에 있는 컨슈머들은 중복되지 않게 토픽을 소비한다.
- 로그스태시를 여러 개 쓰고 싶을 때 사용하는 옵션이며 로그스태시가 여러 개 있더라도 데이터 중복이 발생하지 않고 병렬로 사용할 수 있게 됨.

## group_instance_id

- kafka 특정 버전 이후로 지원하는 group instance id 사용을 위한 것으로, group_id 등과는 다른 개념이다.
- 사용하면 rebalance 등 성능 개선이 있다고 하는데 사용하고 있진 않으므로 확인 차 작성.

## max_partition_fetch_bytes

- 한번에 가져오는 메시지의 양을 결정함. 기본은 1MB.
- 메시지 한건의 데이터가 1MB를 넘으면 병목이 발생할 수 있어 수치 조정을 하면 성능 향상을 꾀할 수 있음. 또한, 한번에 가져오는 메시지의 양이 늘어 더 효율적
- 네트워크 속도가 느리다면 요청 수를 줄일 수 있어 큰 효과를 볼 가능성이 있다.
- 한번 요청에 걸리는 시간이 길어져 다른 consumer에 영향을 끼칠 수 있다.


## 기능 관련

### client_id

- 설정하면 kafka 로그 등에 client id가 찍혀 나와 트러블슈팅에 도움이 될 수 있다.