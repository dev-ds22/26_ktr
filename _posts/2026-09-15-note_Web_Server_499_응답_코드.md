---
layout: single
title: "Web_Server_499_응답_코드"
excerpt: "Web_Server_499_응답_코드"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-15"
last_modified_at: "2026-09-15 12:08:56 +0900"
mermaid: false
---
가장 간단하게는 아래처럼 추출하면 됩니다.

```bash
awk '$6 ~ /"POST/ && $9 == 499 {print}' access.log
```

일반적인 nginx/Apache combined log 형식이 다음처럼 되어 있을 때:

```text
192.168.0.10 - - [14/Sep/2026:12:10:00 +0900] "POST /api/order.do HTTP/1.1" 499 0
```

필드가 보통:

```text
$6 = "POST
$7 = /api/order.do
$8 = HTTP/1.1"
$9 = 499
```

이므로 위 명령이 맞습니다.

조금 더 명확하게 하려면:

```bash
awk '$6 == "\"POST" && $9 == 499 {print}' access.log
```

### 1. URI만 보고 싶다면

```bash
awk '$6 == "\"POST" && $9 == 499 {print $7}' access.log
```

### 2. URI별 발생 건수 집계

```bash
awk '$6 == "\"POST" && $9 == 499 {print $7}' access.log \
| sort \
| uniq -c \
| sort -nr
```

예:

```text
  27 /api/order.do
  13 /login.do
   4 /payment/confirm.do
```

### 3. 특정 날짜만

```bash
awk '$0 ~ /14\/Sep\/2026/ && $6 == "\"POST" && $9 == 499 {print}' access.log
```

특정 시간대:

```bash
awk '$0 ~ /14\/Sep\/2026:12:/ && $6 == "\"POST" && $9 == 499 {print}' access.log
```

현재 499 분석 목적이라면 **POST + 499 + request_time**까지 같이 보는 것이 가장 좋습니다. 로그 마지막 필드가 `$request_time`이라면:

```bash
awk '$6 == "\"POST" && $9 == 499 {
    print "time="$NF, "uri="$7, $0
}' access.log
```

만약 실제 `access.log` 한 줄을 보여주시면, 현재 포맷 기준으로 필드 위치를 정확히 맞춰드릴 수 있습니다.

## 4. 결론

`499`는 **Web Server/nginx가 클라이언트에게 반환하는 정상 HTTP 응답코드가 아닙니다.** nginx가 내부적으로 access log에 기록하는 비표준 코드로,

> **nginx가 응답을 완료하기 전에 클라이언트 측 연결이 먼저 종료됐다**

는 의미입니다. nginx 내부에서도 `499`는 `NGX_HTTP_CLIENT_CLOSED_REQUEST`로 정의되어 있습니다. 

따라서 **가끔 발생하는 499 자체는 일반적인 현상일 수 있습니다.** 사용자가 새로고침하거나 페이지를 이동하거나 브라우저가 불필요한 요청을 취소하면 정상적인 운영 환경에서도 발생할 수 있습니다. 그러나 **특정 URI에서 반복되거나, 50초·60초처럼 일정한 시간 후 발생하거나, 504와 함께 증가한다면 정상적인 단순 사용자 취소로 보면 안 됩니다.**

### 4-1. nginx에서 499가 기록되는 대표 상황

```text
Client
   ↓ request
nginx
   ↓
처리 중...
   ↓
Client가 연결 종료
   X
nginx
   ↓
access.log에 499
```

여기서 중요한 점은 nginx 앞에 L4/Load Balancer가 있다면 nginx가 말하는 `client`는 실제 사용자 브라우저가 아니라 **L4/Load Balancer일 수 있다**는 것입니다.

현재 구조가:

```text
Browser
  ↓
L4 / Load Balancer
  ↓
Web Server (nginx)
  ↓
WAS
```

라면:

```text
L4 → nginx 연결 종료
```

만으로도 nginx에는 `499`가 기록될 수 있습니다.

## 5. 499가 정상적으로 발생할 수 있는 경우

대표적인 정상 또는 비장애성 상황은 다음입니다.

