---
title: "스프링 프레임워크에서 StreamUtils, Resource를 이용해 파일 읽어오기"
subtitle: "스프링 프레임워크의 StreamUtils, Resource를 사용하며 간단한 IO 처리를 하기 위한 팁들"
categories: ['java', 'spring']
tags: ['java', 'spring', 'io', 'utils', 'resources', 'streamutils', 'classpath']
date: 2021-10-23T16:07:04+09:00
slug: spring-utils-io-resources
aliases: ['/post/2021/io-utils-in-spring-framework']
---

스프링에는 유용한 도구들이 많이 준비되어있다. 하지만 주변에서는 그런 존재를 몰라  라이브러리를 따로 사용하는 경우가 많았다. 이 글에서는 스프링 프레임워크의 StreamUtils, Resource 인터페이스와 그 구현체 클래스를 활용하는 팁들을 작성했다.

## StreamUtils 유틸 클래스

- [Spring Framework StreamUtils JavaDoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/StreamUtils.html)
- Spring Framwork 3.2.2 부터 사용 가능

이 유틸 클래스는 스프링 프레임워크 내에 내장되어 있고, Java Input/Output Stream을 다루기 위한 것이다. 실제로 자바에서 Input/Output Stream 다루기엔 보일러플레이트(사전 준비) 코드들이 많다.

