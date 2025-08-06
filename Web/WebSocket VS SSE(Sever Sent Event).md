# WebSocket VS SSE(Sever Sent Event)

---

> 💡 Client(Web Browser) 에 알림을 줄 수 있는 방법을 검토해보다가 알게된 내용을 정리
> 해보고자 한다.

## 1. WebSocket

---

프로그램 간의 메시지를 교환하기 위한 통신 방법 중 하나

- Client 와 Server 간 양방향 통신(Full-Duplex) 이 가능
    - HTTP 통신의 경우 Client 가 요청을 보내야 Server 가 응답을 하는 단방향 통신
- 실시간 네트워킹(Real Time-Networking) 이 가능
- `ws(webSocket)` 프로토콜, `wss(webSocket Secure)` 프로토콜이 존재
- `HTML5`  표준 기술이기에 HTML5 이전인 경우에는 [`Socket.IO`](http://Socket.IO) Library 를 활용해야 함
- WebSocket 이전 기술
    - `Polling`
        - client 가 HTTP Request 를 server 에 계속 요청하는 방식
    - `Long Polling`
        - client 가 HTTP Request 를 server 에 요청하여 메세지(=이벤트)를 받기 전까지
        대기를 하고 있다가 메세지(=이벤트) 가 발생되면 Response 를 받고 연결을
        끊은 후에 다시 Request 를 보내는 방식
    - `Streaming`
        - client 가 HTTP Request 를 server 에 요청하여 연결 통로를 만들고 server 에서만
        메세지(=이벤트) 를 보내는 방식

## 2. Server Sent Evnet(SSE)

---

서버에서 클라이언트로 단방향으로 실시간 이벤트를 전송하는 웹 기술. 

- 클라이언트는 HTTP 프로토콜을 통해 SSE 연결을 설정하고, 서버는 HTTP 응답을 유지한 상태에서 데이터를 전송
- `SSE` 는 재연결 기능을 제공하기 때문에 연결이 끊어졌을 때 자동으로 다시 연결
- `SSE` 는 이벤트 스트림 형태로 데이터를 전송하며, 클라이언트는 이벤트를 수신하여 처리
- Polling 방식에 비해서 서버 리소스와 네트워크 트래픽을 절약할 수 있다.
- HTTP/2 버전을 사용하지 않는 경우, 아주 적은 연결 수로 제한이 존재
    - [https://developer.mozilla.org/en-US/docs/Web/API/EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
- Spring Framework 에서 SSE 개발을 위한 `SseEmitter` 클래스가 제공
    - Spring Framework 4.2 부터 지원
        - [https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/htmlsingle/#mvc-ann-async-sse](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/htmlsingle/#mvc-ann-async-sse)
    - Spring Framework 5 부터는 WebFlux 지원
- HTML5 에서 추가된 `EventSource` 활용하기 위해서는 `Polyfill` 스크립트가 필요
    - #### Polyfill
        - 최신 브라우저에서 지원하는 기능을 이전 브라우저에서도 사용할 수 있도록 해주는
        라이브러리 나 모듈
        - [https://github.com/Modernizr/Modernizr/wiki/HTML5-Cross-Browser-Polyfills](https://github.com/Modernizr/Modernizr/wiki/HTML5-Cross-Browser-Polyfills)
        - [https://github.com/zloirock/core-js](https://github.com/zloirock/core-js)
        
    - #### Babel ?
        1. JavaScript 컴파일러이다. ( [https://babeljs.io/docs/](https://babeljs.io/docs/) )
        2. 트랜스파일러(Transpiler) 라고도 불린다.
        3. ES6, ES7 의 최신 문법을 ES5 문법으로 변환을 해준다.
        
        예를 들어, ES6에서 등장 한 Promise와 같이 ES5에서 변환할 수 없는 대상인 경우에는 `Polyfill`을 사용해야 함.
        

## 3. 참고자료

---

- [https://jsonobject.tistory.com/441](https://jsonobject.tistory.com/441)
- [https://velog.io/@wnguswn7/Project-SseEmitter로-알림-기능-구현하기#️-sse--server-sent-events-](https://velog.io/@wnguswn7/Project-SseEmitter%EB%A1%9C-%EC%95%8C%EB%A6%BC-%EA%B8%B0%EB%8A%A5-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0#%EF%B8%8F-sse--server-sent-events-)
- [https://hyerinoh.github.io/2020/05/30/ws-wss/](https://hyerinoh.github.io/2020/05/30/ws-wss/)
- [https://yeo-computerclass.tistory.com/480](https://yeo-computerclass.tistory.com/480)
- [https://stackoverflow.com/questions/5195452/websockets-vs-server-sent-events-eventsource/5326159](https://stackoverflow.com/questions/5195452/websockets-vs-server-sent-events-eventsource/5326159)
- [https://velog.io/@wnguswn7/Project-SseEmitter로-알림-기능-구현하기](https://velog.io/@wnguswn7/Project-SseEmitter%EB%A1%9C-%EC%95%8C%EB%A6%BC-%EA%B8%B0%EB%8A%A5-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0)