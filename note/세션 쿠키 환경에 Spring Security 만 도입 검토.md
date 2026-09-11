## 1. 결론

현재 구조가
```text
Browser
  │
  │ JSESSIONID Cookie
  ▼
WAS
  │
  └─ HttpSession
       ├─ 로그인 사용자 정보
       ├─ 권한 정보
       └─ 기타 업무 정보
```
라면 Token/JWT로 바꾸지 않고 Spring Security만 도입해도 **도입 효과는 상당히 큽니다.**
목표 구조는 다음과 같습니다.
```text
Browser
  │
  │ JSESSIONID Cookie
  ▼
Spring Security FilterChain
  │
  ├─ Authentication
  ├─ Authorization
  ├─ CSRF
  ├─ Session Fixation Protection
  ├─ Login/Logout
  └─ Exception Handling
       │
       ▼
SecurityContext
       │
       ▼
HttpSession
```
즉,
> **Session과 Cookie는 그대로 사용하면서 인증·인가·세션 보안 책임을 Spring Security로 표준화하는 것**
입니다.
현재와 같은 JSP/Tiles 기반 커머스 시스템이라면 저는 **Token 전환보다 이 방식을 우선적으로 권장**합니다.
Spring Security는 인증정보를 `SecurityContextHolder → SecurityContext → Authentication` 모델로 표준화하며, Spring Security 5에서는 기본적으로 이를 HttpSession에 저장하여 다음 요청에서 복원할 수 있습니다. 
---

## 2. 현재 구조와 도입 후 구조 비교
### 현재
```text
Request
   ↓
Custom Filter
   ↓
Interceptor
   ↓
Controller
   ↓
request.getSession()
   ↓
session.getAttribute("loginInfo")
   ↓
로그인 여부 판단
   ↓
권한 판단
```
프로젝트가 커질수록 다음과 같은 코드가 여러 곳에 분산될 가능성이 높습니다.
```java
HttpSession session = request.getSession(false);
if (session == null) {
    ...
}
LoginVO loginVO =
    (LoginVO) session.getAttribute("LOGIN_INFO");
if (loginVO == null) {
    ...
}
if (!"ADMIN".equals(loginVO.getRole())) {
    ...
}
```

### Spring Security 도입 후
```text
Request
   ↓
springSecurityFilterChain
   ↓
SecurityContext 복원
   ↓
Authentication 확인
   ↓
Authorization 확인
   ↓
Controller
```
Controller는 예를 들어:
```java
Authentication authentication =
        SecurityContextHolder.getContext().getAuthentication();
```
또는:
```java
Principal principal
```
또는:
```java
@AuthenticationPrincipal CustomUserPrincipal principal
```
등을 사용할 수 있습니다.
이렇게 되면 인증 판단의 기준이:
```text
session.getAttribute("LOGIN_INFO")
```
에서:
```text
SecurityContext / Authentication
```
- 으로 이동합니다.
---

## 3. 가장 큰 도입 효과
### 인증 처리 표준화
현재 자체 로그인 로직이:
```text
LoginController
LoginService
Session
Filter
Interceptor
Listener
중복로그인 처리
```

등에 분산되어 있다면 

Spring Security 도입 후:
```text
AuthenticationFilter
        ↓
AuthenticationManager
        ↓
AuthenticationProvider
        ↓
UserDetailsService 또는 기존 LoginService
        ↓
Authentication
        ↓
SecurityContext
```
형태로 표준화됩니다.
Spring Security는 `AuthenticationManager`, `ProviderManager`, `AuthenticationProvider`, `SecurityContextHolder`, `GrantedAuthority` 등을 중심으로 인증/권한 모델을 제공합니다. 
### 효과

