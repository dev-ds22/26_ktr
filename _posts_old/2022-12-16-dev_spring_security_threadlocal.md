---
layout: single
title: "[Spring Security] SecurityContextHolder 와 ThreadLoacal "
excerpt: "SecurityContextHolder, SecurityContext"

categories:
  - tech
tags:
  - [tech, spring security, threadlocal]

toc: false
toc_sticky: true

date: 2022-12-16
last_modified_at: 2022-12-16
---
# Spring Security 에서의 ThreadLocal

- 스프링 시큐리티 내에서는 인증 결과(=인증토큰)를 SecurityContext 라는 곳에 저장
- 이후 처리에서는 전역적으로 사용가능.

## SecurityContext, SecurityContextHolder

### 1. SecurityContext

- Authentication 객체가 저장되는 보관소
- SecurityContextHolder 전략(Strategy)에 따라 SecurityContext의 저장 방식이 다름
- 일반적으로는 ThreadLocal 에 저장, 덕분에 코드 어디서나 Authentication 을 꺼내서 사용가능
- 추가적으로 인증이 완료되면 세션에도 저장됨
- ThreadLocal : 쓰레드마다 갖는 고유한 저장공간이라고 생각하면 된다.


### 2. SecurityContextHolder

- SecurityContext 를 감싸는(저장하는) 객체
- (일반적으로) SecurityContext 저장을 위한 ThreadLocal 를 갖고 있는 객체
- SecurityContext 객체의 저장 방식(전략, Strategy)을 지정
  - MODE_THREADLOCAL : 스레드당 SecurityContext 객체를 할당, 기본값
  - MODE_INHERITABLETHREADLOCAL: 메인, 자식 스레드에서 동일한 SecurityContext 사용
  - MODE_GLOBAL: 프래그램에서 딱 하나의 SecurityContext만 저장

- 각 전략에 따른 전략 클래스가 존재하며, SecurityContextHolder의 메소드 대부분이 이 전략 클래스의 인스턴스에게 작업을 위임하는 형태로 동작.
- SecurityContextHolder.clearContext() : 기존 SecurityContext 정보 초기화

>   
> SecurityContextHolder 의 기본 설정은 SecurityContext 정보를   
> Local Thread 만 공유하도록 되어있기 때문 SecurityContextHolder를  
> 직접 하위 Thread 안에서 호출하여 사용하는것보다, 메인 Thread 에서  
> 호출하여 해당 값을 하위 Thread 에서 참조하도록 하는것이,  
> 성능적으로나 가시적으로도 더 깔끔.
>  
> ParallelStream 혹은 Async 관련된 기능을 사용 시 하위 Thread 에서   
> SecurityContextHolder 를 사용해야하는 경우가 있다면   
> SecurityContextHolder 의 공유 모드를 MODE_INHERITABLETHREADLOCAL  
> 로 낮추는것을 고려필요.  
> 

- 참고: SecurityContextHolder, SecurityContext 를 통한 인증객체 읽는 법  

```java
  Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

## 인증 프로세스와 저장소

![security_authentication](./../../images/tech/image_spring_audentication.png)

- 로그인 시도
- 요청을 받는 스레드 하나 생성
- 해당 스레드의 고유한 저장소인 ThreadLocal이 존재함
- 인증을 실패하면 SecurityContextHolder.clearContext()
- 인증에 성공하면 SecurityContextHolder > SecurityContext 에 인증 토큰 저장
- HttpSession 에도 SecurityContext 저장


### 1. SecurityContextHolder

- SecurityContextHolder 의 java document.
>
> Associates a given SecurityContext with the current  
> execution thread. This class provides a series of static  
> methods that delegate to an instance of SecurityContextHolderStrategy.
>  
>  "이 클래스(SecurityContextHolder)는 다양한 static 메소드를 제공하고, 
> 해당 메소드들은 내부적으로 SecurityContextHolderStrategy에게 위임처리를 한다"  
>

- SecurityContextHolder 의 실제 일처리는 SecurityContextHolderStrategy 가 처리
- SecurityContextHolder 초기화 시, initializeStrategy 메소드 호출 후
- 내부적으로 SecurityContextHolderStrategy 를 결정.
- 기본값은 MODE_THREADLOCAL 모드, 별도 설정이 없을 시 ThreadLocalSecurityContextHolderStrategy 사용.

### 2. ThreadLocalSecurityContextHolderStrategy

- MODE_THREADLOCAL 는 스레드당 하나의 SecurityContext 객체를 할당하는 전략.
- MODE_THREADLOCAL을 위한 SecurityContextHolder 의 내부 전략이 ThreadLocalSecurityContextHolderStrategy이다.
- SecurityContext 객체를 스레드당 하나만 생성한다는 전략을 위해 이 클래스는
ThreadLocal<SecurityContext> 필드를 하나민 소유.
- createEmptyContext()에서 SecurityContext를 생성.

## ThreadLocal의 사용예시 (Spring Security Context Holder)

- Spring Security에서 Context Holder를 ThreadLocal로 구현.
- 이는 각 쓰레드들이 각자의 context holder를 가지고 있다는 것이며 그렇기 때문에 무수한 HTTP 요청으로 쓰레드가 다수 생성되어도 ContextHolder에서 취득한 인증정보들이 꼬이지 않는 것.

```java
final class ThreadLocalSecurityContextHolderStrategy implements
        SecurityContextHolderStrategy {

  private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();

  public SecurityContext getContext() {
    SecurityContext ctx = contextHolder.get();
    if (ctx == null) {
      ctx = createEmptyContext();
      contextHolder.set(ctx);
    }
    return ctx;
  }

  public void setContext(SecurityContext context) {
    Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
    contextHolder.set(context);
  }

  public SecurityContext createEmptyContext() {
    return new SecurityContextImpl();
  }

  // 쓰레드풀을 사용하는 환경에서는 이 부분이 매우 중요.
  public void clearContext() { 
    contextHolder.remove();
  } 
}
```

## ThreadLocal 을 사용할 때 주의점
- 쓰레드 생성비용이 커서 미리 만들어 둔 쓰레드들 재사용
- 쓰레드풀을 관리하는 톰캣(Tomcat) 아저씨를 우리가 사용하는 경우, **이전의 ThreadLocal 변수를 참조하지 않도록 Clear 해주는 작업이 필요.**

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- 위 내용과는 무관(Spring Security Filter인듯해 남겨둠)  
- UrlDecodeFilter.java  
  
```java
  @Component
  public class UrlDecodeFilter extends OncePerRequestFilter {

    public class UrlDecodeWrapper extends HttpServletRequestWrapper {
        HttpServletRequest request;

      public UrlDecodeWrapper(HttpServletRequest request) {
        super(request);
        this.request = request;
      }

      public String[] getParameterValues(String name) {
        String[] result = null;
        try {
            result = Arrays.stream(request.getParameterValues(name))
                    .map(param -> decode(param, "UTF-8"))
                    .toArray(String[]::new);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
      }

      public String decode(String s, String enc) {
        try {
            return URLDecoder.decode(s, enc);
        } catch (UnsupportedEncodingException e) {
            return null;
        }
      }
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
      UrlDecodeWrapper urlDecodeWrapper = new UrlDecodeWrapper((HttpServletRequest) request);
      
      filterChain.doFilter(urlDecodeWrapper, response);
    }
  }
```

  </pre>
</details>