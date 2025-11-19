---
key: jekyll-text-theme
title: 'Prometheus Research'
excerpt: 'Prometheus란 무엇인가 😎'
tags: [Prometheus]
---

# Prometheus란?

* Prometheus는 오픈소스 모니터링 및 알림 도구임. 시계열 데이터베이스를 기반으로 메트릭을 수집하고 저장함. Pull 방식으로 타겟 시스템의 메트릭을 주기적으로 수집하며, PromQL이라는 강력한 쿼리 언어를 제공함.

## Exporter 개념

* Exporter는 Prometheus가 메트릭을 수집할 수 있도록 데이터를 노출하는 에이전트임. 다양한 시스템과 애플리케이션의 메트릭을 Prometheus 포맷으로 변환하여 제공함.

### 주요 Exporter 종류

| Exporter            | 설명                                 |
| ------------------- | ------------------------------------ |
| Node Exporter       | 서버의 하드웨어 및 OS 메트릭 수집    |
| MySQL Exporter      | MySQL 데이터베이스 메트릭 수집       |
| PostgreSQL Exporter | PostgreSQL 데이터베이스 메트릭 수집  |
| Redis Exporter      | Redis 메트릭 수집                    |
| Custom Exporter     | 사용자 정의 애플리케이션 메트릭 수집 |

## 1. Prometheus 설치 및 설정

### prometheus.yml 설정

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'postgresql_exporter'
    static_configs:
      - targets: ['localhost:9187']

  - job_name: 'custom_app'
    static_configs:
      - targets: ['localhost:8000']
    scrape_interval: 30s
```

## 2. Node Exporter 설치

```bash
# Node Exporter 다운로드
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz

# 압축 해제 및 실행
tar xvfz node_exporter-1.6.0.linux-amd64.tar.gz
cd node_exporter-1.6.0.linux-amd64
./node_exporter
```

## 3. PostgreSQL Exporter 설정

```bash
# PostgreSQL Exporter 다운로드
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.13.0/postgres_exporter-0.13.0.linux-amd64.tar.gz

# 압축 해제
tar xvfz postgres_exporter-0.13.0.linux-amd64.tar.gz
cd postgres_exporter-0.13.0.linux-amd64

# 환경변수 설정
export DATA_SOURCE_NAME="postgresql://postgres:password@localhost:5432/postgres?sslmode=disable"

# 실행
./postgres_exporter
```

## 4. Custom Exporter 개발 (Python)

```python
from prometheus_client import start_http_server, Gauge, Counter, Histogram
import time
import random

# 메트릭 정의
REQUEST_COUNT = Counter('app_request_total', 'Total number of requests')
REQUEST_DURATION = Histogram('app_request_duration_seconds', 'Request duration in seconds')
ACTIVE_USERS = Gauge('app_active_users', 'Number of active users')
DB_CONNECTIONS = Gauge('app_db_connections', 'Number of database connections')

def collect_metrics():
    """메트릭 수집 함수"""
    while True:
        # 요청 카운트 증가
        REQUEST_COUNT.inc(random.randint(1, 10))
        
        # 활성 사용자 수 업데이트
        ACTIVE_USERS.set(random.randint(100, 500))
        
        # DB 커넥션 수 업데이트
        DB_CONNECTIONS.set(random.randint(10, 50))
        
        # 요청 처리 시간 기록
        with REQUEST_DURATION.time():
            time.sleep(random.uniform(0.1, 0.5))
        
        time.sleep(5)

if __name__ == '__main__':
    # HTTP 서버 시작 (포트 8000)
    start_http_server(8000)
    print("Exporter running on port 8000")
    collect_metrics()
```

## 5. 메트릭 데이터 가공 및 표준화

```python
import requests
from datetime import datetime

def fetch_prometheus_metrics(prometheus_url, query):
    """Prometheus에서 메트릭 조회"""
    response = requests.get(
        f"{prometheus_url}/api/v1/query",
        params={'query': query}
    )
    return response.json()

