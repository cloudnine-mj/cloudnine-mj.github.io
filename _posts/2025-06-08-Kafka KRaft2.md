---
key: jekyll-text-theme
title: 'Kafka의 새로운 협의 프로토콜인 KRaft -2'
excerpt: 'KRaft에 대하여😎'
tags: [Kafka]
---

# 두 번째 스터디 목표

- Zookeeper를 사용하는 Kafka 모드에서 새롭게 도입된 KRaft 모드로 전환하는 방법에 대해 설명한다.

	* KRaft 모드를 사용하면 물리적인 서버의 관리 효율성을 높이고 리소스를 절약할 수 있습니다. 

	* 그러나 안정성 측면에서는 여전히 Kafka 브로커와는 별도로 KRaft 컨트롤러 노드를 운영하는 것이 권장됩니다. 

	* Apache Kafka는 2021년 2.8 버전에서 KRaft 모드를 처음 도입했으며, 2024년에 발표된 Kafka 4.0부터는 Zookeeper 지원이 완전히 중단되고 KRaft 모드만을 지원하게 되었습니다.




## **KRaft의 구성**

- 전통적인 Zookeeper 모드를 사용하면서 많은 사용자들이 느꼈던 불편함 중 하나는 바로 Zookeeper와 Kafka 서버를 별도로 운영해야 한다는 점이었습니다.
    - 이는 단순히 별도의 애플리케이션 운영 관리를 넘어서, 추가로 별도의 물리적 서버 자원의 할당까지 포함하고 있습니다.

### 가장 많은 FAQ는?

>Zookeeper 물리 서버의 할당과 관련된 주제로, Zookeeper와 Kafka를 동일한 서버에서 실행해도 되는지에 관한 것!

- 사실 Zookeeper는 Kafka를 관리하는 역할을 하므로, 이상적으로는 Kafka와 분리된 별도의 서버에서 운영하는 것을 권장합니다.
- 하지만 이는 강제성을 요구하는 것도 아니고, 서버의 리소스 제약이 있는 경우 Zookeeper와 Kafka를 동일한 서버에서 실행할 수도 있습니다.
- KRaft의 등장 이후 Kafka 사용자들이 환영한 변화중 하나는 Zookeeper의 의존성 제거입니다. 이는 애플리케이션의 관리 단순화뿐만 아니라, 물리적 서버의 리소스 절감도 가능하다고 생각했던 것 같습니다.
- 실제로 KRaft 공개 이후, Zookeeper 서버 없이 Kafka를 운영할 수 있는지에 대한 논의가 많았습니다. 하지만 KRaft모드에서도 여전히 브로커와 KRaft를 분리된 서버에 배치하는 것을 강력히 추천하고 있습니다.
- 따라서 Zookeeper 서버의 리소스 절약을 주된 목적으로 KRaft로 전환을 고려하는 것은 추천하지 않습니다.
- 그럼에도 불구하고 서버 자원이 한정적인 경우라면 브로커와 KRaft를 동일한 서버에 배치하는 것을 고려할 수 있습니다.

### 1. KRaft 모드 - KRaft 별도 구성

- Zookeeper 없는 KRaft 방식으로 Kafka를 운영하되, 컨트롤러(KRaft 서버)를 브로커와 분리해서 따로 배치한 구성

- 별도 구성이란? 컨트롤러(KRaft 서버)와 브로커(Kafka 서버)를 서로 다른 서버에서 따로 운영하는 방식입니다.
    - 컨트롤러 서버 → 클러스터의 리더 선출, 메타데이터 관리 등
    - 브로커 서버 → 실제 데이터 처리 (프로듀서/컨슈머가 사용하는 메시지 처리)
- 공식 문서에 따르면, KRaft 구성의 권장 방식은 브로커와 별개로 KRaft 서버를 할당하여 운영하는 것입니다.
    - 이 구성은 브로커와 컨트롤러를 분리하여 높은 가용성과 안정성을 확보하는데 중점을 둡니다.
