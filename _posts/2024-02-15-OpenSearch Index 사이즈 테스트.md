---
key: jekyll-text-theme
title: 'OpenSearch - Index 사이즈 테스트'
excerpt: ' OpenSearch research 😎'
tags: [OpenSearch, Research]
---

# OpenSearch - Index 사이즈 테스트

## Index 사이즈 테스트 수행 방법

- 인덱스 사이즈별 성능 테스트
- bulk index를 만들어 넣고 특정 조건으로 read한다.
- index setting

```
PUT myindex
{
  "settings": {
    "number_of_shards": 6,
    "number_of_replicas": 1
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
  "timestamp": "2024-02-15T00:58:53.237Z"
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

- 반드시 post로 날려야 된다. get은 body를 담을 수 없음. postman은 잘되는데…
- https 서버기 때문에 basic auth와 insecure skip verify를 반드시 포함해야 한다.
- 10개의 유저가 10000번 수행하면서 성능 측정.
- csv 등 파일로 받을 수 있는데 이건 만번 수행하면 해당결과를 전부 다 기록해서 집계로 보긴 쉽지 않음.

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

- http_req_duration이 가장 중요한 포인트인데 receive, send, waiting을 합한 결과이다. 요청하고 받은 시간을 합친 결과이므로 성능테스트에 가장 중요한 팩터.

```
data_received..................: 15 MB  1.8 MB/s
     data_sent......................: 2.5 MB 289 kB/s
     http_req_blocked...............: avg=7.7µs   min=308ns  med=2.02µs  max=10.8ms  p(90)=3.78µs  p(95)=4.61µs 
     http_req_connecting............: avg=1.32µs  min=0s     med=0s      max=2.53ms  p(90)=0s      p(95)=0s     
     http_req_duration..............: avg=8.44ms  min=1.72ms med=7.7ms   max=40.49ms p(90)=13.67ms p(95)=15.91ms
       { expected_response:true }...: avg=8.44ms  min=1.72ms med=7.7ms   max=40.49ms p(90)=13.67ms p(95)=15.91ms
     http_req_failed................: 0.00%  ✓ 0          ✗ 10000
     http_req_receiving.............: avg=39.91µs min=5.6µs  med=34.88µs max=11.57ms p(90)=64.37µs p(95)=75.32µs
     http_req_sending...............: avg=15.26µs min=2.19µs med=13.12µs max=5.67ms  p(90)=24.43µs p(95)=28.86µs
     http_req_tls_handshaking.......: avg=3.99µs  min=0s     med=0s      max=8.15ms  p(90)=0s      p(95)=0s     
     http_req_waiting...............: avg=8.39ms  min=1.67ms med=7.65ms  max=40.45ms p(90)=13.6ms  p(95)=15.84ms
     http_reqs......................: 10000  1167.12652/s
     iteration_duration.............: avg=8.55ms  min=1.78ms med=7.79ms  max=40.58ms p(90)=13.77ms p(95)=15.98ms
     iterations.....................: 10000  1167.12652/s
     vus............................: 10     min=10       max=10 
     vus_max........................: 10     min=10       max=10
```

## 테스트 결과

- user 10, iteration 10000
- shard 6, replication 1(1인 이유는 insert 시간이 너무 오래 걸려서. 두 배 차이)
- 네트워크가 불안정 -> 3번 수행해서 제일 빠른것으로!

| index cnt\avg | avg     | p90     | p95     | min    | med    |
| :------------ | :------ | :------ | :------ | :----- | :----- |
| 10000건       | 6.14ms  | 9.85ms  | 11.65ms | 1.19ms | 5.61ms |
| 십만건        | 6.44ms  | 9.92ms  | 11.72ms | 1.32ms | 5.95ms |
| 백만건        | 6.38ms  | 10.26ms | 12.28ms | 1.18ms | 5.8ms  |
| 천만건        | 7.06ms  | 10.81ms | 12.76ms | 1.64ms | 6.48ms |
| 2천만건(500M) | 7.29ms  | 11.25ms | 13.14ms | 1.63ms | 6.73ms |
| 5천만건       | 7.64ms  | 12.42ms | 14.7ms  | 1.34ms | 6.93ms |
| 1억건(2.5G)   | 8.39ms  | 13.6ms  | 15.84ms | 1.67ms | 7.65ms |
| 1억건(rep 2)  | 7.99ms  | 12.38ms | 14.11ms | 1.58ms | 7.59ms |
| 10억건(26G)   | 10.42ms | 16.29ms | 18.73ms | 2ms    | 9.62ms |

## 결론

- 내부적으로 캐싱 등이 존재하는 것으로 보임. 첫 시도때 확연하게 느리고 그 이후 테스트에는 속도가 개선되며 일정한 수치를 보이고 있음
- 데이터 건수에 따라 시간차를 보이는 것은 확연하지만, 1G당 얼마가 느려진다~정도로 판단하긴 어려울 듯 단순 패턴만 파악하자면 GB 단위당 약 0.1ms 차이 정도로 추측은 할 수 있을듯 함
