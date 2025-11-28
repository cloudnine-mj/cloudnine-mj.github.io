---
key: jekyll-text-theme
title: 'Airflow + KubernetesExecutor'
excerpt: ' 유연하게, 확장가능하게 😎'
tags: [Airflow]
---

# Airflow + KubernetesExecutor'

## 개념

* KubernetesExecutor는 각 Task를 별도의 Kubernetes Pod로 실행하는 Executor
* Task마다 독립된 환경을 제공하므로 리소스 격리, 확장성, 다양한 의존성 관리가 가능함.

## 설치 (Helm Chart 사용)


```bash
# Helm 저장소 추가
helm repo add apache-airflow https://airflow.apache.org
helm repo update

# values.yaml 생성
cat > values.yaml <<EOF
executor: KubernetesExecutor

# PostgreSQL 설정
postgresql:
  enabled: true
  postgresqlPassword: airflow123

# Redis (KubernetesExecutor는 불필요하지만 선택적)
redis:
  enabled: false

# 웹서버 설정
webserver:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 1000m
      memory: 2Gi

# 스케줄러 설정
scheduler:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 1Gi

# DAG 동기화 (Git-Sync)
dags:
  gitSync:
    enabled: true
    repo: https://github.com/your-org/airflow-dags.git
    branch: main
    subPath: "dags"
EOF

# Airflow 배포
kubectl create namespace airflow
helm install airflow apache-airflow/airflow -n airflow -f values.yaml
```

## 원리

1. **스케줄러**가 실행할 Task를 선택합니다.
2. **KubernetesExecutor**가 Task를 위한 Pod 정의를 생성합니다.
3. Kubernetes API를 통해 **Pod를 생성**하고 Task를 실행합니다.
4. Task 완료 후 Pod는 자동으로 삭제됩니다.
5. 결과는 Airflow 메타데이터 DB에 기록됩니다.

## 코드 

* 업무하면서 활용했던 코드

```python
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from kubernetes.client import models as k8s
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'start_date': datetime(2024, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'kubernetes_pipeline',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False,
)

# 볼륨 설정
volume = k8s.V1Volume(
    name='data-volume',
    persistent_volume_claim=k8s.V1PersistentVolumeClaimVolumeSource(
        claim_name='airflow-data-pvc'
    ),
)

volume_mount = k8s.V1VolumeMount(
    name='data-volume',
    mount_path='/data',
    sub_path=None,
)

# 리소스가 많이 필요한 Task
heavy_processing = KubernetesPodOperator(
    task_id='heavy_processing',
    name='heavy-processing-pod',
    namespace='airflow',
    image='your-repo/data-processor:latest',
    cmds=['python', '/app/heavy_process.py'],
    arguments=['--date', '{{ ds }}'],
    labels={'app': 'airflow', 'task': 'heavy-processing'},
    # 리소스 요청 및 제한
    container_resources=k8s.V1ResourceRequirements(
        requests={'memory': '4Gi', 'cpu': '2000m'},
        limits={'memory': '8Gi', 'cpu': '4000m'},
    ),
    # GPU 사용 예제
    # node_selector={'accelerator': 'nvidia-tesla-v100'},
    volumes=[volume],
    volume_mounts=[volume_mount],
    # 환경 변수
    env_vars={
        'DATABASE_URL': 'postgresql://user:pass@db:5432/dbname',
        'S3_BUCKET': 'my-data-bucket',
    },
    # Secret 사용
    secrets=[
        k8s.V1EnvVar(
            name='API_KEY',
            value_from=k8s.V1EnvVarSource(
                secret_key_ref=k8s.V1SecretKeySelector(
                    name='api-secrets',
                    key='api-key'
                )
            )
        )
    ],
    is_delete_operator_pod=True,  # 완료 후 Pod 삭제
    get_logs=True,  # 로그 수집
    dag=dag,
)

# 가벼운 Task
light_processing = KubernetesPodOperator(
    task_id='light_processing',
    name='light-processing-pod',
    namespace='airflow',
    image='your-repo/data-validator:latest',
    cmds=['python', '/app/validate.py'],
    container_resources=k8s.V1ResourceRequirements(
        requests={'memory': '512Mi', 'cpu': '250m'},
        limits={'memory': '1Gi', 'cpu': '500m'},
    ),
    dag=dag,
)

heavy_processing >> light_processing
```

## 적용할 때 고려했던 점

1. **리소스 최적화**: Task별로 적절한 CPU/메모리를 설정하여 클러스터 리소스를 효율적으로 사용하는 것이 좋음.
2. **이미지 최적화**: Docker 이미지를 최소화하여 Pod 시작 시간을 단축하는 것이 좋음.
3. **PVC 활용**: Task 간 데이터 공유가 필요하면 Persistent Volume을 사용해야 함.
4. **Node Selector**: GPU, 고성능 CPU 등 특정 노드에서 실행이 필요하면 node_selector를 활용함.
5. **Pod Affinity**: 관련 Task들을 같은 노드에서 실행하여 데이터 전송 비용을 줄이도록 해야함.