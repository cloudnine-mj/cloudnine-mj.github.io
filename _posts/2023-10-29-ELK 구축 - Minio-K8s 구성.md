---
key: jekyll-text-theme
title: 'Minio - K8s로 구성'
excerpt: 'Minio Storage Research 😎'
tags: [Minio, k8s, Research]
---



:point_right: Minio Document : [https://min.io/docs/minio/kubernetes/upstream/index.html](https://min.io/docs/minio/kubernetes/upstream/index.html)





# 구성

* K8s는 minio operator가 존재하지만, tenant 분리 등의 조건이 필요하다.

| 구분       | K8s 정보         |
| ---------- | ---------------- |
| Namespace  | minio            |
| Deployment | minio            |
| PVC        | minio-pv-claim   |
| PV         | minio-pv         |
| Service    | minio-appservice |



# Yaml

### minio-pvc.yaml

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-pv-claim
  namespace: minio
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: minio-pv
  namespace: minio
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/minio

```

### minio-deploy.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: minio
  name: minio
  namespace: minio # Change this value to match the namespace metadata.name
spec:
  selector:
    matchLabels:
      app: minio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: quay.io/minio/minio:latest
        args:
          - server
          - /data
          - --console-address
          - ":3000"
        volumeMounts:
        - name: storage
          mountPath: "/data"
        ports:
        - name: minioapi
          containerPort: 9000
        - name: minioconsole
          containerPort: 3000
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: minio-pv-claim
```

### minio-svc.yaml

* minio는 api서버와 console서버를 분리하고 있음. k8s 내부에서 minio를 찔러야 할 일이 생기면 9000번 포트로 설정하거나 minioapi로 설정해 줘야 한다.


```
apiVersion: v1
kind: Service
metadata:
  name: minio-appservice
  namespace: minio
spec:
  type: NodePort
  ports:
    - name: minioapi
      port: 9000
      targetPort: minioapi
      protocol: TCP
    - name: minioconsole
      port: 3000
      targetPort: minioconsole
      nodePort: 31000
      protocol: TCP
  selector:
    app: minio
```