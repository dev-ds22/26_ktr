정확도: 95%

# 결론

대체로 **둘 다 접근 가능**합니다. 다만 먼저 정확히 구분해야 할 점은, `ResourceUrlEncodingFilter`는 **실제 파일명을 바꾸는 기능이 아니라** `encodeURL`을 통해 **클라이언트에 노출되는 public URL을 versioned URL로 바꿔주는 역할**이라는 점입니다. Spring 문서와 Javadoc은 이 필터가 internal resource URL을 external/public URL path로 변환한다고 설명하고, `ResourceUrlProvider`는 클라이언트가 사용해야 할 public URL을 제공하는 중앙 컴포넌트라고 명시합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))  
즉, 예를 들어 실제 리소스가 서버에 `logo.png`로 존재하더라도, 클라이언트 HTML에는 `logo-a12bc3....png` 같은 versioned URL이 나갈 수 있습니다. 이때 **물리 파일명이 바뀌는 것은 아니고**, `VersionResourceResolver`가 버전 문자열이 포함된 요청 경로를 실제 원본 리소스에 매핑해 주는 구조입니다. `ContentVersionStrategy`가 파일명에 해시를 붙인 public URL을 만든다는 점도 Javadoc에 명시되어 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))

## 질문에 대한 직접 답변

|질문|답변|
|---|---|
|변경된 파일이름(예: `app-해시.js`)으로 접근 가능한가?|예. `VersionResourceResolver`가 versioned URL을 실제 리소스로 해석해 주면 가능합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))|
|원본 파일명(예: `app.js`)으로도 접근 가능한가?|보통 예. 기본 정적 리소스 핸들러는 원본 경로도 그대로 서빙하므로, 별도 차단을 하지 않으면 접근 가능한 경우가 일반적입니다. Spring 문서도 `/resources/**` 같은 매핑이 주어지면 해당 상대 경로의 정적 리소스를 그대로 제공한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))|
|`ResourceUrlEncodingFilter`가 원본 URL 접근을 막아주는가?|아니오. 이 필터는 **출력 URL rewrite**용이지 **원본 URL 차단**용이 아닙니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))|

## 실무적으로 어떻게 이해하면 되는가

보통 아래처럼 이해하면 맞습니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))

```text
실제 서버 자원: /resources/js/app.js
클라이언트에 노출되는 URL: /resources/js/app-e36d2e....js
```

하지만 원본 `/resources/js/app.js` 자체가 없어지는 것은 아닙니다. 따라서 **별도 제어를 하지 않으면** 아래 두 요청이 모두 성공할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))

```text
GET /resources/js/app.js
GET /resources/js/app-e36d2e....js
```

## 중요한 주의사항

`ResourceUrlEncodingFilter`만 등록했다고 versioning이 완성되는 것은 아닙니다. **`VersionResourceResolver` 또는 이에 해당하는 version strategy 설정이 같이 있어야** versioned URL이 의미를 가집니다. Spring 공식 문서도 version 삽입은 `VersionResourceResolver`를 통해 구성하고, 그 다음 `ResourceUrlProvider`나 `ResourceUrlEncodingFilter`로 rewrite한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))  
또한 JS/CSS/image라고 해서 접근 가능 여부가 달라지는 것은 아닙니다. 정적 리소스 핸들러 관점에서는 모두 같은 리소스 취급입니다. 다만 CSS 내부 링크는 `CssLinkResourceTransformer`로 추가 rewrite가 가능하다는 차이가 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))

## 운영 관점 권장사항

실무에서는 **“둘 다 접근 가능하더라도, 화면에서는 반드시 versioned URL만 사용”**하는 것이 맞습니다. 이유는 원본 URL로 접근하면 cache busting 효과를 우회하게 되어 브라우저나 CDN에 오래된 파일이 남을 수 있기 때문입니다. `VersionResourceResolver`는 버전이 바뀔 때 URL도 바뀌도록 하여 장기 캐시 전략에 쓰는 컴포넌트라고 Javadoc이 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))  
따라서 정리하면 다음과 같습니다.

- **기본 동작:** 원본 파일명 URL도, versioned URL도 둘 다 살아있는 경우가 많음. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))
    
- **필터 역할:** `ResourceUrlEncodingFilter`는 URL을 바꿔서 내보낼 뿐, 원본 URL 접근을 막지는 않음. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))
    
- **권장 사용:** JSP/HTML/JS에서는 versioned URL만 사용하고, 원본 URL 직접 사용은 피하는 것이 맞음. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/package-summary.html "org.springframework.web.servlet.resource (Spring Framework 7.0.7 API)"))  
    원하시면 다음 답변에서 **원본 파일명 URL은 차단하고 versioned URL만 허용하는 방법**까지 Spring MVC + 웹서버(Nginx/Apache) 기준으로 정리해드리겠습니다.

정확도: 97%

# 결론

`ResourceUrlEncodingFilter`를 써도 **원본 파일명 URL과 변경된(버전이 붙은) URL이 둘 다 외부에서 접근 가능한 이유**는 구조적으로 역할이 분리되어 있기 때문입니다.

