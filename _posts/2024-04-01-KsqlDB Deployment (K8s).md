---
key: jekyll-text-theme
title: 'KsqlDB Deployment (K8s)'
excerpt: 'KsqlDB 설치 😎'
tags: [KsqlDB, KsqlDB 설치, 데이터 가공]
---


# KsqlDB Deployment (K8s)

## kubernetes namespace 생성

* ksqldb cluster를 구성할 kubernetes namespace 생성

```
kubectl create ns kslqdb
```


## strimzi helm으로 설치

* kafka, zookeeper를 k8s에서 배포, 관리하기 위한  strimzi helm으로 설치

```
#helm repo 업데이트
helm repo add strimzi https://strimzi.io/charts/
helm repo update

#image pull
helm pull strimzi/strimzi-kafka-operator

#압축해제
tar -zxvf strimzi-kafka-operator-helm-3-chart-0.32.0.tgz

#디렉토리 이동
cd strimzi-kafka-operator

#헬름차트로 설치
helm install strimzi-kafka-operator . -n ksqldb
#헬름차트의 이름을 strimzi-kafka-operator로 지정
#현재위치를 기준으로 지정
```


## Strimzi를 이용해 Kafka, Zookeeper 배포

### kafka_cluster.yml

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  #kafka cluster
  name: test-cluster
spec:
  entityOperator:
    topicOperator: {}
    userOperator: {}
  kafka:
    config:
      default.replication.factor: 3
      inter.broker.protocol.version: "3.2"
      min.insync.replicas: 2
      offsets.topic.replication.factor: 3
      transaction.state.log.min.isr: 2
      transaction.state.log.replication.factor: 3
    listeners:
      - name: plain
        port: 9092
        tls: false
        type: internal
      - name: tls
        port: 9093
        tls: true
        type: internal
    replicas: 1
    storage:
      type: jbod
      volumes:
        - id: 0
          type: persistent-claim
          size: 30Gi
          deleteClaim: true
    #strimzi 최신버전(0.39.0)은 3.5.0, 3.5.1, 3.5.2, 3.6.0, 3.6.1 만 지원
    version: 3.6.0
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 10Gi
      deleteClaim: true
```

### Kafka, Zookeeper를 K8s에 배포

```
kubectl apply -f kafka_cluster.yaml -n ksqldb

#pod가 정상적으로 실행되었나 확인 
#cluster-entity-operator가 정상적으로 running 상태인지 확인
kubectl get pod -n ksqldb
NAME                                            READY   STATUS    RESTARTS   AGE
strimzi-cluster-operator-7bb5468c59-qkxjb       1/1     Running   0          178m
test-cluster-entity-operator-55bb7d7849-9slqv   2/2     Running   0          177m
test-cluster-kafka-0                            1/1     Running   0          178m
test-cluster-zookeeper-0                        1/1     Running   0          178m

#service가 제대로 적용되었나 확인
kubectl get svc -n ksqldb
NAME                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                        AGE
test-cluster-kafka-bootstrap    ClusterIP      10.108.88.214    <none>        9091/TCP,9092/TCP,9093/TCP                     3h
test-cluster-kafka-brokers      ClusterIP      None             <none>        9090/TCP,9091/TCP,8443/TCP,9092/TCP,9093/TCP   3h
test-cluster-zookeeper-client   ClusterIP      10.102.177.112   <none>        2181/TCP                                       3h1m
test-cluster-zookeeper-nodes    ClusterIP      None             <none>        2181/TCP,2888/TCP,3888/TCP                     3h1m
```

## KsqlDB-server와 KsqlDB-cli 설치

### KsqlDB-server


*  [https://hub.docker.com/r/confluentinc/ksqldb-server/tags](https://hub.docker.com/r/confluentinc/ksqldb-server/tags) 를 참고
* 알맞은 버전의 ksqlDB-server image를 pull (latest 버전 사용)

```
docker pull confluentinc/ksqldb-server:latest
```

* Docker 이미지를 K8s에 배포

	* ksqldb-server.yml
	
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ksqldb-server-pod
  namespace: dataplatform
  labels:
    app: ksqldb-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ksqldb-server
  template:
    metadata:
      labels:
        app: ksqldb-server
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
      nodeSelector:
        kubernetes.io/hostname: k8s-worker-1
      containers:
        - name: ksqldb-server
          image: confluentinc/ksqldb-server:latest
          env:
            - name: KSQL_BOOTSTRAP_SERVERS
              value: "http://34.173.64.192:9092"
            - name: KSQL_KSQL_CONNECT_URL
              value: "http://10.97.208.94:8083"
            - name: KSQL_KSQL_EXTENSION_DIR
              value: /usr/share/java/ksql-server/
            - name: KSQL_CONFIG_DIR
              value: /etc/ksqldb
            - name: KSQL_LOG4J_OPTS
              value: -Dlog4j.configuration=file:/etc/ksqldb/log4j.properties
            - name: KSQL_KSQL_SCHEMA_REGISTRY_URL
              value: http://schema-registry:8081
            - name: KSQL_HEAP_OPTS
              value: "-Xms1G -Xmx1G"
          volumeMounts:
            - name: ksqldb-classpath
              mountPath: /usr/share/java/ksql-server/
      volumes:
        - name: ksqldb-classpath
          persistentVolumeClaim:
            claimName: ksqldb-plugins-pvc
```
	
