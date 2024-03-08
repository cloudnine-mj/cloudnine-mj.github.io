---
key: jekyll-text-theme
title: 'PostgreSQL Vacuum  vs  Autovacuum'
excerpt: ' PostgreSQL Autovacuum Research😎'
tags: [PostgreSQL]
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
