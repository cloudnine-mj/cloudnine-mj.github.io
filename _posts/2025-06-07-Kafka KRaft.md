
key: jekyll-text-theme
title: 'Kafka의 새로운 협의 프로토콜인 KRaft -1'
excerpt: 'KRaft에 대하여😎'
tags: [Kafka]


# 첫 번째 스터디 목표

- KRaft의 등장과 Zookeeper를 사용하면서의 몇 가지 단점들에 대해 살펴보고, Zookeeper 모드와 KRaft 모드의 주요 차이점을 살펴본다.
    - Kafka를 사용하면서 초기에는 최신 버전의 릴리스를 추구했지만, Kafka가 점점 데이터 파이프라인의 중심이 되면서 보다 보수적으로 접근할 필요성이 있음.
        - Kafka의 중요도가 높아졌기 때문에, 버전 업그레이드나 설정 변경, 기능 실험 등을 쉽게 할 수 없게 되었다는 뜻 (즉, 안정성이 최우선이 되므로, 변화에 대해 신중하게 접근해야 한다는 얘기)
    - 이제는 KRaft에 대한 준비와 Zookeeper 모드로 운영 중인 Kafka를 마이그레이션 하는 방법 등에 대해서도 심도 있는 리서치가 필요함.



## **KRaft 사용 시 장점**

- Apache Kafka의 새로운 메커니즘인 KRaft를 사용하면 메타데이터를 효율적으로 관리할 수 있습니다.
- KRaft는 Zookeeper의 의존성을 제거하고, Kafka 클러스터 내 컨트롤러가 선출된 후 메타데이터를 직접 관리합니다. 이로 인해 성능과 안정성이 향상되며, 유지보수가 단순화되고, 병목현상이 줄어듭니다.
  - KRaft는 이런 중요한 정보를 **Zookeeper 대신 Kafka 내부 컨트롤러가 직접 관리하도록 설계**되어 있어서, 더 빠르고 단순하게 메타데이터를 처리할 수 있게 된 것


<details>
<summary>메타데이터란?</summary>

- 메타데이터는 <strong>Kafka 클러스터가 어떻게 구성되고 동작하는지를 설명하는 모든 설정 정보</strong>를 의미함.

<h4>주요 항목</h4>

<ol>
  <li><strong>토픽(Topic) 정보</strong><br/>
    - 어떤 토픽이 있는지<br/>
    - 각 토픽이 몇 개의 파티션을 가지고 있는지<br/>
    - 파티션의 리더 브로커는 누구인지
  </li>

  <li><strong>브로커(Broker) 목록</strong><br/>
    - 현재 클러스터에 어떤 브로커가 참여하고 있는지<br/>
    - 각 브로커의 ID, 상태, 주소 등
  </li>

  <li><strong>컨트롤러 상태</strong><br/>
    - 현재 컨트롤러 역할을 맡고 있는 브로커는 누구인지<br/>
    - 다른 컨트롤러 후보들은 누구인지
  </li>

  <li><strong>ACL 및 권한 정보</strong><br/>
    - 사용자나 애플리케이션이 어떤 리소스에 접근할 수 있는지
  </li>

  <li><strong>ISR 정보 (In-Sync Replica)</strong><br/>
    - 각 파티션에 대해, 복제가 정상적으로 이루어지는 브로커는 누구인지
  </li>
</ol>
</details>



## KRaft의 등장과 배경

- KRaft (Kafka Raft)는 Apache Kafka의 분산 시스템을 관리하기 위해 도입된 새로운 메커니즘입니다.
- 이전까지 Kafka는 Apache ZooKeeper를 사용하여 클러스터 메타데이터의 관리와 조정을 담당했습니다.

```
- Zookeeper의 의존성은 Kafka의 확장성과 유지보수에 여러 제약을 가져왔습니다.
- 이를 해결하기 위해 Kafka 자체 내에서 분산 시스템의 상태를 관리하는 방식을 도입하기로 결정됐습니다.
```

### Zookeeper 사용 시 이슈가 되는 부분들

### **1. 성능적인 부분**

