---
key: jekyll-text-theme
title: 'Kafka의 새로운 협의 프로토콜인 KRaft'
excerpt: 'KRaft에 대하여😎'
tags: [Kafka]
---


# Kafka의 새로운 협의 프로토콜인 KRaft

## 첫 번째 스터디 목표

- KRaft의 등장과 Zookeeper를 사용하면서의 몇 가지 단점들에 대해 살펴보고, Zookeeper 모드와 KRaft 모드의 주요 차이점을 살펴본다.
    - Kafka를 사용하면서 초기에는 최신 버전의 릴리스를 추구했지만, Kafka가 점점 데이터 파이프라인의 중심이 되면서 보다 보수적으로 접근할 필요성이 있음.
        - Kafka의 중요도가 높아졌기 때문에, 버전 업그레이드나 설정 변경, 기능 실험 등을 쉽게 할 수 없게 되었다는 뜻 (즉, 안정성이 최우선이 되므로, 변화에 대해 신중하게 접근해야 한다는 얘기)
    - 이제는 KRaft에 대한 준비와 Zookeeper 모드로 운영 중인 Kafka를 마이그레이션 하는 방법 등에 대해서도 심도 있는 리서치가 필요함.

## KRaft를 사용하면?

- Apache Kafka의 새로운 메커니즘인 KRaft를 사용하면 메타데이터를 효율적으로 관리할 수 있습니다.
- KRaft는 Zookeeper의 의존성을 제거하고, Kafka 클러스터 내 컨트롤러가 선출된 후 메타데이터를 직접 관리합니다. 이로 인해 성능과 안정성이 향상되며, 유지보수가 단순화되고, 병목현상이 줄어듭니다.
    - KRaft는 이런 중요한 정보를 **Zookeeper 대신 Kafka 내부 컨트롤러가 직접 관리하도록 설계**되어 있어서, 더 빠르고 단순하게 메타데이터를 처리할 수 있게 된 것

<details>
<summary>메타데이터란?</summary>

- 메타데이터는 <strong>Kafka 클러스터가 어떻게 구성되고 동작하는지를 설명하는 모든 설정 정보</strong>를 의미함.

<h4>주요 항목</h4>

	1. <strong>토픽(Topic) 정보</strong>  
   - 어떤 토픽이 있는지  
   - 각 토픽이 몇 개의 파티션을 가지고 있는지  
   - 파티션의 리더 브로커는 누구인지

	2. <strong>브로커(Broker) 목록</strong>  
   - 현재 클러스터에 어떤 브로커가 참여하고 있는지  
   - 각 브로커의 ID, 상태, 주소 등

	3. <strong>컨트롤러 상태</strong>  
   - 현재 컨트롤러 역할을 맡고 있는 브로커는 누구인지  
   - 다른 컨트롤러 후보들은 누구인지

	4. <strong>ACL 및 권한 정보</strong>  
   - 사용자나 애플리케이션이 어떤 리소스에 접근할 수 있는지

	5. <strong>ISR 정보 (In-Sync Replica)</strong>  
   - 각 파티션에 대해, 복제가 정상적으로 이루어지는 브로커는 누구인지
</details>



## **KRaft의 등장과 배경**

- KRaft (Kafka Raft)는 Apache Kafka의 분산 시스템을 관리하기 위해 도입된 새로운 메커니즘입니다.
- 이전까지 Kafka는 Apache ZooKeeper를 사용하여 클러스터 메타데이터의 관리와 조정을 담당했습니다.

```
Zookeeper의 의존성은 Kafka의 확장성과 유지보수에 여러 제약을 가져왔고, 이를 해결하기 위해 Kafka 자체 내에서 분산 시스템의 상태를 관리하는 방식을 도입하기로 결정됐습니다.
```

### 😳 Zookeeper 사용 시 이슈가 되는 부분들

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

👉 Kafka를 더욱 강력하고 유연한 시스템으로 만들어 대규모 데이터 처리 환경에서의 효율성을 높일 수 있습니다.

## **Zookeeper 모드 vs KRaft 모드**

- Kafka 클러스터를 Zookeeper 모드(Zookeeper와 병행해서 사용하는 모드)와 KRaft 모드(KRaft가 적용된 모드)라고 구분하고, 각 모드에 대한 차이점을 살펴보겠습니다.

