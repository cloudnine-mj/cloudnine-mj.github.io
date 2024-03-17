---
key: jekyll-text-theme
title: 'OpenSearch Index 생성 Query'
excerpt: ' OpenSearch research 😎'
tags: [Opensearch, ELK, Research]
---

# OpenSearch Index 생성

## OpenSearch Index 생성 쿼리

* Index 생성 시 한꺼번에 모든 Index를 생성하기 보다는 Index 하나씩 하나씩 생성하는 것이 더 효율적이다. (한꺼번에 생성해보니 에러가 발생했던 경험이 있음...)

```
PUT _index_template/<index 명>-template
{
  "index_patterns": [
    "<index 명>-*"
  ],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "plugins.index_state_management.rollover_alias": "<index 명>"
    }
  }
}

PUT %3C<index 명>-%7Bnow%2Fd%7D-000001%3E
{
  "aliases": {
    "<index 명>": {
      "is_write_index": true
    }
  }
}
```