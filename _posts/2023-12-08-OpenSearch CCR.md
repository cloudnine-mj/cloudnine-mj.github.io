---
key: jekyll-text-theme
title: 'ELK 구축 - OpenSearch CCR'
excerpt: ' OpenSearch cold storage research 😎'
tags: [Opensearch, ELK, Research]
---

# OpenSearch CCR

- Opensearch CCR은 Cross-Cluster Replication의 약자로 opensearch 간에 데이터를 복제하는 기능이다.
- Cluster가 아니라 Index를 복제하는 것이므로, 특정 인덱스를 선택해줘야 한다.
- 복제된 Index는 Read-only로만 운영된다.

## CCR 사용

|         | Follower      | Leader       |
| :------ | :------------ | :----------- |
| private | 10.0.0.159    | 10.0.0.197   |
| public  | 192.168.2.153 | 192.168.0.97 |

- 위의 표처럼 두 개의 오픈서치 클러스터가 있다고 가정한다(테스트에선 싱글 노드로 실행했음)
- CCR은 ‘Pull’ 형식이다. 즉, reader 클러스터는 요청에 반응하지 주도적으로 데이터를 전송하는 등의 일을 하지는 않음.
- 그렇기에 대부분의 작업은 follower 노드에서 실행하게 된다.

## Leader node 설정

- 우선 **Follower 노드에서** 리더 노드를 찾을 수 있게 설정해 줘야 한다.
- 마지막의 ccr-leader는 alias 별칭일 뿐 정해져 있는 것은 아니다. ccr-leader라는 클러스터가 해당 주소를 가지고 있다는것을 뜻함.

```
 curl -XPUT -k -H 'Content-Type: application/json' -u 'admin:admin' '<https://localhost:9200/_cluster/settings?pretty>' -d '
{
  "persistent": {
    "cluster": {
      "remote": {
        "ccr-leader": {
          "seeds": ["10.0.0.197:9300"]
        }
      }
    }
  }
}'
```

- 아무 요청도 안 해서 그런가(ㅋㅋㅋㅋㅋ) 로그에는 별 다른 문제는 없다. 

```
[2023-12-07T01:47:04,547][INFO ][o.o.c.s.ClusterSettings  ] [node-1] updating [cluster.remote.ccr-leader.seeds] from [[]] to [["10.0.0.197:9300"]]
[2023-12-07T01:47:04,663][INFO ][o.o.c.s.ClusterSettings  ] [node-1] updating [cluster.remote.ccr-leader.seeds] from [[]] to [["10.0.0.197:9300"]]
[2023-12-07T01:47:04,664][INFO ][o.o.c.s.ClusterSettings  ] [node-1] updating [cluster.remote.ccr-leader.seeds] from [[]] to [["10.0.0.197:9300"]]
[2023-12-07T01:47:04,675][INFO ][o.o.a.u.d.DestinationMigrationCoordinator] [node-1] Detected cluster change event for destination migration
```

## Index 테스트

- **리더 클러스터**에 테스트 용 인덱스를 하나 생성( ccr-testindex)

```
PUT ccr-testindex 
```

- **팔로워 클러스터**에 Replication을 생성한다.
- use_role은 replication 사용 시 권한 부여를 위한 것인데, 보안 목적이 크다. 실제로 replication은 팔로워 노드에서는 read, 리더 노드에서는 write,read 를 사용하는 것이 대전제이기 때문에 이것이 변할 일은 없다. 혹시 모를 보안 사고를 방지하기 위함이라고 보면 될 것이다.

```
PUT _plugins/_replication/follower-01/_start
{
  "leader_alias": "ccr-leader",
  "leader_index": "ccr-testindex",
  "use_roles": {
    "leader_cluster_role": "all_access",
    "follower_cluster_role": "all_access"
  }
}
```

- 제대로 실행되면, follower 노드 로그에서 해당 로그가 발생한다.

```
[2023-12-07T02:12:56,367][INFO ][o.o.r.t.s.ShardReplicationExecutor] [node-1] 
starting persistent replication task: {"leader_alias":"ccr-leader","leader_shard":"[ccr-testindex][0]","leader_index_uuid":"UOUHAV8JTyuDwnz5CjqdOw","follower_shard":"[follower-01][0]","follower_index_uuid":"ilQHVc_hSpWi2sqEULXUww"}, null, 2, {"state":"STARTED"}
```

## Replication 점검

- 복제 현황을 보려면 _status 엔드포인트 사용. ccr-followindex는 인덱스 이름이다. 인덱스를 만들면서 replication을 진행하는 거라고 보면 된다.

```
GET _plugins/_replication/ccr-followindex/_status
{
  "status": "SYNCING",
  "reason": "User initiated",
  "leader_alias": "ccr-leader",
  "leader_index": "ccr-testindex",
  "follower_index": "follower-01",
  "syncing_details": {
    "leader_checkpoint": -1,
    "follower_checkpoint": -1,
    "seq_no": 0
  }
} 
```

- 이제 리더 노드에서 데이터를 입력해본다.

```
POST ccr-testindex/_doc
{
  "The Shining": "Stephen King"
}
```

- follower 노드에서 검색(ccr-followindex)
- 중간에 follower index를 지우고 다시 만들었는데, ‘기존의 데이터’ 또한 복제한다. 차후 DR을 증설하거나 하는 상황에서도 문제 없이 작동한다고 생각하면 될 것 같다.

```
GET ccr-followindex/_search
{
  "took": 4,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 1.0,
    "hits": [
      {
        "_index": "ccr-followindex",
        "_id": "G3gRQowB6VzC8F_e5uNX",
        "_score": 1.0,
        "_source": {
          "The Shining": "Stephen King"
        }
      },
      {
        "_index": "ccr-followindex",
        "_id": "HHgbQowB6VzC8F_e5-Oe",
        "_score": 1.0,
        "_source": {
          "The Shining": "Stephen King"
        }
      }
    ]
  }
}
```

- Replication을 끄려면 stop을 주면 된다. (반드시 body가 있어야 됨.)

```
POST _plugins/_replication/ccr-followerindex/_stop { } 
```

## 결론

- CCR은 기존 클러스터 설정과는 무관하며, 복제될 추가 노드에 remote_cluster_client 옵션을 추가해서 해결한다.
- CCR을 사용 시 follower 노드의 인덱스는 read-only로 작동한다.
- replication은 인덱스 별로 구성할 수 있으며, 1:1 관계만 가능하고 1:N, N:N 설정은 불가능하다. 여러 개의 인덱스가 있다면 작업이 길어질 수 있다는 점을 고려해야 한다
