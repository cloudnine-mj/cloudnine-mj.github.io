---
key: jekyll-text-theme
title: 'ELK 구축 - Ingest'
excerpt: ' Ubuntu 서버에 ELK 구축하기 😎'
tags: [Ingest, ELK]
---

# Ingest

## **1. 유저 생성 및 패스워드 설정**

```
root@ubuntu22:~# useradd -d /home/ingest -s /bin/bash -m ingest
root@ubuntu22:~# passwd ingest
```

## 2. 패키지 다운로드 및 압축 해제
* 패키지 다운로드

```
ingest@ubuntu22:~$ wget https://artifacts.elastic.co/downloads/logstash/logstash-8.1.3-linux-x86_64.tar.gz

ingest@ubuntu22:~$ wget https://downloads.mysql.com/archives/get/p/3/file/mysql-connector-java-5.1.49.tar.gz

ingest@ubuntu22:~$ wget https://download.java.net/java/GA/jdk17.0.1/2a2082e5a09d4267845be086888add4f/12/GPL/openjdk-17.0.1_linux-x64_bin.tar.gz
```

* 패키지 압축해제

```
ingest@ubuntu22:~$ tar -xvf logstash-8.1.3-linux-x86_64.tar.gz
ingest@ubuntu22:~$ tar -xvf mysql-connector-java-5.1.49.tar.gz
ingest@ubuntu22:~$ tar -xvf openjdk-17.0.1_linux-x64_bin.tar.gz
```

## 3. 환경설정
### bash_profile 설정

```
ingest@ubuntu22:~$ vi ~/.bash_profile
```

* vi 편집기에 다음과 같이 작성

```
export JAVA_HOME=/usr/local/src/jdk-17.0.1
export PATH=$PATH:/usr/local/src/jdk-17.0.1/bin:/usr/local/src/logstash-8.1.3/bin
export CLASSPATH=.:$JAVA_HOME/lib/mysql-connector-java-5.1.49-bin.jar:${PWD}:${CLASSPATH}
set -o vi
```

## 4. Kafka.conf 작성
* Ingest가 인덱싱을 하기 위한 kafka.conf 작성

```
ingest@ubuntu22:~/logstash-8.1.3/config$ vi kafka.conf
```

* 다음과 같이 작성

