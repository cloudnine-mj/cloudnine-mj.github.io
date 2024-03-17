---
key: jekyll-text-theme
title: 'Ingest 에러 Troubleshooting'
excerpt: ' Logstash indexer(lngest) 에러 해결 😎'
tags: [Ingest, Troubleshooting]
---

# Ingest error Troubleshooting

## Ingest (Logstash Indexer) 에러 로그

```
[2024-02-13T00:52:25,930][WARN ][logstash.outputs.opensearch][main][5fd481a341e11e4e3b067116e34f3b02e9145e38ed34d1b8d5959b61ebbc894b] Could not index event to OpenSearch. {:status=>400, :action=>["index", {:_id=>nil, :_index=>"oke-metric-filesystem", :routing=>nil}, {"@version"=>"1", "metricset"=>{"period"=>30000, "name"=>"filesystem"}, "cloud"=>{"provider"=>"huawei", "region"=>"", "service"=>{"name"=>"ECS"}, "instance"=>{"id"=>"427d8253-6e30-491e-92ce-4ccf520ce329"}, "availability_zone"=>"nova"}, "event"=>{"dataset"=>"system.filesystem", "module"=>"system", "duration"=>3407627}, "host"=>{"id"=>"ef3336d4949c28f4fde17fe78fc57663", "mac"=>["FA-16-3E-EB-25-6F"], "name"=>"logstash-1", "ip"=>["10.0.19.95", "fe80::f816:3eff:feeb:256f"], "containerized"=>false, "architecture"=>"x86_64", "hostname"=>"logstash-1", "os"=>{"platform"=>"ubuntu", "version"=>"20.04.4 LTS (Focal Fossa)", "name"=>"Ubuntu", "family"=>"debian", "type"=>"linux", "codename"=>"focal", "kernel"=>"5.4.0-105-generic"}}, "system"=>{"filesystem"=>{"device_name"=>"hugetlbfs", "total"=>0, "used"=>{"bytes"=>0}, "free"=>0, "available"=>0, "options"=>"rw,relatime,pagesize=2M"}}, "service"=>{"type"=>"system"}, "fields"=>{"agtId"=>"BM", "instId"=>99, "object_id"=>"427d8253-6e30-491e-92ce-4ccf520ce329", "identifier"=>"427d8253-6e30-491e-92ce-4ccf520ce329", "AgtKindName"=>"metricbeat"}, "identifier"=>"427d8253-6e30-491e-92ce-4ccf520ce329", "agent"=>{"id"=>"a9990d98-12f0-4fc5-9502-04d38c50c0f8", "version"=>"8.12.0", "type"=>"metricbeat", "name"=>"logstash-1", "ephemeral_id"=>"3a7cf019-c17b-4341-860b-04fb50fb2ca0"}, "tags"=>["beats_input_raw_event"], "ecs"=>{"version"=>"8.0.0"}, "@ctime"=>"2024-02-13T00:52:14.312Z", "object_id"=>"427d8253-6e30-491e-92ce-4ccf520ce329", "datetime"=>2024-02-13T00:52:14Z, "@timestamp"=>2024-02-13T00:52:14.312Z, "@itime"=>"2024-02-13T00:52:14.312Z"}], :response=>{"index"=>{"_index"=>"oke-metric-filesystem", "_id"=>"P2Dzn40Bu-Pi-YXVaDa9", "status"=>400, "error"=>{"type"=>"mapper_parsing_exception", "reason"=>"failed to parse field [fields.agtId] of type [long] in document with id 'P2Dzn40Bu-Pi-YXVaDa9'. Preview of field's value: 'BM'", "caused_by"=>{"type"=>"illegal_argument_exception", "reason"=>"For input string: \"BM\""}}}}}
```

## 에러 원인

```
fields:
  agtId: BM  #이게 문제임...
  instId: 99
  AgtKindName: metricbeat
  object_id: 427d8253-6e30-491e-92ce-4ccf520ce329
  identifier: 427d8253-6e30-491e-92ce-4ccf520ce329
```

* agtId에 잘못된 내용 들어가서 데이터 수집팀 로컬에서는 정상적으로 데이터가 전송이 되는데, OpenSearch에서는 데이터가 안보이는 현상이 지속됨.

* 즉, Logstash가 'fields.agtId' 필드를 'long' 타입으로 파싱하려고 시도했지만, 해당 필드에는 'BM(BareMetal)'과 같은 유효한 long 형식이 아닌 값이 있어서 파싱할 수 없다. 이로 인해 매핑 구문 분석 예외가 발생.


## Troubleshooting

- agtId를 원래 넣어야하는  숫자로 바꿔서 데이터를 다시 수집하는 것으로 변경했고, 이후 정상적으로 데이터 수집이 진행됨.
