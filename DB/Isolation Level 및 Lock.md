# Isolation Level 및 Lock 의 종류

---

> 💡 MySQL Transaction 을 정리하면서 관련 내용들을 전체적으로 정리하는 내용입니다.

## 0. 기준

---

- MySql 8.0 버전 기준
- 스토리지 엔진은 InnoDB 기준

## 1. Transaction 을 정의하는 4가지 속성 ( ACID )

---

- 원자성(Automicity)
    - 트랜잭션에 속한 여러 query 를 하나의 단위로 취급하여 전체 query 가 실행되거나 실행되지 않거나 둘 중 하나의 작업만 일어난다.
- 일관성(Consistency)
    - 트랜잭션이 테이블에 변경사항을 적용할 때 테이블 무결성이 깨지지 않도록 한다.
- 고립성(Isolation)
    - 각 각의 트랜잭션을 격리하여 서로 영향을 주어서는 안된다.
- 영속성(Durability)
    - 트랜잭션이 완료되면 DML 로 인한 변경이 저장되도록 보장한다.

## 2. Transaction Isolation Level

---

> 💡 여러 트랜잭션이 동시에 처리될 때, 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 table 을 볼 수 있도록 허용할지 말지를 결정한다.
> 

### 2-1. Isolation Level 종류

> 💡 아래로 갈 수록 고립(격리) 레벨이 높아지고, 동시성이 떨어진다.
> 

- `READ UNCOMMITED`
    - 커밋하지 않은 데이터도 접근 가능
    - (1)트랜잭션에서 COMMIT 되지 않은 데이터를 (2)트랜잭션에서 조회 및 접근이 되는데 혹여 (1)트랜잭션이 ROLLBACK 이 된다면 (2)트랜잭션이 조회 한 레코드는 존재하지 않게 되는데 이러한 상황이 `Dirty Read` 이며 치명적인 문제가 발생.
- `READ COMMITED`
    - 커밋된 데이터만 조회 가능
    - `Phantom Read` , `Non-Repeatable Read` 문제가 발생.
    - (1) 트랜잭션에서 SELECT 로 조회하고 (2) 트랜잭션에서 UPDATE 를 통해 데이터를 업데이트 후에 Commit 을 하게된 후에 (1) 트랜잭션에서 다시 SELECT 를 했을 때 다른 결과값을 가져올 수 있다.