| 상황 | 499 가능성 | 장애 여부 |
|---|---:|---|
| 사용자가 페이지 로딩 중 다른 페이지 이동 | 높음 | 대부분 정상 |
| 사용자가 새로고침 | 높음 | 대부분 정상 |
| 브라우저 탭/창 종료 | 높음 | 대부분 정상 |
| JavaScript가 AJAX 요청 취소 | 높음 | 설계에 따라 정상 |
| 중복 검색 요청 중 이전 요청 취소 | 높음 | 정상일 수 있음 |
| 모바일 네트워크 변경/끊김 | 가능 | 일시적 |
| L4 timeout | 높음 | **장애 후보** |
| WAS 응답 지연 | 간접적으로 높음 | **장애 후보** |
| DB/외부 API 장시간 대기 | 간접적으로 높음 | **장애 후보** |

nginx 측 설명에서도 client가 upstream 응답을 너무 오래 기다리다가 포기하는 것이 499의 흔한 원인으로 언급됩니다. 

## 6. `499`가 일반적이라고 볼 수 있는 경우

예를 들어 다음처럼 랜덤하게 소량 발생한다면 크게 이상하지 않을 수 있습니다.

```text
10:01 /search.do 499 0.23 sec
10:34 /image.jpg 499 0.05 sec
11:15 /ajax/search.do 499 0.31 sec
```

특징은:

```text
발생량이 적음
URI가 일정하지 않음
발생시간도 랜덤
request_time이 짧음
시스템 장애와 시간적으로 연관 없음
```

입니다.

nginx도 사용자가 페이지가 완전히 로드되기 전에 다른 링크를 클릭하면서 기존 연결을 닫는 상황 자체는 비정상적인 시스템 오류가 아닐 수 있다고 설명합니다. 

---

# 7. 반대로 이런 499는 정상으로 보면 안 됨

예를 들어:

```text
POST /order.do 499 rt=50.001
POST /order.do 499 rt=49.999
POST /login.do 499 rt=50.002
POST /api.do   499 rt=50.000
```

이면 매우 다릅니다.

사람이 우연히 계속 정확히 50초에 요청을 취소할 가능성은 낮습니다.

이 경우:

```text
50초 timeout
```

이 시스템 어딘가에 존재할 가능성이 높습니다.

예:

```text
L4 timeout = 50초
nginx timeout = 60초
```

이면:

```text
0초
Browser
 ↓
L4
 ↓
nginx
 ↓
WAS 처리 중

50초
L4 timeout
 ↓
Browser ← 504
 ↓
L4가 nginx 연결 종료

nginx
 ↓
499 기록
```

이 가능합니다.

이것이 지금 조사하고 계신 **Client 504 + Web Server 499** 조합과 매우 잘 맞는 시나리오입니다.

---

# 8. 499는 실제로 클라이언트에게 보내는 응답코드인가?

여기서 정확하게 구분해야 합니다.

아닙니다.

`499`가 발생했다는 것은 이미:

```text
Client connection
    X
nginx
```

상태이기 때문에 nginx가:

```http
HTTP/1.1 499 ...
```

를 client에게 정상적으로 반환했다는 의미가 아닙니다.

nginx가 자체 access log에:

```text
499
```

를 남겨:

> "이 요청은 Client가 먼저 연결을 종료해서 정상적으로 응답하지 못했다."

고 기록한 것입니다.

nginx는 `499`를 일반 `error_page` 대상으로 사용하는 것도 허용하지 않습니다. 이는 일반적인 HTTP response와 성격이 다르기 때문입니다. 

---

# 9. 현재 환경에서는 특히 `$request_time`을 봐야 함

499 자체보다:

```text
$request_time
```

이 훨씬 중요한 지표입니다.

### 9-1-1. 예 1

```text
status=499
request_time=0.082
```

→ 브라우저 취소, navigation, AJAX abort 가능성이 높음.

### 9-1-2. 예 2

```text
status=499
request_time=50.001
```

→ 50초 timeout 강하게 의심.

### 9-1-3. 예 3

```text
status=499
request_time=60.002
```

→ 60초 proxy/LB timeout 강하게 의심.

nginx 측에서도 499 원인 분석을 위해 `$request_time`과 `$upstream_response_time`을 같이 기록해 보라고 권고합니다. 

---

# 10. upstream timing까지 있으면 더 명확함

가능하다면 nginx access log에 다음을 추가하는 것이 좋습니다.

```nginx
$request_time
$upstream_connect_time
$upstream_header_time
$upstream_response_time
$upstream_addr
$upstream_status
```

예를 들어:

