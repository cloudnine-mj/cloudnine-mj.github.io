---
key: jekyll-text-theme
title: 'Airflow + DataHub'
excerpt: ' Airflow + DataHub로 메타데이터 기반 파이프라인 관리 자동화 😎'
tags: [Airflow, Datahub]
---

# Airflow + DataHub

## 개념

* DataHub는 LinkedIn에서 개발한 오픈소스 메타데이터 플랫폼
* Airflow와 연동하여 데이터 파이프라인의 Lineage(계보), 스키마, 소유자 정보 등을 자동으로 수집하고 관리할 수 있음.

## Datahub 설치

```bash
# DataHub CLI 설치
pip install acryl-datahub

# DataHub 서버 설치 (Docker Compose)
python3 -m datahub docker quickstart

# Airflow에 DataHub 플러그인 설치
pip install acryl-datahub[airflow]

# Airflow 설정에 DataHub 연결 추가
# airflow.cfg 또는 환경변수
export DATAHUB_GMS_URL=http://localhost:8080
export DATAHUB_GMS_TOKEN=your-access-token
```

## 원리

1. Airflow DAG에 **DataHub Lineage Backend**를 설정함.
2. Task 실행 시 **Inlet(입력)과 Outlet(출력)** 정보를 메타데이터로 기록함.
3. DataHub가 이 정보를 수집하여 **데이터 계보 그래프**를 구성함.
4. **스키마 정보, 소유자, 태그** 등 추가 메타데이터를 자동으로 수집함.
5. DataHub UI에서 **검색, 탐색, 추적**이 가능함.

## 코드

* 실무에서 활용했던 코드

### 1. Airflow에 DataHub Lineage Backend 설정


```python
# airflow.cfg 수정
[lineage]
backend = datahub_provider.lineage.datahub.DatahubLineageBackend

[datahub]
datahub_conn_id = datahub_rest_default
```

### 2. Connection 설정

```python
# Airflow UI에서 Connection 추가
# Conn Id: datahub_rest_default
# Conn Type: HTTP
# Host: http://localhost:8080
# Extra: {"token": "your-access-token"}
```

### 3. Lineage 정보가 포함된 DAG


```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.lineage.entities import File, Table
from datetime import datetime

dag = DAG(
    'datahub_lineage_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
)

def extract_from_source(**context):
    """소스에서 데이터 추출"""
    # 실제 추출 로직
    print("Extracting data from source")

# Inlet과 Outlet 정의
extract_task = PythonOperator(
    task_id='extract_data',
    python_callable=extract_from_source,
    # 입력: PostgreSQL 테이블
    inlets=[
        Table(
            database='production',
            cluster='postgres_prod',
            name='users',
        )
    ],
    # 출력: S3 파일
    outlets=[
        File(
            url='s3://my-bucket/raw/users/{{ ds }}.parquet'
        )
    ],
    dag=dag,
)

def transform_data(**context):
    """데이터 변환"""
    print("Transforming data")

transform_task = PythonOperator(
    task_id='transform_data',
    python_callable=transform_data,
    inlets=[
        File(url='s3://my-bucket/raw/users/{{ ds }}.parquet')
    ],
    outlets=[
        File(url='s3://my-bucket/processed/users/{{ ds }}.parquet')
    ],
    dag=dag,
)

def load_to_warehouse(**context):
    """데이터 웨어하우스 적재"""
    print("Loading to warehouse")

load_task = PythonOperator(
    task_id='load_data',
    python_callable=load_to_warehouse,
    inlets=[
        File(url='s3://my-bucket/processed/users/{{ ds }}.parquet')
    ],
    outlets=[
        Table(
            database='analytics',
            cluster='snowflake_prod',
            name='dim_users',
        )
    ],
    dag=dag,
)

extract_task >> transform_task >> load_task
```

### 4. DataHub API로 메타데이터 직접 등록


```python
from datahub.emitter.mce_builder import make_dataset_urn
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.schema_classes import (
    DatasetPropertiesClass,
    OwnerClass,
    OwnershipClass,
    OwnershipTypeClass,
)

def register_dataset_metadata(**context):
    """DataHub에 데이터셋 메타데이터 등록"""
    
    emitter = DatahubRestEmitter(
        gms_server='http://localhost:8080',
        token='your-access-token'
    )
    
    # 데이터셋 URN 생성
    dataset_urn = make_dataset_urn(
        platform='snowflake',
        name='analytics.dim_users',
        env='PROD'
    )
    
    # Properties 설정
    properties = DatasetPropertiesClass(
        description='사용자 차원 테이블',
        customProperties={
            'data_format': 'parquet',
            'update_frequency': 'daily',
            'retention_days': '365',
        }
    )
    
    # 소유자 설정
    ownership = OwnershipClass(
        owners=[
            OwnerClass(
                owner='urn:li:corpuser:data-team',
                type=OwnershipTypeClass.DATAOWNER,
            )
        ]
    )
    
    # 메타데이터 전송
    emitter.emit_mcp(
        dataset_urn,
        aspect_name='datasetProperties',
        aspect_value=properties,
    )
    
    emitter.emit_mcp(
        dataset_urn,
        aspect_name='ownership',
        aspect_value=ownership,
    )
    
    print(f"Metadata registered for {dataset_urn}")

metadata_task = PythonOperator(
    task_id='register_metadata',
    python_callable=register_dataset_metadata,
    dag=dag,
)
```

