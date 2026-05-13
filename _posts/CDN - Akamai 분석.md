
# curl을 이용한 Akamai CDN 분석

## 1. 분석 목표

Akamai CDN 분석은 `curl`로 아래 6가지를 확인하는 방식이 가장 실무적입니다.

|분석 항목|확인 값|판단 목적|
|---|---|---|
|CDN 경유 여부|`Server`, `Via`, `X-Cache`, Akamai 디버그 헤더|요청이 Akamai Edge를 타는지 확인|
|캐시 HIT/MISS|`X-Cache`, `X-Cache-Remote`, `Age`|CDN 캐시에서 응답했는지 확인|
|캐시 가능 여부|`X-Check-Cacheable`, `Cache-Control`, `Expires`|해당 URL이 캐시 가능한지 확인|
|캐시 키|`X-Cache-Key`, `X-True-Cache-Key`|쿼리스트링/Host/경로가 캐시 키에 어떻게 반영되는지 확인|
|Origin 재검증|`TCP_REFRESH_HIT`, `TCP_REFRESH_MISS`, Origin 로그|만료 객체 처리 방식 확인|
|성능|`time_starttransfer`, `time_total`|CDN 응답 지연/Origin 지연 구분|
|Akamai는 `Pragma` 요청 헤더를 통해 `X-Cache`, `X-Check-Cacheable`, `X-Cache-Key`, `X-True-Cache-Key`, `X-Akamai-Request-ID` 같은 디버그 응답 헤더를 받을 수 있다고 문서화하고 있습니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers "Pragma headers"))|||

## 2. 기본 헤더 확인

```bash
curl -I https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

확인할 값:

```http
HTTP/2 200
Server: AkamaiGHost
Cache-Control: public, max-age=86400
Age: 120
```

|헤더|의미|판단|
|---|---|---|
|`Server: AkamaiGHost`|Akamai Edge 응답 가능성|Akamai 경유 흔적|
|`Cache-Control`|Origin/CDN 캐시 정책|`public`, `max-age`, `s-maxage` 확인|
|`Age`|캐시 객체가 머문 시간|반복 요청 시 증가하면 캐시 응답 가능성 높음|
|`ETag`, `Last-Modified`|재검증 기준|만료 후 304/재검증에 사용|
|단, 일반 `curl -I`만으로는 Akamai 캐시 상태가 항상 노출되지 않을 수 있습니다. 정확한 분석은 Akamai 디버그용 `Pragma` 헤더를 붙여 확인하는 것이 좋습니다.|||

## 3. Akamai Pragma 디버그 헤더 테스트

### 3.1 가장 많이 쓰는 명령

```bash
curl.exe -I -H "Pragma: akamai-x-cache-on, akamai-x-cache-remote-on, akamai-x-check-cacheable, akamai-x-get-cache-key, akamai-x-get-true-cache-key, akamai-x-get-request-id, akamai-x-serial-no" https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

예상 응답:

```http
X-Cache: TCP_HIT from a23-45-67-89.deploy.akamaitechnologies.com (AkamaiGHost/...)
X-Cache-Remote: TCP_HIT from ...
X-Check-Cacheable: YES
X-Cache-Key: /L/12345/86400/cdn.example.com/css/common.css
X-True-Cache-Key: /cdn.example.com/css/common.css
X-Akamai-Request-ID: abc123...
X-Serial: 12345
```

|Pragma 값|응답 헤더|용도|
|---|---|---|
|`akamai-x-cache-on`|`X-Cache`|Edge 서버의 캐시 처리 결과 확인|
|`akamai-x-cache-remote-on`|`X-Cache-Remote`|상위/Parent 계층 캐시 처리 결과 확인|
|`akamai-x-check-cacheable`|`X-Check-Cacheable`|Akamai 설정 기준 캐시 가능 여부 확인|
|`akamai-x-get-cache-key`|`X-Cache-Key`|CP Code, TTL 등이 포함된 캐시 키 확인|
|`akamai-x-get-true-cache-key`|`X-True-Cache-Key`|실제 해싱에 쓰이는 true cache key 확인|
|`akamai-x-get-request-id`|`X-Akamai-Request-ID`|Akamai 로그 추적용 요청 ID 확인|
|`akamai-x-serial-no`|`X-Serial`|요청에 사용된 설정 serial 확인|
|Akamai 공식 문서 기준 `akamai-x-cache-on`은 Edge 응답이 어떻게 제공됐는지 반환하고, `akamai-x-check-cacheable`은 캐시 가능 여부를 `YES/NO`로 반환하며, `akamai-x-get-cache-key`와 `akamai-x-get-true-cache-key`는 캐시 키 확인에 사용됩니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers "Pragma headers"))|||

