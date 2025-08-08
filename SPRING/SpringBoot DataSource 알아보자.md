# SpringBoot DataSource 알아보자

---

> 💡 프로젝트 세팅 도중 DataSource 관련에 대해서 이번에 정리를 해보고자 함
> 아래 내용은 Spring Boot 2.7.15 버전 기준으로 설명한다.
> 

## 개발 환경

---

- JAVA 11
- Spring Boot 2.7.15

## 개요

---

SpringBoot 프로젝트에서 `application.yml` 을 정리하면서 `spring.datasource`  관련해서 아래와 같은 질문을 받게 되었다.

```yaml

# application.yml

[AS-IS]

spring:
  datasource: 
    hikari:
      jdbc-url: ***
      driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
      username: ***
      password: ***

[TO-BE]

spring:
  datasource: 
    url: ***
    driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
    username: ***
    password: ***

#build.gradle

runtimeOnly 'com.h2database:h2'
```

H2 데이터베이스를 사용하지 않기 때문에 `build.gradle` 에서 관련 `dependency` 인 `runtimeOnly 'com.h2database:h2'` 삭제한 뒤에 `AS-IS` 와 같이 `build` 를 하면 에러가 나고 `TO-BE` 와 같이 `build` 를 하면 성공한다.

### **이유가 무엇인가?**

> 정확한 이유를 모르기에 오랜만에 개인적인 호기심에 해당 내용이 왜 그런 것인지
찾아보기로 하였다.
> 

## 원인을 찾아서

---

### 에러 구문에서 힌트를 찾아보자

---

```java
[WARN] o.s.b.w.s.c.AnnotationConfigServletWebServerApplicationContext.refresh - Exception encountered during context initialization - cancelling refresh attempt: 

org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'dataSourceScriptDatabaseInitializer' defined in class path resource [org/springframework/boot/autoconfigure/sql/init/DataSourceInitializationConfiguration.class]: 

Unsatisfied dependency expressed through method 'dataSourceScriptDatabaseInitializer' parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: 

Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Hikari.class]: 

Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: 

Failed to instantiate [com.zaxxer.hikari.HikariDataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: 

Failed to determine a suitable driver class 
```

- HikariDataSource  인스턴스를 생성할 때 실패
- Factory Method dataSource 에서 Exception 발생
- suitable driver class 를 결정을 실패

```

***************************
APPLICATION FAILED TO START
***************************

Description:

Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.

Reason: Failed to determine a suitable driver class

Action:

Consider the following:
	If you want an embedded database (H2, HSQL or Derby), please put it on the classpath.
	If you have database settings to be loaded from a particular profile you may need to activate it (the profiles local are currently active).
```

- DataSource Configure 실패
- url 속성이 지정되지 않음
- Embedded DataSource 단어 확인
- suitable driver class 를 결정을 실패

### 1. suitable driver class ??

- `spring.datasource.driver-class-name` 과 `spring.datasource.hikari.driver-class-name` 설정 값인 `JDBC Driver`가 잘못 설정되어 있는 것인가 의심
- 기존 `net.sf.log4jdbc.sql.jdbcapi.DriverSpy` → `com.mysql.cj.jdbc.Driver` 변경
- 변경해서 해보았지만 동일한 상황이 발생
- 다른 프로젝트에서 이미 `net.sf.log4jdbc.sql.jdbcapi.DriverSpy` 정의하고 있었기 때문에
이 부분은 문제가 없다고 **결론**

### 2. url 속성이 지정되지 않음 ??

- `spring.datasource.url`  과 `spring.datasource.hikari.jdbc-url`  차이점이 있는 것일까?
    - `spring.datasource.url` : SpringBoot 에서 지원하는 `Connection Pool` 모두 적용되는 URL
    - `spring.datasource.hikari.jdbc-url` : HikariCP 만 적용 할 URL
    - SpringBoot 문서에 보면 아래 문구가 나온다
        > 💡 You should at least specify the URL by setting the `spring.datasource.url` property. Otherwise, Spring Boot tries to auto-configure an embedded database. <br>
        > [SpringBoot DataSource Configuration 설명](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql.datasource.configuration) <br>
        > 즉, SpringBoot 는 `spring.datasource.url` 값을 정의하지 않으면 `Embedded Database` 의 자동 구성을 시도한다.
        

### 3. Embedded Database ??

- 애플리케이션 소프트웨어 밀접히 포함된 Database 이다
    - [Embedded Database Wiki](https://en.wikipedia.org/wiki/Embedded_database)
- SpringBoot 문서 내용 중 중요하다고 본 내용이다
    - In-Memory Embedded Database 를 사용하여 개발하는게 편리한 경우가 있다
    - Spring Boot 는 포함된 H2, HSQL, Derby Database 를 자동 구성할 수 있다
    - build dependency 에 사용하고 싶은 Embedded database 를 추가하면 connection url 을
    설정하지 않아도 된다.
    
    > 💡 It is often convenient to develop applications by using an in-memory embedded database.
    > 
    > Spring Boot can auto-configure embedded [H2](https://www.h2database.com/), [HSQL](https://hsqldb.org/), and >[Derby](https://db.apache.org/derby/) databases.
    > 
    > You need not provide any connection URLs. You need only include a build dependency to the embedded database that you want to use
    > 
    > [SpringBoot Embedded Database 설명](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql.datasource.embedded)
    > 

### 4. DataSourceAutoConfiguration ??

- Spring Boot 에서 auto-configure 설정 소스
    - [gitHub 소스 ( v2.7.15  )](https://github.com/spring-projects/spring-boot/blob/v2.7.15/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jdbc/DataSourceAutoConfiguration.java)
- DataSource 와  ConnectionPool 관련 추가 설명
    - [https://escapefromcoding.tistory.com/737](https://escapefromcoding.tistory.com/737)

## 결론

---

<aside>
💡 위에서 확인된 내용을 가지고 아래와 같이 결론을 내릴 수 있다.

</aside>

- 최종 YAML 파일 설정
    
    ```yaml
    spring:
      datasource: 
        url: ***
        driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
        username: ***
        password: ***
      hikari:
        connectionTimeout: ***
    ```
    
    > H2 dependency 삭제
    > 

- DataSourceAutoConfiguration 을 사용 안해야 하는 경우
    
    ```java
    @SpringBootApplication(exclude={DataSourceAutoConfiguration.class})
    
    or
    
    @EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
    ```
    

- HikariCP를 사용하지 못하는 경우
    - 아래 예시는 Tomcat pool 을 사용하는 경우
        
        ```yaml
        spring:
          datasource: 
            url: ***
            driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
            username: ***
            password: ***
          tomcat:
            max-wait: ***
        ```
        
        > [SpringBoot 에서 지원하는 ConnectionPool](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql.datasource.connection-pool)
        > 
- DataSource 를 여러 개 설정해야 하는 경우
    - 별도로 DataSource 를 구현
    - DataSource 별로 사용 할 ConnectionPool 또한 정의할 수 있다
    - [https://www.baeldung.com/spring-boot-configure-multiple-datasources](https://www.baeldung.com/spring-boot-configure-multiple-datasources)

## 참고자료

---

- [https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#data.sql)
- [https://donghyeon.dev/spring/2020/08/01/스프링부트의-AutoConfiguration의-원리-및-만들어-보기/](https://donghyeon.dev/spring/2020/08/01/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8%EC%9D%98-AutoConfiguration%EC%9D%98-%EC%9B%90%EB%A6%AC-%EB%B0%8F-%EB%A7%8C%EB%93%A4%EC%96%B4-%EB%B3%B4%EA%B8%B0/)