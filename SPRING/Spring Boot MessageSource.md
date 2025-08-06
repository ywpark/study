# Spring Boot MessageSource 이슈

> 💡 Linux 에 프로젝트 배포 후에 message.properties에 정의한 코드값을 가져오지 못하는 
> 상황 발생
> 
> Windows 에서 build 테스트를 해보면 정상 <br>
> Linux 에서 build 테스트를 해보면 코드 값을 가져오지 못하는 상황

- 시스템 환경
    - Spring Boot 2.6.7
    - openjdk 11
    - Ubuntu
    - jenkins
    - embedded tomcat

- Spring Boot 에서 Message Config

```java
@Configuration
public class MessageConfig {

    @Bean
    public MessageSource messageSource() {
        
        ReloadableResourceBundleMessageSource messageSource = new ReloadableResourceBundleMessageSource();

        /**
         * beanName 아래와 같이 한 이유는 SpringBoot에서 message properties 를 읽을 때
         * 아래 정의된 이름에 최종 Lcale 에 따른 문자열을 붙인다.
         * 기본은 Locale 정보가 없는 properties 파일이다.
         *
         * 예) classpath:/messages/messages
         *  1. classpath:/messages/messages.properties
         *  2. classpath:/messages/messages_ko.properties
         *  3. classpath:/messages/messages_ko_KR.properties
         *
         */
        messageSource.setBasename("classpath:/messages/messages");
        messageSource.setDefaultEncoding("UTF-8");
        messageSource.setCacheSeconds(60);
        
        return messageSource;
    }
  
}
```

- MessageSource 에서 메세지 가져오는 부분

```java
public class MessageModule {

	private final MessageSource messageSource;

		/**
     * 한국어 Message 처리
     * @param messageCode messageCode 
     * @param parms Message Param 
     * @return 한국어 메세지 텍스트
     */
    public String getMessage(@NotEmpty String messageCode, String...params) {
        return messageSource.getMessage(messageCode, params, "정의되지 않은 코드입니다", Locale.KOREAN);
    }
}
```

- Resourece 파일

```java

//아래와 같은 위치에 파일을 만듬
resources/messages/messages_ko_KR.properties 
```

- 오류 로그

```
No message found under code 'OK00004' for locale 'ko'.

org.springframework.context.NoSuchMessageException: No message found under code 'OK00004' for locale 'ko'.
at org.springframework.context.support.AbstractMessageSource.getMessage(AbstractMessageSource.java:161)
```

---

# 해결방법

1. 실제 message 를 정의한 properties 에 코드의 존재 유무 체크
2. 다국어 처리 시에 사용하는 정의된 텍스트가 맞는지 체크 ( ko or ko_KR 등 )
3. 위의 2가지 경우도 아니고 Windows 에서 테스트시에 문제가 없는 경우에는 아래 방법 체크

```java

//1번 대안
//MessageSource 에서 Locale 부분을 변경
messageSource.getMessage(messageCode, params, "정의되지 않은 코드입니다", Locale.KOREA);

//2번 대안
//Message Properties 파일의 이름을 아래와 같이 변경
messages_ko.properties
```

---

## 원인

```

1. 원인은 Locale.KOREAN 과 Locale.KOREA 의 차이를 정확하게 모르기에 발생된 문제
2. 둘의 차이점은 country 부분이 "" 이냐 "KR" 로 정의가 되어 있느냐의 차이이다.
3. 문제는 "" 인 경우 즉 Locale.KOREAN 의 경우에는 JVM 에서 기본 Locale 정보를 가져와서
  Spring 에서 메세지 properties 를 찾을 때 활용한다.
4. Windows의 경우에는 JVM 옵션이 KR 로 되어 있기에 기존 Locale.KOREAN 으로 해도 문제가 
  없으나 리눅스 서버의 경우에는 JVM Locale 설정이 US 로 되어 있어서 발생된 문제였다.
5. JVM 옵션 확인 방법은 아래와 같다.
   log.info("JVM Language : " + System.getProperty("user.language"));
   log.info("JVM Locale : " + System.getProperty("user.country"));
```