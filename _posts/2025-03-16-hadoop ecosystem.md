---
key: jekyll-text-theme
title: 'Hadoop ecosystem'
excerpt: ' Hadoop study 😎'
tags: [Hadoop]
---



# Hadoop 에코시스템

Hadoop 에코시스템은 HDFS와 MapReduce를 중심으로 다양한 도구들이 결합된 통합 빅데이터 플랫폼임. 각 도구는 특정 용도에 최적화되어 있으며, 함께 사용하면 강력한 데이터 처리 파이프라인을 구축할 수 있음.

## 1. Hive - SQL 기반 데이터 웨어하우스

**개념**

Hive는 HDFS에 저장된 대용량 데이터를 SQL로 쿼리할 수 있게 해주는 데이터 웨어하우스 시스템임. SQL을 MapReduce 작업으로 변환하여 실행함.

**주요 특징**

- SQL 문법 지원 (HiveQL)
- 테이블 기반 데이터 관리
- 파티셔닝과 버킷팅으로 성능 최적화
- UDF(User Defined Function) 지원

**설치 방법**

```bash
# Hive 다운로드
wget https://downloads.apache.org/hive/hive-3.1.3/apache-hive-3.1.3-bin.tar.gz

# 압축 해제
tar -xzvf apache-hive-3.1.3-bin.tar.gz
sudo mv apache-hive-3.1.3-bin /usr/local/hive

# 환경변수 설정
echo 'export HIVE_HOME=/usr/local/hive' >> ~/.bashrc
echo 'export PATH=$PATH:$HIVE_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Hive 디렉토리 생성
hdfs dfs -mkdir /tmp
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse

# Metastore 초기화
schematool -dbType derby -initSchema

# Hive 시작
hive
```

**기본 사용법**

```sql
-- 데이터베이스 생성
CREATE DATABASE mydb;
USE mydb;

-- 테이블 생성
CREATE TABLE employees (
    id INT,
    name STRING,
    age INT,
    department STRING,
    salary DOUBLE
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

-- 데이터 로드 (HDFS에서)
LOAD DATA INPATH '/user/data/employees.csv' INTO TABLE employees;

-- 데이터 조회
SELECT * FROM employees WHERE age > 30;

-- 집계 쿼리
SELECT department, AVG(salary) as avg_salary
FROM employees
GROUP BY department;

-- 파티션 테이블 생성
CREATE TABLE sales (
    product_id INT,
    amount DOUBLE
)
PARTITIONED BY (year INT, month INT)
STORED AS PARQUET;

-- 파티션에 데이터 삽입
INSERT INTO TABLE sales PARTITION(year=2024, month=12)
SELECT product_id, amount FROM staging_sales
WHERE sale_date LIKE '2024-12%';
```

**실전 예시: 로그 분석**

~~~sql
-- 웹 로그 테이블 생성
CREATE EXTERNAL TABLE web_logs (
    ip STRING,
    timestamp STRING,
    method STRING,
    url STRING,
    status INT,
    size INT
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
    "input.regex" = "([^ ]*) - - \\[([^\\]]*)\\] \"([^ ]*) ([^ ]*) [^\"]*\" ([0-9]*) ([0-9]*)"
)
LOCATION '/logs/web/';

-- 시간대별 트래픽 분석
SELECT 
    HOUR(from_unixtime(unix_timestamp(timestamp, 'dd/MMM/yyyy:HH:mm:ss'))) as hour,
    COUNT(*) as requests
FROM web_logs
GROUP BY HOUR(from_unixtime(unix_timestamp(timestamp, 'dd/MMM/yyyy:HH:mm:ss')))
ORDER BY hour;

