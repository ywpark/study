# OAuth 2.0 적용해보기

---

> 💡서비스에서 자체 회원가입을 유도하여 회원을 관리하는 것도 좋지만 좀 더 쉽게 우리 서비스를 이용할 수 있도록 이미 
> 다른 플랫폼을 사용하고 있는 사용자 데이터에 접근하기 위하여 인증 및 인가를 담당하고 있는 표준 프로토콜인 
> OAuth 를 활용해보고자 함


# 정의

---

- Open Authorization
- 인터넷 사용자들이 비밀번호를 제공하지 않고 다른 웹사이트 상의 자신들의 정보에 대해 웹사이트나 애플리케이션의 접근 권한을 부여할 수 있는 공통적인 수단으로서 사용되는 접근 위임을 위한 개방형 표준

# 구성요소

---


> 💡 `Authorization Server` , `Resource Server` 가 [문서](https://datatracker.ietf.org/doc/html/rfc6749#section-1.2) 상에는 별개로 분리되어 있지만
> 개발자의 선택에 따라서 하나의 서버로 구성할 수 있다.

- `Resource Owner`
    - 리소스 소유자 또는 사용자
    - 우리의 서비스를 이용하면서, 다른 플랫폼( 구글, 네이버, 카카오등) 을 이용하고 있는 사용자
- `Client`
    - 보호된 자원을 사용하려고 접근 요청을 하는 애플리케이션
    - Resource Server의 자원을 이용하고자 하는 서비스, 일반적으로 우리가 개발하려는 서비스
- `Authorization Server`
    - Resource Owner를 인증하고, Client에게 액세스 토큰을 발급해주는 서버
    - 인증/인가를 수행하는 서버로 클라이언트의 접근 자격을 확인하고 Access Token을 발급하여 권한을 부여하는 역할
- `Resource Server`
    - 사용자의 보호된 자원을 호스팅하는 서버
    - 구글, 네이버, 카카오와 같이 사용자의 리소스를 가지고 있는 서버
- `OAUTH FLOW`
    
    ![1.png](/assets/kotlin/1.png)
    
    > 참고) [https://guide.ncloud-docs.com/docs/b2bpls-oauth2](https://guide.ncloud-docs.com/docs/b2bpls-oauth2)
    > 

# 개발 환경

---

- Spring Framework boot 3.4.1
- Spring Security 6.4.2
- Kotlin

# 1. WITHOUT SPRING SECURITY

---

- application.yml
    
    ```yaml
    oauth2:
      naver:
        authorize-url: https://nid.naver.com/oauth2.0/authorize
        client-id: #발급받은 client id#
        client-secret: #발급받은 secret#
        response-type: code
        redirect-uri: https://local.exam.com:8443/login/oauth/naver
        token-url: https://nid.naver.com/oauth2.0/token
        user-info-url: https://openapi.naver.com/v1/nid/me
    ```
    
- login.html
    
    ```html
    
    // 네이버 로그인 버튼 Click 함수
    function signInByNAVER() {
    
        window.open("https://local.exam.com:8443/login/oauth/naver/authorize", "oauthPopup", "menubar=no, toolbar=no");
        window.addEventListener("message", (evt) => {
    
          console.log('evt.origin = ', evt.origin);
          console.log('window.location.origin = ', window.location.origin);
          console.log('message = ', evt.data);
    
          getAccessToken(evt.data);
        });
    
      }
    
    function getAccessToken(code) {
    
      fetch("https://local.exam.com:8443/login/oauth/naver/token?code=" + code, {
        method: 'GET'
      })
              .then(res => {
                return res.json();
              })
              .then(data => {
                console.log("json data = ", data);
                getOAuthUserInfo(data.accessToken, data.tokenType);
              });
    }
    
    function getOAuthUserInfo(accessToken, tokenType) {
    
      fetch("https://local.exam.com:8443/login/oauth/naver/info/me?token=" + accessToken + "&type=" + tokenType, {
        method: "GET"
      })
              .then(res => {
                return res.json();
              })
              .then(data => {
                console.log("naver me info = ", data);
              })
    }
    ```
    
- naver.html
    
    ```html
    <!DOCTYPE html>
    <html lang="en">
    <script>
    
      const oauth_code = '[[${code}]]';
      window.opener.postMessage(oauth_code, window.location.origin);
      window.close();
    
    </script>
    </html>
    ```
    
- OAuthController.kt
    
    ```kotlin
    package com.ywpark.exam.api.shared.controller
    
    import com.ywpark.exam.api.shared.model.TokenResult
    import com.ywpark.exam.api.shared.model.UserInfo
    import com.ywpark.exam.api.shared.service.OAuthNaverService
    import io.github.oshai.kotlinlogging.KotlinLogging
    import jakarta.servlet.http.Cookie
    import jakarta.servlet.http.HttpServletRequest
    import jakarta.servlet.http.HttpServletResponse
    import org.springframework.stereotype.Controller
    import org.springframework.ui.Model
    import org.springframework.web.bind.annotation.CookieValue
    import org.springframework.web.bind.annotation.GetMapping
    import org.springframework.web.bind.annotation.PathVariable
    import org.springframework.web.bind.annotation.RequestMapping
    import org.springframework.web.bind.annotation.RequestParam
    import org.springframework.web.bind.annotation.ResponseBody
    import java.math.BigInteger
    import java.security.SecureRandom
    
    @RequestMapping("/login/oauth")
    @Controller
    class OAuthController(
        private val oAuthNaverService: OAuthNaverService
    ) {
    
        companion object {
            private val logger = KotlinLogging.logger { }
        }
    
        @GetMapping("/{provider}/authorize")
        fun authorize(@PathVariable provider: String, httpServletRequest: HttpServletRequest, httpServletResponse: HttpServletResponse) {
    
            val random = SecureRandom()
            val state: String = BigInteger(130, random).toString(32)
    
            val oauthUrl: String = when(provider) {
                "naver" ->  {
                    oAuthNaverService.startOAuth(state)
                }
                else -> ""
            }
    
            if(oauthUrl.isEmpty()) {
    					// Error 발생 처리
            }
    
            logger.info { "provider : ${provider} , OAuth URL : ${oauthUrl}" }
    
            val stateCookie: Cookie  = Cookie("oauth_state", state)
            stateCookie.isHttpOnly = true
            stateCookie.secure = true
            stateCookie.path = "/"
            httpServletResponse.addCookie(stateCookie)
    
            httpServletResponse.sendRedirect(oauthUrl)
        }
    
        @GetMapping("/{provider}/token")
        @ResponseBody
        fun token(@RequestParam(required = true) code: String): TokenResult {
    
            val random = SecureRandom()
            val state: String = BigInteger(130, random).toString(32)
    
            val response = oAuthNaverService.getAccessToken(code, state)
            logger.info { "token response = ${response}" }
    
            return TokenResult(response.access_token, response.refresh_token, response.token_type, response.expires_in)
        }
    
        @GetMapping("/{provider}/info/me")
        @ResponseBody
        fun infoMe(@RequestParam(required = true) token: String, @RequestParam(required = true) type: String): UserInfo {
            return oAuthNaverService.getMyInfo(token, type);
        }
    
        @GetMapping("/naver")
        fun naver(@RequestParam state: String, @RequestParam code: String, @CookieValue(value = "oauth_state", required = false) cookieValue: String?, model: Model): String {
    
            if(state != cookieValue) {
                // BadRequest
            }
    
            model.addAttribute("code", code);
            return "/oauth/naver";
        }
    
        @GetMapping("/sec/naver")
        fun secNaver(model: Model): String {
            model.addAttribute("code", "code");
            return "/oauth/naver";
        }
    }
    
    ```
    
- OAuthNaverService.kt
    
    ```kotlin
    
    package com.ywpark.exam.api.shared.service
    
    import com.ywpark.exam.api.shared.model.OAuthProfileApiResult
    import com.ywpark.exam.api.shared.model.OAuthTokenApiResult
    import com.ywpark.exam.api.shared.model.UserInfo
    import io.github.oshai.kotlinlogging.KotlinLogging
    import org.springframework.beans.factory.annotation.Value
    import org.springframework.http.HttpEntity
    import org.springframework.http.HttpHeaders
    import org.springframework.http.HttpMethod
    import org.springframework.stereotype.Service
    import org.springframework.web.client.RestTemplate
    import java.net.URLEncoder
    
    @Service
    class OAuthNaverService(
        private val restTemplate: RestTemplate
    ) {
    
        @Value("\${oauth2.naver.authorize-url}")
        lateinit var url: String;
    
        @Value("\${oauth2.naver.client-id}")
        lateinit var clientId: String
    
        @Value("\${oauth2.naver.client-secret}")
        lateinit var clientSecret: String
    
        @Value("\${oauth2.naver.response-type}")
        lateinit var responseType: String
    
        @Value("\${oauth2.naver.redirect-uri}")
        lateinit var redirectUri: String
    
        @Value("\${oauth2.naver.token-url}")
        lateinit var tokenUrl: String
    
        @Value("\${oauth2.naver.user-info-url}")
        lateinit var userInfoUrl: String
    
        companion object {
            private val logger = KotlinLogging.logger { }
        }
    
        fun startOAuth(state: String): String {
            val encUrl = URLEncoder.encode(redirectUri, "UTF-8")
            return "${url}?client_id=${clientId}&response_type=${responseType}&redirect_uri=${encUrl}&state=${state}"
        }
    
        fun getAccessToken(code: String, state: String): OAuthTokenApiResult {
            return restTemplate.getForObject(
                "${tokenUrl}?grant_type=authorization_code&client_id=${clientId}&client_secret=${clientSecret}&code=${code}&state=${state}",
                OAuthTokenApiResult::class.java) ?: throw RuntimeException("OAuth Token API Error")
        }
    
        fun getMyInfo(accessToken: String, tokenType: String): UserInfo {
    
            val headers = HttpHeaders().apply {
                set("Authorization", "${tokenType} ${accessToken}")
            }
    
            val entity = HttpEntity<String>(null, headers)
            val naverResult = restTemplate.exchange("${userInfoUrl}", HttpMethod.GET, entity, OAuthProfileApiResult::class.java).body
                ?: throw RuntimeException("OAuth NAVER PROFILE API Error")
    
            return UserInfo(
                id = naverResult.response.id,
                name = naverResult.response.name,
                email = naverResult.response.email,
                mobile = naverResult.response.mobile
            )
        }
    
    }
    ```
    
    > `lateinit`  : property의 초기화를 지연시키는 역할, 객체 생성 시에는 초기화하지 않고, 나중에 값을 할당할 수 있도록 해주는 키워드
    
    기본적으로 Kotlin 은 클래스에서 프로퍼티 선언하면 반드시 초기화를 해야 함. 그렇지 않으면
    컴파일 에러가 발생 ( `Property must be initialized or be abstract` )
    
    `lateinit` 을 사용하고 싶지 않으면 아래와 같이 활용 가능
    `class OAuthNaverService(@Value("\${oauth2.naver.authorize-url}") val url: String)`
    > 
- Naver 기준의 API 설명 ( 자세한 API 내용은 [문서](https://developers.naver.com/docs/login/api/api.md) 에서 확인 가능 )
    
    > OAuth2 의 경우 표준이기 때문에 다른 플랫폼 또한 거의 비슷한 흐름
    > 
    
    STEP 1. OAuth2 로그인 호출
    
    - URL : [https://nid.naver.com/oauth2.0/authorize](https://nid.naver.com/oauth2.0/authorize)
    - PARAMETER
        - `response_type` : code ( 고정값 )
        - `client_id` : Resource Server 에서 발급받은 client id
        - `redirect_uri` : Token 을 전달 받을 Client 의 URI
                                  Resource Server 에 등록할 때 작성한 URI
                                  URL 인코딩을 적용한 문자열
        - `state` :  CSRF 공격을 방지하기 위한 임의의 문자열
                       로그인 과정 중에 동일한 값을 유지해야 함
    - RESPONSE
        - code :  TOKEN 발급을 위한 인가 코드
        - state : `STEP 1` 에서 생성한 state 값
        - error : 에러 코드
        - error_description : 에러 메세지
    
    STEP 2. TOKEN 발급 
    
    - URL : [https://nid.naver.com/oauth2.0/token](https://nid.naver.com/oauth2.0/token)
    - PARAMETER
        - grant_type : authorization_code ( 고정값 )
                            갱신, 삭제에 대한 구분값도 있지만 이 부분은 제외
        - client_id : Resource Server 에서 발급받은 client id
        - client_secret : Resource Server 에서 발급받은 client secret
        - code : `STEP 1` 에서 발급 받은 인가 코드
        - state : 신규 생성 or `STEP 1` 생성한 state 값
    - RESPONSE
        - access_token : 접근 토큰
        - refresh_token : 갱신 토큰
        - token_type : Bearer ( 고정값 )
        - expires_in : 접근 토큰 유효시간
        - error : 에러 코드
        - error_description : 에러 메세지
    
    STEP 3. 사용자 정보 조회
    
    - URL : [https://openapi.naver.com/v1/nid/me](https://openapi.naver.com/v1/nid/me)
    - HEADER
        - Authorization : `STEP 2` 에서 받은 `${token_type} ${access_token}`
    - RESPONSE
        - resultcode : API 결과 코드
        - message : API 결과 메세지
        - response : 아래에 전달받는 데이터는 Resource Server 에 설정한 항목들에 한하여
                          데이터 받을 수 있음
            - id : 네이버 아이디가 아닌 별도의 유니크한 아이디
            - nickname : 사용자 별명
            - name : 사용자 이름
            - email : 사용자 메일주소
            - gender : 성별
            - age : 연령대
            - birthday : 생일
            - profile_image : 프로필 사진 URL
            - birthyear : 출생 연도
            - mobile : 휴대전화 번호

# 2. USING SPRING SECURITY

---

- build.gradle.kts
    
    ```groovy
    dependencies {
    	implementation("org.springframework.boot:spring-boot-starter-security")
    	implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
    }
    ```
    
- application.yml
    
    ```yaml
    spring:
      security:
        oauth2:
          client:
            provider:
              naver:
                authorization-uri: 'https://nid.naver.com/oauth2.0/authorize'
                token-uri: 'https://nid.naver.com/oauth2.0/token'
                user-info-uri: 'https://openapi.naver.com/v1/nid/me'
                user-name-attribute: response
            registration:
              naver:
                client-id: #발급받은 client id#
                client-secret: #발급받은 secret#
                client-authentication-method: client_secret_post
                redirect-uri: 'https://local.exam.com:8443/login/oauth2/code/naver'
                authorization-grant-type: authorization_code
                client-name: naver
                scope:
                  - profile
    ```
    
- SecurityConfig.kt
    
    ```kotlin
    package com.ywpark.exam.api.shared.config
    
    import com.ywpark.exam.api.shared.security.CustomAuthenticationFailureHandler
    import com.ywpark.exam.api.shared.security.CustomAuthenticationSuccessHandler
    import com.ywpark.exam.api.shared.security.oauth2.OAuth2SuccessHandler
    import com.ywpark.exam.api.shared.security.oauth2.OAuthFailureHandler
    import com.ywpark.exam.api.shared.security.oauth2.OAuth2UserService
    import org.springframework.context.annotation.Bean
    import org.springframework.context.annotation.Configuration
    import org.springframework.http.HttpMethod
    import org.springframework.security.config.annotation.web.builders.HttpSecurity
    import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
    import org.springframework.security.config.http.SessionCreationPolicy
    import org.springframework.security.web.SecurityFilterChain
    
    @Configuration
    @EnableWebSecurity
    class OAuthSecurityConfig(
        private val oAuth2UserService: OAuth2UserService
    ) {
    
        @Bean
        fun configure(http: HttpSecurity): SecurityFilterChain {
    
            http
                .securityMatcher("/views/**", "/login", "/logout", "/oauth2/**", "/login/oauth2/**")
                .csrf { it.disable() }
                .authorizeHttpRequests {
                    it
                        .requestMatchers("/views/user-login", "/views/user-signup", "/oauth2/**", "/login/oauth2/**").permitAll()
                        .requestMatchers("/views/admin").hasAuthority("ADMIN") // hasRole("ADMIN") 은 ROLE_ 이 붙어야 함
                        .anyRequest().authenticated()
                }
                .sessionManagement {
                    it
                        .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                }
                .formLogin {
                    it
                        .loginPage("/views/user-login")
                        .loginProcessingUrl("/login")
                        .successHandler(CustomAuthenticationSuccessHandler())
                        .failureHandler(CustomAuthenticationFailureHandler())
                }
                .oauth2Login {
                    it
                        .loginPage("/views/user-login")
                        .userInfoEndpoint { ue -> ue.userService(oAuth2UserService) }
                        .successHandler(OAuth2SuccessHandler())
                        .failureHandler(OAuthFailureHandler())
                }
    
            return http.build()
        }
    }
    ```
    
    - `formLogin` :  서비스에 직접 회원가입 하여 로그인도 할 수 있기 때문에 적용
    - `oauth2Login`  :  OAuth2 연동을 위하여 정의
        - `loginPage` : 소셜로그인을 할 수 있는 로그인페이지 URL 정의
        - `userInfoEndpoint` : 로그인이 성공 한 뒤에 사용자 정보를 가져오는 서비스 등록
        - `successHandler` : OAuth2User 가 정상적으로 생성된 뒤에 후처리 하는 Handler
        - `failureHandler` : OAuth2User 를 생성하는 도중 문제가 발생하는 경우에 후처리 
                                    하는 Handler
- OAuth2UserService.kt
    
    ```kotlin
    package com.ywpark.exam.api.shared.security.oauth2
    
    import com.ywpark.exam.api.command.domain.user.enums.LoginMethod
    import com.ywpark.exam.api.command.domain.user.service.UserDomainService
    import com.ywpark.exam.api.query.service.UserQueryService
    import com.ywpark.exam.api.shared.security.LoginUser
    import io.github.oshai.kotlinlogging.KotlinLogging
    import org.springframework.security.core.authority.SimpleGrantedAuthority
    import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService
    import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest
    import org.springframework.security.oauth2.core.user.OAuth2User
    import org.springframework.stereotype.Service
    
    @Service
    class OAuth2UserService(
        private val userDomainService: UserDomainService,
        private val userQueryService: UserQueryService
    ) : DefaultOAuth2UserService() {
    
        companion object {
            private val logger = KotlinLogging.logger {}
        }
    
        override fun loadUser(userRequest: OAuth2UserRequest?): OAuth2User {
    
            val accessToken:String = userRequest?.accessToken?.tokenValue ?: ""
            val tokenType:String = userRequest?.accessToken?.tokenType?.value ?: ""
            val expiresIn:Long = userRequest?.accessToken?.expiresAt?.toEpochMilli() ?: 0L
    
            /**
             *
             * super.loadUser(userRequest) 호출 시 아래와 같은 에러 발생
             * Attribute value for 'id' cannot be null
             *
             * yml 파일에서 해당 property 값이 id 로 되어 있어서 발생 된 문제 ( 아래와 같이 response 로 바꿈 )
             *  user-name-attribute: response
             *
             *
             */
            val user:OAuth2User  = super.loadUser(userRequest)
    
            var resultCode: String = user.attributes["resultcode"] as String
            var message: String = user.attributes["message"] as String
            if(resultCode != "00") {
                logger.error { "OAuth2 User Info Error: [$resultCode] $message" }
                throw IllegalArgumentException("OAuth2 사용자 정보 조회 실패")
            }
    
            val userInfo:Map<String, String> = user.attributes["response"] as Map<String, String>
            logger.info { "OAuth2 User Info: $userInfo" }
    
            // 사용자 정보 조회
            val loginUser: LoginUser? = userQueryService.getUserByLoginId(userInfo["email"] as String)?.let {
    
                /**
                 * 사용자 정보가 있으면 token 정보만 갱신한다.
                 */
                userDomainService.updateOAuthToken(
                    it.idx,
                    accessToken,
                    tokenType,
                    expiresIn,
                    resultCode,
                    message)
    
                // 사용자 정보 return
                LoginUser(
                    method = it.type,
                    idx = it.idx,
                    id = it.loginId,
                    name = it.name,
                    email = it.email,
                    password = it.password,
                    authorities = it.roles.map { role -> SimpleGrantedAuthority(role) }.toMutableList(),
                    attributes = null
                )
            }
    
            if(loginUser != null) {
                return loginUser
            }
    
            val createdUser = userDomainService.createOAuthUser(
                name = userInfo["name"] as String,
                email = userInfo["email"] as String,
                phone = userInfo["mobile"] as String,
                method = LoginMethod.fromCode(
                    userRequest?.clientRegistration?.registrationId ?: throw IllegalArgumentException("Invalid Login registrationId")
                ) ?: throw IllegalArgumentException("Invalid Login Method")
            )
    
            userDomainService.createOAuthToken(
                userIdx = createdUser.id!!,
                accessToken = accessToken,
                tokenType = tokenType,
                expiresIn = expiresIn,
                error = resultCode,
                errorDescription = message)
    
            return LoginUser(
                method = createdUser.method.code,
                idx = createdUser.id!!,
                id = createdUser.loginId,
                name = createdUser.name,
                email = createdUser.email,
                password = null,
                authorities = mutableListOf(SimpleGrantedAuthority("USER")),
                attributes = null
            )
        }
    
    }
    ```
    
    - `OAuth2UserRequest` : 소셜로그인이 성공하면 발급되는 `Token` 정보를 확인
        - `refresh_token` 은 확인이 불가능 함
            - 자료를 찾아보면 여러 방법이 있지만 그 중 하나의 방법 소개
            - 소스 코드
                - `OAuth2AccessTokenResponseClient` 클래스 확장
                    
                    ```kotlin
                    
                    package com.ywpark.exam.api.shared.security.oauth2
                    
                    import org.springframework.core.ParameterizedTypeReference
                    import org.springframework.http.*
                    import org.springframework.security.oauth2.client.endpoint.OAuth2AccessTokenResponseClient
                    import org.springframework.security.oauth2.client.endpoint.OAuth2AuthorizationCodeGrantRequest
                    import org.springframework.security.oauth2.core.OAuth2AccessToken
                    import org.springframework.security.oauth2.core.endpoint.OAuth2AccessTokenResponse
                    import org.springframework.stereotype.Component
                    import org.springframework.util.LinkedMultiValueMap
                    import org.springframework.web.client.RestTemplate
                    
                    @Component
                    class CustomOAuth2AccessTokenResponseClient(
                        private val resetTemplate: RestTemplate
                    ) : OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest> {
                    
                        override fun getTokenResponse(authorizationGrantRequest: OAuth2AuthorizationCodeGrantRequest?): OAuth2AccessTokenResponse {
                    
                            val tokenUri = authorizationGrantRequest?.clientRegistration?.providerDetails?.tokenUri ?: ""
                            val headers = HttpHeaders().apply {
                                contentType = MediaType.APPLICATION_FORM_URLENCODED
                            }
                    
                            val body = LinkedMultiValueMap<String, String>().apply {
                                add("grant_type", "authorization_code")
                                add("code", authorizationGrantRequest?.authorizationExchange?.authorizationResponse?.code)
                                add("redirect_uri", authorizationGrantRequest?.authorizationExchange?.authorizationRequest?.redirectUri)
                                add("client_id", authorizationGrantRequest?.clientRegistration?.clientId)
                                add("client_secret", authorizationGrantRequest?.clientRegistration?.clientSecret)
                            }
                    
                            val entity = HttpEntity(body, headers)
                            val response: ResponseEntity<Map<String, Any>> = resetTemplate.exchange(
                                tokenUri, HttpMethod.POST, entity, object : ParameterizedTypeReference<Map<String, Any>>() {}
                            )
                    
                            val tokenResponse = response.body ?: throw RuntimeException("OAuth Token API Error")
                            return OAuth2AccessTokenResponse
                                .withToken(tokenResponse["access_token"] as String)
                                .tokenType(OAuth2AccessToken.TokenType.BEARER)
                                .expiresIn((tokenResponse["expires_in"] as String).toLong())
                                .additionalParameters(mapOf("refresh_token" to tokenResponse["refresh_token"] as String))
                                .build()
                        }
                    }
                    ```
                    
                    > `OAuth2AccessTokenResponse.refreshToken()` 정의 할 수 있지만 최종적으로는 `loadUser()` 에서는 쉽게 접근이 안되서 `additionalParameters()` 에 별도로 데이터를 생성하여 전달한다.
                    
                    그렇게 하면 `userRequest.additionalParameters["refresh_token"]` 이런식으로 token 값을 활용 가능
                    > 
                - SecurityConfig.kt
                    
                    ```kotlin
                    .oauth2Login {
                        it
                            .tokenEndpoint {te ->
                                te.accessTokenResponseClient(customOAuth2AccessTokenResponseClient)
                    
                            }
                    }
                    ```
                    
                    > 기존 설정에 위의 항목만 추가하면 됨
                    > 
    - `super.loadUser(userRequest)` : 발급받은 Token 정보를 활용하여 사용자 정보를 조회
    - Spring Security
- LoginUser.kt
    
    ```kotlin
    package com.ywpark.exam.api.shared.security
    
    import org.springframework.security.core.GrantedAuthority
    import org.springframework.security.core.userdetails.UserDetails
    import org.springframework.security.oauth2.core.user.OAuth2User
    
    class LoginUser(
        private val method: String,
        private val idx: Long,
        private val id: String,
        private val name: String,
        private val email: String,
        private val password: String?,
        private val authorities: MutableCollection<GrantedAuthority>,
        private val attributes: MutableMap<String, Any>?
    ) : UserDetails, OAuth2User {
    
        // 일반로그인 UserDetails
        override fun getAuthorities(): MutableCollection<out GrantedAuthority> = authorities
        override fun getPassword(): String? = password
        override fun getUsername(): String = id
    
        // OAuth2로그인 OAuth2User
        override fun getName(): String = id
        override fun getAttributes(): MutableMap<String, Any>? = attributes
    
    }
    ```
    
    - `FormLogin` 을 사용하는 사용자와 `OAuth2` 를 사용하는 사용자를 다 포함할 수 있는 사용자 정보 객체
    
- Controller.kt
    
    ```kotlin
    package com.ywpark.exam.api.query.controller
    
    import com.ywpark.exam.api.query.model.response.SearchBoardResponse
    import com.ywpark.exam.api.query.service.BoardQueryService
    import com.ywpark.exam.api.shared.security.LoginUser
    import org.springframework.http.ResponseEntity
    import org.springframework.security.core.annotation.AuthenticationPrincipal
    import org.springframework.web.bind.annotation.GetMapping
    import org.springframework.web.bind.annotation.RequestMapping
    import org.springframework.web.bind.annotation.RestController
    
    @RestController
    @RequestMapping("/api/")
    class BoardQueryController(
        private val boardQueryService: BoardQueryService
    ) {
    
        @GetMapping("boards")
        fun list(@AuthenticationPrincipal user: LoginUser): ResponseEntity<List<SearchBoardResponse>> {
            return ResponseEntity.ok(boardQueryService.list())
        }
    
    }
    ```
    
    - `@AuthenticationPrincipal` 를 활용해서 로그인 한 사용자 정보를 가져올 수 있음

## 2-1. 오류 상황

---

- DefaultOAuth2UserService 클래스를 상속받아서 loadUser() 함수에서 로그인 성공 시 응답을 받아야 하는데 응답을 못 받는 상황 발생
    - Security 를 활용하는 경우에는 일반 FormLogin 처럼 2가지의 URL 이 이미 정의가 되어 있음
        - `{baseUrl}/oauth2/authorization/{registrationId}`  : OAuth 로그인 호출 페이지
        - `{baseUrl}/login/oauth2/code/{registrationId}`  : OAuth 로그인 성공 후 콜백 URI
    - [https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html#oauth2login-sample-redirect-uri](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html#oauth2login-sample-redirect-uri)
    - 콜백 URI 를 임의의 URI 로 지정하여 응답 확인이 불가능

# 연관 기술

---

- OpenID Connect (OIDC)
    - OAuth 2.0의 확장
    - OAuth2 의 주요 목적은 인가 이며, 인증 절차를 인가를 받기 위한 절차
    - OpenID 의 주요 목적은 인증

# 고민사항

---

### 1. Spring Security 의 적용 여부

---

- Security 를 활용하면 좀 더 쉽고 빠르게 적용할 수 있지만 Spring Security 에 대한 학습이 어느
정도 필요.
- Security 가 적용되지 않은 상태에서 이미 운영중인 서비스라면 오히려 적용하기가 쉽지 않을 것으로 보이며, 프로젝트 초기 구성 단계하면 고려해 볼 만한 가치는 있음.

### 2. 사용자 정보 관리를 Session VS Token 방식

---

- OAuth 에서 Token 이 발급되는데 서비스 운영에 통일성을 주려면 Token 방식 적용이 제일 좋다고 생각함
- Session 관리와 Token 관리에는 각 각의 장단점이 있기 때문에 서비스에서 추구하는 방향에 맞게
적용하면 될 것으로 판단됨

### 3. Session 관리를 어떻게 할 것인가

---

- 기존 서비스에서 Session 을 활용하고 있다면 OAuth 를 활용하더라도 그대로 유지

### 4. 자동로그인 처리 방법

---

- Spring Security 를 활용한다면 `rememberMe` 기능을 활용할 수 있음
- Spring Security 활용하지 않는다면 `rememberMe` 와 비슷하게 `Token` 을 발행하여 자동로그인을 기능을 적용
    - 로그인 성공 시에 `Token` 을 발행하여 Client 에 `Cookie` 로 전달
    - `Cookie` 로 전달하기에 기본적으로 만료일자를 정의
        - 만료 이후의 절차는 정책에 따라 정의하여 적용
        ( 재 로그인 방식, 만료일을 자동으로 갱신 )
    - 사용자 여러 환경에서 로그인 할 수 있기 때문에 이 부분도 고려해서 `Token` 관리 고민 필요

### 5. 로그아웃 처리 방법

---

- `Client` 에서 로그아웃을 한다고 `Authorization / Resource Server`  에서도 로그아웃이
되면 안되는 것으로 판단됨
- `Session / Cookie`  정보를 삭제 하는 방식으로 진행
    - 보안 상으로 민감하다면 발급받은 `Token` 정보를 만료/폐기 처리

### 6. 발급받은 ACCESS_TOKEN 은 어떻게 관리 할 것인가

---

- 개발 사항에 따라 관리 여부가 결정 될 것으로 판단됨
- API 호출이 빈번하다면 별도로 저장하여 TOKEN 관리가 필요
- 그렇지 않다면 별도로 관리할 필요는 없어 보임

### 7. ACCESS_TOKEN 갱신은 어떻게 처리 할 것인가

---

- `Refresh Token` 은 Cookie 에서 HttpOnly , Secure 속성을 줘서 보안을 좀 더 강화하고 `Access Token` 은 Response Body 로 데이터를 전달하여 관리.
- `Access Token` 만료 전에 Client 에서 갱신 요청
    - Client 에서 만료 시간을 체크하여 일정 시간 전에 갱신 하도록 정의
- Server 에서 401 Status Code 를 응답받으면 Access Token 을  API 를 통하여 재 발급 받은 후에
기존 HTTP API 를 재 호출
- `Access Token` 을 갱신할 때 `Refresh Token` 도 같이 갱신하는 RTR  ( Refresh Token Rotation ) 
방식도 생각해 볼 수 있음

# 참고 자료

---

- [https://developers.naver.com/docs/login/api/api.md](https://developers.naver.com/docs/login/api/api.md)
- [https://m42-orion.tistory.com/161](https://m42-orion.tistory.com/161)
- ChatGPT