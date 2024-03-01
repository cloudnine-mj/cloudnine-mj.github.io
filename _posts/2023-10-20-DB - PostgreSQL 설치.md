---
key: jekyll-text-theme
title: 'PostgreSQL 설치'
excerpt: 'PostgreSQL 😎'
tags: [PostgreSQL, DB]
---

# PostgreSQL 설치

## 1. 유저 생성 및 패스워드 설정

```
root@ubuntu22:~# useradd -d /home/postgres -s /bin/bash -m postgres
root@ubuntu22:~# passwd postgres
```

## 2. 패키지 다운로드 및 압축 해제

```
postgres@ubuntu22:~$ wget https://ftp.postgresql.org/pub/source/v15.2/postgresql-15.2.tar.gz

postgres@ubuntu22:~$ tar xvfz postgresql-15.2.tar.gz
```

## 3. gcc 설치

* root에서 진행

```
root@ubuntu22:~# apt install g++
root@ubuntu22:~# apt install gcc
```

## 4. make 설치

* root에서 진행

```
root@ubuntu22:~# apt install make
```

## 5. make, make install 작업 진행

```
root@ubuntu22:~# su - postgres  
postgres@ubuntu22:~$ cd posgresql-15.2/  
postgres@ubuntu22:~/postgresql-15.2$ ./configure --prefix=/home/postgres/postgresql --without-readline --without-zlib  
postgres@ubuntu22:~/postgresql-15.2$ make  
postgres@ubuntu22:~/postgresql-15.2$ make install  
```

## 6. 환경 파일 설정 및 실행

```
postgres@ubuntu22:~$ vi ~/.bash_profile
```

* 다음과 같이 작성

```
export PG_HOME=/home/postgres/postgresql
export PATH=$PG_HOME/bin:$PATH
export PGDATA=${HOME}/as15/data
export PGDATABASE=postgres
export PGUSER=postgres
export PGPORT=5432
export PGLOCALEDIR=$PG_HOME/share/locale
export MANPATH=$MANPATH:$PG_HOME/share/man
PS1='[$PWD] '
set -o vi
```

* bash_profile 실행

```
postgres@ubuntu22:~$ source .bash_profile
```

**[실행 결과]**
[/home/postgres] 

## 7. initdb 설정

```
[/home/postgres] initdb
```

## 8. properties 수정
* postgresql.conf 와 pg_hba.conf 수정

### postgresql.conf 수정

```
[/home/postgres/as15/data] vi postgresql.conf
```

* 다음과 같이 작성

```
shared_buffers= 2GB # 8GB 서버에서 사용하는 공유메모리 버퍼(메모리의 1/4)  // default : 970096kB  
listen_addresses='*' # // 접속 설정 여부. (Default 값)  
logging_collector=on # default on  
log_destination='stderr' # default stderr  
log_connections=on  #default off  
```

### pg_hba.conf

```
[/home/postgres/as15/data] vi pg_hba.conf
```

* 다음과 같이 conf에 추가

```
host    all             all             0.0.0.0/0            trust
```

