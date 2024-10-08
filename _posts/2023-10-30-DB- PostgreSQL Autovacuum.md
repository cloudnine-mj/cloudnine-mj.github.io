---
key: jekyll-text-theme
title: 'PostgreSQL Vacuum'
excerpt: ' PostgreSQL Autovacuum Research😎'
tags: [PostgreSQL, DB]
---


:star: PostgreSQL DB로 업무하다가 간단한 SQL select 쿼리도 시간이 엄청 오래 걸릴만큼 처리 속도가 느리고, CPU 사용률이 너무 높아져서(데이터 수집 주기에는 90%까지 올라감...) 원인을 찾다가 Vacuum 이라는 것을 찾아냄.

# PostgreSQL Vacuum

## Vacuum이란?

* PostgreSQL에는 Autovacuum 이라는 개념이 존재 (Oracle, MariaDB, MySQL, SQLserver 등에는 존재하지 않는 개념)
* Vacuum은 PostgreSQL의 진공청소기 역할을 하는 동작
* PostgreSQL 사용 시 Vacuum 과 관련된 설정을 제대로 안하면, 데이터베이스 트랜잭션이 증가했을 때, 성능이 느려지는 현상을 겪을 수 있음.
* PostgreSQL을 안정적으로 운용하기 위해서는 Autovacuum에 대해 알아야 함.


## Vacuum이 동작하는 상황

1. XID wraparound를 방지하기 위해 XID를 고정할 때 (XID가 임계점에 도달할 경우 강제로 동작)
2. 임계점 이상으로 늘어난 dead tuple 들을 제거하여 FSM(Free Space Map) 으로 반환하고자 할 때
<br>
> **XID (Transaction ID)**
> DBMS에서 트랜잭션이 발생한 시점을 식별하기 위한 정보 (일종의 시간정보)
> XID는 트랜잭션이 일어날 때마다 하나씩 증가하며, MVCC 모델의 구현 및 읽기 일관성을 위해 사용

## Dead Tuple 이란?
* PostgreSQL에서 모든 데이터는 tuple 형태로 저장 (PostgreSQL에서는 Record 대신 Tuple이라는 용어 사용)
* tuple -> live tuple, dead tuple
* dead tuple은 더이상 사용(참조)되지 않는 tuple
* PostgreSQL의 MVCC로 인해 dead tuple이 발생

> **MVCC (다중 버전 동시성 제어)**
> *  동시 접근을 허용하는 데이터베이스에서 동시성을 제어하기 위해 사용하는 방법
> * MVCC에서 데이터에 접근하는 사용자는 접근한 시점에 데이터베이스의 Snapshot을 읽음. 이 snapshot 데이터에 대한 변경이 완료(commit)될 때 까지의 변경사항은 다른 데이터베이스 사용자가 볼 수 없음.
> * 사용자가 업데이트하면 이전의 데이터를 덮어 씌우는게 아니라 새로운 버전의 데이터를 UNDO 영역에 생성한다. 대신 이전 버전의 데이터와 비교해서 변경된 내용을 기록한다. 이렇게 해서 하나의 데이터에 대해 여러 버전의 데이터가 존재하게 되고, 사용자는 마지막 버전의 데이터를 읽게 된다.

<br>
> **MVCC 특징**
> * 일반적인 RDBMS보다 매우 빠르게 작동
> * 사용하지 않는 데이터가 계속 쌓이게 되므로 데이터를 정리하는 시스템이 필요
> * 데이터 버전이 충돌하면 애플리케이션 영역에서 이러한 문제를 해결해야 함.
> * UNDO 블록 I/O, CR Copy 생성, CR 블록 캐싱 같은 부가적인 작업의 오버헤드 발생

<br>

* PostgreSQL은 데이터 페이지 내에 변경되기 이전 Tuple과 변경된 신규 Tuple을 같은 page에 저장하고 Tuple별로 생성된 시점과 변경된 시점을 기록 및 비교하는 방식으로 MVCC를 제공함.
* 특정 column 혹은 row를 업데이트하는 트랜잭션이 수행될 경우 PostgreSQL은 MVCC 지원을 위해 다음과 같이 동작함.
	* FSM(Free Space Map)에 여유가 있는지 확인 (없다면, FSM을 추가적으로 확보함)
	* FSM의 빈 공간에 업데이트 될 데이터를 기록 -> 이 때 새로운 tuple이 추가됨
	* 기록이 완료되면, 기존 column 또는 row를 가리키는 포인터를 새로 기록된 tuple로 변경함
	* 업데이트 이전 정보가 기록된 공간은 더이상 참조되지 않게 함. -> 이 때 참조되지 않는 tuple을 **dead tuple** 이라고 함.
	* 일련의 과정에서 생성된 dead tuple 은 참조가 되지 않을 뿐 아니라 무의미하게 저장공간만 낭비하고 있는 상태가 됨. 그리고 이런 dead tuple 이 점유하고 있는 공간을 정리하여 FSM 으로 반환하여 재사용 가능하도록 하는 작업을 바로 "vacuum"이라고 함.