| 항목               | 현재 자체 구현       | Spring Security       |
| ---------------- | -------------- | --------------------- |
| 로그인 판단           | 직접 구현          | 표준화                   |
| 사용자 Principal    | 자체 Session VO  | `Authentication`      |
| 권한               | if/Interceptor | URL/Method Security   |
| 인증 실패            | 자체 Redirect    | 표준 Handler            |
| 접근 거부            | 자체 처리          | `AccessDeniedHandler` |
| 로그아웃             | 직접 invalidate  | 표준 Logout             |
| Session Fixation | 직접 대응 필요       | 기본 지원                 |
| CSRF             | 직접 구현          | 기본 지원                 |
| 동시 Session       | 직접 구현          | 기본 기능 존재              |

---

## 4. Session Fixation 공격 대응이 크게 개선됨

Spring Security의 Session 기반 인증을 사용하는 중요한 이유 중 하나입니다.
로그인 전:
```text
JSESSIONID=A
```

로그인 성공 후에도:
```text
JSESSIONID=A
```
를 그대로 사용하면 Session Fixation 공격 가능성을 고려해야 합니다.
Spring Security는 로그인 성공 시 Session ID를 변경하는 Session Fixation 방어 기능을 제공합니다.
Servlet 3.1 이상에서는 기본적으로:
```java
request.changeSessionId();
```
기반 `changeSessionId` 전략을 사용할 수 있습니다. 

즉:
```text
로그인 전
JSESSIONID=AAA
        ↓
로그인 성공
        ↓
JSESSIONID=BBB
```
로 변경됩니다.
### 현재 프로젝트에서 특히 확인해야 할 부분
기존 중복로그인 로직이나 Session 관리 코드가:
```java
Map<String, LoginUser> sessions;
```
처럼 **Session ID를 Key로 직접 관리**한다면 매우 중요합니다.

Spring Security 로그인 성공:
```text
Session ID
AAA
 ↓
BBB
```
가 되므로 기존:
```text
LoginSessionManager
HttpSessionIdListener
Session Listener
중복로그인 Manager
```
와 충돌할 수 있습니다.
- 따라서 **Spring Security 도입 전에 반드시 검증해야 할 최우선 항목**입니다.
---

## 5. CSRF 보호 효과

기존 Session/Cookie 기반 커머스 사이트는 CSRF 공격 대상이 될 수 있습니다.
예를 들어 로그인 상태에서 공격자가:

```text
<form action="https://shop/order/cancel.do" method="post">
```

와 같은 요청을 강제로 발생시키는 경우입니다.
Spring Security의 `CsrfFilter`는 Synchronizer Token Pattern을 사용해 상태 변경 요청에 대한 CSRF 방어를 제공합니다. 
### 하지만 도입 시 가장 많이 장애가 발생하는 부분이기도 함

기존:
```text
<form method="post"
      action="/mypage/update.do">
```

가 있었다면 CSRF 활성화 후:

```text
403 Forbidden
```
이 발생할 수 있습니다.
Form에:
```text
<input type="hidden"
       name="${_csrf.parameterName}"
       value="${_csrf.token}">
```
같은 처리가 필요합니다.
AJAX:
```javascript
$.ajax({
    type: "POST",
    url: "/mypage/update.do",
    headers: {
        "X-CSRF-TOKEN": csrfToken
    }
});
```
도 필요합니다.
### 따라서

```java
http.csrf().disable();
```

를 가장 먼저 넣는 방식은 권장하지 않습니다.
- 기존 POST/PUT/DELETE 호출을 조사하고 CSRF 적용 대상을 설계하는 것이 맞습니다.
---

## 6. URL 권한 관리 효과

기존:
```java
if (loginUser == null) {
    return "redirect:/login.do";
}
```
또는:
```java
if (!"ADMIN".equals(role)) {
    ...
}
```
를 Controller마다 작성할 필요가 줄어듭니다.

