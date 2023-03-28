---
key: jekyll-text-theme
title: 'AWS IAM과 AWS CLI'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---


## AWS IAM 소개

* **IAM = Identify and Access Management** (Global 서비스에 해당)
* 인증 및 권한 관리 서비스
* IAM을 사용하면 AWS 계정 내에서 사용자, 그룹 및 역할을 생성하고 관리할 수 있습니다.
  * 하나의 사용자는 조직 내의 한 사람!
  * Faker, Oner, Zeus는 같이 일하는 개발자 -> Developers 라는 그룹을 생성한 후 Faker, Oner, Zeus를 배치할 수 있습니다.
  * 그룹에는 사용자만 배치할 수 있고, 다른 그룹은 포함시킬 수 없습니다.
  * AWS 계정을 사용하도록 허용하기 위해서 사용자와 그룹을 생성합니다.
  * AWS 허용을 위해서 권한을 부여해야 하며, 권한 부여를 위해 사용자 또는 그룹에게 정책, 또는 IAM 정책이라고 불리는 JSON 문서를 지정할 수 있습니다.
  * AWS 최소 권한의 원칙 적용: 사용자가 필요로 하는 것 이상의 권한을 주지 않는다. (새로운 사용자가 너무 많은 서비스를 실행하여 큰 비용이 발생하거나, 보안 문제를 야기할 수 있기 때문)


## IAM 사용자 및 그룹 실습

###  User(사용자)와 Group(그룹)

* 사용자는 AWS 리소스에 대한 엑세스 권한을 가집니다.

* IAM에서 생성한 사용자는 자격 증명을 사용하여 로그인할 수 있습니다.

* 사용자를 생성해야하는 이유

  * 루트 사용자는 계정에 대한 모든 권한을 가지고 있기 때문에 무엇이든 할 수 있고, 위험한 계정이 될 수 있습니다. (보안상의 이유로 별도의 관리자 계정을 만드는 게 좋습니다.)

### **Users 설정 방법**

##### ① AWS 콘솔 로그인

##### ② IAM 서비스로 접속

##### ③ Access Management (IAM 대시보드)

##### ④ Users

##### ⑤ Add users

  ​	<img src="https://user-images.githubusercontent.com/113915835/228215034-c66ca5e2-0d3e-46e0-b82a-caf8ffbe7903.png" width="80%">

  ​    <img src="https://user-images.githubusercontent.com/113915835/228220781-088ba3e4-b933-40b1-9e2f-166a63bd2fe3.png" width="80%">

  * 자격 증명 방식을 선택할 때는 비밀번호 방식 자격 증명을 활성화합니다. 자동생성을 할 수도 있고, 직접 입력할 수도 있습니다.

##### ⑥ Create Group

  ​	<img src = "https://user-images.githubusercontent.com/113915835/228222067-6a7f1e7c-3e04-4e63-a36a-ff0efac4a18e.png" width="80%">

  ​    <img src = "https://user-images.githubusercontent.com/113915835/228222412-2b28b9bb-7ec7-4a7c-9f96-01883d001c74.png" width = "80%">

  * admin 그룹에 배치된 사용자는 그룹에 부여된 권한을 받을 수 있습니다.
  * 그룹의 권한은 정책을 통해 정의됩니다.
  * 모든 사용자를 그룹에 속하게 하고 관리하는 것이 좋습니다.

##### ⑦ Create User

  ​	<img src ="https://user-images.githubusercontent.com/113915835/228224726-e51f26e8-4124-4b03-a428-365cd0288732.png" width ="80%">

  > 리소스 검색, 비용 분석, 보안 강화 등 다양한 용도로 태그 사용 가능 (선택 사항)

##### ⑧ IAM으로 로그인 하기

  ​	<img src ="https://user-images.githubusercontent.com/113915835/228241855-9c6b16c6-bf4c-4b79-b7b7-d3bae16d3a32.png" width = "80%">

  * IAM 사용자로 로그인 하기 전에 **IAM 서비스로 접속 > IAM  대시보드 > 계정 별칭 > [생성]** 버튼으로 계정 별칭 생성하면 IAM 사용자 로그인 시 편리합니다.


## IAM Policies (정책)





  

  
