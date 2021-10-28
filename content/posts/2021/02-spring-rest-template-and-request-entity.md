---
title: "스프링 RestTemplate 잘 활용하기"
subtitle: "스프링의 RestTemplate 객체의 통신을 OkHttp나 Apache HttpComponent로 교체 해 HTTP를 더 빠르게 요청할 수 있게 설정 및 RestTemplate 사용 시 편리하게 RequestEntity 객체에 담아 요청하기. 번외로 UriComponentsBuilder를 통해 복잡한 URI를 쉽게 생성하기."
categories: ['java', 'spring']
tags: ['java', 'spring', 'utils', 'restful', 'http', 'rest-template', 'okhttp', 'httpcomponents', 'uri']
date: 2021-10-28T21:00:00+09:00
slug: spring-rest-template-and-request-entity-and-uri
---

스프링 내에선 `RestTemplate`란 객체가 존재해 HTTP 요청을 쉽게 할 수 있다. 하지만 `RestTemplate`를 요청 시 생성해서 사용하거나 하는 잘못된 방법이 존재한다. `RestTemplate` 내 HTTP Client 종류와 설명, Bean 등록 설정까지 해보고, `RequestEntity` 객체를 통해 코드 가독성 좋게 요청하는 방법과 번외로 URI를 템플릿화 하여 작성할 수 있는 `UriComponentsBuilder` 클래스 사용법까지 작성해 보았다.

## RestTemplate 클래스 및 사용용도