```text
status=499
rt=50.001
uct=0.002
uht=-
urt=50.000
```

이면:

```text
nginx → WAS 연결: 2ms
WAS 응답: 50초 동안 안 옴
50초 시점에 앞단이 nginx 연결 종료
```

라고 해석할 수 있습니다.

따라서:

```text
네트워크 연결 실패
```

보다는:

```text
WAS / DB / 외부 API 처리 지연
+
앞단 timeout
```

쪽으로 원인을 좁힐 수 있습니다.

---

# 11. POST 요청의 499는 특히 주의

POST 요청에서:

```text
POST /order.do → 499
```

가 발생했다고 해서 WAS에서 업무 처리가 실패했다고 판단해서는 안 됩니다.

가능한 상황:

```text
POST
 ↓
WAS 요청 도달
 ↓
DB 처리 시작
 ↓
50초 지나 L4 timeout
 ↓
사용자는 504 확인
 ↓
nginx 499
 ↓
DB에서는 첫 요청 처리 완료
```

사용자가 다시 버튼을 누르면:

```text
첫 번째 POST → 실제 성공
두 번째 POST → 다시 실행
```

이 되어 중복 주문/중복 등록/중복 상태변경 같은 문제가 발생할 수 있습니다.

따라서 중요한 POST에는:

```text
Request ID
Transaction ID
Unique Constraint
Idempotency 처리
중복 요청 방지
```

가 필요합니다.

---

## 11-1. 지금 상황 기준 판단

현재까지 말씀하신:

```text
Client        : 504
Web Server    : 499
WAS access.log: 없음
```

조합에서는 **499를 단순히 흔한 사용자 취소 로그로 처리하면 안 됩니다.**

특히 먼저 확인할 것은 다음 4개입니다.

1. **499의 `$request_time`이 50초/60초 등 특정 값에 집중되는지**
2. **L4/Load Balancer timeout 값이 얼마인지**
3. **nginx error.log가 `while connecting to upstream`인지 `while reading response header from upstream`인지**
4. **nginx의 `$upstream_connect_time`, `$upstream_response_time`, `$upstream_addr`가 무엇인지**

`499`가 짧은 시간에 랜덤하게 소량 발생하면 일반적일 수 있지만, **504 발생 시점과 일치하고 일정한 request time을 가진 499가 반복된다면 시스템 timeout/응답지연의 중요한 장애 지표로 판단하는 것이 맞습니다.**

## 11-2. 결론

현재 관측된 조합:

```text
Client          : 504 Gateway Timeout
Web Server/nginx: 499
WAS access.log  : 해당 요청 없음
```

이라면, 가장 먼저 의심할 시나리오는 다음입니다.

```text
Client
   ↓
L4 / Load Balancer
   ↓
nginx(Web)
   ↓
WAS
```

```text
① 요청 시작
② nginx가 WAS 응답 또는 WAS 연결을 기다림
③ L4의 timeout이 먼저 도달
④ L4가 Client에 504 반환
⑤ L4가 nginx와의 연결을 종료
⑥ nginx 입장에서는 자신의 Client(L4)가 연결을 끊음
⑦ nginx access.log = 499
```

즉 **Client의 504와 Web Server의 499가 서로 모순되는 것이 아니라, 동일 장애를 서로 다른 계층에서 본 결과일 가능성이 높습니다.**

nginx의 499는 nginx가 응답을 완료하기 전에 client 쪽이 연결을 닫았다는 의미입니다. nginx 앞에 Load Balancer가 있다면 nginx 입장에서 그 "client"는 실제 브라우저가 아니라 Load Balancer일 수 있습니다. nginx 개발자도 upstream 응답이 오래 걸려 client가 포기할 때 499가 흔히 발생한다고 설명합니다. 

다만 **WAS access.log가 없다는 것만으로 "WAS에는 요청이 도달하지 않았다"라고 확정하면 안 됩니다.** 이것이 이번 분석의 가장 중요한 포인트입니다.

---

# 12. 가장 가능성이 높은 시나리오

예를 들어 설정이 다음과 같다고 가정하겠습니다.

```text
L4 backend response timeout = 50초
nginx proxy_read_timeout    = 60초
WAS 실제 처리시간           = 70초
```

실제 흐름:

```text
00.000초
Client
  ↓
L4
  ↓
nginx
  ↓
WAS
```

