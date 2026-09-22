## 결론

Nexacro + Java/JBoss 환경에서 `JSESSIONID`에 `HttpOnly=true`, `Secure=true`를 적용하는 것은 **원칙적으로 권장되며, 일반적인 Nexacro `transaction()` + 서버 `HttpSession` 구조라면 정상 동작해야 합니다.**

다만 장애 가능성이 있는 지점은 명확합니다.

| 구분                | 정상적인 영향                  | 문제가 발생하는 조건                                                   |   위험도 |
| ----------------- | ------------------------ | ------------------------------------------------------------- | ----: |
| `HttpOnly`        | JS에서 JSESSIONID 조회 불가    | `document.cookie`, `getCookieVariable()` 등으로 세션ID를 직접 사용하는 경우 |    높음 |
| `Secure`          | HTTPS 요청에만 JSESSIONID 전송 | HTTP URL이 한 곳이라도 남아 있는 경우                                     | 매우 높음 |
| SSV               | 영향 없음                    | 세션 소실 시 서버가 HTML/302를 반환하는 경우                                 |    높음 |
| Cross Domain      | 별도 조건 필요                 | `withCredentials`, CORS, SameSite 설정 불일치                      | 매우 높음 |
| LB SSL Offloading | 가능                       | 외부 HTTPS→LB→WAS HTTP 구조에서 scheme 인식 오류                        |  중~높음 |
| Nexacro WRE       | 브라우저 쿠키 자동전송은 가능         | Nexacro Cookie API로 Secure/HttpOnly 쿠키를 직접 관리하는 경우            |    높음 |
| NRE               | 버전/런타임별 검증 필요            | 자체 Cookie 저장소에 의존하는 경우                                        |  중~높음 |

특히 **SSV라고 해서 HttpOnly/Secure 적용 방법이 달라지는 것은 아닙니다.** SSV는 HTTP Body의 데이터 직렬화 형식이고, `JSESSIONID`는 HTTP `Cookie` 헤더로 전달되므로 서로 다른 계층입니다. Nexacro 공식 문서에서도 SSV는 Dataset 전송 형식이며 `HttpPlatformRequest/Response`가 HTTP 위에서 이를 처리하는 구조로 설명합니다. ([Tobesoft][1])

---

# 1. 먼저 가장 중요한 구조

정상적인 Nexacro 세션 통신은 다음과 같습니다.

```text
[Nexacro]
    │
    │ transaction()
    │ HTTPS
    │
    │ Cookie: JSESSIONID=ABC...
    │
    │ Body:
    │ SSV:utf-8
    │ Dataset...
    ▼
[WEB/LB]
    ▼
[JBoss / Servlet]
    │
    └─ request.getSession()
          ↓
       JSESSIONID 확인
          ↓
       기존 HttpSession 조회
```

여기서 중요한 점은:

```text
Cookie Header
JSESSIONID=...
```

와

```text
HTTP Body
SSV:utf-8 ...
```

가 **완전히 별도**라는 것입니다.

따라서:

> **SSV → HttpOnly/Secure 적용 불가 또는 불필요**

가 아니라,

> **SSV를 사용하더라도 JSESSIONID에는 HttpOnly/Secure를 정상적으로 적용해야 한다**

가 맞습니다.

---

# 2. HttpOnly 적용 시 실제로 무엇이 달라지는가

예를 들어 JBoss가 다음 응답을 내려준다고 가정합니다.

```http
Set-Cookie: JSESSIONID=ABCDE12345; Path=/; HttpOnly; Secure
```

브라우저/WRE에서는 이후 요청에 자동으로:

```http
Cookie: JSESSIONID=ABCDE12345
```

를 붙입니다.

하지만 JavaScript에서는:

```javascript
document.cookie
```

를 호출해도 `JSESSIONID`가 보이지 않습니다.

OWASP와 Servlet 명세 모두 HttpOnly의 목적을 **JavaScript 등 비-HTTP API에서 세션 쿠키에 접근하지 못하게 하는 것**으로 설명하고 있습니다. 쿠키 자체의 HTTP 요청 전송을 막는 기능은 아닙니다. ([OWASP Cheat Sheet Series][2])

따라서 다음 구조는 문제없습니다.

```javascript
this.transaction(
    "search",
    "Svc::search.do",
    "",
    "dsResult=dsResult",
    "",
    "fnCallback"
);
```

