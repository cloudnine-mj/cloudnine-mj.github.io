---
key: jekyll-text-theme
title: 'Airflow DAG'
excerpt: 'DAG 알아보기😎'
tags: [Airflow]
---


# DAG (Directed Acyclic Graph)란?

* DAG는 방향성 비순환 그래프를 의미하며, Airflow에서는 워크플로우를 정의하는 핵심 개념임. 각 노드는 Task를 나타내고, 엣지는 Task 간의 의존성을 나타냄.

## DAG의 주요 구성요소

| 구성요소 | 설명                                |
| -------- | ----------------------------------- |
| DAG      | 전체 워크플로우를 정의하는 컨테이너 |
| Task     | 실제 작업을 수행하는 단위           |
| Operator | Task를 생성하는 템플릿              |
| Sensor   | 특정 조건이 충족될 때까지 대기      |
| Hook     | 외부 시스템과의 연결 관리           |

## 1. 기본 DAG 구조

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

# DAG 기본 설정
default_args = {
    'owner': 'data_team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email': ['data@example.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

# DAG 정의
with DAG(
    'data_pipeline_dag',
    default_args=default_args,
    description='데이터 파이프라인 DAG',
    schedule_interval='0 2 * * *',  # 매일 오전 2시
    catchup=False,
    tags=['data', 'pipeline'],
) as dag:
    
    def extract_data():
        print("데이터 추출 중...")
        return "extracted_data"
    
    def transform_data(**context):
        ti = context['ti']
        data = ti.xcom_pull(task_ids='extract')
        print(f"데이터 변환 중: {data}")
        return "transformed_data"
    
    def load_data(**context):
        ti = context['ti']
        data = ti.xcom_pull(task_ids='transform')
        print(f"데이터 적재 중: {data}")
    
    # Task 정의
    extract_task = PythonOperator(
        task_id='extract',
        python_callable=extract_data,
    )
    
    transform_task = PythonOperator(
        task_id='transform',
        python_callable=transform_data,
        provide_context=True,
    )
    
    load_task = PythonOperator(
        task_id='load',
        python_callable=load_data,
        provide_context=True,
    )
    
    # Task 의존성 설정
    extract_task >> transform_task >> load_task
```

## 2. 다양한 Operator 활용

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from airflow.operators.email import EmailOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime

with DAG(
    'operator_examples',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
) as dag:
    
    # Bash Operator
    bash_task = BashOperator(
        task_id='run_bash_script',
        bash_command='echo "Hello from Bash" && date',
    )
    
    # Python Operator
    def python_function():
        print("Hello from Python")
    
    python_task = PythonOperator(
        task_id='run_python_function',
        python_callable=python_function,
    )
    
    # PostgreSQL Operator
    postgres_task = PostgresOperator(
        task_id='run_postgres_query',
        postgres_conn_id='postgres_default',
        sql="""
            SELECT COUNT(*) FROM users;
        """,
    )
    
    # Email Operator
    email_task = EmailOperator(
        task_id='send_email',
        to='admin@example.com',
        subject='Airflow DAG 완료',
        html_content='<h3>DAG 실행이 완료되었습니다.</h3>',
    )
    
    bash_task >> python_task >> postgres_task >> email_task
```

## 3. Task 의존성 패턴

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def dummy_task():
    print("Task executed")

with DAG(
    'dependency_patterns',
    start_date=datetime(2024, 1, 1),
    schedule_interval=None,
    catchup=False,
) as dag:
    
    task1 = PythonOperator(task_id='task1', python_callable=dummy_task)
    task2 = PythonOperator(task_id='task2', python_callable=dummy_task)
    task3 = PythonOperator(task_id='task3', python_callable=dummy_task)
    task4 = PythonOperator(task_id='task4', python_callable=dummy_task)
    task5 = PythonOperator(task_id='task5', python_callable=dummy_task)
    
    # 순차 실행
    task1 >> task2 >> task3
    
    # 병렬 실행 후 합류
    task1 >> [task2, task3] >> task4
    
    # 복잡한 의존성
    task1 >> task2
    task1 >> task3
    [task2, task3] >> task4 >> task5
    
    # 또는 set_upstream, set_downstream 사용
    task1.set_downstream([task2, task3])
    task4.set_upstream([task2, task3])
```

## 4. Dynamic Task 생성

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def process_data(item):
    print(f"Processing {item}")

with DAG(
    'dynamic_tasks',
    start_date=datetime(2024, 1, 1),
    schedule_interval=None,
    catchup=False,
) as dag:
    
    items = ['item1', 'item2', 'item3', 'item4', 'item5']
    
    # 동적으로 Task 생성
    tasks = []
    for item in items:
        task = PythonOperator(
            task_id=f'process_{item}',
            python_callable=process_data,
            op_args=[item],
        )
        tasks.append(task)
    
    # 모든 Task를 병렬로 실행
    # tasks는 리스트이므로 자동으로 병렬 실행됨
```

## 5. TaskGroup을 활용한 Task 그룹화

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.utils.task_group import TaskGroup
from datetime import datetime

def dummy_task():
    print("Task executed")

with DAG(
    'taskgroup_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval=None,
    catchup=False,
) as dag:
    
    start = PythonOperator(
        task_id='start',
        python_callable=dummy_task,
    )
    
    # 데이터 처리 그룹
    with TaskGroup('data_processing') as processing_group:
        extract = PythonOperator(
            task_id='extract',
            python_callable=dummy_task,
        )
        transform = PythonOperator(
            task_id='transform',
            python_callable=dummy_task,
        )
        load = PythonOperator(
            task_id='load',
            python_callable=dummy_task,
        )
        
        extract >> transform >> load
    
    # 데이터 검증 그룹
    with TaskGroup('data_validation') as validation_group:
        validate_schema = PythonOperator(
            task_id='validate_schema',
            python_callable=dummy_task,
        )
        validate_quality = PythonOperator(
            task_id='validate_quality',
            python_callable=dummy_task,
        )
        
        [validate_schema, validate_quality]
    
    end = PythonOperator(
        task_id='end',
        python_callable=dummy_task,
    )
    
    start >> processing_group >> validation_group >> end
```

## 6. BranchPythonOperator를 이용한 조건부 실행

```python
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from datetime import datetime

def check_condition(**context):
    """조건에 따라 다른 Task로 분기"""
    execution_date = context['execution_date']
    day_of_week = execution_date.weekday()
    
    # 월요일이면 task_a, 아니면 task_b
    if day_of_week == 0:
        return 'task_a'
    else:
        return 'task_b'

def task_a():
    print("Task A executed (Monday)")

def task_b():
    print("Task B executed (Not Monday)")

with DAG(
    'branch_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
) as dag:
    
    branch_task = BranchPythonOperator(
        task_id='branch',
        python_callable=check_condition,
        provide_context=True,
    )
    
    task_a_op = PythonOperator(
        task_id='task_a',
        python_callable=task_a,
    )
    
    task_b_op = PythonOperator(
        task_id='task_b',
        python_callable=task_b,
    )
    
    branch_task >> [task_a_op, task_b_op]
```

## 7. Sensor를 이용한 대기 처리

```python
from airflow import DAG
from airflow.sensors.filesystem import FileSensor
from airflow.sensors.python import PythonSensor
from airflow.operators.python import PythonOperator
from datetime import datetime
import os

def check_data_ready():
    """데이터 준비 여부 확인"""
    # 실제로는 데이터베이스나 API를 체크
    return os.path.exists('/tmp/data_ready.flag')

def process_data():
    print("데이터 처리 시작")

with DAG(
    'sensor_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@hourly',
    catchup=False,
) as dag:
    
    # 파일이 생성될 때까지 대기
    file_sensor = FileSensor(
        task_id='wait_for_file',
        filepath='/tmp/input_data.csv',
        poke_interval=30,  # 30초마다 체크
        timeout=600,  # 최대 10분 대기
        mode='poke',
    )
    
    # Python 함수로 조건 체크
    python_sensor = PythonSensor(
        task_id='wait_for_condition',
        python_callable=check_data_ready,
        poke_interval=60,
        timeout=3600,
        mode='poke',
    )
    
    process_task = PythonOperator(
        task_id='process_data',
        python_callable=process_data,
    )
    
    [file_sensor, python_sensor] >> process_task
```

## 8. Jinja 템플릿 활용

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from datetime import datetime

def print_context(**context):
    """템플릿 변수 출력"""
    print(f"Execution Date: {context['ds']}")
    print(f"Previous Execution Date: {context['prev_ds']}")
    print(f"Next Execution Date: {context['next_ds']}")

with DAG(
    'template_example',
    start_date=datetime(2024, 1, 1),
    schedule_interval='@daily',
    catchup=False,
) as dag:
    
    # Bash 명령어에서 템플릿 사용
    bash_task = BashOperator(
        task_id='templated_bash',
        bash_command="""
            echo "Execution date: {{ ds }}"
            echo "Previous date: {{ prev_ds }}"
            echo "DAG: {{ dag.dag_id }}"
            echo "Task: {{ task.task_id }}"
        """,
    )
    
    # Python에서 템플릿 사용
    python_task = PythonOperator(
        task_id='templated_python',
        python_callable=print_context,
        provide_context=True,
    )
    
    bash_task >> python_task
```

## 9. SubDAG 활용

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.subdag import SubDagOperator
from datetime import datetime

def create_subdag(parent_dag_id, child_dag_id, default_args):
    """SubDAG 생성 함수"""
    with DAG(
        dag_id=f'{parent_dag_id}.{child_dag_id}',
        default_args=default_args,
        schedule_interval=None,
        catchup=False,
    ) as subdag:
        
        def sub_task():
            print("SubDAG task executed")
        
        for i in range(3):
            PythonOperator(
                task_id=f'sub_task_{i}',
                python_callable=sub_task,
            )
    
    return subdag

# Main DAG
default_args = {
    'owner': 'airflow',
    'start_date': datetime(2024, 1, 1),
}

with DAG(
    'main_dag',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False,
) as dag:
    
    start = PythonOperator(
        task_id='start',
        python_callable=lambda: print("Start"),
    )
    
    subdag_task = SubDagOperator(
        task_id='subdag',
        subdag=create_subdag('main_dag', 'subdag', default_args),
    )
    
    end = PythonOperator(
        task_id='end',
        python_callable=lambda: print("End"),
    )
    
    start >> subdag_task >> end
```

## 10. 데이터 파이프라인 예시 코드

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from datetime import datetime, timedelta
import pandas as pd

def extract_from_source(**context):
    """소스에서 데이터 추출"""
    hook = PostgresHook(postgres_conn_id='source_db')
    
    sql = """
        SELECT * FROM sales
        WHERE created_at >= '{{ ds }}'
        AND created_at < '{{ next_ds }}'
    """
    
    df = hook.get_pandas_df(sql)
    
    # 데이터를 임시 파일로 저장
    filepath = f'/tmp/sales_{context["ds"]}.csv'
    df.to_csv(filepath, index=False)
    
    return filepath

def transform_data(**context):
    """데이터 변환"""
    ti = context['ti']
    filepath = ti.xcom_pull(task_ids='extract')
    
    # CSV 읽기
    df = pd.read_csv(filepath)
    
    # 데이터 변환 작업
    df['total_amount'] = df['quantity'] * df['price']
    df['created_date'] = pd.to_datetime(df['created_at']).dt.date
    
    # 집계
    result = df.groupby('created_date').agg({
        'total_amount': 'sum',
        'quantity': 'sum'
    }).reset_index()
    
    # 변환된 데이터 저장
    output_filepath = f'/tmp/sales_transformed_{context["ds"]}.csv'
    result.to_csv(output_filepath, index=False)
    
    return output_filepath

def load_to_warehouse(**context):
    """데이터 웨어하우스에 적재"""
    ti = context['ti']
    filepath = ti.xcom_pull(task_ids='transform')
    
    hook = PostgresHook(postgres_conn_id='warehouse_db')
    
    # CSV 데이터를 읽어서 삽입
    df = pd.read_csv(filepath)
    
    for _, row in df.iterrows():
        hook.run("""
            INSERT INTO daily_sales (date, total_amount, total_quantity)
            VALUES (%s, %s, %s)
            ON CONFLICT (date) DO UPDATE
            SET total_amount = EXCLUDED.total_amount,
                total_quantity = EXCLUDED.total_quantity
        """, parameters=(row['created_date'], row['total_amount'], row['quantity']))

default_args = {
    'owner': 'data_team',
    'depends_on_past': True,
    'start_date': datetime(2024, 1, 1),
    'email': ['data@example.com'],
    'email_on_failure': True,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    'daily_sales_pipeline',
    default_args=default_args,
    description='일일 매출 데이터 파이프라인',
    schedule_interval='0 1 * * *',  # 매일 오전 1시
    catchup=False,
    max_active_runs=1,
    tags=['sales', 'daily', 'etl'],
) as dag:
    
    # 테이블 준비
    prepare_table = PostgresOperator(
        task_id='prepare_table',
        postgres_conn_id='warehouse_db',
        sql="""
            CREATE TABLE IF NOT EXISTS daily_sales (
                date DATE PRIMARY KEY,
                total_amount DECIMAL(15, 2),
                total_quantity INTEGER,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """,
    )
    
    # ETL 프로세스
    extract = PythonOperator(
        task_id='extract',
        python_callable=extract_from_source,
        provide_context=True,
    )
    
    transform = PythonOperator(
        task_id='transform',
        python_callable=transform_data,
        provide_context=True,
    )
    
    load = PythonOperator(
        task_id='load',
        python_callable=load_to_warehouse,
        provide_context=True,
    )
    
    # 데이터 품질 체크
    quality_check = PostgresOperator(
        task_id='quality_check',
        postgres_conn_id='warehouse_db',
        sql="""
            SELECT COUNT(*) FROM daily_sales
            WHERE date = '{{ ds }}'
            AND total_amount > 0
        """,
    )
    
    # Task 의존성
    prepare_table >> extract >> transform >> load >> quality_check
```

## 참고사항

* DAG를 작성할 때는 idempotent(멱등성)하게 설계하는 게 중요함. 같은 입력에 대해 여러 번 실행해도 같은 결과가 나와야 하고, Task 간 의존성을 명확히 정의해야 함. 또한 너무 많은 Task를 하나의 DAG에 넣지 말고, 적절히 분리하는 게 관리하기 좋음.