---
key: jekyll-text-theme
title: 'PostgreSQL system enable 적용'
excerpt: ' system enable 설정하기 😎'
tags: [PostgreSQL]
---

# PostgreSQL system enable 설정이란?

🎯수행 목적: vm 재기동 시 system enable 되어 자동 기동 될 수 있도록 처리



## 1. Postgresql service 등록

* service 등록은 root에서 진행

```
vi /etc/systemd/system/postgres.service
```

* vi에 다음과 같이 작성

```
[Unit]
Description=postgresql
After=network.target

[Service]
Type=forking
User=postgres
Environment=PGDATA=/home/postgres/as15/data
Environment=PATH=/home/postgres/postgresql/bin
ExecStart=/home/postgres/postgresql/bin/pg_ctl start -D "${PGDATA}" -s -w -t 300
ExecStop=/home/postgres/postgresql/bin/pg_ctl stop -D "${PGDATA}" -s -m
#ExecReload=/home/postgres/postgresql/bin/pg_ctl reload -D "${PGDATA}" -s
#Restart=on-failure

[Install]
WantedBy=multi-user.target                                       
```

 

## 2. postgresql 서비스 실행

```
root@ubuntu22:/etc/systemd/system# systemctl daemon-reload
root@ubuntu22:/etc/systemd/system# systemctl start postgres.service 
```

 
## 3. Postgresql 서비스 상태 확인

```
root@ubuntu22:/etc/systemd/system# systemctl status postgres.service
```



## 4. Postgresql system enable 적용

```
root@ubuntu22:/etc/systemd/system# systemctl enable postgres.service 
```



**[참고]**

```
# 서비스 시작  
start postgres.service  

# 서비스 중단  
stop postgres.service
```