---
layout: single
title: "access_로그_분석"
excerpt: "access_로그_분석"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-15"
last_modified_at: "2026-09-15 12:09:26 +0900"
mermaid: false
---
## 1. 기본 커맨드

WAS `access.log`가 일반적인 Apache/JBoss Access Log 형식이라면 HTTP 응답코드는 보통 **9번째 필드**입니다.

```bash
awk '$9 != 200 {print}' access.log
```

예를 들어 로그가 다음 형태라면:

```text
192.168.0.10 - - [14/Sep/2026:10:50:01 +0900] "GET /main.do HTTP/1.1" 200 1234
192.168.0.11 - - [14/Sep/2026:10:50:02 +0900] "GET /error.do HTTP/1.1" 500 345
192.168.0.12 - - [14/Sep/2026:10:50:03 +0900] "POST /login.do HTTP/1.1" 302 0
```

결과는:

```text
192.168.0.11 - - [14/Sep/2026:10:50:02 +0900] "GET /error.do HTTP/1.1" 500 345
192.168.0.12 - - [14/Sep/2026:10:50:03 +0900] "POST /login.do HTTP/1.1" 302 0
```

입니다.

### 1-1. 먼저 응답코드가 정말 9번째 필드인지 확인

운영 로그에서는 이 확인을 먼저 하는 것이 좋습니다.

```bash
head -10 access.log
```

또는:

```bash
awk '{print $9}' access.log | sort | uniq -c
```

정상적인 형식이라면:

```text
12540 200
   32 302
    5 404
    2 500
```

처럼 나옵니다.

### 1-2. `200`이 아닌 요청을 파일로 저장

```bash
awk '$9 != 200 {print}' access.log > access_not_200.log
```

원본 `access.log`에는 영향을 주지 않습니다.

### 1-3. 오류성 응답만 확인하고 싶다면

`200` 이외에는 `302`, `304` 같은 정상적인 응답도 포함됩니다. 실제 장애 분석이라면 **400 이상만 보는 것**이 더 유용할 수 있습니다.

```bash
awk '$9 >= 400 {print}' access.log
```

5xx만:

```bash
awk '$9 >= 500 && $9 < 600 {print}' access.log
```

4xx만:

```bash
awk '$9 >= 400 && $9 < 500 {print}' access.log
```

### 1-4. 응답코드별 발생 건수

장애 조사에서는 이 명령이 특히 유용합니다.

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

예:

```text
25134 200
  356 302
   78 404
   12 500
    4 503
```

### 1-5. 실시간으로 200 아닌 요청 감시

```bash
tail -F access.log | awk '$9 != 200 {print; fflush()}'
```

5xx만 실시간 확인:

```bash
tail -F access.log | awk '$9 >= 500 && $9 < 600 {print; fflush()}'
```

### 1-6. 특정 날짜/시간대 + 200 아닌 요청

예를 들어 `14/Sep/2026:10` 시간대:

```bash
awk '$0 ~ /14\/Sep\/2026:10/ && $9 != 200 {print}' access.log
```

10:40분대만:

```bash
awk '$0 ~ /14\/Sep\/2026:10:40:/ && $9 != 200 {print}' access.log
```

### 1-7. 운영 장애 확인용 추천 조합

현재처럼 WAS 요청 상태를 조사하는 목적이라면 먼저:

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

그다음:

```bash
awk '$9 >= 400 {print}' access.log
```

그리고 5xx만 별도로:

```bash
awk '$9 >= 500 && $9 < 600 {print}' access.log
```

를 보는 것을 권장합니다.

단, **JBoss EAP 7.2의 `access-log` pattern을 커스텀했다면 `$9`가 응답코드가 아닐 수 있습니다.** `access.log` 실제 한두 줄을 보여주시면 현재 로그 포맷 기준으로 **응답코드, 처리시간, 요청 URI, Client IP까지 정확히 분리하는 `awk` 커맨드**로 만들어 드릴 수 있습니다.

