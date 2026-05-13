---
layout: single
title: "RequestContextHolder"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: 2026-05-13
last_modified_at: 2026-05-13 17:31:44
---

# RequestContextHolder

정확도 94%

## 1. `RequestContextHolder`란?

`RequestContextHolder`는 Spring Web에서 **현재 HTTP 요청 정보를 현재 실행 중인 Thread에 묶어 보관하는 Holder 클래스**입니다. 공식 Javadoc 기준으로 현재 웹 요청을 `RequestAttributes` 형태로 **thread-bound object**로 노출하며, `inheritable=true` 설정 시 자식 Thread로도 상속될 수 있습니다. 또한 Spring MVC의 `DispatcherServlet`은 기본적으로 현재 요청을 `RequestContextHolder`에 노출합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html "RequestContextHolder (Spring Framework 5.3.46 API)"))  
즉, Controller나 같은 요청 Thread 안의 Service 코드에서 아래처럼 현재 요청 객체를 꺼낼 수 있습니다.

```java
ServletRequestAttributes attrs =
        (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
HttpServletRequest request = attrs.getRequest();
```

## 2. 내부 구조

개념적으로는 아래 구조입니다.

```text
현재 요청 Thread
 └─ ThreadLocal<RequestAttributes>
      └─ ServletRequestAttributes
           ├─ HttpServletRequest
           └─ HttpServletResponse
```

Spring 5.3 기준 핵심 타입은 다음과 같습니다.

| 구분                         | 설명                                                                   |
| -------------------------- | -------------------------------------------------------------------- |
| `RequestContextHolder`     | 현재 Thread에 바인딩된 `RequestAttributes`를 보관/조회                           |
| `RequestAttributes`        | request/session scope attribute 접근용 추상 인터페이스                         |
| `ServletRequestAttributes` | Servlet 기반 구현체. 내부적으로 `HttpServletRequest`, `HttpServletResponse` 보관 |
| `ThreadLocal`              | 요청을 처리하는 현재 Thread에서만 조회 가능                                          |
| `InheritableThreadLocal`   | `inheritable=true`일 때 자식 Thread로 상속 가능                               |
- `ServletRequestAttributes`는 Servlet 기반 `RequestAttributes` 구현체이며, **request scope와 HTTP session scope의 attribute 접근을 담당**합니다. 공식 Javadoc에서도 `HttpServletRequest`, `HttpServletResponse`, `HttpSession`을 감싸는 역할을 설명합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/ServletRequestAttributes.html "ServletRequestAttributes (Spring Framework 5.3.46 API)"))

## 3. 주요 메서드

|메서드|설명|실무 주의|
|---|---|---|
|`getRequestAttributes()`|현재 Thread에 바인딩된 `RequestAttributes` 반환|없으면 `null`|
|`currentRequestAttributes()`|현재 Thread의 `RequestAttributes` 반환|없으면 `IllegalStateException`|
|`setRequestAttributes(attrs)`|현재 Thread에 `RequestAttributes` 바인딩|일반 업무 코드에서 직접 호출은 비추천|
|`setRequestAttributes(attrs, true)`|자식 Thread에 상속 가능한 형태로 바인딩|ThreadPool 환경에서 위험|
|`resetRequestAttributes()`|현재 Thread의 요청 컨텍스트 제거|누락 시 Thread 재사용 환경에서 오염 가능|
|공식 Javadoc 기준 `getRequestAttributes()`는 없으면 `null`을 반환하지만, `currentRequestAttributes()`는 바인딩된 객체가 없으면 `IllegalStateException`을 발생시킵니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html "RequestContextHolder (Spring Framework 5.3.46 API)"))|||

## 4. 생명주기 전체 흐름

### 4.1 요청 시작 전

사용자가 URL을 호출하면 요청은 WAS의 요청 처리 Thread에 배정됩니다.

```text
Client 요청
 → WAS Thread 할당
 → Filter Chain
 → DispatcherServlet
 → Controller
```

