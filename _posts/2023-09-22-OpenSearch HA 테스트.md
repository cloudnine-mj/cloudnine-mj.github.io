---
key: jekyll-text-theme
title: 'ELK 구축 - OpenSearch HA 테스트'
excerpt: 'OpenSearch Research  😎'
tags: [OpenSearch, Research]
---

# OpenSearch HA 테스트

## 2 nodes - all master,data로 설정

### setting

- node 1 - master, data

```
cluster.name: mj-cluster

node.name: es-1
node.master: true
node.data: true
node.ingest: false
node.ml: false
xpack.ml.enabled: true
cluster.remote.connect: false

network.host: 0.0.0.0

discovery.seed_hosts: ["192.168.0.155", "192.168.0.179"]

cluster.initial_master_nodes: ["es-1", "es-2"]
```

- node 2 - master, data

```
cluster.name: mj-cluster

node.name: es-2
node.master: true
node.data: true
node.ingest: false
node.ml: false
xpack.ml.enabled: true
cluster.remote.connect: false

network.host: 0.0.0.0

discovery.seed_hosts: ["192.168.0.155", "192.168.0.179"]

cluster.initial_master_nodes: ["es-1", "es-2"]
```

### node 1이 master node일 때

- es-1이 마스터

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.0.155           28          98   1    0.24    0.25     0.20 md        *      es-1
192.168.0.179           54          76   0    0.02    0.14     0.14 md        -      es-2
```

- es-1을 kill하면 retry하면서 에러 발생

```
[2023-09-20T05:04:46,158][WARN ][o.e.c.c.ClusterFormationFailureHelper] [es-2] master not discovered or elected yet, an election requires a node with id [W2D0ORUUSIG0aoI1a5QiPA], have discovered [] which is not a quorum; discovery will continue using [192.168.0.155:9300] from hosts providers and [{es-1}{W2D0ORUUSIG0aoI1a5QiPA}{-Z7SysI0SVahevMiUdfvmw}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true}, {es-2}{5uJvarD8SYiJDfBIW_s2NQ}{xjMGvM5yQmy6mnYmTapEYw}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true}] from last-known cluster state; node term 3, last-accepted version 20 in term 3
```

- es-1을 다시 올리고 es-2를 kill es-1이 master이므로 그냥 노드만 제외시킨다.

```
[2023-09-20T05:05:33,772][INFO ][o.e.c.s.MasterService] [es-1] node-left[{es-2}{5uJvarD8SYiJDfBIW_s2NQ}{xjMGvM5yQmy6mnYmTapEYw}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true} disconnected], term: 4, version: 25, reason: removed {{es-2}{5uJvarD8SYiJDfBIW_s2NQ}{xjMGvM5yQmy6mnYmTapEYw}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true},}
[2023-09-20T05:05:33,784][INFO ][o.e.c.s.ClusterApplierService] [es-1] removed {{es-2}{5uJvarD8SYiJDfBIW_s2NQ}{xjMGvM5yQmy6mnYmTapEYw}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true},}, term: 4, version: 25, reason: Publication{term=4, version=25}
```

### node 2가 master node

- es-2가 master

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.0.179           31          76   4    0.21    0.20     0.16 md        *      es-2
192.168.0.155            8          98   6    0.47    0.28     0.21 md        -      es-1
```

- es-2를 kill. es-1에서 찾기 시작한다.

```
[2023-09-20T05:09:35,186][WARN ][o.e.c.c.ClusterFormationFailureHelper] [es-1] master not discovered or elected yet, an election requires a node with id [iZzhWlE6R7yPMo1Z3kwHfg], have discovered [] which is not a quorum; discovery will continue using [192.168.0.179:9300] from hosts providers and [{es-2}{iZzhWlE6R7yPMo1Z3kwHfg}{bsg7KdWTSr2NQyOEv7E5Rg}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true}, {es-1}{cAWaSbe9Rsi1GxX3qTeLyQ}{bK_t9TChT0SDPoBcrdo1WQ}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true}] from last-known cluster state; node term 2, last-accepted version 16 in term 2
```

