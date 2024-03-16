---
key: jekyll-text-theme
title: 'OpenSearch cold storage'
excerpt: ' OpenSearch cold storage research 😎'
tags: [Opensearch, Cold storage, Research]
---

👉 Docs : [https://blog.min.io/search-with-opensearch-and-minio/](https://blog.min.io/search-with-opensearch-and-minio/)


# Architecture

* Opensearch에서 대용량 데이터를 다룰 때 모든 데이터를 가져갈 수 없으므로 사용 빈도가 낮은 데이터를 분리해야 한다. 해당 데이터를 Cold Data, 반대의 경우는 Hot Data라고 통칭함.

* 일반적으로 Cold Data는 비용이 싸고 효율이 좋은 저장소에 저장하게 되므로, 분리한 데이터를 어떤 식으로 조회하냐가 중요하다.

* s3(minio)를 remote repository로 사용함으로서 느리지만 cold data를 읽게 만드는 것이 가능하다.

* Index는 ILM으로 구성해서 일정 시기가 지난 인덱스를 스냅샷으로 만들고, restore를 수행해서 searchable snapshot으로 만들어야 한다.


## 시스템 구성

### Plugin 설치

* opensearch에서는 자체적으로 s3 repository를 지원하지 않으므로 opensearch plugin을 설치해야 한다.

```
bin/opensearch-plugin install repository-s3
```

### jvm option

* searchable snapshot을 사용하려면 experimental feature 옵션을 통해 기능을 오픈해야 한다. 실험적 기능이지만, 지원한 버전은 2.x 초반대로 신뢰도에 큰 문제가 있진 않아 보인다.

* jvm.option

```
-Dopensearch.experimental.feature.searchable_snapshot.enabled=true
```

### Keystore

* k8s의 secret과 같이 opensearch에서 민감한 키 정보는 암호화하여 저장할 수 있는 키스토어가 존재한다. minio의 접속 정보는 보안이 유지되어야 하므로 두 값은 키로 만들어 줘야 한다.

* 환경 변수로 설정하여 바로 keystore를 등록했다. pipe 없이 사용해서 stdin(cmd)으로 넣어주는 방식도 가능하다.

```
export MINIO_ACCESS_KEY_ID="minio"
export MINIO_SECRET_ACCESS_KEY="okestro2018"

echo $MINIO_ACCESS_KEY_ID | bin/opensearch-keystore add --stdin s3.client.default.access_key
echo $MINIO_SECRET_ACCESS_KEY | bin/opensearch-keystore add --stdin s3.client.default.secret_key
```

### Opensearch yml

* Opensearch를 구성할 때, search role을 줘야 한다. 해당 role은 searchable snapshot을 사용할 수 있게 해 주는 role.

* s3 관련 옵션은 minio endpoint와 protocol(http/https)를 넣어 주면 된다.

```
node.name: snapshots-node
node.roles: [ search ]

s3.client.default.endpoint: "10.0.15.82:4000"
s3.client.default.protocol: "http"
```

## 기능 테스트

### 인덱스 생성

* test-index라는 인덱스에 랜덤 데이터를 100개 넣었다.

### test.py

```
from faker import Faker
from opensearchpy import OpenSearch
import json

host = 'localhost'
port = 5200
auth = ('admin', 'admin')
ca_certs_path = '/home/opensearch/opensearch/config/root-cert.pem'

client = OpenSearch(
    hosts = [{'host': host, 'port': port}],
    http_compress = True,
    http_auth = auth,
    use_ssl = True,
    verify_certs = False,
    ssl_assert_hostname = False,
    ssl_show_warn = False,
    ca_certs = ca_certs_path
        )

fake = Faker()

for _ in range(100):
  document = {
    'name': fake.name(),
    'address': fake.address(),
    'email': fake.email()  
  }

  response = client.index(
    index = 'test-index',
    body=document 
  )

print(response)
```

```
GET _cat/indices
yellow open test-index                   TmdONwa7Tnuaytkl-4Db3A 1 1 100 0  62.3kb  62.3kb
green  open .opensearch-observability    qW7iJn1mTiWE2N2Ah6DQ4g 1 0   0 0    208b    208b
yellow open security-auditlog-2024.01.18 ZZ97aQlvS1ekwepzUy9OsQ 1 1   6 0 100.6kb 100.6kb
green  open .opendistro_security         yBI5RfHoS1i1kef6-9B1bA 1 0  10 0  71.7kb  71.7kb
```

## repository 생성

* snapshot을 저장하기 위한 repository를 생성한다. minio지만 s3로 생성해야 하고, bucket만 지정해 준다. ( path까지 지정할 수 있지만, 문제가 발생해서 일단 없애고 실행)

```
PUT _snapshot/minio-repository
{
  "type": "s3",
  "settings": {
    "bucket": "opensearch"
  }
}
```

## Snapshot 생성

* 생성시 별다른 옵션 없이 index만 지정해 주고 생성했다. 해당 절차는 실제로는 ILM을 적용할 예정이므로 테스트 용도로만 사용.

```
PUT _snapshot/minio-repository/test_snapshot
{
  "indices": "test-index"
}

GET _snapshot/minio-repository/test_snapshot
{
  "snapshots": [
    {
      "snapshot": "test_snapshot",
      "uuid": "1qQrIdp7R4u_S04R0g2CoQ",
      "version_id": 136277827,
      "version": "2.6.0",
      "indices": [
        "test-index"
      ],
      "data_streams": [],
      "include_global_state": true,
      "state": "SUCCESS",
      "start_time": "2024-01-18T05:53:12.937Z",
      "start_time_in_millis": 1705557192937,
      "end_time": "2024-01-18T05:53:13.538Z",
      "end_time_in_millis": 1705557193538,
      "duration_in_millis": 601,
      "failures": [],
      "shards": {
        "total": 1,
        "failed": 0,
        "successful": 1
      }
    }
  ]
}
```

## searchable snapshot restore

* snapshot을 restore 시키기 위해서는 기존의 인덱스를 제거해야 한다.( 다른 이름으로 해도 무방하긴 함 ). 제거 후 restore해서 테스트함.

```
DELETE test-index
```

* restore 시에는 storage_type을 지정해 준다.

```
POST _snapshot/minio-repository/test_snapshot/_restore
{
  "indices": "test-index",
  "storage_type": "remote_snapshot"
}
```

* 제대로 restore 됐다면, indices 정보에서 문제가 없고 index settings에서 remote snapshot이 활성화되어 있다.

```
GET test-index/_settings
{
  "test-index": {
    "settings": {
      "index": {
        "number_of_shards": "1",
        "provided_name": "test-index",
        "searchable_snapshot": {
          "repository": "minio-repository",
          "index": {
            "id": "uIXVktraT3SslqB7pXBCDQ"
          },
          "snapshot_id": {
            "name": "test_snapshot",
            "uuid": "1qQrIdp7R4u_S04R0g2CoQ"
          }
        },
        "creation_date": "1705557178025",
        "store": {
          "type": "remote_snapshot"
        },
        "number_of_replicas": "1",
        "uuid": "QYoc7XHfRqOt2r7u9_xjlg",
        "version": {
          "created": "136277827"
        }
      }
    }
  }
}
```

* remote snapshot이 아닌 인덱스 정보와는 다른 점을 확인할 수 있다.

```
GET test-index/_settings # 기존 정보
{
  "test-index": {
    "settings": {
      "index": {
        "creation_date": "1705557610303",
        "number_of_shards": "1",
        "number_of_replicas": "1",
        "uuid": "pMlr6B4ATbyfmGLv2tBCjw",
        "version": {
          "created": "136277827"
        },
        "provided_name": "test-index"
      }
    }
  }
}
```

* 해당 인덱스는 read-only 취급이기 때문에, index에 데이터를 넣으면 에러가 발생한다.

```
opensearchpy.exceptions.AuthorizationException: 
AuthorizationException(403, 
'cluster_block_exception', 'index [test-index] blocked by: 
[FORBIDDEN/13/remote index is read-only];')
```