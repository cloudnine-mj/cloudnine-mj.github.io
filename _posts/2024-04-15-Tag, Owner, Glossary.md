---
key: jekyll-text-theme
title: 'Tag, Owner, Glossary'
excerpt: ' Datahub Research 😎'
tags: [Datahub]
---



# Tag, Owner, Glossary 

## 개념

* DataHub의 Tag, Owner, Glossary는 메타데이터를 조직화하고 거버넌스를 강화하는 핵심 기능
* 전략적으로 활용하면 데이터 발견성, 책임 소재, 비즈니스 의미 전달이 크게 향상됨.

## Tags (태그)

### Tag 체계 설계


```yaml
# tag_taxonomy.yml
# 태그 분류 체계

# 1. 데이터 분류 (Data Classification)
data_classification:
  - PII                    # 개인식별정보
  - Sensitive              # 민감정보
  - Confidential           # 기밀
  - Internal               # 내부용
  - Public                 # 공개

# 2. 데이터 품질 (Data Quality)
data_quality:
  - Verified               # 검증됨
  - Needs_Review           # 리뷰 필요
  - Deprecated             # 사용 중단 예정
  - Experimental           # 실험적

# 3. 업데이트 주기 (Update Frequency)
update_frequency:
  - Realtime               # 실시간
  - Hourly                 # 시간별
  - Daily                  # 일별
  - Weekly                 # 주별
  - Monthly                # 월별
  - On_Demand              # 요청 시

# 4. 중요도 (Criticality)
criticality:
  - Critical               # 매우 중요
  - High                   # 중요
  - Medium                 # 보통
  - Low                    # 낮음

# 5. 도메인 (Domain)
domain:
  - Sales                  # 매출
  - Marketing              # 마케팅
  - Finance                # 재무
  - Product                # 상품
  - Customer               # 고객

# 6. 환경 (Environment)
environment:
  - Production             # 운영
  - Staging                # 스테이징
  - Development            # 개발
```

### Tag 생성 및 관리


```python
# tag_management.py
from datahub.emitter.rest_emitter import DatahubRestEmitter
from datahub.metadata.schema_classes import (
    TagPropertiesClass,
    GlobalTagsClass,
    TagAssociationClass,
)

emitter = DatahubRestEmitter('http://localhost:8080')

def create_tag(tag_name, description, color_hex="#1890FF"):
    """태그 생성"""
    
    tag_urn = f'urn:li:tag:{tag_name}'
    
    properties = TagPropertiesClass(
        name=tag_name,
        description=description,
        colorHex=color_hex,
    )
    
    emitter.emit_mcp(tag_urn, 'tagProperties', properties)
    print(f"Created tag: {tag_name}")

# 데이터 분류 태그 생성
tags_to_create = [
    {
        'name': 'PII',
        'description': '개인식별정보를 포함하는 데이터',
        'color': '#FF4D4F',  # 빨강
    },
    {
        'name': 'Sensitive',
        'description': '민감한 비즈니스 정보',
        'color': '#FFA940',  # 주황
    },
    {
        'name': 'Verified',
        'description': '데이터 품질이 검증됨',
        'color': '#52C41A',  # 초록
    },
    {
        'name': 'Critical',
        'description': '비즈니스에 매우 중요한 데이터',
        'color': '#722ED1',  # 보라
    },
]

for tag in tags_to_create:
    create_tag(tag['name'], tag['description'], tag['color'])
```

### 자동 태깅

~~~python
# auto_tagging.py
from datahub.ingestion.graph.client import DataHubGraph
from datahub.metadata.schema_classes import GlobalTagsClass, TagAssociationClass
import re