예:
```java
http
    .authorizeRequests()
    .antMatchers("/login/**").permitAll()
    .antMatchers("/admin/**").hasRole("ADMIN")
    .antMatchers("/mypage/**").authenticated()
    .anyRequest().permitAll();
```
구조적으로:
```text
/admin/**
       ↓
ROLE_ADMIN 필수

/mypage/**
       ↓
로그인 필수

/static/**
       ↓
인증 불필요
```
- 같은 정책을 중앙에서 관리할 수 있습니다.
---

## 7. Method 단위 권한 관리

Service 또는 Controller에서:
```java
@PreAuthorize("hasRole('ADMIN')")
public void updateMember(...) {
}
```
처럼 권한을 적용할 수도 있습니다.

특히 URL만으로 권한을 결정하기 어려운 경우:
```text
URL
/mypage/order.do
```
내에서도:
```text
구매자
판매자
관리자
```
권한이 다르다면 Method Security가 유용합니다.
- 다만 기존 대규모 시스템에서는 처음부터 URL Security와 Method Security를 동시에 대량 도입하지 않는 것을 권장합니다.
---

## 8. 로그인 사용자 접근 방식 개선
기존:
```java
HttpSession session = request.getSession();
LoginVO loginVO =
        (LoginVO) session.getAttribute("LOGIN_INFO");
```
에서:
```java
Authentication authentication =
    SecurityContextHolder
        .getContext()
        .getAuthentication();
```
으로 변경됩니다.
더 권장되는 형태는:
```java
public class LoginUserPrincipal
        implements UserDetails, Serializable {

    private static final long serialVersionUID = 1L;

    private long memberSn;
    private String loginId;
    private Collection<? extends GrantedAuthority> authorities;

    ...
}
```
Controller:
```java
public String myPage(
        @AuthenticationPrincipal LoginUserPrincipal principal) {

    long memberSn = principal.getMemberSn();

    ...
}
```


이렇게 하면:
```text
HttpServletRequest
HttpSession
```
- 에 대한 Application Layer의 직접 의존성이 줄어듭니다.
---

## 9. 기존 Session Attribute를 모두 제거할 필요는 없음
Spring Security 도입했다고:
```java
session.setAttribute(...)
```
를 전부 없앨 필요는 없습니다.
다음과 같이 분리하면 됩니다.

| Session 정보  | 처리                   |
| ----------- | -------------------- |
| 로그인 여부      | Spring Security      |
| 사용자 ID      | Principal            |
| 사용자 권한      | `GrantedAuthority`   |
| 인증 상태       | `SecurityContext`    |
| 업무상 임시정보    | 기존 Session 사용 가능     |
| Wizard 진행정보 | 기존 Session 가능        |
| 화면 임시조건     | 필요하면 Session         |
| 장바구니        | 현재 설계에 따라 Session/DB |

즉:
```text
인증관련 Session
→ Spring Security로 이전

업무관련 Session
→ 필요하면 유지
```
- 가 현실적입니다.
---

## 10. 비밀번호 관리 개선
Spring Security의 `PasswordEncoder`를 사용할 수 있습니다.
5.8 계열은:
```text
BCrypt
PBKDF2
SCrypt
Argon2
DelegatingPasswordEncoder
```
등을 지원하고, `DelegatingPasswordEncoder`는 기존 암호 방식과 신규 암호 방식을 단계적으로 공존시키는 데 유용합니다. 

예:
```text
기존
SHA-256(password)
```
를 운영 중이라고 해서 일괄 비밀번호 초기화가 반드시 필요한 것은 아닙니다.

예:
```text
기존 사용자
→ 기존 Hash 검증

로그인 성공
→ BCrypt 재암호화

신규 사용자
→ BCrypt
```
같은 단계적 Migration이 가능합니다.
- 단, **현재 DB 비밀번호 암호화 방식을 먼저 조사해야 합니다.**
---

## 11. 중복 로그인 제어
Spring Security 자체에도 동시 Session 제어 기능이 있습니다.

예:
```java
.sessionManagement()
.maximumSessions(1)
```
개념적으로:
```text
User A
   ├─ Session A
   └─ Session B
```

