---
key: jekyll-text-theme
title: 'Metadata Ingestion 설정'
excerpt: ' Datahub 시작, 적용해보기😎'
tags: [Datahub]
---



# Metadata Ingestion 설정

## 개념

* DataHub는 다양한 데이터 소스에서 메타데이터를 자동으로 수집하는 Ingestion Framework를 제공함.
* 각 소스별로 최적화된 설정과 트러블슈팅 방법을 익히면 효과적인 메타데이터 관리가 가능함.

## Ingestion Framework 구조

```
┌─────────────────────────────────────────────────────┐
│              Recipe 파일 (YAML/JSON)                 │
│          Source → Transformers → Sink               │
└─────────────────────────────────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
    ┌────────┐   ┌──────────┐  ┌────────┐
    │ Source │   │Transform │  │  Sink  │
    │ Config │   │  Config  │  │ Config │
    └────────┘   └──────────┘  └────────┘
```

## 기본 Recipe 구조

```yaml
# basic_recipe.yml
source:
  type: <source_type>
  config:
    # 소스별 설정

# 선택적 변환
transformers:
  - type: <transformer_type>
    config:
      # 변환 설정

sink:
  type: datahub-rest  # 또는 datahub-kafka
  config:
    server: 'http://localhost:8080'
    token: '${DATAHUB_TOKEN}'  # 선택적
```

### MySQL/PostgreSQL Ingestion

```yaml
# mysql_recipe.yml
source:
  type: mysql
  config:
    # 연결 정보
    host_port: "mysql.company.com:3306"
    database: "production"
    
    # 인증
    username: "${MYSQL_USER}"
    password: "${MYSQL_PASSWORD}"
    
    # 또는 SQLAlchemy URI 사용
    # sqlalchemy_uri: "mysql+pymysql://user:pass@host:3306/db"
    
    # 수집 대상 필터링
    database_pattern:
      allow:
        - "production"
        - "analytics"
      deny:
        - "test.*"
        - "dev.*"
    
    table_pattern:
      allow:
        - "customers"
        - "orders"
        - "products"
      deny:
        - ".*_temp"
        - ".*_backup"
    
    schema_pattern:
      allow:
        - "public"
    
    # 뷰 포함 여부
    include_views: true
    include_tables: true
    
    # 프로파일링 (통계 수집)
    profiling:
      enabled: true
      # 전체 테이블 프로파일링 (느림)
      profile_table_level_only: false
      
      # 샘플링
      profile_sample_size: 10000
      
      # 수집할 통계
      include_field_null_count: true
      include_field_distinct_count: true
      include_field_min_value: true
      include_field_max_value: true
      include_field_mean_value: true
      include_field_median_value: true
      include_field_stddev_value: true
      
      # 프로파일링 대상 필터
      profile_table_size_limit: 10  # 10GB 이하만
      profile_table_row_limit: 10000000  # 1000만 행 이하만
    
    # 사용 통계 수집
    stateful_ingestion:
      enabled: true
      
    # 소프트 삭제된 테이블 처리
    remove_stale_metadata: true
    
    # 커스텀 프로퍼티
    domain:
      "production.*": "urn:li:domain:Production"
      "analytics.*": "urn:li:domain:Analytics"

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'

# PostgreSQL은 거의 동일
# source:
#   type: postgres
#   config:
#     host_port: "postgres.company.com:5432"
#     ...
```

## Snowflake Ingestion

