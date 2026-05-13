
Java에서 `System.setProperty("http.proxyHost", "proxy.company.local");`와 같은 방식으로 프록시를 설정하는 것은 **JVM 전역(Global)**에 영향을 미치기 때문에 여러 가지 잠재적인 문제를 유발할 수 있습니다. 

주요 문제점은 다음과 같습니다.

1. 전역적 영향 (JVM-wide Scope)

- **모든 네트워킹에 적용:** 이 설정은 해당 JVM 내에서 실행되는 **모든** `HttpURLConnection` 및 이들을 사용하는 라이브러리(Apache HttpClient, OkHttp 등)에 영향을 미칩니다.
- **원치 않는 트래픽 중계:** 프록시를 거치지 않아도 되는 로컬 서비스(`localhost`, 내부 인트라넷)에 대한 요청까지 모두 프록시 서버로 보내지게 되어, 내부 네트워크 통신이 실패하거나 속도가 느려질 수 있습니다.
- **`nonProxyHosts` 설정 필수:** 이를 방지하기 위해 `System.setProperty("http.nonProxyHosts", "localhost|*.internal.com");`을 통해 우회할 호스트를 명시해야 합니다.

2. 멀티스레드 환경에서의 동시성 문제 (Concurrency)

- **스레드 간 간섭:** 한 스레드에서 프록시를 설정하고 나중에 해제(`null`)하더라도, 다른 스레드에서 이 설정이 적용되는 시점에 따라 프록시 설정이 유동적으로 바뀔 수 있습니다.
- **상태 불일치:** 멀티스레드 애플리케이션(예: Tomcat 등 WAS)에서 동시에 서로 다른 프록시 설정이 필요한 경우, 전역 설정을 켜고 끄는 방식은 스레드 안전(Thread-safe)하지 않아 예상치 못한 오류를 발생시킵니다. 

3. 보안 및 인증 문제

- **인증 정보 노출:** 프록시 서버가 인증을 요구할 경우 `http.proxyUser`와 `http.proxyPassword`를 시스템 속성으로 설정해야 하는데, 이는 코드나 JVM 인자 내에 평문으로 노출될 위험이 있습니다.
- **407 Proxy Authentication Required:** HTTPS 연결 시 프록시 인증(Basic 인증 등)이 필요한 경우, Java 버전에 따라 터널링 보안 설정 때문에 인증에 실패할 수 있습니다.

4. 세션 관리 및 라이브러리 충돌

- **영구적인 설정:** 한번 설정하면 JVM이 종료될 때까지 설정이 유지됩니다.
- **라이브러리별 다른 동작:** 일부 고성능 HTTP 라이브러리는 시스템 속성을 무시하거나 자체 프록시 설정 방식을 선호하여, `System.setProperty`가 작동하지 않는 것처럼 보일 수 있습니다. 

5. 런타임 변경의 어려움

- `System.setProperty`를 통해 변경된 프록시는 즉시 모든 연결에 적용되지 않거나, 이미 연결된 커넥션(Connection Pool)에는 적용되지 않을 수 있습니다.

---

추천하는 대안

전역 설정 대신, **필요한 연결(Connection)에만 지역적으로 프록시를 설정**하는 것이 좋습니다.

java

```
// Proxy 객체를 직접 생성하여 적용
Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxy.company.local", 8080));
URL url = new URL("http://google.com");
HttpURLConnection conn = (HttpURLConnection) url.openConnection(proxy);
// ...
```

코드를 사용할 때는 주의가 필요합니다.

이 방식은 스레드 안전하며, 특정 요청에만 프록시를 적용할 수 있어 안전


# 결론

`Advisor`로 **특정 메서드/요청 구간에만 프록시 설정 로직을 공통 적용**하는 것 자체는 장점이 있습니다. Spring의 `Advisor`는 **Pointcut + Advice** 조합으로 특정 조인포인트에 공통 처리를 묶는 용도라, 프록시 적용 대상을 서비스/클라이언트 호출 메서드로 한정하기 좋습니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop-api/advisor.html?utm_source=chatgpt.com "The Advisor API in Spring"))  
다만 `System.setProperty("http.proxyHost", ...)`는 **Spring 전용 설정이 아니라 JVM 전역 네트워킹 시스템 프로퍼티**이므로, 웹 애플리케이션에서 요청별로 on/off 하는 방식은 매우 조심해야 합니다. Java 문서는 네트워킹 시스템 프로퍼티를 `System.setProperty()`로 바꿀 수 있다고 설명하지만, 이는 프로세스 전역 동작을 바꾸는 성격입니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/17/core/java-networking.html?utm_source=chatgpt.com "10 Java Networking"))  
즉, **AOP로 적용 대상을 “논리적으로” 좁히는 장점은 있지만**, `System.setProperty` 자체는 **물리적으로 JVM 전체에 영향**을 줄 수 있어 동시 요청 환경에서는 권장도가 낮습니다. 이 방식은 단일 스레드 배치나 매우 통제된 환경에서는 가능하지만, 일반적인 Spring 웹 서비스에서는 **HTTP 클라이언트별 프록시 설정**이 더 안전합니다. `java.net.Proxy`와 `ProxySelector`는 프록시 선택을 위한 표준 API입니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/net/Proxy.html?utm_source=chatgpt.com "Proxy (Java SE 26 & JDK 26)"))

# Advisor로 특정 요청에만 프록시 처리할 때의 장점

## 장점