-- 가장 많이 방문한 페이지 Top 10
SELECT url, COUNT(*) as visits
FROM web_logs
WHERE status = 200
GROUP BY url
ORDER BY visits DESC
LIMIT 10;
```


~~~

## 2. HBase - 분산 NoSQL 데이터베이스

**개념**

HBase는 Hadoop 위에서 동작하는 분산 NoSQL 데이터베이스임. 수십억 개의 행과 수백만 개의 열을 가진 대용량 테이블을 실시간으로 읽고 쓸 수 있음.

**주요 특징**

- 컬럼 기반 저장 구조
- 실시간 랜덤 읽기/쓰기
- 자동 샤딩
- 선형적 확장성
- 강력한 일관성

**아키텍처**
```
┌─────────────────────────────────────────────────────────┐
│                    HBase Master                          │
│              (메타데이터 관리 & 조정)                      │
└─────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │RegionServer 1│ │RegionServer 2│ │RegionServer 3│
    ├──────────────┤ ├──────────────┤ ├──────────────┤
    │  Region A    │ │  Region B    │ │  Region C    │
    │  Region D    │ │  Region E    │ │  Region F    │
    └──────────────┘ └──────────────┘ └──────────────┘
              │             │             │
              └─────────────┼─────────────┘
                            ▼
                    ┌─────────────┐
                    │    HDFS     │
                    └─────────────┘
```

**설치 방법**

```bash
# HBase 다운로드
wget https://downloads.apache.org/hbase/2.5.5/hbase-2.5.5-bin.tar.gz

# 압축 해제
tar -xzvf hbase-2.5.5-bin.tar.gz
sudo mv hbase-2.5.5 /usr/local/hbase

# 환경변수 설정
echo 'export HBASE_HOME=/usr/local/hbase' >> ~/.bashrc
echo 'export PATH=$PATH:$HBASE_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# hbase-env.sh 설정
nano $HBASE_HOME/conf/hbase-env.sh
# 다음 줄 추가:
# export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
# export HBASE_MANAGES_ZK=true

# hbase-site.xml 설정
nano $HBASE_HOME/conf/hbase-site.xml
```

```xml
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://localhost:9000/hbase</value>
    </property>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/home/사용자명/zookeeper</value>
    </property>
</configuration>
```

**HBase 시작**

```bash
# HBase 시작
start-hbase.sh

# HBase Shell 접속
hbase shell

# 상태 확인
status
```

**기본 사용법**

```bash
# 테이블 생성
create 'users', 'info', 'contact'

# 테이블 목록 보기
list

# 데이터 삽입
put 'users', 'user1', 'info:name', 'John Doe'
put 'users', 'user1', 'info:age', '30'
put 'users', 'user1', 'contact:email', 'john@example.com'
put 'users', 'user1', 'contact:phone', '010-1234-5678'

# 데이터 조회
get 'users', 'user1'

# 특정 컬럼 조회
get 'users', 'user1', 'info:name'

# 전체 스캔
scan 'users'

# 조건부 스캔
scan 'users', {FILTER => "SingleColumnValueFilter('info', 'age', >, 'binary:25')"}

# 데이터 삭제
delete 'users', 'user1', 'info:age'

# 테이블 삭제
disable 'users'
drop 'users'
```

**Python으로 HBase 사용**

```python
import happybase

# HBase 연결
connection = happybase.Connection('localhost')

# 테이블 생성
connection.create_table(
    'users',
    {
        'info': dict(),
        'contact': dict()
    }
)

# 테이블 열기
table = connection.table('users')

# 데이터 삽입
table.put(b'user1', {
    b'info:name': b'John Doe',
    b'info:age': b'30',
    b'contact:email': b'john@example.com'
})

# 데이터 조회
row = table.row(b'user1')
print(row)

# 배치 삽입
batch = table.batch()
for i in range(1000):
    batch.put(f'user{i}'.encode(), {
        b'info:name': f'User {i}'.encode(),
        b'info:age': str(20 + i % 50).encode()
    })
batch.send()

# 스캔
for key, data in table.scan():
    print(key, data)

