---
key: jekyll-text-theme
title: 'PostgreSQL Vacuum  vs  Autovacuum'
excerpt: ' PostgreSQL Autovacuum Research😎'
tags: [PostgreSQL, Research]
---

# Vacuum이란?

## Vacuum은 휴지통 비우기

* PostgreSQL에서 delete된 데이터는 삭제되었어도 실제로 차지하는 공간은 그대로 남아있음 -> Data Bloat! / 이 공간을 차지하는 데이터를 dead tuple 이라고 부름.

**[참고] Postgres에서 update는 insert와 delete를 수행하기 때문에 update나 delete된 데이터는 dead tuple이 된다.**

* Postgres에서 dead tuple 을 삭제하여 Bloat 의 공간을 여유공간으로 바꾸어 주는것을 Vacuum 이라고 함.

## Vacuum은 Non-blocking 작업

* 데이터를 읽고 쓰는데 영향을 주는 Lock을 Exclusive Lock이라고 함.
* Vacuum은 데이터에 Exclusive Lock을 잡지 않아서 트렌젝션에서 데이터를 읽고 쓰는데 영향을 주지 않음. -> Non-blocking
* 따라서 여러 트렌젝션에서 쓰기작업을 수행하는 테이블에도 vacuum을 병렬로 수행할 수 있음.하지만 vacuum이 실행되면 많은 I/O 트래픽을 유발하기 때문에 다른 트랜젝션에 성능을 저하시킬 수 있다는 단점이 있음.

## Dead tuple -> Free space

* 참조되지 않는 Dead tuple은 vacuum을 통해 삭제되고 비워진 공간은 예비공간으로 남게 됨.
* Database마다 정해진 격리수준이 있고, 격리수준에 따라 지워진 데이터더라도 아직 트랜젝션에서 참조가 될 수 있음. 하지만 참조가 없는 데이터는 Vacuum의 대상!
* Vacuum으로 깨끗해진 데이터 공간은 디스크로 즉시 반환되지 않고 신규데이터를 위한 예비공간으로 남아있게 됨.

**[참고]**
* 테이블의 PK기준 앞이나 중간 일부분이 지워지면 예비공간으로 남아 있음. 하지만, 테이블 전체가 지워지거나 특정 PK값 이후로 데이터가 모두 삭제된 경우에는 예비공간으로 남지 않고 파일 시스템으로 반환됨. 예비공간을 모두 파일시스템으로 반환하고 데이터를 재구성하고 싶으면 VACUUM FULL 을 수행할 수 있음.


# Autovacuum

## Autovacuum = Vacuum + Analyze

* Autovacuum은 dead tuple이 차지하던 공간을 사용가능한 공간으로 변경해주는 vacuum작업과 테이블의 통계정보를 갱신하는 analyze 작업을 동시에 수행함.
* `auto-` 라는 이름에서 알 수 있듯 자동으로 실행되며 실행되는 기준을 가지고 있음. 이 기준을 threshold (문턱값) 이라고 부름.
* Vacuum과 Analyze 두 가지 작업을 동시에 수행하기 때문에 실행기준 또한 vacuum과 analyze 각각 정할 수 있음.

## Threshold

* Threshold(문턱값) = 전체 데이터 수 * 비율 + 최소문턱값
	* 전체 데이터 수 * 비율 : dead tuple이 전체 데이터에서 몇 퍼센트가 될 때 vacuum이 실행될 것인지 의미
	* 최소문턱값 : dead tuple이 최소한 몇 개 이상될 때 실행될 것인가에 대한 고정값을 의미

* 다음과 같이 vacuum과 analyze가 실행되는 수식은 동일하고, 설정하는 변수만 다름.

```
-- vacuum threshold  
autovacuum_vacuum_scale_factor * number of tuples + autovacuum_vacuum_threadhold  
-- analyze threshold 
autovacuum_analyze_scale_factor * number of tuples + autovacuum_analyze_threshold
```

* 해당 값은 전역 설정이 가능하며, 테이블별 상세 설정도 가능함.
* 전역 설정은 간편하지만 테이블에 따라 볼륨이 다르기 때문에 튜닝을 위해서는 테이블별 설정이 도움이 됨.

```
# 전역 설정
alter system set autovacuum_vacuum_scale_factor = 0.1;

# 확인 쿼리
select * from pg_settings where where name = 'autovacuum_vacuum_scale_factor';

# 개별설정
alter table {{schema.table_name}} set (autovacuum_vacuum_scale_factor=0.1);

# 확인쿼리
select relname, reloptions from pg_class where relname={{table_name}};

# 설정 초기화
alter table {{schema.table_name}} reset (autovacuum_vacuum_scale_factor);
```
<br>

> **REFERENCE**
> * [https://americanopeople.tistory.com/370](https://americanopeople.tistory.com/370)
> * [https://nrise.github.io/posts/postgresql-autovacuum/?fbclid=IwAR3axiM0p1cohJVV-rz0Mw-HUaU0vCsP8gi5M8QqN6FXhMWQQfKa38osRS0](https://nrise.github.io/posts/postgresql-autovacuum/?fbclid=IwAR3axiM0p1cohJVV-rz0Mw-HUaU0vCsP8gi5M8QqN6FXhMWQQfKa38osRS0)
> * [https://www.enterprisedb.com/postgres-tutorials/how-tune-postgresql-memory](https://www.enterprisedb.com/postgres-tutorials/how-tune-postgresql-memory)
> * [https://www.percona.com/blog/2018/08/10/tuning-autovacuum-in-postgresql-and-autovacuum-internals/](https://www.percona.com/blog/2018/08/10/tuning-autovacuum-in-postgresql-and-autovacuum-internals/)