JBoss EAP 7.2 / Undertow 기준으로 access log에 **응답시간 `%D`(milliseconds) 또는 `%T`(seconds)가 기록되고 있다면**, 50초 이상 요청은 `awk`로 바로 찾을 수 있습니다. Undertow에서 `%D`는 요청 처리시간(ms), `%T`는 초 단위입니다. 

먼저 현재 로그 한 줄을 확인하십시오.

```bash
head -5 access.log
```

예를 들어 패턴이 다음과 같다고 가정합니다.

```text
192.168.1.10 - - [14/Sep/2026:10:40:31 +0900] "GET /main.do HTTP/1.1" 200 12345 52341
```

마지막 `52341`이 `%D`, 즉 **52.341초**라면:

```bash
awk '$NF >= 50000 {print}' access.log
```

가 가장 간단합니다.

`$NF`는 **마지막 필드**를 의미하므로 응답시간이 로그 맨 마지막에 있을 때 유용합니다.

### 1-8. 응답시간이 `%D` = millisecond인 경우

50초 = `50000ms`이므로:

```bash
awk '$NF >= 50000 {print}' access.log
```

60초 이상:

```bash
awk '$NF >= 60000 {print}' access.log
```

30초 이상:

```bash
awk '$NF >= 30000 {print}' access.log
```

### 1-9. 응답시간이 `%T` = second인 경우

로그 마지막 필드가 초 단위라면:

```bash
awk '$NF >= 50 {print}' access.log
```

입니다.

## 2. 응답시간 기준으로 오래 걸린 순서 정렬

`%D`가 마지막 필드라면:

```bash
awk '$NF >= 50000 {print}' access.log | sort -k$(awk '{print NF; exit}' access.log)nr
```

보다 실무에서는 아래처럼 쓰는 편이 간단합니다.

```bash
awk '$NF >= 50000 {print $NF, $0}' access.log | sort -nr
```

다만 이 명령은 응답시간을 앞에 한 번 더 붙입니다.

예:

```text
125341 192.168.1.10 ... GET /order.do ... 200 ... 125341
72324  192.168.1.11 ... POST /login.do ... 200 ... 72324
52341  192.168.1.12 ... GET /main.do ... 200 ... 52341
```

보기 좋게 초 단위로 출력하려면:

```bash
awk '$NF >= 50000 {
    printf "%.3f sec | %s\n", $NF / 1000, $0
}' access.log
```

예:

```text
52.341 sec | 192.168.1.12 ... "GET /main.do HTTP/1.1" 200 12345 52341
72.324 sec | 192.168.1.11 ... "POST /login.do HTTP/1.1" 200 542 72324
```

## 3. 50초 이상 + HTTP 상태코드도 같이 보고 싶은 경우

일반적인 access-log pattern이:

```text
%h %l %u %t "%r" %s %b %D
```

라면 실제 필드 위치는 대략:

```text
$1   IP
$4~5 시간
$6~8 request
$9   status
$10  bytes
$11  duration(ms)
```

가 됩니다.

따라서:

```bash
awk '$11 >= 50000 {print}' access.log
```

또는:

```bash
awk '$11 >= 50000 {
    printf "time=%.3fs status=%s request=%s %s %s\n",
           $11/1000, $9, $6, $7, $8
}' access.log
```

처럼 사용할 수 있습니다.

예:

```text
time=52.341s status=200 request="GET /main.do HTTP/1.1"
time=61.292s status=500 request="POST /order.do HTTP/1.1"
```

## 4. 특정 날짜/시간대에서 50초 이상

예를 들어 2026-09-14 10시대:

```bash
awk '$0 ~ /14\/Sep\/2026:10:/ && $NF >= 50000 {print}' access.log
```

10:40분대:

