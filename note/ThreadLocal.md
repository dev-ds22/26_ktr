
# Spring Framework 5.3에서 `ThreadLocal` + `ProceedingJoinPoint.proceed()` 기반 Proxy Advisor 구현

## 1. 결론

가능하다. Spring Boot 없이 **Spring Framework 5.3 + XML 설정 + `@Aspect` + `@Around` + `ProceedingJoinPoint.proceed()`** 방식으로 구현하면 된다.  
단, 핵심 원칙은 아래와 같다.

|구분|적용|
|---|---|
|프록시 설정|`System.setProperty()` 사용 금지|
|적용 범위|JVM 전체가 아니라 현재 요청 Thread 범위|
|구현 방식|`ThreadLocal`에 Proxy 정보 저장|
|실제 통신|`URL.openConnection(proxy)` 사용|
|해제 처리|`finally`에서 `ThreadLocal.remove()` 필수|
|Spring AOP는 프록시 기반으로 동작하며 JDK Dynamic Proxy 또는 CGLIB를 사용한다. 또한 `@Around` Advice에서는 대상 메서드를 실행하기 위해 `ProceedingJoinPoint.proceed()`를 호출해야 한다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))||
|Java의 `URL.openConnection(Proxy)`는 지정한 Proxy를 통해 연결하며, 시스템 기본 Proxy 설정보다 우선 적용된다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/URL.html?utm_source=chatgpt.com "URL (Java Platform SE 8 )"))||

## 2. 전체 구조

```text
Controller
  ↓
Service Method
  ↓
@UseProxy("defaultProxy")
  ↓
ProxyAspect Around Advice
  ↓
ProxyContextHolder.set(proxySpec)
  ↓
proceedingJoinPoint.proceed()
  ↓
ExternalHttpClient
  ↓
url.openConnection(proxy)
  ↓
finally ProxyContextHolder.clear()
```

## 3. Maven 의존성

Spring Framework 5.3 기반, Spring Boot 미사용 기준이다.

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.3.39</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>5.3.39</version>
    </dependency>
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.22.1</version>
    </dependency>
