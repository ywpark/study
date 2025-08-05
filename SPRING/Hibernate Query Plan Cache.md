# CASE 1. Hibernate Query Plan Cache

---

<aside>
💡 OOM(Out Of Memory) 가 발생하였고 이 부분 확인 중 위와 같은 내용을 알게되어 정리하고
자 한다.

</aside>

- 개발 환경
    - SpringBoot 2.6.7
    - OpenJdk 11
    - Spring Data JPA 2.6.4
    - Hibernate 5.6.8
    
- HeapDump 내용 분석
    
    ![Heap 영역 차지 부분](/assets/hibernate_query_plan_cache/1.png)
    
    Heap 영역 차지 부분
    
    ![HashMap 항목 부분](/assets/hibernate_query_plan_cache/2.png)
    
    HashMap 항목 부분
    

> OOM 이 발생되는 원인으로 HashMap Object 영역이 너무 커서 이 객체를 원인으로 보고 
Hibernate 의 실행 계획 캐시 ( plan_cache ) 를 Memory 문제로 추측
> 

## 왜 Hibernate  내용이 나올까?

---

![Hirbernate Architecture 이미지](/assets/hibernate_query_plan_cache/3.png)

Hirbernate Architecture 이미지

- Hirbernate 는 JPA 의 구현체라고 볼 수 있다. 
( JAVA ORM 프레임워크 : Hibernate, EclipseLink, DataNucleus 등. )
- 위와 같은 이유로 인하여 JPA 를 사용하고 있다고 하지만 실제로는 Hibernate 또한 알고 있어야
한다.

## Hibernate Query Plan Cache ?

---

<aside>
💡 JPQL query 또는 Criteria query 는 AST ( Abstract Syntax Tree ) 로 파싱이 된다. 이때 Hibernate 에서는 쿼리 컴파일 시간이 걸리기 때문에 QueryPlanCache 를 제공한다.

Native query 의 경우에는 매개변수 및 쿼리 반환 유형에 대한 정보가 ParameterMetadata
에 저장된다.

모든 쿼리는 Hibernate 의 Plan Cache 를 확인하고, 없다면 Plan 을 캐시에 저장한다.

</aside>

> **query compilation takes time : 쿼리 컴파일 시간이 걸린다.**
> 

## Query Plan Cache Options ?

---

### 1. plan_cache_max_size

- plan 의 최대 개수를 정의
- Default 값은 2048

### 2. plan_parameter_metadata_max_size

- 캐시에서 관리되는 parametermetadata 개수를 정의
- Default 값은 128

### 3. in_clause_parameter_padding

- IN 절 쿼리를 사용하는 경우에 Plan Cache를 과하게 사용하는 경우가 존재
- 위의 설정을 default 값을 사용하더라도 혹은 Memory 를 넉넉하게 사용할 수 있더라도
이 설정은 신경 쓸 필요가 있음
- Default 값은 false
- Hiberante 버전이 5.2.18 이후라면 해당 옵션은 설정 필수
- 해당 옵션은 IN절과 관련된 Query 를 캐시를 할 때 2의 제곱단위로 생성하여 관리한다.

```java
# 아래와 같은 쿼리를 수행하게 되는 경우를 예로 한다.

SELECT * FROM A WHERE B IN ( 1 )
SELECT * FROM A WHERE B IN ( 1, 2 )
SELECT * FROM A WHERE B IN ( 1, 2, 3)
SELECT * FROM A WHERE B IN ( 1, 2, 3, 4)

# in_clause_parameter_padding = false
# 아래와 같이 4 개의 쿼리 캐시가 생성된다.

SELECT * FROM A WHERE B IN ( ? ) 
SELECT * FROM A WHERE B IN ( ?, ? ) 
SELECT * FROM A WHERE B IN ( ?, ?, ?)
SELECT * FROM A WHERE B IN ( ?, ?, ?, ?)

# in_clause_parameter_padding = true
# 아래와 같이 2개의 쿼리 캐시만 관리가 된다.
# 단순하게 생각해보면 IN 절에 1 인 경우에는 2의 제곱단위로 캐시를 만들기 위해 IN ( 1, 1 ) 
 이런식으로 값을 채워서 캐시를 관리한다. ( 메모리 비용 절약 )

SELECT * FROM A WHERE B IN ( ?, ? )
SELECT * FROM A WHERE B IN ( ?, ?, ?, ?)
```

