---
key: jekyll-text-theme
title: 'OpenSearch - Shards 사이즈별 성능 테스트'
excerpt: ' OpenSearch research 😎'
tags: [Opensearch, Research]
---

# OpenSearch - Shards 사이즈별 성능 테스트

## 수행 방법

- shards 사이즈별 성능 테스트
- bulk index를 만들어 넣고 특정 조건으로 read한다.
- Index 갯수는 천만 건으로 통일한다.
- index setting

```
PUT myindex 
{
  "settings": {
    "number_of_shards": 20, -- 40, 60, 80...
    "number_of_replicas": 2
  }
}
```

- 데이터는 “value1”이라는 텍스트와 timestamp를 찍어 넣음

```
{
  "_index": "myindex",
  "_id": "32110016",
  "_score": 1.0,
  "_source": {
  "id": "value1",
  "timestamp": "2024-02-16T00:58:53.237Z"
}
```

- read는 전부 다 읽어오는걸 기준으로

```
const requestBody = {
    "query": {
      "term": {
        "id": "value1"
      } 
    } 
  }
```

## 코드

- 10개의 유저가 10000번 수행하면서 성능 측정

```
import http from 'k6/http';
import encoding from 'k6/encoding';

const username = 'admin';
const password = 'admin';

export default function () {
  const credentials = `${username}:${password}`;
  const encodedCredentials = encoding.b64encode(credentials)
  const options = {
    headers: {
      'Authorization': `Basic ${encodedCredentials}`,
      'Content-Type': 'application/json',
    },
  };

  const url = '<https://192.168.0.155:5200/myindex/_search>';

  const requestBody = {
    "query": {
      "term": {
        "id": "value1"
      }
    }
  }

  //const res = http.put(url, options);
  const res = http.post(url, JSON.stringify(requestBody), options);

  //console.log(res.status);
  //console.log(res.body);
}

export const options={
  vus: 10,
  iterations: 10000,
  discardResponseBodies: true,
  insecureSkipTLSVerify: true,
  logLevel: 'error'
}
```

## 결과


| shard cnt | avg     | p90     | p95     | min     | med     |
| :-------- | :------ | :------ | :------ | :------ | :------ |
| 1         | 6.79ms  | 11.11ms | 13.19ms | 1.13ms  | 6.15ms  |
| 20        | 17.48ms | 26.02ms | 29.26ms | 2.79ms  | 16.62ms |
| 40        | 30.76ms | 44.65ms | 50.05ms | 6.17ms  | 29.23ms |
| 50        | 38.23ms | 55.51ms | 61.88ms | 10.24ms | 36.42ms |
