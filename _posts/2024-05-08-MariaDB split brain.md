---
key: jekyll-text-theme
title: 'MariaDB split brain'
excerpt: 'MariaDB Troubleshooting 😎'
tags: [MariaDB, Troubleshooting]
---


# MariaDB split brain

## 문제 발생

* maxscale을 사용했을 때 split brain 현상이 발생하였음

* 원인 자체는 여러 개가 있을 수 있겠지만, max_connection의 에러로 인해 발생

* 환경은 k8s pods로 구성된 sts+maxscale


## 원인

* max connection 문제로 인해 지속적으로 에러가 발생함.

* 에러 자체는 발생할 수 있는데 무슨 이유에선지 maxscale이 서로를 master로 판단해버림.

* split brain의 주요 원인이긴 한데 이것이 왜 그렇게 작동한 건지는 잘 모르겠음.

```
2024-05-08 22:43:11   error  : (212345) [readwritesplit] (Splitter-Service) Lost connection to the master server, closing session. Lost connection to master server while waiting for a result. Connection has been idle for 0 seconds. Error caused by: #HY000: Connection rejected: Too many connections (server0). Last close reason: <none>. Last error: 

## server 1 maxscale
name: [server0] status: [Running] state: [NOT_IN_USE] last opened at: [not opened] last closed at: [not closed] last close reason: [] num sescmd: [0]
name: [server1] status: [Master, Running] state: [NOT_IN_USE] last opened at: [Wed May  8 22:35:09 2024] last closed at: [Wed May  8 22:35:09 2024] last close

## server 2 maxscale
name: [server0] status: [Master, Running] state: [NOT_IN_USE] last opened at: [Wed May  8 22:43:05 2024] last closed at: [Wed May  8 22:43:05 2024] last close reason: [Master connection failed: #HY000: Connection rejected: Too many connections (server0)] num sescmd: [0]
name: [server1] status: [Running] state: [NOT_IN_USE] last opened at: [not opened] last closed at: [not closed] last close reason: [] num sescmd: [0]
```

## 해결

### 기존 DB 제거

* 데이터를 master 기준으로 맞추는 게 낫겠다 판단하였고 server 1은 남겨두고 replica 변경으로 하나를 제거함.

* 마찬가지로 해당 pod의 pvc도 제거해서 아예 DB를 초기화한 후 복구

```
# 예시
$ kubectl scale sts <statefulset-name> --replicas=1
$ k delete pvc mariadb-sts-1
$ kubectl scale sts <statefulset-name> --replicas=2
```

### Slave 재구축

* 원래는 sts 변경으로 될거라 생각했으나 생각대로 되지 않아 수작업으로 진행함.

* 원래는 자동으로 table 데이터 등을 넣도록 되어 있으나 position이 너무 커서인지 제대로 수행하지 못했고, master dump를 통해 시점복구로 동기화하도록 변경했다.

### Master dump

* `master-data=2`를 사용할 경우에 내부에 position, file 정보를 추가해 주기 때문에, 해당 정보를 참고해서 Slave 연결시 사용하면 됨.

```
$ mariadb-dump -u root -p --all-databases --master-data=2 --flush-logs --single-transaction > master_db.sql
```

### Slave import

* 해당 파일을 import

```
$ mariadb -u root -p < master_db.sql
```

### Slave Connect

* Slave는 정보가 다 있긴 하지만 position 등을 지정해 줘야 하므로 설정

* `MASTER_LOG_FILE`, `MASTER_LOG_POS`는 dump file에 적어 놨으므로 복사해 주면 된다.

```
mariadb> stop slave;

mariadb>
CHANGE MASTER TO
MASTER_HOST='<svc host>',
MASTER_USER='repluser',
MASTER_PASSWORD='<password>',
MASTER_LOG_FILE='bin.~~~~',
MASTER_LOG_POS=<position>;

mariadb> start slave;
```

## 변경사항

* MariaDB의 `max_connections`를 변경함.

* 추후 yaml에 추가하는 방향도 생각해 봐야 함. 151은 너무 적은 것 같다.

```
SET GLOBAL max_connections = 500;
SHOW VARIABLES LIKE 'max_connections';
```
