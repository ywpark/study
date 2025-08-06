# SSE(Server-Sent Events) 적용 해보기

---

> 💡서버에서 클라이언트로 실시간 데이터를 보내는 기술
>

## 0. 배경

---

  A 라는 페이지로 사용자가 진입한 후에 서버에서 작업이 완료되면 `A 페이지 → B 페이지` 로 페이지 이동을 하고자 함.

위와 같은 상황에서 아래 방법을 고민하게 됨.

**Polling VS Long Polling VS WebSocket VS SSE** 

여러가지를 고민하다가 `SSE` 를 활용하기로 함

이유는 아래와 같다.

- 사용하고자 하는 기능은 서버에서 클라이언트로 데이터를 보내는 단방향 통신만 필요
- 주기적인 Request 를 통하여 서버 리소스 사용 및 트래픽 발생을 최소한으로 하고자 함

고려사항

- HTTP/2 가 아니라면 동시 접속이 최대 브라우저 당 6개 이며, HTTP/2 의 경우에는
기본값이 100개 이다.
    - [https://developer.mozilla.org/en-US/docs/Web/API/EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
- 최근 브라우저에서는 대부분 지원하지만 IE 의 경우에는 지원되지 않는다.
    - polyfill 라이브러리를 활용하면 가능
    - [https://velog.io/@blessoms2017/Event-Source-Polyfill로-모든-브라우저에서-SSE-구현하기](https://velog.io/@blessoms2017/Event-Source-Polyfill%EB%A1%9C-%EB%AA%A8%EB%93%A0-%EB%B8%8C%EB%9D%BC%EC%9A%B0%EC%A0%80%EC%97%90%EC%84%9C-SSE-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0)
- 단일 서버에서는 문제가 없지만 Scale-Out 을 해야 하는 구조라면 Redis Pub/Sub 기능을 활용하여 구현이 가능함

## 1. 서비스 환경

---

- OpenJDK 8
- SpringFramework 5.3.1
- Spring Boot 2.4
- Tomcat 8.5
- Nginx ( 리버스 프록시 용도로 활용 )

## 2. SSE

---

- Client
    
    ```jsx
    
    const sse = new EventSource("/sse/connect/" + '${id}');
    
    sse.addEventListener('result', (e) => {
        const { data: receivedConnectData } = e;
        const connectData = JSON.parse(receivedConnectData);
    
        if(connectData.status === 'result') {
            sse.close();
            location.href = connectData.moveUrl;
        }
    });
    ```
    
    > [https://developer.mozilla.org/en-US/docs/Web/API/EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
    > 
- Server
    
    ```java
    # SSEController.java 파일
    
    @RequiredArgsConstructor
    @RestController
    @Slf4j
    @RequestMapping("/sse")
    public class SSEController {
    
        private final SseEmitterService sseEmitterService;
    
        @GetMapping(value = "/connect/{id}", produces = "text/event-stream;charset=UTF-8")
        public ResponseEntity<SseEmitter> payConnect(@PathVariable("id") String id, HttpServletResponse response) {
    
            /**
             * nginx 를 사용하는 경우 아래 Header 설정 필요
             *
             * SSE 를 활용한 실시간 응답을 적용하기 위해 응답 데이터를 버퍼링 하지 않고 클라이언트에게 전송
             *
             * Nginx.conf 에서 proxy_buffering 옵션을 통하여 버퍼링을 끌 수 있지만 그렇게되면 장점도 있지만
             * 단점도 있기에 특정 응답에만 버퍼링을 사용하지 않기 위에 아래 헤더 값을 정의함.
             *
             */
            response.setHeader("X-Accel-Buffering", "no");
            return ResponseEntity.ok(sseEmitterService.register(id));
        }
    
    }
    ```
    
    ```java
    # SseEmitterService.java 파일
    
    @RequiredArgsConstructor
    @Service
    @Slf4j
    public class SseEmitterService {
    
        private final SSEClient sseClient;
    
        private final StatusService statusService;
    
        public SseEmitter register(final String id) {
    
            /**
             * client 에서 재 연결 도중에 메세지가 발송되면
             * 유실될 수 있기에 체크를 진행한다.
             */
            StatusDataVO data = statusService.getStatus(id)
                    .orElseThrow(() -> new NullPointerException("상태 없음 :: id : " + id));
            if(data.isComplete()) {
                return sseClient.connectAndSend(id);
            }
    
            return sseClient.connect(id);
        }
    }
    ```
    
    ```java
    # SSEClient.java 파일
    
    @Component
    @Slf4j
    public class SSEClient {
    
        /**
         * Thread-safe 객체를 활용해야 한다.
         */
        private final Map<String, SseEmitter> clients = new ConcurrentHashMap<>();
    
        /**
         * Timeout 시간 60초 정의
         * 클라이언트에서 정의된 시간마다 연결을 끊고 재 연결을 시도한다.
         */
        private final Long TIME_OUT = 60 * 1000L;
    
        /**
         * 서버와의 연결이 안되면 정의된 시간 마다 클라이언트에서 재연결을 시도한다.
         *
         */
        private final Long RECONNECT_TIMEOUT = 1000L;
    
        public SseEmitter connect(final String id) {
    
            try
            {
                SseEmitter sseEmitter = new SseEmitter(TIME_OUT);
                sseEmitter.onCompletion(() -> {
                    log.info("sseEmitter.onCompletion!!");
                    clients.remove(payId);
                });
    
                sseEmitter.onTimeout(() -> {
                    log.info("sseEmitter.onTimeout!!");
                    sseEmitter.complete();
                });
    
                sseEmitter.onError((throwable) -> {
                    log.error("[sseEmitter.onError] 발생 :: ", throwable);
                    sseEmitter.complete();
                });
    
                clients.put(id, sseEmitter);
                log.info("new emitter added :: sseEmitter : {}, id : {}", sseEmitter, id);
                log.info("emitter timeout : {}", sseEmitter.getTimeout());
                log.info("emitter map size: {}", clients.size());
    
                PayStatusResultResDTO initData = PayStatusResultResDTO.builder()
                        .status("connect")
                        .moveUrl("")
                        .build();
    
                sseEmitter.send(SseEmitter.event()
                        .name("result")
                        .data(initData)
                        .reconnectTime(RECONNECT_TIMEOUT));
    
                return sseEmitter;
            }
            catch (IOException e) {
    
                log.error("[connect] IOException 발생 :: id : {}", id, e);
                clients.remove(id);
    
                throw new RuntimeException();
            }
        }
    
        public SseEmitter connectAndSend(final String id) {
    
            try
            {
                SseEmitter sseEmitter = new SseEmitter(TIME_OUT);
    
                /**
                 * 최초 더미 데이터 전송
                 */
                ResultDTO initData = ResultDTO.builder()
                        .status("connect")
                        .moveUrl("")
                        .build();
    
                sseEmitter.send(SseEmitter.event()
                        .name("result")
                        .data(initData)
                        .reconnectTime(RECONNECT_TIMEOUT));
    
                /**
                 * 상태 정보 따른 실제 데이터 전송
                 */
                ResultDTO data = ResultDTO.builder()
                        .status("result")
                        .moveUrl("/test.do?id=" + id)
                        .build();
    
                sseEmitter.send(SseEmitter.event()
                        .name("result")
                        .data(data)
                        .reconnectTime(RECONNECT_TIMEOUT));
                
                return sseEmitter;
            }
            catch (IOException e) {
    
                log.error("[connectAndSend] IOException 발생 :: id : {}", id, e);
                clients.remove(id);
    
                throw new RuntimeException();
            }
        }
    
        public void sendTo(final String id) {
    
            SseEmitter emitter = clients.get(id);
            if(emitter != null) {
    
                try
                {
                    ResultDTO data = ResultDTO.builder()
                            .status("result")
                            .moveUrl("/test.do?id=" + id)
                            .build();
    
                    emitter.send(SseEmitter.event()
                            .name("result")
                            .data(data)
                            .reconnectTime(RECONNECT_TIMEOUT));
                }
                catch (IOException e) {
                    log.error("[sendTo] IOException 발생 :: id : {}", id, e);
                    clients.remove(id);
                }
            }
    
        }
    }
    ```
    
- Nginx
    
    ```groovy
    
    server {
    
        listen 443 ssl;
        server_name a.b.com;
        underscores_in_headers      on;
    
        ssl_certificate /etc/nginx/ssl/abc.pem;
        ssl_certificate_key /etc/nginx/ssl/VX.key.pem;
        ssl_protocols  TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
    
        location /sse/connect/ {
    			
          # 리버스 프록시를 설정하고 정의
          proxy_pass http://localhost:9999;
          
          # 프록시 서버에 client 에서 받은 request 의 header 부분을 pass 할지 여부
          # 기본값이 on 이다. ( = proxy_pass_request_body 값 또한 동일 )
          proxy_pass_request_headers on;
    			
          # 프록시 서버로 HTTP 통신 시 기본값은 1.0
          # SSE 를 사용하기 위해서는 최소 1.1 버전으로 변경 적용
          proxy_http_version 1.1;
          
          # HTTP/1.0 에서는 Connection Header 값이 close 로 되어 있기 때문에
          # 이 부분을 아래와 같이 변경 해주어야 한다.
          # SSE HTTP 지속 연결이 되어야 한다.
          proxy_set_header Connection '';
    
          # 청크 전송 인코딩을 비활성화 
          # ( 서버에서 클라이언트로 응답을 보낼때 조각을 내서 점진적으로 전송 )
          # SSE 는 연속적인 데이터스트림이기에 설정 필요
          chunked_transfer_encoding off;
          
          # Nginx 의 버퍼링 기능을 비활성화 하여 실시간 데이터 전송을 함
          # Spring 에서 response Header 에 [ X-Accel-Buffering : no ] 설정을 
          # 통하여 정의 하였기에 Nginx 설정에서는 제외 
          # proxy_buffering off;
          
          # 캐싱을 사용하지 않기에 설정안함
          # Nginx 에서의 Default 값이 off 임
          # proxy_cache off;
          
          # SSE 를 활용하는 기능 부분에서는 Timeout 이 그렇게 길 필요가 없기 때문에
          # 기본값인 60초를 사용한다. 필요한 경우에는 수정 필요
          # proxy_read_timeout 60s
          
        }
    
        location / {
    
            client_body_temp_path           /tmp/;
            client_body_in_file_only        on;
            client_body_buffer_size         128K;
            client_max_body_size            100M;
    
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_buffer_size   1M;
            proxy_buffers   4 1M;
    
            proxy_pass_request_headers      on;
    
            proxy_pass http://localhost:9999;
        }
    }
    
    ```
    
    > [Nginx Modules Reference](https://nginx.org/en/docs/)
    > 

## 3. 오류

---

### 1. `net::ERR_INCOMPLETE_CHUNKED_ENCODING 200 (OK)`

- 웹브라우저의 Console 에서 확인된 메세지
- 서버에서의 응답이 청크(CHUNKED) 단위로 보내졌는데 클라이언트에서 데이터를 다 받지
못하는 경우 발생하는 에러로 보여짐
- SSE 의 경우 서버에서 스트림 방식이기에 안정성을 위하여  `Chunked Transfer Encoding`  기능을
비활성화 처리함

## 4. 참고자료

---

[https://wimoney.tistory.com/m/entry/Pet-Hub-SSE-실시간-알림-구현시-트러블-슈팅](https://wimoney.tistory.com/m/entry/Pet-Hub-SSE-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%95%8C%EB%A6%BC-%EA%B5%AC%ED%98%84%EC%8B%9C-%ED%8A%B8%EB%9F%AC%EB%B8%94-%EC%8A%88%ED%8C%85)

[https://amaran-th.github.io/Spring/[Spring] Server-Sent Events(SSE)/](https://amaran-th.github.io/Spring/%5BSpring%5D%20Server-Sent%20Events(SSE)/)

[https://jypark1111.tistory.com/264](https://jypark1111.tistory.com/264)