에서:
```text
최대 1 Session
```
으로 제한할 수 있습니다.
Spring Security는 `SessionRegistry`, `ConcurrentSessionControlAuthenticationStrategy`, `ConcurrentSessionFilter` 등을 제공합니다. 
하지만 현재 시스템에서는 **이 기능을 바로 사용하는 것을 권장하지 않습니다.**
- 이유가 중요합니다.
---

## 12. 2-WAS 환경에서 Spring Security 기본 SessionRegistry의 한계
Spring Security 기본:
```java
SessionRegistryImpl
```
은 내부적으로 Session과 Principal 정보를 Map 형태로 관리합니다. 공식 API에도 `ConcurrentMap<Object, Set<String>>`과 `Map<String, SessionInformation>`을 사용하는 구조가 명시되어 있습니다. 
따라서:
```text
WAS 1
SessionRegistry A

WAS 2
SessionRegistry B
```
가 각각 존재하는 다중 WAS 환경에서는 기본 `SessionRegistryImpl`만으로 전역 중복로그인을 판단하는 것은 적합하지 않습니다.

예:
```text
User A
  ↓
WAS #1 로그인
Session A

User A
  ↓
WAS #2 로그인
Session B
```

각 WAS:
```text
WAS #1 → Session A만 알고 있음
WAS #2 → Session B만 알고 있음
```
가능성이 있습니다.
### 따라서 현재 시스템에서는
기존에 구축한:
```text
LoginSessionManager
LoginPreventer
DuplicateLoginSessionFilter
Session Listener
```
등의 중복로그인 구조를 **Spring Security SessionRegistry로 즉시 교체하지 않는 것**이 안전합니다.
먼저:
```text
Spring Security 인증
+
기존 중복로그인 제어
```
를 통합한 뒤 별도로 구조 개선을 판단하는 것이 좋습니다.
- Redis는 이 단계에서 필수가 아닙니다.
---

## 13. `HttpSessionEventPublisher` 충돌 확인
Spring Security의 Concurrent Session 제어를 사용하면 Session 생성/소멸 이벤트를 받기 위해:
```java
HttpSessionEventPublisher
```
를 등록할 수 있습니다. 공식 문서도 SessionRegistry가 Session 종료를 정상적으로 인지하려면 이 Listener가 중요하다고 설명합니다. 
현재 이미:
```text
HttpSessionListener
HttpSessionIdListener
자체 Session Listener
```
가 존재한다면 다음 순서를 반드시 확인해야 합니다.
```text
Spring Security Listener
       +
기존 Listener
       ↓
중복 이벤트 처리?
       ↓
Session Map 중복 삭제?
       ↓
중복로그인 상태 오류?
```
---

## 14. AJAX 처리는 반드시 별도 설계
현재 시스템에서 특히 중요합니다.
기존:
```text
AJAX
 ↓
Session 만료
 ↓
Filter
 ↓
sendRedirect("/login.do")
```
방식은 이미 문제가 될 수 있습니다.

Spring Security 기본 인증 실패 역시 브라우저 페이지 요청에는 Redirect가 적절하지만 AJAX에서는:
```text
302
 ↓
login.jsp HTML
 ↓
AJAX success/error 처리 혼란
```
이 발생할 수 있습니다.

따라서 요청 유형별:
```text
일반 Page Request
→ 302 / login.do

AJAX / JSON Request
→ 401 JSON

권한 없음
→ 403 JSON
```
을 구분해야 합니다.

예:
```text
AuthenticationEntryPoint
AccessDeniedHandler
AuthenticationSuccessHandler
AuthenticationFailureHandler
LogoutSuccessHandler
```
- 를 프로젝트 규칙에 맞게 구현하는 것이 좋습니다.
---

