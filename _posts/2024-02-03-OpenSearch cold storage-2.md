---
key: jekyll-text-theme
title: 'OpenSearch cold storage-2'
excerpt: ' OpenSearch cold storage research 😎'
tags: [Opensearch, Cold storage, Research]
---

# OpenSearch cold storage-2

## remote snapshot restore시 메모리, 디스크 사용량

- restore할때 실제 데이터를 가져오는 것은 아닌 것으로 확인되긴 하지만, 메모리에도 올라가는 것인지에 대한 확인이 필요하다.
- 해당 api로 jvm 힙을 측정할 수 있는데, restore 후 사용량을 측정해 본다.

```
GET <https://localhost:31920/_nodes/stats/jvm> 
```

- 우선 기본 used 힙사이즈 측정

```
$ curl -XGET <https://localhost:31920/_nodes/stats/jvm> --insecure -u admin:admin | jq . | grep heap_used_in_bytes "heap_used_in_bytes": 138779584, "non_heap_used_in_bytes": 241806896, "heap_used_in_bytes": 354486608, "non_heap_used_in_bytes": 245952992, 
```

### Physical restore

- 4만 개 정도로 아주 큰 용량은 아니지만 10M 정도 하는 index를 복원해본다.

```
POST _snapshot/minio-repository/cold_snapshot-cold-index-2024.02.01-000001-2024.02.01-04:39:12.985/_restore
{
  "indices": "cold-index-2024.02.01-000001",
  "include_global_state": false
}
```

- 다시 힙 측정.

```
$ curl -XGET <https://localhost:31920/_nodes/stats/jvm> --insecure -u admin:admin | jq . | grep heap_used_in_bytes
"heap_used_in_bytes": 143714656,
"non_heap_used_in_bytes": 241927792,
"heap_used_in_bytes": 375458128,
"non_heap_used_in_bytes": 246090424,
```

### remote restore

- 기본 힙사이즈

```
$ curl -XGET <https://localhost:31920/_nodes/stats/jvm> --insecure -u admin:admin | jq . | grep heap_used_in_bytes
"heap_used_in_bytes": 342498064,
"non_heap_used_in_bytes": 243143072,
"heap_used_in_bytes": 384341712,
"non_heap_used_in_bytes": 248334760,
```

- remote 복원

```
POST _snapshot/minio-repository/cold_snapshot-cold-index-2024.02.01-000001-2024.02.01-04:39:12.985/_restore
{
  "indices": "cold-index-2024.02.01-000001",
  "include_global_state": false,
  "storage_type": "remote_snapshot"
}
```

- 힙 측정

```
"heap_used_in_bytes": 339352336,
"non_heap_used_in_bytes": 243143072,
"heap_used_in_bytes": 384341712,
"non_heap_used_in_bytes": 248334760,
```

### 결론

- 힙사이즈에 데이터 restore가 끼치는 영향은 없다고 봐도 무방할 듯 함.


## Remote snapshot 시 Disk 사용 여부

- 실제로 remote로 작동해도 실 데이터를 가져오는지 여부를 판단해본다..

### 일반 복원

- 4만 개 정도의 index를 물리 restore 처리함.

```
POST _snapshot/minio-repository/cold_snapshot-cold-index-2024.02.01-000001-2024.02.01-04:39:12.985/_restore
{
  "indices": "cold-index-2024.02.01-000001",
  "include_global_state": false
}
```

- _cat/indices에서 인덱스 id를 알 수 있음. htvZrLIOTZKeSzbg4XYuBw ← 해당 아이디가 이름

```
green open cold-index-2024.02.01-000001   htvZrLIOTZKeSzbg4XYuBw 1 1 40200   0  20.1mb    10mb
```

- 양쪽 노드에서 데이터 크기 확인

```
[opensearch@opensearch-cluster-master-0 indices]$ du -sh htvZrLIOTZKeSzbg4XYuBw/
11M     htvZrLIOTZKeSzbg4XYuBw/
[opensearch@opensearch-cluster-master-1 indices]$ du -sh htvZrLIOTZKeSzbg4XYuBw/
11M     htvZrLIOTZKeSzbg4XYuBw/
```

- 이번에는 지우고 다시 remote restore 처리.

```
POST _snapshot/minio-repository/cold_snapshot-cold-index-2024.02.01-000001-2024.02.01-04:39:12.985/_restore
{
  "indices": "cold-index-2024.02.01-000001",
  "include_global_state": false,
  "storage_type": "remote_snapshot"
}
```

- index 정보

```
green open cold-index-2024.02.01-000001   q511OxlATvisRDgwXTtjLw 1 1 40200 0  20.1mb    10mb
```

- 용량을 보면 272k 등 메타정보 정도만 저장하는걸 알수있음

```
[opensearch@opensearch-cluster-master-0 indices]$ du -sh q511OxlATvisRDgwXTtjLw
272K    q511OxlATvisRDgwXTtjLw
[opensearch@opensearch-cluster-master-1 indices]$ du -sh q511OxlATvisRDgwXTtjLw
260K    q511OxlATvisRDgwXTtjLw
```

### 결론

