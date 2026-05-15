---
layout: single
title: "CDN TTBF 원인 분석"
excerpt: "CDN TTBF 원인 분석"

categories:
  - tech
tags:
  - [tech, memo]

toc: false
toc_sticky: true

date: "2026-05-15"
last_modified_at: "2026-05-15 11:11:47 +0900"
---

## 특정 QueryString 에서만 파일 로딩이 느려지는 현상

![[Pasted image 20260515110851.png]]

## 결론

`selectDigtlKbcGoodsListMng.js?ver=5.035555555`만 느리고, **QueryString을 제거하거나 `ver` 값을 바꾸면 빨라진다면** 가장 유력한 원인은 **`ver=5.035555555` 조합이 CDN에서 별도 Cache Key로 취급되고 있고, 그 캐시 객체만 만료·재검증·Origin Fetch·비정상 캐시 상태에 걸려 있기 때문**입니다.  
삼성SDS CDN 설명에서도 Cache Key에 QueryString 포함 여부에 따라 캐시 동작이 달라지며, QS가 다르면 다른 객체로 인식될 수 있다고 설명합니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))

## 현상 해석

|현상|의미|
|---|---|
|`?ver=5.035555555`만 느림|해당 URL 조합의 CDN 캐시 객체 문제 가능성 높음|
|QueryString 제거 시 빠름|`/js/go/kbc/selectDigtlKbcGoodsListMng.js`가 다른 Cache Key로 처리될 가능성|
|`ver` 값 변경 시 빠름|새 `ver` 값이 다른 Cache Key로 처리되어 느린 캐시 객체를 우회했을 가능성|
|`ver=5.035555555`가 1년 이상 전 값|오래된 캐시 객체가 만료되어 매번 재검증 또는 Origin 조회 중일 가능성|
|TTFB가 길고 Download가 짧음|JS 실행/파일 크기 문제가 아니라 CDN 또는 Origin 응답 대기 문제 가능성|

## 가장 유력한 원인

### 1. `ver=5.035555555`가 별도 CDN Cache Key로 분리됨

현재 URL은 아래처럼 Path와 QueryString으로 구성됩니다.

```text
/js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555555
```

CDN이 QueryString을 Cache Key에 포함하면 아래 URL들은 서로 다른 캐시 객체가 됩니다.

```text
selectDigtlKbcGoodsListMng.js
selectDigtlKbcGoodsListMng.js?ver=5.035555555
selectDigtlKbcGoodsListMng.js?ver=5.035555556
```

