---
key: jekyll-text-theme
title: 'ELK 구축 - Logstash'
excerpt: ' Ubuntu 서버에 ELK 구축하기 😎'
tags: [Logstash, ELK]
---

# Logstash

## **1.유저생성 및 패스워드 설정**

```
root@ubuntu22~# useradd -d /home/logstash -s /bin/bash -m logstash
root@ubuntu22~# passwd logstash
```

## **2. 패키지 다운로드 및 압축 해제**
* 패키지 다운로드

```
logstash@ubuntu22:~$ wget https://artifacts.elastic.co/downloads/logstash/logstash-8.1.3-linux-x86_64.tar.gz

logstash@ubuntu22:~$ wget https://downloads.mysql.com/archives/get/p/3/file/mysql-connector-java-5.1.49.tar.gz

logstash@ubuntu22:~$ wget https://download.java.net/java/GA/jdk17.0.1/2a2082e5a09d4267845be086888add4f/12/GPL/openjdk-17.0.1_linux-x64_bin.tar.gz
```

*  패키지 압축 해제

```
logstash@ubuntu22:~$ tar -xvf logstash-8.1.3-linux-x86_64.tar.gz
logstash@ubuntu22:~$ tar -xvf mysql-connector-java-5.1.49.tar.gz
logstash@ubuntu22:~$ tar -xvf openjdk-17.0.1_linux-x64_bin.tar.gz
```

## **3. 환경설정**
### bash_profile 설정

```
logstash@ubuntu22:~$ vi ~/.bash_profile
```

* vi 편집기에 다음과 같이 작성

```
JAVA_HOME=${HOME}/jdk-17.0.1
PATH=$PATH:${HOME}/jdk-17.0.1/bin:${HOME}/logstash-8.1.3/bin
set -o vi
```

## 4. logstash filter conf 작성
### beat_metric.conf 

* metric 데이터 수집을 위한 conf 작성

```
logstash@ubuntu22:~/logstash-8.1.3/config$ vi beat_metric.conf
```

* vi 편집기에 다음과 같이 작성

