---
key: jekyll-text-theme
title: 'OpenLineage vs DataHub'
excerpt: ' Datahub Research 😎'
tags: [Datahub]
---



# OpenLineage vs DataHub

## 개념

* OpenLineage와 DataHub는 모두 메타데이터 관리와 데이터 계보 추적을 위한 오픈소스 프로젝트이지만, 목적과 접근 방식에서 차이가 존재함. 

## OpenLineage란?

**정의:** OpenLineage는 LF AI & Data Foundation의 프로젝트로, 데이터 계보를 수집하고 표준화하기 위한 오픈 표준입니다. 계보 메타데이터를 수집, 전송, 저장하는 표준 프레임워크를 제공합니다.

**핵심 특징**

1. 표준 스펙: JSON 기반의 계보 이벤트 표준
2. 경량: 계보 수집에 특화, 최소한의 오버헤드
3. 플러그인 아키텍처: 다양한 도구 통합 용이
4. 실시간: 작업 실행 중 실시간 계보 수집
5. 벤더 중립: 특정 도구에 종속되지 않음


## 아키텍처 비교

### OpenLineage 아키텍처

```
┌──────────────────────────────────────────┐
│         Data Processing Tools             │
│  (Airflow, Spark, dbt, Flink 등)         │
└──────────────────────────────────────────┘
                    │
                    │ OpenLineage Events
                    ▼
┌──────────────────────────────────────────┐
│       OpenLineage Client/SDK              │
│    (Python, Java, Scala 클라이언트)        │
└──────────────────────────────────────────┘
                    │
                    │ HTTP POST / Kafka
                    ▼
┌──────────────────────────────────────────┐
│         OpenLineage Backend               │
│          (Marquez 등)                     │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│         Storage & Visualization           │
│      (PostgreSQL, UI)                     │
└──────────────────────────────────────────┘
```

### DataHub 아키텍처

```
┌──────────────────────────────────────────┐
│           Data Sources                    │
│  (MySQL, Kafka, Airflow, Snowflake 등)   │
└──────────────────────────────────────────┘
                    │
                    │ Ingestion Framework
                    ▼
┌──────────────────────────────────────────┐
│         DataHub Ingestion                 │
│     (Pull-based + Push-based)            │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│         DataHub GMS + Frontend            │
│    (메타데이터 서비스 + UI)                 │
└──────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│    Storage (MySQL, Elasticsearch, Kafka) │
└──────────────────────────────────────────┘
```

## 상세 비교표

| 항목             | OpenLineage                          | DataHub                                                   |
|------------------|--------------------------------------|------------------------------------------------------------|
| **목적**          | 계보 수집 표준                        | 통합 메타데이터 플랫폼                                      |
| **범위**          | 데이터 계보 중심                      | 계보 + 스키마 + 거버넌스 + 검색                             |
| **접근 방식**      | Push (이벤트 전송)                   | Pull (주기적 수집) + Push                                   |
| **실시간성**       | 실시간 이벤트                        | 배치 + 실시간                                              |
| **메타데이터 모델** | 계보 이벤트 중심                     | Entity-Aspect 모델                                          |
| **UI**            | 별도 (예: Marquez 등)                | 통합 UI 제공                                               |
| **스키마 관리**     | 제한적                                | 상세한 스키마 메타데이터 제공                               |
| **검색**          | 제한적                                | 강력한 검색 엔진                                            |
| **거버넌스**       | 없음                                  | 태그, 오너, 정책(Tag, Owner, Policy)                       |
| **커뮤니티**       | 성장 중                               | 활발함                                                     |
| **학습 곡선**      | 낮음                                  | 중간-높음                                                  |
| **확장성**        | 높음 (경량 구조)                     | 높음 (마이크로서비스 기반)                                  |
| **설치 복잡도**    | 낮음                                  | 중간-높음                                                  |


## OpenLineage 상세

### OpenLineage 이벤트 구조

