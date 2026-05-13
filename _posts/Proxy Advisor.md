**정확도: 94%**

# `ProceedingJoinPoint.proceed()`를 사용한 Proxy Advisor 구현

## 1. 결론

가능하다. 다만 정확히 말하면 `ProceedingJoinPoint.proceed()`는 Spring의 저수준 `Advisor` API가 아니라 **`@Aspect` + `@Around` 방식의 AspectJ 스타일 AOP**에서 사용하는 방식이다. Spring AOP의 Around Advice는 대상 메서드 실행 전후에 로직을 넣을 수 있고, 대상 메서드를 실제 실행하려면 `proceed()`를 호출해야 한다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/advice.html?utm_source=chatgpt.com "Declaring Advice :: Spring Framework"))  
권장 구조는 아래와 같다.

```text
@UseProxy 선언
  ↓
@Around Advice 실행
  ↓
ThreadLocal에 Proxy 정보 저장
  ↓
proceedingJoinPoint.proceed()
  ↓
외부 통신 Client에서 ProxyContext 확인
  ↓
URL.openConnection(proxy)
  ↓
finally에서 ThreadLocal 제거
```

`System.setProperty()`는 JVM 전체에 영향을 주므로 Advisor/Aspect 안에서 사용하지 않는 것이 좋다. 대신 실제 HTTP 호출 시 `URL.openConnection(Proxy)`를 사용해야 요청 단위 Proxy 적용이 가능하다.

## 2. 구현 구조

|구분|역할|
|---|---|
|`@UseProxy`|Proxy 사용 대상 메서드 표시|
|`ProxySpec`|Proxy 서버 정보 보관|
|`ProxyContextHolder`|현재 실행 Thread의 Proxy 정보 저장|
|`ProxyAspect`|`ProceedingJoinPoint.proceed()` 전후로 ProxyContext 설정/해제|
|`ExternalHttpClient`|`ProxyContextHolder`를 보고 `openConnection(proxy)` 사용|

## 3. Annotation 정의

```java
package com.example.proxy;
import java.lang.annotation.*;
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface UseProxy {
    String host();
    int port();
}
```

## 4. Proxy 정보 DTO

```java
package com.example.proxy;
import java.net.InetSocketAddress;
import java.net.Proxy;
public class ProxySpec {
    private final String host;
    private final int port;
    public ProxySpec(String host, int port) {
        this.host = host;
        this.port = port;
    }
    public Proxy toJavaProxy() {
        return new Proxy(
            Proxy.Type.HTTP,
            new InetSocketAddress(host, port)
        );
    }
    public String getHost() {
        return host;
    }
    public int getPort() {
        return port;
    }
}
```

## 5. ThreadLocal Context

```java
package com.example.proxy;
public final class ProxyContextHolder {
    private static final ThreadLocal<ProxySpec> HOLDER = new ThreadLocal<>();
    private ProxyContextHolder() {
    }
    public static void set(ProxySpec proxySpec) {
        HOLDER.set(proxySpec);
    }
    public static ProxySpec get() {
        return HOLDER.get();
    }
    public static void clear() {
        HOLDER.remove();
    }
}
```

### 주의점

`ThreadLocal`은 WAS Thread Pool 환경에서 반드시 `finally`로 제거해야 한다. 제거하지 않으면 같은 Thread를 재사용하는 다음 요청에 이전 Proxy 정보가 남을 수 있다.

## 6. `ProceedingJoinPoint.proceed()` 기반 Aspect