```
input {
  http {
    port => "5064"
    codec => json
  }
  beats {
    port => "5044"
    codec => json
  }
}

filter {
  mutate {
    add_field => { "[@ctime]" => "%{[@timestamp]}" }
    add_field => { "[@itime]" => "%{[@timestamp]}" }
  }

  if [fields][object_id] {
      mutate {
        copy => { "[fields][object_id]" => "[object_id]" }
      }
  }
  if [fields][identifier] {
      mutate {
        copy => { "[fields][identifier]" => "[identifier]" }
      }
  }


  if [agent][type] == "metricbeat" {
    ##########################################################
    # system.core.*.pct
    ##########################################################
    if [system][core][total][pct] {
      ruby {
        code => "event.set('[system][core][total][pct]', (event.get('[system][core][total][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][user][pct] {
      ruby {
        code => "event.set('[system][core][user][pct]', (event.get('[system][core][user][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][system][pct] {
      ruby {
        code => "event.set('[system][core][system][pct]', (event.get('[system][core][system][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][nice][pct] {
      ruby {
        code => "event.set('[system][core][nice][pct]', (event.get('[system][core][nice][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][idle][pct] {
      ruby {
        code => "event.set('[system][core][idle][pct]', (event.get('[system][core][idle][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][iowait][pct] {
      ruby {
        code => "event.set('[system][core][iowait][pct]', (event.get('[system][core][iowait][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][irq][pct] {
      ruby {
        code => "event.set('[system][core][irq][pct]', (event.get('[system][core][irq][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][softirq][pct] {
      ruby {
        code => "event.set('[system][core][softirq][pct]', (event.get('[system][core][softirq][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][steal][pct] {
      ruby {
        code => "event.set('[system][core][steal][pct]', (event.get('[system][core][steal][pct]') * 100.0).round(2))"
      }
    }


    ##########################################################
    # system.cpu.*.pct
    ##########################################################
    if [system][cpu][user][pct] {
      ruby {
        code => "event.set('[system][cpu][user][pct]', (event.get('[system][cpu][user][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][system][pct] {
      ruby {
        code => "event.set('[system][cpu][system][pct]', (event.get('[system][cpu][system][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][nice][pct] {
      ruby {
        code => "event.set('[system][cpu][nice][pct]', (event.get('[system][cpu][nice][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][idle][pct] {
      ruby {
        code => "event.set('[system][cpu][idle][pct]', (event.get('[system][cpu][idle][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][iowait][pct] {
      ruby {
        code => "event.set('[system][cpu][iowait][pct]', (event.get('[system][cpu][iowait][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][irq][pct] {
      ruby {
        code => "event.set('[system][cpu][irq][pct]', (event.get('[system][cpu][irq][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][softirq][pct] {
      ruby {
        code => "event.set('[system][cpu][softirq][pct]', (event.get('[system][cpu][softirq][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][steal][pct] {
      ruby {
        code => "event.set('[system][cpu][steal][pct]', (event.get('[system][cpu][steal][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][total][pct] {
      ruby {
        code => "event.set('[system][cpu][total][pct]', (event.get('[system][cpu][total][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][user][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][user][norm][pct]', (event.get('[system][cpu][user][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][system][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][system][norm][pct]', (event.get('[system][cpu][system][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][nice][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][nice][norm][pct]', (event.get('[system][cpu][nice][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][idle][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][idle][norm][pct]', (event.get('[system][cpu][idle][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][iowait][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][iowait][norm][pct]', (event.get('[system][cpu][iowait][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][irq][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][irq][norm][pct]', (event.get('[system][cpu][irq][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][softirq][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][softirq][norm][pct]', (event.get('[system][cpu][softirq][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][steal][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][steal][norm][pct]', (event.get('[system][cpu][steal][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][total][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][total][norm][pct]', (event.get('[system][cpu][total][norm][pct]') * 100.0).round(2))"
      }
    }


    ##########################################################
    # system.entropy.*.pct
    ##########################################################
    if [system][entropy][pct] {
      ruby {
        code => "event.set('[system][entropy][pct]', (event.get('[system][entropy][pct]') * 100.0).round(2))"
      }
    }


    ##########################################################
    # system.filesystem.*.pct
    ##########################################################
    if [system][filesystem][used][pct] {
      ruby {
        code => "event.set('[system][filesystem][used][pct]', (event.get('[system][filesystem][used][pct]') * 100.0).round(2))"
      }
    }

    ##########################################################
    # system.memory.*.pct
    ##########################################################
    if [system][memory][used][pct] {
      ruby {
        code => "event.set('[system][memory][used][pct]', (event.get('[system][memory][used][pct]') * 100.0).round(2))"
      }
    }

    if [system][memory][actual][used][pct] {
      ruby {
        code => "event.set('[system][memory][actual][used][pct]', (event.get('[system][memory][actual][used][pct]') * 100.0).round(2))"
      }
    }

    if [system][memory][swap][used][pct] {
      ruby {
        code => "event.set('[system][memory][swap][used][pct]', (event.get('[system][memory][swap][used][pct]') * 100.0).round(2))"
      }
    }
  } # identifier //2023.10.04 add
  else  if "metric_collector" in [agent][type] {
    if "openstack" in [service][type] {
      mutate {
        add_field => {
          "identifier" => ""
        }
      }
      if "vm" in [service][adapter] {
        mutate {
          copy => { "[basic][vm][vm_id]" => "[identifier]" }
        }
      }
      if "loadbalancer" in [service][adapter] {
        if [loadbalancer][listener] {
          mutate {
            add_field => {
              "identifier_message" => "%{[loadbalancer][listener]}"
            }
          }

          grok { match => { "identifier_message" => '.+?id\W+(?<id>[^"]+)' } }

          mutate {
            copy => { "[id]" => "[identifier]" }
            remove_field => [ "id", "identifier_message" ]
          }

        }
        else {
          mutate {
            copy => { "[loadbalancer][id]" => "[identifier]" }
          }
        }
      }
      else if "baremetal" in [service][adapter] {
        mutate {
          copy => { "[baremetal][id]" => "[identifier]" }
        }
      }
      else if "filestorage" in [service][adapter] {
        mutate {
          copy => { "[filestorage][volume][id]" => "[identifier]" }
        }
      }
      else if "network" in [service][adapter] {
        mutate {
          copy => { "[network][id]" => "[identifier]" }
        }
      }
      else if "image" in [service][adapter] {
        mutate {
          copy => { "[image][id]" => "[identifier]" }
        }
      }
    }
    else if  "netapp" in [service][type] {
      mutate {
        add_field => {
          "identifier" => ""
        }
      }
      if "filestorage" in [service][adapter] {
        mutate {
          copy => { "[filestorage][volume][id]" => "[identifier]" }
        }
      }
    }
  }
}


output {
  stdout {
    codec => rubydebug
  }
  file  {
    path => "/home/logstash/logstash-8.1.3_metric/logstash_rx_message.log"
    codec => rubydebug
  }
  kafka {
    bootstrap_servers => "192.168.2.120:9092,192.168.2.121:9092"
    topic_id => ["tuba-metric-topic"]
    codec => "json"
  }
}
```
### beat_meta.conf
* meta 데이터 수집을 위한 conf 작성

