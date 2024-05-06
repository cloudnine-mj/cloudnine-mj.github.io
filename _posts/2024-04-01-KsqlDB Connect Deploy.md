---
key: jekyll-text-theme
title: 'KsqlDB Connect Deploy'
excerpt: 'KsqlDB Connect 설치 😎'
tags: [KsqlDB, KsqlDB connect, 데이터 가공]
---


# KsqlDB Connect Deploy

* 구성하고자 하는 방식 정리

	* Kafka connect(connecor를 관리)배포
	* JDBC, opensearch sink connector 구성

외부 -source connector→ Kakfa(connect) -sink connector->외부

## 구성

* Ubuntu
* Kafka 3.3.2
* Kafka connect 7.3.X


## 설치

* Kafka schema registry 및 service deploy (AVRO형식으로 데이터를 전송할 때 사용)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: schema-registry
  namespace: datatest
spec:
  selector:
    matchLabels:
      app: schema-registry
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: schema-registry
    spec:
      containers:
      - name: my-cluster-schema-registry
        image: confluentinc/cp-schema-registry:7.4.1
        ports:
        - containerPort: 8081
        env:
        - name: SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS
          value: kafka-cluster-kafka-bootstrap:9092 # 배포된 카프카 클러스터 서비스 주소
        - name: SCHEMA_REGISTRY_HOST_NAME
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: SCHEMA_REGISTRY_LISTENERS
          value: http://0.0.0.0:8081
        - name: SCHEMA_REGISTRY_KAFKASTORE_SECURITY_PROTOCOL
          value: PLAINTEXT
---
apiVersion: v1
kind: Service
metadata:
  name: schema-registry
  namespace: dataplatform
spec:
  ports:
  - port: 8081
  clusterIP: None
  selector:
    app: schema-registry
```

* PV, PVC

```
pv-plugins.yml

apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: dataplatform
  name: kafka-connect-plugins
  labels:
    app: kafka-connect
spec:
  storageClassName: local-redis-storage
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /root/kafka-connect-libs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-connect-plugins-config-pvc
  namespace: dataplatform
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-redis-storage
                                                
                                                
pv-kafka-connect.yml

apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: dataplatform
  name: kafka-connect-pv
  labels:
    app: kafka-connect
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /root/kafka-connect
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-connect-config-pvc
  namespace: dataplatform
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
```

## Kafka-connect 및 서비스 deploy

```
kafaka-connect.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-connect-pod
  namespace: dataplatform
  labels:
    app: kafka-connect
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-connect
  template:
    metadata:
      labels:
        app: kafka-connect
    spec:
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
      nodeSelector:
        kubernetes.io/hostname: k8s-worker-1
      containers:
        - name: kafka-connect
          image: confluentinc/cp-kafka-connect:7.3.7
          env:
            - name: CONNECT_BOOTSTRAP_SERVERS
              value: "kafka-cluster-kafka-bootstrap.dataplatform.svc.cluster.local:9092"
            - name: CONNECT_CONFIG_STORAGE_TOPIC
              value: connect-configs
            - name: CONNECT_GROUP_ID
              value: kafka-connect-dev
            - name: CONNECT_KEY_CONVERTER
              value: io.confluent.connect.avro.AvroConverter
            - name: CONNECT_VALUE_CONVERTER
              value: io.confluent.connect.avro.AvroConverter
            - name: CONNECT_OFFSET_STORAGE_TOPIC
              value: "kafka-connect-offsets"
            - name: CONNECT_STATUS_STORAGE_TOPIC
              value: "kafka-connect-status"
            - name: CONNECT_SECURITY_PROTOCOL
              value: "PLAINTEXT"
            - name: CONNECT_SSL_TRUSTSTORE_PASSWORD
              value: secret
            - name: CONNECT_SSL_KEYSTORE_PASSWORD
              value: secret
            - name: CONNECT_PLUGIN_PATH
              value: /usr/share/java/libs,/etc/kafka/connect,/usr/share/java/libs/confluentinc-kafka-connect-jdbc-10.7.4, /usr/share/java/libs/opensearch-connector-for-apache-kafka-3.1.1
            - name: CONNECT_REST_ADVERTISED_HOST_NAME
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL
              value: http://schema-registry:8081
            - name: CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL
              value: http://schema-registry:8081
          volumeMounts:
            - name: config-volume
              mountPath: /etc/kafka/connect
            - name: libs
              mountPath: /usr/share/java/libs
      volumes:
        - name: config-volume
          persistentVolumeClaim:
            claimName: kafka-connect-config-pvc #link pvc created in previous step
        - name: libs
          persistentVolumeClaim:
            claimName: kafka-connect-plugins-config-pvc #link pvc created in previous step
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-connect
spec:
  selector:
    app: kafka-connect
  ports:
    - protocol: TCP
      port: 8083
      targetPort: 8083
      nodePort: 30099
```