```java
package com.example.proxy;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import java.lang.reflect.Method;
@Aspect
@Component
@Order(1)
public class ProxyAspect {
    @Around("@annotation(com.example.proxy.UseProxy) || @within(com.example.proxy.UseProxy)")
    public Object applyProxy(ProceedingJoinPoint joinPoint) throws Throwable {
        UseProxy useProxy = findUseProxy(joinPoint);
        if (useProxy == null) {
            return joinPoint.proceed();
        }
        ProxySpec proxySpec = new ProxySpec(useProxy.host(), useProxy.port());
        try {
            ProxyContextHolder.set(proxySpec);
            return joinPoint.proceed();
        } finally {
            ProxyContextHolder.clear();
        }
    }
    private UseProxy findUseProxy(ProceedingJoinPoint joinPoint) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        UseProxy methodAnnotation = AnnotationUtils.findAnnotation(method, UseProxy.class);
        if (methodAnnotation != null) {
            return methodAnnotation;
        }
        Class<?> targetClass = joinPoint.getTarget().getClass();
        return AnnotationUtils.findAnnotation(targetClass, UseProxy.class);
    }
}
```

## 7. 외부 통신 Client 수정

핵심은 `System.setProperty()`를 제거하고, `ProxyContextHolder`에 값이 있으면 `url.openConnection(proxy)`를 사용하는 것이다.

```java
package com.example.http;
import com.example.proxy.ProxyContextHolder;
import com.example.proxy.ProxySpec;
import org.springframework.stereotype.Component;
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;
@Component
public class ExternalHttpClient {
    public String get(String targetUrl) throws IOException {
        URL url = new URL(targetUrl);
        HttpURLConnection conn = openConnection(url);
        conn.setRequestMethod("GET");
        conn.setConnectTimeout(3000);
        conn.setReadTimeout(5000);
        conn.setUseCaches(false);
        try {
            int status = conn.getResponseCode();
            InputStream inputStream = status >= 200 && status < 300
                ? conn.getInputStream()
                : conn.getErrorStream();
            return readBody(inputStream);
        } finally {
            conn.disconnect();
        }
    }
    private HttpURLConnection openConnection(URL url) throws IOException {
        ProxySpec proxySpec = ProxyContextHolder.get();
        if (proxySpec == null) {
            return (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);
        }
        return (HttpURLConnection) url.openConnection(proxySpec.toJavaProxy());
    }
    private String readBody(InputStream inputStream) throws IOException {
        if (inputStream == null) {
            return "";
        }
        try (BufferedReader br = new BufferedReader(
            new InputStreamReader(inputStream, StandardCharsets.UTF_8))) {
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = br.readLine()) != null) {
                sb.append(line);
            }
            return sb.toString();
        }
    }
}
```

## 8. Service 적용 예시

```java
package com.example.service;
import com.example.http.ExternalHttpClient;
import com.example.proxy.UseProxy;
import org.springframework.stereotype.Service;
@Service
public class ExternalLinkService {
    private final ExternalHttpClient externalHttpClient;
    public ExternalLinkService(ExternalHttpClient externalHttpClient) {
        this.externalHttpClient = externalHttpClient;
    }
    @UseProxy(host = "10.10.10.10", port = 8080)
    public String callByProxy() throws Exception {
        return externalHttpClient.get("https://api.example.com/test");
    }
    public String callDirect() throws Exception {
        return externalHttpClient.get("https://internal.example.com/health");
    }
}
```

## 9. Spring Java Config

Spring Boot 또는 Java Config 기반이면 아래 설정을 추가한다.

```java
package com.example.config;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class AopConfig {
}
```

Spring AOP는 프록시 기반으로 동작하며, JDK Dynamic Proxy 또는 CGLIB 프록시를 사용한다. 인터페이스가 있으면 JDK Dynamic Proxy가 사용될 수 있고, `proxyTargetClass = true`를 사용하면 CGLIB 기반 클래스 프록시를 강제할 수 있다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))

## 10. XML 기반 Spring 설정

레거시 Spring MVC, JBoss7, `dispatcher-servlet.xml` 기반이면 다음처럼 설정한다.

```xml
<!-- AspectJ annotation 기반 AOP 활성화 -->
<aop:aspectj-autoproxy proxy-target-class="true"/>
<!-- Aspect Bean 등록 -->
<bean id="proxyAspect" class="com.example.proxy.ProxyAspect"/>
<!-- 외부 통신 Client -->
<bean id="externalHttpClient" class="com.example.http.ExternalHttpClient"/>
```