def auto_tag_pii_columns():
    """PII 컬럼 자동 태깅"""
    
    graph = DataHubGraph(config=DatahubClientConfig(server='http://localhost:8080'))
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    # PII 패턴
    pii_patterns = [
        r'.*email.*',
        r'.*phone.*',
        r'.*ssn.*',
        r'.*social.*security.*',
        r'.*credit.*card.*',
        r'.*passport.*',
        r'.*driver.*license.*',
    ]
    
    # 모든 데이터셋 조회
    datasets = graph.search(entity_types=['dataset'], query='*', count=1000)
    
    for dataset in datasets:
        dataset_urn = dataset.entity_urn
        
        # 스키마 조회
        schema = graph.get_aspect(dataset_urn, SchemaMetadataClass)
        
        if not schema:
            continue
        
        # 각 필드 확인
        for field in schema.fields:
            field_path = field.fieldPath.lower()
            
            # PII 패턴 매칭
            is_pii = any(re.match(pattern, field_path) for pattern in pii_patterns)
            
            if is_pii:
                # 필드에 PII 태그 추가
                field_urn = f'urn:li:schemaField:({dataset_urn},{field.fieldPath})'
                
                tags = GlobalTagsClass(
                    tags=[TagAssociationClass(tag='urn:li:tag:PII')]
                )
                
                emitter.emit_mcp(field_urn, 'globalTags', tags)
                print(f"Tagged PII: {dataset_urn} - {field.fieldPath}")

# 실행
auto_tag_pii_columns()

# 업데이트 빈도 기반 자동 태깅
def auto_tag_update_frequency():
    """Airflow 스케줄 기반 업데이트 빈도 태깅"""
    
    graph = DataHubGraph(config=DatahubClientConfig(server='http://localhost:8080'))
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    # Airflow DAG 조회
    dags = graph.search(entity_types=['dataFlow'], query='*', count=1000)
    
    for dag in dags:
        dag_urn = dag.entity_urn
        
        # DAG 정보 조회
        flow_info = graph.get_aspect(dag_urn, DataFlowInfoClass)
        
        if not flow_info:
            continue
        
        # 스케줄 파싱
        schedule = flow_info.customProperties.get('schedule', '')
        
        # 태그 결정
        tag = None
        if '@hourly' in schedule or '0 * * * *' in schedule:
            tag = 'Hourly'
        elif '@daily' in schedule or '0 0 * * *' in schedule:
            tag = 'Daily'
        elif '@weekly' in schedule:
            tag = 'Weekly'
        elif '@monthly' in schedule:
            tag = 'Monthly'
        
        if tag:
            tags = GlobalTagsClass(
                tags=[TagAssociationClass(tag=f'urn:li:tag:{tag}')]
            )
            emitter.emit_mcp(dag_urn, 'globalTags', tags)
            print(f"Tagged {tag}: {dag_urn}")
```

### Ownership (소유자)

#### 소유자 타입
```
소유자 유형:
1. Technical Owner (기술 소유자)
   - 데이터 파이프라인 개발/유지보수 담당
   - 기술적 이슈 해결

2. Business Owner (비즈니스 소유자)
   - 데이터 비즈니스 의미 정의
   - 데이터 품질 기준 결정

3. Data Steward (데이터 스튜어드)
   - 데이터 거버넌스 정책 집행
   - 메타데이터 품질 관리

4. Data Owner (데이터 소유자)
   - 전체 책임자
   - 접근 권한 승인
~~~

### 소유자 할당

```python
# ownership_management.py
from datahub.emitter.mce_builder import make_user_urn, make_group_urn
from datahub.metadata.schema_classes import (
    OwnershipClass,
    OwnerClass,
    OwnershipTypeClass,
)

def assign_owners(dataset_urn, owners_config):
    """소유자 할당"""
    
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    owners = []
    
    for owner_config in owners_config:
        owner_type = owner_config['type']
        owner_id = owner_config['id']
        
        # 사용자 또는 그룹
        if owner_config.get('is_group', False):
            owner_urn = make_group_urn(owner_id)
        else:
            owner_urn = make_user_urn(owner_id)
        
        # 소유자 타입 매핑
        type_mapping = {
            'technical': OwnershipTypeClass.TECHNICAL_OWNER,
            'business': OwnershipTypeClass.BUSINESS_OWNER,
            'data_steward': OwnershipTypeClass.DATA_STEWARD,
            'data_owner': OwnershipTypeClass.DATAOWNER,
        }
        
        owners.append(
            OwnerClass(
                owner=owner_urn,
                type=type_mapping[owner_type],
            )
        )
    
    ownership = OwnershipClass(owners=owners)
    emitter.emit_mcp(dataset_urn, 'ownership', ownership)

# 사용 예제
dataset_urn = 'urn:li:dataset:(urn:li:dataPlatform:snowflake,analytics.sales_summary,PROD)'

assign_owners(dataset_urn, [
    {
        'type': 'data_owner',
        'id': 'sales-team',
        'is_group': True,
    },
    {
        'type': 'technical',
        'id': 'john.doe',
        'is_group': False,
    },
    {
        'type': 'business',
        'id': 'sales-manager',
        'is_group': False,
    },
])
```