- Zookeeper를 사용하면서 여러 가지 불편, 제한적인 상황이 있었겠지만 Kafka에서 새로운 방식의 메커니즘 도입을 결정하는데, 아마도 성능적인 이슈가 가장 큰 영향을 끼치지 않았나 싶습니다.
- 브로커는 모든 모든 토픽과 파티션에 대한 메타데이터를 Zookeeper에서 읽어야 하며, 메타데이터의 업데이터는 Zookeeper에서 동기방식으로 일어나고, 브로커에는 비동기방식으로 전달되었습니다.
    - 이 과정에서 메타데이터의 불일치가 발생할 수도 있으며, 컨트롤러 재시작 시 모든 메타데이터를 Zookeeper로부터 읽어야 하는 것도 부담이었습니다.
    - 특히 토픽과 파티션이 많은 대규모 Kafka 클러스터에서는 오랜 시간이 걸리는 등 병목현상이 발생할 수 있습니다.

### **2. 관리적인 부분**

- Zookeeper와 Kafka는 완전히 서로 다른 애플리케이션으로, 서로 다른 구성 파일, 환경, 서비스 데몬을 가지고 있습니다.
- 결국 관리자는 동시에 서로 다른 애플리케이션을 운영해야 합니다. 동시에 두 가지 애플리케이션을 운영한다는 것은 관리자에게 큰 부담이 됩니다.
    - 예를 들어, Zookeeper의 릴리스 노트 확인, 버전 업그레이드, 구성 파일 변경과 동시에 Kafka의 릴리스 노트 확인, 버전 업그레이드, 구성 파일 변경을 해야 합니다.

### **3. 모니터링 등**

- 모든 서비스는 모니터링이 필수이며, Zookeeper와 Kafka 둘 다 모니터링을 해야 합니다.
- 두 애플리케이션은 서로 다른 애플리케이션이므로, 모니터링을 적용하는 방법과 각 애플리케이션에서 보여주는 주요 메트릭도 다릅니다.
- 또한 모니터링에 필요한 필수 메트릭을 이해하고 모니터링하는 방법까지 완전히 다릅니다.
- 그 외에도 각 애플리케이션에서 빈번하게 발생하는 이슈 또는 장애 상황에 개별적으로 대처해야 하며, 두 애플리케이션 간 통신 이슈라도 발생한다면, 매우 곤혹스러울 것입니다.



## **KRaft의 주요 목적**

- KRaft의 주요 목적은 Kafka의 구조를 단순화하고 확장성을 향상시키기 위함입니다.
- 앞에서 언급한 여러 불편한 부분들이 Zookeeper를 제거함으로써 설치, 구성, 유지보수가 단순화될 뿐만 아니라 Kafka의 성능과 안정성 역시 크게 향상됩니다.
- KRaft를 Kafka와 결합하여 운영 복잡성을 줄이고, 데이터 플랫폼의 충추 역할을 하는 Kafka의 전반적인 신뢰성과 관리 용이성을 개선하는데 기여할 수 있습니다.

> Kafka를 더욱 강력하고 유연한 시스템으로 만들어 대규모 데이터 처리 환경에서의 효율성을 높일 수 있습니다.



## **Zookeeper 모드 vs KRaft 모드**

- Kafka 클러스터를 Zookeeper 모드(Zookeeper와 병행해서 사용하는 모드)와 KRaft 모드(KRaft가 적용된 모드)라고 구분하고, 각 모드에 대한 차이점을 살펴보겠습니다.

### **1. Zookeeper 모드**

- Zookeeper 모드는 Zookeeper 앙상블(Ensemble)과 Kafka 클러스터가 존재하며, Kafka 클러스터 중 하나의 브로커가 컨트롤러 역할을 하게 됩니다.
- 컨트롤러는 파티션의 리더를 선출하는 역할을 하며, 리더 선출 정보를 브로커에게 전파하고 Zookeeper에 리더 정보를 기록하는 역할을 합니다.
- 컨트롤러의 선출 작업은 Zookeeper를 통해 이루어지는데, Zookeeper의 임시노드를 통해 이루어집니다.
- 임시노드에 가장 먼저 연결에 성공한 브로커가 컨트롤러가 되고, 다른 브로커들은 해당 임시노드에 이미 컨트롤러가 있다는 사실을 통해 Kafka 클러스터 내 컨트롤러가 있다는 것을 인식하게 됩니다.

> 한 번에 하나의 컨트롤러만 클러스터에 있도록 보장할 수 있도록 한다! <br>
> Zookeeper가 중간에서 컨트롤러를 뽑아주고, 그 컨트롤러가 Kafka 클러스터를 조율하는 방식!

### **2. KRaft 모드**