필요 namespace:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
         http://www.springframework.org/schema/beans
         https://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/aop
         https://www.springframework.org/schema/aop/spring-aop.xsd">
```

## 11. 기존 코드 수정 포인트

### 기존 방식

```java
System.setProperty("http.proxyHost", "10.10.10.10");
System.setProperty("http.proxyPort", "8080");
URL url = new URL(targetUrl);
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
```

### 변경 방식

```java
@UseProxy(host = "10.10.10.10", port = 8080)
public String callByProxy() throws Exception {
    return externalHttpClient.get("https://api.example.com/test");
}
```

실제 통신부:

```java
ProxySpec proxySpec = ProxyContextHolder.get();
HttpURLConnection conn = proxySpec == null
    ? (HttpURLConnection) url.openConnection(Proxy.NO_PROXY)
    : (HttpURLConnection) url.openConnection(proxySpec.toJavaProxy());
```

## 12. 중요 주의점

|항목|주의 내용|
|---|---|
|`proceed()`|호출하지 않으면 대상 메서드가 실행되지 않는다.|
|`finally`|`ProxyContextHolder.clear()`는 반드시 `finally`에서 실행한다.|
|`System.setProperty()`|Aspect 안에서도 사용하지 않는다. JVM 전체에 영향이 있다.|
|Self Invocation|같은 클래스 내부에서 `this.callByProxy()`처럼 호출하면 Spring AOP가 적용되지 않는다. Spring 문서는 self invocation이 프록시를 우회한다고 설명한다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))|
|`private` 메서드|Spring AOP 적용 대상이 아니다.|
|`final` 메서드|CGLIB에서도 override가 불가능하므로 Advice 적용이 제한된다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))|
|비동기|`@Async`, 별도 Executor, 새 Thread에서는 일반 `ThreadLocal`이 전달되지 않는다.|
|프록시 인증|`HttpURLConnection`보다 Apache HttpClient 또는 Java 11 `HttpClient`가 관리하기 쉽다.|
|외부 호출 집중|`new URL(...).openConnection()`이 여러 곳에 흩어져 있으면 Aspect만으로 통제하기 어렵다. 공통 Client로 모아야 한다.|

## 13. `@UseProxy`를 클래스 전체에 적용하는 예

```java
@UseProxy(host = "10.10.10.10", port = 8080)
@Service
public class ProxyExternalService {
    private final ExternalHttpClient externalHttpClient;
    public ProxyExternalService(ExternalHttpClient externalHttpClient) {
        this.externalHttpClient = externalHttpClient;
    }
    public String callA() throws Exception {
        return externalHttpClient.get("https://api-a.example.com/test");
    }
    public String callB() throws Exception {
        return externalHttpClient.get("https://api-b.example.com/test");
    }
}
```

이 경우 클래스 내 모든 외부 호출 메서드가 동일 ProxyContext를 사용한다. 단, 외부에서 Spring Bean 프록시를 통해 호출되어야 한다.

## 14. 최종 권장안

```text
1. System.setProperty 방식 제거
2. 외부 HTTP 호출 코드를 ExternalHttpClient로 집중
3. @UseProxy Annotation 정의
4. @Around + ProceedingJoinPoint.proceed()로 ProxyContext 설정
5. ExternalHttpClient에서 openConnection(proxy) 사용
6. finally에서 ThreadLocal clear
7. Self Invocation, final/private 메서드 여부 점검
8. 동시 요청 테스트로 Proxy 혼선 여부 검증
```

## 15. 최종 코드 핵심

```java
@Around("@annotation(com.example.proxy.UseProxy) || @within(com.example.proxy.UseProxy)")
public Object applyProxy(ProceedingJoinPoint joinPoint) throws Throwable {
    UseProxy useProxy = findUseProxy(joinPoint);
    if (useProxy == null) {
        return joinPoint.proceed();
    }
    try {
        ProxyContextHolder.set(new ProxySpec(useProxy.host(), useProxy.port()));
        return joinPoint.proceed();
    } finally {
        ProxyContextHolder.clear();
    }
}
```

결론적으로 `ProceedingJoinPoint.proceed()` 기반으로 구현 가능하다. 다만 Advisor/Aspect의 역할은 **Proxy 서버 정보를 전역 설정하는 것**이 아니라, 현재 호출 흐름에만 ProxyContext를 지정하는 것이다. 실제 Proxy 적용은 반드시 `HttpURLConnection` 생성 시점에 `URL.openConnection(proxy)`로 처리하는 구조가 안전하다.

**정확도: 93%**

# Java Spring Web에서 `System.setProperty()` 프록시 임시 설정 코드 수정 방법

## 1. 결론

Spring Web/WAS 환경에서 요청 처리 중 아래처럼 `System.setProperty()`로 프록시를 임시 변경하는 방식은 **수정 대상**이다.

```java
System.setProperty("http.proxyHost", proxyHost);
System.setProperty("http.proxyPort", proxyPort);
HttpURLConnection conn = (HttpURLConnection) url.openConnection();
```

`System.setProperty()`는 요청 단위가 아니라 **JVM 전체 전역 설정**이다. 즉, 같은 WAS 안에서 동시에 실행 중인 다른 요청, 배치, 외부 연계 API 호출에도 영향을 줄 수 있다. Java는 `http.proxyHost`, `http.proxyPort`, `https.proxyHost`, `https.proxyPort` 같은 네트워크 프록시 시스템 속성을 제공하지만, 이는 JVM 네트워크 처리에 사용되는 전역 성격의 설정이다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/doc-files/net-properties.html?utm_source=chatgpt.com "Networking Properties"))  
따라서 수정 방향은 아래와 같다.

```text
기존: System.setProperty()로 전역 프록시 변경
수정: URL.openConnection(Proxy proxy)로 요청별 프록시 지정
```

Java 공식 문서도 `URL.openConnection(Proxy)`가 지정한 프록시를 통해 연결하며, 시스템 기본 `ProxySelector` 설정보다 우선한다고 설명한다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/URL.html?utm_source=chatgpt.com "URL (Java Platform SE 8 )"))

## 2. 기존 방식의 문제점

|항목|문제|
|---|---|
|전역 영향|`System.setProperty()`는 현재 요청만이 아니라 WAS JVM 전체에 영향을 준다.|
|동시성 위험|A 요청이 프록시를 설정한 순간 B 요청도 같은 프록시를 탈 수 있다.|
|원복 누락|예외 발생 시 기존 프록시 설정을 복구하지 못하면 장애가 이어질 수 있다.|
|프록시 혼선|외부 API별로 다른 프록시를 써야 하는 경우 설정이 섞일 수 있다.|
|테스트 어려움|요청별 프록시 사용 여부를 추적하기 어렵다.|
|보안 위험|프록시 계정/비밀번호를 시스템 속성이나 로그에 노출할 가능성이 있다.|
|운영 장애|같은 WAS에서 다른 모듈의 외부 통신까지 프록시 영향을 받을 수 있다.|

## 3. 수정 전 코드 예시

```java
public String callExternalApi(String targetUrl) throws IOException {
    System.setProperty("http.proxyHost", "10.10.10.10");
    System.setProperty("http.proxyPort", "8080");
    URL url = new URL(targetUrl);
    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
    conn.setRequestMethod("GET");
    conn.setConnectTimeout(3000);
    conn.setReadTimeout(5000);
    int status = conn.getResponseCode();
    return readBody(conn);
}
```

이 방식은 단일 실행 프로그램에서는 동작할 수 있지만, Spring Web/WAS처럼 다중 요청이 동시에 처리되는 환경에서는 권장하기 어렵다.

## 4. 수정 후 권장 방식: 요청별 `Proxy` 객체 사용

### 4.1 기본 수정 코드

```java
public String callExternalApi(String targetUrl) throws IOException {
    Proxy proxy = new Proxy(
            Proxy.Type.HTTP,
            new InetSocketAddress("10.10.10.10", 8080)
    );
    URL url = new URL(targetUrl);
    HttpURLConnection conn = (HttpURLConnection) url.openConnection(proxy);
    conn.setRequestMethod("GET");
    conn.setConnectTimeout(3000);
    conn.setReadTimeout(5000);
    try {
        int status = conn.getResponseCode();
        return readBody(conn);
    } finally {
        conn.disconnect();
    }
}
```

이 방식은 해당 `HttpURLConnection`에만 프록시를 적용한다. Java 공식 Proxy 가이드는 `openConnection(Proxy)`를 사용하면 지정한 프록시를 통해 연결하고, 시스템 프록시 속성을 포함한 다른 설정을 무시한다고 설명한다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/technotes/guides/net/proxies.html?utm_source=chatgpt.com "Java Networking and Proxies"))

## 5. Spring 서비스 형태로 정리

### 5.1 프록시 설정값 분리

```properties
external.proxy.host=10.10.10.10
external.proxy.port=8080
external.proxy.enabled=true
```

### 5.2 Service 코드

```java
@Service
public class ExternalApiClient {
    @Value("${external.proxy.enabled:false}")
    private boolean proxyEnabled;
    @Value("${external.proxy.host:}")
    private String proxyHost;
    @Value("${external.proxy.port:0}")
    private int proxyPort;
    public String get(String targetUrl) throws IOException {
        URL url = new URL(targetUrl);
        HttpURLConnection conn;
        if (proxyEnabled) {
            Proxy proxy = new Proxy(
                    Proxy.Type.HTTP,
                    new InetSocketAddress(proxyHost, proxyPort)
            );
            conn = (HttpURLConnection) url.openConnection(proxy);
        } else {
            conn = (HttpURLConnection) url.openConnection();
        }
        conn.setRequestMethod("GET");
        conn.setConnectTimeout(3000);
        conn.setReadTimeout(5000);
        conn.setUseCaches(false);
        try {
            int status = conn.getResponseCode();
            if (status >= 200 && status < 300) {
                return readBody(conn.getInputStream());
            }
            return readBody(conn.getErrorStream());
        } finally {
            conn.disconnect();
        }
    }
    private String readBody(InputStream inputStream) throws IOException {
        if (inputStream == null) {
            return "";
        }
        try (BufferedReader br = new BufferedReader(
                new InputStreamReader(inputStream, StandardCharsets.UTF_8))) {
            StringBuilder sb = new StringBuilder();
            String line;
            while ((line = br.readLine()) != null) {
                sb.append(line);
            }
            return sb.toString();
        }
    }
}
```

## 6. 프록시 미사용 대상이 있는 경우

특정 내부망 URL은 프록시를 타면 안 되는 경우가 있다. 이 경우 `Proxy.NO_PROXY` 또는 분기 처리를 사용한다.

```java
private HttpURLConnection openConnection(String targetUrl) throws IOException {
    URL url = new URL(targetUrl);
    if (isInternalUrl(url.getHost())) {
        return (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);
    }
    Proxy proxy = new Proxy(
            Proxy.Type.HTTP,
            new InetSocketAddress(proxyHost, proxyPort)
    );
    return (HttpURLConnection) url.openConnection(proxy);
}
private boolean isInternalUrl(String host) {
    return host.endsWith(".internal.local")
            || host.startsWith("10.")
            || "localhost".equalsIgnoreCase(host);
}
```

`Proxy.NO_PROXY`를 사용하면 해당 연결은 명시적으로 프록시를 사용하지 않는다.

## 7. HTTPS 호출 시 주의점

`https://` URL이라도 HTTP 프록시를 경유하는 경우 일반적으로 `Proxy.Type.HTTP`를 사용한다.

