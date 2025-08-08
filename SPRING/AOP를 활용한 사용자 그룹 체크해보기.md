# AOP를 활용하여 사용자의 그룹 체크

---

> 💡 사용자의 그룹에 따라서 메뉴이동이 제한되고 API 또한 제한이 되도록 개발 내역이 존재하여 해당 문제를 접근한 방식 및 사용법을 제공하고자 함.
> 

## 목표

---

1. 모바일 과 PC에 적용되어야 하기에 서버에서 처리해야 함
2. 어쩔 수 없는 경우가 아니라면 모바일 Native 소스에는 영향이 없어야 함
3. 개발자가 쉽게 이해하고 사용하기가 편해야 함
4. 메뉴 버튼을 누른 경우에 해당 메뉴의 HTML 내용조차 안보여야 함

## 접근 방식

---

> 💡 개발 내역을 보고 생각할 수 있는 부분은 아래의 3가지 부분이었고 최종적으로는 AOP 방식으로 채택하였다.
>     이유는 아래와 같다.
>  1. Annotation 으로 체크 할 수 있음
>  2. 메뉴 & API 가 그룹 별로 정의되어 있는 수가 적어서 해당 처리가 낫다고 판단

- Filter
- Interceptor
- AOP

## 개발 내용

---

- UserGroups Enum
    
    ```java
    # com.msit.exam.common.auth.enums.UserGroups
    
    public enum UserGroups {
    
        GENERAL("GENERAL"),
        EXPERT("EXPERT"),
        INSTITUTE("INSTITUTE"),
        JOURNALIST("JOURNALIST");
    
        private String code;
    
        UserGroups(final String code) {
            this.code = code;
        }
    
        public String getGroupCode() {
            return code;
        }
    
    }
    
    ```
    

- UserGroup Annotation
    
    ```java
    # com.msit.exam.common.auth.annotation.UserGroup 
    
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface UserGroup {
        
        UserGroups[] access();
        
        String message() default "접근할 수 없는 그룹권한입니다";
    }
    ```
    

- Aspect
    
    ```java
    # com.msit.exam.common.aop.UserGroupAspect
    
    @Slf4j
    @RequiredArgsConstructor
    @Component
    @Aspect
    public class UserGroupAspect {
    
        private final TokenUtil tokenUtil;
    
        @Value("${custom.jwt.access.name}")
        private String accessName;
    
        @Around("@annotation(com.msit.exam.common.auth.annotation.UserGroup)")
        public Object checkUserGroup(ProceedingJoinPoint joinPoint) throws Throwable {
    
            MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            Method method = signature.getMethod();
    
            if(validRequiredAnnotation(method.getAnnotations()) == false) {
                log.error("@UserGroup 는 @LoginRequired 필수 설정입니다!!");
                throw new IllegalArgumentException();
            }
    
            HttpServletRequest request = ((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getRequest();
            Cookie cookie = WebUtils.getCookie(request, accessName);
            if(Optional.ofNullable(cookie).isEmpty()) {
                log.error("[UserGroupAspect] cookie 값이 존재하지 않습니다.");
                throw new NullPointerException();
            }
    
            /**
             * 로그인 한 사용자 정보 조회
             *
             * @TODO 사용자 정보가 서버에서 변경되었어도 Cookie에는 안되는 경우가 있다
             *       이거는 이슈나오면 해당 로직 적용한다
             */
            final String accessToken = cookie.getValue().split("\\|")[0];
            var loginUser = tokenUtil.getUserSession(accessToken);
    
            // @UserGroup annotation 조회
            UserGroup userGroup = method.getAnnotation(UserGroup.class);
    
            // @UserGroup annotation 에 설정 한 그룹목록 조회
            var userGroups = userGroup.access();
    
            // 로그인한 사용자 그룹 체크
            boolean includeUserGroup = Arrays.stream(userGroups)
                    .filter(item -> item.getGroupCode().equals(loginUser.getGroupIdx()))
                    .findFirst().isPresent();
    
            if(includeUserGroup) {
                return joinPoint.proceed();
            }
    
            final String servletPath = request.getServletPath();
            if (servletPath.matches("^/?api/(\\w+(-)?/?)+")) {
                /**
                 * API 호출 인 경우
                 *
                 * API URL 형식에 다른 특수문자가 들어오면 위의 정규식은 변경되어야 한다.
                 *
                 */
                throw new NoAuthUserGroupException("");
    
            }
            else {
                /**
                 * View 호출 인 경우
                 */
                HttpServletResponse response = ((ServletRequestAttributes)RequestContextHolder.currentRequestAttributes()).getResponse();
                response.sendError(HttpStatus.UNAUTHORIZED.value(), userGroup.message());
            }
    
            return null;
        }
    
        /**
         * UserGroup Annotation 사용 시 필수 어노테이션 존재 여부 체크
         *
         * @param annotations 메소드에 적용된 annotation 목록
         * @return true : 유효함 , false : 유효하지 않음
         */
        private boolean validRequiredAnnotation(Annotation[] annotations) {
            return Arrays.stream(annotations)
                    .filter(item -> item instanceof LoginRequired)
                    .findFirst().isPresent();
        }
    
    }
    ```
    

