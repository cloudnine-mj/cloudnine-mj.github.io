---
key: jekyll-text-theme
title: 'Airflow에서 REST API 호출하기'
excerpt: ' HttpOperator 😎'
tags: [Airflow]
---

# Airflow에서 REST API 호출하기

## 개념

HttpOperator는 REST API를 호출하고 응답을 처리하는 Operator입니다. 외부 시스템과의 통합, Webhook 전송, API 기반 데이터 수집 등에 사용됩니다.

## 설치

```bash
# HTTP Provider 설치
pip install apache-airflow-providers-http
```

## 원리

1. **HttpHook**을 사용하여 HTTP 요청을 전송
2. Connection에 **Base URL, 인증 정보**를 저장함.
3. **응답 검증 함수**로 성공/실패를 판단함.
4. **XCom**으로 응답 데이터를 다음 Task에 전달함.

### 코드

* 업무에서 활용했던 코드

### 1. 기본 사용법

```python
from airflow import DAG
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.providers.http.sensors.http import HttpSensor
from datetime import datetime
import json

dag = DAG(
    'http_api_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
)

# Connection 설정 (Airflow UI)
# Conn Id: external_api
# Conn Type: HTTP
# Host: https://api.example.com
# Extra: {"Authorization": "Bearer your-token"}

# API 호출
fetch_data = SimpleHttpOperator(
    task_id='fetch_data',
    http_conn_id='external_api',
    endpoint='/data/{{ ds }}',
    method='GET',
    headers={'Content-Type': 'application/json'},
    # 응답 검증
    response_check=lambda response: response.json()['status'] == 'success',
    # 응답을 XCom에 저장
    log_response=True,
    dag=dag,
)

# POST 요청
create_record = SimpleHttpOperator(
    task_id='create_record',
    http_conn_id='external_api',
    endpoint='/records',
    method='POST',
    data=json.dumps({
        'date': '{{ ds }}',
        'value': 100,
    }),
    headers={'Content-Type': 'application/json'},
    dag=dag,
)
```

### 2. 인증 방법들

```python
from airflow import DAG
from airflow.providers.http.operators.http import SimpleHttpOperator
from datetime import datetime
import json
import base64

dag = DAG('http_auth_examples', start_date=datetime(2024, 1, 1), catchup=False)

# 1. Bearer Token 인증 (Connection Extra에 설정)
# Extra: {"Authorization": "Bearer your-token"}

# 2. Basic Auth
basic_auth = SimpleHttpOperator(
    task_id='basic_auth',
    http_conn_id='api_basic_auth',  # Connection에 username/password 설정
    endpoint='/protected',
    method='GET',
    dag=dag,
)

# 3. API Key in Header
api_key_auth = SimpleHttpOperator(
    task_id='api_key_auth',
    http_conn_id='external_api',
    endpoint='/data',
    method='GET',
    headers={'X-API-Key': '{{ conn.external_api.extra_dejson.api_key }}'},
    dag=dag,
)

# 4. OAuth2 (토큰 갱신 포함)
def get_oauth_token(**context):
    """OAuth2 토큰 획득"""
    from airflow.providers.http.hooks.http import HttpHook
    
    hook = HttpHook(method='POST', http_conn_id='oauth_provider')
    
    response = hook.run(
        endpoint='/oauth/token',
        data={
            'grant_type': 'client_credentials',
            'client_id': 'your-client-id',
            'client_secret': 'your-client-secret',
        },
        headers={'Content-Type': 'application/x-www-form-urlencoded'},
    )
    
    token = response.json()['access_token']
    # XCom에 저장
    return token

from airflow.operators.python import PythonOperator

get_token = PythonOperator(
    task_id='get_oauth_token',
    python_callable=get_oauth_token,
    dag=dag,
)

call_api_with_oauth = SimpleHttpOperator(
    task_id='call_api_with_oauth',
    http_conn_id='external_api',
    endpoint='/protected-resource',
    method='GET',
    headers={
        'Authorization': 'Bearer {{ ti.xcom_pull(task_ids="get_oauth_token") }}'
    },
    dag=dag,
)

get_token >> call_api_with_oauth
```

### 3. 응답 처리 및 에러 핸들링


