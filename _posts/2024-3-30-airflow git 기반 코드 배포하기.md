---
key: jekyll-text-theme
title: 'Airflow에서 Git 기반 코드 배포'
excerpt: ' Git!! 😎'
tags: [Airflow]
---

# Airflow에서 Git 기반 코드 배포

## 개념

* Git-Sync는 Git 저장소의 DAG 파일을 주기적으로 동기화하여 Airflow에 배포하는 메커니즘
* CI/CD와 통합하여 버전 관리, 코드 리뷰, 자동 배포를 구현할 수 있음.

## 설치 (Kubernetes Helm Chart)

```
~~~bash
# values.yaml에 Git-Sync 설정
cat > values.yaml <<EOF
dags:
  gitSync:
    enabled: true
    repo: https://github.com/your-org/airflow-dags.git
    branch: main
    rev: HEAD
    depth: 1
    subPath: "dags"
    # SSH Key 사용 (Private Repo)
    sshKeySecret: airflow-ssh-git-secret
    # 동기화 주기
    wait: 60  # 60초마다 체크
    # 자격증명
    credentialsSecret: git-credentials
    # Known Hosts
    knownHosts: |
      github.com ssh-rsa AAAA...

# SSH Key Secret 생성
kubectl create secret generic airflow-ssh-git-secret \
  --from-file=gitSshKey=/path/to/id_rsa \
  -n airflow
EOF

helm install airflow apache-airflow/airflow -n airflow -f values.yaml
```

## 원리
1. **Git-Sync 컨테이너**가 Scheduler/Webserver Pod와 함께 실행됩니다.
2. 주기적으로 **Git 저장소를 Pull**합니다.
3. DAG 파일을 **공유 볼륨**에 복사합니다.
4. Airflow가 **새로운 DAG를 자동으로 감지**합니다.
5. **버전 관리**와 **롤백**이 용이합니다.

## 코드

* 실무에서 활용한 코드

### 1. Git 저장소 구조 - 테스트용

```
airflow-dags/
├── dags/
│   ├── production/
│   │   ├── daily_etl.py
│   │   ├── hourly_sync.py
│   │   └── weekly_report.py
│   ├── staging/
│   │   └── test_pipeline.py
│   └── common/
│       ├── __init__.py
│       ├── hooks/
│       ├── operators/
│       └── utils.py
├── plugins/
│   └── custom_operators/
├── tests/
│   ├── dags/
│   └── plugins/
├── .github/
│   └── workflows/
│       └── ci.yml
├── requirements.txt
└── README.md
```

## 2. CI/CD 파이프라인 (GitHub Actions)


```yaml
# .github/workflows/ci.yml
name: Airflow DAG CI/CD

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: |
          pip install apache-airflow==2.7.0
          pip install -r requirements.txt
          pip install pytest pylint black
      
      - name: Lint DAGs
        run: |
          pylint dags/ --disable=C,R
          black --check dags/
      
      - name: Test DAG Integrity
        run: |
          python -m pytest tests/dags/test_dag_integrity.py
      
      - name: Test DAG Loading
        run: |
          export AIRFLOW_HOME=$(pwd)
          airflow db init
          python tests/dags/test_dag_loading.py
  
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      
      - name: Trigger Airflow Sync
        run: |
          # Git-Sync가 자동으로 감지하므로 별도 액션 불필요
          # 또는 Kubernetes Job으로 즉시 동기화 트리거
          kubectl rollout restart deployment/airflow-scheduler -n airflow
```

### 3. DAG 무결성 테스트


```python
# tests/dags/test_dag_integrity.py
import pytest
import os
from airflow.models import DagBag

def test_no_import_errors():
    """DAG 로딩 시 import 에러가 없는지 확인"""
    dag_bag = DagBag(dag_folder='dags/', include_examples=False)
    
    assert len(dag_bag.import_errors) == 0, \
        f"DAG import errors: {dag_bag.import_errors}"

def test_all_dags_have_tags():
    """모든 DAG가 태그를 가지고 있는지 확인"""
    dag_bag = DagBag(dag_folder='dags/', include_examples=False)
    
    for dag_id, dag in dag_bag.dags.items():
        assert len(dag.tags) > 0, \
            f"DAG {dag_id} has no tags"

def test_all_dags_have_owners():
    """모든 DAG가 소유자를 가지고 있는지 확인"""
    dag_bag = DagBag(dag_folder='dags/', include_examples=False)
    
    for dag_id, dag in dag_bag.dags.items():
        assert dag.default_args.get('owner') is not None, \
            f"DAG {dag_id} has no owner"

def test_no_default_pool():
    """default pool 사용을 금지"""
    dag_bag = DagBag(dag_folder='dags/', include_examples=False)
    
    for dag_id, dag in dag_bag.dags.items():
        for task in dag.tasks:
            assert task.pool != 'default_pool', \
                f"Task {task.task_id} in DAG {dag_id} uses default pool"

def test_retries_configured():
    """모든 Task가 재시도 설정을 가지는지 확인"""
    dag_bag = DagBag(dag_folder='dags/', include_examples=False)
    
    for dag_id, dag in dag_bag.dags.items():
        for task in dag.tasks:
            assert task.retries is not None and task.retries > 0, \
                f"Task {task.task_id} in DAG {dag_id} has no retry configured"
```