```bash
awk '$0 ~ /14\/Sep\/2026:10:40:/ && $NF >= 50000 {print}' access.log
```

현재 장애 발생 시간 `10:43`만 보면:

```bash
awk '$0 ~ /14\/Sep\/2026:10:43:/ && $NF >= 50000 {print}' access.log
```

## 5. 중요한 점: access.log에 응답시간 항목이 있어야 함

기본 `common` 패턴은:

```text
%h %l %u %t "%r" %s %b
```

이고 기본적으로 응답시간이 포함되지 않습니다. Undertow는 `%D` 또는 `%T`를 명시해야 처리시간을 기록합니다. 

예를 들어 설정이:

```xml
<access-log pattern="%h %l %u %t &quot;%r&quot; %s %b %D"/>
```

처럼 되어 있어야 합니다.

또 하나 중요한 점은 EAP 7.x에서 `%D/%T`를 제대로 기록하려면 HTTP listener에서 `record-request-start-time=true` 설정이 필요합니다. 그렇지 않으면 응답시간 값이 `-`로 나오는 사례가 있습니다. 

예:

```xml
<http-listener
    name="default"
    socket-binding="http"
    record-request-start-time="true"/>
```

CLI라면 일반적으로:

```bash
/subsystem=undertow/server=default-server/http-listener=default:write-attribute(name=record-request-start-time,value=true)
```

입니다.

### 5-1. 현재 장애 분석에서 가장 추천하는 명령

응답시간이 마지막 필드 `%D`라는 전제라면 먼저:

```bash
awk '$NF >= 50000 {
    printf "%.3f sec | %s\n", $NF/1000, $0
}' access.log
```

그리고 오래 걸린 순서까지 보고 싶다면:

```bash
awk '$NF >= 50000 {
    print $NF, $0
}' access.log | sort -nr
```

를 사용하십시오.

**다만 현재 access.log 실제 한 줄을 한두 개 보여주시면**, 제가 그 로그 포맷에서 **응답시간이 몇 번째 필드인지 정확히 계산해서 50초 이상 + URI + HTTP status + 응답시간만 깔끔하게 출력하는 명령어**로 만들어드릴 수 있습니다.

## 6. 결론

504 분석의 핵심은 **“504를 누가 만들었는가”와 “어느 구간의 timeout인가”를 숫자로 분리하는 것**입니다.

현재 구조가:

```text
Client
  ↓
L4
  ↓
nginx
  ↓
JBoss EAP 7.2 / Undertow
  ↓
Spring Service
  ↓
MariaDB / 외부 API / Socket
```

라면 다음 순서로 분석하는 것이 가장 효율적입니다.

> **① nginx의 504 시각·URI·소요시간 확인 → ② nginx error.log에서 timeout 종류 확인 → ③ 같은 요청의 WAS 도달 여부/처리시간 확인 → ④ WAS Thread·DB·외부연계 분석 → ⑤ L4/네트워크 확인**

특히 nginx의 기본 `proxy_connect_timeout`과 `proxy_read_timeout`은 각각 60초이므로, **거의 정확히 60초 전후에서 504가 발생한다면 nginx timeout 설정과 강하게 연관**됩니다. 다만 `proxy_read_timeout`은 전체 요청 처리시간 제한이 아니라 **upstream으로부터 연속된 두 read 사이의 대기시간**입니다. 

---

# 7. 504가 의미하는 것

504 Gateway Timeout은 대체로:

```text
Gateway/Proxy
     ↓
Upstream Server
     ↓
정해진 시간 내 응답을 받지 못함
     ↓
504 반환
```

입니다.

예를 들어 nginx가 504를 만들었다면:

```text
Browser
   ↓
nginx
   ↓ 요청
JBoss
   ↓
응답 지연
   ↓
nginx timeout
   ↓
Browser ← 504
```

입니다.

따라서:

```text
504 = JBoss 자체 오류
```

라고 보면 안 됩니다.

오히려:

