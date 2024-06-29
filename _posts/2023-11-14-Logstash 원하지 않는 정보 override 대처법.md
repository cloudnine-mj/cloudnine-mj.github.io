---
key: jekyll-text-theme
title: 'Logstash - 원하지 않는 정보 override 대처법'
excerpt: 'Logstash research 😎'
tags: [Logstash]
---

# Logstash - 원하지 않는 정보 override 대처법

- logstash에서 ECS라고 해서 metric template 같은 걸 자체적으로 만듬
- 이걸 사용하면 http input을 받을때 여러 정보가 들어옴 
- 싫으면 disabled 처리하면 되지만 그것 나름대로 데이터가 좀 난잡하게 들어오게 된다 
- 사용 예시

```
input {
  http {
    #ecs_compatibility => disabled
    port => 8080
    codec => json {}
  }
}

{
    "user_agent" => {
        "original" => "Go-http-client/1.1"
    },
          "http" => {
        "version" => "HTTP/1.1",
        "request" => {
                 "body" => {
                "bytes" => "110"
            },
            "mime_type" => "application/json"
        },
         "method" => "POST"
    },
       "message" => "This is a sample log message",
    "@timestamp" => 2023-11-13T06:22:55.571876362Z,
         "event" => {
        "original" => "{\\"host\\":{\\"host_ip\\":\\"192.168.0.150\\",\\"hostname\\":\\"test\\"},\\"level\\":\\"info\\",\\"message\\":\\"This is a sample log message\\"}"
    },
           "url" => {
        "domain" => "192.168.0.150",
          "path" => "/",
          "port" => 8080
    },
          "host" => {
        "hostname" => "test",
              "ip" => "192.168.0.150"
    },
         "level" => "info",
      "@version" => "1"
}
```

- 문제는 사용자 임의대로 데이터를 만들고 받을 때 중복되는 부분이 있으면 ecs compatibility가 켜져 있을때 원하지 않는 데이터가 override 될 때가 있다는 점임. 
- [host] [ip] 를 메타정보로 넣어주고 싶은데 ecs~ 가 마음대로 sender의 ip를 할당해버림 
- 이 부분은 전처리 과정이라서 기능을 끄거나 할 방법이 (아직까진) 보이지 않음
- 이걸 해결하기 위해선 두 가지 방법이 있다.
	* source data 변경 후 rename
	* source 변경 없이 json parsing

## source data 변경 후 rename

- 아예 host_ip로 할당해서 데이터를 넣고

```
payload := map[string]interface{}{
		"message": "This is a sample log message",
		"level":   "info",
		"host": map[string]string{
			"host_ip": "192.168.0.150",
			"hostname": "test",
		},
```

- filter 설정해서 원하는 필드에 다시 덮어씌우고 remove

```
filter {
  mutate {
    update => { "[host][ip]" => "%{[host][host_ip]}" }
    remove_field => [ "[host][host_ip]"]
  }
}
```

## source 변경 없이 json parsing

- 이 부분은 소스코드 변경 등은 필요 없다.
- 필터 변경으로만 해결할 수 있지만 다소 cost가 더 들어갈 것 같고 번거로움
- json으로 오리지널 데이터를 파싱하고 ip 값을 추출해서 덮어씌우는 방법 

✊주의 : source 부분의 데이터가 어떻게 들어오는지 파악해야 함. 테스트 시엔 [event] [original] 이었지만 다를 수 있음

```
filter {
  json { 
    source => "[event][original]"
    target => "parsed_data"
  }

  mutate {
    update => { "[host][ip]" => "%{[parsed_data][host][ip]}" }
    remove_field => [ "parsed_data" ]
  }
}
```
