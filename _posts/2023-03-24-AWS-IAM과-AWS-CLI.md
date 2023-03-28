---
key: jekyll-text-theme
title: 'AWS IAM과 AWS CLI'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---


## AWS IAM 소개

* **IAM = Identify and Access Management** (Global 서비스에 해당)
* AWS가 제공하는 인증 및 권한 관리 서비스
* IAM을 사용하면 AWS 계정 내에서 사용자, 그룹 및 역할을 생성하고 관리할 수 있습니다.
  * 하나의 사용자는 조직 내의 한 사람!
  * Alice, Bob, Charles는 같이 일하는 개발자 -> Developers 라는 그룹을 생성한 후 Alice, Bob, Charles를 그룹에 배치할 수 있습니다.
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

###  IAM Policies Inheritance

<img src ="https://user-images.githubusercontent.com/113915835/228271582-731d7232-4ceb-4f46-8219-815937b6da7a.png" width="80%">



### IAM Policies Structure

<img src = "https://user-images.githubusercontent.com/113915835/228273195-ff7aca87-d654-4d6e-a154-9eaf6c9319ea.png" width="80%">

* IAM 정책은 AWS 리소스에 대한 액세스를 제어하는 데 사용
* IAM  정책은 JSON 형식의 문서로 작성
  * **Version**: IAM 정책 구조 요소는 버전 숫자를 포함함. 2012-10-17이 일반적인 정책 언어 버전
  * **ID**: 정책을 식별하는 ID (선택사항)
  * **Statement**
    * **Sid**: Statement ID로 문장의 식별자 (선택사항)
    * **Effect**: Statement가 특정 API에 접근하는 것을 허용할 지 거부할 지에 관한 내용 (Allow / Deny) :star:
    * **Principal**: 특정 정책이 적용될 사용자, 계정, 역할로 구성 :star:
    * **Action**: effect에 기반해 허용 및 거부되는 API 호출의 목록 :star:
    * **Resource**: 적용될 Action의 리소스 목록 :star:
    * **Condition**: Statement가 언제 적용될 지 결정

### MFA

* **MFA = Multi-Factor Authentication**

* 비밀번호와 보안장치를 함께 사용하는 것

* 생성한 그룹과 사용자들의 정보가 침해당하지 않도록 보호하는 역할 (IAM에서 보안을 강화하기 위해 사용)

* MFA 장치 옵션

  * 가상 MFA 장치

    * Google Authenticator: 하나의 휴대전화에서만 사용가능합니다.

    * Authy: 여러 장치에서 사용 가능(작동 방식은 동일) 합니다. 컴퓨터와 휴대폰에서 같이 사용할 수 있으며, Authy는 하나의 장치에서도 토큰을 여러 개 지원합니다.

    * 루트 계정, IAM 사용자 또 다른 계정, 그리고 또 다른 IAM 사용자가 지원되는 식으로 가상 MFA 장치에

      원하는 수만큼의 계정 및 사용자 등록이 가능합니다.

  * YubiKey

    * UF2 보안키
    * AWS 제 3자 회사 Yubico의 장치
    * YubiKey는 하나의 보안 키에서 여러 루트의 계정과 IAM 사용자를 지원하므로 하나의 키로도 충분합니다.

  * 하드웨어 키 Fob MFA 장치

    * AWS 제 3자 회사 Gemalto의 장치

  * 하드웨어 키 Fob MFA 장치(for AWS GovCloud)

    * AWS 제 3자 회사 SurePassID가 제공

### IAM Roles

* IAM Role은 실제 사람이 사용하도록 만들어진 것이 아니고, AWS 서비스에 의해 사용되도록 만들어졌습니다.
* EC2 인스턴스(가상서버)가 AWS에서 작업을 수행하기 위해서는 EC2 인스턴스에 권한을 부여해야 하며, 이때 IAM Role을 만들어서 하나의 개체로 만듭니다. 이후 EC2 인스턴스가 AWS에 있는 어떤 정보에 접근하려고 할 때 IAM Role을 사용하게 됩니다.

### IAM Security Tools

* IAM Credentials Report (IAM 자격 증명 보고서)
  * 계정에 있는 사용자와 다양한 Credential 들의 상태를 포함하고 있습니다.
* IAM Access Advisor
  * 사용자에게 부여된 서비스의 권한과 해당 서비스에 마지막으로 액세스한 시간을 보여줍니다. 

### IAM Guideline & Best Practices

* 루트 계정은 AWS 계정을 설정할 때를 제외하고 사용하지 않는 것이 좋습니다.
* 하나의 AWS 사용자는 한 명의 실제 사용자를 의미하므로, 1인 1계정으로 하는 것이 좋습니다.
* 비밀번호 정책을 강력하게 만들어야 합니다. (MFA 사용 등)
* AWS 서비스에 권한을 부여할 때 마다 역할을 만들고 사용해야 합니다.
* CLI나 SDK를 사용할 경우, 반드시 엑세스 키(Access Key)를 만들어야 합니다. 엑세스 키는 절대 노출하지 않도록 합니다.
* IAM Credentials Report와 IAM Access Advisor를 사용하여 계정 권한 감사를 하는 것이 좋습니다. 
* IAM 사용자와 액세스 키를 공유하지 않도록 합니다.

<br/>

> **REFERENCE**
>
> [https://www.udemy.com/](https://www.udemy.com/) (AWS Certified Solutions Architect Associate, Stephane Maarek)
>
> [https://docs.aws.amazon.com/?nc2=h_ql_doc_do](https://docs.aws.amazon.com/?nc2=h_ql_doc_do)