```text
504 = 앞단 Gateway가 뒤쪽 서버를 기다리다가 포기한 결과
```

인 경우가 일반적입니다.

---

# 8. 가장 먼저 504 발생 건수와 시각 확인

nginx access log에서:

```bash
grep ' 504 ' access.log
```

또는 status가 9번째 필드라면:

```bash
awk '$9 == 504 {print}' access.log
```

날짜별:

```bash
grep '14/Sep/2026' access.log | awk '$9 == 504 {print}'
```

특정 시간:

```bash
grep '14/Sep/2026:10:' access.log | awk '$9 == 504 {print}'
```

504 건수:

```bash
awk '$9 == 504 {count++} END {print count}' access.log
```

URI별 발생 건수:

```bash
awk '$9 == 504 {print $7}' access.log \
| sort \
| uniq -c \
| sort -nr
```

이것만으로도:

```text
특정 URI에서만 발생
vs
사이트 전체에서 동시에 발생
```

을 구분할 수 있습니다.

이 차이가 매우 중요합니다.

---

# 9. nginx error.log가 가장 중요

504 발생 시각의 nginx `error.log`를 확인해야 합니다.

일반적으로:

```bash
grep '2026/09/14 10:43' error.log
```

또는:

```bash
grep -E 'upstream timed out|connect\(\) failed|no live upstreams' error.log
```

## 9-1. 대표 메시지 1

```text
upstream timed out (110: Connection timed out)
while reading response header from upstream
```

이 메시지가 나오면:

```text
nginx → WAS 연결 성공
        ↓
WAS에 요청 전달
        ↓
WAS response header 기다림
        ↓
timeout
```

일 가능성이 높습니다.

즉 **WAS 내부 처리 지연**을 가장 먼저 봐야 합니다.

후보:

```text
DB Lock
DB Slow Query
Connection Pool 고갈
Thread Pool 고갈
외부 API 대기
Socket read 대기
GC pause
무한/장기 처리
```

---

## 9-2. 대표 메시지 2

```text
upstream timed out
while connecting to upstream
```

또는 연결 단계 timeout이라면:

```text
nginx
 ↓
WAS TCP connection 시도
 ↓
연결 자체가 제한시간 내 성립하지 않음
```

입니다.

이때는 Spring Controller나 SQL을 먼저 볼 문제가 아닙니다.

확인 대상:

```text
WAS process 상태
Undertow listener
WAS port
TCP backlog
OS socket
방화벽
L4
네트워크
WAS 과부하
```

입니다.

---

## 9-3. 대표 메시지 3

```text
connect() failed (111: Connection refused)
while connecting to upstream
```

이 경우에는 timeout보다 더 명확합니다.

```text
nginx → WAS IP:PORT
             ↓
           거절
```

가능성:

```text
JBoss down
잘못된 port
listener down
서비스 재기동 중
방화벽 REJECT
```

입니다.

보통 이런 경우는 502로 나타나는 경우도 많습니다.

---

# 10. nginx 로그에 반드시 추가하면 좋은 값

504 분석에서 단순:

```text
URI
status
```

만 가지고 있으면 원인을 찾기 어렵습니다.

다음 값들을 기록하는 것을 강하게 권장합니다.

```text
$request_time
$upstream_connect_time
$upstream_header_time
$upstream_response_time
$upstream_status
$upstream_addr
```

nginx 공식 변수의 의미는 다음과 같습니다. `$upstream_connect_time`은 upstream 연결 시간, `$upstream_header_time`은 응답 헤더를 받기까지의 시간, `$upstream_response_time`은 upstream 응답 처리시간입니다. 

예:

```nginx
log_format upstream_timing
    '$remote_addr [$time_local] '
    '"$request" '
    'status=$status '
    'rt=$request_time '
    'uct=$upstream_connect_time '
    'uht=$upstream_header_time '
    'urt=$upstream_response_time '
    'us=$upstream_status '
    'ua=$upstream_addr';

access_log /var/log/nginx/access_timing.log upstream_timing;
```