### **1. Zookeeper 모드**

![79bf6fe27f866c77c6469acb40111ea0c513f37b17d68716d39e516971ed7319.png](Kafka%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%A2%E1%84%85%E1%85%A9%E1%84%8B%E1%85%AE%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%8B%E1%85%B4%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%90%E1%85%A9%E1%84%8F%E1%85%A9%E1%86%AF%E1%84%8B%E1%85%B5%E1%86%AB%20KRaft%2020a22d577d0e80ae9081e970abdc1e1e/79bf6fe27f866c77c6469acb40111ea0c513f37b17d68716d39e516971ed7319.png)

- Zookeeper 모드는 Zookeeper 앙상블(Ensemble)과 Kafka 클러스터가 존재하며, Kafka 클러스터 중 하나의 브로커가 컨트롤러 역할을 하게 됩니다.
- 컨트롤러는 파티션의 리더를 선출하는 역할을 하며, 리더 선출 정보를 브로커에게 전파하고 Zookeeper에 리더 정보를 기록하는 역할을 합니다.
- 컨트롤러의 선출 작업은 Zookeeper를 통해 이루어지는데, Zookeeper의 임시노드를 통해 이루어집니다.
- 임시노드에 가장 먼저 연결에 성공한 브로커가 컨트롤러가 되고, 다른 브로커들은 해당 임시노드에 이미 컨트롤러가 있다는 사실을 통해 Kafka 클러스터 내 컨트롤러가 있다는 것을 인식하게 됩니다.

👉 한 번에 하나의 컨트롤러만 클러스터에 있도록 보장할 수 있도록 한다! 

👉 Zookeeper가 중간에서 컨트롤러를 뽑아주고, 그 컨트롤러가 Kafka 클러스터를 조율하는 방식!

### **2. KRaft 모드**

![크래프트.png](Kafka%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%A2%E1%84%85%E1%85%A9%E1%84%8B%E1%85%AE%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%8B%E1%85%B4%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%90%E1%85%A9%E1%84%8F%E1%85%A9%E1%86%AF%E1%84%8B%E1%85%B5%E1%86%AB%20KRaft%2020a22d577d0e80ae9081e970abdc1e1e/%E1%84%8F%E1%85%B3%E1%84%85%E1%85%A2%E1%84%91%E1%85%B3%E1%84%90%E1%85%B3.png)

- 그림에서 보는 바와 같이 KRaft 모드에서는 Zookeeper가 사라진 것을 알 수 있습니다.
- KRaft 모드는 Zookeeper와의 의존성을 제거하고, Kafka 단일 애플리케이션 내에서 메타데이터 관리 기능을 수행하는 독립적인 구조가 되는 것입니다.
- Zookeeper 모드에서 1개였던 컨트롤러가 3개로 늘어나고, 이들 중 하나의 컨트롤러가 액티브(그림에서 노란색 컨트롤러) 컨트롤러이면서 리더 역할을 담당합니다.
- **리더 역할을 하는 컨트롤러가 write 하는 역할도 하게 됩니다.**

<details>
<summary>📝 리더 컨트롤러가 Write도 담당하는 이유</summary>

####  기본 개념

**`리더 역할을 하는 컨트롤러가 write도 담당한다`**는 것은  단순해 보이지만, **KRaft 모드에서 핵심적인 설계 변화** 중 하나임.


#### 기존(Zookeeper 모드) 구조에서는?

- 메타데이터(토픽, 파티션, 리더 정보 등)는 **Zookeeper가 저장함**
- Kafka 컨트롤러는 **Zookeeper에 쓰라고 요청만 함**
- 즉, **Kafka 자체는 메타데이터를 직접 쓰지 않음**  
  → Zookeeper가 중심에 있었음


#### KRaft 모드에서는?

- **Zookeeper 제거됨**
- Kafka 내부에 **KRaft 컨트롤러가 메타데이터 직접 관리함.**
- Kafka가 **자체적으로 메타데이터를 직접 write**해야 함.


#### 그런데 왜 `리더 컨트롤러만 write`할까?

- 이유: **일관성과 충돌 방지**

> 여러 컨트롤러가 동시에 메타데이터를 수정하면  
> → 충돌 발생, 무결성 깨짐, 시스템 불안정