```json
{
  "eventType": "START",
  "eventTime": "2024-01-01T10:00:00.000Z",
  "run": {
    "runId": "d46e465b-d358-4d32-83d4-df660ff614dd"
  },
  "job": {
    "namespace": "airflow",
    "name": "etl_pipeline.extract_customers",
    "facets": {
      "documentation": {
        "_producer": "https://github.com/OpenLineage/OpenLineage/tree/0.18.0/client/python",
        "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/DocumentationJobFacet.json",
        "description": "Extract customer data from MySQL"
      }
    }
  },
  "inputs": [
    {
      "namespace": "mysql://production-db:3306",
      "name": "production.customers",
      "facets": {
        "schema": {
          "_producer": "https://github.com/OpenLineage/OpenLineage/tree/0.18.0/client/python",
          "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/SchemaDatasetFacet.json",
          "fields": [
            {
              "name": "customer_id",
              "type": "INTEGER"
            },
            {
              "name": "name",
              "type": "VARCHAR"
            }
          ]
        }
      }
    }
  ],
  "outputs": [
    {
      "namespace": "s3://data-lake",
      "name": "raw/customers.parquet",
      "facets": {
        "dataQuality": {
          "_producer": "https://github.com/OpenLineage/OpenLineage/tree/0.18.0/client/python",
          "_schemaURL": "https://openlineage.io/spec/facets/1-0-0/DataQualityMetricsInputDatasetFacet.json",
          "rowCount": 10000,
          "bytes": 1024000
        }
      }
    }
  ],
  "producer": "https://github.com/OpenLineage/OpenLineage/tree/0.18.0/integration/airflow"
}
```

### OpenLineage 사용 예제

**Airflow에서 OpenLineage 사용**

```python
# airflow_openlineage.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

# OpenLineage는 Airflow 2.3+에 기본 통합
# 환경변수로 설정
"""
export OPENLINEAGE_URL=http://marquez:5000
export OPENLINEAGE_NAMESPACE=production
"""

def extract_data(**context):
    """데이터 추출 - OpenLineage가 자동으로 계보 수집"""
    import pandas as pd
    from sqlalchemy import create_engine
    
    engine = create_engine('mysql://user:pass@host/db')
    df = pd.read_sql('SELECT * FROM customers', engine)
    
    # S3에 저장
    df.to_parquet('s3://bucket/raw/customers.parquet')
    
    return {'rows': len(df)}

dag = DAG(
    'openlineage_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
)

extract = PythonOperator(
    task_id='extract_customers',
    python_callable=extract_data,
    dag=dag,
)

# OpenLineage가 자동으로 수집:
# - Job: airflow.openlineage_example.extract_customers
# - Input: mysql://host/db.customers
# - Output: s3://bucket/raw/customers.parquet
# - 실행 시간, 상태, 에러 등
```

**Spark에서 OpenLineage 사용**

```python
# spark_openlineage.py
from pyspark.sql import SparkSession

# OpenLineage 설정
spark = SparkSession.builder \
    .appName("OpenLineage Example") \
    .config("spark.extraListeners", "io.openlineage.spark.agent.OpenLineageSparkListener") \
    .config("spark.openlineage.transport.type", "http") \
    .config("spark.openlineage.transport.url", "http://marquez:5000") \
    .config("spark.openlineage.namespace", "production") \
    .getOrCreate()

# Spark 작업 실행 - OpenLineage가 자동으로 계보 수집
df = spark.read.parquet("s3://bucket/raw/customers.parquet")

transformed = df.filter(df.status == "active") \
    .groupBy("country") \
    .agg({"order_count": "sum", "revenue": "sum"})

transformed.write.parquet("s3://bucket/processed/customer_summary.parquet")

# OpenLineage가 자동으로 수집:
# - Input: s3://bucket/raw/customers.parquet
# - Output: s3://bucket/processed/customer_summary.parquet
# - Transformations: filter, groupBy, agg
# - 실행 메트릭: rows processed, execution time 등
```

**Python SDK 직접 사용**

