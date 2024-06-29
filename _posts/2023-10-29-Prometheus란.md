---
key: jekyll-text-theme
title: 'Prometheus란?'
excerpt: 'Prometheus research 😎'
tags: [Prometheus]
---


# Prometheus란?

- 오픈소스 모니터링, 알람 툴킷
- TSDB를 수집

## 특징

- key/value 쌍으로 구성된 메트릭으로 데이터 모델 제시
- PromQL이라는 쿼리 언어가 존재
- 별도의 스토리지가 필요하지 않음

## 컴포넌트

- Server - metric 수집
- client library 제공
- 짤막한 작업을 위한 push gateway
- 알람 사용을 위한 alertmanager

## Architecture

👉 Prometheus Architecture : [https://prometheus.io/docs/introduction/overview/#architecture](https://prometheus.io/docs/introduction/overview/#architecture)


## 장점 & 단점

### 장점

- 마이크로서비스 운영 시 데이터 수집이 다차원이라 장점이 있다
- 프로메테우스 서버는 Standalone 운영 가능해서 부담이 덜하다
- 장애 문제 진단 등에 강점을 지닌다

### 단점

- 안정성을 위주로 만들어진 시스템이기 때문에 정확도를 요구할 땐 사용하기 힘들다