Nexacro 코드가 `JSESSIONID`를 읽을 필요가 없습니다.

브라우저 또는 Nexacro 통신 계층이 쿠키를 HTTP 요청에 자동 첨부하고 서버에서는:

```java
HttpSession session = request.getSession();
```

으로 사용합니다.

### 반대로 문제가 되는 코드

```javascript
var cookies = document.cookie;
```

또는

```javascript
var sessionId = nexacro.getCookieVariable("JSESSIONID");
```

또는

```javascript
var jsessionid = ...
transaction(... + "?JSESSIONID=" + jsessionid);
```

처럼 **클라이언트 코드가 세션ID를 직접 읽고 사용하는 구조라면 HttpOnly 적용 후 문제가 발생할 수 있습니다.**

---

# 3. `request.getSession()`에는 문제가 없는가?

문제없습니다.

이 부분은 JavaScript와 관계가 없습니다.

```text
HTTPS Request
      │
      ├─ Cookie: JSESSIONID=ABC
      │
      ▼
JBoss Undertow
      │
      ├─ ABC에 해당하는 Session 검색
      │
      ▼
request.getSession()
```

즉 HttpOnly가 차단하는 것은:

```text
JavaScript
    ↓
document.cookie
```

이지,

```text
Browser/Nexacro HTTP Engine
    ↓
HTTP Cookie Header
    ↓
JBoss
```

가 아닙니다.

따라서 기존 시스템이 정상적인 Servlet Session 방식이라면 **HttpOnly 때문에 `request.getSession()`이 동작하지 않는 것은 아닙니다.**

다만 `Secure` 때문에 쿠키 자체가 서버로 오지 않으면 상황이 달라집니다.

예를 들어:

```text
로그인
https://server/login.do

Set-Cookie:
JSESSIONID=ABC; Secure
```

이후:

```text
http://server/select.do
```

로 호출하면 `Secure` 쿠키는 전송되지 않습니다.

그러면:

```java
request.getSession()
```

은 기존 세션 ABC가 아니라 **새 세션을 생성할 수 있습니다.**

인증 필터에서는 이런 이유로 가능하면:

```java
HttpSession session = request.getSession(false);
```

로 기존 세션 존재 여부부터 확인하는 방식이 안전합니다.

---

# 4. Secure가 Nexacro에서 더 주의가 필요한 이유

Nexacro 공식 문서는 Secure 쿠키에 대해 명시적으로 다음 동작을 설명합니다.

```text
HTTP
 → Secure cookie 전송 안 함

HTTPS
 → Secure cookie 전송
```

Nexacro 17 이후 Secure Cookie 지원도 공식적으로 제공되고 있습니다. ([Tobesoft][3])

따라서 운영 서비스가 HTTPS라고 하더라도 다음을 모두 확인해야 합니다.

```text
Main Page
https://xxx

Login
https://xxx

Nexacro transaction
https://xxx

File Upload
https://xxx

File Download
https://xxx

Excel Import/Export
https://xxx

Popup URL
https://xxx

Image/외부 Service
https://xxx
```

가장 위험한 형태는:

```text
화면:
https://xxx

Service Prefix:
http://xxx/service/
```

입니다.

사용자는 HTTPS로 접속했지만 실제 `transaction()` 서비스 URL이 HTTP라면 Secure JSESSIONID가 빠질 수 있습니다.

---

# 5. Nexacro에서 반드시 검색해야 할 코드

적용 전에 소스 전체 검색을 권장합니다.

| 검색 문자열                 | 확인 목적                         |
| ---------------------- | ----------------------------- |
| `document.cookie`      | JSESSIONID 직접 접근 여부           |
| `getCookieVariable`    | Nexacro Cookie API 사용 여부      |
| `setCookieVariable`    | 쿠키 직접 생성 여부                   |
| `removeCookieVariable` | 로그아웃 등 쿠키 직접 제거 여부            |
| `JSESSIONID`           | 세션ID 직접 처리 여부                 |
| `jsessionid`           | URL rewriting 사용 여부           |
| `http://`              | Secure 적용 시 HTTP 호출 존재 여부     |
| `transaction(`         | 모든 서비스 호출 URL 확인              |
| `addcookietovariable`  | Nexacro Cookie→Variable 연계 여부 |
| `networksecurelevel`   | Cross Domain 설정 확인            |

특히 다음이 발견되면 **HttpOnly 적용 전 코드 분석이 필요합니다.**

