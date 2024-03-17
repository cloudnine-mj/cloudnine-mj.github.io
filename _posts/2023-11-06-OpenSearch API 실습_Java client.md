---
key: jekyll-text-theme
title: 'OpenSearch API 실습 - Java client'
excerpt: ' OpenSearch research 😎'
tags: [Opensearch, ELK, Research]
---



:star: 회사에서 OpenSearch 사용하면서 여러 가지 시도를 하다가 OpenSearch API 관련 내용도 한 번 해보기로 했다.


# OpenSearch API 실습 - Java client

## 프로젝트 생성

### Apache maven 설치

* 3.6.3버전은 jdk 17 버전에 최적화 되어 있다.
* apt 사용은 추천하지 않음 (표기는 동일버전이라도 실제 wget으로 얻는것과 다른 패키지가 나와서 wget을 사용하는 게 맞는 것 같다.)

```
$ wget https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.6.3/apache-maven-3.6.3-bin.tar.gz

$ vi ~/.bashrc
export MAVEN_HOME=$HOME/maven
export JAVA_HOME=$HOME/jdk
export PATH=$HOME/script:$JAVA_HOME/bin:$MAVEN_HOME/bin:$PATH

$ mvn --version
Apache Maven 3.6.3 (cecedd343002696d0abb50b32b541b8a6ba2883f)
Maven home: /home/opensearch/maven
Java version: 17.0.1, vendor: Oracle Corporation, runtime: /home/opensearch/pack/jdk-17.0.1
Default locale: en, platform encoding: UTF-8
OS name: "linux", version: "5.4.0-153-generic", arch: "amd64", family: "unix"
```

### 프로젝트 생성

```
mkdir MyJavaProject
cd MyJavaProject

mvn archetype:generate -DgroupId=com.example -DartifactId=my-java-project -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

### package clean

```
mvn clean package 
```

### 앱 실행 테스트

```
java -cp target/my-java-project-1.0-SNAPSHOT.jar com.example.App 
```

 

## pom.xml 설정

* xml 설정을 통해 http client 라이브러리와 의존성 라이브러리를 설치한다.
* linux cli 환경에서 classpath를 못 찾는 경우가 있는데 shade plugin을 설치하면 됨.
* (보통은 IDE에서 알아서 해주는 경우가 많긴 하...엣헴)

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>opensearch_client</artifactId>
  <packaging>jar</packaging>
  <version>1.0-SNAPSHOT</version>
  <name>opensearch_client</name>
  <url>http://maven.apache.org</url>

  <properties>
    <maven.compiler.source>7</maven.compiler.source>
    <maven.compiler.target>7</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>

<!-- http client와 http core 가 필요하다. commons-logging은 없어도 문제는 없음 -->
    <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpclient</artifactId>
      <version>4.5.13</version>
    </dependency>


    <dependency>
      <groupId>org.apache.httpcomponents</groupId>
      <artifactId>httpcore</artifactId>
      <version>4.4.14</version>
    </dependency>
    
    <dependency>
      <groupId>commons-logging</groupId>
      <artifactId>commons-logging</artifactId>
      <version>1.2</version>
    </dependency>
  </dependencies>
  
<!-- shade plugin이 없으면 classpath를 못 찾는다. -->
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.4.1</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```


### 샘플 코드

- App.java는 src/main/java/com/example/ 에 있다. 해당 코드를 수정해주면 됨.

```
package com.example;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManager;
import javax.net.ssl.X509TrustManager;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.client.HttpClient;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.conn.ssl.NoopHostnameVerifier;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

public class App {
    public static void main(String[] args) throws Exception {

        String username = "admin";
        String password = "admin";

        // TLS 설정
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, new TrustManager[] { new X509TrustManager() {
            public void checkClientTrusted(X509Certificate[] arg0, String arg1) throws CertificateException {}
            public void checkServerTrusted(X509Certificate[] arg0, String arg1) throws CertificateException {}
            public X509Certificate[] getAcceptedIssuers() { return null; }
        } }, null);
        
        // NoopHostnameVerifier 부분은 인증서 없이 접속할 수 있게 해주는 부분이다. 반드시 들어가야 함.
        SSLConnectionSocketFactory sslSocketFactory = new SSLConnectionSocketFactory(sslContext,
                NoopHostnameVerifier.INSTANCE);

        // user credential 부분
        CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));
        HttpClient httpClient = HttpClients.custom()
                .setSSLSocketFactory(sslSocketFactory)
                .setDefaultCredentialsProvider(credentialsProvider)
                .build();

        HttpGet httpGet = new HttpGet("https://192.168.0.155:5200");

        HttpResponse response = httpClient.execute(httpGet);

        HttpEntity entity = response.getEntity();
        if (entity != null) {
            String result = EntityUtils.toString(entity);
            System.out.println(result);
        }
    }
}
```

 

## 실행

- 해당 코드를 기반으로 접속 후 원하는 엔드 포인트에 맞춰 작업을 수행하면 된다.
  예시는 노드 정보를 얻는 단순 요청.

```
$ mvn clean package

[INFO] Scanning for projects...
[INFO] 
[INFO] -------------------< com.example:opensearch_client >--------------------
[INFO] Building opensearch_client 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------

...

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  1.268 s
[INFO] Finished at: 2023-07-17T09:30:55Z
[INFO] ------------------------------------------------------------------------

```

```
$ java -cp target/opensearch_client-1.0-SNAPSHOT.jar com.example.App
{
  "name" : "node-1",
  "cluster_name" : "kj-cluster",
  "cluster_uuid" : "3tGvTRKERPqhtjwV0teE6w",
  "version" : {
    "distribution" : "opensearch",
    "number" : "2.6.0",
    "build_type" : "tar",
    "build_hash" : "7203a5af21a8a009aece1474446b437a3c674db6",
    "build_date" : "2023-02-24T18:57:04.388618985Z",
    "build_snapshot" : false,
    "lucene_version" : "9.5.0",
    "minimum_wire_compatibility_version" : "7.10.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```