|항목|의미|
|---|---|
|관심사 분리|비즈니스 코드 안에 `System.setProperty(...)`를 흩뿌리지 않고 AOP로 분리 가능|
|적용 범위 한정|특정 외부 연동 메서드에만 Pointcut 적용 가능|
|중복 제거|여러 API 호출 메서드에 같은 프록시 처리 로직을 재사용 가능|
|정책 집중화|어떤 외부 연동이 프록시를 써야 하는지 한 곳에서 관리 가능|
|후처리 일원화|Around Advice에서 호출 전/후 로깅, 복원, 예외 처리까지 묶기 쉬움|
|Spring 문서 기준으로 `Advisor`는 단일 Advice와 Pointcut을 결합해 특정 join point에 적용하는 구조라, 이런 “특정 외부 연동에만 공통 네트워크 정책 적용” 같은 요구에 잘 맞습니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop-api/advisor.html?utm_source=chatgpt.com "The Advisor API in Spring"))||

## 하지만 치명적인 한계

|항목|문제|
|---|---|
|적용 범위|`System.setProperty`는 현재 요청만이 아니라 JVM 전체에 영향 가능|
|동시성|A 요청이 프록시를 켠 동안 B 요청도 같은 프로퍼티를 볼 수 있음|
|복원 위험|호출 후 원복해도 중간에 다른 스레드가 끼어들 수 있음|
|프로토콜 분리|`http.proxyHost`는 HTTP용이며 HTTPS는 `https.proxyHost`를 별도로 봐야 함|
|Java 문서는 `http.proxyHost`가 HTTP 프록시 설정이고, HTTPS는 `https.proxyHost`/`https.proxyPort`를 별도로 사용한다고 설명합니다. 또한 이런 네트워킹 프로퍼티는 런타임에 `System.setProperty()`로 바꿀 수 있습니다. 바로 이 점 때문에 “가능은 하지만 전역 영향” 문제가 생깁니다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/doc-files/net-properties.html?utm_source=chatgpt.com "Networking Properties"))||

# 언제 프록시를 쓰는 것이 좋은가

Java 공식 문서는 프록시 서버를 주로 **보안(방화벽 통과)** 및 **성능(캐시)** 목적으로 사용한다고 설명합니다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/doc-files/net-properties.html?utm_source=chatgpt.com "Networking Properties"))  
이를 Spring 실무로 풀면 아래 상황입니다.

|상황|프록시 사용 적합성|설명|
|---|---|---|
|사내망에서 외부 API 호출|매우 높음|조직 보안 정책상 직접 인터넷 outbound가 금지된 경우|
|특정 파트너 API만 우회 경로 사용|높음|외부 연동 전용 네트워크 경로를 강제해야 할 때|
|프록시 캐시 이점이 있는 정적/반복 요청|조건부|주로 일반 웹 리소스나 반복 조회에서 의미 있음|
|모든 요청이 동일한 egress 정책을 따라야 함|높음|애플리케이션 전체 공통 프록시 설정이 더 단순|
|사용자 요청마다 다르게 프록시 on/off|낮음|JVM 전역 프로퍼티 방식은 부적합|

# 가장 적절한 예제 상황

## 예제 1: 사용하기 좋은 경우

**사내 ERP 연동 서버**가 외부 결제사 HTTP API를 호출해야 하는데, 외부 인터넷은 반드시 사내 프록시를 경유해야 하는 경우입니다.

- 내부 DB 조회, 내부 MSA 호출은 프록시 불필요
    
- 외부 결제사 호출만 프록시 필요
    
- 이 경우 “결제사 연동 클라이언트”에만 프록시 정책을 적용하는 구조는 합리적입니다  
    이런 요구 자체는 AOP 포인트컷과 잘 맞습니다. 다만 구현은 `System.setProperty`보다 **해당 HTTP 클라이언트 인스턴스에 프록시를 직접 설정**하는 편이 안전합니다. `Proxy`는 특정 연결에 사용할 프록시 설정을 표현하는 표준 클래스입니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/net/Proxy.html?utm_source=chatgpt.com "Proxy (Java SE 26 & JDK 26)"))
    

## 예제 2: 사용하기 나쁜 경우

**일반 사용자 웹 요청마다** 어떤 사용자는 프록시를 쓰고, 어떤 사용자는 직접 나가야 하는 경우입니다.  
이걸 요청마다 `System.setProperty("http.proxyHost", ...)`로 켜고 끄면, 같은 JVM 안 다른 요청도 동시에 영향을 받을 수 있습니다. Java 시스템 프로퍼티가 전역이기 때문입니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/17/core/java-networking.html?utm_source=chatgpt.com "10 Java Networking"))

# AOP + System.setProperty 방식의 “장점은 있으나 권장도는 낮은” 예제

## 개념 예제

```java
@Aspect
@Component
@Order(1)
public class ExternalProxyAspect {
    @Around("execution(* com.example.integration.partner..*(..))")
    public Object withProxy(ProceedingJoinPoint pjp) throws Throwable {
        String oldHost = System.getProperty("http.proxyHost");
        String oldPort = System.getProperty("http.proxyPort");
        try {
            System.setProperty("http.proxyHost", "proxy.company.local");
            System.setProperty("http.proxyPort", "8080");
            return pjp.proceed();
        } finally {
            restore("http.proxyHost", oldHost);
            restore("http.proxyPort", oldPort);
        }
    }
    private void restore(String key, String value) {
        if (value == null) {
            System.clearProperty(key);
        } else {
            System.setProperty(key, value);
        }
    }
}
```

## 이 방식의 장점