```
input {
  kafka {
    bootstrap_servers => "127.0.0.1:9092"
#    topics => ["tuba-topic","tuba-metric-topic"]  2023.09.15 jslee 
    topics => ["delta-topic"]
    codec => "json"
  }
}

filter {
  if "metricbeat" in [agent][type] {
    if "socket_summary" in [metricset][name] {
      drop { }
    }
    if "process" in [metricset][name] {
      drop { }
    }
#    if "fsstat" in [metricset][name] {
#      drop { }
#    }
    if "node03.dev.klid.go.kr" in [host][hostname] {
      drop { }
    }

#windows fiexd type으로 사용 하고 있음.
#    if "filesystem" in [metricset][name] {
#      if "ocfs" not in [system][filesystem][type] and "ext" not in [system][filesystem][type] and "fixed" not in [system][filesystem][type]{
#        drop { }
#      }
#    }
#kubernetes drop
    if "metricbeat" in [agent][type] {
      if "kubernetes" in [service][type] {
        if "proxy" in [metricset][name] or "volume" in [metricset][name] or "node" in [metricset][name] or "apiserver" in [metricset][name] or "system" in [metricset][name] or "state_daemonset" in [metricset][name] or "state_deployment" in [metricset][name] or "state_container" in [metricset][name] or "state_pod" in [metricset][name] or "state_service" in [metricset][name] or "state_replicaset" in [metricset][name] {
          drop { }
        }
      }
    }


    mutate {
      add_field => {
        "tmp_timestamp" => ""
      }
    }

    ruby {
      code => "event.set('tmp_timestamp', event.get('@timestamp').time.strftime('%Y-%m-%d %H:%M:%S'))"
    }

    grok {
      match => {
        "tmp_timestamp" => "%{INT:year_tmp}-%{MONTHNUM:month_tmp}-%{MONTHDAY:day_tmp} %{INT:hour_tmp}:%{INT:minute_tmp}:%{INT:second_tmp}"
      }
    }

    mutate {
      convert => { "second_tmp" => "integer" }
    }

    mutate {
      add_field => {
        "datetime" => "%{year_tmp}-%{month_tmp}-%{day_tmp} %{hour_tmp}:%{minute_tmp}:%{second_tmp}"
      }
      convert => { "datetime" => "string" }

      rename => {
        "[system][process][cgroup][cpuacct][percpu][1]" => "[system][process][cgroup][cpuacct][percpu][c01]"
        "[system][process][cgroup][cpuacct][percpu][2]" => "[system][process][cgroup][cpuacct][percpu][c02]"
        "[system][process][cgroup][cpuacct][percpu][3]" => "[system][process][cgroup][cpuacct][percpu][c03]"
        "[system][process][cgroup][cpuacct][percpu][4]" => "[system][process][cgroup][cpuacct][percpu][c04]"
        "[system][process][cgroup][cpuacct][percpu][5]" => "[system][process][cgroup][cpuacct][percpu][c05]"
        "[system][process][cgroup][cpuacct][percpu][6]" => "[system][process][cgroup][cpuacct][percpu][c06]"
        "[system][process][cgroup][cpuacct][percpu][7]" => "[system][process][cgroup][cpuacct][percpu][c07]"
        "[system][process][cgroup][cpuacct][percpu][8]" => "[system][process][cgroup][cpuacct][percpu][c08]"
        "[system][process][cgroup][cpuacct][percpu][9]" => "[system][process][cgroup][cpuacct][percpu][c09]"
        "[system][process][cgroup][cpuacct][percpu][10]" => "[system][process][cgroup][cpuacct][percpu][c10]"
        "[system][process][cgroup][cpuacct][percpu][11]" => "[system][process][cgroup][cpuacct][percpu][c11]"
        "[system][process][cgroup][cpuacct][percpu][12]" => "[system][process][cgroup][cpuacct][percpu][c12]"
        "[system][process][cgroup][cpuacct][percpu][13]" => "[system][process][cgroup][cpuacct][percpu][c13]"
        "[system][process][cgroup][cpuacct][percpu][14]" => "[system][process][cgroup][cpuacct][percpu][c14]"
        "[system][process][cgroup][cpuacct][percpu][15]" => "[system][process][cgroup][cpuacct][percpu][c15]"
        "[system][process][cgroup][cpuacct][percpu][16]" => "[system][process][cgroup][cpuacct][percpu][c16]"
        "[system][process][cgroup][cpuacct][percpu][17]" => "[system][process][cgroup][cpuacct][percpu][c17]"
        "[system][process][cgroup][cpuacct][percpu][18]" => "[system][process][cgroup][cpuacct][percpu][c18]"
        "[system][process][cgroup][cpuacct][percpu][19]" => "[system][process][cgroup][cpuacct][percpu][c19]"
        "[system][process][cgroup][cpuacct][percpu][20]" => "[system][process][cgroup][cpuacct][percpu][c20]"
        "[system][process][cgroup][cpuacct][percpu][21]" => "[system][process][cgroup][cpuacct][percpu][c21]"
        "[system][process][cgroup][cpuacct][percpu][22]" => "[system][process][cgroup][cpuacct][percpu][c22]"
        "[system][process][cgroup][cpuacct][percpu][23]" => "[system][process][cgroup][cpuacct][percpu][c23]"
        "[system][process][cgroup][cpuacct][percpu][24]" => "[system][process][cgroup][cpuacct][percpu][c24]"
        "[system][process][cgroup][cpuacct][percpu][25]" => "[system][process][cgroup][cpuacct][percpu][c25]"
        "[system][process][cgroup][cpuacct][percpu][26]" => "[system][process][cgroup][cpuacct][percpu][c26]"
        "[system][process][cgroup][cpuacct][percpu][27]" => "[system][process][cgroup][cpuacct][percpu][c27]"
        "[system][process][cgroup][cpuacct][percpu][28]" => "[system][process][cgroup][cpuacct][percpu][c28]"
        "[system][process][cgroup][cpuacct][percpu][29]" => "[system][process][cgroup][cpuacct][percpu][c29]"
        "[system][process][cgroup][cpuacct][percpu][30]" => "[system][process][cgroup][cpuacct][percpu][c30]"
        "[system][process][cgroup][cpuacct][percpu][31]" => "[system][process][cgroup][cpuacct][percpu][c31]"
        "[system][process][cgroup][cpuacct][percpu][32]" => "[system][process][cgroup][cpuacct][percpu][c32]"

        #"[system][load][1]" => "[system][load][1m]"
        #"[system][load][5]" => "[system][load][5m]"
        #"[system][load][15]" => "[system][load][15m]"
        #"[system][load][norm][1]" => "[system][load][norm][1m]"
        #"[system][load][norm][5]" => "[system][load][norm][5m]"
        #"[system][load][norm][15]" => "[system][load][norm][15m]"
      }

      remove_field => [ "year_tmp","month_tmp","day_tmp","hour_tmp","minute_tmp","second_tmp","sec","tmp_timestamp" ]
    }

    date {
      match => ["datetime", "yyyy-MM-dd HH:mm:ss"]
      timezone => "UTC"
      locale => "ko"
      target => "datetime"
    }

  }
  else if "filebeat" in [agent][type] {
    #if "beat" in [message] {
    #  drop { }
    #}

    if "syslog" in [log][file][path] {
      grok {
        match => {
          #message" => "%{HOSTNAME:hostname} %{SYSLOGPROG}: %{GREEDYDATA:tmp_message}"
          "message" => "%{GREEDYDATA:tmp_message}"
        }
      }

      mutate {
        replace => {
          "message" => "%{tmp_message}"
          #"message" => "%{HOSTNAME:hostname} %{SYSLOGPROG}: %{GREEDYDATA:tmp_message}"
          #"message" => "%{message}"
        }

        remove_field => [ "hostname", "tmp_message" ]
      }
    }
  }
}

output {
#  file  {
#    path => "/home/ingest/logstash-8.1.3/logstash_rx_message.log"
#    codec => rubydebug
#  }


  if "ncp" in [agent][type] {
    if "ncp-meta" in [service][type] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-meta-ncp"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else if "ncp-metric" in [service][type] {
      if "cpu" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-ncp-cpu"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "memory" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-ncp-memory"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "network" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-ncp-network"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "disk" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-ncp-disk"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      }
    }
  }
  else if "filebeat" in [agent][type] {
    if "syslog" in [log][file][path] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-log-syslog"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else if "auth" in [log][file][path] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-log-auth"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else if "nova-compute" in [log][file][path] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-nova-compute"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else if "nova-api" in [log][file][path] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-nova-api"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else if "nova-scheduler" in [log][file][path] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-nova-scheduler"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else if "containers" in [log][file][path] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-log-container"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
    else {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-file-test"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
  } else if "metricbeat" in [agent][type] {
    if "kubernetes" in [service][type] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-metric-kube-%{[metricset][name]}"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    } else if "vsphere" in [service][type] {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-metric-vsphere"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    } else {
      opensearch {
        hosts => ["https://127.0.0.1:9200"]
        index => "oke-metric-%{[metricset][name]}"
        user => "admin"
        password => "admin"
        ssl => true
        ssl_certificate_verification => false
      }
    }
  } else if "metric_collector" in [agent][type] {
    if "vmware" in [service][type] {
      if "vsphere" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-vmware-vsphere"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "vcenter" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-vmware-vcenter"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "cluster" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-vmware-cluster"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "host" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-vmware-host"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "datastore" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-vmware-datastore"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "virtualmachine" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-vmware-vm"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      }
    } else if "openstack" in [service][type] {
      if "vm" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-openstack-vm"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "hypervisor" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-openstack-hypervisor"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "loadbalancer" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-openstack-loadbalancer"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "baremetal" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-openstack-baremetal"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "network" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-openstack-network"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "image" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-openstack-image"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      }
    } else if "netapp" in [service][type] {   
      if "filestorage" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-netapp-filestorage"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      }  
    } else if "k8s" in [service][type] {
      if "node" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-k8s-node"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "namespace" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-k8s-namespace"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "pod" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-k8s-pod"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "storage" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-k8s-storage"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "cluster" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-k8s-cluster"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      } else if "workload" in [service][adapter] {
        opensearch {
          hosts => ["https://127.0.0.1:9200"]
          index => "oke-metric-k8s-workload"
          user => "admin"
          password => "admin"
          ssl => true
          ssl_certificate_verification => false
        }
      }
    }
  } else if "filestorage" in [agent][type] {
    opensearch {
      hosts => ["https://127.0.0.1:9200"]
      index => "oke-file-storage"
      user => "admin"
      password => "admin"
      ssl => true
      ssl_certificate_verification => false
    }
  } else {
    opensearch {
      hosts => ["https://127.0.0.1:9200"]
      index => "oke-kube-state-pod"
      user => "admin"
      password => "admin"
      ssl => true
      ssl_certificate_verification => false
    }
  }
}
```