def transform_to_standard_format(raw_metrics):
    """전사 표준 포맷으로 변환"""
    standard_metrics = []
    
    for metric in raw_metrics['data']['result']:
        standard_metric = {
            'metric_name': metric['metric'].get('__name__'),
            'metric_value': float(metric['value'][1]),
            'timestamp': datetime.fromtimestamp(metric['value'][0]).isoformat(),
            'labels': {k: v for k, v in metric['metric'].items() if k != '__name__'},
            'source': 'prometheus',
            'data_type': 'metric'
        }
        standard_metrics.append(standard_metric)
    
    return standard_metrics

# 사용 예시
prometheus_url = "http://localhost:9090"
query = "up"
raw_data = fetch_prometheus_metrics(prometheus_url, query)
standard_data = transform_to_standard_format(raw_data)
```

## 6. PromQL을 활용한 메트릭 쿼리

```promql
# CPU 사용률 조회
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 메모리 사용률 조회
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# PostgreSQL 활성 커넥션 수
pg_stat_database_numbackends

# 초당 요청 수 (RPS)
rate(app_request_total[1m])

# 95 percentile 응답 시간
histogram_quantile(0.95, rate(app_request_duration_seconds_bucket[5m]))
```

## 7. Grafana 연동

```yaml
# grafana datasource 설정 (datasources.yml)
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    editable: true
```

## 8. Alert Rule 설정

```yaml
# alert_rules.yml
groups:
  - name: example_alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for 5 minutes"

      - alert: DatabaseConnectionHigh
        expr: pg_stat_database_numbackends > 100
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "High database connections"
          description: "PostgreSQL connections exceed 100"
```

## 9. 메트릭 저장 및 관리

```python
import psycopg2
from datetime import datetime
import json

def store_metrics_to_db(metrics, db_config):
    """표준화된 메트릭을 데이터베이스에 저장"""
    conn = psycopg2.connect(**db_config)
    cursor = conn.cursor()
    
    # 메트릭 테이블 생성
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS metrics_data (
            id SERIAL PRIMARY KEY,
            metric_name VARCHAR(255),
            metric_value FLOAT,
            timestamp TIMESTAMP,
            labels JSONB,
            source VARCHAR(100),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    
    # 메트릭 데이터 삽입
    for metric in metrics:
        cursor.execute("""
            INSERT INTO metrics_data 
            (metric_name, metric_value, timestamp, labels, source)
            VALUES (%s, %s, %s, %s, %s)
        """, (
            metric['metric_name'],
            metric['metric_value'],
            metric['timestamp'],
            json.dumps(metric['labels']),
            metric['source']
        ))
    
    conn.commit()
    cursor.close()
    conn.close()

# 사용 예시
db_config = {
    'host': 'localhost',
    'database': 'metrics_db',
    'user': 'postgres',
    'password': 'password'
}

store_metrics_to_db(standard_data, db_config)
```

## 10. 모니터링 대시보드 구축

```python
from flask import Flask, jsonify, render_template
import requests

app = Flask(__name__)
PROMETHEUS_URL = "http://localhost:9090"

@app.route('/api/metrics/summary')
def get_metrics_summary():
    """주요 메트릭 요약 정보 제공"""
    metrics = {
        'cpu_usage': get_metric('100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)'),
        'memory_usage': get_metric('(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100'),
        'active_connections': get_metric('pg_stat_database_numbackends'),
        'request_rate': get_metric('rate(app_request_total[1m])'),
    }
    return jsonify(metrics)

def get_metric(query):
    """Prometheus 쿼리 실행"""
    response = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query",
        params={'query': query}
    )
    data = response.json()
    if data['data']['result']:
        return float(data['data']['result'][0]['value'][1])
    return 0.0

@app.route('/dashboard')
def dashboard():
    """모니터링 대시보드 페이지"""
    return render_template('dashboard.html')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## 참고

* Prometheus는 단기 메트릭 저장에 최적화되어 있어서, 장기 보관이 필요한 경우 Thanos나 Victoria Metrics 같은 솔루션을 함께 사용하는 게 좋음. 또한 메트릭 수집 주기와 보관 기간은 시스템 리소스와 요구사항에 맞게 조정해야 함.