-  KRaft 모드에서는 Zookeeper가 사라진 것을 알 수 있습니다.
- KRaft 모드는 Zookeeper와의 의존성을 제거하고, Kafka 단일 애플리케이션 내에서 메타데이터 관리 기능을 수행하는 독립적인 구조가 되는 것입니다.
- Zookeeper 모드에서 1개였던 컨트롤러가 3개로 늘어나고, 이들 중 하나의 컨트롤러가 액티브(그림에서 노란색 컨트롤러) 컨트롤러이면서 리더 역할을 담당합니다.
- **리더 역할을 하는 컨트롤러가 write 하는 역할도 하게 됩니다.**

<details>
<summary> 리더 컨트롤러가 Write도 담당하는 이유</summary>

<h4>기본 개념</h4>

<strong><code>리더 역할을 하는 컨트롤러가 write도 담당한다</code></strong>는 것은 단순해 보이지만,  
<strong>KRaft 모드에서 핵심적인 설계 변화</strong> 중 하나임.

<h4>기존 (Zookeeper 모드) 구조에서는?</h4>

<ul>
  <li>메타데이터 (토픽, 파티션, 리더 정보 등)는 <strong>Zookeeper가 저장함</strong></li>
  <li>Kafka 컨트롤러는 <strong>Zookeeper에 쓰라고 요청만 함</strong></li>
  <li><strong>Kafka 자체는 메타데이터를 직접 쓰지 않음</strong><br/>
    → Zookeeper가 중심에 있었음</li>
</ul>

<h4>KRaft 모드에서는?</h4>

<ul>
  <li><strong>Zookeeper 제거됨</strong></li>
  <li>Kafka 내부에 <strong>KRaft 컨트롤러가 메타데이터 직접 관리</strong></li>
  <li>Kafka가 <strong>자체적으로 메타데이터를 직접 write</strong>해야 함</li>
</ul>

<h4>그런데 왜 <code>리더 컨트롤러만 write</code>할까?</h4>

<ul>
  <li>이유: <strong>일관성과 충돌 방지</strong></li>
</ul>

<blockquote>
여러 컨트롤러가 동시에 메타데이터를 수정하면<br/>
→ 충돌 발생, 무결성 깨짐, 시스템 불안정
</blockquote>

<ul>
  <li><strong>하나의 컨트롤러(리더)만 write 가능</strong></li>
  <li>나머지 컨트롤러는 <strong>read-only 복제본 유지</strong><br/>
    → 장애 발생 시 <strong>빠르게 리더로 승격 가능</strong></li>
</ul>

<h4>핵심 구조</h4>

<ul>
  <li><strong>리더 컨트롤러</strong>: 단 하나만 존재, write 가능</li>
  <li><strong>팔로워 컨트롤러</strong>: 읽기만 함, standby 상태</li>
</ul>

→ 이 구조는 분산 시스템에서 흔히 쓰이는  
<strong>"리더-팔로워 모델"</strong>

<hr/>

<blockquote>
여러 컨트롤러가 동시에 메타데이터를 쓰면 혼란이 생기기 때문에,<br/>
<strong>오직 리더 컨트롤러만 write를 하도록 설계됨</strong><br/>
→ 이는 <strong>분산 시스템의 일관성, 신뢰성, 장애 대응을 위한 핵심 원칙!</strong>
</blockquote>

</details>

- 또한 Zookeeper 노드에서는 메타 데이터 관리를 Zookeeper가 했다면, 이제는 Kafka 내부의 별도 토픽을 이용하여 메타 데이터를 관리합니다.
- 액티브인 컨트롤러가 장애 또는 종료되는 경우, 내부에서는 새로운 합의 알고리즘을 통해 새로운 리더를 선출하게 됩니다.
    - 리더를 선출하는 과정을 간략히 설명드리자면, 후보자들은 적합한 리더를 투표하게 되고 후보자 중 충분한 표를 얻으면, 해당 컨트롤러가 새로운 리더가 됩니다.

>Zookeeper 없이 Kafka가 알아서 리더 뽑고, 메타데이터도 자체적으로 관리함.<br>
>리더 죽으면 컨트롤러들끼리 투표해서 새 리더 뽑는 구조임!



## **KRaft의 성능**