WAS가 DB나 외부 API 등을 기다리고 있다고 하면:

```text
10초
WAS 처리 중

30초
WAS 처리 중

49초
WAS 처리 중
```

50초:

```text
L4
 ↓
"nginx가 아직 응답을 안 줬다"
 ↓
timeout
```

그러면 L4가:

```text
Client ← 504 Gateway Timeout
```

을 반환하고 nginx와의 연결을 종료할 수 있습니다.

nginx 입장에서는:

```text
L4
 X
nginx
```

이므로:

```text
499 Client Closed Request
```

를 기록합니다.

그 후 nginx가 WAS 연결까지 종료할 수 있습니다. nginx의 `proxy_ignore_client_abort` 기본값은 `off`이고, client가 응답을 기다리지 않고 연결을 닫았을 때 proxied server 연결을 종료할 수 있도록 동작합니다. 

결과적으로:

```text
Browser     504
L4          timeout
nginx       499
WAS         처리 중단/connection reset
WAS access  없음 또는 비정상 종료
```

패턴이 가능합니다.

### 12-1-1. 현재 증상과의 부합도

**높습니다.**

특히 499의 처리시간이:

```text
49.9초
50.0초
50.1초
```

처럼 일정하다면 이 시나리오 가능성은 매우 높아집니다.

---

# 13. 두 번째 시나리오: nginx → WAS 연결 자체가 성립하지 않음

다음 경우입니다.

```text
Client
 ↓
L4
 ↓
nginx
 ↓
 X
WAS
```

nginx가 WAS에 TCP 연결을 시도하지만:

```text
SYN
 ↓
응답 없음
 ↓
SYN 재전송
 ↓
...
```

상태일 수 있습니다.

예:

```text
nginx
 ↓
WAS:8080
 ↓
TCP connection 지연
```

그동안 L4는 nginx의 HTTP response를 기다립니다.

```text
L4 timeout = 50초
```

가 먼저 발생하면:

```text
Client ← L4 : 504

L4 → nginx connection 종료
nginx : 499
```

가 됩니다.

그리고 WAS는 애초에 HTTP 요청을 받지 않았기 때문에:

```text
WAS access.log = 없음
```

이 됩니다.

### 13-1-1. 이 경우 가장 먼저 나올 nginx error.log

다음 종류를 찾습니다.

```text
while connecting to upstream
```

또는:

```text
connect() failed
```

또는 TCP timeout 관련 로그입니다.

### 13-1-2. 가능 원인

| 원인 | 설명 |
|---|---|
| WAS listener 과부하 | TCP accept 처리 지연 |
| WAS port 이상 | listener가 제대로 안 받음 |
| Web→WAS Network | packet loss/retransmission |
| Firewall | DROP |
| OS backlog | 연결 요청 적체 |
| WAS CPU/GC | accept/처리 지연 |
| 특정 WAS node 이상 | 2 WAS 중 한쪽만 장애 |

---

# 14. 세 번째 시나리오: 요청은 WAS까지 도달했지만 처리가 완료되지 않음

이것도 상당히 중요합니다.

```text
Client
 ↓
L4
 ↓
nginx
 ↓
WAS
 ↓
Controller
 ↓
Service
 ↓
DB
```

요청은 실제로 WAS까지 들어왔지만:

```text
DB Lock
외부 API
Socket read
DB connection pool
Thread lock
GC pause
```

등에서 대기할 수 있습니다.

예:

```text
10:00:00 WAS request 도착

10:00:01 Service 진입

10:00:02 DB Lock 대기
          ↓
          ↓
          ↓

10:00:50 L4 timeout
```

L4:

```text
504
```

nginx:

```text
499
```

그리고 nginx가 upstream 연결을 종료합니다.

WAS request가 정상적으로 끝나지 못하면, **Undertow access log에서 정상적인 완료 요청으로 확인되지 않을 가능성이 있습니다.**

JBoss EAP Undertow access log는 상태코드와 처리시간 `%D`, `%T` 등을 기록할 수 있기 때문에 완료된 HTTP 처리시간 추적에 사용됩니다. 

따라서:

> WAS access.log 없음  
> = WAS 미도달

이라고 바로 결론 내리면 안 됩니다.

반드시:

```text
WAS server.log
Thread Dump
Controller 진입 로그
DB 로그
```

도 같이 확인해야 합니다.

---

