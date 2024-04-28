---
key: jekyll-text-theme
title: 'Workflow'
excerpt: 'Airflow, Dagster, Prefect + Workflow Task 😎'
tags: [Airflow, Dagster, Prefect+Workflow, Research]
---

# Ariflow Arhcitecture

* Airflow는 scheduler, webserver, worker 세 요소로 이루어짐

* 큰 부하를 일으키는 대규모 airflow의 경우 celery executor를 주로 사용하는데 이게 message queue를 하나 더 사용해야 돼서 아무리 생각해도 부하 측면에서 코스트가 좀 큼

* k8s를 이용할 경우에 worker를 pod으로 컨테이너화 해서 띄울 수 있음


## 구조

### 1. pod Worker의 동작 방식
* K8s Operator를 사용시에 Operator가 작동하고, Image를 pull해서 Local Operator를 실행한다.

* Scheduling은 Pod을 만들고 실행하는 역할을 하고, Pod 내부에서는 Local operator로 실행하며 내부 구조는 Localoperator를 사용하는것과 동일하다. 먼저 Pod을 생성해주는 레이어가 추가되었다는 차이점이라고 볼 수 있음.

### Code 실행 방식
* Python 환경을 맞춘 Image를 pod로 올려 코드를 실행시키는 것과, Module별로 분리한 이미지를 만들어 Pod별로 다르게 실행시키는 것의 차이가 있다. (내부 설정엔 문제가 없음)

* 전자의 경우는 구조가 간단하고 관리측면의 난점이 많지 않으나, PV를 만들어 코드를 통합해야 한다는 단점이 있다. 어차피 Module별로 따로 Pod를 올릴거니 큰 문제는 없을 것임.

* 후자의 경우는 Image가 분리되어 배포 측면에서 깔끔하고, CI/CD에 강점이 생긴다. 구조는 복잡해지지만 PV등은 없으므로 오히려 관리의 난점은 해결될 수 있음.


## Todo
### Module image 만들기

* 현재 Image의 작동방식은 알겠으나 이걸 자유롭게 바꿔가며 테스트해보진 못했으므로 확신이 안 든다.

* Pod operator가 이미지를 불러와서 실행했을 때 어떤 문제가 있는지 고민해 봐야 함.

* 고민하다 보면 어떤 식으로 Code 부분을 실행하고 관리할지에 대한 결론이 나올 수도 있음.

### Helm 탈출

* 현재 Helm으로 구성테스트는 해봤지만 실제로 Deploy해보지 않아서 관리에 난점이 있음

* 설정을 정확하게 알 수 없는 Helm은 피하고 내부 구조 파악을 위해 분해해 봐야 함.

### Log 통합 방안

* Pod는 PV를 통해 Log를 통합하고 관리할 수 있어야 함

* 이 부분은 k8s dependency가 강함.

### CI/CD

* 이미지를 만들고 관리하고 코드를 최대한 배포하기 쉬운 환경 구성에 목표를 둬야 한다

### Externel service

* Log를 Minio 등으로 대체할 수 있는가? code를 어떤식으로 관리할 것인가? 사용할 수 있는 것들이 있다면 사용할 수 있는 것을 찾아야 함.