## Hibernate 옵션 수치 정의

---

- 현재 JVM Heap Size 를 확인

```bash
# grep 부분은 heapsize 만 적용해도 데이터 확인은 가능
# Java 8 이후로는 permsize -> metaspace 이로 명칭 변경

# ~ Java 7
java -XX:+PrintFlagsFinal -version | grep -iE 'heapsize|permsize|threadstacksize'

# Java 8 ~
java -XX:+PrintFlagsFinal -version | grep -iE 'heapsize|metaspace|threadstacksize'
```

![3번.png](/assets/hibernate_query_plan_cache/4.png)

> 위에서 보면 MaxHeapSize 는 대략 480MB 로 볼 수 있다.
> 

<aside>
💡 따로 JAVA 실행 시 HeapSize 를 정의하지 않으면 Default 값으로 잡히는데 이 부분은 대략적으로 아래와 같은 기준에 의해서 잡힌다. 
( OS 환경에 따라 다르기에 정확한 값이 아닌 추정치 데이터로 활용 )

Initial heap size of 1/64 of physical memory up to 1Gbyte
Maximum heap size of 1/4 of physical memory up to 1Gbyte

EX) PC 메모리가 2GB 라고 한다면 아래와 같이 나온다.
Initial heap : 2 * 1024 / 64 = 32MB
Maximum heap : 2 * 1024 / 4 = 512MB

</aside>

- Hibernate 옵션 수치

<aside>
💡 해당 옵션은 정확한 계산 측정 방법은 없으며, 아래 계산 된 숫자는 개수를 의미한다.
메모리 용량은 별도로 측정해야 한다.

아래 지정한 숫자는 아래 예시 기준으로 default 수치를 계산함.
( HQLQueryPlan object occupies approximately 3MB )

Heap Max Size : 4GB
plan_cache_max_size : 1024 
plan_parameter_metadata_max_size : 64

</aside>

> plan_cache_max_size
> 

```bash
plan_cache_max_size = Heap Max Size / 4
                    = 480 / 4 = 120 개
                    = 이 수치는 대략 ~3 * 120 = 360MB
```

> plan_parameter_metadata_max_size
> 

```bash
plan_parameter_metadata_max_size = plan_cache_max_size / 16
                                 = 120 / 16 = 7.5
                                 = 대략 8
```

> in_clause_parameter_padding
> 

```bash
in_clause_parameter_padding = true
```

- Spring Boot 설정

```yaml
spring:
	jpa:
    properties:
      hibernate:
        query:
          plan_cache_max_size: 120 # 쿼리 계획 캐시의 최대 사이즈 ( default : 2048 )
          plan_parameter_metadata_max_size: 8 # Native Query ParameterMetadata 최대 사이즈 ( default : 128 )
          in_clause_parameter_padding: true # Where 조건에서 In절의 Padding Cache 사용 유무 ( default : false )
```

## 참고 링크

---

- [DBA와 개발자가 모두 행복해지는 Hibernate의 in_clause_parameter_padding 옵션 : NHN Cloud Meetup](https://meetup.nhncloud.com/posts/211)
- [java:hibernate:performance [권남] (kwonnam.pe.kr)](https://kwonnam.pe.kr/wiki/java/hibernate/performance)
- [JPA hibernate Query Plan Cache로 인한 OutOfMemory 해결 (velog.io)](https://velog.io/@recordsbeat/JPA-hibernate-Plan-Cache%EB%A1%9C-%EC%9D%B8%ED%95%9C-OutOfMemory-%ED%95%B4%EA%B2%B0)
- [https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html)
- [https://www.baeldung.com/hibernate-query-plan-cache](https://www.baeldung.com/hibernate-query-plan-cache)
- [https://docs.jboss.org/hibernate/orm/5.4/javadocs/org/hibernate/cfg/AvailableSettings.html#IN_CLAUSE_PARAMETER_PADDING](https://docs.jboss.org/hibernate/orm/5.4/javadocs/org/hibernate/cfg/AvailableSettings.html#IN_CLAUSE_PARAMETER_PADDING)