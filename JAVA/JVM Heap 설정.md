# JVM Heap 설정

---

> 💡 JVM Heap 메모리 정의 방식 정리
> 아래 기준으로 수치 정의
> 

### 1. EC2 의 현재 메모리 확인

```bash
free -m or free -h

# m 옵션은 메가바이트 단위로 보여준다.
# h 옵션은 사람이 읽기 쉽게 보여준다
```

![4.png](/assets/java/4.png)
> 전체 메모리는 1851MB 로 확인
각 항목 별 의미는 아래 참고 링크에서 확인해 볼 수 있다
> 



### 2. JVM 기본 메모리 상태

```bash
# Java 8 이후로는 permsize -> metaspace 이로 명칭 변경

# ~ Java 7
java -XX:+PrintFlagsFinal -version | grep -iE 'heapsize|permsize|threadstacksize'

# Java 8 ~
java -XX:+PrintFlagsFinal -version | grep -iE 'heapsize|metaspace|threadstacksize'
```

> 위에서의 변화로 가장 큰 것은 Permsize 를 관리하던 것이 JVM Heap 영역이었는데 MetaSpace 로 변경되면서 Native Memory 가 관리하는 부분
> 

![5.png](/assets/java/5.png)

> 위와 같이 Default 값을 확인해 볼 수 있다. 기본값의 경우에는 할당하는 대략적인 계산 수치가 있는데<br> 
이 부분은 참고 링크에서 확인해 볼 수 있다.
> 

### 3. JVM Default HeapSize 정의

```bash
# 1. Total 메모리
total = 1851 MB

# 2. Default HeapSize
init = 대략 30 MB
max = 대략 454 MB

# 3. Default MetaSpaceSize
init = 대략 21 MB
max = 엄청 큼 ( 상한선을 정의하지 않으면 메모리를 계속 사용한다고 함 )

# 최종 값
Xms1g  > Heap Size 최초 크기
Xmx1g  > Heap Size 최대 크기
XX:MaxMetaspaceSize=256m > MetaSpace 최대 사이즈 정의
XX:NewSize=342m > Old 와 New 의 비율을 1:2 로 정의
XX:MaxNewSize=342m

# 최종 사용 메모리 계산수치
# JVM 자체가 사용하는 메모리 값이 보통 300 ~ 500 이라고 함
1024 + 256 + ( 300 ~ 500 ) : 대략 1780

# Total 메모리 기준으로 대략 70MB 정도의 메모리 여유 분이 존재하여
# 일단 위의 기본값으로 정의
```

## 시행착오

---

- MaxMetaspaceSize 를 먼저 체크하지 않고 발생한 문제

```bash
# 인터넷에서 조사하여 나온 기본 수치로 적용
XX:MaxMetaspaceSize=128m

# 위와 같이 적용하면 얼마 안있다가 서버가 죽는현상이 발생
# OOM 이 발생하여 heapDump 파일이 만들어져 있어서 그거로 분석하다가 힌트를 찾을 수 없어서
# 해당 메모리가 너무 적은것이 아닌가 판단하여 Heap 모니터링으로 Metaspace 영역을 모니터링
# 하던 중 아래와 같은 상황이 발생하면 서버가 죽은 것으로 판단
# 256m 으로 수치 변경 후에는 정상 작동하는 것으로 확인
```

![6.PNG](/assets/java/6.png)

## 참고 링크

---

- [https://pearlluck.tistory.com/107](https://pearlluck.tistory.com/107)
- [https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size)
- [https://epthffh.tistory.com/entry/JVM-메모리-관련-설정](https://epthffh.tistory.com/entry/JVM-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EA%B4%80%EB%A0%A8-%EC%84%A4%EC%A0%95)