- KRaft의 주요 성능 개선 중 하나는 파티션 리더 선출의 최적화입니다. 앞에서 잠깐 언급했지만, 컨트롤러의 주요 역할은 파티션의 리더를 선출하는 것입니다.
- 소수의 파티션에 대한 리더 선출 작업은 Kafka 또는 Kafka를 사용하는 클라이언트들에게 별다른 영향이 없겠으나, 대량의 파티션에 대한 리더 선출 작업은 다소 시간이 소요되며, 이러한 시간은 대량의 데이터 파이프라인의 역할을 하는 Kafka와 클라이언트들에게 매우 크리티컬한 요소일 수 있습니다.
- 따라서 이러한 지연 시간을 방지하고자 Zookeeper 모드의 경우 Kafka 클러스터 전체의 파티션 리미트는 약 200,000개 정도였으나, 리더 선출 과정을 개선한 KRaft 모드에서는 훨씬 더 많은 파티션 생성이 가능합니다.

<details>
<summary>성능 비교 그래프 해석</summary>

<h4>그래프 구성 설명</h4>

<ul>
  <li><strong>X축</strong><br/>
    - Controlled Shutdown Time (정상 종료 시간)<br/>
    - Recovery Time after Uncontrolled Shutdown (비정상 종료 후 복구 시간)
  </li>
  <li><strong>Y축</strong><br/>
    - 시간 (초 단위, Seconds)
  </li>
  <li><strong>막대 색상</strong><br/>
    - 회색: <strong>Zookeeper 기반 Kafka</strong><br/>
    - 남색: <strong>KRaft (Quorum Controller) 기반 Kafka</strong>
  </li>
</ul>

<h4>결과 해석</h4>

<ol>
  <li>
    <strong>정상 종료 시간 (Controlled Shutdown Time)</strong><br/>
    - Zookeeper 기반: <strong>약 180초 (3분)</strong><br/>
    - KRaft 기반: <strong>약 20초 미만</strong><br/>
    → <strong>KRaft가 훨씬 빠르게 종료됨</strong>
  </li>

  <li>
    <strong>비정상 종료 후 복구 시간 (Recovery Time after Uncontrolled Shutdown)</strong><br/>
    - Zookeeper 기반: <strong>약 500초 (8분 이상)</strong><br/>
    - KRaft 기반: <strong>약 40초</strong><br/>
    → <strong>KRaft가 훨씬 빠르게 복구됨</strong>
  </li>
</ol>

<blockquote>
<strong>KRaft 모드(Kafka Quorum Controller)</strong>는<br/>
<strong>Zookeeper보다 훨씬 빠르게 종료 및 복구 가능</strong><br/>
특히 <strong>대규모 파티션(200만 개)</strong> 상황에서도 <strong>성능이 매우 안정적</strong>임
</blockquote>
</details>

- 컨플루언트에서 공개한 KRaft 모드와 Zookeeper 모드 간의 속도를 비교한 [그림](https://docs.confluent.io/platform/current/kafka-metadata/kraft.html)을 살펴보면, 복구 소요시간에서 엄청난 차이를 나타내고 있음을 알 수 있습니다.

### 속도차이가 나는 이유는?

- **컨트롤러가 메타데이터를 메모리에 바로 가지고 있음.**
  
    → Kafka는 브로커와 토픽 등의 정보를 메타데이터로 관리하는데, KRaft에서는 이 정보들이 **메모리에 항상 저장되어 있어** 빠르게 접근할 수 있습니다.
    
    예전에는 Zookeeper에 요청을 보내서 가져와야 했는데, 이 과정을 생략할 수 있게 된거죠.
    
- **Zookeeper를 아예 안 씀**
  
    → Zookeeper가 빠지면서 Kafka 내부에서 직접 메타데이터를 관리하게 되었습니다. 이 덕분에 **복잡한 외부 동기화 작업이 줄고**, 처리 속도가 빨라집니다.
    
- **컨트롤러 장애가 나도 빠르게 복구됨**
  
    → 예전에는 컨트롤러가 죽으면 새로운 컨트롤러가 **Zookeeper에서 데이터를 읽어오느라 시간이 걸렸습니다.** 그런데 KRaft에서는 모든 컨트롤러 후보가 이미 메모리에 최신 정보를 가지고 있어서, **리더 선출이 훨씬 빠르게 이루어집니다.**
    

>KRaft 모드는 메타데이터를 직접 메모리에 가지고 있고, Zookeeper 없이 동작하기 때문에 더 빠르며, 장애 발생 시 복구도 빨라졌습니다.