---
key: jekyll-text-theme
title: 'OpenSearch system enable 적용'
excerpt: ' Ubuntu 서버에 ELK 구축하기 😎'
tags: [OpenSearch]
---



# OpenSearch system enable 적용

🎯수행 목적 : vm 재기동 시 system enable 되어 자동으로 (daemon) 기동될 수 있도록 처리



## 1. start.sh / stop.sh 작성

* 셸(shell) 파일은 opensearch-2.6.0 에서 작성

### start.sh 작성

```
opensearch@ubuntu22:~/opensearch-2.6.0$ vi start.sh
```

* 다음과 같이 작성

```
/home/opensearch/opensearch-2.6.0/bin/opensearch
```

### stop.sh 작성

```
opensearch@ubuntu22:~/opensearch-2.6.0$ vi stop.sh
```

* 다음과 같이 작성

```
ps -ef|grep opensearch-2.6.0|grep -v grep| awk '{print "kill -9 "$2}' |sh -i
```

## 2. OpenSearch service 등록

- service 등록은 root에서 진행

```
root@ubuntu22:~# vi /etc/systemd/system/opensearch.service
```

* 다음과 같이 작성

```
[Unit]
Description=opensearch
After=network.target

[Service]
User=opensearch
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/home/opensearch/jdk-17.0.1/bin  
ExecStart=/home/opensearch/opensearch-2.6.0/bin/opensearch  
ExecStop=/home/opensearch/opensearch-2.6.0/stop  
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## 3. OpenSearch 서비스 실행

```
root@ubuntu22:/etc/systemd/system# systemctl daemon-reload 
root@ubuntu22:/etc/systemd/system# systemctl start opensearch.service 
```

 

## 4. OpenSearch 서비스 상태 확인

```
root@ubuntu22:/etc/systemd/system# systemctl status opensearch.service
```

## 5. OpenSearch system enable 적용

```
root@ubuntu22:/etc/systemd/system# systemctl enable opensearch.service 
```