```javascript
nexacro.getCookieVariable("JSESSIONID")
```

또는

```javascript
document.cookie.indexOf("JSESSIONID")
```

---

# 6. Nexacro 자체 Cookie와 JSESSIONID를 구분해야 한다

여기서 실무적으로 매우 중요합니다.

Nexacro에는 자체 Cookie API가 있습니다.

```javascript
nexacro.setCookieVariable(...)
nexacro.getCookieVariable(...)
nexacro.removeCookieVariable(...)
```

Nexacro 공식 문서에서도 Secure cookie 기능을 제공합니다. ([Tobesoft][4])

하지만 서버 세션용:

```text
JSESSIONID
```

은 가급적 Nexacro 애플리케이션이 직접 관리해서는 안 됩니다.

좋은 구조:

```text
JSESSIONID
    ↓
Browser / Nexacro HTTP stack 관리
    ↓
JBoss Session 관리
```

피해야 할 구조:

```text
JSESSIONID
    ↓
Nexacro getCookieVariable()
    ↓
JavaScript 변수 저장
    ↓
transaction()에 수동 추가
```

후자는 HttpOnly의 보안 목적 자체와 충돌합니다.

---

# 7. WRE에서는 추가 확인이 필요하다

Nexacro 공식 API 문서에는 WRE에서 다음 제약이 명시되어 있습니다.

> 서버에서 Secure 속성을 가진 Cookie를 수신한 경우 Nexacro Environment의 Cookies 영역에 추가되거나 변경되지 않는 제약이 존재합니다. ([Tobesoft][4])

따라서 중요한 구분이 생깁니다.

```text
브라우저의 HTTP Cookie Store
             ≠
Nexacro Environment Cookies
```

즉 브라우저가 서버의 Secure JSESSIONID를 정상 저장하고 자동 전송하는 것과,

```javascript
nexacro.getCookieVariable(...)
```

로 그 값을 가져오는 것은 동일한 동작이라고 가정하면 안 됩니다.

따라서 WRE에서는 반드시 두 가지를 따로 테스트해야 합니다.

```text
1. transaction() 요청에 JSESSIONID가 자동으로 들어가는가?
2. Nexacro getCookieVariable()에서 JSESSIONID를 읽으려고 하는 코드가 있는가?
```

1은 정상이어야 하고, 2에는 의존하지 않는 것이 바람직합니다.

---

# 8. NRE는 별도로 검증하는 것이 안전하다

여기서는 WRE보다 주의를 권합니다.

브라우저 WRE의 HttpOnly 동작은 웹 표준이 명확하지만, Nexacro NRE는 네이티브 통신 계층을 이용하기 때문에 **현재 운영 중인 정확한 Nexacro 버전에서 서버가 설정한 HttpOnly 쿠키의 저장·재전송 동작을 테스트하는 것이 좋습니다.**

이번에 확인한 Nexacro 공식 문서에서는 Secure Cookie 동작은 명확하게 기술되어 있지만, `HttpOnly`와 NRE Cookie API 상호작용에 대해서는 Secure만큼 명확한 설명을 찾기 어렵습니다.

따라서 NRE 운영 시스템이면 추측하지 말고 테스트 케이스로 검증하는 것이 안전합니다.

---

# 9. SSV 사용 시 가장 중요한 실제 장애 포인트

SSV 자체는 HttpOnly/Secure와 충돌하지 않습니다.

문제는 **세션이 없어졌을 때 서버가 무엇을 반환하느냐**입니다.

예를 들어 Secure 설정 문제 때문에 JSESSIONID가 전송되지 않았다고 가정합니다.

```text
Nexacro
   │
   │ SSV transaction
   ▼
Session Filter
   │
   └─ Session 없음
           ↓
       HTTP 302
           ↓
       login.html
```

Nexacro는 원래 다음을 기대합니다.

```text
SSV:utf-8
ErrorCode=...
ErrorMsg=...
Dataset...
```

그런데 실제로:

```html
<html>
...
login page
...
</html>
```

을 받으면 **SSV Parser 오류 또는 transaction 오류**가 발생할 수 있습니다.

이것이 실무에서는 Secure 적용 후 나타나는 문제를 "Nexacro 통신 오류"로 오해하게 만드는 대표적인 형태입니다.

---

# 10. SSV 세션 만료는 SSV 형식으로 처리하는 것이 좋다