아래는 InputStream을 String으로 바꾸는 코드이다.
* 코드 출처 : [How do I read / convert an InputStream into a String in Java? - Stack Overflow](https://stackoverflow.com/a/35446009)

``` java
// 가장 빠르다던 ByteArrayOutputStream 이용: https://stackoverflow.com/a/35446009
public String streamToString(InputStream inputStream) throws IOException {
    ByteArrayOutputStream result = new ByteArrayOutputStream();
    byte[] buffer = new byte[1024];
    for (int length; (length = inputStream.read(buffer)) != -1; ) {
        result.write(buffer, 0, length);
    }
    // JDK 1.7 미만 사용 시 "UTF-8"로 대체
    return result.toString(StandardCharsets.UTF_8.name());
}
...
String str = streamToString(inputStream);
```

물론 위 코드를 유틸클래스를 만들어서 구현하는 방법도 괜찮다. Java 내에선 가장 빠른 코드이기 때문이다. 하지만 저 코드가 생각나지 않을 때는 어떻게 할까? 아니면 String이 아니라 다른 형태로 받고 싶다면? `commons-io` 라이브러리의 `IOUtils` 유틸 클래스와 비슷하게, 스프링 프레임워크는 이미 `StreamUtils` 유틸 클래스를 준비해두었다.

```java
import org.springframework.util.StreamUtils;
import java.nio.charset.StandardCharsets;
...
String str = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);
```

### 자주 쓰는 메소드

1. `static byte[] copyToByteArray(InputStream in)`
   * InputStream을 byte 배열로 반환한다.

2. `static String copyToString(InputStream in, Charset charset)`
   * InputStream을 문자열로 반환한다.

3. `static InputStream emptyInput()`
   * 빈 InputStream을 반환한다.


**1번**인 InputStream을 Byte 배열로 변환하는 메소드와 **2번**인 InputStream을 String으로 변환하는 메소드가 가장 많이 쓰일 것이다.

IO 처리 시에는 특히나 byte 배열을 처리할 일이 많다. Base64 인코딩을 하거나 소켓을 사용할 때에도 byte 배열을 기준으로 사용한다. 

**3번**인 emptyInput() 메소드는 얼핏 보면 무가치 할 것 같지만 의외의 유용성이 높은데, InputStream을 매개변수로 주거나 반환할 때 null 대신 쓰이기 좋을 것이다.


## 표준 문자셋

* [java.nio.charset.StandardCharsets JavaDoc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/nio/charset/StandardCharsets.html)

잠깐 짚고 넘어갈 표준 문자셋은 JDK 1.7부터 존재한다. 실제로 1.7 이전에는 문자셋 설정을 스트링으로 받았었다.

```java
String str = new String(byteArray, "UTF-8");
String str = new String(byteArray, Charset.forName("UTF-8"));
```

이런 식으로 처리 했어야 했는데, 문제는 String 타입의 매개변수의 문자셋 코드가 잘못 입력했을 때이다. 그땐 UnsupportEncodingException으로 인해 프로그램이 중단된다. IOException의 자식 클래스여서 더 심각했다.

그래서 표준 문자셋 객체 상수를 StandardCharsets 클래스에 정의하였다. 따라서 이제는 다음과 같이 안전하게 처리할 수 있다.

```java
String str = new String(byteArray, StandardCharsets.UTF_8);
```

만약, EUC-KR을 쓰고 싶다면? Exception을 처리해야 할 위험성이 계속 존재하지만 다음과 같은 방법으로 해야 한다.

```java
String str = new String(byteArray, Charset.forName("EUC-KR"));
```

## FileCopyUtils 유틸 클래스

* [Spring Framework FileCopyUtils JavaDoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/util/FileCopyUtils.html)

위의 StreamUtils 유틸 클래스는 버전 3.2.2 부터 생긴 반면, FileCopyUtils는 초기 버전에도 존재했다. 유틸 클래스 이름 그대로 파일을 복사하는 정적 메소드가 담겨있다. 자세한 내용은 JavaDoc을 참조 바란다.

물론 JDK 1.7이상이라면 [java.nio.file.Files](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/nio/file/Files.html) 클래스를 사용하는 것이 더 좋다. 더 편리하고 다양한 유틸리티 정적 메소드들이 존재한다.

## Resource 인터페이스와 구현체

* [Spring Framework Manual](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#resources)
* [Spring Framework Resource JavaDoc](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/Resource.html)

Resource는 인터페이스다. 그냥 사용할 수는 없고 다음과 같은 구현된 클래스가 존재한다.

* [UrlResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/UrlResource.html)
* [ClassPathResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/ClassPathResource.html)
* [FileSystemResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/FileSystemResource.html)
* [PathResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/PathResource.html)
* [ServletContextResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/context/support/ServletContextResource.html)
* [InputStreamResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/InputStreamResource.html)
* [ByteArrayResource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/io/ByteArrayResource.html)

jar과 같이 묶이는 파일은 `ClassPathResource`를 이용하고, 일반적인 파일을 읽어올 땐 `FileSystemResource` 를 사용한다. 따라서 리소스에 텍스트 파일을 저장하고 읽어오고 싶을 땐 다음과 같은 방법을 사용할 수 있다.

``` java
import org.springframework.core.io.Resource;
import org.springframework.core.io.ClassPathResource;
import org.springframework.util.StreamUtils;
import java.nio.charset.StandardCharsets;
...
Resource resource = new ClassPathResource("load.txt");
String loaded = StreamUtils.copyToString(resource.getInputStream(), StandardCharsets.UTF_8);
```

간편한 방법이지만 이건 경로를 직접 하드코딩 해야하는 단점도 존재한다. 스프링 프레임워크를 쓰는 이유가 강력한 Dependency Injection(의존 주입)에 의한 IoC 컨테이너를 사용 하기 위함도 있을 것이다.


특히 스프링 부트를 사용한다면 application.properties (또는 yaml)의 존재를 알 것이다. properties와 [@Value](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/annotation/Value.html) 애노테이션의 조합으로 DI를 이용해 불러 올 수도 있다.


``` properties
load.resource=classpath:load.txt
```

``` java
@Component
public class LoadResource {

   private final String loaded;

   @Autowired
   public LoadResource(@Value("${load.resource}") Resource loaded) {
      this.loaded = StreamUtils.copyToStream(loaded.getInputStream(), StandardCharsets.UTF_8);
   }

   public String getLoaded() {
      return loaded;
   }
}
```

또는 이와 같이 Value 애노테이션에 하드코딩하여 넣을 수도 있다.
``` java
@Value("classpath:load.txt")
```


## 마치며

물론 스프링 프레임워크를 사용하지 않아도, Java를 사용한다면 아래와 같은 친숙한 라이브러리들이 있다.


* [commons-io](https://commons.apache.org/proper/commons-io/)
   ``` java
   String str = IOUtils.toString(inputStream, StandardCharsets.UTF_8);
   ```

* [Guava](https://guava.dev/)
   * InputStream을 String으로 변환.
    ``` java
    ByteSource source = new ByteSource() {
    @Override
    public InputStream openStream() throws IOException {
            return inputStream;
    }
    };
    String str = source.asCharSource(StandardCharsets.UTF_8).read();
    ```
   * classpath 내 파일을 읽어 String으로 가져오기.
   ``` java
   String str = Resources.toString(Resources.getResource("load.txt"), StandardCharsets.UTF_8);
   ```

* JDK 9 이상
   ```java
   String str = new String(inputStream.readAllBytes(), StandardCharsets.UTF_8);
   ```



Apache Commons와 Google Guava는 둘 다 많이 찾고 쓰는 자바 라이브러리 묶음들이다. IO부분은 commons-io 라이브러리가 간편하고 강력한 편이지만 개인적으론 Guava가 라이브러리 구성이 좋아 선호한다.


최근 JDK 버전들은 유틸 클래스 등도 강화가 많이 되어, 특정 라이브러리가 없어도 개발이 많이 편해졌다. 그래도 스프링 프레임워크의 강력한 기능과 엮어 사용하면 더욱 더 강력하고 객체지향적이게 사용할 수 있다.