|장점|설명|
|---|---|
|비즈니스 코드 단순화|연동 서비스 메서드는 프록시 로직 몰라도 됨|
|정책 중앙 관리|파트너 연동 패키지 전체에 일괄 적용 가능|
|예외/복원 공통화|finally에서 복원 로직 통합 가능|

## 이 방식의 문제

|문제|설명|
|---|---|
|동시 요청 간 간섭|다른 스레드가 같은 프로퍼티를 읽을 수 있음|
|예측 어려움|복원 타이밍 전에 다른 연동이 시작될 수 있음|
|운영 리스크|장애 시 어떤 요청이 어떤 프록시 상태였는지 추적 어려움|
|즉, **배치/단일 스레드 작업이면 제한적으로 가능**하지만, **다중 사용자 웹 요청 처리에서는 비추천**입니다.||

# 더 좋은 방식: HTTP 클라이언트별 프록시 설정

## 권장 이유

`System.setProperty` 대신 **프록시가 필요한 외부 호출용 클라이언트만 별도 빈으로 분리**하면 요청 간 간섭이 없습니다. Java 표준의 `Proxy`는 연결 단위 프록시 구성을 위한 타입입니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/net/Proxy.html?utm_source=chatgpt.com "Proxy (Java SE 26 & JDK 26)"))

## 예제: JDK `Proxy` 사용 개념

```java
Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxy.company.local", 8080));
```

이런 식의 프록시 객체를 해당 클라이언트에만 연결하면, JVM 전체 프로퍼티를 흔들지 않아도 됩니다. `ProxySelector`도 표준 API이지만, 기본 선택기 변경 역시 전역 정책 성격이 있으므로 요청별 분리 목적이면 개별 클라이언트 설정이 더 안전합니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/26/docs/api/java.base/java/net/Proxy.html?utm_source=chatgpt.com "Proxy (Java SE 26 & JDK 26)"))

# Spring 실무 추천 구조

|상황|추천 방식|
|---|---|
|애플리케이션 전체가 항상 같은 프록시 사용|JVM 옵션 `-Dhttp.proxyHost=...` 또는 공통 네트워크 설정|
|외부 연동 일부만 프록시 사용|프록시 전용 `RestTemplate`/HttpClient 빈 분리|
|배치 작업 한 종류만 프록시 사용|배치 JVM 자체를 프록시 설정으로 실행|
|웹 요청마다 프록시 on/off 필요|`System.setProperty` 비추천, 클라이언트 단위 분리 필수|

# 한 줄 판단

**Advisor를 이용해 특정 외부 연동에만 프록시 정책을 묶는 발상 자체는 좋지만, 구현을 `System.setProperty("http.proxyHost", ...)`로 하면 JVM 전역 영향 때문에 웹 애플리케이션에서는 위험합니다.**  
가장 좋은 패턴은 **“AOP로 대상 선정” + “실제 프록시 적용은 해당 HTTP 클라이언트 인스턴스 수준에서 분리”**입니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop-api/advisor.html?utm_source=chatgpt.com "The Advisor API in Spring"))  


정확도: 97%

# Spring Framework 실무용 Proxy 개선안 기반 `Advisor` 예제

## 1. 권장 방향

`System.setProperty("http.proxyHost", ...)` 방식은 JVM 전역 설정이라 웹 애플리케이션의 동시 요청 환경에서는 다른 요청에도 영향을 줄 수 있습니다. Spring Integration 문서도 HTTP 프록시 적용 방식으로 **시스템 프로퍼티** 외에 **HTTP client request factory 빈을 명시적으로 구성하는 방법**을 제시합니다. 또한 `RestTemplate`은 기본적으로 `SimpleClientHttpRequestFactory`를 사용하며 다른 request factory로 교체할 수 있고, `SimpleClientHttpRequestFactory`에는 `setProxy(Proxy)` 메서드가 있습니다. 따라서 실무에서는 **전역 System Property 변경 대신, 프록시가 걸린 HTTP 클라이언트 빈을 별도로 만들고, Advisor는 “언제 그 클라이언트를 쓸지”만 결정**하게 두는 편이 안전합니다. ([Home](https://docs.spring.io/spring-integration/reference/http/proxy.html "HTTP Proxy configuration :: Spring Integration"))

## 2. 이 예제가 해결하는 문제

|항목|기존 방식(`System.setProperty`)|개선 방식(전용 Proxy Client + Advisor)|
|---|---|---|
|영향 범위|JVM 전역|프록시 전용 HTTP 클라이언트에 한정|
|동시성 안전성|낮음|높음|
|적용 대상 제어|어려움|Advisor/Annotation으로 명확|
|테스트 용이성|낮음|높음|
|운영 추적성|낮음|높음|
|이 구조는 Spring AOP의 `Advisor = Pointcut + Advice` 개념에 잘 맞습니다. 다만 Spring AOP는 **proxy-based AOP**이므로 self-invocation은 가로채지 못하며, 일반적으로 프록시를 통해 들어오는 public 메서드 기준으로 설계해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.0.11.RELEASE/spring-framework-reference/core.html "Core Technologies"))|||

## 3. 적용 대상 예시

아래 같은 경우에 적합합니다.

|사용 사례|프록시 사용 여부|
|---|---|
|사내망에서 외부 결제사/택배사/관세 API 호출|사용 권장|
|내부 MSA 간 호출|보통 직접 연결 권장|
|배치에서 외부 파트너 데이터 수집|사용 권장|
|사용자별로 매 요청마다 다른 프록시 사용|별도 라우팅 설계 필요, 단순 Advisor만으로는 부족|
|대표적인 실무 예는 **내부 주문 서비스는 직접 호출**하고, **외부 파트너 연동만 사내 egress proxy를 통해 호출**하는 구조입니다. Java의 `Proxy`는 이런 연결 단위 프록시 설정을 표현하는 표준 타입입니다. ([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/net/Proxy.html "Proxy (Java Platform SE 8 )"))||