## 15. Filter와 Interceptor 순서가 매우 중요
Spring Security는 기본적으로:
```text
Servlet Container
       ↓
Spring Security Filter Chain
       ↓
DispatcherServlet
       ↓
Spring MVC Interceptor
       ↓
Controller
```
구조입니다.

따라서 기존:
```text
DuplicateLoginSessionFilter
CustomAuthFilter
WebContentInterceptor
LoginInterceptor
```
등과 순서를 명확히 해야 합니다.

권장 책임 분리는:

| 기능               | 담당                     |
| ---------------- | ---------------------- |
| 인증 여부            | Spring Security        |
| 권한               | Spring Security        |
| CSRF             | Spring Security        |
| Session fixation | Spring Security        |
| 로그인 실패           | Spring Security        |
| 업무 공통처리          | Interceptor            |
| 중복로그인            | 기존 로직 → 향후 통합          |
| Encoding/CORS 등  | Filter/Security 정책에 따라 |

특히 다음 같은 중복 코드는 제거 대상입니다.
```text
Spring Security
        ↓
authenticated
        ↓
Interceptor
        ↓
또 loginSession 존재 검사
```
- 두 시스템이 로그인 여부를 각각 판단하면 장애 원인이 됩니다.
---

## 16. Session 생성 정책 확인
Spring Security를 붙이면:
```text
언제 Session이 생성되는가?
```
도 검증해야 합니다.
Spring Security 5에서는 인증된 `SecurityContext`를 HttpSession에 저장하여 인증 상태를 유지할 수 있습니다. 
현재 코드에:
```java
request.getSession();
```
이 많다면 Spring Security와 무관하게 Session이 불필요하게 생성될 수 있습니다.
따라서:
```java
request.getSession(false);
```
로 변경할 수 있는 곳은 변경하는 것이 좋습니다.
특히:
```text
WebContentInterceptor
공통 Filter
로그 Filter
메뉴 Filter
```
- 가 로그인 전부터 Session을 생성하고 있는지 확인해야 합니다.
---

## 17. WAS Session Clustering 영향
현재처럼 Session clustering을 사용한다면 Spring Security 도입 후 Session에:
```text
SPRING_SECURITY_CONTEXT
        ↓
Authentication
        ↓
Principal
```
이 추가됩니다.
따라서 Custom Principal은 가능한 한:
```java
implements Serializable
```
로 만드는 것이 안전합니다.

그리고 Principal 내부에 다음을 넣지 않는 것이 좋습니다.
```java
@Service
DAO
Connection
HttpServletRequest
HttpSession
대용량 VO Graph
```

권장:
```java
public class LoginUserPrincipal
        implements UserDetails, Serializable {

    private static final long serialVersionUID = 1L;

    private long memberSn;
    private String loginId;
    private List<GrantedAuthority> authorities;

}
```
- 클러스터 Session replication 환경에서는 **Principal 크기가 커질수록 Session replication 비용도 증가**합니다.
---

## 18. Session에 회원 VO 전체를 넣는 구조도 개선 기회
현재:
```java
session.setAttribute("loginVO", hugeMemberVO);
```
처럼 많은 정보가 들어간다면 Spring Security 도입 시:
```text
최소 Principal
```
로 변경할 것을 권장합니다.
예:
```text
memberSn
loginId
role
memberType
```
정도만 Authentication에 유지하고:
```text
주소
전화번호
회사 상세정보
상품정보
설정정보
```
등은 필요 시 조회합니다.
- 특히 WAS 2대 Session replication에서는 효과가 큽니다.
---

## 19. Logout 처리 확인
현재:
```java
session.invalidate();
```
만 수행하고 있다면 Spring Security:
```text
/logout
 ↓
SecurityContext 제거
 ↓
Session invalidate
 ↓
Cookie 처리
 ↓
LogoutSuccessHandler
```
로 통합할 수 있습니다.