Nexacro X-API는 `PlatformData`의 `ErrorCode`, `ErrorMsg`를 이용해 클라이언트에 오류를 전달하는 방식을 공식적으로 지원합니다. ([Tobesoft][5])

예를 들어 개념적으로:

```java
PlatformData data = new PlatformData();

VariableList variables = data.getVariableList();

variables.add("ErrorCode", -401);
variables.add("ErrorMsg", "SESSION_EXPIRED");
```

후 Nexacro가 사용하는 SSV 형식으로 반환하는 방식입니다.

예:

```java
HttpPlatformResponse platformResponse =
        new HttpPlatformResponse(
                response,
                PlatformType.CONTENT_TYPE_SSV,
                "UTF-8");

platformResponse.setData(data);
platformResponse.sendData();
```

Nexacro 공식 예제에서도 `PlatformType.CONTENT_TYPE_SSV`로 `PlatformData`를 반환하는 구조를 사용합니다. ([Tobesoft][6])

따라서 SSV 서비스에서는 가능하면:

```text
세션 만료
   ↓
302 → HTML 로그인
```

보다

```text
세션 만료
   ↓
Nexacro SSV ErrorCode
   ↓
transaction callback
   ↓
공통 로그인 처리
```

가 더 안정적입니다.

---

# 11. HTTP Status도 별도로 고려

또 다른 방법은:

```http
HTTP/1.1 401 Unauthorized
```

또는

```http
HTTP/1.1 403 Forbidden
```

을 반환하는 것입니다.

Nexacro에서는 Application `onerror`를 통해 HTTP status code를 확인할 수 있습니다. 공식 기술문서에서도 `ErrorEventInfo.statuscode`를 이용한 HTTP 오류 확인 방법을 안내합니다. ([Tobesoft][7])

따라서 시스템 전체 정책을 다음 중 하나로 통일하는 것이 좋습니다.

```text
A안
HTTP 200
+ SSV ErrorCode = SESSION_EXPIRED

B안
HTTP 401
+ Application.onerror 공통처리

C안
HTTP 401
+ 가능하면 SSV 오류정보도 함께 제공
```

기존 Nexacro 애플리케이션의 callback 구조가 이미 `ErrorCode` 기반이라면 **A 또는 기존 정책과 호환되는 방식**이 수정 범위가 가장 작습니다.

---

# 12. Cross Domain이라면 난이도가 올라간다

예:

```text
Nexacro UI
https://ui.example.com

API
https://api.example.com
```

이면 단순 Secure/HttpOnly만으로 끝나지 않습니다.

Nexacro 공식 문서에서는 Cross Domain WRE에서:

```text
Environment.networksecurelevel = all
```

인 경우 `withCredentials=true`를 이용하며,

서버에서도:

```http
Access-Control-Allow-Credentials: true
```

가 필요합니다.

또한 Credentials 요청에서는:

```http
Access-Control-Allow-Origin: *
```

을 사용할 수 없고 실제 Origin을 지정해야 합니다. ([Tobesoft][8])

예:

```http
Access-Control-Allow-Origin: https://ui.example.com
Access-Control-Allow-Credentials: true
```

여기에 최신 브라우저 환경에서는 `SameSite` 정책까지 영향을 줄 수 있으므로 Cross-Site 구조라면 다음도 함께 검토해야 합니다.

```text
JSESSIONID
Secure
HttpOnly
SameSite
Domain
Path
CORS
withCredentials
```

---

# 13. SSV 자체는 암호화가 아니다

이 부분도 중요합니다.

SSV는:

```text
SSV = 데이터 직렬화 Format
```

입니다.

HTTPS는:

```text
HTTPS = HTTP 전체 통신구간 TLS 암호화
```

입니다.

따라서:

```text
SSV 사용
→ Secure 필요 없음
```

은 잘못된 판단입니다.

Nexacro 공식 자료에서도 SSV는 XML보다 데이터 크기가 작고 parsing 효율이 좋은 통신 형식으로 설명하며, SSV/XML 데이터 변조 등에 대해서는 별도의 암호화 수단이 필요하다고 설명합니다. ([Tobesoft][9])

특히 HTTPS를 사용하면:

```text
HTTP Header
 - Cookie
 - Authorization
 - 기타 Header

HTTP Body
 - SSV Dataset
```

전체가 TLS로 보호됩니다.

따라서 SSV 내부 데이터를 별도 암호화하는 기능이 있더라도 **Secure Cookie를 대체하지 못합니다.**

