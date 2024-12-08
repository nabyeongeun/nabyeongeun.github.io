---
categories: [spring]
tags: [spring,java]
---

새로운 프로젝트를 시작한다고 하면, 그 프로젝트에 필요한 도구를 챙기고 설정을 할 것이다.  
하지만 Spring을 사용한다고 하면 대부분의 프로젝트에 범용적으로 자주 사용되는 것 들이 있다.  
대표적으로 SpringBoot, Tomcat, JPA, MyBatis, Lombok 등이 있다.

프레임워크나 라이브러리는 자주 사용하는 것을 그냥 dependency에 추가하면 된다.  
하지만 기능상으로도 범용적으로 적용되어야 할 것이 있다.  
대표적으로 logger, XSS filter, 문서화 도구 등이 있다.

기존에는 SpringFramework만 주로 사용해왔지만, 새로운 프로젝트에 투입되면서 SpringBoot와 여러가지 도구들을 추가하고, 향후에 다른 프로젝트에도 범용적으로 쓸 만한 도구를 조사하다가 Swagger와 XSS Filter를 추가하기로 생각했다.

처음에는 Swagger를 추가했고 정상기능하는 것을 확인하고 사용하고 있었는데, XSS 필터를 추가 한 이후로 Swagger가 동작하지 않는 것을 확인했다.

XSS 필터 내용은 다음과 같다.

```java
public class HTMLCharacterEscapes extends CharacterEscapes {

    private static final long serialVersionUID = 1L;
    private final int[] asciiEscapes;

    public HTMLCharacterEscapes() {
        asciiEscapes = CharacterEscapes.standardAsciiEscapesForJSON();
        asciiEscapes['<'] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes['>'] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes['&'] = CharacterEscapes.ESCAPE_CUSTOM;
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

        if(ch > 127) // non ASCII character는 이스케이프 처리하지 않음.
            return null;

        return new SerializedString(StringEscapeUtils.escapeHtml4(Character.toString((char) ch)));
    }
}

@Configuration
@RequiredArgsConstructor
@Slf4j
public class XssConfig implements WebMvcConfigurer {

  // 모든 service request의 response에 XSS filter를 적용한다.
  @Override
  public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    converters.add(1, XssConverter());
  }

  private MappingJackson2HttpMessageConverter XssConverter() {

    ObjectMapper mapper = new ObjectMapper();
    mapper.getFactory().setCharacterEscapes(new HTMLCharacterEscapes());
    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter(mapper);

    converter.setSupportedMediaTypes(List.of(MediaType.APPLICATION_JSON));

    return converter;
  }
}
```
```gradle
implementation 'org.apache.commons:commons-text:1.8'
```

