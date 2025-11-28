---
key: jekyll-text-theme
title: 'Airflow와 Prometheus 연동'
excerpt: ' Airflow와 Prometheus 연동으로 모니터링 자동화하기 😎'
tags: [Airflow, Prometheus]
---

# Airflow와 Prometheus 연동

## 개념

* Prometheus는 시계열 데이터베이스 기반의 모니터링 시스템
* Airflow의 메트릭을 Prometheus로 수집하고 Grafana로 시각화하여 파이프라인 상태를 실시간으로 모니터링할 수 있음.

## 설치

```bash
# Airflow에 StatsD Exporter 설정
pip install apache-airflow[statsd]

# airflow.cfg 수정
cat >> ~/airflow/airflow.cfg <<EOF
[metrics]
statsd_on = True
statsd_host = localhost
statsd_port = 8125
statsd_prefix = airflow
EOF

# Prometheus StatsD Exporter 설치 (Docker)
docker run -d \
  --name statsd-exporter \
  -p 9102:9102 \
  -p 8125:9125/udp \
  prom/statsd-exporter:latest \
  --statsd.mapping-config=/tmp/statsd_mapping.yml

# Prometheus 설치
cat > prometheus.yml <<EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'airflow'
    static_configs:
      - targets: ['localhost:9102']
EOF

docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus:latest

# Grafana 설치
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana:latest
```

## 원리

1. Airflow가 **StatsD 형식**으로 메트릭을 전송
2. **StatsD Exporter**가 메트릭을 Prometheus 형식으로 변환함.
3. **Prometheus**가 주기적으로 메트릭을 스크랩하여 저장함.
4. **Grafana**가 Prometheus를 데이터 소스로 사용하여 시각화함.

## 코드

* 실무에서 활용했던 코드

### StatsD Mapping 설정

```yaml
# statsd_mapping.yml
mappings:
  # DAG 실행 시간
  - match: "airflow.dag_processing.last_duration.*"
    name: "airflow_dag_processing_duration"
    labels:
      dag_file: "$1"
  
  # Task 성공/실패
  - match: "airflow.task_instance_finished.*.*.*.*"
    name: "airflow_task_finished"
    labels:
      dag_id: "$1"
      task_id: "$2"
      state: "$3"
  
  # Scheduler 메트릭
  - match: "airflow.scheduler.tasks.running"
    name: "airflow_scheduler_tasks_running"
  
  - match: "airflow.scheduler.tasks.starving"
    name: "airflow_scheduler_tasks_starving"
```

### Custom Metric 추가


```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.stats import Stats
from datetime import datetime
import time

def process_data_with_metrics(**context):
    """메트릭을 수집하는 데이터 처리 함수"""
    
    # 시작 시간 기록
    start_time = time.time()
    
    try:
        # 데이터 처리 로직
        records_processed = 0
        
        for i in range(1000):
            # 실제 처리 로직
            records_processed += 1
            
            # 100개 단위로 메트릭 전송
            if records_processed % 100 == 0:
                Stats.incr('custom.records_processed', 100)
        
        # 처리 완료
        processing_time = time.time() - start_time
        Stats.timing('custom.processing_duration', processing_time)
        Stats.incr('custom.processing_success')
        
        return {'status': 'success', 'records': records_processed}
        
    except Exception as e:
        Stats.incr('custom.processing_failure')
        Stats.incr(f'custom.error.{type(e).__name__}')
        raise

dag = DAG(
    'monitored_pipeline',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@hourly',
    catchup=False,
)

task = PythonOperator(
    task_id='process_data',
    python_callable=process_data_with_metrics,
    dag=dag,
)
```

### Grafana 대시보드 JSON


```json
{
  "dashboard": {
    "title": "Airflow Monitoring",
    "panels": [
      {
        "title": "DAG Success Rate",
        "targets": [
          {
            "expr": "sum(rate(airflow_task_finished{state='success'}[5m])) / sum(rate(airflow_task_finished[5m]))"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Task Duration by DAG",
        "targets": [
          {
            "expr": "avg(airflow_dag_processing_duration) by (dag_file)"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Running Tasks",
        "targets": [
          {
            "expr": "airflow_scheduler_tasks_running"
          }
        ],
        "type": "stat"
      }
    ]
  }
}
```

## 적용할 때 고려했던 점

1. **알람 설정**: Prometheus AlertManager를 사용하여 DAG 실패, Task 지연 등에 대한 알람을 구성해야 함.
2. **커스텀 메트릭**: 회사 메트릭 로직에 맞는 메트릭(처리된 레코드 수, API 호출 수 등)을 추가해야 함.
3. **대시보드 템플릿**: 팀 별로 표준 대시보드 템플릿을 만들어 일관성을 유지해야 함.
4. **보존 정책**: Prometheus 데이터 보존 기간을 적절히 설정하여 스토리지를 관리해야 함.