```java
URL url = new URL("https://api.example.com/test");
Proxy proxy = new Proxy(
        Proxy.Type.HTTP,
        new InetSocketAddress("10.10.10.10", 8080)
);
HttpsURLConnection conn = (HttpsURLConnection) url.openConnection(proxy);
```

`https.proxyHost`, `https.proxyPort`를 `System.setProperty()`로 지정하는 방식도 Java 네트워크 속성에 존재하지만, Spring Web 요청 단위 제어에는 부적합하다. 요청별 제어가 필요하면 `openConnection(proxy)`가 더 안전하다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/doc-files/net-properties.html?utm_source=chatgpt.com "Networking Properties"))

## 8. 프록시 인증이 필요한 경우

프록시 서버가 인증을 요구하면 크게 2가지 방식이 있다.

|방식|설명|권장도|
|---|---|---|
|`Authenticator.setDefault()`|JVM 전역 인증자 설정|낮음|
|`Proxy-Authorization` 헤더|요청별 인증 헤더 설정|제한적 사용|
|Apache HttpClient/OkHttp|요청/클라이언트 단위 인증 제어|권장|
|Java `Authenticator.setDefault()`는 프록시 또는 HTTP 서버가 인증을 요구할 때 사용할 기본 인증자를 설정하지만, 이 역시 기본적으로 전역 설정 성격이 강하다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/Authenticator.html?utm_source=chatgpt.com "Authenticator (Java Platform SE 8 )"))|||