`$request_time`은 클라이언트의 첫 번째 request byte를 받은 시점부터 마지막 response byte를 기록하기 직전까지의 전체 시간입니다. 

---

# 11. 이 값만 있으면 원인 범위가 크게 좁아짐

예를 들어:

```text
status=504
rt=60.002
uct=60.000
uht=-
urt=60.000
```

라면:

### 11-1-1. Case A

```text
uct ≈ 60초
```

즉:

> nginx → WAS 연결 자체가 늦음

을 우선 봅니다.

```text
Network
WAS listener
WAS 부하
TCP backlog
WAS down
```

후보입니다.

---

### 11-1-2. Case B

```text
status=504
rt=60.002
uct=0.003
uht=60.000
urt=60.000
```

연결은:

```text
0.003초
```

만에 성공했습니다.

그런데 response header가 60초 동안 안 왔습니다.

즉:

```text
nginx
 ↓ 3ms 만에 연결
WAS
 ↓
Controller/Service 처리
 ↓
60초 동안 아무 응답 header 없음
```

입니다.

이 경우 WAS/DB/외부 API 쪽이 매우 강한 후보입니다.

---

### 11-1-3. Case C

```text
status=200
rt=65.124
uct=0.002
uht=0.200
urt=65.120
```

response header는 빠르게 받았지만 전체 body가 오래 걸렸습니다.

이 경우:

```text
대용량 response
Streaming
Download
Socket
응답 body 생성 지연
```

등을 봅니다.

---

# 12. timeout 값 확인

nginx 서버에서:

```bash
nginx -T | grep -E \
'proxy_connect_timeout|proxy_read_timeout|proxy_send_timeout'
```

또는:

```bash
grep -R "proxy_.*timeout" /etc/nginx/
```

확인 대상:

```nginx
proxy_connect_timeout
proxy_send_timeout
proxy_read_timeout
```

nginx 기본값은 공식 문서 기준:

```text
proxy_connect_timeout 60s
proxy_send_timeout    60s
proxy_read_timeout    60s
```

입니다. 

예를 들어 장애가:

```text
59.9초
60.0초
60.1초
```

에 집중된다면 굉장히 중요한 단서입니다.

---

# 13. WAS access.log와 반드시 대조

이 부분이 현재 분석에서 특히 중요합니다.

예:

```text
nginx
10:43:00 request
10:44:00 504
```

같은 URI를 WAS access.log에서 찾습니다.

```bash
grep '/target.do' access.log
```

---

## 13-1. 패턴 1: WAS access.log 없음

```text
nginx : 504
WAS access.log : 없음
```

가능성은 두 가지입니다.

### 13-1-1. A. WAS에 아예 도달하지 않음

```text
nginx
 ↓
TCP 연결 실패/timeout
 X
WAS
```

### 13-1-2. B. WAS에 도달했지만 아직 처리가 안 끝남

Undertow access log는 일반적으로 request/response 처리 종료 시 기록되므로:

```text
Controller 진입
 ↓
DB에서 90초 대기
```

하고 있다면 nginx가 60초에 504를 내더라도 **그 시점에는 WAS access.log가 아직 없을 수 있습니다.**

따라서:

> nginx 504 + WAS access 없음  
> = WAS에 요청이 안 왔다

라고 즉시 결론 내리면 안 됩니다.

---

# 14. 매우 중요한 패턴

다음 패턴이 발견되면 원인이 거의 명확합니다.

```text
10:43:00 nginx → 요청
10:44:00 nginx → 504

10:44:15 WAS access.log
             → 동일 URI 200
             → 처리시간 75초
```

이것은:

```text
WAS는 75초 후 정상 처리 완료
        ↑
nginx는 60초에서 이미 포기
```

입니다.