- 그래서 **하나의 컨트롤러(리더)만 write 가능**
- 나머지 컨트롤러는 **read-only 복제본 유지**  
  → 장애 발생 시 **빠르게 리더로 승격 가능**


#### 핵심 구조

- **리더 컨트롤러**: 단 하나만 존재, write 가능  
- **팔로워 컨트롤러**: 읽기만 함, standby 상태  
  → 이 구조는 분산 시스템에서 흔히 쓰이는  
  **"리더-팔로워 모델"**

---

> 여러 컨트롤러가 동시에 메타데이터를 쓰면 혼란이 생기기 때문에,  
> **오직 리더 컨트롤러만 write를 하도록 설계됨**  
> → 이는 **분산 시스템의 일관성, 신뢰성, 장애 대응을 위한 핵심 원칙!**
</details>


- 또한 Zookeeper 노드에서는 메타 데이터 관리를 Zookeeper가 했다면, 이제는 Kafka 내부의 별도 토픽을 이용하여 메타 데이터를 관리합니다.
- 액티브인 컨트롤러가 장애 또는 종료되는 경우, 내부에서는 새로운 합의 알고리즘을 통해 새로운 리더를 선출하게 됩니다.
    - 리더를 선출하는 과정을 간략히 설명드리자면, 후보자들은 적합한 리더를 투표하게 되고 후보자 중 충분한 표를 얻으면, 해당 컨트롤러가 새로운 리더가 됩니다.

👉 Zookeeper 없이 Kafka가 알아서 리더 뽑고, 메타데이터도 자체적으로 관리함.

👉 리더 죽으면 컨트롤러들끼리 투표해서 새 리더 뽑는 구조임!

## **KRaft의 성능**

- KRaft의 주요 성능 개선 중 하나는 파티션 리더 선출의 최적화입니다. 앞에서 잠깐 언급했지만, 컨트롤러의 주요 역할은 파티션의 리더를 선출하는 것입니다.
- 소수의 파티션에 대한 리더 선출 작업은 Kafka 또는 Kafka를 사용하는 클라이언트들에게 별다른 영향이 없겠으나, 대량의 파티션에 대한 리더 선출 작업은 다소 시간이 소요되며, 이러한 시간은 대량의 데이터 파이프라인의 역할을 하는 Kafka와 클라이언트들에게 매우 크리티컬한 요소일 수 있습니다.
- 따라서 이러한 지연 시간을 방지하고자 Zookeeper 모드의 경우 Kafka 클러스터 전체의 파티션 리미트는 약 200,000개 정도였으나, 리더 선출 과정을 개선한 KRaft 모드에서는 훨씬 더 많은 파티션 생성이 가능합니다.

**⬇️ Apache Kafka에서 200만 개 파티션을 사용하는 경우 Zookeeper 기반 구성 vs KRaft(Quorum Controller) 기반 구성**에서 **종료 및 복구 시간**을 비교한 것

![성능비교.png](Kafka%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%A2%E1%84%85%E1%85%A9%E1%84%8B%E1%85%AE%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%8B%E1%85%B4%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%90%E1%85%A9%E1%84%8F%E1%85%A9%E1%86%AF%E1%84%8B%E1%85%B5%E1%86%AB%20KRaft%2020a22d577d0e80ae9081e970abdc1e1e/%E1%84%89%E1%85%A5%E1%86%BC%E1%84%82%E1%85%B3%E1%86%BC%E1%84%87%E1%85%B5%E1%84%80%E1%85%AD.png)

<details>
<summary> 성능 비교 그래프 해석</summary>

#### 그래프 구성 설명

- **X축**
  - Controlled Shutdown Time (정상 종료 시간)
  - Recovery Time after Uncontrolled Shutdown (비정상 종료 후 복구 시간)

- **Y축**
  - 시간 (초 단위, Seconds)

- **막대 색상**
  - 회색: **Zookeeper 기반 Kafka**
  - 남색: **KRaft(Quorum Controller) 기반 Kafka**

#### 결과 해석

1. **정상 종료 시간 (Controlled Shutdown Time)**
   - Zookeeper 기반: **약 180초 (3분)**  
   - KRaft 기반: **약 20초 미만**  
   → **KRaft가 훨씬 빠르게 종료됨**