## 4. 전체 구조

```text
Service method
  └─ Advisor(MethodInterceptor)
      └─ ProxyRouteContextHolder 에 현재 라우트 저장
          └─ ProxyAwareRestOperations 가 direct/proxy RestTemplate 선택
              └─ 외부 API 호출
```

핵심은 `Advisor`가 **proxy on/off를 JVM 전역에 반영하는 것이 아니라**, **현재 호출 흐름에서 어떤 RestTemplate을 사용할지 결정하는 context만 세팅**한다는 점입니다. `RestTemplate`의 request factory를 교체할 수 있다는 점과 `SimpleClientHttpRequestFactory#setProxy(Proxy)`를 활용하는 구조입니다. ([Home](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html?utm_source=chatgpt.com "REST Clients :: Spring Framework"))

## 5. 코드 예제

### 5.1 프록시 라우트 enum

```java
package com.example.proxy;

public enum ProxyRoute {
    DIRECT,
    PARTNER_EGRESS
}
```

### 5.2 Annotation

```java
package com.example.proxy;

import java.lang.annotation.*;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface UseOutboundProxy {
    ProxyRoute value() default ProxyRoute.PARTNER_EGRESS;
}
```

### 5.3 ContextHolder

중첩 호출까지 고려해서 Stack 방식으로 두는 편이 안전합니다.

```java
package com.example.proxy;

import java.util.ArrayDeque;
import java.util.Deque;

public final class ProxyRouteContextHolder {
    private static final ThreadLocal<Deque<ProxyRoute>> HOLDER =
            ThreadLocal.withInitial(ArrayDeque::new);

    private ProxyRouteContextHolder() {
    }

    public static void push(ProxyRoute route) {
        HOLDER.get().push(route);
    }

    public static ProxyRoute current() {
        Deque<ProxyRoute> stack = HOLDER.get();
        return stack.isEmpty() ? ProxyRoute.DIRECT : stack.peek();
    }

    public static void pop() {
        Deque<ProxyRoute> stack = HOLDER.get();
        if (!stack.isEmpty()) {
            stack.pop();
        }
        if (stack.isEmpty()) {
            HOLDER.remove();
        }
    }
}
```

### 5.4 MethodInterceptor

```java
package com.example.proxy;

import java.lang.reflect.Method;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.support.AopUtils;
import org.springframework.core.annotation.AnnotatedElementUtils;

public class UseOutboundProxyMethodInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = invocation.getThis().getClass();
        Method method = AopUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);

        UseOutboundProxy annotation =
                AnnotatedElementUtils.findMergedAnnotation(method, UseOutboundProxy.class);

        if (annotation == null) {
            annotation = AnnotatedElementUtils.findMergedAnnotation(targetClass, UseOutboundProxy.class);
        }

        ProxyRoute route = (annotation != null) ? annotation.value() : ProxyRoute.DIRECT;

        ProxyRouteContextHolder.push(route);
        try {
            return invocation.proceed();
        } finally {
            ProxyRouteContextHolder.pop();
        }
    }
}
```

### 5.5 Advisor

`StaticMethodMatcherPointcutAdvisor`를 사용해 annotation이 붙은 메서드만 잡습니다.

```java
package com.example.proxy;

import java.lang.reflect.Method;
import org.aopalliance.aop.Advice;
import org.springframework.aop.support.StaticMethodMatcherPointcutAdvisor;
import org.springframework.core.annotation.AnnotatedElementUtils;

public class UseOutboundProxyAdvisor extends StaticMethodMatcherPointcutAdvisor {

    public UseOutboundProxyAdvisor(Advice advice) {
        super(advice);
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        return AnnotatedElementUtils.hasAnnotation(method, UseOutboundProxy.class)
                || AnnotatedElementUtils.hasAnnotation(targetClass, UseOutboundProxy.class);
    }
}
```

### 5.6 RestTemplate 구성

프록시가 필요한 `RestTemplate`만 별도 생성합니다.

```java
package com.example.config;

import com.example.proxy.UseOutboundProxyAdvisor;
import com.example.proxy.UseOutboundProxyMethodInterceptor;
import java.net.InetSocketAddress;
import java.net.Proxy;
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.SimpleClientHttpRequestFactory;
import org.springframework.web.client.RestOperations;
import org.springframework.web.client.RestTemplate;

@Configuration
public class OutboundProxyConfig {

    @Bean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        return new DefaultAdvisorAutoProxyCreator();
    }

    @Bean
    public UseOutboundProxyMethodInterceptor useOutboundProxyMethodInterceptor() {
        return new UseOutboundProxyMethodInterceptor();
    }

    @Bean
    public Advisor useOutboundProxyAdvisor(UseOutboundProxyMethodInterceptor interceptor) {
        return new UseOutboundProxyAdvisor(interceptor);
    }

    @Bean
    @Qualifier("directRestOperations")
    public RestOperations directRestOperations() {
        return new RestTemplate();
    }

    @Bean
    @Qualifier("partnerProxyRestOperations")
    public RestOperations partnerProxyRestOperations() {
        SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
        requestFactory.setProxy(
                new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxy.company.local", 8080))
        );
        requestFactory.setConnectTimeout(3000);
        requestFactory.setReadTimeout(5000);
        return new RestTemplate(requestFactory);
    }
}
```