이 시점부터 중요한 점은 **HTTP 요청 1건이 특정 Thread에서 처리되는 동안에만 `RequestContextHolder`가 유효하다**는 것입니다.

### 4.2 요청 컨텍스트 바인딩

Spring MVC에서는 보통 `DispatcherServlet`이 현재 요청을 `RequestContextHolder`에 바인딩합니다. 공식 Javadoc에서도 `DispatcherServlet`이 기본적으로 현재 request를 노출한다고 설명합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html "RequestContextHolder (Spring Framework 5.3.46 API)"))  
개념적으로는 아래와 같습니다.

```java
ServletRequestAttributes attributes =
        new ServletRequestAttributes(request, response);
RequestContextHolder.setRequestAttributes(attributes);
```

Spring MVC가 아닌 서블릿, 외부 서블릿, JSF 등에서 현재 요청을 노출해야 할 때는 `RequestContextFilter` 또는 `RequestContextListener`를 사용할 수 있습니다. `RequestContextFilter`는 현재 요청을 `LocaleContextHolder`와 `RequestContextHolder`에 노출하는 Servlet Filter이며, Spring MVC 내부에서는 `DispatcherServlet`만으로 충분하다고 공식 Javadoc에 설명되어 있습니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/filter/RequestContextFilter.html "RequestContextFilter (Spring Framework 5.3.46 API)"))

### 4.3 Controller / Service 실행 중

요청이 처리되는 동안 같은 Thread에서는 어디서든 아래처럼 현재 요청을 꺼낼 수 있습니다.

```java
ServletRequestAttributes attrs =
        (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
if (attrs != null) {
    HttpServletRequest request = attrs.getRequest();
    String clientIp = request.getRemoteAddr();
}
```

이때 접근 가능한 범위는 **현재 요청 Thread 안**입니다.

```text
Controller
 → Service
 → Repository
 → Utility
```

위 호출 흐름이 같은 요청 Thread에서 수행된다면 `RequestContextHolder` 접근이 가능합니다.

## 5. 요청 완료 시점의 생명주기

요청 처리가 끝나면 Spring은 반드시 `RequestContextHolder`를 정리해야 합니다.  
개념 흐름은 다음과 같습니다.

```text
Controller 처리 완료
 → View 렌더링 또는 Response 작성
 → finally 영역에서 RequestContextHolder 초기화
 → RequestAttributes.requestCompleted() 호출
 → request scope destruction callback 실행
 → accessed session attribute 업데이트
 → WAS Thread 반환
```

`RequestContextFilter`의 공식 구현 설명에서도 filter 처리 후 `finally`에서 context를 reset하고 `attributes.requestCompleted()`를 호출하는 구조가 확인됩니다. ([GitHub](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/web/filter/RequestContextFilter.java "spring-framework/spring-web/src/main/java/org/springframework/web/filter/RequestContextFilter.java at main · spring-projects/spring-framework · GitHub"))  
또한 `AbstractRequestAttributes.requestCompleted()`는 요청 완료를 알리고, request destruction callback 실행 및 접근된 session attribute 업데이트를 수행한다고 공식 Javadoc에 설명되어 있습니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/AbstractRequestAttributes.html?utm_source=chatgpt.com "AbstractRequestAttributes (Spring Framework 5.3.46 API)"))

## 6. 생명주기 요약

```text
[1] HTTP 요청 수신
[2] WAS 요청 Thread 할당
[3] DispatcherServlet 또는 RequestContextFilter가 ServletRequestAttributes 생성
[4] RequestContextHolder에 ThreadLocal로 저장
[5] Controller / Service / Utility에서 현재 요청 조회 가능
[6] 요청 처리 완료
[7] RequestContextHolder.resetRequestAttributes()
[8] ServletRequestAttributes.requestCompleted()
[9] ThreadLocal 제거
[10] WAS Thread가 Thread Pool로 반환
```

## 7. `DispatcherServlet`, `RequestContextFilter`, `RequestContextListener` 차이

