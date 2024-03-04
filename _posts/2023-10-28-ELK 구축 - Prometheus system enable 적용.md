---
key: jekyll-text-theme
title: 'Prometheus system enable 적용'
excerpt: 'system enable 설정하기 😎'
tags: [Prometheus]
---

# Prometheus system enable 설정

## 1. Prometheus서비스 등록

* root에서 진행

```
root@ubuntu22:~# cd /etc/systemd/system/prometheus.service
```

* vi 편집기에 다음과 같이 작성

```
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=root
Restart=on-failure

#Change this line if you download ther
#Prometheus on different path user
ExecStart=/home/prometheus2/prometheus-2.48.0-rc.0.linux-amd64/prometheus \
  --config.file=/home/prometheus2/prometheus-2.48.0-rc.0.linux-amd64/prometheus.yml \
  --storage.tsdb.path=/home/prometheus2/prometheus-2.48.0-rc.0.linux-amd64 \
  --web.console.templates=/home/prometheus2/prometheus-2.48.0-rc.0.linux-amd64/consoles \
  --web.console.libraries=/home/prometheus2/prometheus-2.48.0-rc.0.linux-amd64/console_libraries \
  --web.listen-address=0.0.0.0:19090 \
  --web.enable-admin-api

[Install]
WantedBy=multi-user.target
```

## 2. Prometheus 서비스 실행

```
root@ubuntu22:/etc/systemd/system# systemctl daemon-reload
root@ubuntu22:/etc/systemd/system# systemctl start prometheus.service 
```

## 3. Prometheus 서비스 상태 확인

```
root@ubuntu22:/etc/systemd/system# systemctl status prometheus.service
```

## 4. Prometheus 서비스 등록

```
root@ubuntu22:/etc/systemd/system# systemctl enable prometheus.service
```