# 15. 네 번째 시나리오: 특정 WAS만 이상

2-WAS 구조라면 매우 중요합니다.

```text
                  ┌─ WAS1 정상
Client → L4 → Web ┤
                  └─ WAS2 비정상
```

예:

```text
WAS1
→ 200 정상

WAS2
→ 연결/처리 지연
```

이면 요청마다 간헐적으로:

```text
200
200
504
200
504
200
```

형태가 나타날 수 있습니다.

Web Server 로그에:

```text
$upstream_addr
```

를 남기면 바로 확인할 수 있습니다.

예:

```text
10.10.10.11:8080 → 200
10.10.10.12:8080 → 499 관련 요청 집중
```

이라면:

```text
WAS2
```

를 집중 조사해야 합니다.

---

# 16. 현재 증상에서 상대적으로 가능성이 낮은 시나리오

### 16-1-1. 브라우저 자체가 단순 취소

사용자가:

```text
새로고침
창 닫음
페이지 이동
AJAX Abort
```

한 경우 nginx에서 `499`가 발생할 수 있습니다.

하지만 이 경우 **브라우저에 실제 504 화면이 표시됐다는 사실을 설명하기 어렵습니다.**

브라우저에서 504를 봤다면 누군가:

```text
HTTP/1.1 504 Gateway Timeout
```

을 실제로 생성해서 반환한 것입니다.

따라서 현재는:

```text
Browser 자체 timeout
```

보다:

```text
L4/Load Balancer가 504 생성
```

시나리오가 더 중요합니다.

---

# 17. 가장 먼저 확인할 것: 504를 누가 만들었는가

Chrome:

```text
F12
→ Network
→ 문제 Request
→ Headers
```

에서 Response Header를 봅니다.

확인:

```text
Server:
Via:
Date:
X-...
```

그리고 Response body도 확인합니다.

예를 들어:

```text
504 Gateway Time-out
nginx
```

인지,

Load Balancer 고유 error page인지 확인합니다.

다만 `Server:` 값이 nginx라고 해서 무조건 해당 Web Server가 만들었다고 단정할 수는 없으므로 **L4 로그와 함께 확인**해야 합니다.

---

# 18. Web Server에서는 499의 `request_time`이 가장 중요

지금 반드시 확인할 값입니다.

nginx access log에 다음 값이 있으면 좋습니다.

```text
$request_time
$upstream_connect_time
$upstream_header_time
$upstream_response_time
$upstream_status
$upstream_addr
```

nginx도 499 분석 시 `$request_time`과 `$upstream_response_time`을 같이 기록해서 확인하는 것을 권장합니다. 

예:

```text
status=499
request_time=50.001
upstream_connect_time=0.002
upstream_header_time=-
upstream_response_time=50.000
upstream_addr=10.10.10.21:8080
```

이 로그는 거의 이렇게 해석할 수 있습니다.

```text
Web → WAS 연결
0.002초
→ 정상

WAS response
50초 동안 없음

50초에 앞단 Client(L4)가 nginx 연결 종료

nginx
→ 499
```

### 18-1-1. 이 패턴이면

```text
L4 timeout ≈ 50초
+
WAS response 지연
```

을 최우선으로 봅니다.

---

# 19. 반대로 이렇게 나오면

```text
status=499
request_time=50.001
upstream_connect_time=-
upstream_header_time=-
upstream_response_time=-
```

Web Server가 WAS에 제대로 연결하지 못한 상태일 가능성을 확인해야 합니다.

nginx error.log에서:

```text
while connecting to upstream
```

여부가 중요합니다.

---

# 20. nginx error.log에서 반드시 찾을 메시지

장애 시각을 정확하게 맞춥니다.

예:

```bash
grep '2026/09/14 10:43' /var/log/nginx/error.log
```

또는:

```bash
grep -E \
'upstream timed out|client prematurely closed|connect\(\) failed|connection reset|no live upstreams' \
/var/log/nginx/error.log
```

특히 다음 메시지의 차이가 중요합니다.

| error.log | 판단 |
|---|---|
| `client prematurely closed connection` | L4/Client가 먼저 종료 |
| `while connecting to upstream` | Web→WAS 연결 문제 |
| `while reading response header from upstream` | WAS 연결 성공, 응답 지연 |
| `connect() failed` | WAS port/connect 문제 |
| `upstream timed out` | upstream timeout |
| `connection reset by peer` | WAS/네트워크 측 연결 종료 |