## 사용법

---

- 메뉴(VIEW) 에 특정 사용자 그룹이 정의되어 있는 경우
    
    ```java
    	
    	@GetMapping("/test")
    	@UserGroup(access = { UserGroups.EXPERT }, message = "Expert 권한만 이용 가능해요.")
    	@LoginRequired
    	public ModelAndView test(@LoginUser UserSession userSession) throws Exception {
    		........
    	}
    ```
    
    - `@LoginRequired` :: 필수로 같이 작성해야 하며 같은 메소드에 설정해야 함
    - `@UserGroup(access = { UserGroups.EXPERT }, message = "Expert 권한만 이용 가능해요.")`
        - access : 해당 메뉴(VIEW) 접근할 수 있는 그룹을 정의 ( 여러 개도 작성가능 )
        ex) { UserGroup.GENERAL, UserGroups.EXPERT }
        - message : 사용자에게 안내 되어야 하는 별도 메세지 정의 할 수 있음.
                         없으면 default 메세지 안내
    - `Response`
        - 사용자가 정의된 그룹에 속하지 않는다면 에러를 발생(HTTP 401) 하고 에러 페이지로
        이동하여 Alert 메시지 노출 후 이전 페이지로 이동함

- API 에 호출 시 특정 사용자 그룹이 정의되어 있는 경우
    
    ```java
        
        @GetMapping("/user/my-info")
        @LoginRequired
        @UserGroup(access = { UserGroups.EXPERT })
        public ResponseEntity userMyInfo(@LoginUser UserSession userSession) {
            if (userSession == null) {
                return ResponseBuilder.data(null);
            }
            return ResponseBuilder.data(findUserService.findMyInfoByIdx(userSession.getIdx()));
        }
    ```
    
    > 규칙과 사용법은 VIEW 설정과 동일하다. Response 방식만 다르다
    > 
    - Response
        - 정의된 API URL 형식인 경우에 사용자가 정의된 그룹에 속하지 않는다면 응답을 JSON
        으로 내려준다. ( throw NoAuthUserGroupException() )
        - 현재(24.03.27) result code 값을 100으로 정의되어 있음

## 개선 사항

---

- Client 에서 올라온 Cookie 값을 가지고 사용자의 그룹을 체크하는 부분이 Data 동기화
이슈가 발생할 수 있음
    - 사용자의 그룹의 경우 새벽에 Batch 를 통해서 일괄적으로 계산이 되고 있고, 사용자의
    Access Token 의 만료 시간이 1시간이기 때문에 해당 문제가 그렇게 발생되지 않을거
    같아서 일단 PASS 하기로 함