```python
from airflow import DAG
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.operators.python import PythonOperator
from datetime import datetime
import json

dag = DAG('http_response_handling', start_date=datetime(2024, 1, 1), catchup=False)

def advanced_response_check(response):
    """고급 응답 검증"""
    # HTTP 상태 코드 체크
    if response.status_code != 200:
        raise ValueError(f"API returned {response.status_code}")
    
    # JSON 파싱
    data = response.json()
    
    # 비즈니스 로직 검증
    if data.get('record_count', 0) == 0:
        raise ValueError("No records returned")
    
    if data.get('has_errors'):
        raise ValueError(f"API reported errors: {data.get('errors')}")
    
    # 성공
    return True

def process_api_response(**context):
    """API 응답 처리"""
    ti = context['ti']
    
    # XCom에서 응답 가져오기
    response_text = ti.xcom_pull(task_ids='fetch_data')
    data = json.loads(response_text)
    
    # 데이터 처리
    records = data['records']
    print(f"Processing {len(records)} records")
    
    processed_data = []
    for record in records:
        # 변환 로직
        processed_data.append({
            'id': record['id'],
            'value': record['value'] * 2,
            'processed_at': datetime.now().isoformat(),
        })
    
    return processed_data

fetch_data = SimpleHttpOperator(
    task_id='fetch_data',
    http_conn_id='external_api',
    endpoint='/data',
    method='GET',
    response_check=advanced_response_check,
    log_response=True,
    dag=dag,
)

process_data = PythonOperator(
    task_id='process_data',
    python_callable=process_api_response,
    dag=dag,
)

fetch_data >> process_data
```

### 4. 페이지네이션 처리


```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.http.hooks.http import HttpHook
from datetime import datetime

def fetch_all_pages(**context):
    """페이지네이션 API 전체 데이터 수집"""
    hook = HttpHook(method='GET', http_conn_id='external_api')
    
    all_records = []
    page = 1
    has_more = True
    
    while has_more:
        response = hook.run(
            endpoint=f'/data?page={page}&page_size=100',
        )
        
        data = response.json()
        records = data['records']
        all_records.extend(records)
        
        print(f"Fetched page {page}: {len(records)} records")
        
        # 다음 페이지 확인
        has_more = data.get('has_next_page', False)
        page += 1
        
        # 안전 장치 (무한 루프 방지)
        if page > 1000:
            raise ValueError("Too many pages, possible infinite loop")
    
    print(f"Total records fetched: {len(all_records)}")
    return all_records

dag = DAG('http_pagination', start_date=datetime(2024, 1, 1), catchup=False)

fetch_all = PythonOperator(
    task_id='fetch_all_pages',
    python_callable=fetch_all_pages,
    dag=dag,
)
```

### 5. Retry 및 Timeout 설정


```python
from airflow import DAG
from airflow.providers.http.operators.http import SimpleHttpOperator
from datetime import datetime, timedelta

dag = DAG(
    'http_retry_example',
    start_date=datetime(2024, 1, 1),
    catchup=False,
)

reliable_api_call = SimpleHttpOperator(
    task_id='reliable_api_call',
    http_conn_id='external_api',
    endpoint='/data',
    method='GET',
    # Retry 설정
    retries=5,
    retry_delay=timedelta(minutes=2),
    retry_exponential_backoff=True,  # 지수 백오프
    max_retry_delay=timedelta(minutes=10),
    # Timeout 설정
    extra_options={
        'timeout': 30,  # 30초 타임아웃
        'verify': True,  # SSL 검증
    },
    # 일시적 에러만 재시도
    response_check=lambda response: response.status_code in [200, 201],
    dag=dag,
)
```

### 6. Secrets 관리


```python
from airflow import DAG
from airflow.providers.http.operators.http import SimpleHttpOperator
from airflow.hooks.base import BaseHook
from datetime import datetime

def get_api_credentials():
    """Airflow Secrets Backend에서 자격증명 가져오기"""
    connection = BaseHook.get_connection('external_api')
    extra = connection.extra_dejson
    
    return {
        'api_key': extra.get('api_key'),
        'client_id': extra.get('client_id'),
        'client_secret': extra.get('client_secret'),
    }

dag = DAG('http_secrets', start_date=datetime(2024, 1, 1), catchup=False)

# Jinja 템플릿으로 Secret 사용
secure_api_call = SimpleHttpOperator(
    task_id='secure_api_call',
    http_conn_id='external_api',
    endpoint='/protected',
    method='GET',
    headers={
        'X-API-Key': '{{ conn.external_api.extra_dejson.api_key }}',
        'X-Client-ID': '{{ conn.external_api.extra_dejson.client_id }}',
    },
    dag=dag,
)
```

## 적용할 때 고려했던 점

1. **Timeout 설정**: API 응답이 느린 경우를 대비해 적절한 timeout을 설정
2. **Retry 전략**: 네트워크 오류 등 일시적 장애에 대비해 재시도를 설정하되, 멱등성을 보장해야 함.
3. **Rate Limiting**: API 호출 제한이 있다면 Task 실행 간격을 조정하거나 큐잉 메커니즘을 구현해야 함.
4. **Secrets 관리**: API Key 등 민감한 정보는 Airflow Connection이나 Secrets Backend에 저장해야 함.
5. **로깅**: API 요청/응답을 로깅하여 디버깅을 쉽게 하되, 민감한 정보는 마스킹해야 함.
6. **응답 검증**: 단순히 HTTP 200만 확인하지 말고, 비즈니스 로직 수준의 검증을 수행해야 함.