### 8.1 Basic Proxy 인증 예시

```java
String proxyUser = "proxyUser";
String proxyPassword = "proxyPassword";
String auth = proxyUser + ":" + proxyPassword;
String encodedAuth = Base64.getEncoder()
        .encodeToString(auth.getBytes(StandardCharsets.UTF_8));
conn.setRequestProperty("Proxy-Authorization", "Basic " + encodedAuth);
```

주의: `Proxy-Authorization`은 프록시 인증용 헤더다. 일반 서버 인증용 `Authorization` 헤더와 다르다. 프록시 계정 정보는 로그에 남기지 않아야 한다.

## 9. 기존 코드를 최소 수정해야 하는 경우

### 9.1 비권장이나 임시 방어 코드

부득이하게 기존 `System.setProperty()` 방식을 당장 제거하지 못하면 최소한 기존 값을 백업하고 `finally`에서 복구해야 한다.

```java
public String callWithSystemProxy(String targetUrl) throws IOException {
    String oldHttpProxyHost = System.getProperty("http.proxyHost");
    String oldHttpProxyPort = System.getProperty("http.proxyPort");
    try {
        System.setProperty("http.proxyHost", "10.10.10.10");
        System.setProperty("http.proxyPort", "8080");
        URL url = new URL(targetUrl);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setConnectTimeout(3000);
        conn.setReadTimeout(5000);
        try {
            return readBody(conn.getInputStream());
        } finally {
            conn.disconnect();
        }
    } finally {
        restoreProperty("http.proxyHost", oldHttpProxyHost);
        restoreProperty("http.proxyPort", oldHttpProxyPort);
    }
}
private void restoreProperty(String key, String oldValue) {
    if (oldValue == null) {
        System.clearProperty(key);
    } else {
        System.setProperty(key, oldValue);
    }
}
```

