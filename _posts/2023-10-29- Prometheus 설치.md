---
key: jekyll-text-theme
title: 'Prometheus Query'
excerpt: 'Prometheus Query Research 😎'
tags: [Prometheus, Research]
---

# Prometheus Query

- Query Range의 기본 단위는 5분으로 지정(최근 5분의 데이터)
- 실제 노드에서는 file system, device, network interface 등이 다르므로 원하는 값을 분류하려면 쿼리에 추가되어 함.
- **Log데이터는 Prometheus에서 수집할 순 없고 Loki - promtail이 따로 들어가야 함.**
  - 단, daemonset으로 구성하면 node 로그 수집은 문제는 없음

 

## 노드별 자원 정보 조회

| 설명                       | Query                                                        | 구분자                       |
| :------------------------- | :----------------------------------------------------------- | :--------------------------- |
| 노드별 CPU 사용률          | node_load5                                                   |                              |
| 노드별 메모리 사용률       | (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 | instance                     |
| 노드별 스토리지 사용률     | (1 - (node_filesystem_free_bytes / node_filesystem_size_bytes)) * 100 | instance, device, mountpoint |
| 노드별 네트워크 사용률(rx) | rate(node_network_receive_bytes_total[5m])                   | instance, device             |
| 노드별 네트워크 사용률(tx) | rate(node_network_transmit_bytes_total[5m])                  | instance, device             |

 

## Pod 조회

| 설명                          | Query                                                        |
| :---------------------------- | :----------------------------------------------------------- |
| pod 상태 별로 조회(groupping) | sum by (phase) (kube_pod_status_phase == 1)                  |
| pod 상태별 갯수 조회          | sum by (phase) (kube_pod_status_phase{phase=”<phase>"} == 1) |



## 로그 수집을 위한 Loki Api 사용법

- 예시 - 최근 5개의 log 항목을 가져올 경우

| 항목     | Value                       |
| :------- | :-------------------------- |
| Endpoint | loki/api/v1/query_range     |
| query    | query={filename=”test.log”} |
| limit    | 5                           |
| start    | start_timestamp             |
| end      | end_timestamp               |

 

- curl 구문 예시 

```
curl -G -s "http://<Loki-Server-Address>/loki/api/v1/query_range" \\  --data-urlencode 'query={filename="test.log"}' \\  --data-urlencode 'limit=5' \\  --data-urlencode 'start=<start_timestamp>' \\  --data-urlencode 'end=<end_timestamp>'
```