- es-1을 kill하면 그냥 노드 제외시키고 작동

```
[2023-09-20T05:10:04,053][INFO ][o.e.c.s.MasterService] [es-2] node-left[{es-1}{cAWaSbe9Rsi1GxX3qTeLyQ}{bK_t9TChT0SDPoBcrdo1WQ}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true} disconnected], term: 3, version: 21, reason: removed {{es-1}{cAWaSbe9Rsi1GxX3qTeLyQ}{bK_t9TChT0SDPoBcrdo1WQ}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true},}
[2023-09-20T05:10:04,073][INFO ][o.e.c.s.ClusterApplierService] [es-2] removed {{es-1}{cAWaSbe9Rsi1GxX3qTeLyQ}{bK_t9TChT0SDPoBcrdo1WQ}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true},}, term: 3, version: 21, reason: Publication{term=3, version=21}
```

## 2 nodes - (master,data) , (master)

- 한쪽만 데이터를 가지고 있는 경우 테스트

### setting

- node 1 : data가 없음

```
cluster.name: mj-cluster

node.name: es-1
node.master: true
node.data: false
node.ingest: false
node.ml: false
xpack.ml.enabled: true
cluster.remote.connect: false

network.host: 0.0.0.0

discovery.seed_hosts: ["192.168.0.155", "192.168.0.179"]

cluster.initial_master_nodes: ["es-1", "es-2"]

```

- node 2

```
cluster.name: mj-cluster

node.name: es-2
node.master: true
node.data: true
node.ingest: false
node.ml: false
xpack.ml.enabled: true
cluster.remote.connect: false

network.host: 0.0.0.0

discovery.seed_hosts: ["192.168.0.155", "192.168.0.179"]

cluster.initial_master_nodes: ["es-1", "es-2"]
```

### node 1이 master일 때

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.0.179           38          76  14    0.37    0.17     0.14 md        -      es-2
192.168.0.155           34          98  14    0.33    0.21     0.18 m         *      es-1
```

- es-1을 kill, es-2에서 찾기 시작

```
[2023-09-20T05:15:47,513][WARN ][o.e.c.c.ClusterFormationFailureHelper] [es-2] master not discovered or elected yet, an election requires a node with id [V8X3haeaSiOku3M7tglYFg], have discovered [] which is not a quorum; discovery will continue using [192.168.0.155:9300] from hosts providers and [{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true}, {es-1}{V8X3haeaSiOku3M7tglYFg}{w7AK-K5BT_akxizoz1cxqg}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true}] from last-known cluster state; node term 1, last-accepted version 16 in term 1

```

- es-2를 kill, es-1에서는 노드 제외

```
[2023-09-20T05:16:14,332][INFO ][o.e.c.s.MasterService    ] [es-1] node-left[{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true} disconnected], term: 2, version: 21, reason: removed {{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true},} [2023-09-20T05:16:14,345][INFO ][o.e.c.s.ClusterApplierService] [es-1] removed {{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true},}, term: 2, version: 21, reason: Publication{term=2, version=21} [2023-07-06T05:16:14,332][INFO ][o.e.c.s.MasterService    ] [es-1] node-left[{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true} disconnected], term: 2, version: 21, reason: removed {{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true},}
[2023-09-20T05:16:14,345][INFO ][o.e.c.s.ClusterApplierService] [es-1] removed {{es-2}{Y-XNDs_JR3mUMbehCKsaoQ}{NUe6u_rjTp-SB00CIVBEqA}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true},}, term: 2, version: 21, reason: Publication{term=2, version=21}
```

### node 2가 master일 때

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.0.179           37          76   3    0.15    0.16     0.14 md        *      es-2
192.168.0.155           33          98  19    0.19    0.17     0.17 m         -      es-1
```

- es-2를 kill, es-1에서 찾음