nginx는 client가 upstream 응답을 기다리는 동안 연결을 닫는 경우 499를 기록할 수 있습니다. 

---

# 21. Web Server timeout 설정

확인:

```bash
nginx -T 2>/dev/null | \
grep -E 'proxy_connect_timeout|proxy_read_timeout|proxy_send_timeout'
```

특히:

```text
proxy_connect_timeout
proxy_read_timeout
```

을 봅니다.

하지만 지금 Web 로그가 **504가 아니라 499**라는 것이 중요합니다.

만약 nginx timeout이 먼저였다면 일반적으로:

```text
nginx → 504
```

가 나올 가능성이 더 높습니다.

현재:

```text
nginx → 499
```

라면 nginx timeout보다 **nginx 앞단 timeout이 먼저 발생했다는 가설**이 더 강해집니다.

---

# 22. L4에서 확인할 값

현재 가장 중요한 확인 대상입니다.

운영 담당자에게 단순히:

> L4 이상 있나요?

라고 묻는 것보다 다음 값을 요청하십시오.

| 확인 항목 | 목적 |
|---|---|
| Backend response timeout | nginx 응답 대기 제한 |
| Idle timeout | TCP idle 연결 종료 |
| Connection timeout | nginx 연결 설정 제한 |
| 504 발생 로그 | 실제 L4가 504 생성했는지 |
| Backend 대상 | 어느 Web Server였는지 |
| 요청 시작/종료시간 | nginx 499와 시간 대조 |
| Reset/FIN 원인 | 누가 연결 종료했는지 |
| Health Check | 장애 노드 여부 |

특히:

```text
L4 timeout = 50초
nginx 499 request_time = 50.00x초
```

가 맞아떨어진다면 매우 강한 증거입니다.

---

# 23. WAS에서는 access.log보다 이걸 먼저 확인

WAS access.log가 없다는 이유로 WAS를 제외해서는 안 됩니다.

장애 발생 **동시에** Thread Dump:

```bash
jcmd <PID> Thread.print > thread1.txt
sleep 10
jcmd <PID> Thread.print > thread2.txt
sleep 10
jcmd <PID> Thread.print > thread3.txt
```

에서 확인:

```text
MariaDB JDBC
SocketInputStream
RestTemplate
HttpClient
GenericObjectPool.borrowObject
BLOCKED
WAITING
```

등입니다.

특히 다음 Stack이면 원인 후보가 강합니다.

### 23-1-1. DB

```text
Controller
→ Service
→ DAO
→ MariaDB JDBC
```

### 23-1-2. 외부통신

```text
Controller
→ Service
→ RestTemplate
→ HttpClient
→ Socket read
```

### 23-1-3. DBCP

```text
BasicDataSource.getConnection
→ GenericObjectPool.borrowObject
```

---

# 24. WAS listener와 연결 상태 확인

장애 발생 당시:

```bash
ss -lntp | grep 8080
```

실제 WAS port로 변경합니다.

연결 상태:

```bash
ss -ant | grep ':8080'
```

개수:

```bash
ss -ant | grep ':8080' | wc -l
```

상태별:

```bash
ss -ant | grep ':8080' | awk '{print $1}' | sort | uniq -c
```

확인할 것:

```text
ESTAB
SYN-RECV
CLOSE-WAIT
TIME-WAIT
```

특히 `SYN-RECV`가 비정상적으로 많으면 connection/accept 문제를 확인해야 합니다.

---

# 25. MariaDB도 동시에 확인

장애 발생 당시:

```sql
SHOW FULL PROCESSLIST;
```

특히:

```text
Time
State
Info
```

확인합니다.

```text
Waiting for table metadata lock
Locked
Waiting ...
```

또는 장시간 실행 Query가 있는지 봅니다.

추가:

```sql
SHOW ENGINE INNODB STATUS\G
```

---

# 26. POST + 499라면 추가로 매우 주의

앞서 찾고 계신 요청이 `POST`라면 중요한 운영 리스크가 있습니다.

사용자 입장:

```text
POST 요청
 ↓
504
```

그래서 사용자가 다시 버튼을 누릅니다.

하지만 실제로 첫 요청이 WAS까지 도달해 DB 처리 중이었다면:

```text
첫 POST ─────────→ DB 처리 진행

Client는 504 확인

두 번째 POST ───→ 다시 처리
```