- 이렇게 분리된 구성은 시스템의 장애 포인트를 최소화하고, 하나의 구성 요소에서 문제가 발생했을 시 전체 시스템에 미치는 영향을 줄일 수 있고, 클러스터의 관리 효율성과 시스템의 안정성을 더욱 강화합니다.
- 특히 대규모 Kafka 클러스터를 운영하거나, 엄격한 SLA를 충족해야 하는 비즈니스 환경에서는 반드시 브로커와 별개로 KRaft 서버를 할당하여 사용하기를 추천합니다.

### 2. KRaft 모드 - KRaft 통합 구성

- Zookeeper 없이 KRaft로 운영하면서, 컨트롤러(KRaft)와 브로커를 한 서버에서 같이 실행하는 구성

- 통합구성이란? 컨트롤러(KRaft 역할)와 브로커(Kafka 역할)를 같은 서버에서 함께 실행하는 구성입니다.
    - Kafka를 설치한 서버가 **브로커 역할**도 하고, 동시에 **KRaft 컨트롤러 역할도 같이 함.**
- 서버의 리소스가 한정적인 경우에는 KRaft와 브로커를 동시에 운영할 수 있습니다. 이 경우 브로커와 컨트롤러가 동일한 서버에서 실행되므로, 해당 서버에 장애가 발생하면 컨트롤러 역시 불필요하게 종료될 수 있습니다.
- 이는 시스템의 높은 가용성과 안정성을 유지하기 위한 모범 사례와 상충될 수 있으므로, 추천하는 방식은 아닙니다.
- 보통 이 경우는 작은 규모의 Kafka 클러스터를 운영하거나, 개발 용도의 경우 적합합니다.



## **KRaft 서버 권장 사양**

KRaft가 Kafka와 같이 많은 데이터를 처리하지 않으므로 권장 사양은 생각보다 높지 않습니다.

- CPU: CPU를 공유하는 경우 Dedicated CPU
- MEM: 최소 4GB
- DISK: 최소 SSD 64GB
- JVM: 최소 1GB
- **Dedicated CPU**

<details>
<summary>Dedicated CPU란?</summary>

<ul>
  <li>특정 프로세스나 컨테이너가 사용하는 <strong>CPU 코어를 다른 작업과 공유하지 않고 독점적으로 사용하는 것</strong></li>
</ul>

<h4>일반적인 상황</h4>

<ul>
  <li>보통 서버에서는 여러 프로세스 (예: Kafka, 다른 애플리케이션)가 <strong>같은 CPU 코어를 함께 씀</strong>  
    → 이를 <strong>Shared CPU</strong>라고 함</li>
  <li>→ <strong>성능이 들쑥날쑥</strong>하고 예측하기 어려움</li>
</ul>

<h4>Dedicated CPU란?</h4>

<ul>
  <li>Kafka 같은 중요한 서비스에 <strong>특정 CPU 코어를 하나 또는 여러 개 전용으로 할당</strong></li>
  <li><strong>다른 작업이 그 코어를 사용하지 못하도록 막음</strong></li>
  <li>→ 결과적으로 <strong>성능이 안정적이고 예측 가능함</strong></li>
</ul>

<h4>언제 쓰냐?</h4>

<ul>
  <li>Kafka, Zookeeper, KRaft 같이 <strong>지연에 민감하고 고성능이 필요한 서비스</strong>에 적합</li>
  <li><strong>리소스 경쟁 없이 안정적인 처리량 확보 가능</strong></li>
  <li>→ 그래서 <strong>Dedicated CPU 설정을 권장</strong>함</li>
</ul>

<blockquote>
<strong>Dedicated CPU</strong> = 해당 CPU 코어를 오직 Kafka만 쓰도록 고정하는 것<br/>
→ 다른 서비스랑 나눠 쓰지 않으므로 <strong>성능 안정성</strong> 높아짐
</blockquote>
</details>



## **KRaft 릴리스 정보와 향후 계획**

KRaft는 2021년 Apache Kafka 2.8 버전과 같이 공개되었습니다.