`RestTemplate`은 request factory를 교체할 수 있고, `SimpleClientHttpRequestFactory`에 `setProxy(Proxy)`가 있으므로 위 구성이 가능합니다. Java의 `Proxy`는 HTTP/SOCKS 프록시와 소켓 주소를 표현하는 불변 객체입니다. ([Home](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html?utm_source=chatgpt.com "REST Clients :: Spring Framework"))

### 5.7 라우팅용 HTTP 호출 래퍼

실무에서는 서비스 코드가 프록시 여부를 직접 알지 않게 하는 것이 좋습니다.

```java
package com.example.proxy;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestOperations;

@Component
public class ProxyAwareRestOperations {

    private final RestOperations direct;
    private final RestOperations partnerProxy;

    public ProxyAwareRestOperations(
            @Qualifier("directRestOperations") RestOperations direct,
            @Qualifier("partnerProxyRestOperations") RestOperations partnerProxy) {
        this.direct = direct;
        this.partnerProxy = partnerProxy;
    }

    public <T> ResponseEntity<T> exchange(
            String url,
            HttpMethod method,
            HttpEntity<?> requestEntity,
            Class<T> responseType) {

        return current().exchange(url, method, requestEntity, responseType);
    }

    private RestOperations current() {
        ProxyRoute route = ProxyRouteContextHolder.current();
        return (route == ProxyRoute.PARTNER_EGRESS) ? partnerProxy : direct;
    }
}
```

### 5.8 외부 연동 Client

```java
package com.example.partner;

import com.example.proxy.ProxyAwareRestOperations;
import org.springframework.http.*;
import org.springframework.stereotype.Component;

@Component
public class PartnerOrderClient {

    private final ProxyAwareRestOperations restOperations;

    public PartnerOrderClient(ProxyAwareRestOperations restOperations) {
        this.restOperations = restOperations;
    }

    public String fetchOrder(String orderNo) {
        String url = "https://partner.example.com/api/orders/" + orderNo;

        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Client-Id", "commerce-admin");

        ResponseEntity<String> response = restOperations.exchange(
                url,
                HttpMethod.GET,
                new HttpEntity<>(headers),
                String.class
        );
        return response.getBody();
    }
}
```

### 5.9 서비스

annotation이 붙은 메서드만 프록시 라우트가 활성화됩니다.

```java
package com.example.service;

import com.example.partner.PartnerOrderClient;
import com.example.proxy.UseOutboundProxy;
import org.springframework.stereotype.Service;

@Service
public class PartnerSyncService {

    private final PartnerOrderClient partnerOrderClient;

    public PartnerSyncService(PartnerOrderClient partnerOrderClient) {
        this.partnerOrderClient = partnerOrderClient;
    }

    @UseOutboundProxy
    public String syncPartnerOrder(String orderNo) {
        return partnerOrderClient.fetchOrder(orderNo);
    }

    public String syncInternalOrder(String orderNo) {
        // 이 메서드는 annotation이 없으므로 direct RestTemplate 사용
        return "internal-" + orderNo;
    }
}
```

## 6. 이 예제의 장점

|장점|설명|
|---|---|
|전역 부작용 제거|`System.setProperty`를 쓰지 않으므로 다른 요청에 영향 없음|
|적용 지점 명확|`@UseOutboundProxy`만 보면 프록시 사용 여부가 드러남|
|서비스 코드 단순화|비즈니스 메서드 안에 프록시 설정 코드가 없음|
|테스트 용이|`ProxyAwareRestOperations` 또는 개별 `RestOperations` mocking 가능|
|확장 용이|`PARTNER_A`, `PARTNER_B`, `AUDIT_PROXY` 등으로 route 확장 가능|
|이 방식은 Spring 문서가 설명하는 AOP 프록시 모델의 “공통 관심사 분리”에 맞고, 프록시 설정은 전용 request factory bean으로 분리하므로 운영 안정성이 높습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.0.11.RELEASE/spring-framework-reference/core.html "Core Technologies"))||

## 7. 실무 주의사항

### 7.1 self-invocation 주의

