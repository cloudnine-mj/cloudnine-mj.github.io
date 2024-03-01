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
log_destination='stderr' # default stderr  log_connections=on  #default off  
```

**[참고 - 옵션 정보]**

* wal_level : WAL (Write Ahead Log)에 기록되는 정보(양)를 결정

  | 옵션    | 설명                                                         |
  | ------- | ------------------------------------------------------------ |
  | minimal | 기본값은 충돌 또는 즉시 셧다운으로부터 복구하기 위해 필요한 정보만 기록 |
  | archive | Wal 아카이브에 필요한 로깅만 추가 (9.6 버전부터는 hot_standby와 archive 모두 replica로 대체) |
  | replica | 9.4버전까지는 hot_standby. 9.6 버전부터는 replica라는 값으로 대체되었고, Slave 노드에서 읽기 전용 쿼리에 필요한 정보를 추가하게 됨. |
  | logical | 논리적 디코딩을 지원하는데 필요한 정보를 추가                |

* max_wal_sanders : 스트리밍 기반의 백업 클라이언트로부터의 동시 연결 최대 수를 지정

  * WAL Sender 프로세서는 max_connections 보다 큰 값을 설정할 수 없음.
  * 일반적으로 slave의 수 + 1 으로 많이 설정함.

* wal_keep_size : Slave 서버가 streaming replication을 위해 과거 로그 파일을 가져와야 하는 경우 pg_wal(10 버전 미만은 pg_xlog) 디렉터리에 저장되는 과거 로그 조각 파일의 최소 크기를 지정

  * 즉, Slave 서버를 위해 남겨 놓을 WAL 양을 지정
  * default = 0 (비활성화 상태)
  * PostgreSQL 13 이후 wal_keep_segments -> wal_keep_size로 변경 (단위: MB)
  * Maximum 수가 지정되어 있지 않기 때문에 서버의 디스크 공간, DB 트랜잭션에 따른 wal의 갱신 속도를 고려해서 설정값을 찾아야 함.
  * wal_keep_size = wal_keep_segments * wal_segment_size (일반적으로는 16MB)



## 2. pg_hba_conf 수정


```
[/home/postgres/as15/data] vi pg_hba.conf  # slave 서버 접속 권한 부여
```

* vi 편집기에 다음과 같이 입력
  * replication만 수정하는 게 아니라 IPv4 local connections의 address도 같이 수정해서 진행해야 함.

```
host    replication     all             0.0.0.0/0         trust  
host    all             all             0.0.0.0/0         trust
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

