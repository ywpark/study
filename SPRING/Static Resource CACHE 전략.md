# Static Resource CACHE 전략

---

> 💡 SpringBoot 에서 지원해주는 설정을 통하여 JS, CSS 파일의 CACHE 관리를 좀 더 편하게
> 하고자 작성함
> 

## 환경

---

- SpringBoot 2.7.15
- Thymleaf

## 1. QueryString 을 사용한 CACHE 처리

---

```html
<head>
	<script th:src="@{/front/utils/common-backend.js?v=${property}}"></script>
	<script th:src="@{/front/utils/webview-api.js?v=1234}"></script>
</head>
```

> `JavaScript` 혹은 `css` 파일 뒤에 `queryString` 을 붙여서 Client 에 `CACHE` 되어 있는 파일을 갱신하는 방법을 일반적으로 사용한다.
> 

## 2. SpringBoot 에서 제공하는 Resource Versioning

---

> 💡 아래 설정은 Thymeleaf, FreeMaker 등을 사용하면 auto-configured 를 지원하지만 JSP 등
> SpringBoot 에서 지원하지 않는 템플릿 엔진을 사용하는 경우 별도 구성을 해야 함

- JSP 인 경우 : `ResourceUrlEncodingFilter`  별도로 해당 필터 설정
- 다른 Template 인 경우 : `ResourceUrlProvider`  를 활용
- 기본적인 정보는 아래 SpringBoot 문서에서 확인 가능
    - [https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.servlet.spring-mvc.static-content](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.servlet.spring-mvc.static-content)

### 2-1. chain.strategy.content 정의

```yaml
spring:
  web:
    resources:
      static-locations: classpath:/front # resource 파일 위치 재 정의
      chain:
        strategy:
          content:
            enabled: true 
            paths: "/utils/**"
      cache:
        period: 36000 # cache 유지시간(max-age)(단위 : 초)

```

> 해당 방식은 resource content 를 Hash 값으로 변환하여 활용
> 
- `spring.web.resources.chain.strategy.content.enabled`
    - cotent 캐시 전략 사용 유무
    - default : false
- `spring.web.resources.chain.strategy.content.paths`
    - cotent 캐시 전략을 적용 할 resource 경로
    - default : /**
- 인터넷 자료에 보면 `spring.resource.chain.strategy.content.enable`  설정이 많이 보이는데
이거는 SpringBoot 2.4 에서 위와 같은 설정으로 변경
    - [https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.4.0-RC1-Configuration-Changelog](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.4.0-RC1-Configuration-Changelog)
- 형식 비교
    
    ```
    
    [AS_IS]
    
    http://localhost:8080/front/utils/common-backend.js
    
    [TO_BE]
    
    http://localhost:8080/front/utils/common-backend-d87acd10b8aa3067aa6f43dd4b41ea15.js
    
    ```
    
    > Script 내용을 변경하지 않는다면 뒤에 붙은 Hash 값은 변경되지 않아서 개발자가 별도로 신경 쓰지 않아도 된다.
    > 

### 2-2. chain.strategy.fixed 정의

```yaml
spring:
  web:
    resources:
      static-locations: classpath:/front # 별도 위치면 지정
      chain:
        strategy:
          fixed:
            enabled: true
            paths: "/utils/**"
            version: "v1"
      cache:
        period: 36000 # cache 유지시간(max-age)(단위 : 초)
```

> static resource URL 경로에 특정 Version String 을 붙여서 관리하는 전략
> 
- `spring.web.resources.chain.strategy.fixed.enabled`
    - fixed 캐시 전략 사용 유무
    - default : false
- `spring.web.resources.chain.strategy.fixed.paths`
    - fixed 캐시 전략을 적용 할 resource 경로
    - default : /**
- `spring.web.resources.chain.strategy.fixed.version`
    - resource URL 에 적용 할 version string 정의
- 형식 비교
    
    ```
    
    [AS_IS]
    
    http://localhost:8080/front/utils/common-backend.js
    
    [TO_BE]
    
    http://localhost:8080/front/v1/utils/common-backend.js
    ```
    
    > 정의된 resource path 에 모두 적용되기 때문에 개별 파일로는 관리가 되지 않으며 version 
    string 을 개발자가 관리해야 함
    > 
    

> 💡 위의 설정을 둘 다 적용하고 paths 부분을 동일한 값으로 정의하면 [ strategy.content ] 전략을 따라가는 것으로 확인됨

## 3. NEXT

---

> 💡 현재 서비스에는 필요 없을 거 같아서 따로 알아보지는 않은 내용이지만 추후에 고민할 
> 필요는 있어 보임

- HTTP ETag
    - [https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/ETag](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/ETag)
    - [https://yozm.wishket.com/magazine/detail/1772/](https://yozm.wishket.com/magazine/detail/1772/)
    - [https://velog.io/@tigger/ETag-with-Spring](https://velog.io/@tigger/ETag-with-Spring)
    

## 4. 참고자료

---

- [https://shortstories.gitbook.io/studybook/spring_ad00_b828_c815_b9ac/springc5d0-c11c-static-resource-versioning-d558-ae30](https://shortstories.gitbook.io/studybook/spring_ad00_b828_c815_b9ac/springc5d0-c11c-static-resource-versioning-d558-ae30)
- [https://toss.tech/article/smart-web-service-cache](https://toss.tech/article/smart-web-service-cache)