| 구분                       | 역할                                       | 사용 위치                | 실무 판단                                           |
| ------------------------ | ---------------------------------------- | -------------------- | ----------------------------------------------- |
| `DispatcherServlet`      | Spring MVC 요청 처리 중 request context 자동 노출 | Spring MVC 기본 구조     | 일반 Spring MVC에서는 이것만으로 충분                       |
| `RequestContextFilter`   | Filter 단계에서 request context 노출           | `web.xml`, Filter 등록 | DispatcherServlet 밖의 Filter/Servlet에서도 필요할 때 사용 |
| `RequestContextListener` | Servlet listener로 request context 노출     | `web.xml` listener   | 외부 Servlet 연동 시 사용 가능하지만 고급 옵션은 적음              |

`RequestContextListener`도 현재 요청을 `LocaleContextHolder`와 `RequestContextHolder`에 노출하지만, `RequestContextFilter`와 `DispatcherServlet`에 비해 `threadContextInheritable` 같은 고급 옵션은 부족하다고 공식 Javadoc에서 설명합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextListener.html "RequestContextListener (Spring Framework 5.3.46 API)"))

## 8. 사용 가능/불가능 범위

|위치|사용 가능 여부|설명|
|---|--:|---|
|Controller|가능|DispatcherServlet 처리 Thread 내부|
|HandlerInterceptor|가능|보통 MVC 요청 Thread 내부|
|Service|가능|같은 요청 Thread에서 호출될 때 가능|
|Repository|가능|기술적으로 가능하나 권장도 낮음|
|JSP/View 렌더링|가능|요청 처리 흐름 내부|
|일반 `@Async`|기본 불가|다른 Thread에서 실행됨|
|`ExecutorService`|기본 불가|ThreadLocal이 자동 전달되지 않음|
|Scheduler|불가|HTTP 요청 Thread가 아님|
|Batch|불가|HTTP 요청 생명주기 밖|
|JUnit 단위 테스트|기본 불가|직접 바인딩하지 않으면 없음|

## 9. `@Async`, ThreadPool에서 주의할 점

`RequestContextHolder`는 기본적으로 `ThreadLocal` 기반이므로, 요청 Thread와 다른 Thread에서는 보통 조회되지 않습니다.

```java
@Async
public void asyncWork() {
    RequestContextHolder.getRequestAttributes(); // 대부분 null
}
```

`RequestContextFilter`에는 `threadContextInheritable` 옵션이 있지만 기본값은 `false`입니다. 공식 Javadoc은 이 옵션을 `true`로 바꾸면 자식 Thread에 상속할 수 있지만, `ThreadPoolExecutor`처럼 Thread를 재사용하거나 필요 시 새 Thread를 추가하는 풀에서는 inherited context가 pooled thread에 노출될 수 있으므로 사용하지 말라고 경고합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/filter/RequestContextFilter.html "RequestContextFilter (Spring Framework 5.3.46 API)"))

### 실무 권장 방식

나쁜 방식:

```java
@Async
public void asyncWork() {
    HttpServletRequest request =
        ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
}
```

권장 방식:

```java
public void controllerMethod(HttpServletRequest request) {
    String userId = request.getHeader("X-USER-ID");
    String clientIp = request.getRemoteAddr();
    asyncService.asyncWork(userId, clientIp);
}
```

즉, 비동기 작업에는 `HttpServletRequest` 자체를 넘기지 말고 **필요한 값만 추출해서 전달**하는 것이 안전합니다.

## 10. 안전한 사용 예제

### 10.1 현재 Request 조회 Utility

```java
import javax.servlet.http.HttpServletRequest;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
public final class RequestUtils {
    private RequestUtils() {
    }
    public static HttpServletRequest getCurrentRequestOrNull() {
        ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        return attrs != null ? attrs.getRequest() : null;
    }
    public static String getClientIpOrUnknown() {
        HttpServletRequest request = getCurrentRequestOrNull();
        return request != null ? request.getRemoteAddr() : "UNKNOWN";
    }
}
```

### 10.2 예외를 발생시키는 방식

