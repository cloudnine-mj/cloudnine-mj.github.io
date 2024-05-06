---
key: jekyll-text-theme
title: 'Thanos + Prometheus 통합(경량화)'
excerpt: 'Thanos + Prometheus 😎'
tags: [Thanos, Prometheus]
---


# Thanos + Prometheus 통합(경량화)


* Thanos를 Prometheus 통합 용도로 사용할 시 최소한의 설치로 작동하게 해야 함

* Thanos Sidecar는 반드시 필요하지만 Thanos는 querier만 있어도 큰 문제는 없음



## Thanos querier Deployment

* Thanos querier만 Deploy 해서 사용

* Serviceaccount는 필요 없고, querier는 일종의 파이프라인 통로와 같은 역할이라 TSDB와 같은 것들이 없다. 즉 pv가 필요치 않음. 캐싱용도로 메모리만 사용하므로 PV는 없어도 됨.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: thanos-test
  name: querier
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: querier
  template:
    metadata:
      labels:
        app.kubernetes.io/name: querier
    spec:
      securityContext:
        runAsUser: 1001
        fsGroup: 1001
      containers:
        - name: querier
          image: docker.io/bitnami/thanos:0.31.0
          args:
            - query
            - --log.level=info
            - --endpoint.info-timeout=30s
            - --grpc-address=0.0.0.0:10901
            - --http-address=0.0.0.0:10902
            - --query.replica-label=prometheus_replica
            - --store=sidecar.thanos.svc.cluster.local:10901
            - --store=10.0.0.158:10901
          ports:
            - name: http
              containerPort: 10902
              protocol: TCP
            - name: grpc
              containerPort: 10901
              protocol: TCP
          livenessProbe:
            failureThreshold: 6
            httpGet:
              path: /-/healthy
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 30
          readinessProbe:
            failureThreshold: 6
            httpGet:
              path: /-/ready
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 30
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 2Gi
```

## Thanos querier Service

* Port는 31750으로 노출

```
---
apiVersion: v1
kind: Service
metadata:
  namespace: thanos-test
  name: querier
spec:
  type: NodePort 
  ports:
    - port: 9090
      targetPort: http
      protocol: TCP
      name: http
      nodePort: 31750
    - port: 10901
      targetPort: grpc
      protocol: TCP
      name: grpc
  selector:
    app.kubernetes.io/name: querier
```


## Prometheus Deployment

* Prometheus를 배포 시 추가해줘야 되는 Container가 존재한다.

* Sidecar를 같이 배포해도 되고, 따로 배포해서 pv와 서비스에 붙여주면 되지만 굳이 그럴 이유가 없고 설정도 귀찮아 1 pod 2 container로 설정

* Sidecar를 따로 배포할 때는 pv를 동일하게 마운트하고 prometheus의 서비스를 묶어 주면 된다.

* 10901 : grpc, 10902 : http로 두 개의 포트를 노출시킴

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: thanos-test
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      serviceAccountName: thanos-test
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--storage.tsdb.retention.time=2h"
            - "--storage.tsdb.max-block-duration=2h"
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          resources:
            requests:
              cpu: 500m
              memory: 500M
            limits:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/

        - name: sidecar
          image: quay.io/thanos/thanos:v0.31.0
          args:
            - sidecar
            - "--tsdb.path=/prometheus"
            - "--prometheus.url=http://localhost:9090"
            - "--http-address=0.0.0.0:10902"
            - "--grpc-address=0.0.0.0:10901"
          volumeMounts:
            - name: prometheus-storage-volume
              mountPath: /prometheus/
          ports:
          - containerPort: 10902
            name: http
            protocol: TCP
          - containerPort: 10901
            name: grpc
            protocol: TCP

      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
        - name: prometheus-storage-volume
          emptyDir: {}
```

## Prometheus 설정 변경

* external label을 추가해주면 됨.

```
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
      external_labels:
        cluster: "prom_k8s"
```

## Sidecar service

* Sidecar를 노출시키고 Thanos querier에서 사용할 수 있도록 설정

* label은 그냥 Prometheus의 라벨을 붙여 주면 된다.

```
apiVersion: v1
kind: Service
metadata:
  namespace: thanos-test
  name: sidecar
spec:
  type: ClusterIP
  ports:
    - port: 10901
      targetPort: grpc
      protocol: TCP
      name: grpc
  selector:
    app: prometheus-server
```