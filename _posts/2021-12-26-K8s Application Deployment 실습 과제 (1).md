---
key: jekyll-text-theme
title: 'K8s Application Deployment 과제'
excerpt: ' Docker/K8s 공부하기😎'
tags: [K8s]
---

# 실습 과제

## 20주차 과제

### 진행단계

* `Chapter04 ElasticSearch를 활용한 로그 수집기 구축 - ElasticStack, Kibana 등` 수강
* 2021년 12월 28일 (화)까지
* 과제 수행에 대한 보고서 및 실제 `수행한 명령어` 캡처해서 구글 form에 제출


### 과제 설명

웹 로그 수집을 위한 사이드 컨테이너 및 파이프 라인 구축

* 웹/사이드카 컨테이너를 임의로 선정
  * 해당 웹/사이드카를 선정한 이유 작성
  * `수행한 명령어` 캡처
* 웹 로그를 저장하는 엘라스틱서치가 데이터를 영구적으로 보관할 수 있는 방법을 제시 (작성)
  * 이때 엘라스틱서치가 새 버전으로 업데이트 되어도 데이터가 보존되어야 함
  * `수행한 명령어` 캡처
  * 절차와 디버깅 순서는 로그 확인, 구조와 특징 염두하여 판단

### 작업

* 계정 변경
  * `$ sudo -i`
* 기존에 있던 모든 파드 삭제
  * `# kubectl delete all --all`
  * 강의 때 진행했던 ElasticSearch, Kibana 등 삭제 후 다시 설치
* ElasticSearch, Kibana, Filebeat 모두 `7.14.1` 버전으로 통일
  * 버전이 맞지 않으면 로그 수집이 원활히 이루어지지 않음
* Filebeat  configmap 작성
  * `# vim filebeat-configmap.yml`
    ~~~yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: filebeat-configmap
    data:
      filebeat.yml: |
        filebeat:
          config:
            modules:
              path: /usr/share/filebeat/modules.d/*.yml
              reload:
                enabled: true
          modules:
          - module: nginx
            access:
              var.paths: ["/var/log/nginx/access.log*"]
            error:
              var.paths: ["/var/log/nginx/error.log*"]
        output:
          elasticsearch:
            hosts: ["172.30.5.70:9200"]
    ~~~
  * configmap은 데이터를 key-value 쌍으로 저장하는 데 사용하는 API 오브젝트
    * 기밀이 아닌 데이터 저장
    * configmap 내 저장 데이터는 최대 `1MiB` 미만
  * configmap으로 설정 데이터를 저장, 다른 파드에서 사용
* Filebeat configmap 활성화
  * `# kubectl apply -f filebeat-configmap.yml`
* nginx Pod 작성
  * `# vim nginx-pod.yml`
    ~~~yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-sidecar
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-logs
          mountPath: /var/log/nginx
      - name: filebeat-sidecar
        image: docker.elastic.co/beats/filebeat:7.14.1
        volumeMounts:
        - name: nginx-logs
          mountPath: /var/log/nginx/
        - name: filebeat-config
          mountPath: /usr/share/filebeat/filebeat.yml
          subPath: filebeat.yml
      volumes:
      - name: nginx-logs
      - name: filebeat-config
        configMap:
          name: filebeat-configmap
          items:
          - key: filebeat.yml
            path: filebeat.yml
    ~~~
  * 위에서 작성한 Filebeat configmap을 사용하여 로그 수집
* nginx Pod 활성화
  * `# kubectl apply -f nginx-pod.yml`
* ElasticSearch 설치
  * `# docker run -v /root/task/elasticsearch/logs:/usr/share/elasticsearch/data -d --name es01-test --net elastic -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:7.14.1`
    * `-v` 옵션은 `Bind mount a volume` 지정
* Kibana 설치
  * `# docker run -d --name kib01-test --net elastic -p 5601:5601 -e "ELASTICSEARCH_HOSTS=http://es01-test:9200" kibana:7.14.1`
* nginx Pod 내부에 접속, `curl` 등 호출하여 트래픽 발생
  * `# kubectl exec -it nginx-sidecar -- bash`
  * `# curl localhost:80`
  * `# exit`
* ElasticSearch 로그 내용 확인
  * 브라우저에서 `http://172.30.5.70:5601/` 접속
  * 인덱스 패턴 생성 `filebeat*`
    * `http://172.30.5.70:5601/app/management/kibana/indexPatterns/create`
  * `discover`를 통해 로그 확인