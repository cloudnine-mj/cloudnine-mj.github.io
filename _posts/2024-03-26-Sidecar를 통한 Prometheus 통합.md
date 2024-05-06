---
key: jekyll-text-theme
title: 'Sidecar를 통한 Prometheus 통합'
excerpt: 'Prometheus 통합하기 😎'
tags: [Sidecar, Prometheus]
---

# Sidecar를 통한 Prometheus 통합

* Prometheus 간의 통합은 federation이 있다

* 문제는 이 federation 기능이 대규모의 쿼리를 날리게 되면서 메모리를 차지하고 prometheus instance에 큰 부담을 준다는 사실임.

* thanos sidecar를 통해 이 부분을 개선하고 통합할 수 있음.

* thanos sidecar는 프로메테우스의 tsdb를 직접 attach하기 때문에 메모리 등의 케파 문제에서 상대적으로 부하가 덜하다고 볼 수 있다. 내부적으로 gRPC를 이용하는 것으로 보인다.

## 구성

* K8s로 하나, Vm으로 하나해서 두 개 통합하는 방식으로 구성

### Server 1

* Prometheus
* Sidecar

### Server 2

* Prometheus

* Sidecar

* Querier

	* Querier는 Sidecar가 수집한 정보를 취합하는 역할을 한다.

	* 두 개의 Sidecar를 연동하여 쿼리로 확인할 것임.

## Prometheus 설정

* prometheus.yml

* external_label은 임의대로 설정해도 문제는 없을 것 같고 구분할 수만 있으면 된다.

```
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
  external_labels:
    cluster: "prom_vm"
```

## Thanos 설치

* 설치는 매우 간단함

```
wget <https://github.com/thanos-io/thanos/releases/download/v0.34.1/thanos-0.34.1.linux-amd64.tar.gz>
tar -xvf thanos-0.34.1.linux-amd64.tar.gz
```

### Thanos 설정

* 실행할 때 옵션을 준다.

* tsdb.path : prometheus에서 사용하는 tsdb에 접속할 수 있도록 위치를 설정

* prometheus.url : prometheus의 endpoint

* objstore.config-file : 필수사항은 아니지만, minio를 통해 cold storage를 구성하기로 했으므로 이것도 테스트 용도로 넣어봄.

* http-address : sidecar의 endpoint

* grpc-address : sidecar의 gRPC 프로토콜 엔드포인트. querier에선 해당 포트를 보게 된다.

```
실행할때 옵션을 준다.

tsdb.path : prometheus에서 사용하는 tsdb에 접속할 수 있도록 위치를 설정

prometheus.url : prometheus의 endpoint

objstore.config-file : 필수사항은 아니지만, minio를 통해 cold storage를 구성하기로 했으므로 이것도 테스트 용도로 넣어봄.

http-address : sidecar의 endpoint

grpc-address : sidecar의 gRPC 프로토콜 엔드포인트. querier에선 해당 포트를 보게 된다.
```

## Querier 설정

* 이것도 마찬가지로 기동시 옵션으로 추가할 수 있다.

* 해당 예시는 deployment로 k8s 기동했을 때임.

* store 옵션을 하나 추가할 때마다 항목이 추가된다.

* store=10.0.0.158:10901을 추가함.

```
          args:
            - query
            - --log.level=info
            - --endpoint.info-timeout=30s
            - --grpc-address=0.0.0.0:10901
            - --http-address=0.0.0.0:10902
            - --query.replica-label=prometheus_replica
            - --store=sidecar.thanos.svc.cluster.local:10901
            - --store=10.0.0.158:10901
            # START: Only after you deploy storegateway
            - --store=storegateway.thanos.svc.cluster.local:10901
            # END: Only after you deploy storegateway
            # # START: Mutual TLS
            # - --grpc-client-tls-secure
            # - --grpc-client-tls-cert=/secrets/tls.crt
            # - --grpc-client-tls-key=/secrets/tls.key
            # - --grpc-client-tls-ca=/secrets/ca.crt
            # # END: Mutual TLS
```

## Sidecar 연동 확인

* Query 웹서버에 접속해야 함.

* 정상적으로 연동되었을 경우 sidecar 항목에 추가되고, 라벨을 확인할 수 있다.

* graph에서 쿼리를 통해서 결과 확인 가능.


## 주의 사항

### Prometheus block duration

* Prometheus의 min-block-duration과 max-block-duration을 설정해 줘야 하고 같아야 한다.

* 이는 Sidecar에서 compaction된 데이터를 다루지 않기 위함인데( Thanos Sidecar에서 따로 compaction을 수행함 ) 각 값이 서로 다르면 thanos와 Prometheus간 compaction 시기에 혼란이 올 수 있어 데이터 정립성 문제가 생길 수 있다 판단한 것 같다.

* Prometheus 기동 시 해당 옵션을 줘야 함.

```
--storage.tsdb.min-block-duration=2h
--storage.tsdb.max-block-duration=2h
```

### 쿼리 작성 시 문제

* 쿼리 작성 시 external label 만으로는 조회 불가능
* external label(예시에서는 cluster=’prom_vm’)만으로는 조회가 불가능하다.
* external label은 thanos 내부에서 다르게 취급하기 때문인 것 같음.
* 다른 label과 병용하면 문제는 없다.