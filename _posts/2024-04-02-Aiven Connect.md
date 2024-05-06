---
key: jekyll-text-theme
title: 'Aiven Connect 설치'
excerpt: 'Opensearch Connect 설치/설정 😎'
tags: [KsqlDB, Aiven connect, 데이터 가공]
---

# Aiven Connect 설치

## fat jar build

* Aiven connect 오류를 방지하는 fat jar build 하기

:star: Aiven에서 제공하는 binary 파일을 받아 실행하게 되면, docker, kubernetes 환경에서 jar들이 서로 찾지 못하는 호환성 문제가 발생한다. 
:star: 이를 해결하기 위해 소스코드를 받아 shdowjar를 사용 ->  jar들을 한곳에 묶는 fat jar를 build 한다.


## Github 주소

* [https://github.com/Aiven-Open/opensearch-connector-for-apache-kafka](https://github.com/Aiven-Open/opensearch-connector-for-apache-kafka)

## 수정 및 추가

* intellj에서 프로젝트 실행 후 build.gradle 폴더 수정, plugins 부분에 shdowjar를 실행하는 플러그인 코드를 추가한다.

```
 * Copyright 2019 - 2021 Aiven Oy
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

plugins {
    //shadowjar를 실행하는 플러그인 추가
    id("com.github.johnrengelman.shadow") version "8.1.1"
    
    // https://docs.gradle.org/current/userguide/java_library_plugin.html
    id "java-library"

    // https://docs.gradle.org/current/userguide/jacoco_plugin.html
    id "jacoco"

    // https://docs.gradle.org/current/userguide/distribution_plugin.html
    id "distribution"

    // https://docs.gradle.org/current/userguide/publishing_maven.html
    id "maven-publish"

    // https://docs.gradle.org/current/userguide/idea_plugin.html
    id 'idea'

    // https://plugins.gradle.org/plugin/com.diffplug.spotless
    id "com.diffplug.spotless" version "6.23.0"
    
....
```

* ./gradlew shdowjar 명령어를 실행하거나  build

* build/libs 안에 있는 snapshot으로 만들어진 jar를 클래스 패스에 넣어준다.

```
pod에 접속해 jar가 있는지 확인

k exec -it kafka-connect-pod-585cdd9998-v5mmf bash
ls /usr/share/java/libs
```

## CURL 명령어로 Plugin 확인

```
curl -X GET "http://192.168.2.52:30099/connector-plugins"

root@k8s-master:~/km/ksqldb# curl -X GET http://192.168.2.52:30099/connector-plugins | jq '' 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   613  100   613    0     0   119k      0 --:--:-- --:--:-- --:--:--  119k
[
  {
    "class": "io.aiven.kafka.connect.opensearch.OpensearchSinkConnector",
    "type": "sink",
    "version": "3.2.0-SNAPSHOT"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "type": "sink",
    "version": "10.7.4"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "type": "source",
    "version": "10.7.4"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "7.3.7-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "7.3.7-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "7.3.7-ccs"
  }
]
```
