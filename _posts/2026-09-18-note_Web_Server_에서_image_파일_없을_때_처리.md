---
layout: single
title: "Web_Server_에서_image_파일_없을_때_처리"
excerpt: "Web_Server_에서_image_파일_없을_때_처리"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-18"
last_modified_at: "2026-09-18 14:40:43 +0900"
mermaid: false
---
**부하는 발생합니다.** 다만 보통은 “404 한 건” 자체가 큰 부하라기보다, **없는 상품 이미지 URL이 대량·반복 호출될 때** 파일시스템 조회, nginx worker 처리, access log 기록, CDN 원점(origin) 재요청이 누적되어 문제가 됩니다.

현재 구조가 대략:

```text
Client
  ↓
CDN
  ↓ cache miss
WEB/nginx
  ↓
파일 경로 탐색
  ↓
없음
  ↓
404
```

라면 **WAS/Spring까지 넘기지 않고 nginx에서 끝내는 것이 가장 좋습니다.**

## 1. 실제 어떤 부하가 생기는가

이미지 하나가 없을 때 nginx는 보통 다음 작업을 합니다.

```text
요청 accept
 → URI 해석
 → location 선택
 → filesystem stat/open 확인
 → ENOENT(파일 없음)
 → 404 생성
 → access log 기록
 → client/CDN에 응답
```

CPU 사용량은 한 건당 크지 않지만, 다음과 같이 반복되면 영향이 커집니다.

```text
상품목록 한 페이지: 이미지 100개
그중 없는 이미지: 30개
동시 사용자: 1,000명
```

단순 계산만 해도:

```text
30 × 1,000
= 30,000개의 404 요청
```

이 발생할 수 있습니다.

특히 CDN이 404를 캐시하지 않으면 매번 원점 WEB까지 내려옵니다.

---

# 2. 가장 효과적인 해결책 1: CDN에서 404 Negative Cache

효과가 가장 큽니다.

일반적인 CDN 정책을:

```text
200 → 장기 cache
404 → 짧게 cache
```

로 둡니다.

예:

```text
404 → 1~5분
```

그러면 같은 없는 이미지 요청이:

```text
CDN MISS
 ↓
WEB 404
 ↓
CDN이 404 저장
```

된 이후에는 일정 기간:

```text
Client
 ↓
CDN
 ↓
404 즉시 응답

WEB까지 안 내려옴
```

이 됩니다.

이걸 흔히 **negative caching**이라고 합니다.

없는 파일이 나중에 생성될 수 있으므로 404를 너무 오래 캐시하면 안 됩니다.

상품 이미지라면 보통:

```text
30초 ~ 5분
```

정도의 짧은 negative cache부터 시작하는 것이 안전합니다.

---

# 3. Nginx에서도 없는 파일 lookup 자체를 캐시할 수 있다

현재 같은 상황에서는 이 설정이 매우 효과적일 수 있습니다.

Nginx는 `open_file_cache_errors on`으로 **파일이 존재하지 않는다는 조회 결과도 캐시**할 수 있습니다. 공식 문서에도 file lookup error를 `open_file_cache`로 캐시하려면 `open_file_cache_errors`를 활성화해야 한다고 명시되어 있습니다. :chatgpt-content-reference{index="0"}

예:

```nginx
open_file_cache          max=10000 inactive=60s;
open_file_cache_valid    60s;
open_file_cache_min_uses 2;
open_file_cache_errors   on;
```

의미는:

```text
/image/product/1234.jpg
```

가 없다는 사실을 한 번 filesystem에서 확인한 뒤:

```text
ENOENT
```

결과를 일정 시간 메모리에 보관합니다.

그러면 반복 요청 때마다:

```text
stat()
open()
filesystem lookup
```

을 다시 할 필요가 줄어듭니다.

이 설정은 현재 문제와 정확히 맞습니다.

---

# 4. 이미지 location에만 제한하는 것이 좋다

전체 서버에 무조건 적용하기보다 상품 이미지 경로가 명확하다면 해당 `location`에 적용하는 것이 안전합니다.

예:

```nginx
location /product/image/ {

    root /data/images;

    open_file_cache          max=10000 inactive=60s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;

    try_files $uri =404;
}
```

여기서 중요한 것은:

```nginx
try_files $uri =404;
```

입니다.