# 연결 종료
connection.close()
```

## 3. Pig - 데이터 흐름 스크립팅

**개념**

Pig는 MapReduce 작업을 간단한 스크립트로 작성할 수 있게 해주는 플랫폼임. Pig Latin이라는 고수준 언어를 사용함.

**설치 방법**

```bash
# Pig 다운로드
wget https://downloads.apache.org/pig/pig-0.17.0/pig-0.17.0.tar.gz

# 압축 해제
tar -xzvf pig-0.17.0.tar.gz
sudo mv pig-0.17.0 /usr/local/pig

# 환경변수 설정
echo 'export PIG_HOME=/usr/local/pig' >> ~/.bashrc
echo 'export PATH=$PATH:$PIG_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# Pig 실행
pig
```

**기본 사용법**

```pig
-- 데이터 로드
users = LOAD '/data/users.csv' USING PigStorage(',')
    AS (id:int, name:chararray, age:int, city:chararray);

-- 필터링
adults = FILTER users BY age >= 18;

-- 그룹화
by_city = GROUP adults BY city;

-- 집계
city_counts = FOREACH by_city GENERATE 
    group AS city, 
    COUNT(adults) AS user_count;

-- 정렬
sorted = ORDER city_counts BY user_count DESC;

-- 상위 10개 추출
top10 = LIMIT sorted 10;

-- 결과 저장
STORE top10 INTO '/output/city_stats' USING PigStorage(',');
```

**실전 예시: 로그 분석**

```pig
-- 웹 로그 로드
logs = LOAD '/logs/web/*.log' USING PigStorage(' ')
    AS (ip:chararray, timestamp:chararray, method:chararray, 
        url:chararray, status:int, size:int);

-- 성공한 요청만 필터링
successful = FILTER logs BY status == 200;

-- URL별 그룹화
by_url = GROUP successful BY url;

-- URL별 통계 계산
url_stats = FOREACH by_url GENERATE
    group AS url,
    COUNT(successful) AS hits,
    SUM(successful.size) AS total_bytes,
    AVG(successful.size) AS avg_bytes;

-- 히트 수로 정렬
sorted_urls = ORDER url_stats BY hits DESC;

-- 결과 저장
STORE sorted_urls INTO '/output/url_stats';
```

## 4. Sqoop - 데이터 전송 도구

**개념**

Sqoop은 관계형 데이터베이스(MySQL, PostgreSQL, Oracle 등)와 Hadoop 간의 데이터 전송을 자동화하는 도구임.

**설치 방법**

```bash
# Sqoop 다운로드
wget https://downloads.apache.org/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.0.tar.gz

# 압축 해제
tar -xzvf sqoop-1.4.7.bin__hadoop-2.0.tar.gz
sudo mv sqoop-1.4.7.bin__hadoop-2.0 /usr/local/sqoop

# 환경변수 설정
echo 'export SQOOP_HOME=/usr/local/sqoop' >> ~/.bashrc
echo 'export PATH=$PATH:$SQOOP_HOME/bin' >> ~/.bashrc
source ~/.bashrc

# JDBC 드라이버 복사 (MySQL 예시)
cp mysql-connector-java.jar $SQOOP_HOME/lib/
```

**MySQL에서 HDFS로 가져오기**

```bash
# 단일 테이블 가져오기
sqoop import \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --table employees \
  --target-dir /user/data/employees \
  --num-mappers 4

# 특정 컬럼만 가져오기
sqoop import \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --table employees \
  --columns "id,name,salary" \
  --target-dir /user/data/employees_basic

# WHERE 조건으로 필터링
sqoop import \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --table employees \
  --where "salary > 50000" \
  --target-dir /user/data/high_salary_employees

# 증분 가져오기 (변경된 데이터만)
sqoop import \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --table orders \
  --incremental append \
  --check-column order_id \
  --last-value 1000 \
  --target-dir /user/data/orders