---

# 14. LB에서 SSL 종료하는 구조도 확인

예를 들어 운영 구조가:

```text
Nexacro
   │ HTTPS
   ▼
LB
   │ HTTP
   ▼
WEB
   │ HTTP
   ▼
JBoss
```

일 수도 있습니다.

이 구조에서도 클라이언트 기준으로는 HTTPS이므로:

```text
JSESSIONID; Secure
```

를 사용하는 것이 가능합니다.

Servlet `SessionCookieConfig` 사양도 SSL offloading LB 앞단에서 외부 HTTPS / 내부 HTTP인 경우에도 Secure session cookie를 명시적으로 설정할 수 있는 구조를 설명합니다. ([자카르타® EE][10])

다만 서버 코드에서:

```java
request.isSecure()
request.getScheme()
```

를 이용하여 직접 HTTPS 여부를 판단하고 있다면:

```text
X-Forwarded-Proto
Forwarded
Undertow proxy 설정
```

등도 점검해야 합니다.

---

# 15. 현재 JBoss EAP 7.2 환경에서 `web.xml` 적용 범위

예를 들어:

```xml
<session-config>
    <session-timeout>30</session-timeout>
    <cookie-config>
        <http-only>true</http-only>
        <secure>true</secure>
    </cookie-config>
</session-config>
```

라고 하면 핵심적으로 **Servlet Session Cookie**, 즉 일반적인 환경에서는 `JSESSIONID`에 적용됩니다.

모든 Application Cookie가 자동으로 HttpOnly/Secure가 되는 것은 아닙니다.

Red Hat EAP 7.2 문서도 `http-only` 설정이 **session management cookie에 적용되며 다른 browser cookie에는 적용되지 않는다**고 명확히 설명합니다. ([레드햇 문서][11])

따라서:

```text
JSESSIONID → web.xml 설정 대상

USER_ID
LANG
AUTO_LOGIN
TOKEN
Nexacro custom Cookie
기타 업무 Cookie
```

는 각각 생성 위치에서 별도 점검해야 합니다.

---

# 16. 적용 전에 할 수 있는 가장 효과적인 사전 검증

다음 순서로 DEV에서 검증하면 실제 위험을 거의 걸러낼 수 있습니다.

1. **소스 정적 검색**

   ```text
   JSESSIONID
   document.cookie
   getCookieVariable
   setCookieVariable
   removeCookieVariable
   http://
   transaction(
   ```

   특히 JSESSIONID를 JavaScript에서 직접 사용하는 부분을 찾습니다.

2. **현재 로그인 응답 확인**
   브라우저 개발자도구 → Network → 로그인 요청 → Response Headers에서:

   ```http
   Set-Cookie: JSESSIONID=...; Path=/...
   ```

   를 확인합니다.

3. **HttpOnly만 먼저 DEV 적용**

   ```http
   Set-Cookie: JSESSIONID=...; HttpOnly
   ```

   상태에서 로그인, 조회, 저장, 로그아웃, 세션타임아웃을 확인합니다.

4. **Secure 추가**

   ```http
   Set-Cookie:
   JSESSIONID=...; HttpOnly; Secure
   ```

   로 변경합니다.

5. **모든 transaction 요청 확인**
   Network에서:

   ```http
   Cookie: JSESSIONID=...
   ```

   가 실제 붙어 있는지 확인합니다.

6. **HTTP 요청 존재 여부 확인**
   transaction, upload/download, popup 등을 포함하여 한 건이라도 `http://` 호출이 있는지 확인합니다.

7. **세션 만료 강제 테스트**
   서버에서 session invalidate 후 Nexacro transaction을 호출하여:

   ```text
   HTML 로그인 페이지
   ```

   가 내려오는지,

   ```text
   SSV ErrorCode
   ```

   가 내려오는지 확인합니다.

8. **Cross Domain 테스트**
   UI/API 도메인이 다르면:

   ```text
   networksecurelevel
   withCredentials
   Access-Control-Allow-Origin
   Access-Control-Allow-Credentials
   SameSite
   ```

   를 함께 검사합니다.

9. **WRE/NRE 각각 테스트**
   두 환경을 모두 운영한다면 각각 로그인→transaction→세션유지→로그아웃→재로그인을 테스트합니다.