* [Spring RestTemplate JavaDoc (Spring Framework 3.0 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/RestTemplate.html)

HTTP 요청 시 사용하는 클래스이다. 객체를 생성하고 메소드를 살펴보면 많은 메소드들이 펼쳐져 있다. 각각의 HTTP Method마다 존재하고 매개변수에 따라 여러 메소드들을 골라 사용할 수 있다. 하지만 HTTP Method와 URL만 해도 간단하지 않다. 따라서 아래의 메소드를 사용하는 것을 권장한다.

* `ResponseEntity<T> exchange(RequestEntity<?> entity, Class<T> responseType)`

* `ResponseEntity<T> exchange(RequestEntity<?> entity, ParameterizedTypeReference<T> responseType)`

이 두개의 메소드의 차이는 뒤의 타입 정보 전달의 차이가 존재한다. 첫번째 Class타입은 평상시에 사용하면 되지만, 만약 제네릭 타입을 받아야 할때는 두번째 메소드를 사용할 수 있다. 자세한 사용 방법은 아래 [RestTemplate 잘 사용하기](#resttemplate-잘-사용하기)에서 살펴보겠다.

이 RestTemplate를 그냥 new로 생성 했을 땐 자바에 내장된 [HttpUrlConnection](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/net/HttpURLConnection.html) 클래스를 사용한다. JDK 1.1부터 사용되었고, 버전업하며 크게 개선된 건 없어 단순히 연결 후 요청하고 응답하는 액션만 처리할 뿐이다. 커넥션 방법인 [Connection: keep-alive](https://developer.mozilla.org/ko/docs/Web/HTTP/Headers/Connection)나 [HTTP/2](https://ko.wikipedia.org/wiki/HTTP/2)와 같은 프로토콜에서의 속도 개선사항을 적용받을 수 없다.

## HTTP Client 라이브러리를 RestTemplate에 등록 및 Bean 설정

우리는 두가지 라이브러리 중 하나를 선택해 도움을 받을 수 있다.

* [OkHttp](https://square.github.io/okhttp/)
  
  * HTTP/2 지원.
  
  * Android에서 HTTP 요청 시 사용하는 라이브러리인 Retrofit이 사용하는 라이브러리.
  
  * 코틀린 코드로 작성되어 코틀린 라이브러리가 의존성에 포함.

* [Apache HttpComponents](https://hc.apache.org/)
  
  * 자바로 작성됨.
  
  * 오랫동안 검증된 Apache Commons에서 분리한 프로젝트.
  
  * 버전 4는 HTTP/1.1까지 지원.

라이브러리를 선택했다면 Spring Boot 2.x 기준으로 설정 방법을 설명하겠다.


### OkHttp 라이브러리 설정 방법
   
* [Maven Central 검색 사이트](https://search.maven.org/artifact/com.squareup.okhttp3/okhttp)

  - 아래 코드를 바로 입력하기 보다, Maven 검색 사이트에서 최신 버전을 살펴보고 제공하는 코드를 입력하는 것을 권장한다.

  - 현재 OkHttp 5 alpha 버전이 등록되고 있지만 호환성을 담보하지 않으므로 최신의 4.x.x 버전을 사용 권장한다.

* Maven
  
  ```xml
  <dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.9.2<!-- 2021/10 기준 최신버전 --></version>
  </dependency>
  ```

* Gradle
  
  ```groovy
  implementation 'com.squareup.okhttp3:okhttp:4.9.2'
  ```


### Apache HttpComponents 라이브러리 설정 방법
   
* **주의! HttpComponents 5 이상 버전은 Spring Framework 5.3 에서 사전 설정이 존재하지 않음!**
  
  - 만약 직접 사용 시엔 [ClientHttpRequestFactory](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/http/client/ClientHttpRequestFactory.html) 인터페이스를 이용해 클래스 구현.
  
* [Maven Central 검색 사이트](https://search.maven.org/artifact/org.apache.httpcomponents/httpclient)

  - 아래 코드를 바로 입력하기 보다, Maven 검색 사이트에서 최신 버전을 살펴보고 제공하는 코드를 입력하는 것을 권장한다.

* Maven
  
  ```xml
  <dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.13<!-- 2021/10 기준 최신버전 --></version>
  </dependency>
  ```

* Gradle
  
  ```groovy
  implementation 'org.apache.httpcomponents:httpclient:4.5.13'
  ```


### HttpClientConfiguration 설정 코드

위 OkHttp와 HttpComponents 라이브러리 설정 시에 Spring Boot에서 제공하는 클래스가 있다. `RestTemplateBuilder` 클래스이다.

* [Spring Boot RestTemplateBuilder JavaDoc](https://docs.spring.io/spring-boot/docs/2.5.x/api/org/springframework/boot/web/client/RestTemplateBuilder.html)

이 클래스는 이미 Spring Boot를 사용 하고 있다면, [RestTemplateAutoConfiguration](https://docs.spring.io/spring-boot/docs/2.5.x/api/org/springframework/boot/autoconfigure/web/client/RestTemplateAutoConfiguration.html)에 의해 Bean으로 설정되어있다. 이미 설정된 MessageConverter가 포함된 상태이므로 `RestTemplateBuilder` Bean을 받아 `build()` 메소드로 `RestTemplate` 클래스를 생성하여 Bean으로 등록하면 된다.

* OkHttp 사용 시

  ```java
  import okhttp3.OkHttpClient;
  import org.springframework.boot.web.client.RestTemplateBuilder;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.http.client.OkHttp3ClientHttpRequestFactory;
  import org.springframework.web.client.RestTemplate;

  @Configuration
  public class HttpClientConfiguration {

      @Bean
      public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
          return restTemplateBuilder
              .requestFactory(() -> new OkHttp3ClientHttpRequestFactory(new OkHttpClient()))
              .build();
      }

  }
  ```

* Apache HttpComponents 4 사용 시

  ```java
  import org.apache.http.client.HttpClient;
  import org.apache.http.impl.client.HttpClients;
  import org.springframework.boot.web.client.RestTemplateBuilder;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
  import org.springframework.web.client.RestTemplate;

  @Configuration
  public class HttpClientConfiguration {

      @Bean
      public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
          return restTemplateBuilder
              .requestFactory(() -> new HttpComponentsClientHttpRequestFactory(HttpClients.createDefault()))
              .build();
      }

  }
  ```

만약, Spring Boot 기반이 아닌 구 버전 사용자는 `RestTemplate`를 `new`로 생성 후 등록할 수도 있다.

* OkHttp 사용 시 (Spring Framework 4.3 이상)

  ```java
  import okhttp3.OkHttpClient;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.http.client.OkHttp3ClientHttpRequestFactory;
  import org.springframework.web.client.RestTemplate;

  ...

  @Bean
  public RestTemplate restTemplate() {
      return new RestTemplate(new OkHttp3ClientHttpRequestFactory(new OkHttpClient()));
  }
  ```

* Apache HttpComponents 4 사용 시 (Spring Framework 4.0 이상 권장)

  ```java
  import org.apache.http.client.HttpClient;
  import org.apache.http.impl.client.HttpClients;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
  import org.springframework.web.client.RestTemplate;

  ...

  @Bean
  public RestTemplate restTemplate() {
      return new RestTemplate(new HttpComponentsClientHttpRequestFactory(HttpClients.createDefault());
  }
  ```

### ClientHttpRequestInterceptor 구현 및 등록

`RestTemplate`을 통해 요청 전, 후로 어떠한 작업을 하고 싶다면, `ClientHttpRequestInterceptor` 인터페이스를 구현해 붙이면 된다. 특히 반복적으로 인증 헤더를 붙인다던지, 요청 로그를 찍을 때 유용하다.

* [ClientHttpRequestInterceptor JavaDoc](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/http/client/ClientHttpRequestInterceptor.html)

```java
...
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.util.StreamUtils;

import java.nio.charset.StandardCharsets;
...

private static final Logger log = LoggerFactory.getLogger(HttpClientConfiguration.class);

@Bean
public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder
        .requestFactory(() -> new OkHttp3ClientHttpRequestFactory(new OkHttpClient()))
        .interceptors(loggingInterceptor())
        .build();
}

private ClientHttpRequestInterceptor loggingInterceptor() {
    return (req, reqBody, execution) -> {
        // HTTP Request 로그
        if (log.isDebugEnabled()) {
            log.debug("req {} {}, body: {}", req.getMethodValue(), req.getURI(), new String(reqBody, StandardCharsets.UTF_8));
        }

        // HTTP 요청 실행
        ClientHttpResponse res = execution.execute(req, reqBody);

        // HTTP Response 로그
        if (log.isDebugEnabled()) {
            log.debug("res {}, body: {}", res.getRawStatusCode(), StreamUtils.copyToString(res.getBody(), StandardCharsets.UTF_8));
        }
        return res;
    };
}
```

위와 같은 코드로 인터페이스 내에 메소드는 하나이기 때문에, 람다로 구현할 수 있다. 그리고 `RestTemplate` Bean 생성 시에 인터셉터를 삽입해줄 수 있다.

문제는 `ClientHttpRequestInterceptor` 내 응답 `InputStream`을 현재 상태에서 구현 후 사용 시 인터셉터 내에서는 데이터가 존재하지만, 실제로 `RestTemplate` 객체를 사용한 곳에 리턴 값이 존재하지 않아 예외가 던저지거나 객체 내 값들이 null일 수 있다. 그 이유는 인터셉터 내의 `InputStream`이 한번 사용하면 스트림이 소모되어 다시 사용할 수 없게 되기 때문이다.

그래서 필요한 것이 `BufferingClientHttpRequestFactory` 클래스이다. 이 클래스는 Buffering이 붙은 만큼 `InputStream`을 사용해도 사라지지 않는다.

* [BufferingClientHttpRequestFactory JavaDoc](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/http/client/BufferingClientHttpRequestFactory.html)

이 클래스의 생성자는 간단하게 우리가 실제 사용할 `ClientHttpRequestFactory` 객체를 한번 감싸주면 된다.

```java
...
import org.springframework.http.client.BufferingClientHttpRequestFactory;
...

@Bean
public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder
        .requestFactory(() -> new BufferingClientHttpRequestFactory(
                          new OkHttp3ClientHttpRequestFactory(new OkHttpClient())
                        ))
        .build();
}
```

## RestTemplate 잘 사용하기

먼저 `RestTemplate` 사용 시엔 `new`로 생성하는 것보다, 위에서 `Bean` 등록한 객체를 `Autowired` 또는 생성자로 필드 선언하여 Bean 객체를 의존 주입으로 받아 사용하는 것이 좋다.

```java
@Autowired
private RestTemplate restTemplate;
```

위에서 두 `exchange` 메소드를 언급했었다.

- `ResponseEntity<T> exchange(RequestEntity<?> entity, Class<T> responseType)`

- `ResponseEntity<T> exchange(RequestEntity<?> entity, ParameterizedTypeReference<T> responseType)`

이 메소드를 이용해 HTTP 요청을 보내고 받는 코드 예제를 보여줄 것이다. 이 두 메소드의 차이는 리턴 타입을 받는 형태의 차이이다. 위의 클래스 타입을 매개변수로 받는 메소드는 간단한 `String.class`, `byte[].class`, 제네릭 클래스가 아닌 모든 클래스 타입에 사용할 수 있다.

만약 Map, List나 기타 컬렉션류, 제네릭 클래스일 경우엔 아래의 [ParameterizedTypeReference](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/core/ParameterizedTypeReference.html) 클래스를 객체로 생성하여 타입을 받을 수 있다. 여기서 `ParameterizedTypeReference<T>` 생성자는 `protected`이지만 익명 객체로 아무런 구현 없이 `new ParameterizedTypeReference<Map<String, Object>>() {}`와 같이 생성할 수 있다.

- [Spring RequestEntity JavaDoc (Spring Framework 4.1 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/http/RequestEntity.html)

메소드에서 매개변수로 받는 `RequestEntity`는 `HttpEntity`의 자식 클래스로, 다용도로 사용 할 수 있다. 특히 컨트롤러에서 다음과 같이 매개변수로 선언 시 요청 데이터들을 사용할 수 있다.

```java
@PostMapping("/post")
public void post(RequestEntity<Map<String, Object>> request) {
    ...
}
```

여기서는 RestTemplate의 요청 객체로 선언하여 사용할 수 있다. 아래는 간단한 예제이다.

```java
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.RequestEntity;

import java.net.URI;

...

private Map<String, Object> request() {
    RequestEntity<Void> request = RequestEntity.get(URI.create("https://httpbin.org/anything")).build();
    return restTemplate.exchange(request, new ParameterizedTypeReference<>() {}).getBody();
}
```

조금 더 복잡한 예제인 아래와 같은 형태의 JSON 문서를 POST 요청 하는 것을 해보자.

```json
{
    "send_id": 1,
    "message": "Hello, Mike!",
    "receiver": ["mike", "anonymous"],
    "is_beep": true
}
```

이러한 JSON 객체를 요청하고 응답하는 메소드는 다음과 같다.

```java
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.RequestEntity;

import java.net.URI;
import java.util.Arrays;
import java.util.Map;
import java.util.HashMap;

...

private Map<String, Object> request() {
    Map<String, Object> requestBodyMap = new HashMap<>();

    requestBodyMap.put("send_id", 1);
    requestBodyMap.put("message", "Hello, Mike!");
    requestBodyMap.put("receiver", Arrays.asList("mike", "anonymous"));
    requestBodyMap.put("is_beep", true);

    RequestEntity<Map<String, Object>> request =
            RequestEntity.post(URI.create("https://httpbin.org/anything"))
                        .body(requestBodyMap);

    return restTemplate.exchange(request, new ParameterizedTypeReference<>() {}).getBody();
}
```

간단한 형태는 `Map<String, Object>` 타입으로 주고 받을 수 있고, 복잡한 형태는 Value Object 클래스를 만들어 사용할 수도 있다.


### Exception

`RestTemplate` 객체를 사용해서 HTTP 요청 후 응답 상태가 만약 [4XX](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#client_error_responses)를 받게 된다면, `HttpClientErrorException` 예외를 던진다. [5XX](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status#server_error_responses)일 경우엔 `HttpServerErrorException` 예외를 던진다. 이 두 예외 클래스는 `HttpStatusCodeException`의 자식 클래스이다. 사실 이 예외 클래스들은 계층 관계가 복잡한데, 여러가지 예외 형태를 묶다보니 생긴 일인 듯 하다.

또한 `RestTemplate` 사용 중 발생하는 예외는 `RestClientException`의 하위 클래스이다.

* [RestClientException (Spring 3.0 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/RestClientException.html)

  - 모든 RestTemplate의 Exception 처리를 담당한다. HTTP 응답 에러 뿐만 아니라 I/O 에러, JSON 이나 XML 포멧 등을 처리 하기 위한 MessageConverter 작동 중 에러 등 모두를 포함한다.

* [RestClientResponseException (Spring 4.3 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/RestClientResponseException.html)

  - HTTP 응답 에러를 위한 예외 클래스이다. 모든 응답 상태를 잡으려면 이 예외 발생 객체를 처리하면 된다.

* [UnknownHttpStatusCodeException (Spring 4.2 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/UnknownHttpStatusCodeException.html)
  
  - HTTP 응답 에러 중 표준이 아닌 알 수 없는 응답 상태에 대한 예외 처리를 위한 클래스이다.

* [HttpStatusCodeException (Spring 3.0 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/HttpStatusCodeException.html)

  - HTTP 응답 에러 중 알려진 응답 상태에 대한 예외 처리를 위한 클래스이다.

* [HttpClientErrorException (Spring 3.0 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/HttpClientErrorException.html)

  - HTTP 4XX Client Error 응답 시에 던져지는 예외 클래스이다.

* [HttpServerErrorException (Spring 3.0 이상)](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/client/HttpServerErrorException.html)

  - HTTP 5XX Server Error 응답 시에 던져지는 예외 클래스이다.

위와 같은 예외 클래스 계층이 존재하며, 필요한 목적에 따라 사용할 수 있다. 보통은 `RestClientResponseException` 타입만 `catch`해서 사용하면 응답하는 모든 상태에 대한 처리를 할 수 있다.

## 번외, UriComponentBuilder

`URI.create(String uri)` 정적 생성 메소드는 문자열로 간단하게 URI 객체를 생성할 수 있지만 매개변수 문자열이 잘못된 형식일 때 `URISyntaxException` 예외가 던져지는 위험이 존재하고, URI 형태가 비슷하면서 어떠한 경로나 쿼리 변수만 다르게 템플릿화 시키고 싶다면 문자열로 직접 사용하는 것은 하드코딩이 될 수 있다.

그래서 안전하고 템플릿 형태로 만들 수 있는 `UriComponentsBuilder` 클래스와 `UriComponents` 클래스 두개의 사용법을 간단히 소개한다.

* [Spring UriComponentsBuilder JavaDoc](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/util/UriComponentsBuilder.html)
* [Spring UriComponents JavaDoc](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/util/UriComponents.html)
* [JDK java.net.URI JavaDoc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/net/URI.html)

`UriComponentsBuilder` 클래스는 new 생성자가 아닌, 정적 생성 메서드가 몇가지 존재한다.

* `UriComponentsBuilder.fromHttpUrl(String httpUrl)`
  - 일반적으로 사용하는 주소창의 URL을 문자열로 받는다.
  - `https://www.google.com/search?q=resttemplate` 과 같은 형태.

* `UriComponentsBuilder.fromPath(String path)`
  - 프로토콜이나 도메인이 빠진 경로 형태의 문자열을 받을 수 있다.
  - `/auth/tokens` 와 같은 형태
  - `{variable}`과 같이 템플릿을 위한 변수 설정도 가능하다.

* `UriComponentsBuilder.fromUri(URI uri)`
  - `java.net.URI` 객체를 받는 형태

* `UriComponentsBuilder.fromUriString(String uri)`
  - URI의 문자열 형태를 받을 수 있다.
  - URI는 웹 주소 뿐만 아니라 `tel:+82-2-0000-0000`, `mailto:someone@example.com?subject=hello` 같은 형태도 가능.

* `UriComponentsBuilder.newInstance()`
  - 아무런 매개변수 없이 일단 객체를 생성한다.

* 기타 다른 메소드는 JavaDoc 참조

정적 생성 메서드를 매개로 Builder 패턴으로 필요한 것을 붙여가며 객체 생성이 가능하다. 주요 빌더 메서드들은 다음과 같다.

* `scheme(String scheme)`, `host(String host)`, `port(int port)`
  - scheme는 http, https, ftp와 같은 프로토콜이 될 수 있다. 또는 tel, mailto와 같은 정보 단위일 수 있다.
  - host는 `google.com`, `youtube.com`, `127.0.0.1`과 같은 ip나 hostname 이다.
  - port는 프로토콜 통신 시 HTTP 80, HTTPS 443과 같은 기본 포트가 아닐 시 지정한다.

* `path(String path)`, `pathSegment(String... pathSegments)`
  - path는 `/`부터 시작하며 `/v3/auth/tokens`와 같은 위치를 나타낸다.
  - pathSegment는 경로의 조각이며 `auth/token`과 같은 조각을 뜻한다.

* `replacePath(String path)`
  - replacePath는 이미 지정한 경로를 지우고 새로 지정하는 것이다.

* `queryParam(String name, Object... values)`, `queryParam(String name, Collection<?> values)`
  - queryParam은 `?` 뒤의 매개변수를 지정하는 것이며, URI에서 `?query=http&page=2`와 같은 형태다.
  - values 뒤에 여러 개의 객체 또는 객체 컬렉션을 지정할 수 있으며, `?query=first&query=second`와 같이 붙는다.

* `queryParams(MultiValueMap<String, String> params)`
  - [MultiValueMap](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/util/MultiValueMap.html)으로 위의 파라메터를 한꺼번에 지정할 수 있으며, 사용 시엔 구현체인 [LinkedMultiValueMap](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/util/LinkedMultiValueMap.html) 클래스를 객체로 생성하여 사용한다.

* `uri(URI uri)`, `uriComponents(UriComponents uriComponents)`
  - URI 객체나 UriComponents 객체를 합칠 수 있다.

* `UriComponents build()`, `UriComponents buildAndExpand(Object... uriVariableValues)`
  - 모두 다 조합 후 build 메소드를 통해 UriComponents 객체를 생성한다.
  - 만약 Path 내 `{variable}`과 같은 템플릿 변수가 있다면 buildAndExpand를 통해 앞에서부터 순서대로 문자열이나 toString이 가능한 객체를 삽입시킨다.

* 기타 다른 메소드는 JavaDoc 참조

만약 `UriComponentsBuilder` 객체를 통해 `UriComponents` 객체를 생성했다면, `toUri()` 메소드를 통해 우리가 사용하기 위한 실제 `java.net.URI` 객체를 생성하여 사용할 수 있다.

```java
UriComponents hostUriComponents = UriComponentsBuilder.fromHttpUrl("https://www.example.com").build();
URI uri = UriComponentsBuilder.fromPath("/test/{testNumber}")
              .uriComponents(hostUriComponents)
              .queryParam("query", "hello")
              .buildAndExpand("1").toUri();
```

URI 구조가 복잡하거나, 코드 분기에 따라 URI 형태가 바뀌어야 한다면 `UriComponentsBuilder` 클래스를 잘 사용하면 쉽게 URI 객체를 생성할 수 있다.

## 마치며

스프링은 HTTP 요청 시에 [Spring WebFlux](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web-reactive.html) 내 [WebClient](https://docs.spring.io/spring-framework/docs/5.3.x/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html) 클래스를 사용하는 것을 Spring Framework 5 부턴 권장하고 있다. 하지만 서블릿 기반으로 개발한 코드 위에 `WebClient` 하나 만을 위해 라이브러리 의존성 추가 하는 것도 이상하다. 또한 [Project Reactor](https://projectreactor.io/) 라이브러리의 비동기 코드를 이해해야 사용하기 수월하다.

따라서 아직도 서블릿 기반 코드에서는 `RestTemplate` 클래스가 자주 사용하게 된다. 하지만 부적절하게 사용되는 케이스를 많이 보았었다. 이 글이 `RestTemplate` 클래스를 잘 사용할 수 있게 도와주는 가이드 역할이 되었으면 한다.