# 전체 데이터베이스 가져오기
sqoop import-all-tables \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --warehouse-dir /user/data/mydb
```

**HDFS에서 MySQL로 내보내기**

~~~bash
# Hive 테이블을 MySQL로 내보내기
sqoop export \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --table employees_export \
  --export-dir /user/hive/warehouse/employees \
  --input-fields-terminated-by ',' \
  --num-mappers 4

# 업데이트 모드로 내보내기
sqoop export \
  --connect jdbc:mysql://localhost:3306/mydb \
  --username root \
  --password mypassword \
  --table employees \
  --update-key id \
  --export-dir /user/data/updated_employees
```


~~~

## 5. Flume - 로그 수집 도구

**개념**

Flume은 대용량의 로그 데이터를 효율적으로 수집하고 HDFS로 전송하는 분산 시스템임.

**아키텍처**
```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Source    │  -->  │   Channel   │  -->  │    Sink     │
│ (데이터 수집)  │       │  (버퍼링)    │       │ (데이터 전송)  │
└─────────────┘       └─────────────┘       └─────────────┘
```

**설치 방법**

```bash
# Flume 다운로드
wget https://downloads.apache.org/flume/1.11.0/apache-flume-1.11.0-bin.tar.gz

# 압축 해제
tar -xzvf apache-flume-1.11.0-bin.tar.gz
sudo mv apache-flume-1.11.0-bin /usr/local/flume

# 환경변수 설정
echo 'export FLUME_HOME=/usr/local/flume' >> ~/.bashrc
echo 'export PATH=$PATH:$FLUME_HOME/bin' >> ~/.bashrc
source ~/.bashrc
```

**기본 설정 파일 작성**

**웹 서버 로그 수집 예시 (flume-weblog.conf)**

```properties
# Agent 이름
agent.sources = weblog
agent.channels = memChannel
agent.sinks = hdfsSink

# Source 설정 (파일 감시)
agent.sources.weblog.type = exec
agent.sources.weblog.command = tail -F /var/log/apache/access.log
agent.sources.weblog.channels = memChannel

# Channel 설정 (메모리 버퍼)
agent.channels.memChannel.type = memory
agent.channels.memChannel.capacity = 10000
agent.channels.memChannel.transactionCapacity = 1000

# Sink 설정 (HDFS 저장)
agent.sinks.hdfsSink.type = hdfs
agent.sinks.hdfsSink.channel = memChannel
agent.sinks.hdfsSink.hdfs.path = /logs/web/%Y/%m/%d
agent.sinks.hdfsSink.hdfs.filePrefix = access_log
agent.sinks.hdfsSink.hdfs.fileSuffix = .log
agent.sinks.hdfsSink.hdfs.rollInterval = 3600
agent.sinks.hdfsSink.hdfs.rollSize = 134217728
agent.sinks.hdfsSink.hdfs.rollCount = 0
agent.sinks.hdfsSink.hdfs.fileType = DataStream
agent.sinks.hdfsSink.hdfs.writeFormat = Text
agent.sinks.hdfsSink.hdfs.useLocalTimeStamp = true
```

**Flume 실행**

```bash
# Flume agent 시작
flume-ng agent \
  --conf $FLUME_HOME/conf \
  --conf-file flume-weblog.conf \
  --name agent \
  -Dflume.root.logger=INFO,console
```

**네트워크 소켓 수신 예시 (flume-netcat.conf)**

```properties
# Agent 설정
agent.sources = netcatSource
agent.channels = memChannel
agent.sinks = loggerSink

# Netcat Source (포트 44444에서 수신)
agent.sources.netcatSource.type = netcat
agent.sources.netcatSource.bind = localhost
agent.sources.netcatSource.port = 44444
agent.sources.netcatSource.channels = memChannel

# Memory Channel
agent.channels.memChannel.type = memory
agent.channels.memChannel.capacity = 1000

# Logger Sink (콘솔 출력)
agent.sinks.loggerSink.type = logger
agent.sinks.loggerSink.channel = memChannel
```