가 가능해집니다.

즉:

> **504가 발생했다고 해서 첫 번째 POST가 처리되지 않았다고 판단하면 안 됩니다.**

주문/결제/회원등록/상태변경 같은 요청이면:

```text
Transaction ID
Request ID
중복처리 방지
Idempotency
DB unique constraint
```

등을 확인해야 합니다.

---

# 27. 현재 상황의 가능성 순위

현재 확보된 정보만으로 독립적으로 판단하면:

| 순위 | 시나리오 | 가능성 |
|---:|---|---:|
| **1** | WAS/Upstream 지연 → L4 timeout → Client 504 → nginx 499 | **높음** |
| **2** | Web→WAS TCP 연결 지연/실패 → L4 timeout | 높음 |
| **3** | WAS 요청 도달 후 DB/API/Pool에서 장기 대기 → 연결 abort | 중~높음 |
| **4** | 특정 WAS node/listener 장애 | 중간 |
| **5** | Web↔WAS network packet loss | 중간 |
| **6** | 단순 Browser 요청 취소 | 낮음 |

단, **1번과 2번을 구분하는 결정적인 자료는 nginx `error.log`와 `$upstream_connect_time`입니다.**

---

# 28. 가장 빠른 판정 절차

현재 장애가 다시 발생하면 다음 순서대로 같은 Request를 맞추면 됩니다.

```text
Client Network
│
│ 504 발생시간
│
▼
L4
│
│ 실제 504 생성 여부
│ timeout 값
│
▼
nginx access.log
│
│ 499
│ request_time
│ upstream_connect_time
│ upstream_header_time
│ upstream_response_time
│ upstream_addr
│
▼
nginx error.log
│
│ connecting to upstream ?
│ reading response header ?
│ client prematurely closed ?
│
▼
WAS
│
├─ access.log
├─ server.log
├─ Thread Dump
└─ socket 상태
```

## 28-1. 이번 장애에서 가장 중요한 판정 기준

### 28-1-1. 패턴 A

```text
Client          504 @ 50초
nginx           499 rt=50초
nginx uct       0.002초
nginx uht       -
WAS access      없음
```

→ **WAS 연결은 성공했지만 응답을 못 받고 있는 상황을 우선 조사**

---

### 28-1-2. 패턴 B

```text
Client          504 @ 50초
nginx           499 rt=50초
nginx uct       -
error.log       while connecting to upstream
WAS access      없음
```

→ **Web → WAS 연결 문제**

---

### 28-1-3. 패턴 C

```text
Client          504 @ 50초
nginx           499 rt=50초
WAS server.log  Controller 진입 확인
WAS access      없음
Thread dump     JDBC/Socket 대기
```

→ **WAS 내부 처리 지연**

---

### 28-1-4. 패턴 D

```text
Client          504 @ 50초
nginx           499 rt=50초
WAS1 정상
WAS2에서만 반복
```

→ **특정 WAS node 장애**

---

## 28-2. 현재 제가 가장 우선해서 확인할 3가지

**첫 번째는 nginx 499의 `$request_time`입니다.** `49.9~50.1초`처럼 일정한지 보십시오.

**두 번째는 nginx `error.log`입니다.** 동일 시각에:

```text
while connecting to upstream
```

인지:

```text
while reading response header from upstream
```

인지 확인하면 **WAS에 연결조차 못 했는지 / 연결 후 응답을 기다린 것인지**가 갈립니다.

**세 번째는 L4의 timeout 값입니다.** L4가 50초, nginx가 60초라면 현재의:

```text
Client 504
+
nginx 499
```

조합을 매우 자연스럽게 설명할 수 있습니다.

그리고 **WAS access.log 없음은 아직 WAS 무관의 증거가 아닙니다.** nginx가 L4 disconnect를 감지해 upstream 연결을 닫으면 WAS 쪽 HTTP exchange도 중간 종료될 수 있기 때문입니다. nginx는 기본적으로 `proxy_ignore_client_abort off`입니다. 

가능하다면 다음으로 **실제 Web Server 499 로그 한 줄과 같은 시각의 `error.log` 한두 줄**을 주시면, `L4 문제 / Web→WAS 연결 문제 / WAS 내부 지연` 세 가지 중 어디가 가장 유력한지 로그 필드 단위로 바로 판정할 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