즉:

> **nginx `proxy_read_timeout` < WAS 처리시간**

입니다.

이때 단순히:

```nginx
proxy_read_timeout 120s;
```

로 늘리는 것은 **응급 조치일 수는 있지만 근본 해결은 아닙니다.**

왜 WAS가 75초나 걸렸는지를 봐야 합니다.

---

# 15. WAS에서 50초 이상 요청 검색

JBoss access.log에 응답시간 `%D`가 마지막 필드(ms)라면:

```bash
awk '$NF >= 50000 {print}' access.log
```

60초 이상:

```bash
awk '$NF >= 60000 {print}' access.log
```

오래 걸린 순서:

```bash
awk '$NF >= 50000 {
    print $NF, $0
}' access.log | sort -nr
```

초 단위 표시:

```bash
awk '$NF >= 50000 {
    printf "%.3f sec | %s\n", $NF/1000, $0
}' access.log
```

Red Hat도 JBoss EAP access log에 처리시간을 포함시켜 느린 요청을 추적하는 방법을 안내하고 있습니다. 

---

# 16. WAS에서 다음으로 Thread Dump 확인

504가 발생하고 있는 **바로 그 시점**의 Thread Dump가 매우 중요합니다.

JDK 11이면:

```bash
jcmd <PID> Thread.print > thread_1.txt
```

10초 간격으로 3회 정도:

```bash
jcmd <PID> Thread.print > thread_1.txt
sleep 10

jcmd <PID> Thread.print > thread_2.txt
sleep 10

jcmd <PID> Thread.print > thread_3.txt
```

또는:

```bash
jstack <PID> > thread_dump.txt
```

확인할 상태:

```text
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
```

보다 더 중요한 것은 **어디에서 멈춰 있는지**입니다.

예:

```text
java.net.SocketInputStream.socketRead0
```

또는:

```text
org.mariadb.jdbc...
```

또는:

```text
org.apache.http...
```

등입니다.

---

# 17. Thread Dump별 판단

### 17-1-1. DB 대기

```text
Controller
 → Service
 → DAO
 → MariaDB JDBC
```

에서 여러 dump 동안 동일하게 정지:

```text
DB Slow Query
DB Lock
Connection 문제
```

가능성이 큽니다.

---

### 17-1-2. 외부 API

```text
RestTemplate
 ↓
HttpClient
 ↓
SocketInputStream.read
```

이면:

```text
외부 API response 지연
read timeout 미설정
Proxy 서버 지연
```

을 봅니다.

---

### 17-1-3. DB Connection Pool

예:

```text
GenericObjectPool.borrowObject()
BasicDataSource.getConnection()
```

근처에서 다수 Thread가 기다린다면:

```text
DBCP connection pool 고갈
```

가능성이 매우 높습니다.

현재 DBCP 설정에서 `maxWaitMillis=5000` 같은 제한이 있다면 보통 무한 대기는 방지되지만, 실제 운영 설정과 일치하는지 확인해야 합니다. 

---

# 18. MariaDB 확인

504 발생 시점에:

```sql
SHOW FULL PROCESSLIST;
```

확인합니다.

특히:

```text
Time
State
Info
```

를 봅니다.

의심 상태:

```text
Waiting for table metadata lock
Locked
Waiting for ...
Sending data
```

또한:

```sql
SHOW ENGINE INNODB STATUS\G
```

로:

```text
LATEST DETECTED DEADLOCK
TRANSACTIONS
LOCK WAIT
```

를 봅니다.

특정 Query가 50~60초 이상 지속된다면 504와 직접 연관시켜야 합니다.

---

# 19. Connection Pool 모니터링

현재처럼 DBCP2를 사용한다면 최소 다음 값을 모니터링하는 것이 좋습니다.

```text
numActive
numIdle
maxTotal
maxWaitMillis
```

예를 들어:

