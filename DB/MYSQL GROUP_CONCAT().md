# MYSQL GROUP_CONCAT() 이슈

> 💡 mysql 쿼리에서 group_concat() 사용하는 경우에 발생했던 오류를 공유하고자 한다
> 

## 1. 발생 현상


> 💡 쿼리에서 `GROUP_CONCAT`() 함수를 사용하여 데이터를 변환하는데 특정 길이 이후에는 <br>
> cut off(짤림) 현상이 되는 것을 발견하였다.

## 2. 원인

```sql

-- DB에서는 group_concat 의 최대 길이가 제한되어 있다.
-- 기본값은 1024 이다 

show variables like '%group_concat_max_len%'

```

> 위와 같은 세팅이 되어 있기에 짧림 현상이 발생함.
> 

## 3. 해결방법

1. DB 의 서버 설정에 해당 변수 [ group_concat_max_len ] 값을 크게 세팅하면 된다
2. SESSION 별로 별도 설정하여 처리한다. 

---

> 💡 위의 해결 방법 중 2번 항목으로 처리하는 방식을 다룬다.  아래 글을 읽어보고 2번 방식으로 결정 함.
> [http://www.gurubee.net/article/81272](http://www.gurubee.net/article/81272)

- 문제가 발생되는 select query 조회하기 전에 아래 쿼리를 실행한 뒤에 select query 를 조회한다.

```sql
<insert id="regSessionVariableGroupConcatMaxLen" parameterType="int">
	<!-- group concat max len 변수 를 Session 단위로 관리한다. -->
	SET SESSION group_concat_max_len = #{maxLen}
</insert>
```