## 4. `X-Cache` 결과 해석

|`X-Cache` 값|의미|실무 판단|
|---|---|---|
|`TCP_HIT`|객체가 fresh 상태로 캐시에 있고 disk cache에서 제공|정상 캐시 HIT|
|`TCP_MEM_HIT`|객체가 memory cache에서 제공|가장 빠른 캐시 HIT 가능성|
|`TCP_MISS`|캐시에 없어 Origin에서 가져옴|최초 요청 또는 캐시 미적용|
|`TCP_REFRESH_HIT`|stale 객체를 Origin에 재검증했고 성공적으로 캐시 객체 사용|재검증 발생, Origin 확인 있음|
|`TCP_REFRESH_MISS`|stale 객체 재검증 결과 Origin에서 새 객체 취득|캐시 만료 후 새 버전 수신|
|`TCP_REFRESH_FAIL_HIT`|Origin 재검증 실패, stale 객체 제공|Origin 장애/연결 실패 가능성|
|`TCP_IMS_HIT`|클라이언트의 `If-Modified-Since` 요청에 대해 fresh 객체 제공|브라우저 재검증 조건 관련|
|`TCP_NEGATIVE_HIT`|404 등 부정 응답이 캐시되어 HIT|404/오류 응답 캐시 여부 확인 필요|
|Akamai 문서는 `TCP_HIT`, `TCP_MISS`, `TCP_REFRESH_HIT`, `TCP_REFRESH_MISS`, `TCP_REFRESH_FAIL_HIT`, `TCP_IMS_HIT`, `TCP_NEGATIVE_HIT`, `TCP_MEM_HIT` 등의 의미를 구분해 설명합니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers "Pragma headers"))|||

## 5. MISS → HIT 반복 분석

같은 URL을 2~3회 반복 호출합니다.

```bash
URL="https://cdn.example.com/img/banner.png"
curl -I -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" "$URL"
curl -I -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" "$URL"
curl -I -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" "$URL"
```

정상 패턴:

|호출|예상 결과|해석|
|--:|---|---|
|1회차|`TCP_MISS`, `X-Check-Cacheable: YES`|Edge에 없어 Origin에서 취득|
|2회차|`TCP_HIT` 또는 `TCP_MEM_HIT`|Akamai Edge 캐시 응답|
|3회차|`TCP_HIT` + `Age` 증가|캐시 유지 확인|
|문제 패턴:|||
|패턴|가능 원인|확인 방법|
|---|---|---|
|항상 `TCP_MISS`|캐시 정책 미적용, TTL 없음, 쿠키/헤더 조건, 쿼리스트링 분산|`X-Check-Cacheable`, `Cache-Control`, `X-True-Cache-Key` 확인|
|`X-Check-Cacheable: NO`|Akamai 룰 또는 Origin 헤더상 캐시 불가|Property Manager 캐시 룰, `no-store`, `private` 확인|
|`TCP_REFRESH_HIT` 반복|TTL이 너무 짧거나 재검증 조건 과다|`Cache-Control`, Akamai TTL, Origin 로그 확인|
|`TCP_NEGATIVE_HIT`|404/오류 응답 캐시|오류 응답 캐시 TTL 정책 확인|

## 6. no-cache 요청 재현

브라우저 강력 새로고침이나 DevTools 조건에 따라 `Cache-Control: no-cache`, `Pragma: no-cache` 요청이 발생할 수 있습니다.