```text
maxTotal = 100
numActive = 100
numIdle   = 0
```

이 지속된다면:

```text
Request
 ↓
getConnection()
 ↓
Pool available connection 없음
 ↓
대기
```

상태가 발생할 수 있습니다.

이때 SQL 자체가 느리지 않아도 504가 발생할 수 있습니다.

---

# 20. JVM/OS도 동시에 확인

504 발생 시점에:

```bash
top
```

메모리:

```bash
free -m
```

CPU/Run Queue:

```bash
vmstat 1
```

Disk:

```bash
iostat -x 1
```

Socket:

```bash
ss -ant
```

WAS port:

```bash
ss -ant | grep ':8080'
```

예를 들어 `SYN-RECV`가 비정상적으로 많거나 accept queue 관련 이상이 있다면 HTTP application보다 network/listener 계층을 봐야 합니다.

---

# 21. GC pause 확인

Java 11 환경에서 Full GC가 길게 발생하면:

```text
nginx
 ↓
WAS
 ↓
Stop-The-World
 ↓
60초간 response 없음
 ↓
504
```

도 이론적으로 가능합니다.

확인:

```text
GC log
```

에서 장애 시각 기준:

```text
Pause Full
Pause Young
Allocation Failure
```

등을 확인하십시오.

단, **60초짜리 GC는 정상 상황이 아니므로** 발견된다면 매우 강한 원인 후보입니다.

---

# 22. 외부 API / Socket이 있는 서비스

다음 구조는 특히 중요합니다.

```text
Browser
 ↓
Spring Controller
 ↓
외부 인증 API
 ↓
Proxy
 ↓
외부 서버
```

여기서 timeout이:

```text
connectTimeout = 없음
readTimeout    = 없음
```

이면:

```text
외부 서버가 응답 안 함
 ↓
WAS Thread 대기
 ↓
nginx 60초
 ↓
504
```

가 가능합니다.

따라서 모든 외부 연결에:

```text
Connection Pool timeout
Connect timeout
Read/Socket timeout
```

을 명확히 설정해야 합니다.

---

# 23. L4에서 확인할 것

nginx/WAS에 이상이 없을 때 L4에서:

```text
Backend health 상태
Connection 수
Connection timeout
Idle timeout
Reset
Retransmission
Backend 선택
특정 WAS 편중
```

을 확인합니다.

특히 2-WAS 구조라면:

```text
WAS1 정상
WAS2 지연
```

일 수 있습니다.

그러면:

```text
전체 요청이 아니라 약 50%에서만 간헐적 504
```

같은 패턴이 나타날 수 있습니다.

이때 nginx 또는 L4 로그에서 반드시:

```text
upstream_addr
```

를 남겨야 합니다.

예:

```text
10.0.1.11:8080 → 200
10.0.1.12:8080 → 504 다수
```

가 보이면 바로 특정 노드로 좁힐 수 있습니다.

---

# 24. 504 지속 모니터링에 권장하는 로그

가장 중요한 개선입니다.

nginx access log에 최소:

```nginx
log_format timing
    '$remote_addr '
    '[$time_local] '
    '"$request" '
    'status=$status '
    'bytes=$body_bytes_sent '
    'rt=$request_time '
    'uct=$upstream_connect_time '
    'uht=$upstream_header_time '
    'urt=$upstream_response_time '
    'us=$upstream_status '
    'ua=$upstream_addr';
```

를 남기십시오.

이 한 줄이면 대부분의 504를 크게 세 가지로 나눌 수 있습니다.

| 패턴 | 우선 원인 |
|---|---|
| `uct`가 김 | nginx ↔ WAS 연결 |
| `uct` 짧고 `uht`가 김 | WAS 내부 처리 |
| `uht` 짧고 `urt`가 김 | response body/stream 처리 |

---

# 25. 실시간 504 모니터링

```bash
tail -F access.log | grep --line-buffered ' 504 '
```