```java
public static HttpServletRequest getRequiredRequest() {
    ServletRequestAttributes attrs =
            (ServletRequestAttributes) RequestContextHolder.currentRequestAttributes();
    return attrs.getRequest();
}
```

이 방식은 요청 Thread가 아닌 곳에서 호출되면 `IllegalStateException`이 발생할 수 있습니다.

## 11. 운영 관점 주의사항

|항목|설명|
|---|---|
|ThreadLocal 누수|직접 `setRequestAttributes()`를 사용했다면 반드시 `finally`에서 `resetRequestAttributes()` 필요|
|ThreadPool 오염|요청 정보가 다른 요청/작업 Thread에 남으면 보안 사고 가능|
|Service 의존성 증가|Service가 HTTP request에 의존하면 테스트와 재사용성이 낮아짐|
|비동기 처리|`@Async`, `CompletableFuture`, 별도 `Executor`에서는 기본적으로 사용 불가|
|인증 정보|Spring Security 사용 시 request 직접 접근보다 `SecurityContextHolder` 또는 명시적 파라미터 전달 검토|
|Spring 5.3/6 차이|Spring 5.3은 `javax.servlet`, Spring 6 이상은 `jakarta.servlet` 기반|

## 12. 가장 중요한 결론

`RequestContextHolder`의 생명주기는 **HTTP 요청 Thread의 생명주기와 거의 동일**합니다.

```text
요청 시작 → ThreadLocal 바인딩 → 요청 처리 중 조회 가능 → 요청 완료 → ThreadLocal 제거
```

실무에서는 다음 기준으로 사용하는 것이 안전합니다.

1. Controller, Interceptor, 공통 로깅, 요청 기반 Utility에서는 제한적으로 사용 가능합니다.    
2. 핵심 Service 로직이 `RequestContextHolder`에 직접 의존하는 구조는 피하는 것이 좋습니다.
3. `@Async`, Scheduler, Batch, ThreadPool에서는 기본적으로 사용할 수 없다고 보는 것이 안전합니다.
4. 비동기 작업에는 request 객체를 넘기지 말고, `userId`, `clientIp`, `traceId` 같은 필요한 값만 추출해서 전달하는 방식이 가장 안정적입니다.


일반적인 **동기 Spring MVC 요청**이라면 하나의 HTTP request는 보통 다음 흐름에서 **하나의 WAS 요청 처리 Thread**로 처리됩니다.

```text
Client Request
→ WAS Thread 1개 할당
→ Filter
→ DispatcherServlet
→ HandlerInterceptor
→ Controller
→ Service
→ Repository
→ View/Response
→ Thread 반환
```

이 경우 `Controller → Service` 호출은 같은 Thread 안에서 이어지므로 `RequestContextHolder.getRequestAttributes()`로 같은 요청 정보를 조회할 수 있습니다. Spring 공식 Javadoc에서도 `RequestContextHolder`는 현재 웹 요청을 **thread-bound RequestAttributes object** 형태로 노출한다고 설명하며, `DispatcherServlet`이 기본적으로 현재 요청을 노출한다고 명시합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html "RequestContextHolder (Spring Framework 5.3.46 API)"))

## 핵심 답변

|질문|답|
|---|---|
|하나의 request는 Controller와 Service에서 하나의 Thread로 처리되는가?|**일반 동기 처리에서는 그렇다.**|
|Controller와 Service에서 같은 `RequestContextHolder` 값을 보는가?|**같은 Thread라면 같은 `RequestAttributes`를 본다.**|
|요청별로 별도 `RequestContextHolder`가 생성되는가?|**아니다. `RequestContextHolder` 클래스가 요청별로 생성되는 것은 아니다.**|
|요청별로 생성되는 것은 무엇인가?|보통 요청별 `ServletRequestAttributes`가 생성되어 현재 Thread의 `ThreadLocal`에 바인딩된다.|
|요청 완료 후에는 어떻게 되는가?|현재 Thread의 `RequestAttributes`가 reset되고 Thread는 WAS Thread Pool로 반환된다.|

## 정확한 구조