### 규칙 기반 소유자 할당

~~~python
# rule_based_ownership.py
import re

def determine_owners(dataset_urn, ownership_rules):
    """규칙 기반 소유자 결정"""
    
    platform, name, env = parse_urn(dataset_urn)
    
    owners = []
    
    for rule in ownership_rules:
        # 플랫폼 매칭
        if rule.get('platform') and rule['platform'] != platform:
            continue
        
        # 이름 패턴 매칭
        if rule.get('name_pattern'):
            if not re.match(rule['name_pattern'], name):
                continue
        
        # 환경 매칭
        if rule.get('env') and rule['env'] != env:
            continue
        
        # 규칙 매칭 성공 - 소유자 추가
        for owner in rule['owners']:
            owners.append({
                'type': owner['type'],
                'id': owner['id'],
                'is_group': owner.get('is_group', False),
            })
    
    return owners

# 소유자 규칙 정의
ownership_rules = [
    {
        'platform': 'snowflake',
        'name_pattern': '.*\\.fact_.*',
        'env': 'PROD',
        'owners': [
            {'type': 'technical', 'id': 'data-engineering', 'is_group': True},
            {'type': 'business', 'id': 'analytics-team', 'is_group': True},
        ],
    },
    {
        'platform': 'snowflake',
        'name_pattern': '.*\\.dim_.*',
        'env': 'PROD',
        'owners': [
            {'type': 'technical', 'id': 'data-engineering', 'is_group': True},
        ],
    },
    {
        'platform': 'mysql',
        'name_pattern': 'sales\\..*',
        'owners': [
            {'type': 'data_owner', 'id': 'sales-team', 'is_group': True},
        ],
    },
]

# 모든 데이터셋에 소유자 할당
def auto_assign_all_owners():
    """모든 데이터셋에 규칙 기반 소유자 할당"""
    
    graph = DataHubGraph(config=DatahubClientConfig(server='http://localhost:8080'))
    
    datasets = graph.search(entity_types=['dataset'], query='*', count=1000)
    
    for dataset in datasets:
        dataset_urn = dataset.entity_urn
        
        # 이미 소유자가 있으면 스킵
        ownership = graph.get_aspect(dataset_urn, OwnershipClass)
        if ownership and len(ownership.owners) > 0:
            continue
        
        # 규칙 기반 소유자 결정
        owners = determine_owners(dataset_urn, ownership_rules)
        
        if owners:
            assign_owners(dataset_urn, owners)
            print(f"Assigned owners to {dataset_urn}")

auto_assign_all_owners()
```

### Glossary (용어 사전)

#### Glossary 구조 설계
```
Business Glossary 구조:

├── Sales (판매)
│   ├── Customer (고객)
│   │   ├── Active Customer (활성 고객)
│   │   ├── Churn (이탈)
│   │   └── LTV (고객 생애 가치)
│   ├── Revenue (매출)
│   │   ├── Gross Revenue (총 매출)
│   │   ├── Net Revenue (순 매출)
│   │   └── ARR (연간 반복 매출)
│   └── Order (주문)
│       ├── Conversion Rate (전환율)
│       └── AOV (평균 주문 금액)
│
├── Marketing (마케팅)
│   ├── Campaign (캠페인)
│   ├── CTR (클릭률)
│   ├── CAC (고객 획득 비용)
│   └── ROAS (광고 수익률)
│
└── Product (상품)
    ├── SKU (재고 관리 단위)
    ├── Category (카테고리)
    └── Inventory (재고)
~~~

### Glossary Term 생성