```python
# openlineage_sdk_example.py
from openlineage.client import OpenLineageClient
from openlineage.client.run import RunEvent, RunState, Run, Job
from openlineage.client.facet import (
    SqlJobFacet,
    SchemaDatasetFacet,
    SchemaField,
)
from datetime import datetime
import uuid

# OpenLineage 클라이언트
client = OpenLineageClient(url="http://marquez:5000")

# Run ID 생성
run_id = str(uuid.uuid4())

# Job 정의
job = Job(
    namespace="custom-pipeline",
    name="customer_etl",
    facets={
        "sql": SqlJobFacet(
            query="SELECT * FROM customers WHERE created_at > '2024-01-01'"
        )
    }
)

# Run 정의
run = Run(runId=run_id)

# START 이벤트
start_event = RunEvent(
    eventType=RunState.START,
    eventTime=datetime.now().isoformat(),
    run=run,
    job=job,
    producer="https://github.com/my-org/custom-pipeline",
    inputs=[
        {
            "namespace": "mysql://prod-db:3306",
            "name": "production.customers",
            "facets": {
                "schema": SchemaDatasetFacet(
                    fields=[
                        SchemaField(name="customer_id", type="INTEGER"),
                        SchemaField(name="name", type="VARCHAR"),
                        SchemaField(name="email", type="VARCHAR"),
                    ]
                )
            }
        }
    ],
    outputs=[
        {
            "namespace": "s3://data-lake",
            "name": "processed/customers_2024.parquet",
        }
    ]
)

client.emit(start_event)

try:
    # 실제 작업 수행
    process_customers()
    
    # COMPLETE 이벤트
    complete_event = RunEvent(
        eventType=RunState.COMPLETE,
        eventTime=datetime.now().isoformat(),
        run=run,
        job=job,
        producer="https://github.com/my-org/custom-pipeline",
    )
    client.emit(complete_event)
    
except Exception as e:
    # FAIL 이벤트
    fail_event = RunEvent(
        eventType=RunState.FAIL,
        eventTime=datetime.now().isoformat(),
        run=run,
        job=job,
        producer="https://github.com/my-org/custom-pipeline",
    )
    client.emit(fail_event)
```

## DataHub와 OpenLineage 통합

* DataHub는 OpenLineage 이벤트를 수신하여 메타데이터로 저장할 수 있음.

```yaml
# datahub_openlineage_integration.yml

# 1. OpenLineage → Kafka → DataHub
# OpenLineage 설정
export OPENLINEAGE_TRANSPORT_TYPE=kafka
export OPENLINEAGE_TRANSPORT_TOPIC=openlineage.events
export OPENLINEAGE_KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# 2. DataHub Actions로 OpenLineage 이벤트 소비
# actions_config.yml
source:
  type: kafka
  config:
    bootstrap: "localhost:9092"
    topic: "openlineage.events"
    consumer_group_id: "datahub-openlineage-consumer"

actions:
  - name: openlineage_to_datahub
    type: custom
    module: openlineage_transformer
    class: OpenLineageToDataHubAction
```

~~~python
# openlineage_transformer.py
from datahub_actions.action.action import Action
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.schema_classes import (
    UpstreamLineageClass,
    UpstreamClass,
    DatasetLineageTypeClass,
)