같은 클래스 내부에서 `this.syncPartnerOrder()`처럼 호출하면 Advisor가 적용되지 않을 수 있습니다. Spring AOP는 proxy-based라서 **프록시를 거쳐 들어온 호출만** 가로챕니다. ([Home](https://docs.spring.io/spring-framework/docs/5.0.11.RELEASE/spring-framework-reference/core.html "Core Technologies"))

### 7.2 외부 연동 전용 bean으로 분리

`PartnerOrderClient`, `CarrierClient`, `PaymentGatewayClient`처럼 **외부 연동 bean을 명확히 분리**하는 것이 좋습니다. 그러면 프록시 정책 적용 지점도 분명해집니다. ([Home](https://docs.spring.io/spring-framework/docs/5.0.11.RELEASE/spring-framework-reference/core.html "Core Technologies"))

### 7.3 HTTP/HTTPS 프록시 구분

기존 시스템 프로퍼티 방식은 `http.proxyHost`와 `https.proxyHost`가 구분됩니다. request factory bean 방식으로 전환하더라도 실제 네트워크 정책은 HTTP/HTTPS 대상과 조직 프록시 정책을 함께 확인해야 합니다. ([Home](https://docs.spring.io/spring-integration/reference/http/proxy.html "HTTP Proxy configuration :: Spring Integration"))

## 8. 언제 쓰는 것이 좋은가

|상황|추천도|설명|
|---|---|---|
|외부 파트너 API만 프록시 경유 필요|매우 높음|이 예제와 가장 잘 맞음|
|배치에서 일부 잡만 프록시 필요|높음|잡 서비스 메서드에 annotation 적용|
|내부망/외부망 혼합 호출|높음|direct/proxy client 분리가 유용|
|모든 outbound가 동일 프록시 사용|보통|이 경우는 애플리케이션 전체 공통 client 설정이 더 단순|
|사용자별 프록시 정책이 다름|낮음|별도 라우팅/멀티테넌트 설계 필요|

## 9. 한 줄 정리

**실무에서는 `System.setProperty`를 Advisor로 감싸는 대신, `Advisor`는 “프록시 사용 대상 결정”만 하고 실제 프록시 적용은 `SimpleClientHttpRequestFactory#setProxy(...)`가 설정된 전용 `RestTemplate` 빈으로 분리하는 방식이 가장 안전합니다.**  


# Spring Framework 5.3 + Apache HttpClient 기반 실무용 `Advisor` 예제

## 핵심 요지

이전의 `System.setProperty("http.proxyHost", ...)` 방식 대신, **Apache HttpClient가 적용된 `RestTemplate`를 2개 분리**하고, `Advisor`는 **어떤 메서드가 proxy 경로를 써야 하는지만 결정**하게 만드는 것이 실무적으로 더 안전합니다. Spring 5.3의 `RestTemplate`는 `ClientHttpRequestFactory`를 바꿔 Apache HttpComponents를 사용할 수 있고, 그 경우 인증, 커넥션 풀링 같은 HTTP 라이브러리별 설정을 활용할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html?utm_source=chatgpt.com "Integration"))  
또한 Apache HttpClient 4.5의 `DefaultProxyRoutePlanner`는 **JVM system property를 보지 않고 항상 지정한 프록시를 사용**하므로, 요청별 프록시 정책을 분리하기에 적합합니다. 반대로 `SystemDefaultRoutePlanner`는 JVM의 프록시 설정을 따라갑니다. ([Apache HttpComponents](https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/connmgmt.html "Chapter 2. Connection management"))

## 언제 이 구조가 좋은가

|상황|권장 여부|이유|
|---|--:|---|
|외부 파트너 API만 사내 프록시를 타야 함|높음|내부 호출과 외부 호출을 분리 가능|
|배치 중 일부 연동만 프록시 필요|높음|잡/서비스 단위로 정책 부여 가능|
|모든 outbound가 동일 프록시 사용|보통|Advisor 없이 공통 `HttpClient` 1개면 충분|
|웹 요청마다 프록시 on/off를 전역 프로퍼티로 변경|낮음|JVM 전역 영향 위험|

## 설계 구조

```text
Service Method
  -> Advisor(MethodInterceptor)
     -> ThreadLocal 에 현재 Route 저장
        -> ProxyAwareRestOperations 가 direct / proxy RestTemplate 선택
           -> Apache HttpClient 기반 호출
```

## 이 구조의 핵심은 **AOP는 “대상 판정”**, **Apache HttpClient는 “실제 네트워크 설정”**을 담당하게 분리하는 것입니다. Spring AOP의 `Advisor`는 Pointcut + Advice 조합으로 특정 메서드에 공통 정책을 적용하는 용도에 적합합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

## 1) Maven 의존성 예시

Spring Framework 5.3 환경에서는 `RestTemplate`에 Apache HttpComponents 기반 `ClientHttpRequestFactory`를 붙여 사용하는 방식이 일반적입니다. 실무적으로는 Spring 5.3 + Apache HttpClient 4.5.x 조합이 가장 무난합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html?utm_source=chatgpt.com "Integration"))

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
        <version>5.3.39</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>5.3.39</version>
    </dependency>
    <dependency>
        <groupId>org.apache.httpcomponents</groupId>
        <artifactId>httpclient</artifactId>
        <version>4.5.14</version>
    </dependency>
</dependencies>
```

---

## 2) Proxy Route 구분 enum

```java
package com.example.outbound;

public enum ProxyRoute {
    DIRECT,
    PARTNER_PROXY
}
```

---

## 3) Annotation

프록시가 필요한 외부 연동 메서드에만 붙입니다.

```java
package com.example.outbound;

import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface UseOutboundProxy {
    ProxyRoute value() default ProxyRoute.PARTNER_PROXY;
}
```

---

## 4) ThreadLocal Holder

동일 요청 흐름 안에서 어떤 라우트를 쓸지 저장합니다.

```java
package com.example.outbound;

import java.util.ArrayDeque;
import java.util.Deque;

public final class ProxyRouteContextHolder {
    private static final ThreadLocal<Deque<ProxyRoute>> HOLDER =
            ThreadLocal.withInitial(ArrayDeque::new);

    private ProxyRouteContextHolder() {
    }

    public static void push(ProxyRoute route) {
        HOLDER.get().push(route);
    }

    public static ProxyRoute current() {
        Deque<ProxyRoute> stack = HOLDER.get();
        return stack.isEmpty() ? ProxyRoute.DIRECT : stack.peek();
    }

    public static void pop() {
        Deque<ProxyRoute> stack = HOLDER.get();
        if (!stack.isEmpty()) {
            stack.pop();
        }
        if (stack.isEmpty()) {
            HOLDER.remove();
        }
    }
}
```

---

## 5) MethodInterceptor

```java
package com.example.outbound;

import java.lang.reflect.Method;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
import org.springframework.aop.support.AopUtils;
import org.springframework.core.annotation.AnnotatedElementUtils;

public class UseOutboundProxyMethodInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = invocation.getThis().getClass();
        Method method = AopUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);

        UseOutboundProxy ann =
                AnnotatedElementUtils.findMergedAnnotation(method, UseOutboundProxy.class);

        if (ann == null) {
            ann = AnnotatedElementUtils.findMergedAnnotation(targetClass, UseOutboundProxy.class);
        }

        ProxyRoute route = (ann != null) ? ann.value() : ProxyRoute.DIRECT;

        ProxyRouteContextHolder.push(route);
        try {
            return invocation.proceed();
        } finally {
            ProxyRouteContextHolder.pop();
        }
    }
}
```

---

## 6) Advisor

```java
package com.example.outbound;

