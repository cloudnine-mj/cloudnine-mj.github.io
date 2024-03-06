---
key: jekyll-text-theme
title: 'PostgreSQL Autovacuum'
excerpt: ' PostgreSQL Autovacuum Research😎'
tags: [PostgreSQL]
---


:star: PostgreSQL DB로 업무하다가 간단한 SQL 쿼리도 시간이 엄청 오래 걸릴만큼 처리 속도가 느리고, CPU 사용률이 너무 높아져서 원인을 찾다가 AutoVacuum 이라는 것을 찾아냄.

# Autovacuum(Vacuum) 이란?

* PostgreSQL에는 Autovacuum 이라는 개념이 존재 (Oracle, MariaDB, MySQL, SQLserver 등에는 존재하지 않는 개념)
* PostgreSQL 사용 시 Vacuum 과 관련된 설정을 제대로 하지 않는다면 데이터베이스의 트랜잭션이 증가했을 때, 성능이 느려지는 현상을 겪을 수 있음.
* PostgreSQL 을 안정적으로 운용하기 위해서는 Autovacuum에 대해 알아야 함.


# Autovacuum이 동작하는 상황

1. XID wraparound를 방지하기 위해 XID를 고정할 때 (XID가 임계점에 도달할 경우 강제로 동작)
2. 임계점 이상으로 늘어난 dead tuple 들을 제거하여 FSM(Free Space Map) 으로 반환하고자 할 때

# Dead Tuple 이란?