```bash
curl -I \
  -H "Cache-Control: no-cache" \
  -H "Pragma: no-cache, akamai-x-cache-on, akamai-x-check-cacheable, akamai-x-get-cache-key" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

해석:

|결과|의미|조치|
|---|---|---|
|`TCP_HIT`|클라이언트 no-cache에도 Edge 캐시 사용|운영 정책상 문제 없을 수 있음|
|`TCP_REFRESH_HIT`|Origin 재검증 후 캐시 객체 제공|Origin 로그 증가 여부 확인|
|`TCP_REFRESH_MISS`|Origin에서 새 객체 취득|TTL/ETag/Last-Modified 확인|
|`TCP_MISS`|Origin까지 요청|반복 발생 시 Origin 부하 가능성 검토|
|주의: `Pragma` 헤더 하나에 `no-cache`와 Akamai 디버그 값을 같이 넣을 때는 쉼표로 나열합니다. 다만 실제 운영 정책에 따라 클라이언트의 `no-cache` 요청을 Akamai가 어떻게 처리하는지는 Property 설정과 Origin 헤더에 따라 달라질 수 있으므로 응답 헤더와 Origin access log를 같이 봐야 합니다.|||

## 7. GET 방식으로 실제 본문까지 확인

`HEAD` 요청과 실제 `GET` 요청의 처리 결과가 다를 수 있으므로, 중요한 리소스는 `GET`으로도 확인합니다.

```bash
curl -s -D headers.txt -o body.bin \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable, akamai-x-get-cache-key" \
  https://cdn.example.com/js/app.js
cat headers.txt
sha256sum body.bin
```

CDN과 Origin 파일 비교:

```bash
curl -s -o cdn.bin https://cdn.example.com/js/app.js
curl -s -o origin.bin https://origin.example.com/js/app.js
sha256sum cdn.bin origin.bin
```

|결과|판단|
|---|---|
|해시 동일|CDN과 Origin 파일 동일|
|해시 다름|Akamai에 오래된 객체가 남아 있거나 경로가 다를 수 있음|
|CDN만 과거 파일|Purge 또는 파일명 버전 관리 필요|

## 8. 쿼리스트링 캐시 키 분석

정적 리소스 버전 관리에서 `?ver=...`를 쓰는 경우 캐시 키 반영 여부가 중요합니다.

```bash
curl -I \
  -H "Pragma: akamai-x-cache-on, akamai-x-get-true-cache-key, akamai-x-get-cache-key" \
  "https://cdn.example.com/css/import.css?ver=1"
curl -I \
  -H "Pragma: akamai-x-cache-on, akamai-x-get-true-cache-key, akamai-x-get-cache-key" \
  "https://cdn.example.com/css/import.css?ver=2"
```

판단:

|비교 결과|의미|
|---|---|
|`X-True-Cache-Key`가 다름|쿼리스트링이 캐시 키에 포함됨|
|`X-True-Cache-Key`가 같음|쿼리스트링이 캐시 키에서 제외/무시됨|
|`ver`가 바뀌어도 같은 파일 응답|CDN 또는 Origin 라우팅 정책 확인 필요|
|Akamai 문서 기준 `X-True-Cache-Key`는 MD5 해싱에 사용되는 true cache key를 반환하며, scheme이나 HTTP method는 포함하지 않습니다. `X-Cache-Key`는 요청에 사용된 캐시 키와 CP Code, TTL 정보를 포함합니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers "Pragma headers"))||

## 9. 쿠키 영향 분석

정적 리소스 요청에 쿠키가 붙으면 캐시 정책이 달라질 수 있습니다. 실제 브라우저 요청을 재현하려면 `Cookie`를 넣어 비교합니다.

```bash
# 쿠키 없는 요청
curl -I \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable, akamai-x-get-true-cache-key" \
  https://cdn.example.com/js/app.js
# 쿠키 있는 요청
curl -I \
  -H "Cookie: JSESSIONID=test123" \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable, akamai-x-get-true-cache-key" \
  https://cdn.example.com/js/app.js