**테스트**

```bash
# Flume 실행
flume-ng agent \
  --conf $FLUME_HOME/conf \
  --conf-file flume-netcat.conf \
  --name agent

# 다른 터미널에서 데이터 전송
telnet localhost 44444
Hello Flume!
```

## 6. Oozie - 워크플로우 스케줄러

**개념**

Oozie는 Hadoop 작업을 조정하고 스케줄링하는 워크플로우 엔진임. MapReduce, Pig, Hive, Sqoop 등의 작업을 순차적 또는 병렬로 실행할 수 있음.

**워크플로우 예시 (workflow.xml)**

```xml
<workflow-app name="data-pipeline" xmlns="uri:oozie:workflow:0.5">
    <start to="sqoop-import"/>
    
    <!-- Sqoop으로 데이터 가져오기 -->
    <action name="sqoop-import">
        <sqoop xmlns="uri:oozie:sqoop-action:0.4">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <command>import --connect jdbc:mysql://localhost/mydb --table users --target-dir /data/users</command>
        </sqoop>
        <ok to="hive-process"/>
        <error to="fail"/>
    </action>
    
    <!-- Hive로 데이터 처리 -->
    <action name="hive-process">
        <hive xmlns="uri:oozie:hive-action:0.5">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <script>process_users.hql</script>
        </hive>
        <ok to="end"/>
        <error to="fail"/>
    </action>
    
    <kill name="fail">
        <message>Workflow failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    
    <end name="end"/>
</workflow-app>
```

**Coordinator 설정 (매일 실행)**

```xml
<coordinator-app name="daily-data-pipeline" frequency="${coord:days(1)}"
                 start="2024-01-01T00:00Z" end="2024-12-31T23:59Z"
                 timezone="Asia/Seoul"
                 xmlns="uri:oozie:coordinator:0.4">
    <action>
        <workflow>
            <app-path>${workflowPath}</app-path>
        </workflow>
    </action>
</coordinator-app>
```

## 7. Zookeeper - 분산 코디네이션

**개념**

Zookeeper는 분산 시스템을 위한 코디네이션 서비스임. 설정 관리, 네이밍 서비스, 분산 동기화, 그룹 서비스를 제공함.

**주요 기능**

- 설정 정보 중앙 관리
- 리더 선출
- 분산 락
- 서비스 디스커버리

**설치 및 실행**

```bash
# Zookeeper 다운로드
wget https://downloads.apache.org/zookeeper/zookeeper-3.8.2/apache-zookeeper-3.8.2-bin.tar.gz

# 압축 해제
tar -xzvf apache-zookeeper-3.8.2-bin.tar.gz
sudo mv apache-zookeeper-3.8.2-bin /usr/local/zookeeper

# 설정 파일 생성
cp /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg

# Zookeeper 시작
/usr/local/zookeeper/bin/zkServer.sh start

# CLI 접속
/usr/local/zookeeper/bin/zkCli.sh
```

**기본 명령어**

~~~bash
# 노드 생성
create /myapp "app_data"

# 노드 조회
get /myapp

# 노드 목록
ls /

# 노드 업데이트
set /myapp "new_data"

# 노드 삭제
delete /myapp

# 상태 확인
stat /myapp
```
~~~

## 에코시스템 통합 예시

**실시간 데이터 파이프라인**
```
[웹 서버 로그]
       ↓
   [Flume] ─────→ [Kafka] ─────→ [Spark Streaming]
                                         ↓
                                    [HBase] ← 실시간 조회
                                         ↓
                                     [HDFS] ← 장기 보관
                                         ↓
                                     [Hive] ← 배치 분석
                                         ↓
                                     [BI 도구]
```

**배치 ETL 파이프라인 (Oozie 조정)**
```
1. [Sqoop] MySQL → HDFS
2. [Pig] 데이터 정제
3. [Hive] 집계 및 변환
4. [Sqoop] HDFS → Data Warehouse