```yaml
# snowflake_recipe.yml
source:
  type: snowflake
  config:
    # 계정 정보
    account_id: "xy12345.us-east-1"
    warehouse: "COMPUTE_WH"
    
    # 인증
    username: "${SNOWFLAKE_USER}"
    password: "${SNOWFLAKE_PASSWORD}"
    # 또는 키 페어 인증
    # private_key_path: "/path/to/private_key.p8"
    # private_key_password: "${PRIVATE_KEY_PASSWORD}"
    
    # 역할
    role: "ACCOUNTADMIN"
    
    # 수집 대상
    database_pattern:
      allow:
        - "ANALYTICS_DB"
        - "WAREHOUSE_DB"
    
    schema_pattern:
      allow:
        - "PUBLIC"
        - "REPORTING"
      deny:
        - ".*_BACKUP"
    
    table_pattern:
      allow:
        - "FACT_.*"
        - "DIM_.*"
      deny:
        - ".*_TEMP"
    
    # Snowflake 특화 옵션
    include_tables: true
    include_views: true
    include_external_tables: true
    include_materialized_views: true
    
    # 사용 통계 (쿼리 로그)
    include_usage_stats: true
    start_time: "-7 days"  # 최근 7일
    end_time: "now"
    
    # 테이블 Lineage (쿼리 파싱)
    include_table_lineage: true
    include_column_lineage: true  # 컬럼 레벨 계보
    
    # 쿼리 태그 기반 필터링
    email_domain: "company.com"  # 이 도메인 사용자만
    
    # 프로파일링
    profiling:
      enabled: true
      profile_table_level_only: true  # 테이블 레벨만 (빠름)
      max_workers: 5  # 병렬 처리
    
    # 태그 및 주석 수집
    extract_tags: true
    
    # 성능 최적화
    check_role_grants: false

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## BigQuery Ingestion


```yaml
# bigquery_recipe.yml
source:
  type: bigquery
  config:
    # GCP 프로젝트
    project_id: "my-gcp-project"
    
    # 인증
    # 방법 1: 서비스 계정 JSON 파일
    credential:
      project_id: "my-gcp-project"
      private_key_id: "..."
      private_key: "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
      client_email: "datahub@my-project.iam.gserviceaccount.com"
      client_id: "..."
    
    # 방법 2: 파일 경로
    # credential_path: "/path/to/service-account.json"
    
    # 수집 대상
    project_ids:
      - "my-gcp-project"
      - "another-project"
    
    dataset_pattern:
      allow:
        - "analytics.*"
        - "warehouse.*"
      deny:
        - ".*_backup"
        - "scratch_.*"
    
    table_pattern:
      allow:
        - "fact_.*"
        - "dim_.*"
    
    # BigQuery 특화 옵션
    include_tables: true
    include_views: true
    include_external_tables: true
    
    # 사용 통계 (쿼리 로그)
    include_usage_statistics: true
    usage_lookback_days: 7
    
    # Lineage (쿼리 파싱)
    include_table_lineage: true
    use_exported_bigquery_audit_metadata: true  # Audit 로그 사용
    
    # 프로파일링
    profiling:
      enabled: true
      profile_table_level_only: false
      
      # 비용 절감을 위한 샘플링
      use_sampling: true
      sample_size: 10000
      
      # Dry run으로 비용 확인
      max_workers: 5
    
    # 파티션 테이블 처리
    partition_support_enabled: true
    
    # 레이블 및 태그 수집
    extract_policy_tags: true
    extract_labels: true

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## Kafka Ingestion