- `REPEATABLE READ`
    - 일반적인 RDBMS 에는 변경 전의 레코드를 `언두 공간(Undo) 공간`에 백업을 하기 때문에 변경 전/후 데이터가 모두 존재 하므로, 동일한 레코드에 대해 여러 버전의 데이터가 존재한다고 하여 `MVCC(Multi-Version Concurrency Control)` 라고 부른다.
    - MVCC 를 이용해 한 트랜잭션 내에서 동일한 결과를 보장하지만, 새로운 레코드 추가되는 경우에는 부정합이 발생.
    - 각 각의 트랜잭션에는 고유한 트랜잭션 번호가 존재 하며 해당 정보는 백업 레코드(Undo) 공간에 함께 저장된다. 해당 데이터가 불필요해진다고 판단하는 시점에 백그라운드 쓰레드를 통해 삭제 된다.
    - (1)트랜잭션이 시작되면 번호가 부여되기 때문에 다른 (2)트랜잭션에서 동일한 레코드에 대해서 수정이 이루어지고 COMMIT 을 하더라도 (1)번 트랜잭션에서 동일한 레코드를 조회하더라도 (2)번 내용이 적용되지 않은 상태의 데이터를 동일하게 얻을 수 있다.
     ( 트랜잭션 아이디가 존재하기 때문에 (1) 트랜잭션에서 재 조회해도 동일한 데이터를 얻을 수 있다 )
    - 데이터가 추가되는 것을 막지 않기 때문에 SELECT로 조회한 경우 트랜잭션이 끝나기 전에 다른 트랜잭션에서 추가한 레코드가 SELECT 될 수 있는데 이것을 `Phantom Read` 라고 한다. MVCC 덕분에 일반적인 조회의 경우 Phantom Read 가 발생하지 않는다.
     하지만, 발생할 수도 있는데 잠금(Lock)을 사용하는 경우이다. 이 경우에는 일반적인 RDBMS 와 MySQL 을 비교해 볼 수 있다.
        - 일반적인 RDBMS
            - (1) 트랜잭션에서 `SELECT ... FOR UPDATE` 구문으로 베타적 잠금을 걸어서 데이터를 조회, 다른 (2)트랜잭션에서 INSERT 를 하려고 하면 따로 GAP LOCK 이 존재하지 않기 때문에 INSERT 가 되고 COMMIT 진행된다.
            - 그 뒤에 (1) 트랜잭션에서 다시 `SELECT ... FOR UPDATE` 로 데이터 조회를 하면 (2)트랜잭션에서 추가된 데이터가 조회된다 `( Phantom Read )`
              이 경우도 MVCC 때문에 문제 없을 것으로 보이지만 `SELECT FOR UPDATE` , `SELECT FOR SHARE` 의 경우에는 Undo 로그가 아닌 테이블에서 직접 조회를 하기 때문에 INSERT 된 데이터가 같이 조회가 된다. 언두 로그는 잠금 장치가 없기 때문이다.
        - MySQL
            - GAP LOCK 이 있기 때문에 해당 격리 레벨에서는 Phantom Read 가 발생하지 않는다. 오히려 `Lock Timeout` 이 발생될 수 있다.  위와 동일한 상태로 트랜잭션을 실행하면 (2)트랜잭션이 INSERT 가 안되고 Gap Lock 에 의해서 잠금 대기를 하고 (1) 트랜잭션은 두 번 모두 동일한 SELECT 결과값을 얻고 COMMIT 후에 (2)트랜잭션 또한 COMMIT이 된다.
    - `트랜잭션 안에서의 단순 SELECT 는 일관된 데이터 조회를 보장 받지만, 트랜잭션 없이 실행된다면 데이터의 정합성이 깨지는 순간이 발생될 수 있다.`
- `SERIALIZABLE`
    - 여러 트랜잭션이 동일한 레코드에 동시 접근할 수 없다.
    - `트랙잰션이 순차적으로 처리되어야 하므로 동시 처리 성능이 매우 떨어진다.`
    - 기본은 순수한 SELECT 작업에는 레코드 잠금이 없이 실행되지만, 해당 격리수준에서는
    NEXT KEY LOCK을 읽기 잠금(공유락, Shared Lock ) 를 건다.
    - 그렇기에 다른 트랜잭션에서 넥스트 키 락이 걸린 레코드에 추가/수정/삭제할 수 없다.
    - 가장 안전한 격리 수준이지만 성능이 매우 떨어지기에 특별한 상황이 아닌 경우에는
    써서는 안된다.

### 2-2. Isolation Level 에 따른 문제점 ( 부정합 문제 )

- `Dirty Read`
    - 한 트랜잭션에서 변경한 사항을 아직 COMMIT 하기 전임에도 불구하고 다른 트랜잭션에서 읽을 수 있는 현상
- `Non-Repeatable Read`
    - 단일 트랜잭션 내에서 반복된 조회의 결과가 다르게 나타나는 현상
- `Phantom Read`
    - 단일 트랜잭션 내에서 조회 결과가 다르게 나타나는 현상. `Non-Repeatable Read` 현상과
    비슷한 범주로 생각할 수 있다. 그러나 다른 트랜잭션의 `INSERT` 혹은 `DELETE` 로 인해 데이터가 보였다 안보였다하는 조회 결과가 나타나는 현상

## 3. MySQL Lock

---

> 💡 MySQL에서 사용하는 Lock은 크게 스토리지 엔진 레벨, MySQL 엔진 레벨로 나뉜다.
> 스토리지 엔진 레벨의 잠금은 테이블의 데이터를 다루기 위한 락이며, MySQL 엔진 레벨의
> 잠금은 테이블이나 데이터베이스 등과 같은 부분을 위한 Lock 이다.
> 

### 3-1. Shared and Exclusive Locks

