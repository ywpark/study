# JAVA 7 TrustManager 이슈

---

> 💡JAVA 8 에서 SSL 인증서 무시 로직이 있는데 이 부분이 JAVA 7 에서 동작을 안하면서
생긴 이슈 해결

## 1. 문제 확인

---

- JAVA 8 에서 동작하던 코드가 JAVA 7 에서 아래와 같은 Exception 발생

```
java.net.SocketException: Connection reset
      at java.net.SocketInputStream.read(SocketInputStream.java:196)
      at java.net.SocketInputStream.read(SocketInputStream.java:122)
      at sun.security.ssl.InputRecord.readFully(InputRecord.java:442)
      at sun.security.ssl.InputRecord.read(InputRecord.java:480)
      at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:934)
      at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1332)
      at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1359)
      at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1343)
      at sun.net.www.protocol.https.HttpsClient.afterConnect(HttpsClient.java:563)
      at sun.net.www.protocol.https.AbstractDelegateHttpsURLConnection.connect(AbstractDelegateHttpsURLConnection.java:185)
      at sun.net.www.protocol.http.HttpURLConnection.getInputStream(HttpURLConnection.java:1301)
      at java.net.HttpURLConnection.getResponseCode(HttpURLConnection.java:468)
      at sun.net.www.protocol.https.HttpsURLConnectionImpl.getResponseCode(HttpsURLConnectionImpl.java:338)
      at net.enber.msinfotech.common.utils.PpurioMessagingUtils.getPpurioAccessToken(PpurioMessagingUtils.java:145)
      at net.enber.msinfotech.common.utils.PpurioMessagingUtils.sendPpurioMessage(PpurioMessagingUtils.java:66)
      at net.enber.msinfotech.api.services.MediaService.sendPpurioMessage(MediaService.java:4002)
      at net.enber.msinfotech.api.controllers.MediaController.processPpurioSendSmsLms(MediaController.java:1871)
      at net.enber.msinfotech.api.controllers.MediaController.handleRequestInternal(MediaController.java:165)
      at org.springframework.web.servlet.mvc.AbstractController.handleRequest(AbstractController.java:153)
      at org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter.handle(SimpleControllerHandlerAdapter.java:48)
      at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:933)
      at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:867)
      at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:953)
      at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:844)
      at javax.servlet.http.HttpServlet.service(HttpServlet.java:655)
      at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:829)
      at javax.servlet.http.HttpServlet.service(HttpServlet.java:764)
      at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
      at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
      at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:52)
      at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
      at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
      at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:88)
      at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:106)
      at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
      at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
      at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:196)
      at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97)
      at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542)
      at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:135)
      at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:81)
      at org.apache.catalina.valves.AbstractAccessLogValve.invoke(AbstractAccessLogValve.java:698)
      at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78)
      at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:364)
      at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:624)
      at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
      at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:831)
      at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1673)
      at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
      at org.apache.tomcat.util.threads.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1191)
      at org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:659)
      at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
      at java.lang.Thread.run(Thread.java:745)
```

## 2. 문제 원인 및 해결

---

> 💡여러 원인이 있을 수 있지만 Try 해 본 방법만 작성을 함.

- Base64 인코딩 문제
    - 외부 서비스와 API 통신을 할 때 HTTP Header 에 Base64로 인코딩한 문자열을 전달하는데
    JAVA7 과 JAVA8 에서의 Base64 의 라이브러리가 달라서 생기는 문제라고 의심
    - JAVA 7 : `org.apache.commons.codec.binary.Base64`
        
        ```java
        String headerValue = new String(Base64.encodeBase64("문자열").getBytes()));
        ```
        
    - JAVA 8 : `java.util.Base64`
        
        ```java
         String headerValue = Base64.getEncoder().encodeToString(("문자열").getBytes());
        ```
        
    - 확인 결과 동일한 문자열을 return 하는 것으로 확인하여 패스

- 방화벽 문제
    - 외부서비스에 방화벽 이슈가 아닌가 하여 확인 하였지만, 해당 부분도 문제가 없는 것으로
    확인하여 패스
    
- `HttpsURLConnection` 에 모든 인증서를 신뢰하도록 구성하기 위해 사용한 `TrustManager` 구현한
코드 문제
    - 코드 문제로 확인하였으며 TO-BE 와 같이 변경하여 문제를 해결
    - AS-IS
        
        ```java
         private void trustAllCerts() throws NoSuchAlgorithmException, KeyManagementException {
              TrustManager[] trustAllCerts = new TrustManager[]{
                      new X509TrustManager() {
                          public X509Certificate[] getAcceptedIssuers() {
                              return null;
                          }
        
                          public void checkClientTrusted(X509Certificate[] chain, String authType) {
                          }
        
                          public void checkServerTrusted(X509Certificate[] chain, String authType) {
                          }
                      }
              };
              SSLContext sc = SSLContext.getInstance("SSL");
              sc.init(null, trustAllCerts, new java.security.SecureRandom());
              HttpsURLConnection.setDefaultSSLSocketFactory(sc.getSocketFactory());
          }
        ```
        
    - TO-BE
        
        ```java
         private void trustAllCerts() throws NoSuchAlgorithmException, KeyManagementException {
              TrustManager[] trustAllCerts = new TrustManager[]{
                      new X509TrustManager() {
                          public X509Certificate[] getAcceptedIssuers() {
                              return null;
                          }
        
                          public void checkClientTrusted(X509Certificate[] chain, String authType) {
                          }
        
                          public void checkServerTrusted(X509Certificate[] chain, String authType) {
                          }
                      }
              };
              SSLContext sc = SSLContext.getInstance("TLSv1.2");
              sc.init(null, trustAllCerts, new java.security.SecureRandom());
              HttpsURLConnection.setDefaultSSLSocketFactory(sc.getSocketFactory());
          }
        ```
        

## 3. 원인 분석

---

- SSL VS TLS
    - `SSL protocol`
        - `Secure Sockets Layer` : 보안 소켓 계층을 의미
        - 버전 : 1.0 , 2.0 , 3.0
        - 사용여부 : 현재는 보안상 이슈로 인하여 대부분이 미사용. 레거시 서버에서는 사용될 수도 있음
    - `TLS protocol`
        - `Transport Layer Security`  : 전송 계층 보안을 의미
        - 버전 : 1.0, 1.1, 1.2, 1.3
        - 사용여부 : 대부분의 서버가 1.2 혹은 1.3 버전을 사용함
- JAVA 7 에서의 의미
    
    ```java
    SSLContext sc = SSLContext.getInstance("SSL");
    ```
    
    > JAVA 7 의 기본 SSL/TLS 버전이 TLSv1.0 을 기본값으로 하고 있으며 `SSL` 로 SSLContext 를
    생성하는 경우에는 SSL 3.0 과 TLSv1.0 을 지원하는 소켓팩토리를 생성
    > 
- JAVA 버전에 따른 기본 TLS 버전
    
    ![image.png](/assets/java_trustmanager/1.png)
    
- 위와 같은 상황이기 때문에 외부서비스에서는 TLS1.2 를 활용되고 있을 수 있기 때문에 아래와
같이 변경해서 처리
    
    ```java
    SSLContext sc = SSLContext.getInstance("TLSv1.2");
    ```
    
- 혹시나 JAVA 소스 파일을 변경을 하지 못한다면 JVM 옵션 설정을 통해서도 처리할 수 있다.
    
    ```
    -Dhttps.protocols=TLSv1.2
    ```