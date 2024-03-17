---
key: jekyll-text-theme
title: 'ELK 구축 - OpenSearch Shard, Heap '
excerpt: 'OpenSearch Research  😎'
tags: [OpenSearch, Research]
---

👉 Docs : [https://www.elastic.co/kr/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster](https://www.elastic.co/kr/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)


# OpenSearch Shard, Heap


## 크기 제한

- 세팅으로 변경 가능하고, 기본적으로 node당 1000이 default hard cap이 아니라서 변경이 가능하다.

```
cluster.max_shards_per_node : 1000
```

## Shard memory sizing

- ElasticSearch 8.3 이전 기준으로, 1G당 shard 20개 미만으로 유지를 추천. Heap이 30GB라고 가정하면 600개의 Shard가 문제가 생기지 않는 선으로 생각되기는 함.
- 각 노드당 600개 기준이므로, replica set을 적게 가져가면 6개 shard 기준으로 최소 100개의 인덱스를 만들 수 있다고 할 수 있다고 함.
- Heap이 부족하면 GC가 자주 돌아 성능 문제가 생길 수 있고, Out of memory 내부 에러가 발생해 JVM이 opensearch를 죽일 수 있다.
- jvm 모니터링 관리는 stat으로 볼 수 있다.

```
GET _nodes/stats/jvm
"jvm": {
  "timestamp": 1689744266419,
  "uptime_in_millis": 14264293,
  "mem": {
    "heap_used_in_bytes": 652706296,
    "heap_used_percent": 60,
    "heap_committed_in_bytes": 1073741824,
    "heap_max_in_bytes": 1073741824,
    "non_heap_used_in_bytes": 215941712,
    "non_heap_committed_in_bytes": 219152384,
    "pools": {
      "young": {
        "used_in_bytes": 406847488,
        "max_in_bytes": 0,
        "peak_used_in_bytes": 641728512,
        "peak_max_in_bytes": 0,
        "last_gc_stats": {
          "used_in_bytes": 0,
          "max_in_bytes": 0,
          "usage_percent": -1
        }
      },
      "old": {
        "used_in_bytes": 236421624,
        "max_in_bytes": 1073741824,
        "peak_used_in_bytes": 369102848,
        "peak_max_in_bytes": 1073741824,
        "last_gc_stats": {
          "used_in_bytes": 236421624,
          "max_in_bytes": 1073741824,
          "usage_percent": 22
        }
      },
      "survivor": {
        "used_in_bytes": 9437184,
        "max_in_bytes": 0,
        "peak_used_in_bytes": 66584576,
        "peak_max_in_bytes": 0,
        "last_gc_stats": {
          "used_in_bytes": 9437184,
          "max_in_bytes": 0,
          "usage_percent": -1
        }
      }
    }
  }
}
```

## Shard 갯수와 성능

- Shard 갯수가 성능에 영향을 끼치긴 하지만 많으면 안좋다 적으면 좋다의 기준은 없음.
- DB처럼 OLTP성으로 많은 작업을 수행할때 Shard가 많으면 메리트가 있지만 요청이 적은데 Shard가 너무 많으면 어차피 큐로 처리하므로 더 오래 걸릴 수 있다.
- 오래된 인덱스는 index shrink 작업을 통해 샤드 수를 줄일 수 있는 것 같다. 경우에 따라서 시도해볼 수는 있을 것 같다.