* Tuple이 생성되거나 변경된 시점을 각 Tuple 내 xmin, xmax라는 메타데이터 field에 기록하여 어떤 Tuple을 읽을 수 있는지 버전 관리를 하게 됨.
  - **xmin** – Tuple을 insert하거나 update하는 시점의 Transaction ID를 갖는 메타데이터
    Insert의 경우 insert된 신규 Tuple의 xmin에 해당 시점의 Transaction ID가 할당되고
    UPDATE의 경우 update된 신규 Tuple의 xmin에 해당 시점의 Transaction ID가 할당됨.
  - **xmax** – Tuple을 delete하거나 update하는 시점의 Transaction ID를 갖는 메타데이터
    Delete의 경우 변경되기 이전 tupe의 xmax에 해당 시점의 Transaction ID가 할당됨.
    UPDATE의 경우 변경되기 이전 Tuple의 xmax와 update된 신규 Tuple의 xmin에는 해당 시점의 Transaction ID가 할당되고, update된 신규 Tuple의 xmax에는 NULL이 할당됨.

### 요약

* PostgreSQL 의 MVCC 구현체는 update/delete 트랜잭션이 일어날 때 dead tuple 을 남김.
* dead tuple 을 정리하기 위해 Vacuum 이라는 task가 만들어지게 됨.
* Vacuum 명령어는 수동으로 구동됨. 그리고 Vacuum 이 수행중일 때 해당 테이블은 lock 이 걸리며 모든 트랜잭션이 거부됨. :star:
* 테이블에 lock 을 걸지 않으면서, 정기적으로, 그리고 자동으로 vacuuming 을 수행하는 Autovacuum 이 필요함.



## Dead Tuple이 야기하는 문제들

* PostgreSQL 에서 update 는 사실 상 insert 와 동일한 동작이라 볼 수 있으며, delete 역시 수행하더라도 해당 데이터가 저장된 tuple 은 Vacuum 없이는 FSM 으로 반환되지 않고 저장소에서 삭제되지도 않음.
* PostgreSQL 에서 Data Bloat 즉,  데이터베이스의 저장 공간이 늘어나는 부작용을 만들어 냄.
* 가장 큰 문제는 저장 공간이 무한정 늘어난다는 것.
* update/delete 트랜잭션이 자주 일어나면 일어날수록, 저장공간 사용량은 급속도로 불어나며 이로 인해 추가적인 문제점이 발생하게 됨.
	* PostgreSQL이 select 트랜잭션을 수행할 때 :  live tuple을 디스크에서 읽어들일 때 일정한 용량의 chunk 단위로 파일을 읽어냄.
	* 만일 이 때 읽어들인 chunk 에 정리되지 않은 dead tuple 이 포함 되어 있을 경우 원하는 live tuple 을 읽기 위해 더 많은 디스크 I/O 가 발생하게 됨. -> 증가된 디스크 I/O 는 결과적으로 select 트랜잭션의 성능 저하를 가져옴.
* PostgreSQL 은 주기적으로 테이블의 통계정보를 갱신하는 작업을 수행하여 이를 통해 최적의 쿼리 계획을 수립함.
	* dead tuple 로 인해 쿼리 성능이 저하되는 경우가 자주 생기면, 통계 수집기는 인덱스가 멀쩡히 있음에도 인덱스를 사용하지 말라는 황당한 판단을 내리는 경우도 발생함.

> **Data Bloat**
> Bloat은 빈번한 업데이트 및 삭제로 인해 테이블 및 인덱스에 누적되는 불필요한 데이터. Bloat을 사용하면 데이터베이스 크기가 예상보다 커지고 쿼리 성능에 영향을 줄 수 있음.

###  쉽게 가보자.

* 다른 DBMS에서도 데이터를 읽을 때 block 단위로 메모리에 올리듯이 PostgreSQL도 page 단위로 읽어와 메모리에 올림.
* 다만 PostgreSQL에서는 MVCC 구현 방식의 특성상 Dead Tuple도 같은 page 내에 저장되기 때문에 page에 live Tuple뿐만 아니라 불필요한 Dead Tuple도 포함되어 있음.
* page의 크기는 default 8KB로 한정되어 있고 Dead Tuple이 쓸모없이 공간만 차지하기 때문에
필요한 live Tuple을 읽기 위해서는 더 많은 page를 읽어야 하고 이는 곧 더 많은 디스크 I/O 발생으로 이어지게 됨.

* 즉,  dead tuple이 없는 상태에서는 id 1~8의 데이터를 읽기 위해 단 2개의 페이지만 읽으면 되는데 dead tuple이 많은 상태에서는 똑같은 데이터를 읽어도 2배의 페이지를 읽어야 한다는 것.

* Dead Tuple은 테이블 공간의 비효율적 사용으로 인한 디스크 사용률 증가 이슈뿐만 아니라
쿼리 성능에도 악영향을 끼치게 되는 PostgreSQL의 골칫덩이...

<br>PostgreSQL의 기본 설정은 최고의 성능을 내기 보다는 가능한 다양한 기기에서 잘 동작할 수 있도록 보수적으로 잡혀있는 것 같다.