```yaml
# kafka_recipe.yml
source:
  type: kafka
  config:
    # Kafka 연결
    connection:
      bootstrap: "kafka-broker-1:9092,kafka-broker-2:9092"
      schema_registry_url: "http://schema-registry:8081"
      
      # 인증
      consumer_config:
        security.protocol: "SASL_SSL"
        sasl.mechanism: "PLAIN"
        sasl.username: "${KAFKA_USER}"
        sasl.password: "${KAFKA_PASSWORD}"
    
    # 수집 대상 토픽
    topic_patterns:
      allow:
        - "events.*"
        - "logs.*"
      deny:
        - ".*test.*"
        - ".*temp.*"
    
    # 스키마 수집 (Schema Registry)
    platform_instance: "production-kafka"
    
    # 도메인 매핑
    domain:
      "events.*": "urn:li:domain:Events"
      "logs.*": "urn:li:domain:Logging"

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## S3 / HDFS Ingestion

```yaml
# s3_recipe.yml
source:
  type: s3
  config:
    # AWS 인증
    aws_config:
      aws_access_key_id: "${AWS_ACCESS_KEY}"
      aws_secret_access_key: "${AWS_SECRET_KEY}"
      aws_region: "us-east-1"
    
    # S3 경로
    path_specs:
      - include: "s3://my-bucket/data-lake/raw/**/*.parquet"
        exclude:
          - "s3://my-bucket/data-lake/raw/**/_*"  # 숨김 파일
        
        # 테이블 추출 패턴
        table_name: "raw_{table}"  # 파일명에서 테이블명 추출
        
        # 파티션 처리
        partition_pattern: "s3://my-bucket/data-lake/raw/{table}/year={year}/month={month}/day={day}/*.parquet"
        
        # 스키마 추출 (Parquet/Avro)
        enable_schema_inference: true
        
        # 파일 통계
        profile_patterns:
          allow:
            - ".*\\.parquet"
    
    # 프로파일링
    profiling:
      enabled: true
      max_file_size: "1GB"

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## Airflow Ingestion

```yaml
# airflow_recipe.yml
source:
  type: airflow
  config:
    # Airflow 메타데이터 DB 연결
    conn_id: "airflow_db"
    
    # 또는 직접 연결
    host_port: "postgres:5432"
    database: "airflow"
    username: "${AIRFLOW_DB_USER}"
    password: "${AIRFLOW_DB_PASSWORD}"
    
    # Airflow UI URL
    webserver_url: "http://airflow.company.com"
    
    # 수집 옵션
    capture_ownership_info: true
    capture_tags_info: true
    capture_executions: true  # 실행 이력
    
    # DAG 필터
    dag_pattern:
      allow:
        - "prod_.*"
        - "etl_.*"
      deny:
        - "test_.*"
        - "dev_.*"
    
    # Lineage 수집
    capture_lineage: true
    
    # 스케줄 정보
    extract_dag_schedule: true

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## Tableau Ingestion

```yaml
# tableau_recipe.yml
source:
  type: tableau
  config:
    # Tableau Server 연결
    connect_uri: "https://tableau.company.com"
    site: "production"  # 사이트명 (없으면 "")
    
    # 인증
    username: "${TABLEAU_USER}"
    password: "${TABLEAU_PASSWORD}"
    # 또는 Personal Access Token
    # token_name: "${TABLEAU_TOKEN_NAME}"
    # token_value: "${TABLEAU_TOKEN_VALUE}"
    
    # 수집 대상
    projects:
      - "Sales Analytics"
      - "Marketing Dashboards"
    
    # 워크북/대시보드 필터
    workbook_pattern:
      allow:
        - "Production.*"
      deny:
        - ".*Draft.*"
    
    # Lineage 수집 (대시보드 → 데이터소스)
    extract_lineage: true
    
    # 사용 통계
    extract_usage_stats: true
    stateful_ingestion:
      enabled: true
    
    # 커스텀 메타데이터
    extract_tags: true
    extract_owners: true

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## Transformers (변환)