이렇게 하면 파일이 없을 때:

```text
nginx
 ↓
바로 404
```

로 끝납니다.

---

# 5. 절대로 Spring까지 보내지 않는 것이 중요

안 좋은 구조:

```text
/image/product/123.jpg
   ↓
nginx
   ↓
파일 없음
   ↓
proxy_pass
   ↓
JBoss
   ↓
DispatcherServlet
   ↓
Spring HandlerMapping
   ↓
ResourceHandler
   ↓
404
```

이렇게 되면 단순 이미지 하나 때문에:

```text
WEB thread
WAS socket
Undertow worker
Servlet filter
Interceptor
Spring DispatcherServlet
HandlerMapping
Exception 처리
Logging
```

까지 타게 됩니다.

이건 낭비가 큽니다.

권장:

```text
/image/**
   ↓
nginx가 직접 처리
   ↓
존재 → 파일 반환
없음 → nginx 404
```

입니다.

---

# 6. Spring을 사용해야 하는 구조라면

Spring MVC 5.3의 `ResourceHandler`도 정적 리소스를 처리할 수 있습니다.

예:

```java
@Override
public void addResourceHandlers(
        ResourceHandlerRegistry registry) {

    registry.addResourceHandler("/product/image/**")
            .addResourceLocations("file:/data/images/")
            .setCachePeriod(86400);
}
```

Spring 5.3은 `ResourceHandlerRegistry`를 통해 이미지/CSS/JS 같은 정적 리소스를 처리하고 cache header도 설정할 수 있습니다. :chatgpt-content-reference{index="1"}

하지만 현재 목적이 **부하 절감**이라면:

```text
nginx > Spring ResourceHandler
```

입니다.

왜냐하면 Spring 방식은 이미 WAS까지 들어온 뒤이기 때문입니다.

---

# 7. 더 좋은 방법: 없는 이미지 대신 기본 이미지 반환

상품 이미지라면 이 방법도 매우 효과적입니다.

현재:

```text
/image/product/A.jpg
 ↓
없음
 ↓
404
```

대신:

```text
/image/product/A.jpg
 ↓
없음
 ↓
/images/no-image.png
```

을 반환합니다.

Nginx:

```nginx
location /product/image/ {

    root /data/images;

    try_files $uri /images/no-image.png;
}
```

그러면:

```text
404 로그
404 재요청
브라우저 broken image
```

를 모두 줄일 수 있습니다.

다만 한 가지 주의가 있습니다.

없는 URL에 대해:

```text
200 + no-image.png
```

를 반환하면 CDN이 그 URL을 `200`으로 오래 캐시할 수 있습니다.

그 뒤 실제 이미지가 등록되어도 CDN이 계속 기본 이미지를 보여줄 수 있습니다.

따라서 상품 이미지가 나중에 생성되는 구조라면 캐시 정책을 조정해야 합니다.

---

# 8. 더 안전한 기본 이미지 처리

두 가지 선택지가 있습니다.

### 8-1-1. 방법 A
404 상태 그대로 + 기본 이미지 body

```nginx
error_page 404 =404 /images/no-image.png;
```

다만 내부 처리 방식은 실제 nginx 설정에 따라 세심하게 테스트해야 합니다.

### 8-1-2. 방법 B
404는 그대로 반환하고 CDN이 404를 짧게 캐시

현재 환경에서는 이쪽을 더 권장합니다.

```text
실제 파일 존재
 → 200 장기 cache

파일 없음
 → 404 60초 cache
```

이 구조가 의미적으로도 정확합니다.

---

# 9. Nginx proxy cache를 쓰는 경우 404도 캐시 가능

만약 이미지 origin이 nginx 뒤의 다른 서버라서 `proxy_pass`하는 구조라면:

```nginx
proxy_cache_valid 200 10m;
proxy_cache_valid 404 1m;
```

처럼 404를 캐시할 수 있습니다.

Nginx 공식 문서도 `proxy_cache_valid 404 1m;` 형태로 404 응답을 별도 TTL로 캐시할 수 있다고 설명합니다. :chatgpt-content-reference{index="2"}

예:

```nginx
location /product/image/ {

    proxy_cache IMAGE_CACHE;

    proxy_cache_valid 200 1h;
    proxy_cache_valid 404 1m;

    proxy_pass http://image_origin;
}
```

