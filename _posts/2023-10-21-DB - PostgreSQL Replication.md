---
key: jekyll-text-theme
title: 'Postgresql Replication'
excerpt: 'PostgreSQL 😎'
tags: [PostgreSQL, DB]
---


# Master 설정
* 1번 서버 - 192.168.2.120

## 1. postgresql.conf 수정

```
[/home/postgres/as15/data] vi postgresql.conf
```

* vi 편집기에 다음과 같이 입력


```
listen_addresses = '*'  
shared_buffers = 2G  
archive_mode = on  
archive_command = 'cp %p /home/postgres/backups/archive_backup/%f'  
wal_level = replica  
logging_collector=on # default on  
log_destination='stderr' # default  
stderr log_connections=on  #default off  
```

## 2. pg_hba_conf 수정


```
[/home/postgres/as15/data] vi pg_hba.conf
```

* vi 편집기에 다음과 같이 입력
  * replication만 수정하는 게 아니라 IPv4 local connections의 address도 같이 수정해서 진행해야 함.

```
host    replication     all             0.0.0.0/0         trust  host    all             all             0.0.0.0/0         trust
```



## 3. slot create & search

```
[/home/postgres] pg_ctl start # 서버 실행  
[/home/postgres] psql  
postgres=# select * from pg_create_physical_replication_slot('standby1_slot');  
postgres=# select slot_name, slot_type, active from pg_replication_slots;  
```


## 4. Replica User Create

```
postgres=# CREATE USER repluser WITH ENCRYPTED PASSWORD 'repluser' login;  
postgres=# alter user repluser replication;
```

<br>

# Slave 설정 
* 2번 서버 - 192.168.2.121

## 1. Data 복제

```
[/home/postgres] mkdir -p archive_backup  
[/home/postgres] pg_basebackup -U repluser -h 192.168.2.120 -p 5432 -D $PGDATA -P -R -v -Xs  
[/home/postgres] pg_ctl start # 서버 실행
```

## 2. properties 수정

### postgresql.conf 수정

```
[/home/postgres/as15/data] vi postgresql.conf
```

* vi 편집기에서 다음과 같이 수정

```
listen_addresses = '*'  
hot_standby = on  
hot_standby_feedback = on  
primary_conninfo='host=192.168.2.120 port=5432 user=repluser password=repluser' #master ip
promote_trigger_file = '/home/postgres/as15/data/failover_trigger'  
primary_slot_name = 'standby1_slot'  
```

### pg_hba.conf 수정

```
[/home/postgres/as15/data] vi pg_hba.conf
```

* vi 편집기에서 아래와 같이 수정

```
host    replication     all           0.0.0.0/0          trust   host    all             all           0.0.0.0/0          trust
```

### 3. replication 확인하기


```
[/home/postgres/backups/archive_backup] ps -ef c| grep wal
postgres   31890   31861  0 10:35 ?        00:00:00 postgres: walreceiver streaming 0/5000238  
postgres   31892   20866  0 10:38 pts/1    00:00:00 grep --color=auto wal 

[/home/postgres/as15/data] ps -ef | grep wal 
postgres   52649   52644  0 10:35 ?        00:00:00 postgres: walwriter  
postgres   52655   52644  0 10:35 ?        00:00:00 postgres: walsender repluser 192.168.2.121(57436) streaming 0/5000238   postgres   52694   51772  0 10:39 pts/1    00:00:00 grep --color=auto wal
```

## **확인 작업**

* Master에서 psql 로 테이블 생성한 후 Slave에서도 조회가 되는지 확인

```
# Master

[/home/postgres/as15/data] psql
postgres=# CREATE TABLE employees (                             
    employee_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    hire_date DATE
);
postgres=# select * from employees;
```

```
# Slave

[/home/postgres/as15/data] psql
postgres=# select * from employees;
```

