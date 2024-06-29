---
key: jekyll-text-theme
title: 'Kafka - Topic, Consumer, Consumer Group'
excerpt: 'Kafka Research😎'
tags: [Kafka]
---

# Kafka - Topic, Consumer, Consumer Group



## Topic

- 데이터를 최종적으로 저장하는 곳인데, 데이터를 구분하기 위한 저장소
- 토픽은 데이터베이스 테이블이나 파일시스템의 폴더와 유사한 성질을 가지고 있다.
- 토픽에 프로듀서는 데이터를 넣고, 컨슈머가 데이터를 가져간다.
- 토픽은 목적에 따라 각각의 이름을 가질 수 있는데, 어떤 데이터를 담는지에 따라 명확하게 토픽명을 만드는 것이 좋다.

### Multi Sub

- Consumer(client와 비슷함)를 만들고 특정 토픽에 요청하면 Consumer에서 데이터를 전달함.
- 1~10의 데이터가 있다고 치면 일반적인 큐는 한번만 요청할 수 있으나, 여러 Consumer에서 요청하면 같은 데이터를 여러번 Sub 가능함.

### Partition

- 토픽은 여러 개의 파티션으로 구성
- parallel로 메시지 처리를 할 수 있음
- 분산 처리와 저장이 가능
- 파티션은 저장소 안에 분리된 공간으로 데이터를 더 빨리, 더 많이 보내고 처리하기 위해 만들어진 것.
- 파티션 번호는 0부터 시작하며, 각 파티션 마다 고유한 offset을 가지고 있음.
<br>
> **offset**
> consumer에서 메시지를 어디까지 읽었는지 저장하는 값
> offset=3 인 경우 offset 0, 1, 2은 메시지를 읽은 것으로 생각할 수 있다.

### Data Retention

- 메시지 유효 기간을 두고 기한이 지나면 관리 가능한 기능을 제공
- Sub과 동시에 삭제하는 기능 또한 가능

## Consumer

- Topic이 RDB의 테이블이라면 Consumer는 Client
- Consumer는 일종의 역할을 뜻하며, 실제로 일을 하는 것은 Consumer Instance. Consumer Instance가 생성되어 Consumer 역할을 수행한다고 이해하면 된다.
- Consumer가 생성되어 Topic에서 데이터를 읽을 때 Offset이 생성되며, 이것 또한 topic으로 상태가 관리됨. ( offset topic이 replication이 되어야 하는 이유로 직결됨 )

## Consumer Group

- Consumer Group은, 병렬 처리를 위한 개념이다.
- Consumer Group - Consumer Instance는 1:N의 관계를 가진다
- 토픽의 데이터를 순서를 보장하며 Sub할 수 있게 해주며, 여러 ap가 붙어 처리하면서 데이터 중복이나 유실을 방지하기 위한 것
- Consumer Instance가 4개이고 Topic의 데이터가 1~100이라고 했을때, 이렇게 된다. Instance 1: 1~25 Instance 2: 26~50 …
- 물론 실제로 1~25까지 들어가는 것은 아니고 4개의 Instance를 취합하면 1~100의 데이터가 모두 들어가 있게 되는 셈.

### Multi-Topic

- 하나의 Consumer에서 여러 개의 토픽에 접근이 가능하다.
- (다른 개념이지만) Statement-Transaction의 관계와 유사하다고 생각함
- 여러 개의 Consumer를 핸들링 하는것보다 하나의 Consumer를 핸들링해서 여러 토픽에 접근하는 것이므로 사용의 용이성을 가질 수 있다
- 여러 개의 Consumer Instance를 유지해도 사용에 문제는 없으며 Transaction처럼 민감한 개념은 아니고 편의성의 차이라고 보면 될 것 같다.