10. **LB가 존재한다면 SSL 종료지점 확인**

    ```text
    Client→LB = HTTPS
    LB→WEB = ?
    WEB→WAS = ?
    ```

    를 확인하고 WAS의 `request.getScheme()` 및 `request.isSecure()` 상태도 확인합니다.

---

# 17. 특히 테스트해야 할 업무 기능

단순 조회 몇 건 성공했다고 배포하면 위험합니다.

| 테스트                 | HttpOnly 영향 | Secure 영향 |   중요도 |
| ------------------- | ----------: | --------: | ----: |
| 최초 로그인              |           △ |       ★★★ | 매우 높음 |
| 일반 조회 transaction   |           △ |       ★★★ | 매우 높음 |
| 등록/수정 SSV           |           △ |       ★★★ | 매우 높음 |
| 세션 Timeout          |          ★★ |        ★★ | 매우 높음 |
| 로그아웃                |          ★★ |         ★ |    높음 |
| 새 창/Popup           |           ★ |        ★★ |    높음 |
| File Upload         |           △ |       ★★★ |    높음 |
| File Download       |           △ |       ★★★ |    높음 |
| Excel Import/Export |           △ |       ★★★ |    높음 |
| Cross Domain API    |          ★★ |       ★★★ | 매우 높음 |
| NRE                 |          ★★ |        ★★ |    높음 |
| WRE                 |          ★★ |       ★★★ |    높음 |

---

# 18. 현재 시스템에 대한 권장 적용 전략

현재와 같이 **Java/Spring + JBoss EAP + Nexacro + HttpSession + SSV** 구조라면 저는 다음 형태를 권장합니다.

```xml
<session-config>
    <session-timeout>30</session-timeout>
    <cookie-config>
        <http-only>true</http-only>
        <secure>true</secure>
    </cookie-config>
</session-config>
```

전제 조건은 다음입니다.

```text
외부 서비스 전체 HTTPS
        │
        ├─ JSESSIONID 직접 JS 접근 없음
        │
        ├─ Nexacro transaction HTTPS
        │
        ├─ Secure cookie가 transaction에 자동 포함
        │
        ├─ 세션 만료 시 HTML Redirect 대신
        │   Nexacro가 처리 가능한 응답
        │
        └─ Cross Domain이면 CORS/credentials 별도 검증
```

이 조건들이 만족되면 **HttpOnly + Secure 적용 때문에 SSV 통신을 변경할 필요는 없습니다.**

---

## 최종 판단

가장 중요한 판단을 세 가지로 압축하면 다음과 같습니다.

**첫째**, `HttpOnly`는 Nexacro의 정상적인 `transaction()` 통신이나 서버의 `request.getSession()`을 방해하지 않습니다. 문제가 되는 것은 **클라이언트 JavaScript가 JSESSIONID를 직접 읽도록 구현된 경우**입니다. JBoss EAP 7.2에서도 HttpOnly session cookie 사용을 공식적으로 지원합니다. ([레드햇 문서][11])

**둘째**, 실제 장애 가능성은 HttpOnly보다 **Secure 쪽이 더 큽니다.** 서비스 URL 중 하나라도 HTTP이거나 Cross Domain/CORS 설정이 맞지 않으면 JSESSIONID가 누락되어 갑자기 세션 만료처럼 보일 수 있습니다. Nexacro도 Secure Cookie는 HTTPS에서만 전송한다고 명확히 정의하고 있습니다. ([Tobesoft][4])

**셋째**, SSV 자체와 HttpOnly/Secure는 충돌하지 않습니다. 오히려 주의해야 할 것은 **쿠키가 누락되었을 때 서버가 302+HTML 로그인 페이지를 반환하여 SSV parser가 실패하는 상황**입니다. 세션 만료/인증 실패를 Nexacro의 `ErrorCode/ErrorMsg` 또는 공통 HTTP 오류 처리 체계로 통일하는 것이 실무적으로 중요합니다. Nexacro X-API는 `PlatformData`와 `ErrorCode/ErrorMsg`를 통한 오류 전달을 공식적으로 지원합니다. ([Tobesoft][5])

현재 시스템에서는 **1차로 소스에서 `document.cookie`, `getCookieVariable`, `JSESSIONID`, `http://`, `transaction(` 5가지를 전수 검색하는 것**이 가장 효과적인 사전점검입니다. 그 결과를 보면 HttpOnly/Secure 적용 시 실제 수정이 필요한 부분을 거의 확정할 수 있습니다.