class OpenLineageToDataHubAction(Action):
    """OpenLineage 이벤트를 DataHub 메타데이터로 변환"""
    
    def __init__(self, config: dict):
        self.datahub_url = config.get('datahub_url', 'http://localhost:8080')
        self.emitter = DatahubRestEmitter(self.datahub_url)
    
    def act(self, event: dict) -> None:
        """OpenLineage 이벤트 처리"""
        
        if event.get('eventType') != 'COMPLETE':
            return  # 완료 이벤트만 처리
        
        # OpenLineage 이벤트 파싱
        job = event['job']
        inputs = event.get('inputs', [])
        outputs = event.get('outputs', [])
        
        # Output 데이터셋 URN 생성
        for output in outputs:
            output_urn = self._convert_to_datahub_urn(output)
            
            # Input 데이터셋 URN 생성
            upstream_urns = [
                self._convert_to_datahub_urn(inp) for inp in inputs
            ]
            
            # Lineage 생성
            lineage = UpstreamLineageClass(
                upstreams=[
                    UpstreamClass(
                        dataset=urn,
                        type=DatasetLineageTypeClass.TRANSFORMED,
                    )
                    for urn in upstream_urns
                ]
            )
            
            # DataHub에 전송
            self.emitter.emit_mcp(output_urn, 'upstreamLineage', lineage)
    
    def _convert_to_datahub_urn(self, openlineage_dataset: dict) -> str:
        """OpenLineage 데이터셋을 DataHub URN으로 변환"""
        
        namespace = openlineage_dataset['namespace']
        name = openlineage_dataset['name']
        
        # 플랫폼 추출 (예: mysql://host:port → mysql)
        platform = namespace.split('://')[0] if '://' in namespace else 'unknown'
        
        # DataHub URN 생성
        from datahub.emitter.mce_builder import make_dataset_urn
        
        return make_dataset_urn(
            platform=platform,
            name=name,
            env='PROD'
        )
    
    def close(self) -> None:
        pass
~~~


## 선택 기준

### OpenLineage를 선택해야 하는 경우

✓ 계보 추적이 주 목적
✓ 실시간 계보가 중요
✓ 여러 도구 간 계보 통합 필요
✓ 경량 솔루션 선호
✓ 표준 기반 접근 원함
✓ 기존 도구에 최소 영향


**사용 사례**
- Airflow, Spark, dbt 등 여러 도구의 계보 통합
- 실시간 데이터 품질 모니터링
- CI/CD 파이프라인에서 계보 검증
- 경량 계보 추적 시스템 구축

### DataHub를 선택해야 하는 경우

```
✓ 통합 메타데이터 플랫폼 필요
✓ 데이터 카탈로그 + 검색 필요
✓ 데이터 거버넌스 강화
✓ 스키마 관리 중요
✓ 팀 협업 기능 필요
✓ UI/UX 중요
```

**사용 사례**
- 전사 데이터 카탈로그 구축
- 데이터 디스커버리 및 검색
- 데이터 거버넌스 정책 관리
- 메타데이터 기반 협업
- 데이터 소유권 및 품질 관리

### 함께 사용하기 (Best of Both Worlds)

```
# 추천 아키텍처

OpenLineage (실시간 계보 수집)
         ↓
    Kafka Topic
         ↓
DataHub Actions (변환)
         ↓
  DataHub (저장 + UI)
```

**장점**

- OpenLineage의 실시간성 + 표준화
- DataHub의 풍부한 메타데이터 + UI
- 각 도구의 강점 활용

**구현**

* 통합 아키텍처 구현
	* 1. Airflow에서 OpenLineage 활성화
	* 2. OpenLineage → Kafka
	* 3. DataHub Actions로 Kafka 이벤트 소비
	* 4. DataHub에 메타데이터 저장
	* 5. DataHub UI에서 통합 뷰 제공


## 마이그레이션 전략

### OpenLineage → DataHub 마이그레이션

#### Phase 1: 병렬 운영 (1-2개월)
- OpenLineage 계속 사용
- DataHub 파일럿 (핵심 데이터셋)
- 데이터 품질 비교

#### Phase 2: 점진적 전환 (2-3개월)
- DataHub로 신규 파이프라인 등록
- 기존 파이프라인 단계적 마이그레이션
- OpenLineage는 계보 전용으로 유지

#### Phase 3: 통합 (1개월)
- OpenLineage → DataHub 통합
- OpenLineage를 DataHub의 계보 소스로 활용
- 전체 메타데이터는 DataHub에서 관리


## 적용 시 고려할 점

1. **POC 수행**: 두 도구 모두 작은 범위에서 테스트
2. **요구사항 명확화**: 조직의 실제 니즈를 먼저 파악
3. **리소스 고려**: 운영 가능한 인프라와 팀 역량 고려
4. **통합 고려**: 반드시 하나만 선택할 필요 없음
5. **커뮤니티 활용**: 각 프로젝트의 커뮤니티 지원 활용