또는 status가 9번째 필드:

```bash
tail -F access.log |
awk '$9 == 504 {print; fflush()}'
```

최근 504 개수:

```bash
grep ' 504 ' access.log | wc -l
```

URI별:

```bash
awk '$9 == 504 {print $7}' access.log |
sort |
uniq -c |
sort -nr
```

---

# 26. 실무 분석 우선순위

현재 환경에서는 다음 순서로 진행하는 것을 권장합니다.

| 순위 | 확인 대상 | 판단 목적 |
|---:|---|---|
| **1** | nginx access.log | 504 시간·URI·WAS 확인 |
| **2** | nginx error.log | connect/read/send timeout 구분 |
| **3** | nginx timeout 설정 | 60초 패턴 확인 |
| **4** | WAS access.log | 도달 및 실제 처리시간 |
| **5** | WAS Thread Dump | DB/API/Socket/Pool 대기 확인 |
| **6** | MariaDB PROCESSLIST | Lock/Slow Query |
| **7** | DBCP 상태 | Pool 고갈 |
| **8** | GC/CPU/Memory | JVM/OS stall |
| **9** | 외부 API | Connect/read timeout |
| **10** | L4 | 특정 WAS/connection 이상 |

---

# 27. 원인 판별표

장애 현장에서 이 표를 사용하면 빠릅니다.

| nginx | WAS access | 판단 |
|---|---|---|
| 504, `uct≈60s` | 없음 | nginx→WAS 연결 문제 |
| 504, `uct≪1s`, `uht≈60s` | 나중에 200/60초+ | **WAS 처리 지연** |
| 504, `uct≪1s`, `uht≈60s` | 없음 | WAS 장기 실행/중단 여부 확인 |
| 504 | WAS 500 | WAS 오류 + gateway 동작 확인 |
| 504가 특정 WAS만 | 특정 노드 이상 | 해당 WAS 집중 조사 |
| 504가 모든 WAS 동시 | 전 구간 | DB/공통 외부 API/Infra 의심 |
| 504가 특정 URI만 | 해당 URI 느림 | Service/SQL/API 분석 |
| 504가 거의 정확히 60초 | 반복 | **timeout 설정과 강한 연관** |

---

## 27-1. 지금 상황에서 가장 먼저 권장하는 5개 명령

nginx 서버 접근이 가능하다면:

```bash
# 1. 504 찾기
grep ' 504 ' /var/log/nginx/access.log

# 2. timeout 관련 error 확인
grep -E \
'upstream timed out|connect\(\) failed|no live upstreams' \
/var/log/nginx/error.log

# 3. nginx timeout 설정 확인
nginx -T 2>/dev/null |
grep -E 'proxy_connect_timeout|proxy_read_timeout|proxy_send_timeout'

# 4. upstream timing 로그 설정 여부 확인
nginx -T 2>/dev/null |
grep -E 'request_time|upstream_connect_time|upstream_header_time|upstream_response_time'

# 5. 해당 시간 WAS access.log 비교
grep '14/Sep/2026:10:43' access.log
```

**가장 중요한 개선은 nginx에 `$request_time`, `$upstream_connect_time`, `$upstream_header_time`, `$upstream_response_time`, `$upstream_addr`, `$upstream_status`를 남기는 것입니다.** 이 값들이 있으면 다음 504부터는 “nginx→WAS 연결 문제인지, WAS 내부가 느린 것인지”를 추측이 아니라 수치로 판정할 수 있습니다. 

현재처럼 **WAS에만 접근 가능한 상황**에서도 분석 절차를 별도로 만들 수 있습니다. 그 경우에는 `access.log + 처리시간 + thread dump + MariaDB PROCESSLIST + server.log`만으로 **“WAS 원인 / WAS 이전 구간 원인”을 판정하는 체크리스트와 실제 명령어**를 구성하는 것이 다음 단계로 가장 유용합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
