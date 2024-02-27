---
key: jekyll-text-theme
title: 'Logstash system enable 적용'
excerpt: 'system enable 설정하기 😎'
tags: [Logstash]
---

# Logstash system enable 설정이란?

🎯수행 목적 : vm 재기동 시 system enable 되어 자동으로 (daemon) 기동될 수 있도록 처리

## 1. 서비스 등록

```
logstash@ubuntu22:~/logstash-8.1.3$ su - root
root@ubuntu22:~# vi /etc/systemd/system/logstash.service
```

* vi 편집기에 다음과 같이 작성

```
[Unit]
Description=Logstash

[Install]
WantedBy=multi-user.target

[Service]
User=logstash
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin:/home/logstash/jdk-17.0.1/bin
#Type=forking # 자식 프로세스 생성 후 실행이 필요할 경우 지정
ExecStart=/home/logstash/logstash-8.1.3/bin/logstash -f /home/logstash/logstash-8.1.3/beat_metric.conf
ExecStop=/home/logstash/logstash-8.1.3/stop
Restart=on-failure
```

## 2. 서비스 등록을 위한 명령어 실행

```
root@ubuntu22:~# systemctl daemon-reload
root@ubuntu22:~# systemctl start logstash.service
root@ubuntu22:~# systemctl status logstash.service
```


