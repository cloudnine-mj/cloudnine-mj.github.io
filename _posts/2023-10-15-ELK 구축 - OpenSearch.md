---
key: jekyll-text-theme
title: 'ELK 구축 - OpenSearch'
excerpt: ' Ubuntu 서버에 ELK 구축하기 😎'
tags: [OpenSearch, ELK]
---

# OpenSearch

## **1. 유저 생성 및 패스워드 설정**

```
root@ubuntu22:~# useradd -d /home/opensearch -s /bin/bash -m opensearch
root@ubuntu22:~# passwd opensearch
```



## **2. ‘opensearch’ 유저로 변경 (root → opensearch)**

```
root@ubuntu22:~# su - opensearch
opensearch@ubuntu22:~$
```



## **3. 패키지 다운로드 및 압축 해제**

```
opensearch@ubuntu22:~$ wget <https://download.java.net/java/GA/jdk17.0.1/2a2082e5a09d4267845be086888add4f/12/GPL/openjdk-17.0.1_linux-x64_bin.tar.gz>

opensearch@ubuntu22:~$ tar -xvf opensearch-2.6.0-linux-x64.tar.gz
```



## **4. 환경 설정**

### **1) bash_profile 설정**

```
opensearch@ubuntu22:~$ vi ~/.bash_profile
```

- vi 편집기에 다음과 같이 입력

```
JAVA_HOME=${HOME}/jdk-17.0.1
PATH=$PATH:${HOME}/jdk-17.0.1/bin
export PATH=${HOME}/opensearch-2.6.0/bin:${PATH}
set -o vi
```



### **2) /etc/sysctl.conf 설정**

```
root@ubuntu22:~# vi /etc/sysctl.conf
```

- vi 편집기에 다음과 같이 입력

```
vm.max_map_count=262144
```

- 아래 명령어 실행

```
root@ubuntu22:~# sudo sysctl -p
root@ubuntu22:~# sudo swapoff -a
```



### **3) /etc/security/limits.conf 설정**

```
root@ubuntu22:~# vi /etc/security/limits.conf
```

- vi 편집기에 다음과 같이 입력

```
opensearch           -       nofile          65535
```



## **5. key 생성**

- Opensearch 유저 변경

```
root@ubuntu22:~# su - opensearch
```

- key 생성하기

```
opensearch@ubuntu22:~/opensearch-2.6.0$ mkdir -p key
opensearch@ubuntu22:~/opensearch-2.6.0$ cd key
```

```
# Private key 생성 (root-key.pem)
opensearch@ubuntu22:~/opensearch-2.6.0/key$ openssl genrsa -out root-key.pem 2048

# root certificate 생성 (root-cert.pem)
opensearch@ubuntu22:~/opensearch-2.6.0/key$ openssl req -new -x509 -sha256 -key root-key.pem -out root-cert.pem -days 730 -subj "/C=KR/ST=Seoul/L=Seoul/O=Okestro/OU=DP/CN=opensearch/emailAddress=mj.kang3@naver.com"

# private key 생성 (admin-key.pem)
opensearch@ubuntu22:~/opensearch-2.6.0/key$ openssl genrsa -out admin-key-temp.pem 2048

# PKCS#8 포맷으로 변환
opensearch@ubuntu22:~/opensearch-2.6.0/key$ openssl pkcs8 -inform PEM -outform PEM -in admin-key-temp.pem -topk8 -nocrypt -v1 PBE-SHA1-3DES -out admin-key.pem

# csr certificate 생성
opensearch@ubuntu22:~/opensearch-2.6.0/key$ openssl req -new -key admin-key.pem -out admin.csr -subj "/C=KR/ST=Seoul/L=Seoul/O=Okestro/OU=DP/CN=opensearch/emailAddress=mj.kang3@naver.com"

# admin.pem 생성
opensearch@ubuntu22:~/opensearch-2.6.0/key$ openssl x509 -req -in admin.csr -CA root-cert.pem -CAkey root-key.pem -CAcreateserial -sha256 -out admin.pem -days 730
```

- OpenSearch의 config/ 디렉터리로 복사

```
opensearch@ubuntu22:~/opensearch-2.6.0/key$ cd ..
opensearch@ubuntu22:~/opensearch-2.6.0$ cp key/* config/
```



## **6. opensearch.yml 파일 설정하기**

### **[http]**

```
opensearch@mj-test-4:~/opensearch-2.6.0$ vi config/opensearch.yml
```

- 다음과 같이 작성

