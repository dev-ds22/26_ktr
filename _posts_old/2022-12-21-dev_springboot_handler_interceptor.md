---
layout: single
title: "[SPRING] Handler Interceptor"
excerpt: "Handler Interceptor 설명"

categories:
  - tech
tags:
  - [tech, spring boot, logback]

toc: false
toc_sticky: true

date: 2022-12-21
last_modified_at: 2022-12-21
---
# Handler Interceptor

![handler_interceptor](./../../images/tech/spring_handler_interceptor.png)

- 클라이언트 Http Request
- Spring Container내에서 Handler Mapping(수행할 handler 결정) 
- HandlerAdaptor(결정된 핸들러 수행) 전/후 과정에서 interceptor 동작.
- 핸들러(컨트롤러)의 수정없이 핸들러 수행 전/후처리 동작을 추가하여 핸들러(컨트롤러)의 반복적인 코드를 제거하기 위한 목적으로 사용.
- HandlerMapping이 결정한 handler을 HandlerAdapter 수행 전, 후로 가로체어 추가적인 작업이 가능. 
- View 렌더링 이후 클라이언트에게 Response를 전달하기 전 추가적인 작업이 가능.
- Filter와 매우 유사하나 세분화된 작업에 사용
  - ex) Handler 반복 코드 제거, 권한 확인.

```java
  package org.springframework.web.servlet;

  public interface HandlerInterceptor {

      default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
          return true;
      }

      default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {

      }

      default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {

      }
  }
```
  

- preHandle : 지정된 컨트롤러의 동작 이전에 수행할 동작 (사전 제어).
- postHandle : 지정된 컨트롤러의 동작 이후에 처리할 동작 (사후 제어).
  - Spring MVC의 Dispatcher Servlet이 화면을 처리하기 전에 동작.
- afterCompletion : Dispatcher Servlet의 화면 처리가 완료된 이후 처리할 동작.

## 동작 순서 (DispatcherServlet의 관점)

- 1. HandlerMapping 할당
- 2. HandlerAdapter 할당
- 3. **preHandler**
- 4. HandlerAdapter 수행
- 5. **postHandler**
- 6. View 렌더링
- 7. **afterCompletion**

### HandlerMapping 사용할 핸들러 결정

- HandlerExecutionChain에 사용할 핸들러를 지정함

### HandlerAdapter 사용할 핸들러 결정

- preHandler() 메소드 호출
  - HandlerExecutionChain에 여러 인터셉터를 두어 지정된 핸들러를 처리. 이를 통해 interceptor는 HTTP 오류를 보내거나 False를 반환하여 실행 체인 중단가능.

```java
  if (!mappedHandler.applyPreHandle(processedRequest, response)) {
    return;
  }
```
  
- return값이 false 면, 바디가 없는 200 ok반환. (response에 응답바디 설정 가능)
- true면 다음 작업이 실행

### HandlerAdapter 수행
  
### postHandler() 메소드 호출
  
### View 렌더링 수행
  
### afterCompletion 호출

## Multi Interceptor

- 2개 이상의 Interceptor가 등록되었을 때의 동작순서.

- 1. HandlerMapping 할당  
- 2. HandlerAdapter 할당  
- 3-1. preHandler 1  
- 3-2. preHandler 2  
- 4. HandlerAdapter 수행  
- 5-1. postHandler 2  
- 5-2. postHandler 1  
- 6. View 렌더링  
- 7-1. afterCompletion 2  
- 7-2. afterCompletion 1  

## 코드 확인

```java
  @Configuration
  public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
      registry.addInterceptor(new GumpInterceptor()).order(2);
      registry.addInterceptor(new GumpGumpInterceptor()).order(1)
    }
  }
```

- 기본적으로 선언 순선에 따라 실행, order메소드를 통해 순서 지정.

### 커스텀 인터셉터가 여러개 일 때, 특정 url을 지정방법.

```java
  @Override
  public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new GumpInterceptor()).addPathPatterns("/test/**");
    registry.addInterceptor(new GumpGumpInterceptor()).addPathPatterns("/test/**");
  }
```


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- LoggingInterceptor.java

```java
  @Slf4j
  @RequiredArgsConstructor
  @Component
  public class LoggingInterceptor implements HandlerInterceptor {
    private final ObjectMapper objectMapper;

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
      log.info("afterCompletion");
      if (request.getClass().getName().contains("SecurityContextHolderAwareRequestWrapper")
      ||request.getClass().getName().contains("StandardMultipartHttpServletRequest")) return;

      final ContentCachingRequestWrapper cachingRequest = (ContentCachingRequestWrapper) request;
      final ContentCachingResponseWrapper cachingResponse = (ContentCachingResponseWrapper) response;

      if (cachingRequest.getContentType() != null && cachingRequest.getContentType().contains("application/json")) {
        if (cachingRequest.getContentAsByteArray() != null && cachingRequest.getContentAsByteArray().length != 0){
          log.info("Request Body : {}", objectMapper.readTree(cachingRequest.getContentAsByteArray()));
        }
      }

      if (cachingResponse.getContentType() != null && cachingResponse.getContentType().contains("application/json")) {
        if (cachingResponse.getContentAsByteArray() != null && cachingResponse.getContentAsByteArray().length != 0) {
            log.info("Response Body : {}", objectMapper.readTree(cachingResponse.getContentAsByteArray()));
        }
      }
    }
  }
```

  </pre>
</details>