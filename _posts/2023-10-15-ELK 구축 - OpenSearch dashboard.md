---
key: jekyll-text-theme
title: 'ELK 구축 - OpenSearch dashboard'
excerpt: ' Ubuntu 서버에 ELK 구축하기 😎'
tags: [OpenSearch dashboard, ELK]
---

# OpenSearch dashboard

## **1. ‘opensearch’ 유저에서 진행**

```
root@ubuntu22:~# su - opensearch
```



## **2. 패키지 다운로드 및 압축 해제**

```
opensearch@ubuntu22:~$ wget https://artifacts.opensearch.org/releases/bundle/opensearch-dashboards/2.6.0/opensearch-dashboards-2.6.0-linux-x64.tar.gz

opensearch@ubuntu22:~$ tar -xvf opensearch-dashboards-2.6.0-linux-x64.tar.gz
```



## **3. key 생성하기**

```
# Private key 생성 (root-key.pem)
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0/config$ openssl genrsa -out root-key.pem 2048
```

```
# root certificate 생성 (root-cert.pem)
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0/config$ openssl req -new -x509 -sha256 -key root-key.pem -out root-cert.pem -days 730 -subj "/C=KR/ST=Seoul/L=Seoul/O=Okestro/OU=DP/CN=opensearch/emailAddress=mj.kang3@naver.com"
```

```
# private key 생성 (admin-key.pem)
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0/config$ openssl genrsa -out admin-key-temp.pem 2048
```

```
# PKCS#8 포맷으로 변환
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0/config$ openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem
```

```
# csr certificate 생성
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0/config$ openssl req -new -key admin-key.pem -out admin.csr -subj "/C=KR/ST=Seoul/L=Seoul/O=Okestro/OU=DP/CN=opensearch/emailAddress=mj.kang3@naver.com"
```

```
# admin.pem 생성
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0/config$ openssl x509 -req -in admin.csr -CA root-cert.pem -CAkey root-key.pem -CAcreateserial -sha256 -out admin.pem -days 730
```



## **4. opensearch_dashboards.yml 설정**

### **[http]**

- opensearch_dashboards.yml 수정
    - ‘opensearch.hosts’에는 서버 ip 작성

```
server.host: "0.0.0.0"
opensearch.hosts: ["http://127.0.0.1:8200"]
opensearch.ssl.verificationMode: none
opensearch.username: "admin"
opensearch.password: "admin"
opensearch.requestHeadersWhitelist: [authorization, securitytenant]

server.ssl.enabled: false
#server.ssl.certificate: "/home/opensearch/opensearch-dashboards-2.6.0/config/root-cert.pem"
#server.ssl.key: "/home/opensearch/opensearch-dashboards-2.6.0/config/root-key.pem"

opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: [kibana_read_only]
opensearch_security.cookie.secure: false
opensearch_security.session.ttl: 60000
```

### **[https]**

- opensearch_dashboards.yml 수정
    - ‘opensearch.hosts’에는 서버 ip 작성

```
server.host: "0.0.0.0"
opensearch.hosts: ["https://127.0.0.1:8200"]
opensearch.ssl.verificationMode: none
opensearch.username: "admin"
opensearch.password: "admin"
opensearch.requestHeadersWhitelist: [authorization, securitytenant]

server.ssl.enabled: true
server.ssl.certificate: "/home/opensearch/opensearch-dashboards-2.6.0/config/root-cert.pem"
server.ssl.key: "/home/opensearch/opensearch-dashboards-2.6.0/config/root-key.pem"

opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: [kibana_read_only]
opensearch_security.cookie.secure: false
opensearch_security.session.ttl: 60000
```



## **5. OpenSearch 실행**

- OpenSearch Dashboard 실행을 위해서는 먼저 OpenSearch를 먼저 실행해야 함.

```
opensearch@ubuntu22:~/opensearch-2.6.0$ nohup bin/opensearch > /dev/null 2>&1 &
```

```
# OpenSearch가 정상적으로 실행되었는지 확인
opensearch@ubuntu22:~/opensearch-2.6.0$ netstat -tnlp
```



## **6. OpenSearch dashboard 실행**

```
# 백그라운드 모드에서 OpenSearch Dashboard 실행
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0$ nohup bin/opensearch-dashboards > /dev/null 2>&1 &
```

```
# OpenSearch Dashboard가 정상적으로 실행되었는지 확인
opensearch@ubuntu22:~/opensearch-dashboards-2.6.0$ netstat -tnlp
```



### **[실행 결과]**

- 접속 주소 : http://127.0.0.1:5601 (https는 https://127.0.0.1:5601 로 접속 가능)