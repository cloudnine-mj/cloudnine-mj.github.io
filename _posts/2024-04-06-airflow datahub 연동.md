---
key: jekyll-text-theme
title: 'Airflow 파이프라인과 DataHub 연동하기'
excerpt: ' Datahub 연동!! 😎'
tags: [Datahub]
---



# Airflow 파이프라인과 DataHub 연동하기

## 개념

* Airflow DAG를 DataHub와 연동하면 데이터 파이프라인의 메타데이터를 자동으로 수집하고, 데이터 계보(lineage)를 시각화할 수 있음.
* Airflow의 Inlet/Outlet과 DataHub Lineage Backend를 사용하여 Task 간 데이터 흐름을 자동으로 추적함.

## 설치

```bash
# Airflow에 DataHub Provider 설치
pip install acryl-datahub[airflow]
pip install acryl-datahub-airflow-plugin

# 또는 requirements.txt에 추가
cat >> requirements.txt <<EOF
acryl-datahub[airflow]==0.12.0.0
acryl-datahub-airflow-plugin==0.12.0.0
EOF

pip install -r requirements.txt
```

## Airflow 설정

### airflow.cfg 수정

```ini
[lineage]
# DataHub Lineage Backend 활성화
backend = datahub_provider.lineage.datahub.DatahubLineageBackend

[datahub]
# DataHub 서버 정보 (환경변수로도 가능)
enabled = True
conn_id = datahub_rest_default
```

### Connection 설정

```python
# Airflow UI에서 Connection 추가
# Admin → Connections → +

# 또는 CLI로 추가
airflow connections add datahub_rest_default \
    --conn-type 'datahub-rest' \
    --conn-host 'http://localhost:8080' \
    --conn-extra '{"token": ""}'

# 또는 Python으로 추가
from airflow.models import Connection
from airflow.utils.db import create_session

conn = Connection(
    conn_id='datahub_rest_default',
    conn_type='datahub_rest',
    host='http://localhost:8080',
    extra='{"timeout": 30}'
)

with create_session() as session:
    session.add(conn)
    session.commit()
```

## 기본 Lineage 수집

### 예제 1: Inlet/Outlet 사용

~~~python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.lineage.entities import Table, File
from datetime import datetime

def extract_from_db(**context):
    """DB에서 데이터 추출"""
    print("Extracting data from MySQL")
    # 실제 추출 로직
    return {'status': 'success', 'rows': 1000}

def transform_data(**context):
    """데이터 변환"""
    print("Transforming data")
    return {'status': 'success', 'rows': 950}

def load_to_warehouse(**context):
    """데이터 웨어하우스에 적재"""
    print("Loading to Snowflake")
    return {'status': 'success'}

dag = DAG(
    'datahub_lineage_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
    tags=['datahub', 'lineage'],
)

# Task with Inlet (입력 데이터)
extract = PythonOperator(
    task_id='extract_data',
    python_callable=extract_from_db,
    # 입력: MySQL 테이블
    inlets=[
        Table(
            database='production',
            cluster='mysql_prod',
            name='raw_events',
        )
    ],
    # 출력: S3 파일
    outlets=[
        File(url='s3://data-lake/raw/events/{{ ds }}.parquet')
    ],
    dag=dag,
)

transform = PythonOperator(
    task_id='transform_data',
    python_callable=transform_data,
    inlets=[
        File(url='s3://data-lake/raw/events/{{ ds }}.parquet')
    ],
    outlets=[
        File(url='s3://data-lake/processed/events/{{ ds }}.parquet')
    ],
    dag=dag,
)

load = PythonOperator(
    task_id='load_to_warehouse',
    python_callable=load_to_warehouse,
    inlets=[
        File(url='s3://data-lake/processed/events/{{ ds }}.parquet')
    ],
    outlets=[
        Table(
            database='analytics',
            cluster='snowflake_prod',
            name='fact_events',
        )
    ],
    dag=dag,
)