> 💡 InnoDB 는 S locks, X locks 의 2 가지 타입이 있는 row-level lock 을 구현합니다. 
> 
> 원문) `InnoDB` implements standard row-level locking where there are two types of locks, [shared (`S`) locks](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_shared_lock) and [exclusive (`X`) locks](https://dev.mysql.com/doc/refman/8.4/en/glossary.html#glos_exclusive_lock).
> 

- `Shared lock ( S lock )`
    - 다른 트랜잭션에서 잠긴 객체를 읽을 수는 있지만, 쓰기는 불가능
- `Exclusive lock ( X lock )`
    - 다른 트랜잭션에서 접근하는 것을 막음. 데이터를 쓰는 경우 ( `INSERT`, `UPDATE` ) 에
    사용됨

### 3-2. Intention Locks

> 💡 테이블을 대상으로 하는 lock. 테이블 내에 Row, 페이지와 같이 더 낮은 수준의 lock을 걸지 
> 알려주는 lock 이다.

> EX)
>   트랜잭션 A 가 특정 ROW 에 X lock 을 걸고 트랜잭션 B 가 테이블 전체를 수정하고자 한다.
> B는 당연히 A가 끝날 때까지 기다려야 하는데 B 가 모든 ROW 에 대해서 lock 을 획득할 수
> 있는지 체크해야 하는 것은 상당히 비효율적이다.
> 그렇기에 A 가 X lock 을 거는 시점에 테이블에 IX lock 을 걸면 B는 모든 ROW를 체크하지 
> 않고도 lock 을 획득할 수 있는지 알 수 있다.

- `Intention shared lock ( IS lock )`
- `Intention exclusive lock ( IX lock )`

> Lock 사이의 호환성
아래 그림에서 `Conflict` 는 기존 트랜잭션에 걸려 있는 Lock 이 풀릴 때까지 기다려야함.
> 

