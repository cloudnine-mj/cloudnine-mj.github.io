---
key: jekyll-text-theme
title: 'Docker_K8s 부트캠프 최종과제-movie_api'
excerpt: ' Docker/K8s 공부하기😎'
tags: [Docker, K8s]
---

# 과제 수행 방법

## 도커 설정
* `Dockerfile` or `Dockercompose` 빌드 시 기존 도커 컨텍스트 변경 여부 확인
  * 강의 중간에 vscode에서 원활한 원격 접속을 위해 컨텍스트 변경했었음
  * `$ docker context create --docker host=tcp://172.30.7.133:2375 my-remote`
    * `172.30.7.133:2375` 는 kakaoicloud VM IP 주소
  * `$ docker context use my-remote`
* 기존 `default` 컨텍스트로 변경
  * `$ docker context use default`

## Movie API 구현
* `Kotlin`, `SpringBoot`, `JPA`를 활용해 API 서버 구현
  * JPA와 메모리 디비를 활용해 CRUD 구현
* `Gradle`의 `bootJar` 명령을 활용해 해당 앱의 Jar 파일 생성
* CRUD 기능 구현 후 도커 이미지 생성(빌드), 도커 허브에 업로드
  * `$ sudo docker build -t kangj/assignment_movie-api .`
    * `$ sudo docker build -t <계정명>/<저장소명>:[태그명] .`
    * 버전 태그를 태깅했었으나 지속적인 수정으로 생략
  * 도커 허브에 접속, 로그인 후에 해당 이미지를 업로드할 `Repository` 생성
    * `$ docker login`
    * `$ docker push kangj/assignment_movie-api`
  * `yaml` 설정 파일의 `metaname` 등의 필드에는 언더바를 사용할 수 없음
    ~~~yaml
    metadata:
      name: assignment_movie-api
    ~~~
    * 에러
        ~~~console
        root@kmj-01:~# kubectl apply -f assignment_movie.yaml
        
        Error from server (Invalid): error when creating "assignment_movie.yaml": Pod "assignment_movie-api" is invalid: [metadata.name: Invalid value: "assignment_movie-api": a lowercase RFC 1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*'), spec.containers[0].name: Invalid value: "assignment_movie-api": a lowercase RFC 1123 label must consist of lower case alphanumeric characters or '-', and must start and end with an alphanumeric character (e.g. 'my-name',  or '123-abc', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?')]
        Error from server (Invalid): error when creating "assignment_movie.yaml": Service "assignment_movie-api" is invalid: metadata.name: Invalid value: "assignment_movie-api": a DNS-1035 label must consist of lower case alphanumeric characters or '-', start with an alphabetic character, and end with an alphanumeric character (e.g. 'my-name',  or 'abc-123', regex used for validation is '[a-z]([-a-z0-9]*[a-z0-9])?')
        ~~~
    * 따라서 `assignment_movie-api` 값을 `assignment-movie-api`와 같은 형태로 모두 변경
  * `nodePort` 설정시 `30000-32767` 범위에서 설정
  * `ImagePullBackOff` 에러인 경우 도커 로그인
    * 도커 로그인 에러 발생
      ~~~console
      root@kmj-01:~# docker login
      Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
      Username: kangjung
      Password:
      Error saving credentials: error storing credentials - err: exit status 1, out: `Cannot autolaunch D-Bus without X11 $DISPLAY`
      ~~~
    * `# apt-get install gnupg2 pass` 명령 수행
      * 디지털 서명, 인증과 관련된 도구(툴) 설치
      ~~~console
      root@kmj-01:~# docker login
      Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
      Username: kangjung
      Password:
      WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
      Configure a credential helper to remove this warning. See
      https://docs.docker.com/engine/reference/commandline/login/#credentials-store
      
      Login Succeeded
      ~~~

## Movie UI 구현
* `Kotlin`, `SpringBoot`, UI(프론트) 서버 구현
  * `html` 파일 등의 경로를 인식하기 위해 `thymeleaf` 사용
* `Gradle`의 `bootJar` 명령을 활용해 해당 앱의 Jar 파일 생성

## 테스트
* 웹 브라우저에서 `172.30.7.133:30088` 접속하여 테스트