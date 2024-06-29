---
key: jekyll-text-theme
title: 'Kafka connect JDBC Sink connector'
excerpt: 'JDBC Connect 설치/설정 😎'
tags: [KsqlDB, JDBC Sink connect]
---

# Kafka connect JDBC Sink connector

:star: 카프카에서 데이터베이스(혹은 다른 외부 저장소)로 데이터를 보내는 connector


## Table 생성

```
sinkTest DB에 생성
create table users(     
id int primary key,     
user_id varchar(20),     
pwd varchar(20),     
name varchar(20));

#sourceTest DB에 생성
create table users(     
id int auto_increment primary key,     
user_id varchar(20),     
pwd varchar(20),     
name varchar(20));
```


## Sink connector 생성

```
http://192.168.2.52:30099/connectors

{
    "name": "mariadbSinkTest",
    "config": {
        "connector.class" : "io.confluent.connect.jdbc.JdbcSinkConnector",
        "connection.url" : "jdbc:mariadb://mariadb:3306/sinkTest",
        "connection.user" : "root",
        "connection.password" : "secret",
        
        #데이터베이스 테이블 자동생성 무조건 false 권장 
        "auto.create": "false",
        "auto.evolve": "false",
        "insert.mode": "upsert",
        
        #primary key 설정
        "pk.mode": "record_value",
        "pk.field": "id",
        "table.name.format":"users",
        
        #topic 설정
        "topics": "mariadb-connector-users",
        "key.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter": "org.apache.kafka.connect.json.JsonConverter",
        "value.converter.schemas.enable": "true",
        "key.converter.schemas.enable": "true",
        "max.tasks": 1
    }
}
```

## sink table에서 확인

* source table에 데이터를 insert 하고 sink table에서 확인

```
MariaDB [sourceTest]> insert into users(user_id, pwd, name) values("test12123", "test2", "tes4131322");

MariaDB [sinkTest]> select * from users;
+----+-----------+-------+------------+
| id | user_id   | pwd   | name       |
+----+-----------+-------+------------+
|  1 | test12123 | test2 | tes411322  |
|  2 | test12123 | test2 | tes4131322 |
|  3 | test12123 | test2 | tes411322  |
|  4 | 3         | tt2   | 1322       |
+----+-----------+-------+------------+
```