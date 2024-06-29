---
key: jekyll-text-theme
title: 'Python code 정리'
excerpt: ' coding test 준비 😎'
tags: [Coding]
---

👉 코딩테스트에서 자주 쓸 수 있는 Python 코드 정리

# Python code 정리



## Python 입력문 정리

* a = input()      =>  한 줄을 문자열로 입력 받는다. 
	*  abc de =>  a = 'abc de'
* b = int(input())   => 한 줄을 정수로 입력 받는다. 
	* 3 => a = 3,   3 5 => error,   35 => a = 35
* arr = input().split()   =>  한 줄을 입력 받아서, 공백 기준으로 분리해 리스트에 저장.    
	* abc de 3 12 =>   arr = ['abc', 'de', '3', '12']
* a, b, c = map(int, input().split()) => 한 줄에 여러 정수를 입력 받을 때 사용.  
	* 3 12 123 => a, b, c = 3, 12, 123
* arr = list(map(int, input().split()))  =>  한 줄에 여러 정수를 입력 받아서 리스트에 저장.    
	* 3 12 123 => arr = [3, 12, 123]


## 정수론 코드 정리

```
n % 10 => 일의자리를 가져오는 연산
n //= 10 => 일의자리를 지우는 연산
```

```
total = 0
while n > 0:
     total += n % 10
     n //= 10
```