```python
# glossary_management.py
from datahub.emitter.mce_builder import make_glossary_term_urn
from datahub.metadata.schema_classes import (
    GlossaryTermInfoClass,
    GlossaryTermsClass,
    GlossaryTermAssociationClass,
)

def create_glossary_term(
    term_id,
    name,
    definition,
    parent_term=None,
    related_terms=None,
    custom_properties=None,
):
    """용어 사전 항목 생성"""
    
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    term_urn = make_glossary_term_urn(term_id)
    
    term_info = GlossaryTermInfoClass(
        definition=definition,
        name=name,
        termSource='Business Glossary',
        sourceRef=None,
        sourceUrl=None,
        rawSchema=None,
        customProperties=custom_properties or {},
    )
    
    # 상위 용어 (계층 구조)
    if parent_term:
        term_info.parentNode = make_glossary_term_urn(parent_term)
    
    # 관련 용어
    if related_terms:
        term_info.relatedTerms = [
            make_glossary_term_urn(rt) for rt in related_terms
        ]
    
    emitter.emit_mcp(term_urn, 'glossaryTermInfo', term_info)
    print(f"Created term: {term_id}")

# 용어 사전 생성 예제
glossary_terms = [
    {
        'id': 'Customer',
        'name': '고객',
        'definition': '우리 서비스를 사용하거나 제품을 구매한 개인 또는 기업',
        'custom_properties': {
            'owner': 'sales-team',
            'approved_by': 'cto',
            'approval_date': '2024-01-01',
        },
    },
    {
        'id': 'ActiveCustomer',
        'name': '활성 고객',
        'definition': '최근 90일 이내에 구매 또는 로그인 이력이 있는 고객',
        'parent_term': 'Customer',
        'custom_properties': {
            'measurement_period': '90 days',
            'last_updated': '2024-01-15',
        },
    },
    {
        'id': 'Churn',
        'name': '고객 이탈',
        'definition': '활성 고객에서 비활성 고객으로 전환되는 것. 90일간 활동이 없으면 이탈로 간주',
        'parent_term': 'Customer',
        'related_terms': ['ActiveCustomer', 'LTV'],
        'custom_properties': {
            'calculation': 'churned_customers / total_customers * 100',
        },
    },
    {
        'id': 'LTV',
        'name': '고객 생애 가치',
        'definition': '한 고객이 생애 동안 회사에 가져다주는 총 매출',
        'parent_term': 'Customer',
        'related_terms': ['Revenue', 'Churn'],
        'custom_properties': {
            'formula': 'Average Order Value × Purchase Frequency × Customer Lifespan',
        },
    },
    {
        'id': 'Revenue',
        'name': '매출',
        'definition': '제품 또는 서비스 판매로 인한 수익',
    },
    {
        'id': 'GrossRevenue',
        'name': '총 매출',
        'definition': '환불, 할인 등을 차감하기 전의 총 판매 금액',
        'parent_term': 'Revenue',
    },
    {
        'id': 'NetRevenue',
        'name': '순 매출',
        'definition': '환불, 할인, 세금 등을 차감한 후의 실제 수익',
        'parent_term': 'Revenue',
        'related_terms': ['GrossRevenue'],
        'custom_properties': {
            'formula': 'Gross Revenue - Refunds - Discounts - Taxes',
        },
    },
]

for term in glossary_terms:
    create_glossary_term(
        term_id=term['id'],
        name=term['name'],
        definition=term['definition'],
        parent_term=term.get('parent_term'),
        related_terms=term.get('related_terms'),
        custom_properties=term.get('custom_properties'),
    )
```

### 데이터셋에 용어 연결