* 배포
	
```
kubectl apply -f ksqldb-server.yml -n ksqldb

#pod 확인
kubectl get pod -n ksqldb
NAME                                            READY   STATUS    RESTARTS   AGE
ksqldb-server-pod                               1/1     Running   0          77m
strimzi-cluster-operator-7bb5468c59-qkxjb       1/1     Running   0          178m
test-cluster-entity-operator-55bb7d7849-9slqv   2/2     Running   0          177m
test-cluster-kafka-0                            1/1     Running   0          178m
test-cluster-zookeeper-0                        1/1     Running   0          178m
```
	
### KsqlDB-CLI

*  [https://hub.docker.com/r/confluentinc/ksqldb-server/tags](https://hub.docker.com/r/confluentinc/ksqldb-server/tags) 를 참고
	
* 알맞은 버전의 ksqlDB-CLI image를 pull (latest 버전 사용)

* docker 이미지를 K8s에 배포

	* ksqldb-cli.yml

```
apiVersion: v1
kind: Pod
metadata:
  name: ksqldb-cli-pod
spec:
  containers:
  - name: ksqldb-cli-container
    image: confluentinc/ksqldb-cli:latest
    command:
      - "/bin/sh"
      - "-c"
      - "while true; do sleep 30; done"
    args:
      - "-Command"
      - "& { while ($true) { Start-Sleep -Seconds 30 } }"
    env:
      - name: KSQL_SERVER 
        value: "ksql http://ksqldb-server:8088"
```

```
# 배포
get pod -n ksqldb
NAME                                            READY   STATUS    RESTARTS   AGE
ksqldb-cli-pod                                  1/1     Running   0          136m
ksqldb-server-pod                               1/1     Running   0          77m
strimzi-cluster-operator-7bb5468c59-qkxjb       1/1     Running   0          178m
test-cluster-entity-operator-55bb7d7849-9slqv   2/2     Running   0          177m
test-cluster-kafka-0                            1/1     Running   0          178m
test-cluster-zookeeper-0                        1/1     Running   0          178m
```

## 로컬에서 cli 접속

* kubernetes에서 배포했기 때문에 로컬에서 접속하기위해선 port forwarding이 필요 
* 따라서 로드밸런서를 추가할 yml 파일을 아래와 같이 작성함.

	* ksqldb-server-loadbalancer.yml

```
apiVersion: v1
kind: Service
metadata:
  name: ksqldb-server-service
spec:
  selector:
    app: ksqldb-server  #ksqldb-server.yml의 label과 일치해야함
  ports:
    - protocol: TCP
      port: 8088
      targetPort: 8088
  type: LoadBalancer
```

```
# port-forwarding
kubectl port-forward service/ksqldb-server-service 8088:8088 -n ksqldb
Forwarding from 127.0.0.1:8088 -> 8088
```


## 접속 확인

* 웹 브라우저로 cluster 상태 확인 http://127.0.0.1:8088/info

* CLI 접속

```
kubectl exec -it ksqldb-cli-pod -n ksqldb -- /bin/bash
```
	* ksql http://ksqldb-server-service:8088  ← 서비스 이름:port로 접속해야 함.