`RequestContextHolder` 자체는 요청마다 새로 만들어지는 객체가 아니라, **static Holder 유틸리티 클래스**에 가깝습니다.  
중요한 것은 내부의 `ThreadLocal`입니다.

```text
RequestContextHolder
 └─ static ThreadLocal<RequestAttributes>
      ├─ Thread-A → Request-1의 ServletRequestAttributes
      ├─ Thread-B → Request-2의 ServletRequestAttributes
      └─ Thread-C → Request-3의 ServletRequestAttributes
```

즉, 요청별로 별도 `RequestContextHolder`가 있는 것이 아니라, **Thread별로 다른 `RequestAttributes` 값이 바인딩**됩니다.

## 동기 요청 기준 생명주기

```text
[1] 요청 수신
[2] WAS가 요청 처리 Thread 할당
[3] DispatcherServlet 진입
[4] ServletRequestAttributes 생성
[5] RequestContextHolder에 현재 Thread 기준으로 바인딩
[6] Controller 실행
[7] Service 실행
[8] Repository 실행
[9] 응답 생성
[10] RequestContextHolder reset
[11] WAS Thread Pool로 Thread 반환
```

`RequestContextHolder.setRequestAttributes()`는 주어진 `RequestAttributes`를 현재 Thread에 바인딩하고, `resetRequestAttributes()`는 현재 Thread의 값을 초기화합니다. `getRequestAttributes()`는 현재 Thread에 바인딩된 값을 반환하며 없으면 `null`을 반환합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html "RequestContextHolder (Spring Framework 5.3.46 API)"))

## 예시

### Controller

```java
@GetMapping("/test")
public String test() {
    System.out.println("controller thread = " + Thread.currentThread().getName());
    testService.execute();
    return "ok";
}
```

### Service

```java
@Service
public class TestService {
    public void execute() {
        System.out.println("service thread = " + Thread.currentThread().getName());
        ServletRequestAttributes attrs =
            (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attrs.getRequest();
        System.out.println("uri = " + request.getRequestURI());
    }
}
```

일반 동기 요청이면 로그는 보통 같은 Thread로 나옵니다.

```text
controller thread = default task-15
service thread    = default task-15
```

이 경우 Controller와 Service는 같은 요청의 `ServletRequestAttributes`를 공유합니다.

## 동시 요청이 들어오면 어떻게 되는가?

예를 들어 동시에 3개의 요청이 들어오면 보통 WAS Thread Pool에서 서로 다른 Thread가 배정됩니다.

```text
Request-1 → default task-11 → RequestAttributes-1
Request-2 → default task-12 → RequestAttributes-2
Request-3 → default task-13 → RequestAttributes-3
```

각 요청은 같은 `RequestContextHolder` 클래스를 사용하지만, 내부 `ThreadLocal` 값은 Thread마다 다르기 때문에 서로 섞이지 않습니다.

## 주의: 항상 하나의 Thread라고 단정하면 안 되는 경우

아래 경우에는 Controller와 Service가 항상 같은 Thread라고 보면 안 됩니다.

|경우|설명|
|---|---|
|`@Async`|Service 메서드가 별도 Executor Thread에서 실행됨|
|`CompletableFuture`|직접 만든 비동기 Thread에서 실행될 수 있음|
|`ExecutorService`|요청 Thread와 별도 Thread 사용|
|`Callable` 반환|Spring MVC가 `AsyncTaskExecutor`를 통해 작업 실행|
|`DeferredResult`|결과를 다른 Thread에서 설정 가능|
|`WebAsyncTask`|별도 비동기 실행 흐름 사용|
|Servlet async|요청 Thread가 해제되고 이후 async dispatch로 재진입 가능|
|Spring MVC는 `DeferredResult`, `Callable`, `WebAsyncTask` 같은 Servlet 비동기 요청 처리를 지원합니다. 공식 문서 기준 `DeferredResult`는 다른 Thread에서 결과를 생산할 수 있고, `Callable`의 반환값은 설정된 `AsyncTaskExecutor`를 통해 실행될 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html "Asynchronous Requests :: Spring Framework"))||