- remote restore 시 데이터를 실제로 가져오지 않고 meta 정보만 가져오는 것으로 판단할 수 있음.
- 실제로 search를 실행해서 읽어도 데이터를 가져오지만 내부 용량이 늘어나진 않음.
- 

## _cat/indices 데이터 불일치

- _cat/indices를 사용할 경우 실제 데이터 갯수와 해당 호출 결과 갯수가 차이가 날 때가 있다.
- [https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-refresh.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-refresh.html)
- elasticsearch의 예시긴 하지만 opensearch도 비슷하며, search 요청이 없을 경우 갱신되지 않는다.
- 가장 큰 문제는 ISM이 해당 메타데이터를 보고 판단한다는 것이다. (count로 ism을 rollover할시에 문제가 될 수 있음)
- 실제 운영 상황에 갱신되지 않을 일은 없다고 판단할 수 있으나, 테스트시에는 주기적으로 해당 API를 호출해 줌으로써 ISM을 원활하게 돌아가게 만들 수 있다.

```
POST cold-index/_refresh 
```

## ISM 사용 주기

- ISM은 특정 주기마다 ISM 조건에 맞는 인덱스가 있는지 검사하는 과정을 거친다.
- 기본이 10분이라 테스트시 문제가 발생할 수 있으므로 해당 값을 바꿔 주면 테스트할때 도움이 될 수 있다.

# Troubleshooting

## ISM 구성 전 bucket의 내용물이 없어야 함

- Bucket에 기존 snapshot 등의 내용이 있으면 문제가 생길 수 있다
- 직접 볼 순 없으나 기존에 만들어진 snapshot의 메타도 같이 데이터로 존재하므로 해당 데이터를 불러오면서 문제가 생길 수 있다.
- bucket을 분리하거나 삭제하고 재생성해야 한다.

```
"info": {
      "cause": "[minio-repository] Could not read repository data because the contents of the repository do not match its expected state. This is likely the result of either concurrently modifying the contents of the repository by a process other than this cluster or an issue with the repository's underlying storage. The repository has been disabled to prevent corrupting its contents. To re-enable it and continue using it please remove the repository from the cluster and add it again to make the cluster recover the known state of the repository from its physical contents.",
      "message": "Failed to get status of snapshot [index=cold-index-2024.02.01-000001]"
```

## Retsore 시 반드시 Cluster로 구성되어 있어야 함

- 1 node로 구성할 경우 사실상 repository 기능을 사용할 수 없다.
- 제대로 된 클러스터가 구성되어 있지 않으면 복구 후에 index shard 문제로 meta만 복구되고 실제 데이터는 볼 수 없다.
- indices로 확인 시 메타데이터가 보이지 않게 된다.

```
red open cold-index-2024.02.01-000002   C6tlnSi4T9OSWQKQT2bCew 1 1
green open .opendistro-job-scheduler-lock NYxCdbmTSuug4PNCP1lZMw 1 1    6 34  64.7kb  47.7kb
green open security-auditlog-2024.02.01   nRdvdRG-S_ydbDH3GsQnXg 1 1   18  0 366.5kb 183.2kb
green open .kibana_1                      GJyvhp7tRkyQMmHsTK8lHA 1 1    1  0  10.3kb   5.1kb
green open .opendistro_security           vkyUa3lASfqio8FmHD6dFw 1 1   10  0 136.7kb  71.7kb
green open security-auditlog-2024.01.31   7Q8cXVPCT8qD8exhOlwPTg 1 1  152  0 931.7kb 475.2kb
```

## bucket Path 지정시 발생 에러

- snapshot 저장 시 Path를 지정해 줬을 때 저장은 하는데 restore시 인식하지 못하는 문제가 있다.
- 정확한 이유를 알 순 없지만 정황상 S3와 minio의 path 관리 구조가 미묘하게 달라서 생기는 문제로 추측된다.
- Bucket별로 구분하고 path는 지정해 주지 않으면 문제 없이 파일을 인식하게 된다.

```
User
Caused by: org.opensearch.common.io.stream.NotSerializableExceptionWrapper: 
execution_exception: 
SnapshotMissingException[[minio-repository:test_snapshot/Xbx6urFmTXynKvPAj18caA] is missing]; 
nested: NoSuchFileException[Blob object 
[indices/PcE85FE3Q0GN0vu_yjjkng/0/snap-Xbx6urFmTXynKvPAj18caA.dat] not found: 
The specified key does not exist. 
(Service: Amazon S3; Status Code: 404; Error Code: NoSuchKey; 
Request ID: 17AB5242B2C1CD1A; S3 Extended Request ID: dd9025bab4ad464b049177c95eb6ebf374d3b3fd1af9251148b658df7ac2e3e8; Proxy: null)];
```

## remote snapshot 기능 사용 불가 메시지

- jvm option에서 기능을 활성화지 않았을 때 생기는 문제이다.

```
POST _snapshot/minio-repository/test_snapshot/_restore
{
  "indices": "test-index",
  "storage_type": "remote_snapshot",
}

{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "Unsupported parameter storage_type. Feature flag is not enabled for this experimental feature"
      }
    ],
    "type": "illegal_argument_exception",
    "reason": "Unsupported parameter storage_type. Feature flag is not enabled for this experimental feature"
  },
  "status": 400
}
```