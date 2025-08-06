# Spring graceful 적용

> 💡 Spring Application 을 현재는 Kill -9 pid 형식으로 프로세스를 종료하고 있다. 기존에는
> 생각하지 못했는데 이렇게 되었을 때 위험성이 크다고 판단.
> 
> Spring Boot 에서 지원하고 있는 graceful 기능을 활용해보고자 한다.

---

- 서버 환경
    - Spring Boot  2.6
    - OpenJdk 11
    - Jenkins

> 💡 jar 파일 구동 시 JVM RemoteDebug 옵션을 활용하고 있다 <br>
> 💡 Spring Boot 2.3 이전 버전에서는 아래와 같은 방식이 아닌 별도의 방식으로 구현해야 함.
> 이 부분에 대해서는 해당 글에서는 다루지 않는다.

---

## 1. Spring Boot 설정 하기

```yaml

# shutdown 방식 정의
server:
    shutdown: graceful # Default 값은 immediate 이다.

# Shutdown 시 아래에 정의한 시간만큼 Request처리를 한다.
# 즉, Request 처리하는데 정의한 시간보다 오래걸린다면 해당 Request 유실되기에
# 아래 시간은 충분한 검증 후 정의하는 것이 좋다
spring:
    lifecycle:
        timeout-per-shutdown-phase: 30s #Default 값은 30초이다
```

## 2. kill process 변경

```bash

# 현재 아래에서 사용하는 PID 는 아래와 같은 방식으로 찾고 있다.
# $(ps -ef | grep java | grep ABC* | awk '{print $2}')

#AS-IS
kill -9 pid

#TO-BE
kill -15 pid
```

---

## 현재 발견된 이슈 사항

1. 프로세스를 Kill 한 이후에 다시 프로세스가 올라오지 않는 현상 ( Jenkins 에서 로그 확인 가능 )

```bash
ERROR: transport error 202: bind failed: Address already in use
ERROR: JDWP Transport dt_socket failed to initialize, TRANSPORT_INIT(510)
JDWP exit error AGENT_ERROR_TRANSPORT_INIT(197): No transports initialized [./src/jdk.jdwp.agent/share/native/libjdwp/debugInit.c:735]
```

- 추측되는 원인

```
위에서 언급한 거 처럼 현재 어플리케이션 실행 시 Java Remote Debugging 옵션을 주고 있다.
이 부분에서 적용된 sokect 이 kill -15 pid 로는 kill 되지 않는 것으로 추측하고 있다
```

- 해결 방법

```

1. 2번 배포를 진행
 - 최초 Jenkins 에서 배포를 하면 기존 프로세스는 kill 되고 다시 프로세스가 안 올라온다.
 - 하지만 배포를 한번 더 해보면 정상적으로 프로세스가 올라오는 것을 확인 할 수 있다.

2. Java Remote Debugging 옵션 제거
 - jar 실행 시 해당 옵션을 제거하면 정상적으로 kill 되고 프로세스가 올라온다.

```