> [Spring Boot에서 JSON API에 XSS Filter 적용하기](https://jojoldu.tistory.com/470)

![1](/assets/post/2024-09-27/img.png)

Swagger 화면 데이터를 가져오는 `http://localhost:8080/v3/api-docs`를 호출한 결과 이상하게 나오는 것을 확인했다.

![2](/assets/post/2024-09-27/img_1.png)


`"`이 `&quot;`으로 치환된 것으로 보아 XSS필터가 적용된 것이 원인 인 것으로 보인다.


```java
public HTMLCharacterEscapes() {
        asciiEscapes = CharacterEscapes.standardAsciiEscapesForJSON();
        asciiEscapes['<'] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes['>'] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes['&'] = CharacterEscapes.ESCAPE_CUSTOM;
//        asciiEscapes['\"'] = CharacterEscapes.ESCAPE_CUSTOM; swagger errors
        asciiEscapes['('] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes[')'] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes['#'] = CharacterEscapes.ESCAPE_CUSTOM;
        asciiEscapes['\''] = CharacterEscapes.ESCAPE_CUSTOM;
    }
```

그래서 필터에서 `"`을 치환하지 않도록 수정했다.

![3](/assets/post/2024-09-27/img_2.png)

결과 값이 달라지긴 했지만 여전히 Swagger는 사용할 수 없는 상태였으므로 자세한 원인 분석이 필요했다.

```java
@RestController
public class OpenApiWebMvcResource extends OpenApiResource {
    
    @Operation(
        hidden = true
    )
    @GetMapping(
        value = {"${springdoc.api-docs.path:#{T(org.springdoc.core.utils.Constants).DEFAULT_API_DOCS_URL}}"},
        produces = {"application/json"}
    )
    public String openapiJson(HttpServletRequest request, @Value("${springdoc.api-docs.path:#{T(org.springdoc.core.utils.Constants).DEFAULT_API_DOCS_URL}}") String apiDocsUrl, Locale locale) throws JsonProcessingException {
        return super.openapiJson(request, apiDocsUrl, locale);
    }
}
```

내가 생성한 서비스는 정상적으로 수행되고, Swagger의 api-docs서비스에 문제가 생겼기 때문에 두 서비스를 비교해 보았다.  
내 서비스는 dto를 반환하고 있지만 Swagger는 String을 반환하고 있다.  
원래라면 dto든 String이든 `HttpMessageConverter`가 처리 해 줄 것 이기 때문에 별로 신경을 안써도 됐겠지만, 이 경우에는 Custom HttpMessageConver를 적용했기 때문에 String과 dto를 처리하는 데에 차이가 생긴것으로 보인다.

위의 추론으로 `api-docs`의 결과값 맨 앞과 뒤에 붙는 `"`를 제거해야 할 필요가 있어 보였다.

```java
@Bean
public FilterRegistrationBean<SwaggerFilter> swaggerFilter() {

    FilterRegistrationBean<SwaggerFilter> registrationBean = new FilterRegistrationBean<>();
    registrationBean.setFilter(new SwaggerFilter());
    registrationBean.addUrlPatterns("/v3/api-docs");

    return registrationBean;
}

public class SwaggerFilter implements Filter {

  @Override
  public void doFilter(final ServletRequest request, final ServletResponse response, final FilterChain chain) throws IOException, ServletException {
    ByteResponseWrapper byteResponseWrapper = new ByteResponseWrapper((HttpServletResponse) response);
    ByteRequestWrapper byteRequestWrapper = new ByteRequestWrapper((HttpServletRequest) request);

    chain.doFilter(byteRequestWrapper, byteResponseWrapper);

    String jsonResponse = new String(byteResponseWrapper.getBytes(), response.getCharacterEncoding());

    response.getOutputStream().write((new com.google.gson.JsonParser().parse(jsonResponse).getAsString())
      .getBytes(response.getCharacterEncoding()));
  }

  @Override
  public void destroy() {

  }

  static class ByteResponseWrapper extends HttpServletResponseWrapper {

    private PrintWriter writer;
    private ByteOutputStream output;

    public byte[] getBytes() {
      writer.flush();
      return output.getBytes();
    }

    public ByteResponseWrapper(HttpServletResponse response) {
      super(response);
      output = new ByteOutputStream();
      writer = new PrintWriter(output);
    }

    @Override
    public PrintWriter getWriter() {
      return writer;
    }

    @Override
    public ServletOutputStream getOutputStream() {
      return output;
    }
  }

  static class ByteRequestWrapper extends HttpServletRequestWrapper {

    byte[] requestBytes = null;
    private ByteInputStream byteInputStream;


    public ByteRequestWrapper(HttpServletRequest request) throws IOException {
      super(request);
      ByteArrayOutputStream baos = new ByteArrayOutputStream();

      InputStream inputStream = request.getInputStream();

      byte[] buffer = new byte[4096];
      int read = 0;
      while ((read = inputStream.read(buffer)) != -1) {
        baos.write(buffer, 0, read);
      }

      replaceRequestPayload(baos.toByteArray());
    }

    @Override
    public BufferedReader getReader() {
      return new BufferedReader(new InputStreamReader(getInputStream()));
    }

    @Override
    public ServletInputStream getInputStream() {
      return byteInputStream;
    }

    public void replaceRequestPayload(byte[] newPayload) {
      requestBytes = newPayload;
      byteInputStream = new ByteInputStream(new ByteArrayInputStream(requestBytes));
    }
  }

  static class ByteOutputStream extends ServletOutputStream {

    private ByteArrayOutputStream bos = new ByteArrayOutputStream();

    @Override
    public void write(int b) {
      bos.write(b);
    }

    public byte[] getBytes() {
      return bos.toByteArray();
    }

    @Override
    public boolean isReady() {
      return false;
    }

    @Override
    public void setWriteListener(WriteListener writeListener) {

    }
  }

  static class ByteInputStream extends ServletInputStream {

    private InputStream inputStream;

    public ByteInputStream(final InputStream inputStream) {
      this.inputStream = inputStream;
    }

    @Override
    public int read() throws IOException {
      return inputStream.read();
    }

    @Override
    public boolean isFinished() {
      return false;
    }

    @Override
    public boolean isReady() {
      return false;
    }

    @Override
    public void setReadListener(ReadListener readListener) {

    }
  }
}
```

```gradle
implementation 'com.google.code.gson:gson:2.11.0'
```

> [Endpoint "/api-docs" doesn't work with custom GsonHttpMessageConverter](https://stackoverflow.com/questions/59393191/endpoint-api-docs-doesnt-work-with-custom-gsonhttpmessageconverter)

그래서 response를 재작성 하여 `"`가 삭제되도록 `/v3/api-docs/`에만 필터를 적용했다.

![4](/assets/post/2024-09-27/img_3.png)  
![5](/assets/post/2024-09-27/img_4.png)

그 이후로 정상 동작하는 것이 확인되었다.

이 문제는 XSS필터(`HttpMessageConverter`이지만 편의상 필터라고 칭한다)를 추가했기때문에 발생하는 문제다.

만약에 XSS를 `HttpMessageConverter`가 아니라 처음부터 `Filter`로 구현했으면 이런 문제가 생기지 않았을 것이다.  
다만, `Filter`로 구현하였을 경우에는 새로운 서비스를 추가할 때마다 `Filter`에 새로운 URL을 추가해야하는 번거로움이 따르고, 휴먼에러가 발생할 확률이 크다.  
그에 비해 Swagger는 `v3/api-docs`하나만 필터를 적용하면 되기 때문에 지금 당장의 문제해결에는 시간이 꽤 걸렸지만 장기적으로 더 유리할 것으로 보인다.