## 5. logstash.yml 설정 변경
* logstash 8.x version에서는 logstash.yml 변경 필수

```
ingest@ubuntu22:~/logstash-8.1.3/config$ vi logstash.yml
```

* vi 편집기에 다음과 같이 작성

```
config.reload.automatic: true
pipeline.ecs_compatibility: disabled 
path.data: /home/ingest/logstash-8.1.3/path_data/data
path.logs: /home/ingest/logstash-8.1.3/path_data/logs
```

## 6. Plugin 설치

```
ingest@ubuntu22:~/logstash-8.1.3$ bin/logstash-plugin install logstash-output-opensearch
```

## 7. config 변경

```
ingest@ubuntu22:~/logstash-8.1.3$ cd /home/ingest/logstash-8.1.3/vendor/bundle/jruby/2.5.0/gems/logstash-output-opensearch-1.2.0-java/lib/logstash/outputs/opensearch/templates/ecs-disabled/

ingest@ubuntu22:~/logstash-8.1.3/vendor/bundle/jruby/2.5.0/gems/logstash-output-opensearch-1.2.0-java/lib/logstash/outputs/opensearch/templates/ecs-disabled$ cp 1x.json 2x.json
```

## 8. Ingest 실행

```
ingest@ubuntu22:~/logstash-8.1.3$ nohup bin/logstash -f /home/ingest/logstash-8.1.3/config/kafka.conf > /dev/null 2>&1 &
```

