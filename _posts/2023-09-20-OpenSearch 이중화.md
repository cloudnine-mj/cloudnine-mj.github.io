---
key: jekyll-text-theme
title: 'ELK 구축 - OpenSearch 이중화'
excerpt: 'OpenSearch Research  😎'
tags: [OpenSearch, Research]
---

👉 Docs : [https://opensearch.org/docs/latest/tuning-your-cluster/](https://opensearch.org/docs/latest/tuning-your-cluster/)

# OpenSearch 이중화


## Roles

* Role은 설정에 따라 한개가 될 수도 여러 개가 될 수도 있다.
* Cluster manager, data가 병용될 수도 있고 data만 담당하는 노드가 있을 수 있음.

### Cluster manager

- 일반적으로 Master node로 칭하는 그 개념
- 클러스터 상태, 구성 관리, shard 관리 등을 담당함

### Cluster manager 후보

- Cluster manager가 될 수 있는 노드를 칭함
- Fail-over 구성을 위해 미리 설정해놓는 노드

### Data node

- 실제로 데이터가 저장되어 있는 노드
- mysql의 데이터 볼륨…등을 생각하면 될 듯함

### Ingest node

- Ingest pipeline을 사용하기 위한 노드
- [https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html)
- 데이터 전처리를 해주는 기능으로 볼 수 있을 것 같다.

### Coordinating node

- Client의 요청을 분배해주는 노드
- 병목을 방지하기 위한 성능용 노드라고 보면 될 듯.

예를 들어, node가 두개라면

| node 1          | node 2 |
| :-------------- | :----- |
| cluster manager | 후보   |
| data            | data   |

와 같이 구성하면 HA-data node를 확보할 수 있을 것이고 세개라면

| node 1          | node 2 | node 3       |
| :-------------- | :----- | :----------- |
| cluster manager | 후보   | coordinating |
| data            | data   | no data      |

이런 식으로도 구성할 수 있을 것이다.

# 설정

- 해당 설정으로 진행

| node 1          | node 2 |
| :-------------- | :----- |
| cluster manager | 후보   |
| data            | data   |

| node 1        | node 2        |
| :------------ | :------------ |
| 192.168.0.155 | 192.168.0.179 |



```
# 노드 공통 설정 #

# 모든 노드 동일하게 만들어 줘야 함
cluster.name: my_cluster

# auth 관련한 설정. SSL이니 뭐니 이것저것 설정방법은 많으나 일단 끄고 진행
# 기본 설정이 false로 되어 있어 그냥 실행하면 반드시 에러를 맞닥뜨리게 된다.
plugins.security.disabled: true

# 원하는 network interface IP를 설정. IP가 두개라고 가정했을 때 원하는 IP를 바인딩하는 개념
# 아무 IP나 상관없으면 0.0.0.0 설정
network.host: 0.0.0.0

# port 관련 설정 #
# Cluster node간 통신을 위해 사용되는 port
transport.tcp.port: 5300 
# http(api 등) 통신을 위한 port
http.port: 5200

# cluster 관련 설정 #
# 모든 노드를 다 적어줘야 discovery 가능하다.
# 다른 포트를 사용할 시에는 transport.tcp.port를 뒤에 붙여주면 된다.
discovery.seed_hosts: ["192.168.0.155:5300", "192.168.0.179:5300"]

# 초기 기동시 master로 설정될 node
# 노드 동일해야 함
cluster.initial_master_nodes: ["node-1"]
```

```
#node 1
node.name: node-1

# node.roles와 node.master, data는 동일한 개념이므로 편한 대로 쓰면 됨
node.roles: [master, data]
#node.master: true
#node.data: true
```

```
#node 2
node.name: node-2
# 해당 노드는 cluster.initial_master_nodes가 node-1로 정해져 있으므로 master 후보가 된다.
node.roles: [master, data]

```

# 기동, 확인



```
bin/opensearch

-- log 
[2023-09-18T04:48:00,743][INFO ][o.o.a.u.d.DestinationMigrationCoordinator] [node-1] Detected cluster change event for destination migration
[2023-09-18T04:48:01,396][INFO ][o.o.c.r.a.AllocationService] [node-1] Cluster health status changed from [YELLOW] to [GREEN] (reason: [shards started [[.opensearch-observability][0]]])
```

- _cat/nodes 엔드포인트를 통해 현재 기동중인 노드 확인이 가능함 *이 붙어 있는 게 master 노드

```
<http://192.168.0.155:5200/_cat/nodes?v>

ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role node.roles  cluster_manager name
192.168.0.155           17          98   0    0.05    0.12     0.05 dm        data,master *               node-1
192.168.0.179           35          98   0    0.11    0.21     0.09 dm        data,master -               node-2

```

# Troubleshooting

- Cluster uuid 에러

	* uuid 관련 문제가 발생할 경우(cluster validation이 안 된다 등…), 클러스터 구성이 이미 되어 있고 data가 만들어져서 새로운 cluster 를 만들 수 없는 경우가 있다.

	* elasticsearch는 이런 문제가 덜한데 opensearch는 자동으로 사용하는 플러그인들이 메타정보를 생성할 수 있어 종종 발생할 수 있다.


```
# 에러 예시
join validation on cluster state with a different cluster uuid ORIGIN_CLUSTER_UUID than 
local cluster uuid DATA_NODES_CLUSTER_UUID, rejecting
```

* 노드의 datafile을 삭제하면 된다.

```
rm -rf opensearch/data/*
```