단 기존:
```text
로그인 사용자 Map 제거
중복로그인 Manager 제거
로그아웃 이력 DB 기록
SSO Logout
외부 시스템 Logout
```
- 이 있다면 `LogoutHandler`에 통합하거나 기존 Service를 연결해야 합니다.
---

## 20. Cookie 보안은 Spring Security 도입만으로 끝나지 않음
Spring Security를 도입했다고:
```text
JSESSIONID
Secure
HttpOnly
SameSite
```
설정이 모두 자동으로 최적화된다고 생각하면 안 됩니다.
계속 확인해야 합니다.
```text
HTTPS Only
Secure=true
HttpOnly=true
SameSite=Lax 또는 정책에 맞는 값
Session Timeout
Cookie Path
Cookie Domain
URL Rewriting 금지
```
- Nginx/WAS 설정과 같이 확인해야 합니다.
---

## 21. Spring Security가 해결하지 않는 것
중요합니다.
Spring Security를 도입했다고 다음이 자동 해결되지는 않습니다.

| 보안 문제            |    자동 해결 |
| ---------------- | -------: |
| 인증               |        O |
| 인가               |        O |
| Session Fixation |        O |
| CSRF             |        O |
| 비밀번호 Hash API    |        O |
| XSS              |        X |
| SQL Injection    |        X |
| 파일 업로드 취약점       |        X |
| IDOR             |      부분적 |
| 업무 권한 오류         |    설계 필요 |
| Cookie 정책 전체     |    별도 설정 |
| DB 암호화           |        X |
| 개인정보 Masking     |        X |
| 중복로그인 다중 WAS     | 별도 설계 필요 |

---

## 22. 장점/단점 최종 비교

| 항목               | 장점             | 단점/주의                      |
| ---------------- | -------------- | -------------------------- |
| 인증               | 표준 구조          | 기존 로그인 대규모 수정              |
| 권한               | 중앙 관리          | 권한 Matrix 설계 필요            |
| Session          | 보안 기능 강화       | 기존 Session Manager 충돌      |
| Session Fixation | 기본 보호          | Session ID 기반 로직 점검        |
| CSRF             | 강력한 보호         | 기존 POST/AJAX 수정            |
| AJAX             | Handler 표준화 가능 | 기본 Redirect 그대로 쓰면 문제      |
| 비밀번호             | Encoder 표준     | 기존 Hash Migration 필요       |
| Logout           | 표준화            | 기존 로그아웃 부가기능 통합            |
| 중복로그인            | 기본 기능 있음       | Multi-WAS는 별도 고려           |
| 유지보수             | 크게 개선          | 초기 Migration 비용            |
| Redis            | 불필요            | 분산 SessionRegistry 필요 시 검토 |
| Token            | 불필요            | Stateless 장점은 없음           |

---

## 23. Spring 5.3에서 사용할 Spring Security 버전
여기에는 2026년 현재 중요한 문제가 있습니다.
Spring Security 5.x를 유지해야 한다면 **5.8.x가 사실상 최종 5.x 계열**이고, Spring 측도 5.x에 머무는 경우 5.8로 업데이트하는 것을 권장했습니다. 

Spring Security 6은:
```text
JDK 17+
jakarta.servlet.*
Spring Framework 6
```
전환을 전제로 하기 때문에 현재 Spring 5.3/JDK 11 기반 시스템에 그대로 도입할 대상이 아닙니다. 

그러나 중요한 점:
> **Spring Framework 5.3.x와 Spring Security 5.8.x의 Open Source Support는 2024-08-31 종료되었습니다.** 
2026년에도 상용 Spring Enterprise에서는 Spring Security 5.8 계열의 보안 패치가 계속 제공되고 있으며, 예를 들어 2026년 상용 5.8.x 릴리스도 존재합니다. 
따라서 신규 도입 시:
```text
Spring Security 5.8.x
```
라고만 결정하면 안 되고:
```text
① Spring Enterprise 상용 패치 사용
또는
② 사내 승인된 5.8 OSS artifact + 취약점 대응체계
또는
③ 중장기 Spring 6/JDK17 Migration 계획
```