### 4. 환경별 설정 분리

```python
# dags/common/config.py
import os

ENVIRONMENT = os.getenv('AIRFLOW_ENV', 'production')

CONFIGS = {
    'production': {
        'database': 'postgresql://prod-db:5432/analytics',
        's3_bucket': 'prod-data-lake',
        'email_recipients': ['data-team@company.com'],
    },
    'staging': {
        'database': 'postgresql://staging-db:5432/analytics',
        's3_bucket': 'staging-data-lake',
        'email_recipients': ['dev-team@company.com'],
    },
    'development': {
        'database': 'postgresql://localhost:5432/analytics',
        's3_bucket': 'dev-data-lake',
        'email_recipients': ['developer@company.com'],
    },
}

def get_config(key):
    """환경별 설정 가져오기"""
    return CONFIGS[ENVIRONMENT].get(key)

# dags/daily_etl.py
from common.config import get_config

default_args = {
    'owner': 'data-team',
    'email': get_config('email_recipients'),
}

# Task에서 사용
def extract_data(**context):
    database_url = get_config('database')
    # ...
```

### 5. DAG Versioning


```python
# dags/versioned_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

# Git Commit Hash를 DAG ID에 포함
GIT_COMMIT = os.getenv('GIT_COMMIT_SHA', 'unknown')[:7]
DAG_VERSION = 'v2.1.0'

dag = DAG(
    f'data_pipeline_{DAG_VERSION}',
    default_args={
        'owner': 'data-team',
        'start_date': datetime(2024, 1, 1),
    },
    schedule_interval='@daily',
    catchup=False,
    tags=['production', DAG_VERSION, GIT_COMMIT],
    # DAG 문서화
    doc_md=f"""
    # Data Pipeline
    
    **Version**: {DAG_VERSION}
    **Git Commit**: {GIT_COMMIT}
    **Last Updated**: {datetime.now().isoformat()}
    
    ## Description
    This DAG processes daily data...
    
    ## Changelog
    - v2.1.0: Added data quality checks
    - v2.0.0: Migrated to new data warehouse
    - v1.0.0: Initial version
    """,
)
```

### 6. 롤백 전략

```bash
# 이전 버전으로 롤백
git revert HEAD
git push origin main

# 또는 특정 커밋으로
git reset --hard <commit-hash>
git push -f origin main

# Kubernetes에서 즉시 적용
kubectl rollout restart deployment/airflow-scheduler -n airflow
kubectl rollout restart deployment/airflow-webserver -n airflow

# 특정 DAG만 pause (긴급 상황)
airflow dags pause <dag_id>
```

### 7. Branch 전략

```bash
main (production)
  ← develop (staging)
      ← feature/new-pipeline
      ← hotfix/critical-bug
```

```yaml
# Git-Sync 환경별 설정
production:
  branch: main
  
staging:
  branch: develop

# Helm values-production.yaml
dags:
  gitSync:
    branch: main
    
# Helm values-staging.yaml
dags:
  gitSync:
    branch: develop
```

## 적용할 때 고려한 점

1. **Pre-commit Hooks**: Commit 전에 자동으로 linting, 테스트를 실행해보기
2. **PR 필수**: Main 브랜치 직접 커밋을 금지하고, Pull Request + 코드 리뷰를 필수화 함.
3. **자동화된 테스트**: CI에서 DAG 로딩, import 에러, 설정 검증을 자동으로 수행하도록 함.
4. **환경 분리**: Production, Staging, Development 환경을 별도 Git 브랜치로 관리함.
5. **버전 관리**: DAG에 버전 정보를 포함하여 추적 가능성을 높임.
6. **문서화**: 각 DAG의 목적, 소유자, 변경 이력을 DAG 파일 내에 문서화 함.
7. **Secrets 분리**: 민감한 정보는 Git에 커밋하지 않고 Kubernetes Secret이나 Vault를 사용함.