```
[2023-09-20T05:19:08,007][WARN ][o.e.c.c.ClusterFormationFailureHelper] [es-1] master not discovered or elected yet, an election requires a node with id [tRCimpeHQOmchi_1GfpT-w], have discovered [] which is not a quorum; discovery will continue using [192.168.0.179:9300] from hosts providers and [{es-2}{tRCimpeHQOmchi_1GfpT-w}{WOloom5ZRO-Qa3HmGwLLjQ}{192.168.0.179}{192.168.0.179:9300}{xpack.installed=true}, {es-1}{CyNVz5OOTQaa6KFanQDfXw}{CJxW-K62Q4KQboZ0f1sL2A}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true}] from last-known cluster state; node term 2, last-accepted version 16 in term 2
```

- es-1을 kill, es-1에서 제외

```
[2023-09-20T05:19:43,066][INFO ][o.e.c.s.MasterService    ] [es-2] node-left[{es-1}{CyNVz5OOTQaa6KFanQDfXw}{CJxW-K62Q4KQboZ0f1sL2A}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true} disconnected], term: 3, version: 21, reason: removed {{es-1}{CyNVz5OOTQaa6KFanQDfXw}{CJxW-K62Q4KQboZ0f1sL2A}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true},}
[2023-09-20T05:19:43,106][INFO ][o.e.c.s.ClusterApplierService] [es-2] removed {{es-1}{CyNVz5OOTQaa6KFanQDfXw}{CJxW-K62Q4KQboZ0f1sL2A}{192.168.0.155}{192.168.0.155:9300}{xpack.installed=true},}, term: 3, version: 21, reason: Publication{term=3, version=21}
```

## node 3 test

