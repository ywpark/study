# Redis 가 왜 빠를까?

---

# Redis 란?

---

- Remote Dictionary Server
- In-Memory 데이터 구조 저장소

# 특징

---

### 1. In-Memory 데이터 스토리지

- 모든 데이터를 메인 메모리(RAM)에 저장하기 때문에 DISK I/O 가 발생하지 않아서 매우 빠른 읽기 및 쓰기 성능 제공

### 2. Key-Value 데이터 모델

### 3. 다양한 데이터 구조 지원

### 4. Single-Threaded 모델

- 복잡한 동기화 문제와 Lock 오버헤드를 피할 수 있음

### 5. 영속성(Persistence) 지원

- RDB (Redis Database) : 특정 시점에 메모리 데이터를 바이너리 파일로 스냅샷 저장
- AOF (Append Only File) : 모든 쓰기 명령을 로그 파일에 기록하기 떄문에 데이터 복구가 가능함
    - AOF 설정이 되어 있는 상태에서  `flushall` 명령어를 사용해서 데이터가 삭제되더라도
    AOF 파일을 열어서 `flushall` 삭제 후 Redis 를 재시작하면 복원이 가능

### 6. 확장성(Scalability) 과 고가용성(High Availability)

- 복제 (Replication) : 마스터-슬레이브 구조가 가능
- Sentinel
    - Redis 모니터링
    - Fail-Over 서비스
    - Master 를 감시하여 문제가 발생한 경우 다른 Slave 를 Master 로 승격 시킴
        - 기존 Master 가 정상화 되어도 Master 가 아닌 Slave 로 됨
    - 쿼럼(Quorum) : Sentinel 이 마스터가 다운되었다고 판정하는 기준
        - Quorum 2 : 최소 2 대이상의 Sentinel 이 마스터가 다운되었다고 인지해야만 Fail-Over가 진행됨
- Cluster : 여러 Redis 노드에 데이터를 분산 저장 가능

# 사용 사례

---

- Caching
- Session Management
- Message Broker
- Distributed Lock
- Sorted Sets 을 활용한 실시간 순위 혹은 카운터

# 이유

---

### 1. I/O Multiplexing ( + Event Loop )

- `Blocking I/O` 모델의 경우에는 `multi-process` `multi-thread` 로 입출력 다중화를 구현할 수 있지만 복잡한 이슈가 생기기 때문에 성능상 이점이 없다 ( `IPC, 동기화(Semaphore, Mutex)` )
- `Multiplexing` 모델의 경우에는 하나의 스레드에서 다수의 클라이언트에 연결된 소켓(FD, 파일 디스크립터)을 관리하면서 소켓에 이벤트(read/write)가 발생할 때만 해당 이벤트를 처리하도록 구현함으로써 더 적은 리소스를 사용.
- 기본은 `select()` 라는 함수를 호출해서 여러 개의 소켓 파일 중 read 함수로 데이터 호출이 가능한 소켓이 생길 때까지 `대기`
- 대기 하는 방식에 따른 기법
    - `select()` : 어떤 FD 에서 이벤트가 발생했는지 확인할기 위해서 fd_set 테이블 전체 검사 
                          `O(n)`
    - `poll()` :  실제 FD 개수만큼만 loop 를 검사하여 이벤트 발생을 체크하기 때문에 select() 보다
                    는 조금 나을 수는 있음 `O(n)`
    - `epoll()` : `select()` `poll()` 과 달리 FD의 상태가 kernel에서 관리하여 상태가 바뀐 것을
                    직접 통지해주는 방식. 즉 fd_set 을 검사하기 위해 loop를 돌 필요가 없다 `O(1)`
        - Edge Trigger 방식
        - Level Trigger 방식

### 2. 최적화된 데이터 구조

- 내부적으로 매우 효율적이고 최적화된 자료구조를 구현
- 시간 복잡도 (Time Complexity)
    - 대부분의 핵심 연산들이 O(1), O(log N)  으로 설계
- 문자열(String)
    - SDS ( Simple Dynamic String ) 자체 문자열 자료 구조를 사용
        - 같이 할당(embstr)
        - 각각 할당(raw)
- 리스트(List)
    - Doubly Linked List
    - Zip List
    - Quick List
- 해시(Hash)
    - 필드-값 페어 컬렉션
- Set
    - 일반적인 Set 자료 구조와 동일
    - `SMEMEBERS`  명령어의 경우 `O(N)` 이므로 주의해야함
- Sorted Set
    - ZSet 이라고도 불림
    - Skip List
    - Zip List

# 기타

---

<aside>
💡

I/O 작업 이란 ?
 : user space(User Mode) 에서 직접 수행할 수 없기 때문에 user process 가 
  kernel(Kernel Mode) 에 시스템 콜을 하여 하드웨어 장치와 통신.

</aside>

### 1. Synchronous/Asynchronous

- Synchronous
    - 호출 후 작업이 완료될 때까지 기다렸다가 다음 작업 진행
- Asynchronous
    - 호출 후 작업 완료를 기다리지 않고 다음 작업 진행

### 2. Blocking/Non-Blocking

- Blocking
    - 호출한 함수가 반환될 때까지 호출자(스레드)의 제어권이 넘어가서 다른 일을 할 수 없음.
- Non-Blocking
    - 호출한 함수가 즉시 반환되어 호출자(스레드)가 다른 일을 할 수 있음.

# 참고자료

---

[https://blog.naver.com/n_cloudplatform/222189669084](https://blog.naver.com/n_cloudplatform/222189669084)

[https://blog.naver.com/n_cloudplatform/222255261317](https://blog.naver.com/n_cloudplatform/222255261317)

[https://loosie.tistory.com/872](https://loosie.tistory.com/872)