---
key: jekyll-text-theme
title: 'Prometheus Label 추출'
excerpt: 'Prometheus Lable 추출하는 방법 찾아보기 😎'
tags: [Prometheus, Research]
---

# Prometheus Label 추출

* Query로 할 방법을 고민해봤으나, 프로메테우스는 ‘value값은 숫자여야 한다’라는 원칙을 고수하고 있다.

* 즉 query를 통해서 value값을 얻어오는 함수 등을 이용해서 받아오는 것이 불가능하다.

* 우회할 방법은 대략 두 가지 정도가 된다.
	* Prometheus API를 통한 추출
	* Grafana templating을 이용한 추출



# 1. Prometheus API를 통한 추출

- api에서 series를 제공하는데, 다음과 같이 구성됨.
- phase, namespace 등의 정보를 기본적으로 제공하고, 해당 데이터를 가공해서 목록을 뽑아내야 한다.
- Query 결과값으로도 동일한 방법은 가능하지만, 상세한 데이터를 제공하기 때문에 series가 사용하기에 적절함.
- 단, 반드시 추출 과정이 필요하고, 바로 1 call로 데이터를 가져올 수는 없음. query 등으로 해결은 불가능하다.


```
GET api/v1/series?match[]=kube_pod_status_phase
{
  "status": "success",
  "data": [
    {
      "__name__": "kube_pod_status_phase",
      "instance": "kube-state-metrics.kube-system.svc.cluster.local:8080",
      "job": "kube-state-metrics",
      "namespace": "airflow",
      "phase": "Failed",
      "pod": "airflow-postgresql-0",
      "uid": "5bf1011c-714f-4e1c-8552-b50c11697fde"
    },
    // 다른 라벨 셋...
  ]
}
```

# 3. Grafana templating을 이용한 추출

👉 [How to extract label values from Prometheus metrics in Grafana | Grafana Labs](https://grafana.com/blog/2023/02/23/how-to-extract-label-values-from-prometheus-metrics-in-grafana/)


* Grafana에서 Promethues 데이터를 로드하고, templating을 통해 쉽게 가져오는 방법이 있는 것으로 보인다. label_values() 와 같은 함수의 형태가 있다고 한다.
* Grafana에서 table을 만들고 해당 데이터를 api로 Call하는 방법도 고민해 볼 수 있다.
* 장기적으로 Grafana는 구성되어 들어갈 가능성이 높으니 Grafana와 접목한다면 오히려 괜찮은 수단이 될 수 있음.
* grafana 또한 api를 제공하고 해당 데이터를 받아올 수 있기 때문에 데이터를 가공하는 것보다 편한 수단이 될 수 있다.