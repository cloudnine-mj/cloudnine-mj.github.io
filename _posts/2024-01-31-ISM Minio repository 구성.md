---
key: jekyll-text-theme
title: 'ISM Minio repository 구성'
excerpt: ' OpenSearch cold storage 😎'
tags: [Opensearch, Cold storage]
---



# ISM + minio repository 구성

 

- 테스트하는 프로그램은 60초에 100개씩 인덱스를 넣는 간단한 python 프로그램으로 해결해보려고 함.

| k8s Namespace      | opensearch-cold                                              | ism policy      | cold-policy         |
| ------------------ | ------------------------------------------------------------ | --------------- | ------------------- |
| Index name         | cold-index                                                   | index template  | cold-index-template |
| Index name pattern | cold-index-2024.01.31-000001                                 | repository name | minio-repository    |
| snapshot pattern   | cold_snapshot-cold-index-2024.01.31-000003-2024.01.31-06:07:46.977 |                 |                     |

 

## test.py

 

- nohup으로 k8s master에서 계속 쏠 수 있게 함.

 

```
# test.py
import json
import time
from datetime import datetime
from faker import Faker
from opensearchpy import OpenSearch

host = '10.0.0.150'
port = 31920
auth = ('admin', 'admin')
ca_certs_path = '/root/hunjun/opensearch/root-ca.pem'

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

while True:
  current_time = datetime.now().isoformat()
  for _ in range(100):
    document = {
      'name': fake.name(),
      'address': fake.address(),
      'email': fake.email(),
    }

    response = client.index(
      index = 'cold-index',
      body=document 
    )

  print(f"{current_time} {response}")

  time.sleep(60)
```

 

# ISM 정책, 설정

 

## ism policy

 

- ISM 정책은 doc count로 테스트
- min_doc_count를 1000으로 설정해서 index당 1000개가 넘으면 snapshot을 찍고 rollover 실행

 

```
PUT _plugins/_ism/policies/cold-policy
{
  "policy": {
    "description": "cold index rollover",
    "default_state": "rollover",
    "states": [
      {
        "name": "rollover",
        "actions": [
          {
            "rollover": {
              "min_doc_count": 1000
            }
          }
        ],
        "transitions": [
          {
            "state_name": "create_snapshot",
            "conditions": {
              "min_doc_count": 1000
            }
          }
        ]
      },
      {
        "name": "create_snapshot",
        "actions": [
          {
            "snapshot": {
              "repository": "minio-repository",  
              "snapshot": "cold_snapshot-{{ctx.index}}"
            }
          }
        ],
        "transitions": [
          {
            "state_name": "wait"
          }
        ]
      },
      {
        "name": "wait",
        "actions": [],
        "transitions": []
      }
    ],
    "ism_template": {
      "index_patterns": ["cold-*"],
      "priority": 100
    }
  }
}
```

 

## Index template

 

- shard, replica는 테스트용으로 1 설정

 

```
PUT _index_template/cold-index-template
{
  "index_patterns": [
    "cold-index-*"
  ],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "plugins.index_state_management.rollover_alias": "cold-index"
    }
  }
}
```

 

## Index create

 

```
PUT %3Ccold-index-%7Bnow%2Fd%7D-000001%3E
{
  "aliases": {
    "cold-index": {
      "is_write_index": true
    }
  }
}
```

 

# 확인

 

## Repository 확인

 

```
GET _snapshot/minio-repository
{
  "minio-repository": {
    "type": "s3",
    "settings": {
      "bucket": "opensearch"
    }
  }
}
```

 

## policy 확인

 

```
GET _plugins/_ism/policies
{
  "policies": [
    {
      "_id": "cold-policy",
      "_seq_no": 784,
      "_primary_term": 1,
      "policy": {
        "policy_id": "cold-policy",
        "description": "cold index rollover",
        "last_updated_time": 1706754833751,
        "schema_version": 17,
        "error_notification": null,
        "default_state": "rollover",
        "states": [
          {
            "name": "rollover",
            "actions": [
              {
                "retry": {
                  "count": 3,
                  "backoff": "exponential",
                  "delay": "1m"
                },
                "rollover": {
                  "min_doc_count": 1000
                }
              }
            ],
            "transitions": [
              {
                "state_name": "create_snapshot",
                "conditions": {
                  "min_doc_count": 1000
                }
              }
            ]
          },
          {
            "name": "create_snapshot",
            "actions": [
              {
                "retry": {
                  "count": 3,
                  "backoff": "exponential",
                  "delay": "1m"
                },
                "snapshot": {
                  "repository": "minio-repository",
                  "snapshot": "cold_snapshot-{{ctx.index}}"
                }
              }
            ],
            "transitions": [
              {
                "state_name": "wait"
              }
            ]
          },
          {
            "name": "wait",
            "actions": [],
            "transitions": []
          }
        ],
        "ism_template": [
          {
            "index_patterns": [
              "cold-*"
            ],
            "priority": 100,
            "last_updated_time": 1706754833751
          }
        ]
      }
    }
  ],
  "total_policies": 1
}
```

 

## index template

 

```
GET _index_template/cold-index-template
{
  "index_templates": [
    {
      "name": "cold-index-template",
      "index_template": {
        "index_patterns": [
          "cold-index-*"
        ],
        "template": {
          "settings": {
            "index": {
              "number_of_shards": "1",
              "number_of_replicas": "1",
              "plugins": {
                "index_state_management": {
                  "rollover_alias": "cold-index"
                }
              }
            }
          }
        },
        "composed_of": []
      }
    }
  ]
}
```

 

## Alias 확인

 

```
GET cold-index
{
  "cold-index-2024.02.01-000001": {
    "aliases": {
      "cold-index": {
        "is_write_index": true
      }
    },
    "mappings": {
      "properties": {
        "address": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "email": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        },
        "name": {
          "type": "text",
          "fields": {
            "keyword": {
              "type": "keyword",
              "ignore_above": 256
            }
          }
        }
      }
    },
    "settings": {
      "index": {
        "number_of_shards": "1",
        "plugins": {
          "index_state_management": {
            "rollover_alias": "cold-index"
          }
        },
        "provided_name": "<cold-index-{now/d}-000001>",
        "creation_date": "1706754840807",
        "number_of_replicas": "1",
        "uuid": "nw86Vq6mRFe9ICT-bBULRQ",
        "version": {
          "created": "136277827"
        }
      }
    }
  }
}
```

 