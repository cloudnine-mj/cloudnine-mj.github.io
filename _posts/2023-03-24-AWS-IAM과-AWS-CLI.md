---
key: jekyll-text-theme
title: 'AWS IAM과 AWS CLI'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---


## AWS IAM 소개

* IAM = Identify and Access Management (Global 서비스에 해당)
* 인증 및 권한 관리 서비스
* IAM을 사용하면 AWS 계정 내에서 사용자, 그룹 및 역할을 생성하고 관리할 수 있습니다.
  * 하나의 사용자는 조직 내의 한 사람!
  * Faker, Oner, Zeus는 같이 일하는 개발자 -> Developers 라는 그룹을 생성한 후 Faker, Oner, Zeus를 배치할 수 있습니다.
  * 그룹에는 사용자만 배치할 수 있고, 다른 그룹은 포함시킬 수 없습니다.
  * AWS 계정을 사용하도록 허용하기 위해서 사용자와 그룹을 생성합니다.
  * AWS 허용을 위해서 권한을 부여해야 하며, 권한 부여를 위해 사용자 또는 그룹에게 정책, 또는 IAM 정책이라고 불리는 JSON 문서를 지정할 수 있습니다.
  * AWS에서는 최소 권한의 원칙 적용: 사용자가 필요로 하는 것 이상의 권한을 주지 않는다. (새로운 사용자가 너무 많은 서비스를 실행하여 큰 비용이 발생하거나, 보안 문제를 야기할 수 있기 때문)


## IAM 사용자 및 그룹 실습

###  Users(사용자)

* 사용자는 AWS 리소스에 대한 엑세스 권한을 가짐

* IAM에서 생성한 사용자는 자격 증명을 사용하여 로그인할 수 있음

* Users 설정 방법

  #### ① AWS 콘솔 로그인

  #### ② IAM 서비스로 접속

  ​	<img src="https://user-images.githubusercontent.com/113915835/228197363-5a4a4b87-6e0a-45c8-8929-299a1e8e514e.png" width="80%">

  #### ③ Access Management (IAM 대시보드)

  ​    <img src="https://user-images.githubusercontent.com/113915835/228213555-6c6cb756-1199-4fd7-8986-dbda2dff581d.png" width="80%">
  
  #### ④ Users
  
  ​	<img src="https://user-images.githubusercontent.com/113915835/228214421-95bd0fe5-442c-4773-804d-efce2b012f99.png" width="80%">
  
  #### ⑤ Add users
  
  ​	<img src="https://user-images.githubusercontent.com/113915835/228215034-c66ca5e2-0d3e-46e0-b82a-caf8ffbe7903.png" width="80%">
  
  ​    <img src="https://user-images.githubusercontent.com/113915835/228215420-feab2c7a-52c2-4e3e-81b2-ffe2b5f6a771.png" width="80%">

## IAM Policies (정책)



  

  