![출처) [https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#innodb-intention-locks](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#innodb-intention-locks)](/assets/mysql_isolation/1.png)

출처) [https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#innodb-intention-locks](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#innodb-intention-locks)

### 3-3 스토리지 엔진 레벨

- `레코드 락 (Record Lock)`
    - 상당히 작은 공간으로 관리하기 때문에 레코드 락 -> 페이지 락 -> 테이블 락으로 범위가 
    상승되지는 않는다 ( 해당 현상은 락 에스컬레이션이라고 함 )
        - `Lock Escalation` ( 락 에스컬레이션 ) => Lock 의 범위를 상승시키는 현상
    - 중요 포인트는 테이블의 레코드가 아닌 `인덱스의 레코드를 잠근다`는 차이가 있다.
    - Where 절에 있는 condition 컬럼에 따라서 index 가 걸려있다면 해당 기준으로 검색된 데이터 전체가 레코드 잠금상태가 된다. 만약 적당한 인덱스가 없다면 모든 테이블의 레코드에 Lock 을 걸고 테이블을 풀스캔 하면서 작업을 처리.
    - PK가 없는 테이블이라면 내부적으로 자동 생성된 PK를 이용해 설정

- `갭 락 (Gap Lock)`
    - 개념일 뿐 자체적으로 사용되지 않으며 Next Key Lock 의 일부로 활용
    - 레코드와 레코드 사이의 간격을 잠금으로써 레코드의 생성, 수정, 삭제를 제어
    - 현재 트랜잭션에서 조회를 할 때, 다른 트랜잭션에서 임의의 데이터가 추가되지 않도록 
     잠그려면 `SELECT ... FOR UPDATE`,  `LOCK IN SHARE MODE` 구문을 활용
      Lock은 트랜잭션이 커밋 또는 롤백 될 때 해제된다.
        - SELECT ... FOR UPDATE  -> 베타적 잠금(비관적 잠금, 쓰기 잠금)
        - LOCK IN SHARE MODE -> 읽기 잠금
    - 인덱스 범위 조건 중에서 실제 레코드를 제외하고, 데이터가 추가될 수 있는 범위에 걸리게 된다.
        - 데이터를 조회할 때 1<= X <= 10 이렇게 조회하고 실제 인덱스 레코드에는 4,5,6 데이터만 있는 상황
            - X = 4,5,6 :: 레코드 락
            - X = 1,2,3,7,8,9,10 :: 갭 락
        - 갭 락은 아직 존재하지는 않지만 지정된 범위에 해당하는 인덱스 테이블 공간을 대상으로 거는 잠금
    - `데이터에 유일성이 보장되는 작업에는 갭 락이 발생되지 않는다`.
- `넥스트 키 락 (Next Key Lock)`
    - 레코드 락과 갭 락을 합친 잠금
    - binary log 에 기록되는 query 를 리플리카(slave) 서버에서 실행 될 때, 소스(master) 에서 만들어낸 결과를 동일하게 만들도록 보장하기 위한 목적
    - Next key lock 과 gap lock 으로 인하여 dead lock 이 생각보다 자주 발생.
        - binary log 포맷을 ROW 형태로 바꾸면 Next Key lock ,Gap lock 을 줄일 수 있다고 하나
        안전성은 아직 확인이 어렵다.
- `자동 증가 락 (Auto Increment Lock)`
    - 흔히 사용하는 AUTO_INCREMENT 컬럼이 적용되어 있는 테이블에 INSERT 혹은 REPLACE 와 같이 신규로 레코드가 저장되는 쿼리에만 사용
    - 중복되지 않고 순차적으로 데이터가 생성되기 때문에 테이블 수준의 잠금이라고 볼 수 있음
    - 대부분의 경우 잠깐 실행되었다가 Lock 이 즉시 해제 되기 때문에 크게 문제가 되는 
    경우는 없음
    - 잠금을 최소화하기 위해 한번 증가하면 절대 자동으로 줄어들지 않는다. 즉, 트랜잭션이 실패하여 롤백되어도 증가된 값은 복구되지 않고 유지된다.

### 3-4. Mysql 엔진 레벨

> 💡 모든 스토리지 엔진에 영향을 미친다.
>

- `글로벌 락 ( Global Lock )`
    - `FLUSH TABLES WITH READ LOCK` 명령으로 lock 을 획득할 수 있음
    - MySQL 서버에 존재하는 모든 테이블에 lock 을 건다.
    - 다른 트랙잭션에서 `SELECT` 를 제외한 DDL, DML 문장은 lock 이 풀릴 때까지 대기
- `테이블 락 ( Table Lock )`
    - 개별 테이블 단위로 lock 을 건다
    - 명시적 또는 묵시적으로 lock 을 획득할 수 있다
    - `LOCK TABLES table_name [ READ | WRITE ]`  명령어를 통해 명시적으로 lock 을 획득
    하게 되면 `UNLOCK TABLES` 명령어로 lock 해제 처리 해야함
    - `InnoDB 엔진은 기본 Record Lock 이기 때문에 영향이 미비`
- `네임드 락 ( Named Lock )`
    - `GET_LOCK()` 명령어를 통해 임의의 문자열에 대한 잠금을 설정할 수 있으며
    `RELEASE_LOCK()` 을 통해 잠금을 해제
    - 사용자가 지정한 문자열에 대해서 획득하고 해제한다.
        - 대상이 테이블이나 레코드, 데이터베이스 객체가 아니다.
- `메타데이터 락 ( Metadata Lock )`
    - 데이터베이스 객체(테이블, 뷰) 의 이름이나 구조를 변경하는 경우에 획득
    - 명시적으로 획득 불가
    - 테이블의 이름을 변경하는 경우 자동으로 획득
        - 원본 과 변경될 이름 2개 모두 잠금 처리

## 4. 참고 자료

---

- [https://hudi.blog/transaction-isolation-level/](https://hudi.blog/transaction-isolation-level/)
- [https://mangkyu.tistory.com/298](https://mangkyu.tistory.com/298)
- [https://cano721.tistory.com/213](https://cano721.tistory.com/213)
- [https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#innodb-intention-locks](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html#innodb-intention-locks)
- [https://chanos.tistory.com/entry/MySQL-DB의-동시성-제어를-위한-Lock과-MVCC](https://chanos.tistory.com/entry/MySQL-DB%EC%9D%98-%EB%8F%99%EC%8B%9C%EC%84%B1-%EC%A0%9C%EC%96%B4%EB%A5%BC-%EC%9C%84%ED%95%9C-Lock%EA%B3%BC-MVCC)