```yaml
# recipe_with_transformers.yml
source:
  type: mysql
  config:
    host_port: "localhost:3306"
    database: "production"
    username: "root"
    password: "password"

transformers:
  # 1. 패턴 기반 태그 추가
  - type: pattern_add_dataset_tags
    config:
      tag_pattern:
        rules:
          ".*_pii$": ["PII", "Sensitive"]
          ".*_customer.*": ["Customer_Data"]
          "fact_.*": ["Fact_Table"]
          "dim_.*": ["Dimension_Table"]
  
  # 2. 도메인 할당
  - type: pattern_add_dataset_domain
    config:
      domain_pattern:
        rules:
          "sales.*": "urn:li:domain:Sales"
          "marketing.*": "urn:li:domain:Marketing"
          "finance.*": "urn:li:domain:Finance"
  
  # 3. 소유자 추가
  - type: pattern_add_dataset_ownership
    config:
      owner_pattern:
        rules:
          "sales.*":
            - "urn:li:corpuser:sales-team"
          "marketing.*":
            - "urn:li:corpuser:marketing-team"
  
  # 4. 용어 사전 매핑
  - type: pattern_add_dataset_terms
    config:
      term_pattern:
        rules:
          ".*customer.*":
            - "urn:li:glossaryTerm:Customer"
          ".*revenue.*":
            - "urn:li:glossaryTerm:Revenue"
  
  # 5. 설명 추가
  - type: simple_add_dataset_properties
    config:
      properties:
        description: "Auto-generated from MySQL production database"
  
  # 6. 태그 제거
  - type: simple_remove_dataset_tags
    config:
      tag_pattern: ".*_temp"

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## 배치 Ingestion 실행

```bash
# 1. 단일 실행
datahub ingest -c mysql_recipe.yml

# 2. Dry run (실제 전송 안함)
datahub ingest -c mysql_recipe.yml --dry-run

# 3. 디버그 모드
datahub ingest -c mysql_recipe.yml --debug

# 4. 결과 파일 저장
datahub ingest -c mysql_recipe.yml --report-to report.json

# 5. 환경변수 사용
export MYSQL_USER=root
export MYSQL_PASSWORD=password
datahub ingest -c mysql_recipe.yml
```

## 스케줄링

### Airflow DAG

```python
# ingestion_dag.py
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'email_on_failure': True,
    'email': ['data-team@company.com'],
}

dag = DAG(
    'datahub_metadata_ingestion',
    default_args=default_args,
    schedule_interval='0 2 * * *',  # 매일 02:00
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['datahub', 'metadata'],
)

# MySQL Ingestion
mysql_ingest = BashOperator(
    task_id='ingest_mysql',
    bash_command='datahub ingest -c /configs/mysql_recipe.yml',
    dag=dag,
)

# Snowflake Ingestion
snowflake_ingest = BashOperator(
    task_id='ingest_snowflake',
    bash_command='datahub ingest -c /configs/snowflake_recipe.yml',
    dag=dag,
)

# Airflow Ingestion (자기 자신)
airflow_ingest = BashOperator(
    task_id='ingest_airflow',
    bash_command='datahub ingest -c /configs/airflow_recipe.yml',
    dag=dag,
)

# 병렬 실행
[mysql_ingest, snowflake_ingest] >> airflow_ingest
```

### Kubernetes CronJob


```yaml
# datahub-ingestion-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: datahub-mysql-ingestion
  namespace: datahub
spec:
  schedule: "0 2 * * *"  # 매일 02:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: ingestion
            image: acryldata/datahub-ingestion:latest
            command:
            - /bin/sh
            - -c
            - |
              datahub ingest -c /configs/mysql_recipe.yml
            volumeMounts:
            - name: config
              mountPath: /configs
            env:
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-credentials
                  key: username
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-credentials
                  key: password
          volumes:
          - name: config
            configMap:
              name: datahub-recipes
          restartPolicy: OnFailure
```

## 성능 최적화

### 1. 프로파일링 최적화

```yaml
# 큰 테이블 프로파일링 최적화
profiling:
  enabled: true
  
  # 테이블 레벨만 (빠름)
  profile_table_level_only: true
  
  # 샘플링
  use_sampling: true
  sample_size: 10000
  
  # 크기/행 수 제한
  profile_table_size_limit: 10  # GB
  profile_table_row_limit: 10000000
  
  # 병렬 처리
  max_workers: 5
  
  # 특정 컬럼 제외
  exclude_field_patterns:
    - ".*_id$"
    - ".*_timestamp$"