- 한 개 더 붙여서 테스트, node 1이 master

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role node.roles           cluster_manager name
192.168.0.179           41          70   0    0.14    0.10     0.09 dm        cluster_manager,data -               node-2
192.168.0.155           46          83   0    0.02    0.14     0.16 dm        cluster_manager,data *               node-1
192.168.0.149           13          44  11    0.74    1.35     0.82 dm        cluster_manager,data -               node-3 
```

- node 1을 kill했을 경우, 알아서 적합한 애가 승격 예시에서는 node 3이 승격됨.

```
[2023-09-20T05:44:06,072][INFO ][o.o.c.s.ClusterApplierService] [node-2] cluster-manager node changed {previous [], current [{node-3}{2Zs13JQcS8ydhdkyHNTp1w}{is6Ge0UISQCBShg0R_Hr4g}{192.168.0.149}{192.168.0.149:5300}{dm}{shard_indexing_pressure_enabled=true}]}, term: 4, version: 30, reason: ApplyCommitRequest{term=4, version=30, sourceNode={node-3}{2Zs13JQcS8ydhdkyHNTp1w}{is6Ge0UISQCBShg0R_Hr4g}{192.168.0.149}{192.168.0.149:5300}{dm}{shard_indexing_pressure_enabled=true}}
[2023-09-20T05:44:06,073][INFO ][o.o.a.u.d.DestinationMigrationCoordinator] [node-2] Detected cluster change event for destination migration
[2023-09-20T05:44:06,236][INFO ][o.o.d.PeerFinder         ] [node-2] setting findPeersInterval to [1s] as node commission status = [true] for local node [{node-2}{xoovvHheRWqTLko7PspKlg}{22eZK69ZS92U_hCs9Py6hg}{192.168.0.179}{192.168.0.179:5300}{dm}{shard_indexing_pressure_enabled=true}]
[2023-09-20T05:44:06,519][INFO ][o.o.c.s.ClusterApplierService] [node-2] removed {{node-1}{dzqQbH2pR1OaYQ5BQElHzQ}{NSFWBbMXSASlgGZItP866w}{192.168.0.155}{192.168.0.155:5300}{dm}{shard_indexing_pressure_enabled=true}}, term: 4, version: 31, reason: ApplyCommitRequest{term=4, version=31, sourceNode={node-3}{2Zs13JQcS8ydhdkyHNTp1w}{is6Ge0UISQCBShg0R_Hr4g}{192.168.0.149}{192.168.0.149:5300}{dm}{shard_indexing_pressure_enabled=true}}
```

```
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role node.roles           cluster_manager name
192.168.0.149           17          45   1    0.29    1.12     0.77 dm        cluster_manager,data *               node-3
192.168.0.179           45          70   0    0.05    0.08     0.08 dm        cluster_manager,data -               node-2
```

- 승격된 node3을 죽이면 node 2는 작동 중지

```
[2023-09-20T05:45:34,591][WARN ][o.o.c.c.ClusterFormationFailureHelper] [node-2] cluster-manager not discovered or elected yet, an election requires at least 2 nodes with ids from [2Zs13JQcS8ydhdkyHNTp1w, xoovvHheRWqTLko7PspKlg, dzqQbH2pR1OaYQ5BQElHzQ], have discovered [{node-2}{xoovvHheRWqTLko7PspKlg}{22eZK69ZS92U_hCs9Py6hg}{192.168.0.179}{192.168.0.179:5300}{dm}{shard_indexing_pressure_enabled=true}] which is not a quorum; discovery will continue using [192.168.0.155:5300, 192.168.0.149:5300] from hosts providers and [{node-3}{2Zs13JQcS8ydhdkyHNTp1w}{is6Ge0UISQCBShg0R_Hr4g}{192.168.0.149}{192.168.0.149:5300}{dm}{shard_indexing_pressure_enabled=true}, {node-2}{xoovvHheRWqTLko7PspKlg}{22eZK69ZS92U_hCs9Py6hg}{192.168.0.179}{192.168.0.179:5300}{dm}{shard_indexing_pressure_enabled=true}] from last-known cluster state; node term 4, last-accepted version 33 in term 4
```

- node 1을 작동시키면 투표권이 있는 두 노드가 있기 때문에 문제없이 재기동 해당 케이스에선 node 2가 master가 됨.

```
[2023-09-20T05:45:54,881][INFO ][o.o.c.c.FollowersChecker ] [node-2] FollowerChecker{discoveryNode={node-3}{2Zs13JQcS8ydhdkyHNTp1w}{is6Ge0UISQCBShg0R_Hr4g}{192.168.0.149}{192.168.0.149:5300}{dm}{shard_indexing_pressure_enabled=true}, failureCountSinceLastSuccess=1, [cluster.fault_detection.follower_check.retry_count]=3} marking node as faulty
[2023-09-20T05:45:55,119][INFO ][o.o.c.s.ClusterApplierService] [node-2] cluster-manager node changed {previous [], current [{node-2}{xoovvHheRWqTLko7PspKlg}{22eZK69ZS92U_hCs9Py6hg}{192.168.0.179}{192.168.0.179:5300}{dm}{shard_indexing_pressure_enabled=true}]}, added {{node-1}{dzqQbH2pR1OaYQ5BQElHzQ}{gZQfjtgoTJ6Ex_wnPPB2kQ}{192.168.0.155}{192.168.0.155:5300}{dm}{shard_indexing_pressure_enabled=true}}, term: 5, version: 34, reason: Publication{term=5, version=34}

```


## 결론!!

- **2 node 상황에서 cluster HA는 불가능하다. ( node의 roles과 무관하게 )**
- **3 node 상황에서 cluster HA는 문제없이 작동한다. ( 모든 node가 cluster_manager여야 함)**
- **Opensearch, elasticsearch 동일함.**
- 한번 cluster manager로 결정되고 나면 모두가 온라인 상태에서 cluster manager를 fail-over하는 것은 불가능하다.
- 공식 문서 참조 : [https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-small-clusters.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-small-clusters.html)
- split-brain 상황 때문에 그렇다고 하는데, 이를 방지하기 위해 투표 형식으로 선출한다. cluster manager의 선출은 초기 설정을 제외하면 투표로 이루어지고, 과반수가 넘어야 하기 때문에 2 node 상황에서 선출이 불가능하다고 한다.

```
We recommend you set only one of your two nodes to be master-eligible. 
This means you can be certain which of your nodes is the elected master of the cluster. 
The cluster can tolerate the loss of the other master-ineligible node. 
If you set both nodes to master-eligible, two nodes are required for a master election. 
Since the election will fail if either node is unavailable, 
your cluster cannot reliably tolerate the loss of either node.
```