까지 같이 결정해야 합니다.
- 이 부분은 보안 Framework 신규 도입 심사에서 매우 중요합니다.
---

## 24. 신규 Infrastructure는 필요한가?
이번 조건:
```text
Token 사용 안 함
HttpSession 유지
JSESSIONID 유지
```
라면:
### Redis 필요 없음
```text
Browser
 ↓
JSESSIONID
 ↓
WAS Session
 ↓
Spring SecurityContext
```
만으로 가능합니다.

추가로 필요한 것은 Library와 Configuration입니다.
```text
spring-security-core
spring-security-web
spring-security-config
```
등입니다.

따라서:
```text
Redis
OAuth Server
JWT Key Server
Token Store
```
- 등의 신규 Infrastructure는 **Spring Security 도입 자체에는 전혀 필요하지 않습니다.**
---

## 25. 현재 시스템에서 가장 먼저 확인해야 할 항목
우선순위를 매기면 다음과 같습니다.

| 우선순위 | 점검사항                                    |   중요도 |
| ---: | --------------------------------------- | ----: |
|    1 | 현재 로그인 성공 시 Session 생성/Attribute        | ★★★★★ |
|    2 | `getSession()` / `getAttribute()` 전체 조사 | ★★★★★ |
|    3 | 기존 인증 Filter/Interceptor                | ★★★★★ |
|    4 | 중복로그인 관련 Filter/Listener/Manager        | ★★★★★ |
|    5 | Session ID 직접 저장/비교 코드                  | ★★★★★ |
|    6 | AJAX 인증 만료 처리                           | ★★★★★ |
|    7 | POST/AJAX CSRF 영향                       | ★★★★★ |
|    8 | WAS Session clustering                  | ★★★★★ |
|    9 | Principal Serializable 여부               | ★★★★☆ |
|   10 | 로그인/로그아웃 URL                            | ★★★★☆ |
|   11 | 권한 Role 구조                              | ★★★★☆ |
|   12 | 비밀번호 Hash 방식                            | ★★★★☆ |
|   13 | Cookie Secure/HttpOnly/SameSite         | ★★★★☆ |
|   14 | Static resource 제외 정책                   | ★★★☆☆ |
|   15 | Error/Exception page                    | ★★★☆☆ |

---

## 26. 실제 도입 절차 권장안
저라면 현재 프로젝트에서는 **Big Bang 방식으로 전환하지 않습니다.**
### Phase 1 — 현황 조사
다음 문자열을 프로젝트 전체 검색합니다.
```text
getSession(
getAttribute(
setAttribute(
invalidate(
getRequestedSessionId
isRequestedSessionIdValid
JSESSIONID
HttpSessionListener
HttpSessionIdListener
Filter
HandlerInterceptor
login
logout
role
auth
```

그리고:
```text
인증
권한
업무 Session
중복로그인
기타
```
로 분류합니다.

### Phase 2 — Spring Security Framework만 도입
```text
SecurityFilterChain
AuthenticationManager
AuthenticationProvider
UserDetailsService 또는 기존 Service Adapter
CustomUserPrincipal
```
를 구성합니다.
아직:
```text
기존 Login Session
기존 중복로그인
```
을 대규모 제거하지 않습니다.

### Phase 3 — 인증 기준을 SecurityContext로 변경
기존:
```java
session.getAttribute("LOGIN_INFO")
```
를 점진적으로:
```java
SecurityContextHolder.getContext()
        .getAuthentication()
```
으로 전환합니다.

### Phase 4 — URL 권한 중앙화
```text
/mypage/**
/seller/**
/admin/**
/none/**
```
등을 Role Matrix로 작성하고 Security Configuration으로 이전합니다.