```

|결과|의미|
|---|---|
|쿠키 유무와 관계없이 `TCP_HIT`|정적 리소스 캐시 정책 양호|
|쿠키 요청만 `TCP_MISS`|쿠키 기반 bypass 또는 캐시 키 분리 가능성|
|`X-True-Cache-Key`가 달라짐|쿠키/헤더가 캐시 키에 반영될 가능성|
|운영 커머스 사이트에서는 정적 리소스 전용 도메인을 별도로 두고 쿠키가 붙지 않게 하는 구성이 유리합니다.||

## 10. 압축 응답 확인

JS/CSS는 압축 여부도 봐야 합니다.

```bash
curl -I \
  -H "Accept-Encoding: br, gzip" \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" \
  https://cdn.example.com/js/app.js
```

확인:

|헤더|정상 예|
|---|---|
|`Content-Encoding`|`br` 또는 `gzip`|
|`Vary`|`Accept-Encoding`|
|`X-Cache`|`TCP_HIT` 또는 `TCP_MISS`|
|압축 조건에 따라 캐시 객체가 분리될 수 있으므로 `Accept-Encoding`을 넣은 경우와 넣지 않은 경우를 비교하는 것이 좋습니다.||

## 11. 응답 시간 측정

```bash
curl -o /dev/null -s \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" \
  -w "status=%{http_code}\nnamelookup=%{time_namelookup}\nconnect=%{time_connect}\ntls=%{time_appconnect}\nstarttransfer=%{time_starttransfer}\ntotal=%{time_total}\n" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

|항목|의미|해석|
|---|---|---|
|`time_namelookup`|DNS 조회 시간|DNS/CNAME 지연 확인|
|`time_connect`|TCP 연결 시간|Edge까지의 네트워크 지연|
|`time_appconnect`|TLS 핸드셰이크 시간|HTTPS 연결 비용|
|`time_starttransfer`|첫 바이트까지 시간|CDN HIT/MISS, Origin 지연 영향|
|`time_total`|전체 완료 시간|파일 크기, 압축, 네트워크 영향|
|HIT인데 `time_starttransfer`가 높으면 Edge 위치, TLS, 네트워크, 클라이언트 경로 문제를 보고, MISS에서만 높으면 Origin 응답 지연 가능성을 우선 확인합니다.|||

## 12. 특정 IP/Origin으로 우회 테스트

DNS 변경 없이 특정 IP로 요청하려면 `--resolve`를 사용합니다.

```bash
curl -I \
  --resolve cdn.example.com:443:203.0.113.10 \
  -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

용도:

|용도|설명|
|---|---|
|신규 CDN 설정 검증|DNS 전환 전 특정 Edge/Origin IP로 확인|
|Origin 직접 비교|Host 헤더는 유지하고 IP만 Origin으로 지정|
|특정 경로 문제 분석|네트워크/DNS 경로 차이 확인|

## 13. Enhanced Debug 방식

최신 Akamai 설정에서는 기존 `Pragma` 방식보다 **Enhanced Debug**를 권장할 수 있습니다. Akamai 문서에 따르면 Enhanced Debug는 기존 Pragma debugging을 대체하는 더 안전하고 기능이 많은 방식이며, 고객이 정의한 secret key로 생성한 시간 제한 인증 토큰을 `Akamai-Debug` 요청 헤더에 넣어 사용합니다. ([Akamai TechDocs](https://techdocs.akamai.com/property-mgr/docs/enhanced-debug "Enhanced Debug"))  
예시 형식:

```bash
curl -I \
  -H "Akamai-Debug: <AUTH_TOKEN> cache" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