릴리스 내용을 통해 KRaft의 주요 릴리스 내역과 향후 계획에 대해 알아보겠습니다.([참고 페이지](https://cwiki.apache.org/confluence/display/KAFKA/KIP-833%3A+Mark+KRaft+as+Production+Ready))

- 2021년 Kafka 2.8과 KRaft early access
- 2022년 Kafka 3.3과 KRaft Production ready
- 2023년 Kafka 3.5과 Zookeeper 모드 deprecated, Zookeeper 마이그레이션 Preview
- 2023년 Kafka 3.6과 Zookeeper 마이그레이션 GA
- 2024년 Kafka 3.7과 Zookeeper 모드 지원 마지막 버전
- 2024년 Kafka 4.0과 KRaft only 모드 지원

Apache Kafka 3.7 버전이 Zookeeper 모드를 지원하는 마지막 버전이고, 이후 Kafka 4.0 버전의 경우는 KRaft 모드로만 사용해야 합니다.

현재까지는 Zookeeper 모드로 사용해도 무방하지만, Kafka 4.0 공개 후 Kafka의 필수 최신 기능 등을 사용하기 위해서는 KRaft 모드로의 마이그레이션이 필수입니다.



## **Zookeeper 모드 -> KRaft 모드 마이그레이션**

- 최근 Kafka를 최초 구성하시는 분들은 처음부터 KRaft 모드로 구성하여 사용하시는 분들도 일부 있습니다.
- 하지만 현업에서 Kafka를 활발히 사용하시는 분들은 대부분 Zookeeper 모드를 사용하고 있을 거라 생각됩니다.

>KRaft의 마이그레이션 상세 매뉴얼 형식보다는 전반적인 마이그레이션 방법에 대해 설명하겠습니다.<br>
>KRaft의 마이그레이션 방법은 다음과 같은 순서로 진행합니다.


### 현재 Zookeeper 버전 모드 -> 1차 Kafka 버전 업그레이드(브리지 모드) -> 2차 KRaft 모드 마이그레이션

- Zookeeper 모드로 Kafka를 사용하면서 KRaft 모드로의 전환은 단순한 업그레이드가 아닙니다.
- 내부적으로 사용하는 API와 구성의 근본적인 변화로 인해 Zookeeper 모드에서 KRaft 모드로 다이렉트 마이그레이션은 불가능합니다.
- 이러한 복잡한 이슈를 해결하기 위해 Kafka는 브리지 모드를 도입하였습니다.
- 브리지 모드는 Zookeeper 모드와 KRaft 모드를 동시에 지원하며 두 모드 사이의 다리 역할을 하게 됩니다.
- Zookeeper 마이그레이션 GA가 지원되는 Kafka 3.6 버전이 Zookeeper에서 KRaft로의 전환을 고려할 때 브리지 모드로 고려해야 할 버전으로 추천합니다.

<details>
<summary> 브리지 모드(Bridge Mode)란?</summary>

<p><strong>Zookeeper 모드 → KRaft 모드로 전환</strong>할 때  
<strong>중간 단계에서 둘 다 동시에 사용하는 혼합 모드</strong></p>

<h4>왜 필요한가?</h4>

<ul>
  <li>Kafka의 기존 방식(Zookeeper)과 새로운 방식(KRaft)은 <strong>완전히 다름</strong></li>
  <li>→ API, 메타데이터 구조, 리더 선출 방식까지 모두 변경됨</li>
  <li>→ <strong>단순 업그레이드로는 전환 불가</strong></li>
  <li>Kafka가 이 복잡함을 해결하려고 만든 전환용 구조가 바로 <strong>브리지 모드</strong><br/>
    → Zookeeper와 KRaft를 <strong>동시에 돌릴 수 있게 해줌</strong></li>
</ul>

<h4>브리지 모드에서 할 수 있는 일</h4>

<ul>
  <li>Zookeeper가 기존 클러스터 메타데이터를 계속 관리</li>
  <li>KRaft는 일부 기능을 학습하거나 병행 실행</li>
  <li>사용자는 이 환경에서 점진적으로 Zookeeper 의존도를 줄이며<br/>
    <strong>KRaft 기반 클러스터로 안전하게 전환 가능</strong></li>
</ul>

<h4>브리지 모드를 쓰는 이유</h4>

<ul>
  <li>Zookeeper에서 KRaft로 <strong>한 번에 이동하는 건 위험하고 어렵기 때문</strong></li>
  <li><strong>운영 중단 없이, 안정적으로 이전</strong>하기 위한 방법</li>
  <li>Kafka 3.6부터 <strong>브리지 모드 지원(GA)</strong><br/>
    → Kafka 3.6이 Zookeeper to KRaft 전환에 가장 적합한 버전으로 추천됨</li>
</ul>

<blockquote>
<strong>브리지 모드</strong> = Zookeeper와 KRaft를 동시에 사용해,<br/>
<strong>점진적으로 KRaft로 이전할 수 있게 해주는 Kafka의 전환용 모드</strong>
</blockquote>
</details>


### 1단계: Kafka 3.0 버전 + Zookeeper 모드 -> Kafka 3.6 버전 + Zookeeper 모드


- 현재 사용하고 있는 Kafka 버전에서 Zookeeper 마이그레이션 GA가 지원되는 Kafka 3.6 버전으로 먼저 업그레이드를 해야 합니다.
- 버전 업그레이드 시 현재 Kafka의 상태 파악, 사전 준비, 업그레이드 계획 수립, 테스트 및 검증, 실제 업그레이드 실행, 사후 점검 등의 상세한 작업등을 진행해야 합니다.
- 업그레이드 작업 시 업그레이드 전략은 롤링 업그레이드 방식으로 진행할 수 있으며, 클라이언트의 타임아웃이 일부 발생하기는 하지만 무중단으로 업그레이드가 가능합니다.
- 참고 도서:  <[Kafka, 데이터 플랫폼의 최강자](https://www.yes24.com/Product/Goods/59789254)>, <[실전 Kafka 개발부터 운영까지](https://www.yes24.com/Product/Goods/104410708)>


### 2단계: Kafka 3.6 버전 + Zookeeper 모드 -> Kafka 3.6 버전 + KRaft 모드

- 버전 업그레이드를 성공했다면, 현재 실행 중인 Kafka의 버전은 3.6입니다.
- 이제 KRaft 모드로 마이그레이션을 하기 위한 사전 준비가 완료되었으며, 실제 마이그레이션에 대한 상세 매뉴얼은 컨플루언트 [공식 문서](https://docs.confluent.io/platform/current/installation/migrate-zk-kraft.html)를 참고할 수 있습니다.
    - 해당 문서에서 설명하는 대로 따라 하면 무리 없이 KRaft로 전환을 할 수 있습니다.
- 마이그레이션 과정에서 특히 주의 깊게 확인해야 하는 부분은 바로 로그입니다. 작업 중간에 로그를 주기적으로 확인하면서 문제가 없는지 체크해야 합니다.
- 마이그레이션 작업 중 로그를 확인하다 보면 아래와 같이 Quorum, KRaft와 같은 단어들을 확인할 수 있습니다.

```scala
$ cat /usr/local/kafka/logs/controller.log
[2024-03-00 00:00:00,754] INFO [QuorumController id=3003] Creating new QuorumController with clusterId xfKvdoiPRr6Kj3uaaMrFIQ. ZK migration mode is enabled. (org.apache.kafka.controller.QuorumController)
[2024-03-00 00:00:00,612] INFO [ZooKeeperClient KRaft Migration] Initializing a new session to kcluster01.foo.bar:2181,kcluster02.foo.bar:2181,kcluster03.foo.bar:2181. (kafka.zookeeper.ZooKeeperClient)
[2024-03-00 00:00:00,832] INFO [ZooKeeperClient KRaft Migration] Waiting until connected. (kafka.zookeeper.ZooKeeperClient)
```

> **Quorum:** KRaft 모드에서는 새로운 협의 알고리즘과 투표 기반의 리더 선출 과정에서 사용하는 중요한 개념입니다. Quorum 관련 로그는 클러스터의 상태와 선출 과정 등을 이해하는데 도움이 됩니다.
>

> **KRaft**: KRaft 모드의 동작 상태나 관련된 경고, 에러 메시지등이 기록되므로 관련 로그를 주의 깊게 확인해야 합니다.
>

또한, 컨트롤러의 리더 변경이 일어나게 되면, 아래와 같은 로그도 확인할 수 있습니다.

```scala
[2024-03-00 00:00:00,851] INFO [QuorumController id=3003] In the new epoch 7, the leader is (none). (org.apache.kafka.controller.QuorumController)
[2024-03-00 00:00:00,908] INFO [QuorumController id=3003] In the new epoch 7, the leader is 3001. (org.apache.kafka.controller.QuorumController)
```

KRaft 마이그레이션 작업 역시 롤링 업그레이드 방식으로 진행하며,

테스트 결과에 따르면  클라이언트의 타임아웃이 일부 발생하기는 하지만 무중단으로 업그레이드가 가능합니다.

### 3단계: Kafka 3.6 버전 + KRaft 모드

- KRaft 모드로의 전환은 Kafka의 운영을 단순화하고, Zookeeper에 대한 의존성을 제거하는 중요한 변화입니다.
- 이 변화로 인해 Kafka는 클러스터의 메타 데이터관리를 자체적으로 수행할 수 있으며, 이에 따라 새로운 도구들이 필요합니다.
- kafka-metadata-quorum 도구는 KRaft 모드에서 클러스터 ID, 현재 리더, 팔로워 LAG 등 중요한 정보를 조회하는 데 사용되며, 현재 상태와 리더, 팔로워 LAG 등을 파악하는데 필수적인 도구입니다.

```scala
$ /usr/local/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server  kcluster01.foo.bar:9092,kcluster02.foo.bar:9092,kcluster03.foo.bar:9092 describe --status ClusterId:              AcVsv1FpSq-0yX3iTY01lw 
LeaderId:               3002 
LeaderEpoch:            5 
HighWatermark:          1699 
MaxFollowerLag:         0 
MaxFollowerLagTimeMs:   493 
CurrentVoters:          [3001,3002,3003] 
CurrentObservers:       [1,2,3] 

# 리더 변경 후 
$ /usr/local/kafka/bin/kafka-metadata-quorum.sh --bootstrap-server  kcluster01.foo.bar:9092,kcluster02.foo.bar:9092,kcluster03.foo.bar:9092 describe --status ClusterId:              AcVsv1FpSq-0yX3iTY01lw
LeaderId:               3001
LeaderEpoch:            6
HighWatermark:          2161
MaxFollowerLag:         0
MaxFollowerLagTimeMs:   0
CurrentVoters:          [3001,3002,3003]
CurrentObservers:       [1,2,3]
```

위의 과정처럼 현재 실행 중인 Kafka는 Kafka 3.6 버전 + KRaft 모드이므로, KRaft 모드로의 마이그레이션이 완료할 수 있습니다.

필요 시  Kafka 4.0 버전(KRaft only)으로도 업그레이드도 가능합니다.

업그레이드 전 고려 사항이나 필요한 작업 등은 향후 공개될 Apache Kafka 4.0 버전의 릴리스를 보면서 진행하면 될 것 같습니다.



## **참고(KRaft의 발음에 대해)**

- 발음의 통일은 효과적인 커뮤니케이션을 용이하게 하고 기술적 대화의 명확성에 기여한다고 생각합니다.
- 결론부터 말씀드리자면, KRaft의 발음은 '크래프트'가 맞습니다!
- 또한 [해외 사이트](https://www.baeldung.com/kafka-shift-from-zookeeper-to-kraft)와 컨플루언트(Confluent)의 공식 [YouTube 채널](https://www.youtube.com/watch?v=EUwwNnVyc4c)에서도 '크래프트'라는 발음이 권장되고 있는 것을 확인할 수 있습니다

> Kafka, in its architecture, has recently shifted from ZooKeeper to a quorum-based controller that uses a new consensus protocol called Kafka Raft, shortened as Kraft (pronounced “craft”).