2. **비정상 종료 후 복구 시간 (Recovery Time after Uncontrolled Shutdown)**
   - Zookeeper 기반: **약 500초 (8분 이상)**  
   - KRaft 기반: **약 40초**  
   → **KRaft가 훨씬 빠르게 복구됨**


> **KRaft 모드(Kafka Quorum Controller)**는  
> **Zookeeper보다 훨씬 빠르게 종료 및 복구 가능**  
> 특히 **대규모 파티션(200만 개)** 상황에서도 **성능이 매우 안정적**임
</details>

- 컨플루언트에서 공개한 KRaft 모드와 Zookeeper 모드 간의 속도를 비교한 [그림](https://docs.confluent.io/platform/current/kafka-metadata/kraft.html)을 살펴보면, 복구 소요시간에서 엄청난 차이를 나타내고 있음을 알 수 있습니다.

### 😎 속도차이가 나는 이유는?

- **컨트롤러가 메타데이터를 메모리에 바로 가지고 있음.**
  
    → Kafka는 브로커와 토픽 등의 정보를 메타데이터로 관리하는데, KRaft에서는 이 정보들이 **메모리에 항상 저장되어 있어** 빠르게 접근할 수 있습니다.
    
    예전에는 Zookeeper에 요청을 보내서 가져와야 했는데, 이 과정을 생략할 수 있게 된거죠.
    
- **Zookeeper를 아예 안 씀**
  
    → Zookeeper가 빠지면서 Kafka 내부에서 직접 메타데이터를 관리하게 되었습니다. 이 덕분에 **복잡한 외부 동기화 작업이 줄고**, 처리 속도가 빨라집니다.
    
- **컨트롤러 장애가 나도 빠르게 복구됨**
  
    → 예전에는 컨트롤러가 죽으면 새로운 컨트롤러가 **Zookeeper에서 데이터를 읽어오느라 시간이 걸렸습니다.** 그런데 KRaft에서는 모든 컨트롤러 후보가 이미 메모리에 최신 정보를 가지고 있어서, **리더 선출이 훨씬 빠르게 이루어집니다.**
    

👉 KRaft 모드는 메타데이터를 직접 메모리에 가지고 있고, Zookeeper 없이 동작하기 때문에 더 빠르며, 장애 발생 시 복구도 빨라졌습니다.


## 두 번째 스터디 목표
- Zookeeper를 사용하는 Kafka 모드에서 새롭게 도입된 KRaft 모드로 전환하는 방법에 대해 설명한다.

```
KRaft 모드를 사용하면 물리적인 서버의 관리 효율성을 높이고 리소스를 절약할 수 있습니다. 그러나 안정성 측면에서는 여전히 Kafka 브로커와는 별도로 KRaft 컨트롤러 노드를 운영하는 것이 권장됩니다. Apache Kafka는 2021년 2.8 버전에서 KRaft 모드를 처음 도입했으며, 2024년에 발표된 Kafka 4.0부터는 Zookeeper 지원이 완전히 중단되고 KRaft 모드만을 지원하게 되었습니다.
```

## **KRaft의 구성**

- 전통적인 Zookeeper 모드를 사용하면서 많은 사용자들이 느꼈던 불편함 중 하나는 바로 Zookeeper와 Kafka 서버를 별도로 운영해야 한다는 점이었습니다.
    - 이는 단순히 별도의 애플리케이션 운영 관리를 넘어서, 추가로 별도의 물리적 서버 자원의 할당까지 포함하고 있습니다.

### 💁🏻 가장 많은 FAQ 중에 하나는 Zookeeper 물리 서버의 할당과 관련된 주제로, Zookeeper와 Kafka를 동일한 서버에서 실행해도 되는지에 관한 것!

- 사실 Zookeeper는 Kafka를 관리하는 역할을 하므로, 이상적으로는 Kafka와 분리된 별도의 서버에서 운영하는 것을 권장합니다.
- 하지만 이는 강제성을 요구하는 것도 아니고, 서버의 리소스 제약이 있는 경우 Zookeeper와 Kafka를 동일한 서버에서 실행할 수도 있습니다.
- KRaft의 등장 이후 Kafka 사용자들이 환영한 변화중 하나는 Zookeeper의 의존성 제거입니다. 이는 애플리케이션의 관리 단순화뿐만 아니라, 물리적 서버의 리소스 절감도 가능하다고 생각했던 것 같습니다.
- 실제로 KRaft 공개 이후, Zookeeper 서버 없이 Kafka를 운영할 수 있는지에 대한 논의가 많았습니다. 하지만 KRaft모드에서도 여전히 브로커와 KRaft를 분리된 서버에 배치하는 것을 강력히 추천하고 있습니다.
- 따라서 Zookeeper 서버의 리소스 절약을 주된 목적으로 KRaft로 전환을 고려하는 것은 추천하지 않습니다.
- 그럼에도 불구하고 서버 자원이 한정적인 경우라면 브로커와 KRaft를 동일한 서버에 배치하는 것을 고려할 수 있습니다.

### 1. KRaft 모드 - KRaft 별도 구성

→ Zookeeper 없는 KRaft 방식으로 Kafka를 운영하되, 컨트롤러(KRaft 서버)를 브로커와 분리해서 따로 배치한 구성

![별도구성.png](Kafka%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%A2%E1%84%85%E1%85%A9%E1%84%8B%E1%85%AE%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%8B%E1%85%B4%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%90%E1%85%A9%E1%84%8F%E1%85%A9%E1%86%AF%E1%84%8B%E1%85%B5%E1%86%AB%20KRaft%2020a22d577d0e80ae9081e970abdc1e1e/%E1%84%87%E1%85%A7%E1%86%AF%E1%84%83%E1%85%A9%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC.png)

<details>
<summary>아키텍처 해석</summary>

**👉 KRaft 컨트롤러 노드와 브로커 노드를 서로 다른 서버에 배치**한 구조


### Kafka 클러스터 전체 (빨간색 테두리)

- Kafka 클러스터 전체를 나타냄.  
- 내부에 **KRaft 컨트롤러들**과 **브로커들**이 포함되어 있음.

### 핵심 구성 요소

#### 1. KRaft 영역 (위쪽 상단)

- 컨트롤러 노드가 3개 있음  
  `컨트롤러`, `컨트롤러`, `컨트롤러(노란색)`  
  → 이 중 **노란색 컨트롤러는 "리더" 역할을 맡고 있는 액티브 컨트롤러!**

- 이 컨트롤러들은 Kafka 클러스터의  
  **메타데이터 관리와 리더 선출 등을 담당**함.

#### 2. 브로커 영역 (하단)

- 브로커1 ~ 브로커5:  
  실제 데이터를 처리하고,  
  프로듀서/컨슈머와 통신하는 Kafka 서버들
</details>


- 별도 구성이란? 컨트롤러(KRaft 서버)와 브로커(Kafka 서버)를 서로 다른 서버에서 따로 운영하는 방식입니다.
    - 컨트롤러 서버 → 클러스터의 리더 선출, 메타데이터 관리 등
    - 브로커 서버 → 실제 데이터 처리 (프로듀서/컨슈머가 사용하는 메시지 처리)
- 공식 문서에 따르면, KRaft 구성의 권장 방식은 브로커와 별개로 KRaft 서버를 할당하여 운영하는 것입니다.
    - 이 구성은 브로커와 컨트롤러를 분리하여 높은 가용성과 안정성을 확보하는데 중점을 둡니다.
- 이렇게 분리된 구성은 시스템의 장애 포인트를 최소화하고, 하나의 구성 요소에서 문제가 발생했을 시 전체 시스템에 미치는 영향을 줄일 수 있고, 클러스터의 관리 효율성과 시스템의 안정성을 더욱 강화합니다.
- 특히 대규모 Kafka 클러스터를 운영하거나, 엄격한 SLA를 충족해야 하는 비즈니스 환경에서는 반드시 브로커와 별개로 KRaft 서버를 할당하여 사용하기를 추천합니다.

### 2. KRaft 모드 - KRaft 통합 구성

→ Zookeeper 없이 KRaft로 운영하면서, 컨트롤러(KRaft)와 브로커를 한 서버에서 같이 실행하는 구성

![통합구성.png](Kafka%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%A2%E1%84%85%E1%85%A9%E1%84%8B%E1%85%AE%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%8B%E1%85%B4%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%90%E1%85%A9%E1%84%8F%E1%85%A9%E1%86%AF%E1%84%8B%E1%85%B5%E1%86%AB%20KRaft%2020a22d577d0e80ae9081e970abdc1e1e/%E1%84%90%E1%85%A9%E1%86%BC%E1%84%92%E1%85%A1%E1%86%B8%E1%84%80%E1%85%AE%E1%84%89%E1%85%A5%E1%86%BC.png)

<details>
<summary> 아키텍처 해석</summary>

👉 **컨트롤러와 브로커를 같은 서버에 함께 두는 방식**


### 전체 테두리: Kafka 클러스터 전체

- 클러스터 내부에는 컨트롤러(KRaft)와 브로커가 존재함  
- 여기서 컨트롤러와 브로커가 **같은 서버 안에서 함께 실행**됨


### 주요 구성 요소

#### 1. 브로커 + 컨트롤러가 함께 있는 노드들

- 브로커1 + 컨트롤러  
- 브로커3 + 컨트롤러 (노란색 = **액티브 리더 컨트롤러**)  
- 브로커4 + 컨트롤러  

→ 이 Kafka 노드들은  
**데이터도 처리하고(Kafka 브로커)**  
동시에 **클러스터 메타데이터도 관리(KRaft 컨트롤러)** 함.

#### 2. 브로커만 있는 노드들

- 브로커2, 브로커5는 컨트롤러 역할 없음  
- 이 노드들은 순수하게 Kafka 데이터 처리만 수행
</details>

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

- 특정 프로세스나 컨테이너가 사용하는 **CPU 코어를 다른 작업과 공유하지 않고 독점적으로 사용하는 것**

### 일반적인 상황

- 보통 서버에서는 여러 프로세스(예: Kafka, 다른 애플리케이션)가  
  **같은 CPU 코어를 함께 씀** → 이를 **Shared CPU**라고 함  
  → **성능이 들쑥날쑥**하고 예측하기 어려움

### Dedicated CPU란?

- Kafka 같은 중요한 서비스에 **특정 CPU 코어를 하나 또는 여러 개 전용으로 할당**  
- **다른 작업이 그 코어를 사용하지 못하도록 막음**  
→ 결과적으로 **성능이 안정적이고 예측 가능함**


### 언제 쓰냐?

- Kafka, Zookeeper, KRaft 같이 **지연에 민감하고 고성능이 필요한 서비스**에 적합  
- **리소스 경쟁 없이 안정적인 처리량 확보** 가능  
→ 그래서 **Dedicated CPU 설정을 권장**함

---

> **Dedicated CPU = 해당 CPU 코어를 오직 Kafka만 쓰도록 고정하는 것**  
> → 다른 서비스랑 나눠 쓰지 않으므로 **성능 안정성** 높아짐
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

☑️ KRaft의 마이그레이션 상세 매뉴얼 형식보다는 전반적인 마이그레이션 방법에 대해 설명하겠습니다.

☑️ KRaft의 마이그레이션 방법은 다음과 같은 순서로 진행합니다.


### 현재 Zookeeper 버전 모드 -> 1차 Kafka 버전 업그레이드(브리지 모드) -> 2차 KRaft 모드 마이그레이션

- Zookeeper 모드로 Kafka를 사용하면서 KRaft 모드로의 전환은 단순한 업그레이드가 아닙니다.
- 내부적으로 사용하는 API와 구성의 근본적인 변화로 인해 Zookeeper 모드에서 KRaft 모드로 다이렉트 마이그레이션은 불가능합니다.
- 이러한 복잡한 이슈를 해결하기 위해 Kafka는 브리지 모드를 도입하였습니다.
- 브리지 모드는 Zookeeper 모드와 KRaft 모드를 동시에 지원하며 두 모드 사이의 다리 역할을 하게 됩니다.
- Zookeeper 마이그레이션 GA가 지원되는 Kafka 3.6 버전이 Zookeeper에서 KRaft로의 전환을 고려할 때 브리지 모드로 고려해야 할 버전으로 추천합니다.

<details>
<summary> 브리지 모드(Bridge Mode)란?</summary>

**Zookeeper 모드 → KRaft 모드로 전환**할 때  **중간 단계에서 둘 다 동시에 사용하는 혼합 모드**

### 왜 필요한가?

- Kafka의 기존 방식(Zookeeper)과 새로운 방식(KRaft)은 **완전히 다름**  
  → API, 메타데이터 구조, 리더 선출 방식까지 다 바뀜  
  → 그래서 **단순 업그레이드로는 전환 불가**

- Kafka가 이 복잡함을 해결하려고 만든 전환용 구조가 바로 **브리지 모드**  
  → Zookeeper와 KRaft를 **동시에 돌릴 수 있게 해줌**

### 브리지 모드에서 할 수 있는 일

- Zookeeper가 기존 클러스터 메타데이터를 계속 관리함  
- KRaft는 일부 기능을 학습하거나 병행 실행  
- 사용자는 이 환경에서 점진적으로 Zookeeper 의존도를 줄이며  
  **KRaft 기반 클러스터로 안전하게 전환 가능**

### 브리지 모드를 쓰는 이유

- Zookeeper에서 KRaft로 **한 번에 이동하는 건 위험하고 어렵기 때문**  
- **운영 중단 없이, 안정적으로 이전**하기 위한 방법  
- Kafka 3.6부터 **브리지 모드 지원(GA)**  
  → Kafka 3.6이 Zookeeper to KRaft 전환에 가장 적합한 버전으로 추천됨

> **브리지 모드 = Zookeeper와 KRaft를 동시에 사용해, 점진적으로 KRaft로 이전할 수 있게 해주는 Kafka의 전환용 모드**
</details>


☑️ 오늘 스터디에서는 현재 사용 중인 Zookeeper 모드를 사용하고 있으며 Kafka 버전이 3.0이라고 가정하고 설명하겠습니다.


### 1단계: Kafka 3.0 버전 + Zookeeper 모드 -> Kafka 3.6 버전 + Zookeeper 모드


- 현재 사용하고 있는 Kafka 버전에서 Zookeeper 마이그레이션 GA가 지원되는 Kafka 3.6 버전으로 먼저 업그레이드를 해야 합니다.
- 버전 업그레이드 시 현재 Kafka의 상태 파악, 사전 준비, 업그레이드 계획 수립, 테스트 및 검증, 실제 업그레이드 실행, 사후 점검 등의 상세한 작업등을 진행해야 합니다.
- 업그레이드 작업 시 업그레이드 전략은 롤링 업그레이드 방식으로 진행할 수 있으며, 클라이언트의 타임아웃이 일부 발생하기는 하지만 무중단으로 업그레이드가 가능합니다.
- 참고 도서:  <[Kafka, 데이터 플랫폼의 최강자](https://www.yes24.com/Product/Goods/59789254)>, <[실전 Kafka 개발부터 운영까지](https://www.yes24.com/Product/Goods/104410708)>


### 2단계: Kafka 3.6 버전 + Zookeeper 모드 -> Kafka 3.6 버전 + KRaft 모드

- 버전 업그레이드를 성공했다면, 현재 실행 중인 Kafka의 버전은 3.6입니다.
- 이제 KRaft 모드로 마이그레이션을 하기 위한 사전 준비가 완료되었으며, 실제 마이그레이션에 대한 상세 매뉴얼은 컨플루언트 [공식 문서](https://docs.confluent.io/platform/current/installation/migrate-zk-kraft.html)를 참고할 수 있습니다.
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

### **참고(KRaft의 발음에 대해)**

발음의 통일은 효과적인 커뮤니케이션을 용이하게 하고 기술적 대화의 명확성에 기여한다고 생각합니다.

결론부터 말씀드리자면, KRaft의 발음은 '크래프트'가 맞습니다!

또한 [해외 사이트](https://www.baeldung.com/kafka-shift-from-zookeeper-to-kraft)와 컨플루언트(Confluent)의 공식 [YouTube 채널](https://www.youtube.com/watch?v=EUwwNnVyc4c)에서도 '크래프트'라는 발음이 권장되고 있는 것을 확인할 수 있습니다

> Kafka, in its architecture, has recently shifted from ZooKeeper to a quorum-based controller that uses a new consensus protocol called Kafka Raft, shortened as Kraft (pronounced “craft”).