import java.lang.reflect.Method;
import org.aopalliance.aop.Advice;
import org.springframework.aop.support.StaticMethodMatcherPointcutAdvisor;
import org.springframework.core.annotation.AnnotatedElementUtils;

public class UseOutboundProxyAdvisor extends StaticMethodMatcherPointcutAdvisor {
    public UseOutboundProxyAdvisor(Advice advice) {
        super(advice);
    }

    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        return AnnotatedElementUtils.hasAnnotation(method, UseOutboundProxy.class)
                || AnnotatedElementUtils.hasAnnotation(targetClass, UseOutboundProxy.class);
    }
}
```

---

## 7) Apache HttpClient 기반 설정

여기서 실무 포인트는 3가지입니다.

1. **direct용 HttpClient**
    
2. **proxy용 HttpClient**
    
3. **proxy 인증 필요 시 `CredentialsProvider` 설정**  
    Apache HttpClient의 `HttpClientBuilder`는 `setProxy`, `setRoutePlanner`, `setDefaultCredentialsProvider`, `setMaxConnTotal`, `setMaxConnPerRoute` 등을 제공합니다. 또한 connection manager는 커넥션 생명주기와 persistent connection 접근을 관리합니다. ([Apache HttpComponents](https://hc.apache.org/httpcomponents-client-4.5.x/current/httpclient/apidocs/org/apache/http/impl/client/HttpClientBuilder.html "HttpClientBuilder (Apache HttpClient 4.5.14 API)"))
    

```java
package com.example.config;

import com.example.outbound.*;
import org.apache.http.HttpHost;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.conn.DefaultProxyRoutePlanner;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.impl.client.CloseableHttpClient;
import org.springframework.aop.Advisor;
import org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestOperations;
import org.springframework.web.client.RestTemplate;

@Configuration
public class OutboundProxyHttpClientConfig {
    @Bean
    public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
        return new DefaultAdvisorAutoProxyCreator();
    }

    @Bean
    public UseOutboundProxyMethodInterceptor useOutboundProxyMethodInterceptor() {
        return new UseOutboundProxyMethodInterceptor();
    }

    @Bean
    public Advisor useOutboundProxyAdvisor(UseOutboundProxyMethodInterceptor interceptor) {
        return new UseOutboundProxyAdvisor(interceptor);
    }

    @Bean
    public PoolingHttpClientConnectionManager poolingConnectionManager() {
        PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(200);
        cm.setDefaultMaxPerRoute(50);
        return cm;
    }

    @Bean
    public RequestConfig defaultRequestConfig() {
        return RequestConfig.custom()
                .setConnectTimeout(3000)
                .setSocketTimeout(5000)
                .setConnectionRequestTimeout(2000)
                .build();
    }

    @Bean
    @Qualifier("directRestOperations")
    public RestOperations directRestOperations(
            PoolingHttpClientConnectionManager cm,
            RequestConfig requestConfig) {

        CloseableHttpClient httpClient = HttpClientBuilder.create()
                .setConnectionManager(cm)
                .setDefaultRequestConfig(requestConfig)
                .disableCookieManagement()
                .build();

        HttpComponentsClientHttpRequestFactory rf =
                new HttpComponentsClientHttpRequestFactory(httpClient);
        return new RestTemplate(rf);
    }

    @Bean
    @Qualifier("partnerProxyRestOperations")
    public RestOperations partnerProxyRestOperations(
            PoolingHttpClientConnectionManager cm,
            RequestConfig requestConfig) {

        HttpHost proxy = new HttpHost("proxy.company.local", 8080);

        CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(
                new AuthScope("proxy.company.local", 8080),
                new UsernamePasswordCredentials("proxyUser", "proxyPassword")
        );

        CloseableHttpClient httpClient = HttpClientBuilder.create()
                .setConnectionManager(cm)
                .setDefaultRequestConfig(requestConfig)
                .setRoutePlanner(new DefaultProxyRoutePlanner(proxy))
                .setDefaultCredentialsProvider(credentialsProvider)
                .disableCookieManagement()
                .build();

        HttpComponentsClientHttpRequestFactory rf =
                new HttpComponentsClientHttpRequestFactory(httpClient);
        return new RestTemplate(rf);
    }
}
```

### 왜 이 구성이 좋은가

|항목|설명|
|---|---|
|`DefaultProxyRoutePlanner`|JVM system property와 분리된 고정 proxy 사용|
|`PoolingHttpClientConnectionManager`|멀티스레드 환경에서 커넥션 재사용과 관리에 적합|
|`CredentialsProvider`|proxy 인증을 client 단위로 분리 가능|
|`RequestConfig`|timeout을 client 단위로 일관되게 적용 가능|
|Apache HttpClient는 route planner로 프록시 경로를 계산하고, `CredentialsProvider`로 인증 정보를 제공하며, connection manager로 커넥션 생명주기를 관리합니다. ([Apache HttpComponents](https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/connmgmt.html "Chapter 2. Connection management"))||

---

## 8) 실제 라우팅용 래퍼

서비스 코드가 `RestTemplate` 2개를 직접 알지 않게 감싸는 것이 좋습니다.

```java
package com.example.outbound;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestOperations;