## 비동기 처리 시 흐름 예

```java
@GetMapping("/async")
public Callable<String> async() {
    System.out.println("controller thread = " + Thread.currentThread().getName());
    return () -> {
        System.out.println("callable thread = " + Thread.currentThread().getName());
        return "ok";
    };
}
```

예상 로그:

```text
controller thread = default task-15
callable thread   = mvc-task-1
```

이 경우 최초 Controller 진입 Thread와 실제 작업 Thread가 달라질 수 있습니다.

## 실무 판단 기준

### 1. 일반 Controller → Service 호출

```java
controllerMethod() {
    service.method();
}
```

이 구조는 보통 같은 Thread입니다.

```text
Request Thread = Controller Thread = Service Thread
```

### 2. Service 내부에서 비동기 호출

```java
service.method() {
    asyncService.asyncMethod();
}
```

이 구조는 다른 Thread일 수 있습니다.

```text
Request Thread ≠ Async Service Thread
```

### 3. 별도 Thread에서 `RequestContextHolder` 접근

```java
new Thread(() -> {
    RequestContextHolder.getRequestAttributes(); // 보통 null
}).start();
```

기본적으로 `ThreadLocal`은 다른 Thread로 자동 전달되지 않으므로 `null`이 나올 수 있습니다.

## 운영 관점 핵심 주의사항

`RequestContextHolder`는 요청 Thread 안에서는 편리하지만, Service 계층이 강하게 의존하면 구조가 나빠질 수 있습니다.  
권장 방식은 다음과 같습니다.

```java
@GetMapping("/order")
public String order(HttpServletRequest request) {
    String clientIp = request.getRemoteAddr();
    String userAgent = request.getHeader("User-Agent");
    orderService.order(clientIp, userAgent);
    return "ok";
}
```

즉, Service에서 `RequestContextHolder`를 직접 꺼내기보다 Controller나 Interceptor에서 필요한 값만 추출해서 Service에 명시적으로 전달하는 방식이 더 안전합니다.

## 최종 정리

```text
동기 Spring MVC 요청:
요청 1개 = 보통 WAS Thread 1개로 Controller/Service 처리
요청별 RequestContextHolder 생성 = 아님
요청별 ServletRequestAttributes 생성/바인딩 = 맞음
ThreadLocal 기준으로 요청 정보 분리 = 맞음
요청 완료 후 reset = 맞음
비동기/별도 Thread 사용 시 같은 Thread 보장 = 아님
```

따라서 실무적으로는 **“동기 MVC 흐름에서는 하나의 요청이 하나의 Thread에서 처리되고, 그 Thread에 요청별 `ServletRequestAttributes`가 `RequestContextHolder`를 통해 바인딩된다”**고 이해하면 됩니다.


일반적인 Spring Framework 기반 WAS에서 **`RequestContextHolder` 객체가 요청마다 여러 개 생성되는 구조는 아닙니다.**  
정확히는 다음처럼 이해하면 됩니다.

```text
RequestContextHolder 클래스 = JVM/ClassLoader 기준으로 로딩되는 static Holder
요청별로 생성되는 객체 = ServletRequestAttributes
요청별 분리 기준 = ThreadLocal
```

## 핵심 정리

|구분|생성/관리 방식|
|---|---|
|`RequestContextHolder`|Spring의 static 유틸성 Holder 클래스|
|`RequestAttributes`|요청마다 현재 Thread에 바인딩되는 요청 컨텍스트|
|`ServletRequestAttributes`|Servlet 환경에서 실제 request/response를 감싸는 구현체|
|`ThreadLocal`|Thread별로 다른 `RequestAttributes`를 보관|
|요청 완료 후|현재 Thread의 `RequestAttributes` 제거|

## 구조 예시

```text
WAS JVM
└─ Web Application ClassLoader
   └─ RequestContextHolder class
      ├─ static ThreadLocal<RequestAttributes>
      │  ├─ default task-1 → Request-1의 ServletRequestAttributes
      │  ├─ default task-2 → Request-2의 ServletRequestAttributes
      │  └─ default task-3 → Request-3의 ServletRequestAttributes
      └─ static InheritableThreadLocal<RequestAttributes>
```

