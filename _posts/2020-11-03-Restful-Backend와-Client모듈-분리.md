---
layout: post
title:  "Backend와 Client모듈 분리(작성중)"
date:   2020-11-03
excerpt: "<a target=_black href=\"https://github.com/PotatoWhite/backend-client\">https://github.com/PotatoWhite/backend-client</a>"
image: ""
---

## 목표

- Gradle Multi Module 생성, RESTful API Server실습, Client를 이용한 RESTful API 호출

## 사유
단지 RESTful API가 아니더라도, Client-Server의 관점에서 살펴볼 때 Server는 Client의 대리자입니다. 이는 Server가 Client의 할일을 위임 받는 것이고, Cient의 요구에 따라 Server의 규격이 변경되겠지만, 현실에서 다수(단지 양을 이야기하는 것이 아님)의 Client를 Server가 처리할 때는 연동 규격의 변경은 Client들을 재개발 시킬 수 있습니다.

Server는 Client가 할일(Job)을 위임하는 대리자 입니다. Server가 Job을 수행할 때 필요한 최소한의 정보가 있습니다. 이 정보가 문맥에 따라서  Parameter 또는 Form이라고 불리웁니다.

예를 들어 Potato라는 계정을 사용하는 사용자가 차 한잔 주세요. 라는 요청이 있습니다.

```java
Client potato = new Client();
Server server = new Server();

Tea tea = server.getATea();
potato.addTea(tea);
```

위 상황에서 potato는 Tea의 내용이 어떻든 일단 하나 받습니다. 추가적으로 만일 potato가 Tea 종류를 녹차나 홍차를 골라서 주문한다면 다음과 같이 표현됩니다.


```java
Client potato = new Client();
Server server = new Server();

Tea tea = server.getATea(TypeOfTea.GREEN);
potato.addTea(tea);
```

추가적으로 getATea를 호출할떄 enum 형식의 GREEN을 함께 호출합니다. 이러한 동작이 하나의 Application내에서 일어나는 일이라면 TypeOfTea 또는 Tea 모두 큰일 아닐 수 있습니다. 

하지만 이러한 주문이 네트워크를 타고 날아드는 호출에 Server가 약 수십 종의 Microservice에서 사용된다면? 예제의 Tea와 TypeOfTea는 네트워크를 타고 전달되어야 하므로 Application에서 미리 해당 규격이 공유되어야 합니다.

협업도구를 사용한다면 그나마 다행이겠지만 연동규격서를 이메일로 보내고, 각 연동 지점마다 들어오는 문의사항 응대에... 
실제 작업이 크지 않더라도 하나의 작은 기능 배포를 위해 배보다 배꼽이 큰일을 우리는 종종 격을 수 있습니다. 이는 요즘 그렇게 중요하다는 시스템의 Agility를 떨어 뜨리고 새로운 비즈니스도 전개 느릴수 밖에 없습니다.

예를 들어 카카오톡 등의 메신저 로그인을 위한 API 변경이 발생하였을 때 카카오톡 메신저, 카카오택시 등 다수의 Client가 변경되어야 합니다. 이를 이용하는 3rd Party App들은 말 할 것도 없지요.

만일 Server에서 API를 개발할 때 Client Module까지 만들어서 제공한다면.. 이라 한다면 많은 서버개발자들의 원망을 살 수 있겠지만, 그리 어려운 일은 아닙니다. 약간의 수고스러움을 더한다면 유연성 확보할 수 있습니다. 조금 Old Fashion이긴 Server 자신을 호출하는 Lib를 배포하여 사용하게 하면 Client는 조금 쉬운방법(?)으로 연동을 할 수 있을 것 입니다. 프로젝트에서 사용할 Nexus가 있다면 프로젝트에 활력을 넣을 수 있습니다. 
 
본 실습 내용은 Server가 RESTful API와 관련 DTO, 호출할 수 있는 Client모듈을 제공하고, DTO + Client 모듈을 Nexus를 통해 제공하는 예제를 진행합니다.

## 실습 설명

1. Gradle을 이용하 Client와 Server가 분리된 프로젝트 환경을 만들고
2. Server를 구현하고
3. Client를 구현하고 Nexus에 Upload 
4. Test룔 Application을 만들어 Nexus에 있는 Cient를 받아서 호출해 본다. 

## 0. 사전환경

- Gradle 설치
- JDK 설치

## 1. Gradle Multi Module 구성

### 1.1 Gradle Multi Module 준비

적당한 위치에 Directory를 생성한다.

```sh
> mkdir demo-project
> cd demo-project
```

settings.gradle 파일을 생성하고 다음 내용을 붙여 넣는다.

```gradle
rootProject.name = 'demo-project'
include 'demo-service'
include 'demo-client'
```

build.gradle을 생성하고 다음 내용을 붙여 넣는다.

```groovy
buildscript {
    ext {
        springBootVersion = '2.3.5.RELEASE'
        springCloudVersion = 'Hoxton.SR8'
    }

     repositories {
        mavenCentral()
    }

    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

allprojects {
    group = 'me.potato'
    version = '0.1-SNAPSHOT'
}

subprojects {
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'
    apply plugin: 'java'

    sourceCompatibility = '11'


    repositories {
        mavenCentral()
        jcenter()
    }

    dependencyManagement {
        imports {
            mavenBom "org.springframework.boot:spring-boot-dependencies:${springBootVersion}"
        }
        imports {
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
    dependencies {

        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
    }


    task initSourceFolders {
        sourceSets*.java.srcDirs*.each {
            if (!it.exists()) {
                it.mkdirs()
            }
        }
        sourceSets*.resources.srcDirs*.each {
            if (!it.exists()) {
                it.mkdirs()
            }
        }
    }

}
```