@Component
public class ProxyAwareRestOperations {
    private final RestOperations direct;
    private final RestOperations proxy;

    public ProxyAwareRestOperations(
            @Qualifier("directRestOperations") RestOperations direct,
            @Qualifier("partnerProxyRestOperations") RestOperations proxy) {
        this.direct = direct;
        this.proxy = proxy;
    }

    public <T> ResponseEntity<T> exchange(
            String url,
            HttpMethod method,
            HttpEntity<?> requestEntity,
            Class<T> responseType) {

        return current().exchange(url, method, requestEntity, responseType);
    }

    private RestOperations current() {
        return ProxyRouteContextHolder.current() == ProxyRoute.PARTNER_PROXY
                ? proxy
                : direct;
    }
}
```

---

## 9) 외부 연동 Client 예제

```java
package com.example.partner;

import com.example.outbound.ProxyAwareRestOperations;
import org.springframework.http.*;
import org.springframework.stereotype.Component;

@Component
public class PartnerApiClient {
    private final ProxyAwareRestOperations rest;

    public PartnerApiClient(ProxyAwareRestOperations rest) {
        this.rest = rest;
    }

    public String getPartnerOrder(String orderNo) {
        String url = "https://partner.example.com/api/orders/" + orderNo;

        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Client-Id", "commerce-admin");

        ResponseEntity<String> response = rest.exchange(
                url,
                HttpMethod.GET,
                new HttpEntity<>(headers),
                String.class
        );
        return response.getBody();
    }
}
```

---

## 10) Service 예제

`@UseOutboundProxy`가 붙은 메서드만 proxy client를 타게 됩니다.

```java
package com.example.service;

import com.example.outbound.UseOutboundProxy;
import com.example.partner.PartnerApiClient;
import org.springframework.stereotype.Service;

@Service
public class PartnerSyncService {
    private final PartnerApiClient partnerApiClient;

    public PartnerSyncService(PartnerApiClient partnerApiClient) {
        this.partnerApiClient = partnerApiClient;
    }

    @UseOutboundProxy
    public String syncPartnerOrder(String orderNo) {
        return partnerApiClient.getPartnerOrder(orderNo);
    }

    public String callInternalApi(String orderNo) {
        return "internal-" + orderNo;
    }
}
```

---

## 11) 실무에서 바로 써도 되는 이유

|항목|이유|
|---|---|
|전역 부작용 없음|`System.setProperty`를 건드리지 않음|
|동시 요청 안전성 높음|프록시 여부가 `HttpClient` 인스턴스 단위로 분리됨|
|확장 쉬움|`PARTNER_A_PROXY`, `CARRIER_PROXY` 등 route 추가 가능|
|운영 추적 쉬움|어느 메서드가 프록시를 타는지 annotation 기준으로 명확|
|성능 대응 가능|connection pool, timeout, 인증을 개별 조정 가능|
|Spring은 `RestTemplate`에 Apache HttpComponents 기반 request factory를 붙일 수 있고, 그 factory는 underlying HTTP client의 인증·풀링 등 설정을 활용할 수 있습니다. Apache HttpClient의 builder 역시 proxy, credentials provider, request config, connection limits를 직접 설정하도록 설계되어 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html?utm_source=chatgpt.com "Integration"))||

---

## 12) 주의사항

### 12.1 self-invocation

같은 클래스 내부에서 `this.syncPartnerOrder()`처럼 호출하면 Spring AOP 프록시를 거치지 않아 `Advisor`가 적용되지 않을 수 있습니다. Spring AOP는 프록시 기반이므로 외부 프록시 경유 호출 기준으로 설계해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/integration.html "Integration"))

### 12.2 connection manager 공유 여부

위 예제는 동일 `PoolingHttpClientConnectionManager`를 공유했지만, **proxy 대상과 direct 대상의 성격이 많이 다르면 manager도 분리**하는 편이 더 안전합니다. Apache HttpClient의 connection manager는 커넥션 생성과 persistent connection 수명주기 및 접근 동기화를 담당합니다. ([Apache HttpComponents](https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/connmgmt.html "Chapter 2. Connection management"))

### 12.3 HTTPS proxy / CONNECT / 인증 방식

## 프록시가 Basic 외에 NTLM, SPNEGO, Kerberos 등을 요구할 수 있으므로, 실제 사내 proxy 정책에 맞는 인증 스킴 확인이 필요합니다. Apache HttpClient는 표준 인증 방식과 NTLM, SPNEGO 같은 비표준 인증도 지원합니다. ([Apache HttpComponents](https://hc.apache.org/httpcomponents-client-4.5.x/current/tutorial/html/authentication.html?utm_source=chatgpt.com "Chapter 4. HTTP authentication"))

## 13) 한 줄 결론

**Spring Framework 5.3 실무에서는 `Advisor`로 대상 메서드만 선택하고, 실제 프록시 적용은 Apache HttpClient가 설정된 전용 `RestTemplate`로 분리하는 방식이 가장 안전하고 운영 친화적입니다.**  
원하시면 다음 단계로 **proxy 인증 포함 로그/timeout/retry/pool 분리까지 반영한 운영형 버전**으로 확장해드리겠습니다.