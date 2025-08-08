# QueryDsl JOIN 서브쿼리 사용

> 💡 querydsl 에서 서브쿼리를 사용한 내용을 정리한다.
> 

## 1. 문제상황

---

> LEFT JOIN 절에 별도의 SubQuery가 필요한 상황이 발생.
> 

```sql
# 하고자 하는 쿼리 형식

SELECT 
	  a.id
, 	  a.nm
,	  IF(c.id is null, '미존재', '존재')
FROM A as a
INNER JOIN B as b on a.id = b.id
LEFT JOIN (
	SELECT
		id
	FROM C
	WHERE 
		type = '01'
	GROUP BY 
		id
	ORDER BY  
		null
) as c on a.id = c.id
```

## 2. 현재상황

---

> 기존 쿼리가 querydsl 로 만들어져 있음
> 

```java
var list = queryFactory
			    .select(Projections.constructor(TestDTO.class,
			             a.id
			            ,a.nm
			            ,a.txt
			            ))
			    .from(a)
			    .innerJoin(b).on(a.id.eq(a.b.id))
			    .where(
			            a.delYn.eq(DelType.NOT_DELETE),
			            eqAprv(param.getAprvType()),
			            eqSearchTxt(param.getSearchType(), param.getSearchTxt()),
			            eqProductType(param.getPrdctType()),
			            a.chfAcntYn.eq(YNType.YES)
			          )
			    .orderBy(a.nm.asc())
			    .offset(pageable.getOffset())
			    .limit(pageable.getPageSize())
			    .fetch();
```

- QueryDsl 에서는 서브쿼리가 SELECT , WHERE 절 에만 적용 가능한 것으로 확인
- SELECT 절에 SubQuery를 넣을 필요없다고 판단.
- LEFT 절에 필요한 SubQuery 를 별도의 JAVA 로직으로 풀 수 있지만, 그럴만한 데이터가
아니라고 판단하여 queryDsl 방식으로 접근

## 3. 해결방법

---

> Hibernate 에서 제공하는 Subselect 를 활용하였음
> 
1. Subselect 를 쓰기 위하여 별도의 Entity 를 생성

```java

@Getter
@Entity
@Subselect(
        "SELECT " +
        "id " +
        "FROM A " +
        "WHERE type = \"01\" " +
        "GROUP BY id " +
        "ORDER BY NULL "
)
@Immutable
@Synchronize("A")
public class ASubQuery {

    @Id
    @Column(name = "id")
    private String id;
}
```

> `@Subselect` : 쿼리 결과를 Entity로 매핑할 수 있도록 지원해주는 기능 <br>
>             ( VIew 와 같이 read 만 한다 ) <br>
> `@Immutable` : Entity 를 매핑하다보니 데이터를 변경하면 하이버네이트에서 update를 할 수 
>                       있는데 이 부분을 방지하기 위한 설정 <br>
> `@Synchronize` : 정의한 테이블의 변경 이력 확인 ( 변경이 된 내역이 있다면 flush 하고 조회 )
> 
- 참고자료
[https://leejaedoo.github.io/repository_select_jpa/](https://leejaedoo.github.io/repository_select_jpa/)

1. 1번에서 만든 Entity를 LEFT JOIN 절에서 활용

```java
var list = queryFactory
			    .select(Projections.constructor(TestDTO.class,
			             a.id
			            ,a.nm
			            ,a.txt
			            ,aSubQuery.id
			            ))
			    .from(a)
			    .innerJoin(b).on(a.id.eq(a.b.id))
          		.leftJoin(aSubQuery).on(a.b.eq(aSubQuery.id))
			    .where(
			            a.delYn.eq(DelType.NOT_DELETE),
			            eqAprv(param.getAprvType()),
			            eqSearchTxt(param.getSearchType(), param.getSearchTxt()),
			            eqProductType(param.getPrdctType()),
			            a.chfAcntYn.eq(YNType.YES)
			          )
			    .orderBy(a.nm.asc())
			    .offset(pageable.getOffset())
			    .limit(pageable.getPageSize())
			    .fetch();
```

## 4. 개선사항

---

```java
IF(c.id is null, '미존재', '존재')

위의 부분을 querydsl 로 풀어보려고 했으나 원하는 내용을 가진 method 를 발견 못했다.
현재는 DTO 에서 처리하도록 되어 있다.
```

## 5. 추가사항

---

> 💡 Ver.24-04-02
> 

- `@Subselect` 구문을 사용할 때 조건 절 `parameter` 가 고정 값이 아니라 변수 형태가 필요한 경우
    - 자료를 찾아보면 `@Subselect` , `@Filter` , `@FilterDef` 를 결합하여 해결한 형태가 보이는데
    이건 공식적으로 지원하지 않는 기능이라고 함
        
        > [https://discourse.hibernate.org/t/adding-parameters-to-a-subselect-entity-using-filterdef-in-hibernate-6-3-1/9157](https://discourse.hibernate.org/t/adding-parameters-to-a-subselect-entity-using-filterdef-in-hibernate-6-3-1/9157)
        > 
    - 이런 경우에는 `@NamedNativeQuery` , `@SqlResultSetMapping`  을 활용하여 처리할 수 있다
        
        > [https://vladmihalcea.com/jpa-sqlresultsetmapping/](https://vladmihalcea.com/jpa-sqlresultsetmapping/)
        > 
    - 24.04.02 기준 별도로 테스트는 못함