- `ResourceUrlEncodingFilter`는 **응답에 실려 나가는 URL을 바꿔주는 역할**만 합니다.
    
- 실제 정적 리소스 요청을 받아 주는 것은 `ResourceHttpRequestHandler` + `ResourceResolver` 체인입니다.
    
- 그 resolver 체인 안의 `VersionResourceResolver`가 **버전 URL도 원본 리소스로 역매핑**해 줍니다.
    
- 동시에 원본 URL 자체는 정적 리소스 핸들러 매핑(`/resources/**` 등)으로 **그대로 살아 있기 때문**에, 별도 차단을 하지 않으면 원본도 계속 접근됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/ResourceUrlEncodingFilter.html "ResourceUrlEncodingFilter (Spring Framework 7.0.7 API)"))
    

# 소스 기준으로 보면 왜 그런가

## 1) `ResourceUrlEncodingFilter`는 “출력 URL 변경”만 담당

Spring 공식 Javadoc은 `ResourceUrlEncodingFilter`를 **`HttpServletResponse#encodeURL`을 override해서 internal resource URL을 external/public URL로 바꾸는 필터**라고 설명합니다. 즉, 이 필터는 **클라이언트에게 내려보내는 HTML/JSP의 URL 문자열**을 바꾸는 계층이지, 들어오는 HTTP 요청을 차단하거나 원본 경로를 비활성화하는 계층이 아닙니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/ResourceUrlEncodingFilter.html "ResourceUrlEncodingFilter (Spring Framework 7.0.7 API)"))  
실제 소스 흐름도 같습니다.

- `doFilter(...)`에서 request/response를 wrapper로 감쌉니다.
    
- `ResourceUrlEncodingResponseWrapper#encodeURL(String url)`에서 `request.resolveUrlPath(url)`를 시도합니다.
    
- 성공하면 바뀐 URL을 `super.encodeURL(...)`로 넘기고,
    
- 실패하면 **그냥 원래 URL을 그대로** `super.encodeURL(url)`로 넘깁니다.  
    즉 이 필터는 “버전 URL로 바꿔서 내려보낼 수 있으면 바꾸고, 아니면 원본 그대로 둔다”는 동작입니다. 소스상으로도 request/response wrapper와 `encodeURL` override, 그리고 `resolveUrlPath(url)` 실패 시 원래 URL로 fallback하는 흐름이 확인됩니다. ([GitHub](https://raw.githubusercontent.com/spring-projects/spring-framework/main/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/ResourceUrlEncodingFilter.java "raw.githubusercontent.com"))
    

## 2) URL을 실제로 버전형으로 바꾸는 주체는 `ResourceUrlProvider`

Spring 공식 문서는 versioning 설정 후 `ResourceUrlProvider`를 사용하면 **resolver/transformer chain 전체를 적용해 URL을 다시 쓸 수 있고**, 이를 `ResourceUrlEncodingFilter`로 JSP/Thymeleaf 등에서 투명하게 적용할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))  
`ResourceUrlProvider` Javadoc도 이 컴포넌트를 **클라이언트가 사용해야 할 public URL path를 얻는 중앙 컴포넌트**라고 설명합니다. 또한 `getForLookupPath(...)`는 resource handler mapping과 resolver chain을 이용해 **외부에 노출할 URL**을 계산합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/ResourceUrlProvider.html "ResourceUrlProvider (Spring Framework 7.0.0 API)"))  
즉 순서는 아래와 같습니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))

|단계|동작|
|---|---|
|JSP/템플릿에서 `<c:url>`, URL 태그, `encodeURL()` 호출|`ResourceUrlEncodingFilter`가 가로챔|
|필터 내부 request wrapper|`ResourceUrlProvider.getForLookupPath(...)` 호출|
|`ResourceUrlProvider`|resource handler + resolver chain으로 “외부용 URL” 계산|
|결과|원본 `/resources/js/app.js`가 `/resources/js/app-해시.js` 같은 public URL로 바뀌어 응답에 실림|
|여기서 중요한 점은 **응답 HTML의 링크를 versioned URL로 바꿔 줄 뿐**, 서버가 원본 URL을 폐기하거나 차단하지는 않는다는 것입니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/ResourceUrlEncodingFilter.html "ResourceUrlEncodingFilter (Spring Framework 7.0.7 API)"))||

## 3) 원본 파일명이 계속 접근되는 이유: 정적 리소스 핸들러가 원본 경로를 그대로 서빙

Spring 정적 리소스 문서는 `/resources/**` 같은 매핑이 들어오면 그 상대 경로를 사용해 `/public`, `classpath:/static/` 등의 위치에서 정적 리소스를 찾아 서빙한다고 설명합니다. 즉 `/resources/js/app.js`라는 원본 경로 요청은 **원래부터 유효한 정적 리소스 요청**입니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))  
그리고 `VersionResourceResolver` 소스를 보면, **들어오는 요청을 처리할 때 첫 단계가 원본 경로 그대로 `chain.resolveResource(request, requestPath, locations)`를 시도하는 것**입니다. 이 단계에서 리소스를 찾으면 그대로 반환합니다. 즉 원본 URL이 실제 파일과 매칭되면 여기서 바로 성공합니다. ([GitHub](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/VersionResourceResolver.java "spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/VersionResourceResolver.java at main · spring-projects/spring-framework · GitHub"))  
정리하면 원본 URL이 계속 살아 있는 직접 이유는 다음입니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))