Akamai 문서도 QueryString 또는 일부 QueryString을 캐시 객체 구분에 사용할지 제어할 수 있다고 설명합니다. ([Akamai TechDocs](https://techdocs.akamai.com/adaptive-media-delivery/docs/cache-key-query-param-amd "Cache Key Query Parameters"))  
따라서 `ver=5.035555555`만 느리다는 것은 **그 Cache Key에 해당하는 객체만 문제가 있다**는 신호입니다.

## 2. 1년 이상 된 `ver` 값으로 인해 캐시가 만료되어 재검증 중일 가능성

정적 리소스는 보통 버전이 붙어 있으면 장기 캐시를 적용합니다. web.dev도 versioned resource에는 `Cache-Control: max-age=31536000` 같은 장기 캐시 설정을 권장합니다. ([web.dev](https://web.dev/articles/http-cache "Prevent unnecessary network requests with the HTTP Cache  |  Articles  |  web.dev"))  
그런데 `ver=5.035555555`가 1년 이상 전 값이라면 다음 상황이 가능합니다.

```text
1. CDN Edge에 예전에 캐시됨
2. TTL 만료
3. 요청 때마다 CDN이 Origin에 재검증 또는 재조회
4. Origin 응답이 느려 TTFB 증가
5. QueryString 제거/변경 시 다른 캐시 객체를 타서 빠르게 응답
```

MDN 기준으로 `max-age`는 응답이 생성된 뒤 얼마 동안 fresh로 재사용 가능한지를 의미하며, `Age` 헤더는 응답 생성 후 경과 시간 판단에 사용됩니다. ([MDN 웹 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control "Cache-Control header - HTTP | MDN"))

## 3. 해당 Cache Key만 CDN Edge에 “나쁜 상태”로 남아 있을 가능성

CDN에서는 특정 URL 객체만 다음 상태가 될 수 있습니다.

```text
- 특정 Edge에만 오래된 객체가 남음
- 객체는 있으나 TTL 만료 상태
- 매번 If-Modified-Since / If-None-Match 재검증 발생
- Origin 응답 지연으로 TTFB 증가
- 일부 Edge에서만 Cache Miss 또는 Refresh Miss 반복
- 해당 URL만 압축/캐시 정책 예외 적용
```

삼성SDS Global CDN 기능 설명에도 Cache Key 설정, 캐시 정책, 캐시 만료 시간, TTL 도달 후 Edge 캐시 Purge 기능이 포함되어 있습니다. ([Samsung SDS](https://www.samsungsds.com/en/network-global-cdn/global-cdn.html "Global CDN | Cloud Service | Samsung SDS"))

## 4. CDN이 해당 QueryString 요청을 Origin까지 그대로 전달하고 있을 가능성

CDN 설정에 따라 QueryString이 Origin 요청에도 전달될 수 있습니다.  
이 경우 Origin 로그에는 아래 요청이 그대로 찍힐 수 있습니다.

```text
GET /js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555555
```

Origin Nginx 또는 WAS가 정적 파일을 처리할 때 QueryString 자체는 보통 파일 경로 탐색에 직접 영향이 작지만, 다음 경우에는 느려질 수 있습니다.

```text
- 해당 경로가 정적 파일 직접 서빙이 아니라 WAS 또는 fallback 라우팅을 탐
- 오래된 파일 버전 요청에 대해 Origin에서 404 fallback 또는 내부 rewrite 발생
- CDN이 캐시하지 않고 매번 Origin으로 전달
- Origin에서 gzip 압축을 매번 동적으로 수행
- Origin 파일 접근 I/O가 지연됨
```

## 확인해야 할 핵심 헤더

문제 URL과 빠른 URL을 비교해야 합니다.

```text
느린 URL:
https://focdn.gcdn.samsungsdscloud.com/js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555555
빠른 URL:
https://focdn.gcdn.samsungsdscloud.com/js/go/kbc/selectDigtlKbcGoodsListMng.js
https://focdn.gcdn.samsungsdscloud.com/js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555556
```

비교 항목:

```http
Cache-Control
Age
Expires
ETag
Last-Modified
Content-Length
Content-Encoding
Vary
X-Cache
X-Cache-Key
Via
Server-Timing
```

특히 아래처럼 판단합니다.

|헤더|느린 URL에서 의심할 값|
|---|---|
|`X-Cache`|`MISS`, `TCP_MISS`, `REFRESH_MISS`, `BYPASS`|
|`Age`|없음, `0`, 또는 매번 초기화|
|`Cache-Control`|`public`만 있고 `max-age` 없음|
|`Vary`|불필요하게 많은 값|
|`ETag`|URL별로 다르거나 재검증 반복|
|`Content-Encoding`|느린 URL만 압축 미적용 또는 동적 압축|
|`Content-Length`|느린 URL만 파일 크기 다름|

## CDN에 확인 요청할 사항

### 1. Cache Key 정책

```text
/js/go/kbc/selectDigtlKbcGoodsListMng.js 경로에서 QueryString을 Cache Key에 포함하는가?
ver 파라미터 전체 값을 Cache Key에 포함하는가?
QueryString 순서까지 Cache Key에 포함하는가?
특정 QueryString 요청을 cache bypass 처리하는 규칙이 있는가?
```

핵심 질문:

```text
selectDigtlKbcGoodsListMng.js와 selectDigtlKbcGoodsListMng.js?ver=5.035555555는 CDN에서 같은 객체인가, 다른 객체인가?
```

## 2. 느린 URL의 Cache Hit/Miss 로그

```text
해당 URL의 최근 요청이 HIT인지 MISS인지 확인
특정 Edge 서버에서만 MISS가 반복되는지 확인
Origin Shield 사용 여부 확인
Cache Refresh 또는 Revalidation이 반복되는지 확인
```

## 3. TTL/만료 정책

```text
해당 JS 경로의 CDN TTL은 몇 초인가?
Origin의 Cache-Control을 따르는가, CDN 정책으로 override하는가?
1년 이상 된 ver 값의 캐시 객체가 TTL 만료 후 어떻게 처리되는가?
stale 상태에서 Origin 재검증을 수행하는가?
```

## 4. Purge 상태

```text
ver=5.035555555 URL 객체만 Purge 가능한가?
Path 기준 Purge와 QueryString 포함 Purge가 어떻게 다른가?
해당 객체가 일부 Edge에만 남아 있는지 확인 가능한가?
Purge 후 첫 요청과 두 번째 요청의 응답 시간이 어떻게 달라지는가?
```

삼성SDS 설명에 따르면 Origin 콘텐츠가 갱신되어도 Edge 캐시가 만료되기 전에는 기존 캐시를 사용할 수 있고, 즉시 갱신이 필요하면 Purge로 Edge 캐시를 제거하는 방식이 사용됩니다. ([Samsung SDS](https://www.samsungsds.com/kr/techreport/networking-cdn.html "전보다 더 빠르고 효율적인 콘텐츠 전달을 위한 솔루션, CDN  | 클라우드 기술 백서 | 삼성SDS"))

## 5. Origin Fetch 시간

```text
CDN Edge → Origin 요청 시간이 긴가?
Origin Nginx access log의 request_time은 얼마인가?
Origin에서 해당 URL이 200인지 304인지 확인
Origin에서 해당 파일을 정적 파일로 직접 서빙하는가?
Origin 응답에 Cache-Control, ETag, Last-Modified가 정상 포함되는가?
```

## 운영 측면 대책

### 단기 대책

|우선|대책|설명|
|--:|---|---|
|1|`ver` 값 변경|현재 느린 Cache Key를 우회하는 즉시 조치|
|2|CDN Purge|`?ver=5.035555555` 객체를 명시적으로 제거|
|3|CDN Preload|Purge 후 해당 JS를 Edge에 미리 적재|
|4|헤더 비교|느린 URL과 빠른 URL의 Cache-Control/Age/X-Cache 비교|
|5|Origin 로그 확인|CDN Miss 시 Origin 응답 시간이 긴지 확인|

## 중기 대책

```text
1. JS 파일명에 content hash 적용
2. QueryString 버전 방식 최소화
3. 정적 JS는 long TTL 적용
4. 배포 시 CDN Purge/Preload 자동화
5. CDN Cache Key에서 불필요한 QueryString 제외 검토
```

권장 방식:

```text
기존:
selectDigtlKbcGoodsListMng.js?ver=5.035555555
개선:
selectDigtlKbcGoodsListMng.5f3a9c2.js
```

정적 리소스 URL에 버전 또는 해시를 포함하고, 해당 리소스가 fresh인 동안 변경되지 않는다면 `immutable`을 사용할 수 있습니다. MDN도 version/hash가 포함된 URL과 long `max-age`를 함께 쓰는 cache-busting 패턴을 설명합니다. ([MDN 웹 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control "Cache-Control header - HTTP | MDN"))

## 권장 Cache-Control 예시

파일명이 해시 기반으로 바뀌는 구조라면:

```http
Cache-Control: public, max-age=31536000, immutable
```

QueryString 버전 방식을 유지한다면 최소한:

```http
Cache-Control: public, max-age=31536000
```

단, 파일 내용이 바뀌면 반드시 URL도 바뀌어야 합니다.

## 실무 판단

현재 상황은 아래 가능성이 가장 높습니다.

```text
1. ver=5.035555555가 CDN에서 별도 Cache Key로 잡힘
2. 해당 객체가 1년 이상 된 오래된 캐시 객체이거나 TTL 만료 상태
3. 매번 CDN이 Origin 재검증 또는 Origin Fetch 수행
4. Origin 응답이 느려 TTFB가 커짐
5. QueryString 제거/변경 시 다른 Cache Key를 타면서 빠르게 응답
```

## 최종 조치 순서

```text
1. DevTools 또는 curl로 느린 URL/빠른 URL의 응답 헤더 비교
2. CDN에 Cache Key 기준 확인
3. CDN 로그에서 ver=5.035555555의 HIT/MISS/REFRESH/BYPASS 확인
4. 해당 URL만 Purge 후 재측정
5. Purge 후 첫 요청만 느리고 두 번째부터 빠르면 Cache Miss/Origin Fetch 문제로 확정
6. Purge 후에도 매번 느리면 CDN Cache Rule 또는 Origin 응답 문제 확인
7. 장기적으로 파일명 hash + long TTL 구조로 전환
```

핵심은 **`ver=5.035555555` 값 자체가 느리게 만드는 것이 아니라, 그 값이 만들어낸 CDN 캐시 객체 또는 캐시 정책 상태가 느린 경로를 타고 있는 것**입니다.

## 용어 정정

질문에서 말한 `TTBF`는 일반적으로 **`TTFB(Time To First Byte)`**를 의미합니다. TTFB는 브라우저가 요청을 시작한 뒤 서버 또는 캐시로부터 **첫 번째 바이트를 받을 때까지의 시간**이며, DNS/TCP/TLS 연결 시간과 서버 응답 대기 시간이 포함됩니다. ([MDN 웹 문서](https://developer.mozilla.org/en-US/docs/Glossary/Time_to_first_byte?utm_source=chatgpt.com "Time to First Byte (TTFB) - Glossary - MDN Web Docs"))

## CDN 원인으로 TTFB가 길어지는 대표 경우

|구분|원인|현상|
|---|---|---|
|1|CDN Cache Miss|CDN Edge에 캐시가 없어 Origin으로 요청 전달|
|2|Expired/Revalidation|캐시는 있으나 TTL 만료로 Origin 재검증|
|3|QueryString Cache Key 분리|`?ver=...` 값별로 다른 캐시 객체 생성|
|4|Cache Bypass|쿠키, 인증 헤더, Cache-Control 정책 때문에 캐시 우회|
|5|Origin Fetch 지연|CDN이 Origin 응답을 기다림|
|6|Edge 지역별 캐시 편차|특정 CDN Edge만 느림|
|7|Purge 후 Cold Cache|Purge 직후 첫 요청이 Origin으로 감|
|8|압축/변환 처리 지연|CDN 또는 Origin에서 gzip/br 변환 지연|
|9|WAF/Bot/보안 정책|CDN Edge에서 보안 검사로 지연|
|10|Origin Shield/Tiered Cache 지연|Edge → Shield → Origin 경로에서 대기|

## 1. CDN Cache Miss

가장 흔한 원인입니다. CDN Edge에 해당 리소스가 없으면 CDN은 Origin 서버에서 파일을 가져와야 합니다. Cloudflare 기준으로 `MISS`는 리소스가 캐시에 없어 Origin Web Server에서 제공되었다는 의미입니다. ([Cloudflare Docs](https://developers.cloudflare.com/cache/concepts/cache-responses/?utm_source=chatgpt.com "Cloudflare cache responses"))

```text
브라우저
→ CDN Edge
→ 캐시 없음
→ Origin 요청
→ Origin 응답 대기
→ CDN 응답 시작
→ TTFB 증가
```

### 대책

```text
1. 해당 URL의 CDN HIT/MISS 로그 확인
2. 정적 JS/CSS/Image 경로에 CDN TTL 적용
3. 배포 직후 CDN Preload 또는 Cache Priming 수행
4. 자주 호출되는 중요 파일은 Purge 후 즉시 Preload
```

## 2. TTL 만료 후 Origin 재검증

CDN에 캐시 객체가 있어도 TTL이 만료되면 CDN은 Origin에 재검증하거나 새 파일을 가져올 수 있습니다. `Cache-Control: max-age=N`은 응답이 생성된 뒤 N초 동안 fresh 상태로 재사용 가능하다는 의미입니다. ([MDN 웹 문서](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control?utm_source=chatgpt.com "Cache-Control header - HTTP - MDN Web Docs"))

```text
캐시 객체 존재
→ TTL 만료
→ CDN이 Origin에 If-Modified-Since / If-None-Match 재검증
→ Origin 응답 지연
→ TTFB 증가
```

### 대책

정적 파일이면 아래처럼 장기 캐시를 검토합니다.

```http
Cache-Control: public, max-age=31536000, immutable
```

단, `immutable` 또는 장기 TTL을 쓰려면 **파일 내용 변경 시 URL도 반드시 변경**되어야 합니다. MDN은 `max-age`, `s-maxage`, `stale-while-revalidate` 같은 Cache-Control 지시어를 통해 캐시 신선도와 재검증 동작을 제어할 수 있다고 설명합니다. ([MDN 웹 문서](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Cache-Control?utm_source=chatgpt.com "Cache-Control - HTTP - MDN Web Docs"))

## 3. QueryString 때문에 Cache Key가 분리되는 경우

현재 이슈와 가장 관련이 큽니다.

```text
selectDigtlKbcGoodsListMng.js
selectDigtlKbcGoodsListMng.js?ver=5.035555555
selectDigtlKbcGoodsListMng.js?ver=5.035555556
```

CDN이 QueryString을 Cache Key에 포함하면 위 3개는 서로 다른 캐시 객체로 처리될 수 있습니다. Akamai 문서에서도 Query String Parameter를 Cache Key에 포함하거나 제외하는 정책을 설정할 수 있으며, 기본적으로 full query string이 Cache Key에 포함될 수 있다고 설명합니다. ([Akamai TechDocs](https://techdocs.akamai.com/property-mgr/docs/cache-key-query-param?utm_source=chatgpt.com "Cache Key Query Parameters"))

### 대책

```text
1. CDN Cache Key에 ver 파라미터가 포함되는지 확인
2. ver가 단순 cache-busting 용도라면 Cache Key 정책 정리
3. 불필요한 QueryString은 Cache Key에서 제외 검토
4. 장기적으로 QueryString 버전보다 파일명 해시 방식 권장
```

권장 구조:

```text
기존:
selectDigtlKbcGoodsListMng.js?ver=5.035555555

개선:
selectDigtlKbcGoodsListMng.5f3a9c2.js
```

## 4. Cache Bypass 정책

CDN이 정적 파일을 캐시하지 않고 매번 Origin으로 보내면 TTFB가 길어질 수 있습니다.

대표 원인:

```text
Cache-Control: no-cache
Cache-Control: no-store
Pragma: no-cache
Authorization 헤더 존재
Cookie 기준 Bypass 정책
Set-Cookie 응답
Vary 헤더 과다
CDN Rule에서 특정 경로 Bypass
```

Cloudflare 문서 기준으로 `BYPASS`, `MISS`, `EXPIRED`, `REVALIDATED` 같은 상태는 캐시 사용 여부와 Origin 조회 여부를 판단하는 데 사용됩니다. ([Cloudflare Docs](https://developers.cloudflare.com/cache/concepts/cache-responses/?utm_source=chatgpt.com "Cloudflare cache responses"))

### 대책

```text
1. JS/CSS/Image 정적 경로에서 Cookie 제거 가능 여부 검토
2. 정적 파일 응답에 Set-Cookie가 붙는지 확인
3. Authorization 헤더가 정적 리소스 요청에 붙는지 확인
4. CDN Rule에서 /js/** 경로가 Cache Bypass인지 확인
5. Vary: * 또는 과도한 Vary 헤더 제거 검토
```

## 5. Origin Fetch 지연

CDN 원인처럼 보이지만 실제로는 CDN이 Origin 응답을 기다리는 경우입니다.

```text
CDN MISS
→ Origin 요청
→ Origin Nginx/WAS/Storage 응답 지연
→ CDN도 응답 시작 불가
→ 브라우저 TTFB 증가
```

### Origin에서 확인할 것

```text
1. Nginx access log의 request_time
2. upstream_response_time
3. 해당 JS 파일 직접 접근 시 응답 시간
4. 파일 존재 여부
5. 정적 파일이 WAS로 라우팅되는지 여부
6. gzip/br 압축을 매번 동적으로 수행하는지 여부
7. Disk I/O 지연 여부
```

### 대책

```text
1. 정적 파일은 WAS가 아니라 Nginx 또는 Object Storage에서 직접 서빙
2. gzip/br 사전 압축 파일 사용
3. Origin의 정적 파일 경로 rewrite/fallback 제거
4. CDN Origin Shield 사용 검토
5. Origin 서버 성능 지표와 CDN MISS 시간 비교
```

## 6. 특정 CDN Edge만 느린 경우

사용자 위치 또는 DNS에 따라 다른 CDN Edge로 접속할 수 있습니다. 특정 Edge에만 캐시가 없거나, 해당 Edge와 Origin 간 네트워크가 느리면 일부 사용자만 TTFB가 길어질 수 있습니다.

### 대책

```text
1. CDN 로그에서 Edge POP별 응답 시간 확인
2. 느린 요청의 Edge 서버 식별
3. 동일 URL을 여러 지역에서 측정
4. 특정 POP에서만 MISS/EXPIRED 반복 여부 확인
5. CDN 업체에 POP 단위 장애 또는 라우팅 이슈 확인 요청
```

## 7. Purge 이후 Cold Cache

CDN Purge를 하면 기존 캐시 객체가 제거됩니다. 이후 첫 요청은 Origin으로 가기 때문에 첫 사용자 요청의 TTFB가 길어질 수 있습니다.

```text
Purge
→ Edge 캐시 삭제
→ 첫 요청 MISS
→ Origin Fetch
→ TTFB 증가
→ 이후 요청 HIT이면 빨라짐
```

### 대책

```text
1. Purge 후 주요 JS/CSS 파일 Preload
2. 배포 자동화에 CDN Preload 단계 추가
3. 전체 Purge보다 변경 파일 단위 Purge 사용
4. QueryString 포함 URL까지 정확히 Purge
```

## 8. 압축 처리 지연

CDN이나 Origin이 JS 파일을 요청 시점마다 gzip 또는 brotli로 압축하면 TTFB가 늘어날 수 있습니다.

### 확인 헤더

```http
Content-Encoding: gzip
Content-Encoding: br
Vary: Accept-Encoding
```

### 대책

```text
1. JS 파일 사전 압축
2. Origin에서 gzip_static 또는 brotli_static 검토
3. CDN에서 압축 캐시 객체가 별도로 저장되는지 확인
4. Accept-Encoding별 Cache Key 분리 여부 확인
```

## 9. CDN 보안 기능으로 인한 지연

WAF, Bot Detection, DDoS Protection, Header Inspection이 정적 파일 경로에도 과도하게 적용되면 TTFB가 증가할 수 있습니다.

### 대책

```text
1. /js/**, /css/**, /images/** 경로에 보안 룰이 과도하게 적용되는지 확인
2. 정적 파일 경로는 필요한 최소 보안 룰만 적용
3. WAF 로그에서 해당 URL 검사 시간 확인
4. Bot Challenge 또는 JavaScript Challenge 적용 여부 확인
```

## 10. Origin Shield 또는 Tiered Cache 경로 지연

CDN 구성이 아래처럼 되어 있으면 Edge가 바로 Origin으로 가지 않고 중간 Shield 또는 상위 캐시를 거칠 수 있습니다.

```text
User
→ Edge POP
→ Regional Cache / Origin Shield
→ Origin
```

이 구조는 일반적으로 Origin 부하를 줄이는 데 유리하지만, 특정 계층에서 지연이 생기면 TTFB가 늘어날 수 있습니다.

### 대책

```text
1. Origin Shield 사용 여부 확인
2. Shield POP 위치 확인
3. Edge → Shield, Shield → Origin 시간 분리 확인
4. 특정 Shield 구간 장애 여부 확인
```

## CDN에 확인 요청할 항목

아래 내용은 CDN 운영사 또는 인프라팀에 그대로 요청해도 됩니다.

```text
1. 해당 URL의 Cache Key가 무엇인지 확인
2. QueryString 전체가 Cache Key에 포함되는지 확인
3. ver 파라미터가 Cache Key에 포함되는지 확인
4. 해당 URL의 최근 요청이 HIT/MISS/EXPIRED/BYPASS 중 무엇인지 확인
5. Edge POP별 응답 시간과 HIT Ratio 확인
6. 해당 URL만 TTL이 다르게 적용되는지 확인
7. Origin Cache-Control을 따르는지, CDN TTL Override가 있는지 확인
8. 해당 URL 객체가 일부 Edge에서만 stale 상태인지 확인
9. Purge 이력이 있는지 확인
10. Purge 시 QueryString 포함 URL까지 제거되는지 확인
11. Origin Fetch 시간이 긴지 확인
12. WAF/Bot/보안 룰이 정적 JS 경로에도 적용되는지 확인
13. gzip/br 압축이 Edge에서 동적으로 수행되는지 확인
14. Origin Shield 또는 Tiered Cache 구간에서 지연이 있는지 확인
```

## 확인해야 할 응답 헤더

느린 URL과 빠른 URL을 반드시 비교해야 합니다.

```http
Cache-Control
Age
Expires
ETag
Last-Modified
Content-Length
Content-Encoding
Vary
X-Cache
CF-Cache-Status
Via
Server-Timing
```

판단 기준:

|헤더|판단|
|---|---|
|`X-Cache: HIT`|CDN 캐시 응답 가능성 높음|
|`X-Cache: MISS`|Origin Fetch 가능성|
|`CF-Cache-Status: HIT`|Cloudflare 캐시 적중|
|`CF-Cache-Status: MISS`|Origin에서 가져옴|
|`CF-Cache-Status: EXPIRED`|캐시 만료 후 Origin 재조회|
|`Age` 있음|캐시에서 제공된 응답 가능성|
|`Age` 없음 또는 0|MISS/BYPASS/새 응답 가능성|
|`Cache-Control`에 `max-age` 없음|캐시 신선도 불명확|
|`Vary` 과다|Cache Key 분산 가능성|

## 진단용 curl 예시

```bash
curl -I -H "Cache-Control: no-cache" "https://도메인/js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555555"
curl -I "https://도메인/js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555555"
curl -I "https://도메인/js/go/kbc/selectDigtlKbcGoodsListMng.js"
curl -I "https://도메인/js/go/kbc/selectDigtlKbcGoodsListMng.js?ver=5.035555556"
```

확인 포인트:

```text
1. no-cache 요청에서만 느린가?
2. 동일 URL 반복 호출 시 두 번째부터 빨라지는가?
3. QueryString 제거 URL과 헤더가 다른가?
4. ver 변경 URL과 Cache-Control/Age/X-Cache가 다른가?
5. Content-Length가 동일한가?
```

## 현상별 판단표

|현상|가능성|
|---|---|
|첫 요청만 느리고 두 번째부터 빠름|CDN MISS 후 HIT|
|매번 느림|Cache Bypass 또는 TTL 만료 재검증 반복|
|특정 `ver`만 느림|QueryString Cache Key 객체 문제|
|QueryString 제거 시 빠름|QueryString 포함 Cache Key 또는 Rule 차이|
|`Age`가 계속 0|캐시 미적중 또는 재검증 반복|
|`X-Cache: TCP_MISS` 반복|Origin Fetch 반복|
|`X-Cache: TCP_HIT`인데 느림|Edge 자체 지연, 네트워크, 보안 처리 가능성|
|`Content Download`는 짧고 `Waiting`만 김|CDN/Origin 응답 대기 문제|

## 최종 대책 우선순위

```text
1. 느린 URL과 빠른 URL의 응답 헤더 비교
2. CDN Cache Key에 QueryString 포함 여부 확인
3. 해당 URL의 HIT/MISS/EXPIRED/BYPASS 로그 확인
4. Origin Fetch 시간 확인
5. 해당 URL 단위 CDN Purge 후 재측정
6. Purge 후 Preload 수행
7. 정적 파일에 명확한 Cache-Control max-age 적용
8. QueryString 버전 방식에서 파일명 해시 방식으로 전환
9. 정적 파일 경로의 WAF/Bot/Bypass Rule 정리
10. 배포 시 CDN Purge/Preload 자동화
```

## 핵심 요약

CDN 때문에 TTFB가 길어지는 핵심 구조는 아래입니다.

```text
브라우저가 느린 것이 아니라,
CDN이 즉시 응답하지 못하고
Origin 조회, 재검증, Cache Miss, Bypass, Edge 처리 지연을 거치기 때문에
첫 바이트 응답이 늦어지는 것
```

현재처럼 특정 JS의 특정 `ver` 값에서만 느리다면 **CDN Cache Key + QueryString + TTL 만료/재검증 + Origin Fetch** 조합을 최우선으로 확인하는 것이 맞습니다.

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