### Phase 5 — CSRF 적용
```text
JSP Form
AJAX
File Upload
Payment
External Callback
```
을 각각 분리합니다.
특히 결제/PG Callback URL을 일반 CSRF 정책에 무조건 넣으면 외부 연계가 실패할 수 있으므로 정확한 예외 정책이 필요합니다.

### Phase 6 — Login/Logout Handler 이전
```text
AuthenticationSuccessHandler
AuthenticationFailureHandler
AuthenticationEntryPoint
AccessDeniedHandler
LogoutHandler
LogoutSuccessHandler
```
로 표준화합니다.

### Phase 7 — 기존 인증 Filter/Interceptor 제거
안정화된 후:
```text
LoginCheckInterceptor
SessionCheckFilter
AuthFilter
```
처럼 Spring Security와 중복되는 부분을 제거합니다.

### Phase 8 — 중복로그인 구조 별도 통합
마지막으로:
```text
기존 Duplicate Login
        VS
Spring Security Concurrent Session
```
- 을 비교하고 하나의 구조로 통합합니다.
---

## 27. 권장 Target Architecture
현재 시스템이라면 최종적으로 다음 구조를 권장합니다.
```text
                    Browser
                       │
                       │ JSESSIONID
                       ▼
                     Nginx
                       │
                       ▼
             Spring Security FilterChain
                       │
           ┌───────────┼────────────┐
           │           │            │
     Authentication   CSRF     Authorization
           │                        │
           ▼                        │
     SecurityContext                │
           │                        │
           ▼                        │
       HttpSession                  │
           │                        │
      ┌────┴────┐                   │
      │         │                   │
   WAS #1     WAS #2                │
      │         │                   │
      └────┬────┘                   │
           │                        │
     Session Cluster                │
           │                        │
           └──────────────┬─────────┘
                          ▼
                     Controller
                          │
                          ▼
                       Service
                          │
                          ▼
                       MariaDB
```
여기에는:
```text
JWT X
Access Token X
Refresh Token X
Redis X
LocalStorage X
```
- 입니다.
---

## 28. 현재 프로젝트 기준 최종 평가

| 평가항목                       | 판단          |
| -------------------------- | ----------- |
| Spring Security 도입 필요성     | **높음**      |
| Token 전환 필요성               | 현재 단계에서는 낮음 |
| 기존 Session 유지              | **가능 / 권장** |
| 기존 JSESSIONID 유지           | **가능**      |
| Redis 추가                   | 불필요         |
| 보안 개선 효과                   | **높음**      |
| 개발 구조 개선                   | **높음**      |
| 초기 영향 범위                   | 중~높음        |
| JSP/Tiles 호환성              | 좋음          |
| AJAX 수정 가능성                | 높음          |
| 중복로그인 영향                   | **매우 높음**   |
| Session Cluster 영향         | 검증 필수       |
| Spring Security 5.8 OSS 지원 | **종료**      |
| 장기 Upgrade 계획              | 필요          |

## 최종적으로 권장하는 방향

현재 26_KTR 구조에서는:
```text
Token 도입
      X

Redis 도입
      X

Spring Security
      O

HttpSession
      O

JSESSIONID Cookie
      O
```
로 시작하는 것이 가장 합리적입니다.

특히 도입 목적을: `Session을 없애기 위한 것` 이 아니라 **`현재 Filter/Interceptor/Controller에 분산된 인증·인가·Session 보안 로직을 Spring Security라는 하나의 표준 보안 계층으로 통합하는 것`** 으로 잡는 것이 맞습니다.
그리고 현재 시스템에서는 

- ① Session ID 변경과 기존 중복로그인 로직의 충돌
- ② AJAX의 302/401/403 처리
- ③ CSRF 적용에 따른 기존 POST 호출 장애
- ④ 2 WAS Session clustering에서 CustomPrincipal serialization
- ⑤ Spring Security 5.8의 보안 패치 정책 

이 다섯 항목이 실제 도입 성패를 좌우할 핵심 점검사항입니다.