demo-client/build.gradle 파일을 생성하고 다음 내용을 붙여넣는다.

```groovy
bootJar {
    enabled(false)
}

jar {
    enabled(true)
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    annotationProcessor 'org.projectlombok:lombok'

    // for Unit Test
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    testImplementation 'io.projectreactor:reactor-test'
    testAnnotationProcessor 'org.projectlombok:lombok'
    testCompileOnly 'org.projectlombok:lombok'

}
```

demo-service/build.gradle 파일을 생성하고 다음 내용을 붙여넣는다.

```groovy
bootJar {
    enabled(true)
}

jar {
    enabled(true)
}

dependencies {
    implementation('org.springframework.boot:spring-boot-starter-web')
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.h2database:h2'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'

    // for Unit Test
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    testImplementation 'io.projectreactor:reactor-test'
    testAnnotationProcessor 'org.projectlombok:lombok'
    testCompileOnly 'org.projectlombok:lombok'
}
```

### 1.2 Gradle build

```sh
> gradle build

BUILD SUCCESSFUL in 559ms
2 actionable tasks: 2 executed
```

## 2. demo-service 구현

### 2.1 root package 생성

```sh
❯ mkdir -p ./demo-service/src/main/java/me/potato/demoservice
```

### 2.2 Application.java 작성
./demo-service/src/main/java/me/potato/demoservice/Application.java 를 생성하고 다음 내용을 붙여 넣는다.

```java
package me.potato.demoservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}

```

여기까지 준비 되었다면, VS Code를 이용해서 프로젝트를 Open한다.
```sh
> code .

또는 VS Code 'File > Open' 메뉴를 이용해서 프로젝트 디렉토리를 연다.
```

### 2.3 Gradle Build 및 실행

프로젝트를 빌드하고 실행해본다. 여기까지 잘 따라왔다면 아래의 내용이 동작해야한다.

빌드
```sh
> gradle bootjar

BUILD SUCCESSFUL in 7s
2 actionable tasks: 2 executed
```

실행
```sh
> java -jar ./demo-service/build/libs/demo-service-0.1-SNAPSHOT.jar

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.3.5.RELEASE)

2020-11-03 20:14:58.645  INFO 54490 --- [           main] me.potato.demoservice.Application        : Starting Application on Potatoui-MacBookPro.local with PID 54490 (/Users/potato/personalSpace/test/demo-service/build/libs/demo-service-0.1-SNAPSHOT.jar started by potato in /Users/potato/personalSpace/test)
2020-11-03 20:14:58.648  INFO 54490 --- [           main] me.potato.demoservice.Application        : No active profile set, falling back to default profiles: default
2020-11-03 20:14:59.205  INFO 54490 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Bootstrapping Spring Data JPA repositories in DEFERRED mode.
2020-11-03 20:14:59.223  INFO 54490 --- [           main] .s.d.r.c.RepositoryConfigurationDelegate : Finished Spring Data repository scanning in 11ms. Found 0 JPA repository interfaces.
2020-11-03 20:14:59.769  INFO 54490 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-11-03 20:14:59.783  INFO 54490 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-11-03 20:14:59.783  INFO 54490 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.39]
2020-11-03 20:14:59.855  INFO 54490 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-11-03 20:14:59.856  INFO 54490 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1152 ms
2020-11-03 20:15:00.062  INFO 54490 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-11-03 20:15:00.077  INFO 54490 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-11-03 20:15:00.341  INFO 54490 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2020-11-03 20:15:00.431  INFO 54490 --- [         task-1] o.hibernate.jpa.internal.util.LogHelper  : HHH000204: Processing PersistenceUnitInfo [name: default]
2020-11-03 20:15:00.481  WARN 54490 --- [           main] JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2020-11-03 20:15:00.522  INFO 54490 --- [         task-1] org.hibernate.Version                    : HHH000412: Hibernate ORM core version 5.4.22.Final
2020-11-03 20:15:00.748  INFO 54490 --- [         task-1] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.1.0.Final}
2020-11-03 20:15:00.918  INFO 54490 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-11-03 20:15:00.920  INFO 54490 --- [           main] DeferredRepositoryInitializationListener : Triggering deferred initialization of Spring Data repositories…
2020-11-03 20:15:00.921  INFO 54490 --- [           main] DeferredRepositoryInitializationListener : Spring Data repositories initialized!
2020-11-03 20:15:00.929  INFO 54490 --- [         task-1] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.H2Dialect
2020-11-03 20:15:00.933  INFO 54490 --- [           main] me.potato.demoservice.Application        : Started Application in 2.751 seconds (JVM running for 3.216)
2020-11-03 20:15:01.222  INFO 54490 --- [         task-1] o.h.e.t.j.p.i.JtaPlatformInitiator       : HHH000490: Using JtaPlatform implementation: [org.hibernate.engine.transaction.jta.platform.internal.NoJtaPlatform]
2020-11-03 20:15:01.231  INFO 54490 --- [         task-1] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
```


### 2. Client-Server 간의 동작의 형식화