```
input {
  http {
    port => "5054"
    codec => json
  }
}

filter {
  mutate {
    add_field => { "[@ctime]" => "%{[@timestamp]}" }
    add_field => { "[@itime]" => "%{[@timestamp]}" }
  }

  if [fields][object_id] {
      mutate {
        copy => { "[fields][object_id]" => "[object_id]" }
      }
  }
  if [fields][identifier] {
      mutate {
        copy => { "[fields][identifier]" => "[identifier]" }
      }
  }

  if [agent][type] == "metricbeat" {
    ##########################################################
    # system.core.*.pct
    ##########################################################
    if [system][core][total][pct] {
      ruby {
        code => "event.set('[system][core][total][pct]', (event.get('[system][core][total][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][user][pct] {
      ruby {
        code => "event.set('[system][core][user][pct]', (event.get('[system][core][user][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][system][pct] {
      ruby {
        code => "event.set('[system][core][system][pct]', (event.get('[system][core][system][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][nice][pct] {
      ruby {
        code => "event.set('[system][core][nice][pct]', (event.get('[system][core][nice][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][idle][pct] {
      ruby {
        code => "event.set('[system][core][idle][pct]', (event.get('[system][core][idle][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][iowait][pct] {
      ruby {
        code => "event.set('[system][core][iowait][pct]', (event.get('[system][core][iowait][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][irq][pct] {
      ruby {
        code => "event.set('[system][core][irq][pct]', (event.get('[system][core][irq][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][softirq][pct] {
      ruby {
        code => "event.set('[system][core][softirq][pct]', (event.get('[system][core][softirq][pct]') * 100.0).round(2))"
      }
    }

    if [system][core][steal][pct] {
      ruby {
        code => "event.set('[system][core][steal][pct]', (event.get('[system][core][steal][pct]') * 100.0).round(2))"
      }
    }


    ##########################################################
    # system.cpu.*.pct
    ##########################################################
    if [system][cpu][user][pct] {
      ruby {
        code => "event.set('[system][cpu][user][pct]', (event.get('[system][cpu][user][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][system][pct] {
      ruby {
        code => "event.set('[system][cpu][system][pct]', (event.get('[system][cpu][system][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][nice][pct] {
      ruby {
        code => "event.set('[system][cpu][nice][pct]', (event.get('[system][cpu][nice][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][idle][pct] {
      ruby {
        code => "event.set('[system][cpu][idle][pct]', (event.get('[system][cpu][idle][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][iowait][pct] {
      ruby {
        code => "event.set('[system][cpu][iowait][pct]', (event.get('[system][cpu][iowait][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][irq][pct] {
      ruby {
        code => "event.set('[system][cpu][irq][pct]', (event.get('[system][cpu][irq][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][softirq][pct] {
      ruby {
        code => "event.set('[system][cpu][softirq][pct]', (event.get('[system][cpu][softirq][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][steal][pct] {
      ruby {
        code => "event.set('[system][cpu][steal][pct]', (event.get('[system][cpu][steal][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][total][pct] {
      ruby {
        code => "event.set('[system][cpu][total][pct]', (event.get('[system][cpu][total][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][user][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][user][norm][pct]', (event.get('[system][cpu][user][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][system][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][system][norm][pct]', (event.get('[system][cpu][system][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][nice][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][nice][norm][pct]', (event.get('[system][cpu][nice][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][idle][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][idle][norm][pct]', (event.get('[system][cpu][idle][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][iowait][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][iowait][norm][pct]', (event.get('[system][cpu][iowait][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][irq][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][irq][norm][pct]', (event.get('[system][cpu][irq][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][softirq][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][softirq][norm][pct]', (event.get('[system][cpu][softirq][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][steal][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][steal][norm][pct]', (event.get('[system][cpu][steal][norm][pct]') * 100.0).round(2))"
      }
    }

    if [system][cpu][total][norm][pct] {
      ruby {
        code => "event.set('[system][cpu][total][norm][pct]', (event.get('[system][cpu][total][norm][pct]') * 100.0).round(2))"
      }
    }


    ##########################################################
    # system.entropy.*.pct
    ##########################################################
    if [system][entropy][pct] {
      ruby {
        code => "event.set('[system][entropy][pct]', (event.get('[system][entropy][pct]') * 100.0).round(2))"
      }
    }


    ##########################################################
    # system.filesystem.*.pct
    ##########################################################
    if [system][filesystem][used][pct] {
      ruby {
        code => "event.set('[system][filesystem][used][pct]', (event.get('[system][filesystem][used][pct]') * 100.0).round(2))"
      }
    }

    ##########################################################
    # system.memory.*.pct
    ##########################################################
    if [system][memory][used][pct] {
      ruby {
        code => "event.set('[system][memory][used][pct]', (event.get('[system][memory][used][pct]') * 100.0).round(2))"
      }
    }

    if [system][memory][actual][used][pct] {
      ruby {
        code => "event.set('[system][memory][actual][used][pct]', (event.get('[system][memory][actual][used][pct]') * 100.0).round(2))"
      }
    }

    if [system][memory][swap][used][pct] {
      ruby {
        code => "event.set('[system][memory][swap][used][pct]', (event.get('[system][memory][swap][used][pct]') * 100.0).round(2))"
      }
    }
  }
}


output {
  stdout {
    codec => rubydebug
  }
  kafka {
    bootstrap_servers => "127.0.0.1:9092"
    topic_id => ["tuba-meta-topic"]
    codec => "json"
  }
}
```

## **5. Logstash input 플러그인 설치**

```
logstash@ubuntu22:~/logstash-8.1.3$ logstash-plugin install logstash-output-opensearch
```

## **6. Logstash 실행**

```
logstash@ubuntu22:~/logstash-8.1.3$ nohup bin/logstash -f /home/logstash/logstash-8.1.3/config/beat_metric.conf 2>&1 1>/dev/null &
```