단, 이 방식도 동시 요청 문제는 해결하지 못한다. `try-finally`는 원복 누락 위험만 줄일 뿐, WAS 전체에 순간적으로 영향을 주는 문제는 그대로 남는다.

## 10. 동시성까지 막기 위한 임시 Lock 방식

```java
private static final Object PROXY_LOCK = new Object();
public String callWithProxyLock(String targetUrl) throws IOException {
    synchronized (PROXY_LOCK) {
        return callWithSystemProxy(targetUrl);
    }
}
```

이 방식은 동시에 프록시 설정이 섞이는 위험은 줄일 수 있지만, 모든 외부 호출이 직렬화되어 성능이 떨어진다. 운영 트래픽이 있는 Spring Web 애플리케이션에서는 근본 해결책이 아니다.

## 11. 실무 권장 최종 구조

```text
Controller
  ↓
Service
  ↓
ExternalApiClient
  ↓
URL.openConnection(Proxy)
```

Controller에서는 프록시를 알 필요가 없고, 외부 통신 전용 Client 클래스에서만 프록시를 처리한다.

```java
@RestController
@RequiredArgsConstructor
public class ApiTestController {
    private final ExternalApiClient externalApiClient;
    @GetMapping("/api/test-proxy")
    public String testProxy() throws IOException {
        return externalApiClient.get("https://api.example.com/health");
    }
}
```