즉, **`RequestContextHolder`는 하나의 공통 Holder처럼 존재**하고, 요청마다 다른 값은 내부 `ThreadLocal`에 저장됩니다.  
Spring 공식 Javadoc에서도 `RequestContextHolder`는 현재 웹 요청을 `RequestAttributes` 형태의 **thread-bound object**로 노출한다고 설명하며, `getRequestAttributes()`는 “현재 Thread에 바인딩된 `RequestAttributes`”를 반환한다고 명시합니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html?utm_source=chatgpt.com "RequestContextHolder (Spring Framework 5.3.46 API)"))

## 요청별 실제 동작

동기 Spring MVC 요청 기준 흐름은 다음과 같습니다.

```text
[1] HTTP 요청 수신
[2] WAS 요청 Thread 할당
[3] DispatcherServlet 진입
[4] ServletRequestAttributes 생성
[5] RequestContextHolder 내부 ThreadLocal에 저장
[6] Controller 실행
[7] Service 실행
[8] 응답 완료
[9] RequestContextHolder.resetRequestAttributes()
[10] ThreadLocal 값 제거
```

따라서 `Controller`와 `Service`가 같은 요청 Thread에서 실행되면 같은 `ServletRequestAttributes`를 조회합니다.

## 중요한 오해 정리

### 오해 1. 요청마다 `RequestContextHolder`가 생성된다

아닙니다.  
`RequestContextHolder`는 요청 객체가 아니라 **현재 Thread에 묶인 요청 정보를 꺼내는 static 접근점**에 가깝습니다.

### 오해 2. WAS 전체에서 정말 물리적으로 딱 1개만 존재한다

대부분의 단일 웹 애플리케이션에서는 그렇게 볼 수 있지만, 정확히는 **ClassLoader 기준**입니다.  
예를 들어 같은 WAS에 여러 WAR가 배포되어 있고 각 WAR가 Spring 라이브러리를 따로 로딩하면, 애플리케이션별 ClassLoader마다 `RequestContextHolder` 클래스가 따로 로딩될 수 있습니다.

```text
JBoss/WAS
├─ app1.war ClassLoader → RequestContextHolder
├─ app2.war ClassLoader → RequestContextHolder
└─ shared module ClassLoader → 공용 Spring 로딩 시 공유 가능
```

실무적으로 한 애플리케이션 안에서는 **하나의 `RequestContextHolder` static Holder + Thread별 값 분리**로 이해하면 됩니다.

## 예시 코드

```java
ServletRequestAttributes attrs =
    (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
```

위 코드는 전역 객체에서 값을 가져오는 것처럼 보이지만, 실제 의미는 아래에 가깝습니다.

```java
현재 실행 중인 Thread에 바인딩된 RequestAttributes를 가져온다
```

공식 Javadoc 기준 `getRequestAttributes()`는 현재 Thread에 바인딩된 값이 없으면 `null`을 반환하고, `currentRequestAttributes()`는 없으면 `IllegalStateException`을 발생시킵니다. ([Spring Enterprise Documentation](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/web/context/request/RequestContextHolder.html?utm_source=chatgpt.com "RequestContextHolder (Spring Framework 5.3.46 API)"))

## 최종 답

```text
일반적인 Spring MVC WAS에서 RequestContextHolder는 요청마다 생성되지 않는다.
한 애플리케이션 ClassLoader 기준으로 static Holder처럼 존재한다.
요청마다 생성/바인딩되는 것은 ServletRequestAttributes이다.
요청별 분리는 RequestContextHolder 내부의 ThreadLocal로 처리된다.
```

따라서 질문에 대한 답은 **“일반적으로 하나로 보면 맞지만, 정확히는 애플리케이션 ClassLoader 기준 하나의 static Holder이고, 요청별 데이터는 ThreadLocal에 따로 저장된다”**입니다.

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
