---
key: jekyll-text-theme
title: 'OpenSearch - shell 파일 적용'
excerpt: 'Kafka Research😎'
tags: [OpenSearch, ELK]
---


# OpenSearch shell 파일 적용

## 목표

* OpenSearch 데이터 적재 확인 자동화를 위해 shell 파일을 작성하고 crontab을 적용한다.


### Shell 파일

```
curl -XGET "https://localhost:9200/*/_search" -u admin:admin --insecure -H 'Content-Type: application/json' -d'
{ 
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h/h",
              "lt": "now/h"
            }
          }
        },::
        {
          "exists": {
            "field": "$line"
          }
        }
      ]
    }
  }
}'|jq|grep value|awk '{print $2}'|cut -d "," -f1|head -1
```

```
#!/bin/bash
  
today=`date +%Y-%m-%d`
mkdir /root/mj/index_check/log_$today
log=/root/mj/index_check/log_$today/check_index.txt
fail_log=/root/mj/index_check/log_$today/fail_log.txt

cat /dev/null > $log
cat /dev/null > $fail_log

while read line
do

generate_post_data()
{
cat << EOF
{ 
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h/h"
            }
          }
        },
        {
          "exists": {
            "field": "$line"
          }
        }
      ]
    }
  }
}
EOF
}

result=$(curl --header 'Content-Type: application/json' \
  --data "$(generate_post_data)" \
  -XGET "https://<주소>/*/_search" \
  -u admin:admin --insecure|jq|grep value|awk '{print $2}'|cut -d "," -f1|head -1)



if [[ $result -gt 0 ]]; then
        echo "$line is here" >> $log
else
        echo "$line is not okay " >> $log
        echo "$line is fail, please check it" >> $fail_log
fi
```


## crontab 적용

```
# m h  dom mon dow   command
0 * * * * sh /home/logstash/logstash-8.1.3_metric/logstash_delete.sh
0 0,7 * * * bash /home/logstash/index_check/index_check.sh
0 * * * * cat /dev/null > /home/logstash/logstash-8.1.3_metric/logstash_rx_message.log
```
