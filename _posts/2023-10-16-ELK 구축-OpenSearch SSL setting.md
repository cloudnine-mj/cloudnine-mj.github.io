---
key: jekyll-text-theme
title: 'OpenSearch SSL setting'
excerpt: ' OpenSearch research 😎'
tags: [OpenSearch, ELK]
---

👉 Docs : [https://opensearch.org/docs/1.0/security-plugin/configuration/tls/](https://opensearch.org/docs/1.0/security-plugin/configuration/tls/)

# OpenSearch SSL setting

🎯 OpenSearch 구축 시 HTTPS 적용할 때 많은 어려움을 겪었고 생각보다 많이 힘들었다.... 그래서 관련 부분 리서치를 해봤다.

## Transport layer

- Transport layer는 Cluster 설정 시의 ssl 설정이다. 

### plugins.security.ssl.transport.pemkey_filepath

- Server의 private key를 넣어줘야 하는 프로퍼티.
- 문서엔 pcks#8 기준인데 굳이 그럴 필요는 없음.

### plugins.security.ssl.transport.pemcert_filepath

- 이건 Server의 인증서를 넣어줘야 한다.
- X509 규격을 반드시 사용해야 한다고 한다.

### plugins.security.ssl.transport.pemtrustedcas_filepath

- ‘인증 기관’의 인증서를 넣어줘야 한다.

### plugins.security.ssl.transport.enforce_hostname_verification

- 인증서엔 Common name이 있는데, 이 값은 원래 hostname이어야 한다.
- 그런데 인증서마다 그걸 다 넣어주기도 귀찮고 하니, false로 하는 것이 편하다.

## 결론

- Transport 쪽 프로퍼티는 이렇게 되겠다

```
plugins.security.ssl.transport.enforce_hostname_verification: false
# 서버 인증서. server.cert
plugins.security.ssl.transport.pemcert_filepath: server.cert
# 서버 private key. . server.key
plugins.security.ssl.transport.pemkey_filepath: server_pkcs.key
# 인증 기관 인증서. ca.cert
plugins.security.ssl.transport.pemtrustedcas_filepath: ca.cert
```

## HTTPS layer

- HTTPS layer는 api server, 즉 우리가 curl 등을 통해 접속하는 웹서버의 https 규격이다.

### plugins.security.ssl.http.enabled: true

- http ssl 기능을 켜기 위한 프로퍼티.

### plugins.security.ssl.http.pemcert_filepath

- server의 인증서

### plugins.security.ssl.http.pemkey_filepath

- Server의 key

### plugins.security.ssl.http.pemtrustedcas_filepath

- 인증 기관의 인증서

### 결론

- Https쪽 프로퍼티는 이렇게.

```
plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: server.cert
plugins.security.ssl.http.pemkey_filepath: server.key
plugins.security.ssl.http.pemtrustedcas_filepath: ca.cert
```

## dn certificate 설정

### plugins.security.nodes_dn

- 각 cluster 간 통신을 위한 식별 정보를 적어줘야 한다
- common name은 정규 표현식을 사용할 수 있다는 점이 특이점.

### plugins.security.authcz.admin_dn

- 이쪽은 admin 작업을 위한 식별 정보를 따로 관리한다. CN name으로 구별할 수 있을 것이다.
- admin이 여러 명이면 안 되니, 식별정보는 정규 표현식 없이 고정하는 것을 추천.

# opensearch 설정

- key 생성 부분 

```
# 인증 기관 key, cert
openssl genrsa -out ca.key 2048
openssl req -new -x509 -key ca.key -out ca.cert -subj "/C=KR/ST=Seoul/O=O*******/CN=opensearch"

# server의 key, cert, csr
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/C=KR/ST=Seoul/O=O*******/CN=opensearch"
openssl x509 -req -in server.csr -CA ca.cert -CAkey ca.key -CAcreateserial -out server.cert
```

- opensearch.yml

```
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.pemcert_filepath: server.cert
plugins.security.ssl.transport.pemkey_filepath: server.key
plugins.security.ssl.transport.pemtrustedcas_filepath: ca.cert

plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: server.cert
plugins.security.ssl.http.pemkey_filepath: server.key
plugins.security.ssl.http.pemtrustedcas_filepath: ca.cert

plugins.security.allow_unsafe_democertificates: true
plugins.security.allow_default_init_securityindex: true

# 파일을 만들 때 설정해준 부분들을 지정해준다
plugins.security.authcz.admin_dn:
  - CN=opensearch,O=O*******,ST=Seoul,C=KR
plugins.security.nodes_dn:
  - CN=opensearch,O=O*******,ST=Seoul,C=KR

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".plugins-ml-model-group", ".plugins-ml-model", ".plugins-ml-task", ".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opensearch-notifications-*", ".opensearch-notebooks", ".opensearch-observability", ".ql-datasources", ".opendistro-asynchronous-search-response*", ".replication-metadata-store", ".opensearch-knn-models"]
node.max_local_storage_nodes: 3
```
