# XSS(Cross-site Scripting) 방지

---

> 💡 XSS 보안 적용을 위해서 조사한 내용입니다.
> 

## 1. XSS 공격이란?

---

> 💡 공격자가 상대방의 브라우저에 스크립트가 실행되도록 해 사용자의 세션을 가로채거나, 웹사이트를 변조하거나, 악의적 콘텐츠를 삽입하거나, 피싱 공격을 진행하는 것을 말합니다.
> 

- [https://namu.wiki/w/XSS](https://namu.wiki/w/XSS)
- [https://velog.io/@kylexid/XSS-Cross-Site-Scripting](https://velog.io/@kylexid/XSS-Cross-Site-Scripting)

## 2. 공격 방지 방법

---

### 2-1. Naver 의 lucy-xss-servlet-filter

---

- XSS 공격 방지를 검색하여 찾아보면 제일 많은 레퍼런스 문서가 존재
    - 프로젝트에 적용 방법은 예제 사이트 참고
- 간단한 설정 만으로 XSS 공격 방지를 적용
    - queryString 에 대한 데이터는 아래와 같이 데이터가 변환된다.
        
        ```
        
        #client 에서 아래와 같이 데이터를 보냄
        <li>list</li>
        
        #server 에서 아래와 같이 데이터를 꺼냄
        &lt;li&gt;list&lt;/li&gt;
        ```
        
    - JSON 같은 Request Body 에 담기는 **데이터는 적용되지 않음**
        - 이 부분을 해결한 것이 Response 데이터를 내릴 때 MessageConvert 를 활용
        - CharacterEscapes 를 재구현 하여 MappingJackson2HttpMessageConvert 를 재정의
            - 즉, Request 에서는 원문 그대로 데이터를 저장하고 client 에게 데이터를 JSON 으로
            전달할 때 Convert 해버린다.
                
                ```java
                public class HtmlCharacterEscapes extends CharacterEscapes {
                  private final int[] asciiEscapes;
                
                  public HtmlCharacterEscapes() {
                    // 1. XSS 방지 처리할 특수 문자 지정
                    asciiEscapes = CharacterEscapes.standardAsciiEscapesForJSON();
                    asciiEscapes['<'] = CharacterEscapes.ESCAPE_CUSTOM;
                    asciiEscapes['>'] = CharacterEscapes.ESCAPE_CUSTOM;
                    asciiEscapes['\"'] = CharacterEscapes.ESCAPE_CUSTOM;
                    asciiEscapes['('] = CharacterEscapes.ESCAPE_CUSTOM;
                    asciiEscapes[')'] = CharacterEscapes.ESCAPE_CUSTOM;
                    asciiEscapes['#'] = CharacterEscapes.ESCAPE_CUSTOM;
                    asciiEscapes['\''] = CharacterEscapes.ESCAPE_CUSTOM;
                  }
                
                  @Override
                  public int[] getEscapeCodesForAscii() {
                    return asciiEscapes;
                  }
                
                  @Override
                  public SerializableString getEscapeSequence(int ch) {
                    return new SerializedString(StringEscapeUtils.escapeHtml4(Character.toString((char) ch)));
                  }
                }
                ```
                
                ```java
                @Configuration
                @RequiredArgsConstructor
                public class XssConfig implements WebMvcConfigurer {
                
                  private final ObjectMapper objectMapper;
                
                  @Bean
                  public MappingJackson2HttpMessageConverter jsonEscapeConverter() {
                    ObjectMapper copy = objectMapper.copy();
                    copy.getFactory().setCharacterEscapes(new HtmlCharacterEscapes());
                    return new MappingJackson2HttpMessageConverter(copy);
                  }
                }
                ```
                
- [GitHub 주소](https://github.com/naver/lucy-xss-servlet-filter) 
    - [DOM ? SAX ?](https://velog.io/@floyd/Java-DOM%EA%B3%BC-SAX)
- 이모지 관련 이슈 처리 :: [https://hello-backend.tistory.com/168](https://hello-backend.tistory.com/168)

### 2-2. jsoup 을 활용한 방식

---

- jsoup ??
    - Java HTML  Parser
- jsoup 라이브러리에서 별도로 관리되는 화이트리스트 정보가 존재하며, 이 부분은 커스텀 가능
- 기본은 문자데이터 단위로 되어 있기 때문에 별도로 Filter 정의 필요
    - Filter 와 Interceptor 를 고민하다가 Filter 로 결정
- [jsoup 테스트 사이트](https://try.jsoup.org/)
- 적용 방법
    
    ```groovy
    # jsoup dependency 설정
    
    implementation 'org.jsoup:jsoup:1.17.2'
    ```
    
    ```java
    
    # Filter 정의
    
    @Slf4j
    @Order(1)
    @Component
    public class XssEscapeHtmlFilter extends OncePerRequestFilter {
    
        @Override
        protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {
            if (isTarget(request.getRequestURI()) && "POST".equals(request.getMethod())) {
    
                MediaType mediaType = MediaType.parseMediaType(request.getContentType());
                if(mediaType.equals(MediaType.APPLICATION_JSON)) {
                    filterChain.doFilter(new XssEscapeHtmlRequestWrapper(request), response);
                }
                else {
                    filterChain.doFilter(request, response);
                }
            }
            else
            {
                filterChain.doFilter(request, response);
            }
        }
    
        private boolean isTarget(String requestURI) {
            final List<String> targets = List.of(
                "/api"
            );
            return targets.stream()
                .anyMatch(requestURI::startsWith);
        }
    }
    ```
    
    ```java
    
    @Slf4j
    public class XssEscapeHtmlRequestWrapper extends HttpServletRequestWrapper {
    
        private String escapeHtmlBody = null;
    
        public XssEscapeHtmlRequestWrapper(HttpServletRequest request) {
            super(request);
            escapeHtmlBody = this.escapeHtml(request);
        }
    
        private String escapeHtml(HttpServletRequest request) {
    
            try
            {
                var inputStream = request.getInputStream();
                var originBody = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);
    
                var outputSettings = new OutputSettings();
                outputSettings.prettyPrint(false); // 개행문자 변경하지 않도록 옵션 적용
    
                return Jsoup.clean(originBody, "", Safelist.relaxed().preserveRelativeLinks(true), outputSettings);
            } catch (IOException e) {
                log.error("[XssEscapeHtmlRequestWrapper.escapeHtml] Exception 발생 :: {}", e.getMessage());
                return "";
            }
        }
    
        @Override
        public ServletInputStream getInputStream() throws IOException {
            final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(escapeHtmlBody.getBytes());
            var servletInputStream = new ServletInputStream() {
                @Override
                public boolean isFinished() {
                    return false;
                }
    
                @Override
                public boolean isReady() {
                    return false;
                }
    
                @Override
                public void setReadListener(ReadListener listener) {
    
                }
    
                @Override
                public int read() throws IOException {
                    return byteArrayInputStream.read();
                }
            };
            return servletInputStream;
        }
    
        @Override
        public BufferedReader getReader() throws IOException {
            return new BufferedReader(new InputStreamReader(this.getInputStream()));
        }
    }
    ```
    
- `Jsoup.clean()` : 해당 함수를 통해 텍스트를 체크 한다
- `preserveRelativeLinks(true)` : 상대 경로를 허용
- `Safelist` 지원
    - 외부에서 `Whitelist` 로 예제가 많은데 `1.14.2`  버전부터 `deprecated` 됨
        - [https://www.javadoc.io/doc/org.jsoup/jsoup/1.14.2/deprecated-list.html](https://www.javadoc.io/doc/org.jsoup/jsoup/1.14.2/deprecated-list.html)
    - 항목 별 허용하는 TAG가 다르기 때문에 document 를 보는게 제일 좋다
        - [https://jsoup.org/apidocs/org/jsoup/safety/Safelist.html](https://jsoup.org/apidocs/org/jsoup/safety/Safelist.html)
        - `Safelist.none()`
        - `Safelist.simpleText()`
        - `Safelist.basic()`
        - `Safelist.basicWithImages()`
        - `Safelist.relaxed()`
- 허용하는 HTML TAG를 추가할 수도 있고 Attribute 또한 추가 가능하다
- Jsoup 관련 간단한 테스트 코드
    
    ```java
    import org.jsoup.Jsoup;
    import org.jsoup.safety.Safelist;
    import org.junit.jupiter.api.Test;
    
    public class JsoupTest {
    
        /**
         * 에디터에서 본문에 script 를 넣으면 기본적으로 Escape 변환 처리되어 처리됨
         * 하지만 HTML Code block 을 사용하면 Escape 되지 않고 그대로 저장처리 됨
         *
         *
         */
        private String contents = "<p>XSS 최종 테스트_내용 시작</p><p>&nbsp;</p><p>1</p><p>&nbsp;</p><p>2</p><p>&nbsp;</p><p>3</p><p>&nbsp;</p><div class=\"raw-html-embed\">HTML 코드 조각으로 작성\n"
            + "\n"
            + "<script>alert('코드조각 alert');</script></div><p>&nbsp;</p><p>본문 내용으로 작성</p><p>&lt;script&gt;alert('본문조각 alert');&lt;/script&gt;</p><p>&nbsp;</p><p>이미지 추가&nbsp;</p><figure class=\"image\"><img style=\"aspect-ratio:450/572;\" src=\"https://msmart.s3.ap-northeast-2.amazonaws.com/file/newsmkt/news/temp/2024_04_09/018e1bafb0ad72b48edfe661c65bda42/018ec136f3d37d00897afb026998b8ce.PNG\" width=\"450\" height=\"572\"></figure><p>이미지 추가 끝</p>";
    
        private String contents_relative = "<p>XSS 최종 테스트_내용 시작</p><p>&nbsp;</p><p>1</p><p>&nbsp;</p><p>2</p><p>&nbsp;</p><p>3</p><p>&nbsp;</p><div class=\"raw-html-embed\">HTML 코드 조각으로 작성\n"
            + "\n"
            + "<script>alert('코드조각 alert');</script></div><p>&nbsp;</p><p>본문 내용으로 작성</p><p>&lt;script&gt;alert('본문조각 alert');&lt;/script&gt;</p><p>&nbsp;</p><p>이미지 추가&nbsp;</p><figure class=\"image\"><img style=\"aspect-ratio:450/572;\" src=\"/2024_04_09/018e1bafb0ad72b48edfe661c65bda42/018ec136f3d37d00897afb026998b8ce.PNG\" width=\"450\" height=\"572\"></figure><p>이미지 추가 끝</p>";
    
        @Test
        public void safeList_none_처리_결과() {
            String escapeHtml = Jsoup.clean(contents, Safelist.none());
            System.out.println("Safelist.none() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_simpleText_처리_결과() {
            String escapeHtml = Jsoup.clean(contents, Safelist.simpleText());
            System.out.println("Safelist.simpleText() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_basic_처리_결과() {
            String escapeHtml = Jsoup.clean(contents, Safelist.basic());
            System.out.println("Safelist.basic() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_basicWithImage_처리_결과() {
            String escapeHtml = Jsoup.clean(contents, Safelist.basicWithImages());
            System.out.println("Safelist.basicWithImages() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_relaxed_처리_결과() {
            String escapeHtml = Jsoup.clean(contents, Safelist.relaxed());
            System.out.println("Safelist.relaxed() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_relaxed_custom_tag_적용() {
    
            Safelist custom_relaxed = Safelist.relaxed()
                .addAttributes("img", "style")
                .addTags("figure")
                .addAttributes("figure", "class");
    
            String escapeHtml = Jsoup.clean(contents, custom_relaxed);
            System.out.println("[custom] Safelist.relaxed() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_img_src_가_상대경로_인_경우_사라짐() {
            String escapeHtml = Jsoup.clean(contents_relative, Safelist.relaxed());
            System.out.println("[img 상대경로] Safelist.relaxed() > escapeHtml = " + escapeHtml);
        }
    
        @Test
        public void safeList_img_src_가_상대경로_가_안사라지게_세팅() {
            /**
             * 2번째 파라미터인 baseUri 를 "" 으로 세팅하면 동일하게 사라진다.
             */
            String escapeHtml = Jsoup.clean(contents_relative, "http://localhost", Safelist.relaxed().preserveRelativeLinks(true));
            System.out.println("[img 상대경로 안사라지게] Safelist.relaxed() > escapeHtml = " + escapeHtml);
        }
    
    }
    
    ```
    

### 2-3. AWS 의 WAF 서비스 사용

---

- Web Application Firewall
- 기본 Rule Set 에 XSS 공격 방지도 포함되어 있기는 하다 ( OWASP 적용 )
    - OWASP ? Open Web Application Security Proejct
        - 오픈소스 웹 애플리케이션 보안 프로젝트
        - 매년 OWASP TOP10 이라는 형식으로 발표가 된다.
            - [https://owasp.org/API-Security/editions/2023/en/0x11-t10/](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
            - [https://oobwrite.com/entry/OWASP-TOP10-2023-소개-및-정리](https://oobwrite.com/entry/OWASP-TOP10-2023-%EC%86%8C%EA%B0%9C-%EB%B0%8F-%EC%A0%95%EB%A6%AC)

## 용어 정리

---

- HTML Entity
    - HTML 에는 미리 예약된 문자가 존재하는데 이런 문자를 HTML 예약어(reserved characters)
    라고 표현
    - HTML 예약어를 HTML 코드로 사용하면 브라우저는 다른 의미로 해석
    - HTML 예약어를 사용하던 의미 그대로 사용하기 위해 별도로 만든 문자셋을 엔티티라고 함
    - [https://www.freeformatter.com/html-entities.html](https://www.freeformatter.com/html-entities.html)
- Escape
    - HTML 에서 사용이 불가능한 특정 문자를 HTML로 변환하는 행위
        - < 문자를 &lt; 이렇게 변환하는 행위
    - HTML Entity 정의된 항목을 브라우저 상에서 표현 시 Escape 처리를 해야함

## 참고자료

---

- [https://jsoup.org/](https://jsoup.org/)
- [https://offbyone.tistory.com/375](https://offbyone.tistory.com/375)
- [https://earth-ing.tistory.com/entry/Java보안-에디터-XSS-방지하기-Jsoup-사용법](https://earth-ing.tistory.com/entry/Java%EB%B3%B4%EC%95%88-%EC%97%90%EB%94%94%ED%84%B0-XSS-%EB%B0%A9%EC%A7%80%ED%95%98%EA%B8%B0-Jsoup-%EC%82%AC%EC%9A%A9%EB%B2%95)
- [https://www.tcpschool.com/html/html_text_entities](https://www.tcpschool.com/html/html_text_entities)