extract >> transform >> load
```

#### DataHub에서 확인
```
1. DataHub UI (http://localhost:9002)
2. Pipelines 메뉴 클릭
3. "datahub_lineage_example" 검색
4. Lineage 탭에서 데이터 흐름 확인:

production.raw_events (MySQL)
         ↓
s3://data-lake/raw/events/*.parquet
         ↓
s3://data-lake/processed/events/*.parquet
         ↓
analytics.fact_events (Snowflake)
~~~

## 고급 Lineage 설정

### 예제 2: DataHub Emitter 직접 사용

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datahub.emitter.mce_builder import make_dataset_urn
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.schema_classes import DatasetLineageTypeClass, UpstreamClass, UpstreamLineageClass
from datetime import datetime

def process_with_lineage(**context):
    """Lineage 정보를 직접 전송"""
    
    # DataHub Emitter
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    # 다운스트림 데이터셋 (결과)
    downstream_urn = make_dataset_urn(
        platform='snowflake',
        name='analytics.user_summary',
        env='PROD'
    )
    
    # 업스트림 데이터셋 (소스)
    upstream_urns = [
        make_dataset_urn(
            platform='mysql',
            name='production.users',
            env='PROD'
        ),
        make_dataset_urn(
            platform='mysql',
            name='production.events',
            env='PROD'
        ),
    ]
    
    # Lineage 정보 구성
    upstream_lineage = UpstreamLineageClass(
        upstreams=[
            UpstreamClass(
                dataset=urn,
                type=DatasetLineageTypeClass.TRANSFORMED,
            )
            for urn in upstream_urns
        ]
    )
    
    # Lineage 전송
    emitter.emit_mcp(
        downstream_urn,
        'upstreamLineage',
        upstream_lineage
    )
    
    print(f"Lineage registered: {len(upstream_urns)} upstreams → {downstream_urn}")

dag = DAG(
    'advanced_lineage_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
)

task = PythonOperator(
    task_id='process_data',
    python_callable=process_with_lineage,
    dag=dag,
)
```

### 예제 3: SQL 기반 컬럼 레벨 Lineage

```python
from airflow import DAG
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datahub.emitter.mce_builder import make_dataset_urn
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.schema_classes import (
    FineGrainedLineageClass,
    FineGrainedLineageDownstreamTypeClass,
    FineGrainedLineageUpstreamTypeClass,
)
from datetime import datetime

# SQL 쿼리
CREATE_SUMMARY_SQL = """
CREATE TABLE analytics.user_summary AS
SELECT 
    u.user_id,
    u.name,
    u.email,
    COUNT(o.order_id) as order_count,
    SUM(o.amount) as total_spent
FROM production.users u
LEFT JOIN production.orders o ON u.user_id = o.customer_id
GROUP BY u.user_id, u.name, u.email
"""

def register_column_lineage(**context):
    """컬럼 레벨 Lineage 등록"""
    
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    downstream_urn = make_dataset_urn(
        platform='postgres',
        name='analytics.user_summary',
        env='PROD'
    )
    
    # 컬럼별 Lineage 정의
    column_lineages = [
        # user_id 컬럼
        FineGrainedLineageClass(
            upstreamType=FineGrainedLineageUpstreamTypeClass.FIELD_SET,
            upstreams=[
                'urn:li:schemaField:(urn:li:dataset:(urn:li:dataPlatform:postgres,production.users,PROD),user_id)'
            ],
            downstreamType=FineGrainedLineageDownstreamTypeClass.FIELD,
            downstreams=[
                'urn:li:schemaField:(urn:li:dataset:(urn:li:dataPlatform:postgres,analytics.user_summary,PROD),user_id)'
            ],
        ),
        # order_count 컬럼 (집계)
        FineGrainedLineageClass(
            upstreamType=FineGrainedLineageUpstreamTypeClass.FIELD_SET,
            upstreams=[
                'urn:li:schemaField:(urn:li:dataset:(urn:li:dataPlatform:postgres,production.orders,PROD),order_id)'
            ],
            downstreamType=FineGrainedLineageDownstreamTypeClass.FIELD,
            downstreams=[
                'urn:li:schemaField:(urn:li:dataset:(urn:li:dataPlatform:postgres,analytics.user_summary,PROD),order_count)'
            ],
            transformOperation='COUNT',
        ),
    ]
    
    # Lineage 전송
    for lineage in column_lineages:
        emitter.emit_mcp(downstream_urn, 'fineGrainedLineage', lineage)
    
    print("컬럼 레벨 Lineage 등록 완료")

dag = DAG(
    'column_lineage_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
)

create_summary = PostgresOperator(
    task_id='create_user_summary',
    postgres_conn_id='postgres_prod',
    sql=CREATE_SUMMARY_SQL,
    dag=dag,
)

register_lineage = PythonOperator(
    task_id='register_column_lineage',
    python_callable=register_column_lineage,
    dag=dag,
)

create_summary >> register_lineage
```

## DAG 메타데이터 자동 수집

### Airflow Ingestion recipe

```yaml
# airflow_recipe.yml
source:
  type: airflow
  config:
    # Airflow 연결 정보
    conn_id: airflow_db  # 또는 직접 지정
    # host_port: localhost:5432
    # database: airflow
    # username: airflow
    # password: airflow
    
    # 메타데이터 베이스 URL
    webserver_url: http://localhost:8080
    
    # 수집 옵션
    include_lineage: true
    capture_ownership_info: true
    capture_tags_info: true
    
    # DAG 필터
    dag_pattern:
      allow:
        - "prod_.*"
        - "etl_.*"
      deny:
        - "test_.*"
        - "dev_.*"

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```


```bash
# Ingestion 실행
datahub ingest -c airflow_recipe.yml

# 스케줄링 (Cron)
0 */6 * * * /usr/local/bin/datahub ingest -c /path/to/airflow_recipe.yml
```

## DataHub Plugin 활용

### DataHub Airflow Plugin 설치


```python
# airflow/plugins/datahub_plugin.py
from airflow.plugins_manager import AirflowPlugin
from datahub_provider.entities import Dataset, DataJob, DataFlow

class DataHubPlugin(AirflowPlugin):
    name = "datahub_plugin"
    hooks = []
    operators = []
    sensors = []
    macros = []
    executors = []
    admin_views = []
    flask_blueprints = []
    menu_links = []
```

### Custom Operator with DataHub



```python
from airflow.models import BaseOperator
from datahub.emitter.mce_builder import make_dataset_urn
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.schema_classes import DatasetPropertiesClass

class DataHubAwareOperator(BaseOperator):
    """DataHub 메타데이터를 자동으로 등록하는 Operator"""
    
    def __init__(
        self,
        input_datasets,
        output_datasets,
        *args,
        **kwargs
    ):
        super().__init__(*args, **kwargs)
        self.input_datasets = input_datasets
        self.output_datasets = output_datasets
    
    def execute(self, context):
        # 실제 작업 수행
        result = self._do_work(context)
        
        # DataHub에 메타데이터 등록
        self._register_to_datahub(context, result)
        
        return result
    
    def _do_work(self, context):
        """실제 작업 로직"""
        pass
    
    def _register_to_datahub(self, context, result):
        """DataHub에 메타데이터 등록"""
        emitter = DatahubRestEmitter('http://localhost:8080')
        
        for dataset_info in self.output_datasets:
            dataset_urn = make_dataset_urn(
                platform=dataset_info['platform'],
                name=dataset_info['name'],
                env='PROD'
            )
            
            properties = DatasetPropertiesClass(
                description=dataset_info.get('description', ''),
                customProperties={
                    'airflow_dag_id': context['dag'].dag_id,
                    'airflow_task_id': context['task'].task_id,
                    'execution_date': str(context['execution_date']),
                    'rows_processed': result.get('rows', 0),
                }
            )
            
            emitter.emit_mcp(dataset_urn, 'datasetProperties', properties)

# 사용 예제
task = DataHubAwareOperator(
    task_id='etl_task',
    input_datasets=[
        {'platform': 'mysql', 'name': 'production.orders'}
    ],
    output_datasets=[
        {
            'platform': 'snowflake',
            'name': 'analytics.daily_sales',
            'description': '일별 매출 집계 테이블'
        }
    ],
    dag=dag,
)
```

## 실시간 Lineage 업데이트


```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.apache.kafka.operators.produce import ProduceToTopicOperator
from datahub.emitter.kafka_emitter import DatahubKafkaEmitter
from datahub.metadata.schema_classes import UpstreamLineageClass
from datetime import datetime

def emit_lineage_to_kafka(**context):
    """Kafka로 Lineage 이벤트 전송"""
    
    # Kafka Emitter
    emitter = DatahubKafkaEmitter(
        bootstrap='localhost:9092',
        topic='MetadataChangeProposal_v1'
    )
    
    downstream_urn = make_dataset_urn(
        platform='snowflake',
        name='analytics.real_time_metrics',
        env='PROD'
    )
    
    upstream_lineage = UpstreamLineageClass(
        upstreams=[
            UpstreamClass(
                dataset=make_dataset_urn(
                    platform='kafka',
                    name='events.user_actions',
                    env='PROD'
                ),
                type=DatasetLineageTypeClass.TRANSFORMED,
            )
        ]
    )
    
    # Kafka로 전송 (비동기)
    emitter.emit_mcp(downstream_urn, 'upstreamLineage', upstream_lineage)
    emitter.flush()
    
    print("Lineage event sent to Kafka")

dag = DAG(
    'realtime_lineage_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@hourly',
    catchup=False,
)

task = PythonOperator(
    task_id='emit_lineage',
    python_callable=emit_lineage_to_kafka,
    dag=dag,
)
```

## 모니터링 및 검증


```python
# DAG가 DataHub에 등록되었는지 확인
from datahub.ingestion.graph.client import DatahubClientConfig, DataHubGraph

def verify_dag_registration():
    """DAG 등록 확인"""
    
    graph = DataHubGraph(
        config=DatahubClientConfig(server='http://localhost:8080')
    )
    
    # DAG 검색
    results = graph.search(
        entity_types=['dataFlow'],
        query='datahub_lineage_example',
    )
    
    for result in results:
        print(f"Found DAG: {result.entity_urn}")
        
        # Lineage 확인
        lineage = graph.get_lineage(
            entity_urn=result.entity_urn,
            direction='DOWNSTREAM',
            max_hops=3,
        )
        
        print(f"Downstream datasets: {len(lineage.edges)}")
        for edge in lineage.edges:
            print(f"  - {edge.destinationUrn}")

verify_dag_registration()
```

## 적용할 때 고려했던 점

1. **점진적 적용**: 모든 DAG를 한 번에 연동하지 말고, 중요한 파이프라인부터 시작
2. **Lineage 검증**: 자동 수집된 Lineage가 정확한지 주기적으로 확인
3. **성능 고려**: Lineage 전송이 Task 실행 시간에 영향을 주지 않도록 비동기 처리
4. **표준화**: 데이터셋 이름, 플랫폼 이름을 일관되게 사용
5. **문서화**: 각 Task의 입출력 데이터를 명확히 문서화
6. **모니터링**: DataHub Ingestion 실패 시 알림 설정