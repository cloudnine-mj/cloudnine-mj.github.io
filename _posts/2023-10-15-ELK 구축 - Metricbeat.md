---
key: jekyll-text-theme
title: 'ELK 구축 - Metricbeat'
excerpt: ' Ubuntu 서버에 ELK 구축하기 😎'
tags: [Metricbeat, ELK]
---

# Metricbeat

## **1. 유저 생성 및 패스워드 설정**

```
root@ubuntu22:~# useradd -d /home/metricbeat -s /bin/bash -m metricbeat
root@ubuntu22:~# passwd metricbeat
```

## 2. 패키지 다운로드 및 압축 해제

* 패키지 다운로드

```
metricbeat@ubuntu22:~$ wget https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-7.11.0-linux-x86_64.tar.gz
```

* 패키지 압축 해제

```
metricbeat@ubuntu22:~$ tar -xvf metricbeat-7.11.0-linux-x86_64.tar.gz
```

## 3. 환경 설정

### metricbeat.yml 설정

```
metricbeat@ubuntu22:~/metricbeat-7.11.0-linux-x86_64$ vi metricbeat.yml
```

* 다음과 같이 작성

```
metricbeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false
setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression
output.logstash:
  hosts: ["127.0.0.1:5044"]
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - drop_fields:
      when:
        has_fields: ['system.filesystem']
      fields:
        - "system.filesystem.available"
        - "system.filesystem.files"
        - "system.filesystem.free_files"
        - "system.filesystem.mount_point"
        - "system.filesystem.used.bytes"
        - "system.filesystem.inode"
        - "system.filesystem.inode.pct"
        - "system.filesystem.inode.used"
      #ignore_missing: true
  - drop_fields:
      when:
        has_fields: ['system.memory']
      fields:
        - "system.memory.hugepages"
        - "system.memory.page_stats"
        - "system.memory.swap"
  - drop_fields:
      when:
        has_fields: ['system.cpu']
      fields:
        - "system.cpu.idle.pct"
        - "system.cpu.idle.ticks"
        - "system.cpu.iowait.pct"
        - "system.cpu.iowait.ticks"
        - "system.cpu.irq.norm.pct"
        - "system.cpu.irq.pct"
        - "system.cpu.irq.ticks"
        - "system.cpu.nice.norm.pct"
        - "system.cpu.nice.pct"
        - "system.cpu.nice.ticks"
        - "system.cpu.softirq.norm.pct"
        - "system.cpu.softirq.pct"
        - "system.cpu.softirq.ticks"
        - "system.cpu.steal.norm.pct"
        - "system.cpu.steal.pct"
        - "system.cpu.steal.ticks"
        - "system.cpu.system.pct"
        - "system.cpu.system.ticks"
        - "system.cpu.total.pct"
        - "system.cpu.user.pct"
        - "system.cpu.user.ticks"
  - drop_fields:
      when:
        has_fields: ['system.diskio']
      fields:
        - "system.diskio.io"
        - "system.diskio.iostat.queue"
        - "system.diskio.iostat.read"
        - "system.diskio.iostat.request"
        - "system.diskio.iostat.service_time"
        - "system.diskio.iostat.write"
        - "system.diskio.read.count"
        - "system.diskio.read.time"
        - "system.diskio.write.count"
        - "system.diskio.write.time"
        - "system.diskio.serial_number"
  - drop_fields:
      when:
        has_fields: ['system.load']
      fields:
        - "system.load.1"
        - "system.load.15"
        - "system.load.5"
        - "system.load.cores"
        - "system.load.norm.1"
        - "system.load.norm.15"
  - drop_fields:
      when:
        has_fields: ['system.network']
      fields:
        - "system.network.in.dropped"
        - "system.network.in.errors"
        - "system.network.in.packets"
        - "system.network.out.dropped"
        - "system.network.out.errors"
        - "system.network.out.packets"
# runtime/cgo: pthread_create failed: Operation not permitted 해결을 위한 설정 추가       
seccomp:
  default_action: allow
  syscalls:
  - action: allow
    names:
    - rseq
```

### system.yml 설정

```
metricbeat@ubuntu22:~/metricbeat-7.11.0-linux-x86_64$ vi system.yml
```

* 다음과 같이 작성

```
- module: system
  period: 10s
  metricsets:
    - cpu
- module: system
  period: 10s
  metricsets:
    - diskio
- module: system
  period: 10s
  metricsets:
    - filesystem
  filesystem.ignore_types: []
- module: system
  period: 10s
  metricsets:
    - load
- module: system
  period: 10s
  metricsets:
    - memory
- module: system
  period: 10s
  metricsets:
    - network
- module: system
  period: 10s
  metricsets:
    - uptime
```

## 4. Metricbeat 실행

```
metricbeat@ubuntu22:~/metricbeat-7.11.0-linux-x86_64$ ./metricbeat -e -c metricbeat.yml
```