```

### 2. 병렬 처리

```python
# parallel_ingestion.py
from concurrent.futures import ThreadPoolExecutor
import subprocess

recipes = [
    'mysql_recipe.yml',
    'snowflake_recipe.yml',
    'bigquery_recipe.yml',
]

def run_ingestion(recipe):
    """Ingestion 실행"""
    cmd = f'datahub ingest -c {recipe}'
    result = subprocess.run(cmd, shell=True, capture_output=True)
    
    if result.returncode != 0:
        print(f"Failed: {recipe}")
        print(result.stderr.decode())
    else:
        print(f"Success: {recipe}")

# 병렬 실행
with ThreadPoolExecutor(max_workers=3) as executor:
    executor.map(run_ingestion, recipes)
```

### 3. 증분 수집 (Stateful Ingestion)

```yaml
# incremental_recipe.yml
source:
  type: mysql
  config:
    host_port: "localhost:3306"
    database: "production"
    username: "root"
    password: "password"
    
    # 상태 저장 활성화
    stateful_ingestion:
      enabled: true
      
      # 상태 저장 위치
      state_provider:
        type: datahub
        config:
          datahub_api:
            server: "http://localhost:8080"
      
      # 제거된 엔티티 처리
      remove_stale_metadata: true

sink:
  type: datahub-rest
  config:
    server: 'http://localhost:8080'
```

## 트러블슈팅

### 1. 연결 오류

```bash
# 테스트 스크립트
# test_connection.py
from sqlalchemy import create_engine

# MySQL 테스트
engine = create_engine('mysql+pymysql://user:pass@host:3306/db')
with engine.connect() as conn:
    result = conn.execute("SELECT 1")
    print(f"MySQL OK: {result.fetchone()}")

# Snowflake 테스트
from snowflake.connector import connect

conn = connect(
    account='xy12345.us-east-1',
    user='username',
    password='password',
    warehouse='COMPUTE_WH'
)
cursor = conn.cursor()
cursor.execute("SELECT CURRENT_VERSION()")
print(f"Snowflake OK: {cursor.fetchone()}")
```

### 2. 메모리 부족

```yaml
# 메모리 절약 설정
source:
  type: mysql
  config:
    # 배치 크기 줄이기
    profiling:
      enabled: true
      max_workers: 2  # 병렬도 낮추기
      profile_sample_size: 1000  # 샘플 크기 줄이기
    
    # 테이블 수 제한
    table_pattern:
      allow:
        - "important_table_1"
        - "important_table_2"
```

### 3. 권한 오류


```sql
-- MySQL 최소 권한
GRANT SELECT ON database.* TO 'datahub_user'@'%';
GRANT SELECT ON information_schema.* TO 'datahub_user'@'%';

-- Snowflake 최소 권한
GRANT USAGE ON WAREHOUSE COMPUTE_WH TO ROLE datahub_role;
GRANT USAGE ON DATABASE analytics_db TO ROLE datahub_role;
GRANT USAGE ON SCHEMA analytics_db.public TO ROLE datahub_role;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics_db.public TO ROLE datahub_role;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics_db.public TO ROLE datahub_role;

-- 사용 통계 수집 권한
GRANT IMPORTED PRIVILEGES ON DATABASE snowflake TO ROLE datahub_role;
```

## 적용할 때 고려해야 할 점

1. **점진적 수집**: 모든 테이블을 한 번에 수집하지 말고 중요한 것부터
2. **프로파일링 선택**: 필요한 테이블만 프로파일링하여 비용 절감
3. **스케줄 분산**: 모든 소스를 동시에 수집하지 말고 시간 분산
4. **모니터링**: Ingestion 성공/실패를 모니터링하고 알림 설정
5. **문서화**: 각 Recipe의 목적과 스케줄을 문서화
6. **테스트**: 프로덕션 적용 전 dry-run으로 테스트