```
http.port: 8200
cluster.name: test-cluster
node.name: node-1
network.host: 0.0.0.0

discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]

node.roles: ["cluster_manager","data"]

cluster.max_shards_per_node: 10000

# 코드 추가
plugins.security.disabled: true

# plugins.security.ssl.transport.enforce_hostname_verification: false
# plugins.security.ssl.transport.pemcert_filepath: admin.pem
# plugins.security.ssl.transport.pemkey_filepath: admin-key.pem
# plugins.security.ssl.transport.pemtrustedcas_filepath: root-cert.pem
# plugins.security.ssl.http.enabled: true
# plugins.security.ssl.http.pemcert_filepath: admin.pem
# plugins.security.ssl.http.pemkey_filepath: admin-key.pem
# plugins.security.ssl.http.pemtrustedcas_filepath: root-cert.pem

# plugins.security.allow_unsafe_democertificates: true
# plugins.security.allow_default_init_securityindex: true

# 파일을 만들 때 설정해준 부분들을 지정해준다

# plugins.security.authcz.admin_dn: EMAILADDRESS=mj.kang3@naver.com,CN=opensearch,OU=DP,O=Okestro,L=Seoul,ST=Seoul,C=KR
# plugins.security.nodes_dn:EMAILADDRESS=mj.kang3@naver.com,CN=opensearch,OU=DP,O=Okestro,L=Seoul,ST=Seoul,C=KR

# plugins.security.audit.type: internal_opensearch
# plugins.security.enable_snapshot_restore_privilege: true
# plugins.security.check_snapshot_restore_write_privileges: true
# plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
# plugins.security.system_indices.enabled: true
# plugins.security.system_indices.indices: [".plugins-ml-model-group", ".plugins-ml-model", ".plugins-ml-task", ".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opensearch-notifications-*", ".opensearch-notebooks", ".opensearch-observability", ".ql-datasources", ".opendistro-asynchronous-search-response*", ".replication-metadata-store", ".opensearch-knn-models"]

node.max_local_storage_nodes: 3

path.data: /home/opensearch/opensearch-2.6.0/path_data/data
path.logs: /home/opensearch/opensearch-2.6.0/path_data/logs
```



### **[https]**

```
http.port: 8200
cluster.name: test-cluster
node.name: node-1
network.host: 0.0.0.0

discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]

node.roles: ["cluster_manager", "data"]

cluster.max_shards_per_node: 10000

plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.pemcert_filepath: admin.pem
plugins.security.ssl.transport.pemkey_filepath: admin-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: root-cert.pem

plugins.security.ssl.http.enabled: true
plugins.security.ssl.http.pemcert_filepath: admin.pem
plugins.security.ssl.http.pemkey_filepath: admin-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: root-cert.pem

plugins.security.allow_unsafe_democertificates: true
plugins.security.allow_default_init_securityindex: true

# 파일을 만들 때 설정해준 부분들을 지정해준다

plugins.security.authcz.admin_dn:
  - EMAILADDRESS=mj.kang3@naver.com,CN=opensearch,OU=DP,O=Okestro,L=Seoul,ST=Seoul,C=KR
plugins.security.nodes_dn:
  - EMAILADDRESS=mj.kang3@naver.com,CN=opensearch,OU=DP,O=Okestro,L=Seoul,ST=Seoul,C=KR

plugins.security.audit.type: internal_opensearch
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".plugins-ml-model-group", ".plugins-ml-model", ".plugins-ml-task", ".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opensearch-notifications-*", ".opensearch-notebooks", ".opensearch-observability", ".ql-datasources", ".opendistro-asynchronous-search-response*", ".replication-metadata-store", ".opensearch-knn-models"]

node.max_local_storage_nodes: 3

path.data: /home/opensearch/opensearch-2.6.0/path_data/data
path.logs: /home/opensearch/opensearch-2.6.0/path_data/logs
```



## **7. shell 파일 작성**

### **1) start.sh**

```
opensearch@ubuntu22:~/opensearch-2.6.0$ vi start.sh
```

- start.sh에 다음과 같이 작성

```bash
#!/bin/bash
opensearch -d -p os.pid
```

- ‘chmod’ 로 권한 변경

```
opensearch@ubuntu22:~/opensearch-2.6.0$ chmod +x start.sh
```



### **2) stop.sh**

```
opensearch@ubuntu22:~/opensearch-2.6.0$ vi stop.sh
```

- [stop.sh](http://stop.sh) 에 다음과 같이 작성

```bash
#!/bin/bash
kill `cat os.pid`
```

- ‘chmod’ 로 권한 변경

```
opensearch@ubuntu22:~/opensearch-2.6.0$ chmod +x stop.sh
```



## **8. 구동하기**

- 포그라운드 모드에서 정상적으로 실행이 되는 지 확인한 후, 백그라운드 모드로 진행

```
# 포그라운드에서 실행
# 'bin/opensearch'로 실행 시 log 확인 가능
opensearch@ubuntu22:~/opensearch-2.6.0$ bin/opensearch
# 백그라운드에서 실행
opensearch@ubuntu22:~/opensearch-2.6.0$ nohup bin/opensearch > /dev/null 2>&1 &
# port 켜졌는지 확인
opensearch@ubuntu22:~/opensearch-2.6.0$ netstat -tnlp
```



**[참고]**

```
# 포그라운드에서 정지
opensearch@ubuntu22:~/opensearch-2.6.0$ kill -9 <PID/Program name 숫자>
```

```
# 백그라운드에서 정지
opensearch@ubuntu22:~/opensearch-2.6.0$ ./stop.sh
```

```
# port 꺼졌는지 확인
opensearch@ubuntu22:~/opensearch-2.6.0$ netstat -tnlp
```