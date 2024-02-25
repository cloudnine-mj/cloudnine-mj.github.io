---
key: jekyll-text-theme
title: 'Ingest system enable 적용'
excerpt: 'system enable 설정하기 😎'
tags: [Ingest]
---



# Ingest system enable 적용

🎯수행 목적 : vm 재기동 시 system enable 되어 자동으로 (daemon) 기동될 수 있도록 처리



## 1. start.sh / stop.sh 작성

* 셸(shell) 파일은 logstash-8.1.3 에서 작성

### start.sh 작성

```
ingest@ubuntu22:~/logstash-8.1.3$ vi start.sh
```

* 다음과 같이 작성

```
/home/ingest/logstash-8.1.3/bin/logstash -f kafka.conf
```

### stop.sh 작성

```
ingest@ubuntu22:~/logstash-8.1.3$ vi stop.sh
```

* 다음과 같이 작성

```
ps -ef|grep logstash-8.1.3|grep -v grep| awk '{print "kill -9 "$2}' |sh -i
```

## 2. Ingest service 등록

- service 등록은 root에서 진행

```
root@ubuntu22:~# vi /etc/systemd/system/ingest.service
```

* 다음과 같이 작성

```
[Unit]
Description=ingest
After=network.target

[Service]
User=ingest
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/home/ingest/jdk-17.0.1/bin  
ExecStart=/home/ingest/logstash-8.1.3/bin/logstash -f /home/ingest/logstash-8.1.3/kafka.conf  
ExecStop=/home/ingest/logstash-8.1.3/stop  
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## 3. Ingest 서비스 실행

```
root@ubuntu22:/etc/systemd/system# systemctl daemon-reload 
root@ubuntu22:/etc/systemd/system# systemctl start ingest.service 
```

 

## 4. Ingest 서비스 상태 확인

```
root@ubuntu22:/etc/systemd/system# systemctl status ingest.service
```

## 5. Ingest system enable 적용

```
root@ubuntu22:/etc/systemd/system# systemctl enable ingest.service 
```