---

# 10. 404 access log 비용도 무시하면 안 된다

없는 이미지가 대량 발생하면 실제 filesystem lookup보다 **access.log가 더 큰 비용**이 될 수도 있습니다.

예:

```text
404 이미지 요청 100만 건/일
```

이면:

```text
access.log
```

에 100만 줄이 추가됩니다.

따라서 특정 이미지 경로에 대해서는 별도의 로그 정책을 둘 수 있습니다.

예:

```nginx
location /product/image/ {

    access_log /var/log/nginx/image_access.log image_format;

    try_files $uri =404;
}
```

또는 운영 정책이 허용하면 반복적인 정적 404는 로그를 줄일 수 있습니다.

```nginx
log_not_found off;
```

이 설정은 특히 nginx error log의 `open() failed ... No such file` 노이즈 감소에 도움이 됩니다.

다만 **access_log까지 완전히 끄는 것은 초기에는 권장하지 않습니다.**

먼저 404 발생량을 측정해야 합니다.

---

# 11. 제가 추천하는 실무 설정

현재처럼 CDN → nginx → 정적 파일 구조라면 다음 방향을 권장합니다.

```nginx
location /product/image/ {

    root /data/images;

    # 존재/미존재 filesystem 결과 cache
    open_file_cache          max=50000 inactive=60s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;

    # 없는 파일은 WAS로 넘기지 않고 nginx에서 종료
    try_files $uri =404;

    # filesystem 404 error log 노이즈 감소
    log_not_found off;

    # 정상 이미지는 브라우저/CDN cache 활용
    expires 1d;
}
```

핵심은:

```text
open_file_cache_errors on
+
try_files ... =404
```

입니다.

---

# 12. CDN까지 포함한 이상적인 구조

최종적으로는:

```text
Client
  ↓
CDN
  │
  ├─ 200 cache
  │    → 정상 이미지는 Origin 접근 없음
  │
  └─ 404 negative cache
       → 같은 없는 파일 반복 Origin 접근 방지
          ↓
        nginx
          │
          ├─ open_file_cache
          │
          ├─ 존재 → file
          │
          └─ 없음 → nginx 자체 404
                       X
                       │
                     JBoss
```

가 가장 좋습니다.

즉 **Spring/JBoss는 상품 이미지 404 처리에서 완전히 제외**하는 것이 이상적입니다.

---

# 13. 현재 504/CLOSE_WAIT 문제와는 성격이 다르다

없는 이미지 404 요청이 많으면 WEB 서버 connection/thread/file lookup/log I/O는 증가합니다.

하지만 정상적인 nginx 정적 404라면:

```text
수 ms 또는 그 이하
```

수준으로 매우 빠르게 처리되는 경우가 많아서, 이것만으로 WAS 60초 지연이나 CLOSE_WAIT 대량 발생의 직접 원인이 될 가능성은 낮습니다.

다만 다음 구조라면 이야기가 달라집니다.

```text
이미지 404
 ↓
nginx
 ↓
JBoss proxy
 ↓
Spring
 ↓
filesystem / NAS / NFS
```

이 경우는 WAS thread/socket을 소비하기 때문에 트래픽이 크면 실제 성능 문제로 확대될 수 있습니다.

---

## 13-1. 적용 우선순위

현재 환경에서는 다음 순서가 가장 효과적입니다.

1. **없는 이미지 요청이 WAS까지 내려가는지 먼저 확인**
2. 내려간다면 nginx `try_files ... =404`로 즉시 차단
3. `open_file_cache_errors on`으로 없는 파일 조회 결과 캐시 :chatgpt-content-reference{index="3"}
4. CDN에서 404 negative caching을 30초~5분 적용
5. `log_not_found off`로 반복적 filesystem error 로그 감소
6. 정상 이미지는 CDN/browser 장기 캐시
7. 상품 DB에서 이미지 존재 여부를 관리하여 애초에 존재하지 않는 URL을 HTML에 생성하지 않는 것이 최상위 해결책

특히 **가장 근본적인 해결은 상품 데이터 단계에서 없는 이미지 URL 자체를 생성하지 않고 기본 이미지 URL을 내려주는 것**입니다. 이 경우 CDN, WEB 어느 곳에도 불필요한 404 요청이 발생하지 않습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