### 5. 스키마 자동 수집

```python
from datahub.emitter.mce_builder import make_dataset_urn, make_schema_field_urn
from datahub.metadata.schema_classes import (
    SchemaMetadataClass,
    SchemaFieldClass,
    SchemaFieldDataTypeClass,
    StringTypeClass,
    NumberTypeClass,
)

def update_schema_metadata(**context):
    """데이터베이스 스키마 정보를 DataHub에 업데이트"""
    from airflow.hooks.postgres_hook import PostgresHook
    
    hook = PostgresHook(postgres_conn_id='postgres_prod')
    
    # 스키마 정보 조회
    schema_info = hook.get_records("""
        SELECT column_name, data_type 
        FROM information_schema.columns 
        WHERE table_name = 'users'
    """)
    
    # DataHub 스키마 형식으로 변환
    fields = []
    for col_name, data_type in schema_info:
        field = SchemaFieldClass(
            fieldPath=col_name,
            type=SchemaFieldDataTypeClass(
                type=StringTypeClass() if 'char' in data_type else NumberTypeClass()
            ),
            nativeDataType=data_type,
        )
        fields.append(field)
    
    schema = SchemaMetadataClass(
        schemaName='users',
        platform='urn:li:dataPlatform:postgres',
        version=0,
        fields=fields,
        hash='',
        platformSchema=None,
    )
    
    # DataHub에 전송
    emitter = DatahubRestEmitter(
        gms_server='http://localhost:8080',
        token='your-access-token'
    )
    
    dataset_urn = make_dataset_urn(
        platform='postgres',
        name='production.users',
        env='PROD'
    )
    
    emitter.emit_mcp(
        dataset_urn,
        aspect_name='schemaMetadata',
        aspect_value=schema,
    )

schema_task = PythonOperator(
    task_id='update_schema',
    python_callable=update_schema_metadata,
    dag=dag,
)
```

### 6. DataHub에서 메타데이터 조회


```python
from datahub.ingestion.graph.client import DatahubClientConfig, DataHubGraph

def query_datahub_metadata(**context):
    """DataHub에서 메타데이터 조회"""
    
    graph = DataHubGraph(
        config=DatahubClientConfig(
            server='http://localhost:8080',
            token='your-access-token'
        )
    )
    
    # 데이터셋 정보 조회
    dataset_urn = 'urn:li:dataset:(urn:li:dataPlatform:snowflake,analytics.dim_users,PROD)'
    
    # Properties 조회
    properties = graph.get_aspect(
        entity_urn=dataset_urn,
        aspect_type=DatasetPropertiesClass,
    )
    print(f"Description: {properties.description}")
    
    # Lineage 조회 (업스트림)
    lineage = graph.get_lineage(
        entity_urn=dataset_urn,
        direction='UPSTREAM',
        max_hops=3,
    )
    
    print("Upstream datasets:")
    for edge in lineage.edges:
        print(f"  - {edge.sourceUrn}")
    
    # 검색
    search_results = graph.search(
        entity_types=['dataset'],
        query='users',
        start=0,
        count=10,
    )
    
    for result in search_results:
        print(f"Found: {result.entity_urn}")

query_task = PythonOperator(
    task_id='query_metadata',
    python_callable=query_datahub_metadata,
    dag=dag,
)
```

## 적용할 때 고려했던 점

1. **일관된 네이밍**: 데이터셋 이름을 일관되게 사용하여 Lineage 추적을 정확하게 해야 함. (이슈 발생...)
2. **자동화**: 스키마 변경 시 자동으로 DataHub에 반영되도록 파이프라인을 구성함.
3. **태그 활용**: PII, 민감정보 등의 태그를 추가하여 데이터 거버넌스를 강화해야 함.
4. **문서화**: 각 데이터셋의 비즈니스 의미를 description에 명확히 기록해야 함.
5. **알람 연동**: 중요 데이터셋의 스키마 변경, 데이터 품질 이슈를 DataHub에서 추적하고 알람을 받도록 하는 것이 좋음.
