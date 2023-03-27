---
key: jekyll-text-theme
title: 'AWS IAM과 AWS CLI'
excerpt: 'AWS Study 서비스 정리 - AWSKRUG Beginner Certification Study 😎'
tags: [AWS, SAA, cloud, study] 
---


## AWS IAM

* IAM = Identify and Access Management (Global 서비스에 해당)

* IAM을 사용하면 AWS 계정 내에서 사용자, 그룹 및 역할을 생성하고 관리할 수 있습니다.

  * Faker, Oner, Zeus는 같이 일하는 개발자 -> Developers 라는 그룹을 생성한 후 Faker, Oner, Zeus를 배치할 수 있습니다.

  * 그룹에는 사용자만 배치할 수 있고, 다른 그룹은 포함시킬 수 없습니다.
  
  * AWS 계정을 사용하도록 허용하기 위해서 사용자와 그룹을 생성합니다.
  
  * AWS 허용을 위해서 권한을 부여해야 하며, 권한 부여를 위해 사용자 또는 그룹에게 정책, 또는 IAM 정책이라고 불리는 JSON 문서를 지정할 수 있습니다.
  
  * AWS에서는 최소 권한의 원칙 적용: 사용자가 필요로 하는 것 이상의 권한을 주지 않는다. (새로운 사용자가 너무 많은 서비스를 실행하여 큰 비용이 발생하거나, 보안 문제를 야기할 수 있기 때문)
  
    
  
  
  
  