1. `/resources/**` 매핑 자체가 원본 경로 요청을 받는다.
    
2. resolver 체인이 먼저 **원본 경로 그대로** 리소스를 찾는다.
    
3. 찾으면 그대로 응답한다.  
    그래서 별도 제어가 없으면 `/resources/js/app.js`는 계속 외부 접근 가능합니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))
    

## 4) 변경된 파일명도 접근되는 이유: `VersionResourceResolver`가 역매핑

`VersionResourceResolver` Javadoc은 이 resolver가 **version string이 포함된 요청 경로를 해석**해서, 장기 캐시 전략에서 사용할 수 있게 한다고 설명합니다. 또한 content-based version의 예로 `css/main-e36d2e...css` 같은 형태를 직접 제시합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/VersionResourceResolver.html "VersionResourceResolver (Spring Framework 7.0.7 API)"))  
실제 소스에서도 동작이 명확합니다.

- 먼저 `chain.resolveResource(request, requestPath, locations)`로 원본처럼 바로 찾는 것을 시도
    
- 실패하면 `getStrategyForPath(requestPath)`로 버전 전략 확인
    
- `extractVersion(requestPath)`로 버전 추출
    
- `removeVersion(requestPath, candidateVersion)`로 **원본 경로(simplePath)** 복원
    
- 그 원본 경로로 다시 `chain.resolveResource(...)`
    
- 실제 파일 내용으로 계산한 버전과 요청 URL의 버전이 일치하면 `FileNameVersionedResource` 반환 ([GitHub](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/VersionResourceResolver.java "spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/VersionResourceResolver.java at main · spring-projects/spring-framework · GitHub"))  
    즉 `/resources/js/app-해시.js` 요청이 들어와도 내부적으로는:
    

```text
/resources/js/app-해시.js
→ 버전 추출
→ /resources/js/app.js 로 복원
→ 실제 파일 찾기
→ 버전 검증 후 응답
```

이 흐름으로 처리되기 때문에 **변경된 파일명도 접근 가능**합니다. ([GitHub](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/VersionResourceResolver.java "spring-framework/spring-webmvc/src/main/java/org/springframework/web/servlet/resource/VersionResourceResolver.java at main · spring-projects/spring-framework · GitHub"))

# 핵심 원인 한 문장 요약

**원본 URL은 “원래 정적 리소스 핸들러가 직접 서빙”하고, 변경된 URL은 “VersionResourceResolver가 원본으로 역변환해서 서빙”하기 때문에 둘 다 접근됩니다.** `ResourceUrlEncodingFilter`는 그 중 어느 쪽도 차단하지 않고, 단지 화면에 어떤 URL을 출력할지 바꿀 뿐입니다. ([Home](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/static-resources.html "Static Resources :: Spring Framework"))

# 소스 관점 요약표

|구성요소|역할|원본 URL 접근에 미치는 영향|변경된 URL 접근에 미치는 영향|
|---|---|---|---|
|`ResourceUrlEncodingFilter`|응답의 `encodeURL()` 결과 rewrite|차단 안 함|차단 안 함|
|`ResourceUrlProvider`|클라이언트에 노출할 public URL 계산|원본을 versioned URL로 바꿔 출력 가능|public URL 생성|
|정적 리소스 핸들러(`/resources/**`)|원본 파일 서빙|가능하게 함|resolver chain 통해 가능|
|`VersionResourceResolver`|버전 URL 생성/해석|원본 요청이면 보통 그대로 통과|버전 URL을 원본 파일로 역매핑|
|([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/ResourceUrlProvider.html "ResourceUrlProvider (Spring Framework 7.0.0 API)"))||||

# 실무적으로 의미하는 바

따라서 `ResourceUrlEncodingFilter`를 적용했다고 해서 보안적으로 “원본 파일명은 외부에서 못 보게 된다”라고 이해하면 안 됩니다. 기본 동작은 **표시되는 링크를 versioned URL로 바꾸는 것**이지, **원본 URL 폐쇄**가 아닙니다. 원본 URL까지 막고 싶다면 별도의 웹서버 규칙, Spring Security 규칙, 커스텀 리소스 핸들러 정책이 추가로 필요합니다. 이 점은 Spring 문서와 소스의 역할 분리 구조상 자연스러운 결과입니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/resource/ResourceUrlEncodingFilter.html "ResourceUrlEncodingFilter (Spring Framework 7.0.7 API)"))  
원하시면 다음 답변에서 **“원본 파일명 URL은 차단하고 versioned URL만 허용”하는 실제 구현 방법**을 Spring MVC / Nginx 기준으로 바로 정리해드리겠습니다.