```python
# link_terms_to_datasets.py

def link_term_to_dataset(dataset_urn, term_ids):
    """데이터셋에 용어 연결"""
    
    emitter = DatahubRestEmitter('http://localhost:8080')
    
    terms = GlossaryTermsClass(
        terms=[
            GlossaryTermAssociationClass(urn=make_glossary_term_urn(term_id))
            for term_id in term_ids
        ],
        auditStamp=None,
    )
    
    emitter.emit_mcp(dataset_urn, 'glossaryTerms', terms)
    print(f"Linked terms {term_ids} to {dataset_urn}")

# 자동 연결 (이름 기반)
def auto_link_terms():
    """데이터셋/컬럼 이름 기반 자동 용어 연결"""
    
    graph = DataHubGraph(config=DatahubClientConfig(server='http://localhost:8080'))
    
    # 용어-패턴 매핑
    term_patterns = {
        'Customer': [r'.*customer.*', r'.*user.*', r'.*client.*'],
        'Revenue': [r'.*revenue.*', r'.*sales.*', r'.*income.*'],
        'Order': [r'.*order.*', r'.*purchase.*', r'.*transaction.*'],
        'Product': [r'.*product.*', r'.*item.*', r'.*sku.*'],
    }
    
    datasets = graph.search(entity_types=['dataset'], query='*', count=1000)
    
    for dataset in datasets:
        dataset_urn = dataset.entity_urn
        platform, name, env = parse_urn(dataset_urn)
        
        matched_terms = []
        
        # 데이터셋 이름 매칭
        for term, patterns in term_patterns.items():
            if any(re.match(pattern, name.lower()) for pattern in patterns):
                matched_terms.append(term)
        
        if matched_terms:
            link_term_to_dataset(dataset_urn, matched_terms)

auto_link_terms()
```

## 통합 거버넌스 워크플로우


```python
# governance_workflow.py

def governance_onboarding(dataset_urn):
    """새 데이터셋의 거버넌스 온보딩"""
    
    print(f"\n=== Governance Onboarding: {dataset_urn} ===\n")
    
    # 1. 자동 태깅
    print("Step 1: Auto-tagging...")
    auto_tags = determine_tags(dataset_urn)
    apply_tags(dataset_urn, auto_tags)
    print(f"  Applied tags: {auto_tags}")
    
    # 2. 소유자 할당
    print("Step 2: Assigning owners...")
    owners = determine_owners(dataset_urn, ownership_rules)
    assign_owners(dataset_urn, owners)
    print(f"  Assigned owners: {[o['id'] for o in owners]}")
    
    # 3. 용어 연결
    print("Step 3: Linking glossary terms...")
    terms = determine_terms(dataset_urn)
    link_term_to_dataset(dataset_urn, terms)
    print(f"  Linked terms: {terms}")
    
    # 4. 품질 체크
    print("Step 4: Running quality checks...")
    quality_score = run_quality_checks(dataset_urn)
    print(f"  Quality score: {quality_score}")
    
    # 5. 알림
    print("Step 5: Notifying stakeholders...")
    notify_stakeholders(dataset_urn, owners, tags=auto_tags, quality=quality_score)
    
    print(f"\n=== Onboarding Complete ===\n")

def determine_tags(dataset_urn):
    """데이터셋에 적용할 태그 결정"""
    
    graph = DataHubGraph(config=DatahubClientConfig(server='http://localhost:8080'))
    
    tags = []
    
    # 스키마 기반 태그
    schema = graph.get_aspect(dataset_urn, SchemaMetadataClass)
    if schema:
        for field in schema.fields:
            field_name = field.fieldPath.lower()
            
            # PII 감지
            if any(pii in field_name for pii in ['email', 'phone', 'ssn', 'passport']):
                tags.append('PII')
                break
    
    # 이름 기반 태그
    platform, name, env = parse_urn(dataset_urn)
    
    if 'fact_' in name:
        tags.append('Fact_Table')
    elif 'dim_' in name:
        tags.append('Dimension_Table')
    
    if env == 'PROD':
        tags.append('Production')
    
    # 중복 제거
    return list(set(tags))

# DataHub Actions와 연동
# actions_config.yml에 추가:
"""
actions:
  - name: governance_onboarding
    type: custom
    module: governance_workflow
    class: GovernanceOnboardingAction
    event_type: EntityChangeEvent
    event:
      entity_type: dataset
      operation: CREATE
"""
```

## 적용 시 고려해야 할 점

1. **태그 표준화**: 조직 전체가 사용할 태그 체계를 먼저 정의
2. **소유자 책임**: 소유자 역할과 책임을 명확히 정의하고 공유
3. **용어 사전 관리**: 비즈니스 팀과 협력하여 용어 정의 작성
4. **자동화 우선**: 가능한 한 자동으로 태그, 소유자, 용어 할당
5. **주기적 리뷰**: 분기별로 메타데이터 품질 리뷰 수행
6. **교육**: 팀원들에게 올바른 사용법 교육