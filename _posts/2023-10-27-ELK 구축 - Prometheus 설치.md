---
key: jekyll-text-theme
title: 'Prometheus 설치'
excerpt: 'Prometheus installation 😎'
tags: [Prometheus]
---

# Prometheus



👉 Prometheus Document : [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)




## 1. 유저 생성 및 패스워드 설정

```
root@ubuntu22:~# useradd -d /home/prometheus -s /bin/bash -m prometheus  

root@ubuntu22:~# passwd prometheus
```

## 2. 다운로드 및 압축 해제

```
prometheus@ubuntu22:~$ wget https://github.com/prometheus/prometheus/releases/download/v2.48.0-rc.0/prometheus-2.48.0-rc.0.linux-amd64.tar.gz  

prometheus@ubuntu22:~$ tar xzvf prometheus-2.48.0-rc.0.linux-amd64.tar.gz
```

## 3. Prometheus 실행

* 포그라운드 실행

```
prometheus@ubuntu22:~/prometheus-2.48.0-rc.0.linux-amd64$ ./prometheus  
```

* 백그라운드 실행 (nohup)

```
prometheus@ubuntu22:~/prometheus-2.48.0-rc.0.linux-amd64$ nohup ./prometheus > /dev/null 2>&1 &
```

**[참고]**

```
# 포트를 바꿔서 실행하고 싶을 떄
# ./prometheus --web.listen-address=0.0.0.0:19090
# (default port가 9090이므로 port 변경 시에는 prometheus.yml에서
# static_config에 tartget에 포트를 변경을 해줘야 한다.)

이때 nohup으로 하는 백그라운드 실행은

prometheus@ubuntu22:~/prometheus-2.48.0-rc.0.linux-amd64$ nohup ./prometheus --web.listen-address=0.0.0.0:19090 > /dev/null 2>&1 &
```

## 4. Prometheus 서비스 실행 확인

* 9090 포트 열렸는지 확인

```
prometheus@ubuntu22:~/prometheus-2.48.0-rc.0.linux-amd64$ netstat -tnlp
```

* 접속 

```
<서버 IP>:9090
```
