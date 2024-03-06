---
key: jekyll-text-theme
title: 'PostgreSQL Autovacuum'
excerpt: ' PostgreSQL Autovacuum Research😎'
tags: [PostgreSQL]
---


:star: PostgreSQL DB로 업무하다가 간단한 SQL select 쿼리도 시간이 엄청 오래 걸릴만큼 처리 속도가 느리고, CPU 사용률이 너무 높아져서 원인을 찾다가 Autovacuum 이라는 것을 찾아냄.

# Autovacuum이란?

* PostgreSQL에는 Autovacuum 이라는 개념이 존재 (Oracle, MariaDB, MySQL, SQLserver 등에는 존재하지 않는 개념)
* PostgreSQL 사용 시 Vacuum 과 관련된 설정을 제대로 하지 않는다면 데이터베이스 트랜잭션이 증가했을 때, 성능이 느려지는 현상을 겪을 수 있음.
* PostgreSQL 을 안정적으로 운용하기 위해서는 Autovacuum에 대해 알아야 함.


# Autovacuum이 동작하는 상황

1. XID wraparound를 방지하기 위해 XID를 고정할 때 (XID가 임계점에 도달할 경우 강제로 동작)
2. 임계점 이상으로 늘어난 dead tuple 들을 제거하여 FSM(Free Space Map) 으로 반환하고자 할 때
<br>
> **XID (Transaction ID)**
> DBMS에서 트랜잭션이 발생한 시점을 식별하기 위한 정보 (일종의 시간정보)
> XID는 트랜잭션이 일어날 때마다 하나씩 증가하며, MVCC 모델의 구현 및 읽기 일관성을 위해 사용

# Dead Tuple 이란?
* PostgreSQL에서 모든 데이터는 tuple 형태로 저장
* tuple -> live tuple, dead tuple
* dead tuple은 더이상 사용(참조)되지 않는 tuple
* dead tuple은 PostgreSQL이 MVCC를 구현한 방법으로 인해 발생

> **MVCC (다중 버전 동시성 제어)**
> *  동시 접근을 허용하는 데이터베이스에서 동시성을 제어하기 위해 사용하는 방법
> * MVCC에서 데이터에 접근하는 사용자는 접근한 시점에 데이터베이스의 Snapshot을 읽는다. 이 snapshot 데이터에 대한 변경이 완료(commit)될 때 까지의 변경사항은 다른 데이터베이스 사용자가 볼 수 없다.
> * 사용자가 업데이트하면 이전의 데이터를 덮어 씌우는게 아니라 새로운 버전의 데이터를 UNDO 영역에 생성한다. 대신 이전 버전의 데이터와 비교해서 변경된 내용을 기록한다. 이렇게 해서 하나의 데이터에 대해 여러 버전의 데이터가 존재하게 되고, 사용자는 마지막 버전의 데이터를 읽게 된다.

<br>
> **MVCC 특징**
> * 일반적인 RDBMS보다 매우 빠르게 작동
> * 사용하지 않는 데이터가 계속 쌓이게 되므로 데이터를 정리하는 시스템이 필요
> * 데이터 버전이 충돌하면 애플리케이션 영역에서 이러한 문제를 해결해야 함.
> * UNDO 블록 I/O, CR Copy 생성, CR 블록 캐싱 같은 부가적인 작업의 오버헤드 발생

<br>

* 특정 column 혹은 row를 업데이트하는 트랜잭션이 수행될 경우 PostgreSQL은 MVCC 지원을 위해 다음과 같이 동작함.
	* FSM(Free Space Map)에 여유가 있는지 확인 (없다면, FSM을 추가적으로 확보함)
	* FSM의 빈 공간에 업데이트 될 데이터를 기록 -> 이 때 새로운 tuple이 추가됨
	* 기록이 완료되면, 기존 column 또는 row를 가리키는 포인터를 새로 기록된 tuple로 변경함
	* 업데이트 이전 정보가 기록된 공간은 더이상 참조되지 않게 함. -> 이 때 참조되지 않는 tuple을 **dead tuple** 이라고 함.
	* 일련의 과정에서 생성된 dead tuple 은 참조가 되지 않을 뿐 아니라 무의미하게 저장공간만 낭비하고 있는 상태가 됨. 그리고 이런 dead tuple 이 점유하고 있는 공간을 정리하여 FSM 으로 반환하여 재사용 가능하도록 하는 작업을 바로 "vacuum"이라고 함.

### 요약

* PostgreSQL 의 MVCC 구현체는 update/delete 트랜잭션이 일어날 때 dead tuple 을 남김.
* dead tuple 을 정리하기 위해 Vacuum 이라는 task 가 만들어지게 됨.
* Vacuum 명령어는 수동으로 구동됨. 그리고 Vacuum 이 수행중일 때 해당 테이블은 lock 이 걸리며 모든 트랜잭션이 거부됨. :star:
* 테이블에 lock 을 걸지 않으면서, 정기적으로, 그리고 자동으로 vacuuming 을 수행하는 Autovacuum 이 필요함.