</dependencies>
```

> 이미 프로젝트에서 Spring 5.3.x 버전을 쓰고 있다면 전체 Spring 모듈 버전은 기존 프로젝트 버전과 맞추는 것이 안전하다.

## 4. 설정 파일: `proxy.properties`

```properties
external.proxy.default.host=10.10.10.10
external.proxy.default.port=8080
external.proxy.default.enabled=true
```

운영에서는 프록시 계정, 비밀번호, 내부망 예외 도메인을 별도 설정으로 분리하는 것이 좋다.

## 5. Annotation: `@UseProxy`

```java
package com.example.proxy;
import java.lang.annotation.*;
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface UseProxy {
    String value() default "default";
}
```

사용 예:

```java
@UseProxy("default")
public String callExternalApi() {
    return externalHttpClient.get("https://api.example.com/test");
}
```

## 6. Proxy 설정 객체

```java
package com.example.proxy;
import java.net.InetSocketAddress;
import java.net.Proxy;
public class ProxySpec {
    private final String host;
    private final int port;
    private final boolean enabled;
    public ProxySpec(String host, int port, boolean enabled) {
        this.host = host;
        this.port = port;
        this.enabled = enabled;
    }
    public boolean isEnabled() {
        return enabled;
    }
    public Proxy toJavaProxy() {
        if (!enabled) {
            return Proxy.NO_PROXY;
        }
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

## 7. Proxy 설정 Registry

```java
package com.example.proxy;
import java.util.Collections;
import java.util.Map;
public class ProxyConfigRegistry {
    private final Map<String, ProxySpec> proxySpecMap;
    public ProxyConfigRegistry(Map<String, ProxySpec> proxySpecMap) {
        this.proxySpecMap = proxySpecMap == null
            ? Collections.emptyMap()
            : proxySpecMap;
    }
    public ProxySpec getProxySpec(String name) {
        ProxySpec proxySpec = proxySpecMap.get(name);
        if (proxySpec == null) {
            throw new IllegalArgumentException("Proxy 설정을 찾을 수 없습니다. name=" + name);
        }
        return proxySpec;
    }
}
```

## 8. ThreadLocal Context

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

`ThreadLocal`은 WAS Thread Pool에서 Thread가 재사용되므로 `remove()`가 필수다. 제거하지 않으면 다음 요청이 이전 요청의 Proxy 설정을 물려받을 수 있다.

## 9. `ProceedingJoinPoint.proceed()` 기반 Aspect

```java
package com.example.proxy;
import java.lang.reflect.Method;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.aop.support.AopUtils;
import org.springframework.core.annotation.AnnotatedElementUtils;
public class ProxyAspect {
    private final ProxyConfigRegistry proxyConfigRegistry;
    public ProxyAspect(ProxyConfigRegistry proxyConfigRegistry) {
        this.proxyConfigRegistry = proxyConfigRegistry;
    }
    @Around("@annotation(com.example.proxy.UseProxy) || @within(com.example.proxy.UseProxy)")
    public Object applyProxy(ProceedingJoinPoint joinPoint) throws Throwable {
        UseProxy useProxy = findUseProxy(joinPoint);
        if (useProxy == null) {
            return joinPoint.proceed();
        }
        ProxySpec proxySpec = proxyConfigRegistry.getProxySpec(useProxy.value());
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
        Class<?> targetClass = joinPoint.getTarget() != null
            ? joinPoint.getTarget().getClass()
            : method.getDeclaringClass();
        Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);
        UseProxy methodAnnotation =
            AnnotatedElementUtils.findMergedAnnotation(specificMethod, UseProxy.class);
        if (methodAnnotation != null) {
            return methodAnnotation;
        }
        UseProxy classAnnotation =
            AnnotatedElementUtils.findMergedAnnotation(targetClass, UseProxy.class);
        if (classAnnotation != null) {
            return classAnnotation;
        }
        return AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), UseProxy.class);
    }
}
```

## 10. 실제 HTTP 호출 Client

```java
package com.example.http;
import com.example.proxy.ProxyContextHolder;
import com.example.proxy.ProxySpec;
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.Proxy;
import java.net.URL;
import java.nio.charset.StandardCharsets;
public class ExternalHttpClient {
    private int connectTimeoutMillis = 3000;
    private int readTimeoutMillis = 5000;
    public String get(String targetUrl) throws IOException {
        URL url = new URL(targetUrl);
        HttpURLConnection conn = openConnection(url);
        conn.setRequestMethod("GET");
        conn.setConnectTimeout(connectTimeoutMillis);
        conn.setReadTimeout(readTimeoutMillis);
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
        Proxy proxy = proxySpec == null ? Proxy.NO_PROXY : proxySpec.toJavaProxy();
        return (HttpURLConnection) url.openConnection(proxy);
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
    public void setConnectTimeoutMillis(int connectTimeoutMillis) {
        this.connectTimeoutMillis = connectTimeoutMillis;
    }
    public void setReadTimeoutMillis(int readTimeoutMillis) {
        this.readTimeoutMillis = readTimeoutMillis;
    }
}
```

## 11. Service 적용 예시

```java
package com.example.service;
import com.example.http.ExternalHttpClient;
import com.example.proxy.UseProxy;
public class ExternalLinkService {
    private ExternalHttpClient externalHttpClient;
    @UseProxy("default")
    public String callByProxy() throws Exception {
        return externalHttpClient.get("https://api.example.com/test");
    }
    public String callDirect() throws Exception {
        return externalHttpClient.get("https://internal.example.com/health");
    }
    public void setExternalHttpClient(ExternalHttpClient externalHttpClient) {
        this.externalHttpClient = externalHttpClient;
    }
}
```

## 12. XML 설정: `applicationContext.xml`

Spring Boot 없이 XML 기반으로 사용하는 구성이다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
         http://www.springframework.org/schema/beans
         https://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/aop
         https://www.springframework.org/schema/aop/spring-aop.xsd
         http://www.springframework.org/schema/context
         https://www.springframework.org/schema/context/spring-context.xsd
         http://www.springframework.org/schema/util
         https://www.springframework.org/schema/util/spring-util.xsd">
    <context:property-placeholder location="classpath:proxy.properties"/>
    <aop:aspectj-autoproxy proxy-target-class="true"/>
    <bean id="defaultProxySpec" class="com.example.proxy.ProxySpec">
        <constructor-arg value="${external.proxy.default.host}"/>
        <constructor-arg value="${external.proxy.default.port}"/>
        <constructor-arg value="${external.proxy.default.enabled}"/>
    </bean>
    <util:map id="proxySpecMap" key-type="java.lang.String" value-type="com.example.proxy.ProxySpec">
        <entry key="default" value-ref="defaultProxySpec"/>
    </util:map>
    <bean id="proxyConfigRegistry" class="com.example.proxy.ProxyConfigRegistry">
        <constructor-arg ref="proxySpecMap"/>
    </bean>
    <bean id="proxyAspect" class="com.example.proxy.ProxyAspect">
        <constructor-arg ref="proxyConfigRegistry"/>
    </bean>
    <bean id="externalHttpClient" class="com.example.http.ExternalHttpClient">
        <property name="connectTimeoutMillis" value="3000"/>
        <property name="readTimeoutMillis" value="5000"/>
    </bean>
    <bean id="externalLinkService" class="com.example.service.ExternalLinkService">
        <property name="externalHttpClient" ref="externalHttpClient"/>
    </bean>
</beans>
```

## 13. Controller에서 호출 예시

```java
package com.example.web;
import com.example.service.ExternalLinkService;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ResponseBody;
@Controller
public class ProxyTestController {
    private ExternalLinkService externalLinkService;
    @GetMapping("/test/proxy-call")
    @ResponseBody
    public String proxyCall() throws Exception {
        return externalLinkService.callByProxy();
    }
    public void setExternalLinkService(ExternalLinkService externalLinkService) {
        this.externalLinkService = externalLinkService;
    }
}
```

XML에 Controller까지 수동 등록하려면:

```xml
<bean id="proxyTestController" class="com.example.web.ProxyTestController">
    <property name="externalLinkService" ref="externalLinkService"/>
</bean>
```

컴포넌트 스캔을 쓴다면:

```xml
<context:component-scan base-package="com.example"/>
```

## 14. 내부망 URL은 Proxy 제외하는 개선 코드

실무에서는 내부망 API는 프록시를 타면 장애가 나는 경우가 있다. 이 경우 `ExternalHttpClient`에 예외 처리를 넣는다.

```java
private HttpURLConnection openConnection(URL url) throws IOException {
    if (isNoProxyTarget(url.getHost())) {
        return (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);
    }
    ProxySpec proxySpec = ProxyContextHolder.get();
    Proxy proxy = proxySpec == null ? Proxy.NO_PROXY : proxySpec.toJavaProxy();
    return (HttpURLConnection) url.openConnection(proxy);
}
private boolean isNoProxyTarget(String host) {
    return "localhost".equalsIgnoreCase(host)
        || host.startsWith("127.")
        || host.startsWith("10.")
        || host.startsWith("192.168.")
        || host.endsWith(".internal.local");
}
```

## 15. 프록시 인증이 필요한 경우

`HttpURLConnection`에서 요청별 Proxy 인증을 단순 처리하려면 `Proxy-Authorization` 헤더를 사용한다.

```java
String auth = proxyUser + ":" + proxyPassword;
String encoded = java.util.Base64.getEncoder()
    .encodeToString(auth.getBytes(StandardCharsets.UTF_8));
conn.setRequestProperty("Proxy-Authorization", "Basic " + encoded);
```

다만 프록시 인증, 커넥션 풀, 재시도, TLS 상세 제어가 필요하면 `HttpURLConnection`보다 Apache HttpClient 사용이 더 적합하다.

## 16. 실무 주의점

|항목|주의|
|---|---|
|`System.setProperty()`|사용하지 않는다. JVM 전체 프록시가 바뀐다.|
|`ThreadLocal.clear()`|반드시 `finally`에서 호출한다.|
|`proceed()`|호출하지 않으면 대상 메서드가 실행되지 않는다.|
|Self Invocation|같은 클래스 내부에서 `this.callByProxy()`로 호출하면 Spring AOP가 적용되지 않는다. Spring AOP는 프록시 기반이므로 자기 자신 호출은 프록시를 우회한다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))|
|`final` 메서드|CGLIB에서도 advice 적용이 제한된다.|
|`private` 메서드|Spring AOP 적용 대상이 아니다.|
|비동기|`@Async`, 별도 Executor, 새 Thread에서는 일반 `ThreadLocal`이 전달되지 않는다.|
|외부 호출 분산|`new URL(...).openConnection()`이 여러 클래스에 흩어져 있으면 통제가 어렵다. 공통 Client로 모아야 한다.|
|JBoss/WAS|Thread Pool 재사용 때문에 `ThreadLocal.remove()` 누락은 운영 장애 원인이 될 수 있다.|

## 17. 검증 방법

### 17.1 프록시 적용 확인 로그 추가

`ExternalHttpClient.openConnection()`에 임시 로그를 넣는다.

```java
ProxySpec proxySpec = ProxyContextHolder.get();
if (proxySpec != null && proxySpec.isEnabled()) {
    System.out.println("Proxy enabled. host=" + proxySpec.getHost() + ", port=" + proxySpec.getPort());
} else {
    System.out.println("Proxy disabled.");
}
```

### 17.2 테스트 순서

```text
1. @UseProxy("default") 붙은 메서드 호출
2. Proxy enabled 로그 확인
3. 프록시 서버 access log 확인
4. @UseProxy 없는 메서드 호출
5. Proxy disabled 로그 확인
6. 동시 요청 테스트
7. ThreadLocal 값이 다음 요청으로 누수되지 않는지 확인
```

## 18. 최종 권장 소스 핵심

```java
@Around("@annotation(com.example.proxy.UseProxy) || @within(com.example.proxy.UseProxy)")
public Object applyProxy(ProceedingJoinPoint joinPoint) throws Throwable {
    UseProxy useProxy = findUseProxy(joinPoint);
    if (useProxy == null) {
        return joinPoint.proceed();
    }
    ProxySpec proxySpec = proxyConfigRegistry.getProxySpec(useProxy.value());
    try {
        ProxyContextHolder.set(proxySpec);
        return joinPoint.proceed();
    } finally {
        ProxyContextHolder.clear();
    }
}
```

```java
ProxySpec proxySpec = ProxyContextHolder.get();
Proxy proxy = proxySpec == null ? Proxy.NO_PROXY : proxySpec.toJavaProxy();
HttpURLConnection conn = (HttpURLConnection) url.openConnection(proxy);
```

## 19. 최종 판단

Spring Framework 5.3, Spring Boot 미사용 환경에서도 이 구조는 실무 적용 가능하다.  
다만 반드시 지켜야 할 조건은 다음 3가지다.

1. `System.setProperty()`로 프록시를 설정하지 않는다.
    
2. `ThreadLocal`은 `finally`에서 반드시 제거한다.
    
3. 실제 통신 코드는 반드시 공통 Client로 모아 `url.openConnection(proxy)`를 사용한다.