Enhanced Debug의 `cache` 옵션은 기존 Pragma의 `akamai-x-cache-on`, `akamai-x-cache-remote-on`, `akamai-x-check-cacheable`, `akamai-x-get-cache-key`, `akamai-x-get-true-cache-key`에 대응하며, 응답으로 `X-Cache`, `X-Cache-Remote`, `X-Check-Cacheable`, `X-Cache-Key`, `X-True-Cache-Key`, `Akamai-Cache-Status` 등의 캐시 관련 정보를 받을 수 있습니다. ([Akamai TechDocs](https://techdocs.akamai.com/property-mgr/docs/enhanced-debug "Enhanced Debug"))

|방식|장점|주의|
|---|---|---|
|`Pragma` 디버그|간단한 curl 테스트 가능|설정에 따라 비활성화될 수 있음|
|`Akamai-Debug` Enhanced Debug|보안성/정보량 우수|토큰 생성과 Property 설정 필요|
|Akamai Ion 문서도 Enhanced Debug가 Akamai 서버를 통과하는 요청을 디버깅하는 방식이며, 기존 Pragma debugging 기능을 포함하면서 더 안전하고 추가 정보를 제공한다고 설명합니다. ([Akamai TechDocs](https://techdocs.akamai.com/ion/docs/enhanced-debug "Enhanced Debug"))|||

## 14. 실무 분석 순서

|순서|명령|목적|
|--:|---|---|
|1|`curl -I URL`|기본 응답 헤더 확인|
|2|`curl -I -H "Pragma: akamai-x-cache-on..." URL`|Akamai 캐시 상태 확인|
|3|동일 URL 3회 반복|`TCP_MISS → TCP_HIT` 확인|
|4|`Cache-Control: no-cache` 추가|강제 재검증 상황 확인|
|5|`X-True-Cache-Key` 비교|쿼리스트링/쿠키/헤더 캐시 키 영향 확인|
|6|`curl -s -o` + `sha256sum`|CDN/Origin 파일 동일성 확인|
|7|`curl -w`|TTFB/총 응답 시간 측정|
|8|Origin access log 확인|HIT 시 Origin 요청 감소 검증|

## 15. 운영 점검용 명령 모음

```bash
# 1. 기본 헤더
curl -I https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css

# 2. Akamai 캐시 상태
curl -I \
  -H "Pragma: akamai-x-cache-on, akamai-x-cache-remote-on, akamai-x-check-cacheable, akamai-x-get-cache-key, akamai-x-get-true-cache-key, akamai-x-get-request-id" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css

# 3. 반복 HIT 확인
for i in 1 2 3; do
  curl -I \
    -H "Pragma: akamai-x-cache-on, akamai-x-check-cacheable" \
    https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css | egrep -i "HTTP/|X-Cache|X-Check-Cacheable|Age|Cache-Control"
done

# 4. no-cache 요청 재현
curl -I \
  -H "Cache-Control: no-cache" \
  -H "Pragma: no-cache, akamai-x-cache-on, akamai-x-check-cacheable" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
  
# 5. 응답 시간 측정
curl -o /dev/null -s \
  -w "status=%{http_code}\nstarttransfer=%{time_starttransfer}\ntotal=%{time_total}\n" \
  https://focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

## 16. 결론

curl로 Akamai CDN을 분석할 때 핵심은 **`X-Cache`와 `X-Check-Cacheable`을 함께 보는 것**입니다. `TCP_HIT`이면 Edge 캐시 응답, `TCP_MISS`이면 Origin 취득, `TCP_REFRESH_HIT/MISS`이면 만료 객체 재검증 흐름입니다. 여기에 `X-True-Cache-Key`로 쿼리스트링/쿠키/헤더에 따른 캐시 키 분산을 확인하고, Origin access log로 실제 Origin 호출 감소 여부를 검증하면 됩니다. 최신 설정에서는 가능하면 `Akamai-Debug` 기반 Enhanced Debug도 함께 검토하는 것이 좋습니다.

```shell
HTTP/1.1 200 OK
Content-Type: text/css
Content-Length: 18436
Server: nginx
Last-Modified: Wed, 15 Apr 2026 07:40:32 GMT
ETag: "69df40f0-4804"
Access-Control-Allow-Origin: *
Accept-Ranges: bytes
Date: Fri, 24 Apr 2026 01:21:24 GMT
X-Cache: TCP_HIT from a23-214-21-71.deploy.akamaitechnologies.com (AkamaiGHost/22.5.0-aaef44c942a33f2d231f7120051a5b09) (-)
X-Cache-Key: S/L/1652/1569501/7d/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
X-Cache-Key-Extended-Internal-Use-Only: S/L/1652/1569501/7d/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css vcd=17551
X-True-Cache-Key: /L/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css vcd=17551
X-Serial: 1652
Connection: keep-alive
X-Check-Cacheable: YES
X-Akamai-Request-ID: aebf8700
```

# curl 결과 라인별 해석

## 1. 전체 판단 요약

| 판단 항목     | 결과                                 | 의미                         |
| --------- | ---------------------------------- | -------------------------- |
| 응답 성공 여부  | `HTTP/1.1 200 OK`                  | CSS 파일 정상 응답               |
| CDN 경유 여부 | `X-Cache: TCP_HIT ... AkamaiGHost` | Akamai Edge CDN을 통해 응답     |
| 캐시 상태     | `TCP_HIT`                          | Origin이 아니라 Akamai 캐시에서 응답 |
| 캐시 가능 여부  | `X-Check-Cacheable: YES`           | Akamai 기준 캐시 가능한 객체        |
| 캐시 TTL 추정 | `X-Cache-Key ... /7d/...`          | Akamai 캐시 키상 TTL이 7일로 보임   |
| 리소스 종류    | `Content-Type: text/css`           | CSS 정적 리소스                 |
| 최종 결론     | 정상 CDN HIT                         | 현재 결과만 보면 CDN 캐시가 정상 동작 중  |
Akamai 공식 문서 기준 `X-Cache`는 Edge가 객체를 어떻게 제공했는지 보여주고, `TCP_HIT`는 캐시된 객체를 제공했다는 의미입니다. `X-Check-Cacheable`은 응답이 캐시 가능한지 `YES/NO`로 표시하며, `X-Cache-Key`는 요청에 사용된 캐시 키와 serial number, CP code, TTL 정보를 포함합니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers?utm_source=chatgpt.com "Pragma headers"))

## 2. 라인별 상세 해석

| 라인                                                                                                 | 의미                   | 이번 결과 해석                                                                                    |
| -------------------------------------------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------- |
| `HTTP/1.1 200 OK`                                                                                  | HTTP 응답 상태 코드        | 요청한 CSS 파일이 정상적으로 반환됨. `200`이므로 파일 존재 및 응답 성공                                               |
| `Content-Type: text/css`                                                                           | 응답 본문 MIME 타입        | 브라우저가 이 응답을 CSS 파일로 해석함                                                                     |
| `Content-Length: 18436`                                                                            | 응답 본문 크기             | CSS 파일 크기가 18,436 bytes. `HEAD` 요청이면 본문 없이 크기만 표시될 수 있음                                     |
| `Server: nginx`                                                                                    | 응답 서버 식별 정보          | Origin 또는 CDN이 전달한 서버 정보가 `nginx`로 표시됨. 단, 실제 응답은 아래 `X-Cache`상 Akamai Edge에서 캐시 HIT로 제공됨   |
| `Last-Modified: Wed, 15 Apr 2026 07:40:32 GMT`                                                     | 원본 리소스의 마지막 수정 시각    | 이 CSS 파일은 원본 기준 2026-04-15 16:40:32 KST에 마지막 수정된 것으로 보임. `Last-Modified`는 조건부 요청 재검증에 사용 가능 |
| `ETag: "69df40f0-4804"`                                                                            | 리소스 버전 식별자           | 브라우저/CDN이 이후 재검증 시 “내가 가진 파일과 서버 파일이 같은가”를 비교하는 데 사용 가능                                     |
| `Access-Control-Allow-Origin: *`                                                                   | CORS 허용 범위           | 모든 Origin에서 이 CSS 응답을 공유 가능. CSS/폰트/이미지 등 정적 리소스에서 자주 사용됨                                   |
| `Accept-Ranges: bytes`                                                                             | 부분 다운로드 지원           | 클라이언트가 `Range` 요청으로 파일 일부만 받을 수 있음. 대형 파일, 미디어, 다운로드 재개에 유용                                 |
| `Date: Fri, 24 Apr 2026 01:21:24 GMT`                                                              | 응답 생성/전달 시각          | 한국 시간 기준 2026-04-24 10:21:24 KST에 응답된 것으로 볼 수 있음                                            |
| `X-Cache: TCP_HIT from a23-214-21-71.deploy.akamaitechnologies.com ...`                            | Akamai Edge 캐시 처리 결과 | 핵심 라인. `TCP_HIT`이므로 Akamai Edge 캐시에 있던 객체가 반환됨. `a23-214-21-71...`는 응답한 Akamai Edge 서버 식별자  |
| `X-Cache-Key: S/L/1652/1569501/7d/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css` | Akamai 캐시 키          | Akamai가 이 객체를 캐시에서 찾을 때 사용한 키 정보. `7d`가 포함되어 있어 캐시 TTL이 7일로 설정된 것으로 해석 가능                   |
| `X-Cache-Key-Extended-Internal-Use-Only: ... vcd=17551`                                            | Akamai 내부 확장 캐시 키    | 내부 진단용 확장 정보. 일반 운영 판단에는 `X-Cache`, `X-Cache-Key`, `X-True-Cache-Key`를 우선 봄                 |
| `X-True-Cache-Key: /L/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css vcd=17551`   | 실제 캐시 해싱 기준 키        | Akamai가 실제 캐시 객체 식별에 쓰는 true cache key. 현재 URL은 쿼리스트링 없이 Host + Path 기준으로 캐시되는 것으로 보임       |
| `X-Serial: 1652`                                                                                   | Akamai 설정 serial 정보  | 이 요청에 적용된 Akamai 설정 버전/serial 식별값. `X-Cache-Key`의 `1652`와도 일치                               |
| `Connection: keep-alive`                                                                           | TCP 연결 유지            | 응답 후 연결을 바로 닫지 않고 재사용 가능하게 유지                                                               |
| `X-Check-Cacheable: YES`                                                                           | Akamai 기준 캐시 가능 여부   | 이 URL은 Akamai 설정상 캐시 가능한 객체로 판단됨                                                            |
| `X-Akamai-Request-ID: aebf8700`                                                                    | Akamai 요청 추적 ID      | Akamai 측 로그/지원 문의 시 요청 추적에 사용할 수 있는 ID                                                      |
`Last-Modified`는 원본 서버가 판단한 리소스 최종 수정 시각이며 조건부 요청의 검증자로 사용될 수 있고, `Access-Control-Allow-Origin`은 해당 응답이 어떤 Origin과 공유될 수 있는지 나타냅니다. `Content-Length`는 응답 본문 크기를 bytes 단위과 공유될 수 있는지 나타냅니다.

## 3. 가장 중요한 Akamai 라인 해석

### 3.1 `X-Cache: TCP_HIT`

```http
X-Cache: TCP_HIT from a23-214-21-71.deploy.akamaitechnologies.com (AkamaiGHost/22.5.0-aaef44c942a33f2d231f7120051a5b09) (-)
```

| 구간                      | 의미                      |
| ----------------------- | ----------------------- |
| `TCP_HIT`               | Akamai 캐시에서 바로 응답       |
| `from a23-214-21-71...` | 응답한 Akamai Edge 서버      |
| `AkamaiGHost/22.5.0...` | Akamai Edge 서버 소프트웨어 식별 |
| `(-)`                   | 추가 상태 정보 없음 또는 미표시      |
이 라인만 봐도 **Origin까지 가지 않고 Akamai CDN 캐시에서 응답했다**고 판단할 수 있습니다.

### 3.2 `X-Cache-Key`

```http
X-Cache-Key: S/L/1652/1569501/7d/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css
```

| 구간                                  | 추정 의미                 | 설명                              |
| ----------------------------------- | --------------------- | ------------------------------- |
| `S/L`                               | Akamai 내부 캐시 키 prefix | Akamai 내부 표현                    |
| `1652`                              | Serial                | 현재 적용된 Akamai 설정 serial과 일치     |
| `1569501`                           | CP Code로 추정           | Akamai 계약/속성 단위 과금·로그 분류 식별자 계열 |
| `7d`                                | TTL                   | 캐시 유지 시간이 7일로 설정된 것으로 해석 가능     |
| `focdn.gcdn.samsungsdscloud.com`    | Host                  | 요청한 CDN 도메인                     |
| `/css/common/swiper-bundle.min.css` | Path                  | 캐시 대상 CSS 경로                    |
Akamai 문서상 `X-Cache-Key`는 요청의 캐시 키와 serial number, CP code, TTL 정보를 포함합니다. 따라서 이 결과의 `/7d/`는 현재 객체가 7일 TTL 정책을 받는다는 강한 근거입니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers?utm_source=chatgpt.com "Pragma headers"))

### 3.3 `X-True-Cache-Key`

```http
X-True-Cache-Key: /L/focdn.gcdn.samsungsdscloud.com/css/common/swiper-bundle.min.css vcd=17551
```

| 항목                                  | 의미                                          |
| ----------------------------------- | ------------------------------------------- |
| `focdn.gcdn.samsungsdscloud.com`    | 캐시 키에 Host가 포함됨                             |
| `/css/common/swiper-bundle.min.css` | 캐시 키에 Path가 포함됨                             |
| 쿼리스트링 없음                            | 현재 요청 URL에는 쿼리스트링이 없거나 캐시 키에 표시되지 않음        |
| `vcd=17551`                         | Akamai 내부 진단/설정 관련 값으로 보이며 일반 운영 판단의 핵심은 아님 |
Akamai 문서상 `X-True-Cache-Key`는 MD5 해싱에 사용되는 true cache key이며, scheme이나 HTTP method는 포함하지 않습니다. ([Akamai TechDocs](https://techdocs.akamai.com/edge-diagnostics/docs/pragma-headers?utm_source=chatgpt.com "Pragma headers"))

## 4. 이 결과에서 특히 중요한 점

|체크 항목|결과|판단|
|---|---|---|
|Akamai CDN 경유|확인됨|`X-Cache`, `AkamaiGHost` 존재|
|캐시 HIT|확인됨|`TCP_HIT`|
|캐시 가능|확인됨|`X-Check-Cacheable: YES`|
|TTL|7일로 추정|`X-Cache-Key`에 `/7d/` 포함|
|Origin 호출 여부|이번 응답만 보면 Origin 미호출 가능성 높음|정확히는 Origin access log로 최종 확인|
|Cache-Control 헤더|현재 결과에는 없음|Akamai 정책 TTL로 캐시되고 있을 가능성|
|Age 헤더|현재 결과에는 없음|없어도 `TCP_HIT`이면 Akamai 캐시 응답 판단 가능|

## 5. 주의할 점

| 주의 항목                                    | 설명                                                              |
| ---------------------------------------- | --------------------------------------------------------------- |
| `Server: nginx`만 보고 Origin 응답으로 오해하면 안 됨 | `X-Cache: TCP_HIT ... AkamaiGHost`가 있으므로 CDN 캐시 응답으로 판단하는 것이 맞음 |
| `Age`가 없다고 CDN 캐시 미동작은 아님                | Akamai 디버그 헤더의 `TCP_HIT`가 더 직접적인 판단 근거                          |
| `Cache-Control`이 없어도 Akamai에서 캐시될 수 있음   | CDN Property 설정에서 TTL을 강제 지정하면 Origin 헤더와 별개로 캐시 가능             |
| 정확한 Origin 미호출 검증은 Origin access log 필요  | `TCP_HIT`상 Origin 미호출 가능성이 높지만, 운영 증빙은 access log와 함께 보는 것이 좋음  |

## 6. 최종 결론

현재 curl 결과는 **Akamai CDN이 정상적으로 동작하고 있고, 해당 CSS 파일은 Akamai Edge 캐시에서 `TCP_HIT`로 반환된 상태**입니다. 특히 `X-Check-Cacheable: YES`와 `X-Cache-Key`의 `/7d/`를 보면, 해당 리소스는 Akamai 설정상 캐시 가능하며 약 7일 TTL 정책을 받는 정적 CSS 리소스로 해석할 수 있습니다. 운영 점검 관점에서는 같은 URL을 2~3회 반복 호출하고, 동시에 Origin access log에 요청이 찍히지 않는지 확인하면 “CDN 캐시 HIT로 Origin 부하가 줄고 있다”는 증빙까지 가능합니다.