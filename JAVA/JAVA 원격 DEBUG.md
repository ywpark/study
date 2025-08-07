# JVM 원격 DEBUG 세팅

> 💡 Java Application Remote Debugging 이란 ?
> Remote Host 운영되고 있는 JAVA application 을 Local Host 에서 Debug 를 한다.

> 💡 Remote Debug 포트는 8765 로 정의
> 

- 환경
    - Ubuntu
    - JAVA 11
    - SpringBoot ( Embeded Tomcat )
    - Intellij

## STEP 1. JAVA 버전 확인 및 실행 Option 확인

```

- JAVA 5 ~ 8 이후는 아래 옵션을 적용한다.
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8765 

- JAVA 9 ~ 이후는 아래 옵션을 적용한다.
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8765 

- JAVA 5 는 아래 옵션을 적용한다.
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8765
```

> 💡 JAVA 5는 확인해보지 않음.
> 

> 💡 suspend=y 자바 어플리케이션이 IDE 연결될 때까지 기다렸다가 연결되면 실행
> suspend=n 자바 어플리케이션이 일반적으로 실행되지만, breakpoints에서 멈춘다.
> 

> 💡 JVM option 중에 address 부분이 있는데 이 부분은 JAVA 버전에 따라 다른것으로 확인이
> 된다. ( intellij 옵션으로 확인함 )
> 

## STEP 2. IntelliJ 세팅 하는 법

1. Run > Edit Configurations 메뉴로 이동
2. [ + ] 버튼을 눌러서 Remote JVM Debug 를 추가한다.

![1.png](/assets/java/1.png)

1. 아래와 이미지와 같이 Host 와 Port 주소를 입력한다.
Remote Host 에 세팅한 값을 입력해야 한다.

![2.png](/assets/java/2.png)

1. 그 후에 Debug 로 실행하면 아래와 같이 연결이되면서 Debugging 을 할 수 있다.

![3.png](/assets/java/3.png)

---

## 주의 사항

> 💡 Remote HOST 에 있는 JVM 에 debugging 을 하기 때문에 해당 서버에 붙는 어플리케이션이
> 다 같이 Debugging 이 걸릴 수 있으니 주의해야 한다.
> 
> suspend 옵션의 경우 특별한 경우가 아니라면 N 으로 기본설정을 하는게 좋다

---

- 참고 자료

[https://www.baeldung.com/java-application-remote-debugging](https://www.baeldung.com/java-application-remote-debugging)

[https://mahmoudanouti.wordpress.com/2019/07/07/remote-debugging-java-applications-with-jdwp/](https://mahmoudanouti.wordpress.com/2019/07/07/remote-debugging-java-applications-with-jdwp/)