## 12. `HttpURLConnection` 유지 vs 대체 검토

|방식|장점|단점|판단|
|---|---|---|---|
|`HttpURLConnection + Proxy`|기존 코드 수정량 적음|기능 제한, 인증/풀링 관리 불편|단기 수정|
|Apache HttpClient|프록시, 인증, 타임아웃, 커넥션풀 제어 우수|의존성 추가|실무 권장|
|OkHttp|API 단순, 프록시 설정 명확|의존성 추가|가능|
|Java 11 `HttpClient`|JDK 기본, 비동기 지원|Java 8 환경 불가|JDK 11 이상 권장|
|현재 코드가 이미 `HttpURLConnection` 기반이라면 1차 수정은 `openConnection(proxy)`로 충분하다. 다만 프록시 인증, 커넥션풀, 재시도, 상세 타임아웃, 운영 로그가 중요하면 Apache HttpClient나 Java 11 `HttpClient`로 교체하는 것이 더 낫다.||||

## 13. 수정 전후 비교

|구분|기존|수정|
|---|---|---|
|프록시 설정|`System.setProperty()`|`new Proxy(...)`|
|적용 범위|JVM 전체|해당 요청 연결|
|동시성|위험|안전|
|원복 필요|필요|불필요|
|외부 API별 프록시|어려움|쉬움|
|Spring Web 적합성|낮음|높음|

## 14. 최종 적용 순서

1. `System.setProperty("http.proxyHost", ...)` 코드 제거
    
2. `Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress(host, port))` 생성
    
3. `url.openConnection(proxy)`로 변경
    
4. 내부망 또는 예외 대상은 `Proxy.NO_PROXY` 분기
    
5. 프록시 인증 필요 시 인증 방식 별도 검토
    
6. `connectTimeout`, `readTimeout` 필수 설정
    
7. `finally`에서 `disconnect()` 또는 stream close 처리
    
8. 프록시 계정/비밀번호는 소스에 하드코딩하지 않고 설정/Secret으로 관리
    
9. WAS 동시 요청 테스트로 다른 외부 통신에 영향이 없는지 확인
    
10. 장기적으로 Apache HttpClient, Java 11 `HttpClient`, Spring `RestTemplate/WebClient` 전환 검토
    

## 15. 핵심 코드만 정리

```java
Proxy proxy = new Proxy(
        Proxy.Type.HTTP,
        new InetSocketAddress("10.10.10.10", 8080)
);
URL url = new URL("https://api.example.com/test");
HttpURLConnection conn = (HttpURLConnection) url.openConnection(proxy);
conn.setRequestMethod("GET");
conn.setConnectTimeout(3000);
conn.setReadTimeout(5000);
try {
    int status = conn.getResponseCode();
    String body = status >= 200 && status < 300
            ? readBody(conn.getInputStream())
            : readBody(conn.getErrorStream());
} finally {
    conn.disconnect();
}
```

> 결론적으로 Spring Web 환경에서는 `System.setProperty()`로 프록시를 임시 설정하는 방식을 제거하고, `URL.openConnection(Proxy)`를 사용해 **요청별 프록시 적용 방식**으로 수정하는 것이 맞다.