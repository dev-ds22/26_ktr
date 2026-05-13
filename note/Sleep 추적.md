정확도: 96%

# `SHOW FULL PROCESSLIST` 결과의 `Host` 값 해석

## 1. 결론

`SHOW FULL PROCESSLIST`의 `Host`는 **MariaDB에 접속한 클라이언트의 호스트 정보**입니다. 즉, DB 서버 입장에서 **어느 애플리케이션 서버/WAS/배치 서버/DB 클라이언트가 접속했는지**를 보여주는 값입니다. MariaDB 공식 문서도 `SHOW PROCESSLIST`의 `Host` 컬럼을 “client가 연결된 host”로 설명합니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/show/show-processlist?utm_source=chatgpt.com "SHOW PROCESSLIST | Server | MariaDB Documentation"))

```text
Host = DB에 접속한 클라이언트 주소
```

예:

```text
192.168.10.21:53214
```

해석:

```text
192.168.10.21 = MariaDB에 접속한 클라이언트 서버 IP
53214 = 클라이언트 측 임시 포트
```

## 2. `Host` 값 구성

보통 TCP/IP 접속이면 다음 형식으로 보입니다.

```text
클라이언트IP:클라이언트임시포트
```

|예시|해석|
|---|---|
|`192.168.10.21:53214`|`192.168.10.21` 서버에서 MariaDB로 접속한 세션|
|`10.20.1.15:42188`|`10.20.1.15` WAS 또는 배치 서버에서 접속|
|`localhost`|DB 서버 내부에서 로컬 접속|
|`localhost:xxxxx`|로컬 TCP 접속으로 보이는 경우|
|`app01.company.local:xxxxx`|DNS reverse lookup 또는 hostname 형태로 표시된 경우|
|`Host` 뒤의 포트는 MariaDB 서버 포트가 아닙니다. MariaDB 서버 포트가 보통 `3306`이라도 `Host`에는 클라이언트가 접속할 때 사용한 **client-side ephemeral port**가 표시될 수 있습니다.||

## 3. `Host`를 볼 때 가장 중요한 점

`Host`는 “DB 서버 주소”가 아니라 **DB에 접속한 쪽의 주소**입니다.

```text
잘못된 해석:
Host = MariaDB 서버 IP
```

```text
정확한 해석:
Host = MariaDB에 접속한 클라이언트 IP 또는 호스트명
```

따라서 `Sleep` connection이 많을 때는 `Host`를 기준으로 **어느 WAS 또는 배치 서버가 많은 connection을 점유하는지**를 확인할 수 있습니다.

## 4. 실무 분석 SQL

### 4.1 Host별 connection 수 확인

```sql
SELECT
    SUBSTRING_INDEX(HOST, ':', 1) AS CLIENT_HOST,
    USER,
    DB,
    COMMAND,
    COUNT(*) AS CNT
FROM information_schema.PROCESSLIST
GROUP BY
    SUBSTRING_INDEX(HOST, ':', 1),
    USER,
    DB,
    COMMAND
ORDER BY CNT DESC;
```

이 결과로 다음을 판단합니다.

```text
특정 CLIENT_HOST에서 Sleep이 집중됨
→ 해당 WAS/배치 서버의 Connection Pool 설정 또는 Connection Leak 확인
```

### 4.2 특정 Host의 오래된 Sleep 확인

```sql
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE,
    INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
ORDER BY TIME DESC
LIMIT 50;
```

MariaDB의 `PROCESSLIST`에서 `TIME`은 해당 thread가 현재 상태에 머문 시간입니다. 즉 `COMMAND='Sleep'`이면 `TIME`은 Sleep 상태로 머문 초 단위 시간으로 해석할 수 있습니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/information-schema/information-schema-tables/information-schema-processlist-table?utm_source=chatgpt.com "Information Schema PROCESSLIST Table | Server - MariaDB"))

### 4.3 Host별 Sleep 수만 확인

```sql
SELECT
    SUBSTRING_INDEX(HOST, ':', 1) AS CLIENT_HOST,
    COUNT(*) AS SLEEP_CNT
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
GROUP BY SUBSTRING_INDEX(HOST, ':', 1)
ORDER BY SLEEP_CNT DESC;
```

## 5. `Host` 기준으로 보는 장애 해석

|현상|해석|확인 대상|
|---|---|---|
|한 IP에서 Sleep 100개|특정 WAS/배치 서버가 connection을 많이 보유|해당 서버의 DBCP `numActive`, `numIdle`|
|여러 IP에 Sleep이 10개씩 분산|각 WAS의 `minIdle=10` 합산 가능|WAS 대수, DataSource 개수|
|특정 IP에서 `Query`/`Locked` 많음|해당 서버 요청이 장기 쿼리 또는 Lock 유발|SQL, Thread Dump, 트랜잭션 범위|
|`localhost`가 많음|DB 서버 내부 프로그램 또는 로컬 접속|로컬 배치, 모니터링, 백업 스크립트|
|알 수 없는 외부 IP|비정상 접속 또는 미식별 시스템 가능|계정, 방화벽, 애플리케이션 서버 목록|

## 6. DBCP 장애와 연결해서 해석

이전 설정이 다음과 같다면:

```xml
<property name="initialSize" value="10"/>
<property name="maxTotal" value="100"/>
<property name="maxIdle" value="10"/>
<property name="minIdle" value="10"/>
```

`Host`별로 다음처럼 나오는 것은 정상 가능성이 있습니다.

```text
10.10.1.11 Sleep 10
10.10.1.12 Sleep 10
10.10.1.13 Sleep 10
10.10.1.14 Sleep 10
```

해석:

```text
WAS 4대 × minIdle 10 = 기본 Sleep 40개
```

하지만 이렇게 나오면 위험합니다.

```text
10.10.1.11 Sleep 100
```

단일 WAS에서 `Sleep`이 100개라면 다음을 의심해야 합니다.

```text
1. 해당 서버에 DataSource가 여러 개 있음
2. DBCP 설정이 실제로 maxIdle=10으로 적용되지 않음
3. 애플리케이션이 connection을 빌린 채 Pool에 반환하지 않음
4. 긴 트랜잭션 내부에서 SQL 실행 후 외부 API/파일/메일 대기 중
5. Connection leak 발생
```

## 7. Host만으로 알 수 없는 것

`Host`는 “어디에서 접속했는지”만 알려줍니다. 아래는 `Host`만으로 확정할 수 없습니다.

|알 수 없는 것|추가 확인 방법|
|---|---|
|Pool에 반납된 idle인지|DBCP `numIdle`, `numActive` 확인|
|Connection leak인지|Thread Dump, DBCP abandoned log 확인|
|어떤 Java 메서드가 잡고 있는지|WAS Thread Dump, APM, SQL 로그 확인|
|트랜잭션이 열려 있는지|`information_schema.INNODB_TRX` 확인|
|어떤 DataSource인지|애플리케이션 설정, DB 계정/DB명 분리 확인|

## 8. 바로 적용 가능한 확인 절차

```text
1. SHOW FULL PROCESSLIST에서 Host별 connection 수를 집계한다.
2. Host의 IP가 어느 WAS/배치 서버인지 매핑한다.
3. 특정 Host에 Sleep이 집중되어 있는지 확인한다.
4. 같은 시점에 해당 WAS의 DBCP numActive, numIdle, numWaiters를 확인한다.
5. Host별 Sleep 수가 minIdle × DataSource 수와 맞는지 계산한다.
6. Sleep 수가 비정상적으로 많으면 Thread Dump와 Connection Leak을 확인한다.
7. Query/Locked가 많은 Host는 SQL 실행계획과 Lock 상태를 확인한다.
```

## 최종 정리

```text
SHOW FULL PROCESSLIST의 Host는 MariaDB에 접속한 클라이언트 서버 주소이다.
`IP:PORT` 형식이면 IP는 접속한 WAS/배치/클라이언트 서버이고, PORT는 클라이언트 임시 포트이다.
Sleep connection 분석 시 Host를 기준으로 어느 서버가 connection을 많이 만들고 있는지 먼저 찾아야 한다.
단, Host만으로는 pool idle인지 active borrowed connection인지 알 수 없으므로 DBCP numActive/numIdle과 반드시 같이 봐야 한다.
```



정확도: 96%

# MariaDB Lock 때문에 `Sleep` 상태로 보이는 프로세스 확인 방법

## 1. 핵심 결론

`DB Lock 때문에 Sleep 상태에 들어간 쿼리`라는 표현은 정확히는 약간 구분해야 합니다.

```text
Lock을 기다리는 쿼리 = 보통 COMMAND가 Query이고 STATE에 Lock wait 관련 상태가 보임
Lock을 잡고 있는 세션 = 현재 SQL을 실행하지 않으면 COMMAND가 Sleep으로 보일 수 있음
```

즉, 장애 상황에서 중요한 것은 보통 **Sleep 상태인 세션이 Lock을 “기다리는” 것이 아니라, 이전 SQL로 Lock을 잡은 채 Commit/Rollback하지 않고 Sleep 상태로 남아 다른 쿼리를 막는 경우**입니다. MariaDB `INNODB_TRX`는 현재 실행 중인 InnoDB 트랜잭션과 상태(`RUNNING`, `LOCK WAIT` 등)를 보여주고, `INNODB_LOCK_WAITS`는 어떤 트랜잭션이 어떤 트랜잭션에 의해 막혔는지 매핑합니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/information-schema/information-schema-tables/information-schema-innodb-tables/information-schema-innodb_trx-table?utm_source=chatgpt.com "Information Schema INNODB_TRX Table | Server - MariaDB"))

## 2. 바로 확인하는 핵심 SQL

### 2.1 누가 누구를 막고 있는지 확인

```sql
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread_id,
    p_wait.USER AS waiting_user,
    p_wait.HOST AS waiting_host,
    p_wait.DB AS waiting_db,
    p_wait.COMMAND AS waiting_command,
    p_wait.TIME AS waiting_time,
    p_wait.STATE AS waiting_state,
    r.trx_state AS waiting_trx_state,
    r.trx_started AS waiting_trx_started,
    r.trx_wait_started AS waiting_started,
    r.trx_query AS waiting_query,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread_id,
    p_block.USER AS blocking_user,
    p_block.HOST AS blocking_host,
    p_block.DB AS blocking_db,
    p_block.COMMAND AS blocking_command,
    p_block.TIME AS blocking_time,
    p_block.STATE AS blocking_state,
    b.trx_state AS blocking_trx_state,
    b.trx_started AS blocking_trx_started,
    b.trx_query AS blocking_query
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r
    ON w.REQUESTING_TRX_ID = r.TRX_ID
JOIN information_schema.INNODB_TRX b
    ON w.BLOCKING_TRX_ID = b.TRX_ID
LEFT JOIN information_schema.PROCESSLIST p_wait
    ON r.TRX_MYSQL_THREAD_ID = p_wait.ID
LEFT JOIN information_schema.PROCESSLIST p_block
    ON b.TRX_MYSQL_THREAD_ID = p_block.ID
ORDER BY
    p_wait.TIME DESC;
```

해석 기준:

|항목|의미|
|---|---|
|`waiting_thread_id`|Lock 때문에 기다리는 세션 ID|
|`waiting_query`|막혀 있는 SQL|
|`blocking_thread_id`|Lock을 잡고 막고 있는 세션 ID|
|`blocking_command`|막고 있는 세션의 현재 상태. 여기서 `Sleep`이면 매우 중요|
|`blocking_query`|현재 실행 중인 쿼리. Sleep이면 `NULL`일 수 있음|
|`blocking_trx_started`|Lock을 잡은 트랜잭션 시작 시각|

## 3. Sleep 상태로 Lock을 잡고 있는 세션만 찾기

```sql
SELECT
    b.trx_mysql_thread_id AS blocking_thread_id,
    p_block.USER,
    p_block.HOST,
    p_block.DB,
    p_block.COMMAND,
    p_block.TIME AS sleep_time_sec,
    b.trx_state,
    b.trx_started,
    TIMESTAMPDIFF(SECOND, b.trx_started, NOW()) AS trx_age_sec,
    b.trx_tables_locked,
    b.trx_rows_locked,
    b.trx_query AS current_query,
    r.trx_mysql_thread_id AS waiting_thread_id,
    r.trx_query AS waiting_query
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r
    ON w.REQUESTING_TRX_ID = r.TRX_ID
JOIN information_schema.INNODB_TRX b
    ON w.BLOCKING_TRX_ID = b.TRX_ID
LEFT JOIN information_schema.PROCESSLIST p_block
    ON b.TRX_MYSQL_THREAD_ID = p_block.ID
WHERE p_block.COMMAND = 'Sleep'
ORDER BY
    trx_age_sec DESC;
```

이 결과가 나오면 의미는 다음입니다.

```text
Sleep 상태인 DB 세션이 열린 트랜잭션을 유지하고 있고,
그 트랜잭션이 잡고 있는 Lock 때문에 다른 쿼리가 대기 중이다.
```

## 4. 열린 트랜잭션 중 Sleep 상태인 세션 전체 확인

Lock wait가 아직 발생하지 않았더라도, 장시간 열린 트랜잭션은 위험합니다.

```sql
SELECT
    t.trx_mysql_thread_id AS thread_id,
    p.USER,
    p.HOST,
    p.DB,
    p.COMMAND,
    p.TIME AS process_time_sec,
    t.trx_state,
    t.trx_started,
    TIMESTAMPDIFF(SECOND, t.trx_started, NOW()) AS trx_age_sec,
    t.trx_tables_locked,
    t.trx_rows_locked,
    t.trx_lock_structs,
    t.trx_query
FROM information_schema.INNODB_TRX t
LEFT JOIN information_schema.PROCESSLIST p
    ON t.TRX_MYSQL_THREAD_ID = p.ID
WHERE p.COMMAND = 'Sleep'
ORDER BY
    trx_age_sec DESC;
```

이 조회에서 `trx_age_sec`가 크고 `trx_rows_locked`가 0보다 크면, 현재 SQL은 안 돌고 있어도 트랜잭션이 Lock을 오래 잡고 있을 가능성이 있습니다.

## 5. `SHOW FULL PROCESSLIST`만으로 1차 확인

```sql
SHOW FULL PROCESSLIST;
```

확인 포인트:

|컬럼|확인 내용|
|---|---|
|`Id`|세션 ID. `KILL` 대상이 될 수 있는 값|
|`User`|접속 계정|
|`Host`|접속한 WAS/배치/클라이언트 서버|
|`Command`|`Sleep`, `Query` 등 현재 명령 상태|
|`Time`|현재 상태로 머문 시간|
|`State`|Lock wait, Sending data 등 세부 상태|
|`Info`|현재 실행 중인 SQL. Sleep이면 보통 `NULL`|
|MariaDB `SHOW PROCESSLIST`는 현재 실행 중인 thread 정보를 보여주며, `Time`은 현재 상태에 머문 시간입니다. `SHOW FULL PROCESSLIST`를 사용하면 잘리지 않은 SQL 정보를 확인하는 데 유리합니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/show/show-engine-innodb-status?utm_source=chatgpt.com "SHOW ENGINE INNODB STATUS \| Server - MariaDB"))||

## 6. InnoDB 상세 Lock 상태 확인

```sql
SHOW ENGINE INNODB STATUS\G
```

확인할 구간:

```text
LATEST DETECTED DEADLOCK
TRANSACTIONS
------------
```

MariaDB 공식 문서 기준으로 `SHOW ENGINE INNODB STATUS`는 InnoDB 엔진 상태를 상세히 보여주며, deadlock, transaction, buffer pool, I/O 정보를 확인하는 데 사용됩니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/show/show-engine-innodb-status?utm_source=chatgpt.com "SHOW ENGINE INNODB STATUS | Server - MariaDB"))

## 7. 원인별 해석

|현상|가능한 원인|조치 방향|
|---|---|---|
|`blocking_command = Sleep`|트랜잭션 미종료 상태로 대기|애플리케이션 트랜잭션 범위 확인|
|`waiting_trx_state = LOCK WAIT`|다른 트랜잭션의 Lock 대기|Blocking 세션 추적|
|`blocking_trx_started`가 오래됨|장기 트랜잭션|Commit/Rollback 누락, 긴 Service 로직 확인|
|`blocking_query`가 `NULL`|현재 실행 SQL은 없음|이전 SQL이 Lock을 잡은 뒤 Sleep 상태|
|특정 `HOST` 집중|특정 WAS/배치 문제|해당 서버 Thread Dump, DBCP 상태 확인|

## 8. Spring/JDBC/DBCP 관점에서 흔한 원인

```java
@Transactional
public void process() {
    mapper.updateOrder();     // 여기서 row lock 획득
    externalApi.call();       // 이 동안 DB에서는 Sleep처럼 보일 수 있음
    mapper.updateResult();
}
```

이 경우 DB에서는 현재 SQL 실행이 없으므로 `Sleep`처럼 보일 수 있지만, Spring 트랜잭션이 끝나지 않았기 때문에 Connection과 Lock이 유지될 수 있습니다. Spring `DataSourceTransactionManager`는 JDBC Connection을 트랜잭션에 바인딩해 commit/rollback 시점까지 관리하므로, 트랜잭션 범위가 길면 DB Connection과 Lock 점유 시간도 길어질 수 있습니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/information-schema/information-schema-tables/information-schema-innodb-tables/information-schema-innodb_trx-table?utm_source=chatgpt.com "Information Schema INNODB_TRX Table | Server - MariaDB"))

## 9. 급한 경우 `KILL` 전 확인

`blocking_thread_id`가 명확하고 업무 영향 판단이 끝났다면 다음으로 종료할 수 있습니다.

```sql
KILL 12345;
```

단, 운영에서 바로 `KILL`하기 전에 아래를 확인해야 합니다.

```text
1. blocking_thread_id가 정말 Lock을 잡고 있는 세션인지
2. 해당 Host가 어느 WAS/배치 서버인지
3. 해당 세션이 중요한 배치/정산/주문 처리 중인지
4. Rollback 비용이 큰 대량 트랜잭션인지
5. 같은 문제가 재발하지 않도록 애플리케이션 원인을 확인했는지
```

## 10. 최종 점검 순서

```text
1. INNODB_LOCK_WAITS + INNODB_TRX + PROCESSLIST 조인으로 waiting/blocking 관계를 확인한다.
2. blocking_command가 Sleep인 세션을 찾는다.
3. blocking_thread_id의 HOST를 확인해 어느 WAS/배치 서버인지 찾는다.
4. blocking_trx_started와 trx_age_sec로 장기 트랜잭션 여부를 판단한다.
5. SHOW ENGINE INNODB STATUS\G로 InnoDB Lock/Deadlock 상세를 확인한다.
6. 해당 WAS에서 Thread Dump와 DBCP numActive/numIdle을 확인한다.
7. @Transactional 내부의 외부 API, 파일, 메일, 긴 루프, 수동 Connection close 누락을 점검한다.
8. 긴급 상황이면 blocking_thread_id만 제한적으로 KILL한다.
```

## 최종 정리

```text
DB Lock 장애에서 Sleep 세션은 보통 Lock을 기다리는 쿼리가 아니라,
Lock을 잡은 트랜잭션이 Commit/Rollback 없이 쉬고 있는 상태일 가능성이 높다.
따라서 SHOW FULL PROCESSLIST만 보지 말고,
INNODB_LOCK_WAITS, INNODB_TRX, PROCESSLIST를 조인해서
waiting_thread_id와 blocking_thread_id를 반드시 구분해야 한다.
```


정확도: 96%

# MariaDB `Sleep` 상태 프로세스 추적 방법

## 1. 결론

MariaDB에서 `Sleep` 상태 프로세스를 추적할 때 핵심은 **“현재 SQL을 실행하지 않는 세션”이 단순 idle인지, 애플리케이션이 잡고 있는 active connection인지, 열린 트랜잭션/Lock을 유지 중인지**를 구분하는 것입니다. `SHOW FULL PROCESSLIST`는 현재 실행 중인 thread와 `Info`를 보여주지만, `Sleep` 상태에서는 현재 실행 SQL이 없어서 `Info`가 `NULL`인 경우가 많습니다. `FULL`을 쓰지 않으면 `Info`가 100자까지만 보입니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/show/show-processlist?utm_source=chatgpt.com "SHOW PROCESSLIST | Server | MariaDB Documentation"))

## 2. 1차 확인: Sleep 세션 분포 확인

```sql
SELECT
    SUBSTRING_INDEX(HOST, ':', 1) AS CLIENT_HOST,
    USER,
    DB,
    COMMAND,
    COUNT(*) AS CNT
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
GROUP BY
    SUBSTRING_INDEX(HOST, ':', 1),
    USER,
    DB,
    COMMAND
ORDER BY CNT DESC;
```

해석:

```text
특정 HOST에 Sleep 집중
→ 해당 WAS/배치 서버의 Connection Pool, 트랜잭션, Connection Leak 우선 점검

여러 HOST에 10개씩 분산
→ WAS 수 × DataSource 수 × minIdle 설정에 따른 정상 idle 가능성
```

`information_schema.PROCESSLIST`는 실행 중인 thread의 상세 정보와 실행 시간을 제공하며, `ID`는 connection identifier입니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/information-schema/information-schema-tables/information-schema-processlist-table?utm_source=chatgpt.com "Information Schema PROCESSLIST Table | Server - MariaDB"))

## 3. 오래된 Sleep 세션 확인

```sql
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    STATE,
    INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND = 'Sleep'
ORDER BY TIME DESC
LIMIT 50;
```

판단 기준:

|항목|해석|
|---|---|
|`TIME` 큼|해당 세션이 Sleep 상태로 오래 머문 상태|
|`HOST` 특정 서버 집중|특정 WAS/배치 서버 문제 가능|
|`INFO` NULL|현재 실행 중인 SQL 없음|
|`DB` 동일|특정 업무 DB 커넥션 풀 문제 가능|

## 4. 가장 중요: Sleep인데 열린 트랜잭션인지 확인

`Sleep`이 단순 idle이면 보통 큰 문제는 아닙니다. 하지만 **트랜잭션이 열린 채 Sleep**이면 Lock, undo, MVCC 리소스, connection 점유 문제가 생길 수 있습니다.

```sql
SELECT
    t.TRX_MYSQL_THREAD_ID AS THREAD_ID,
    p.USER,
    p.HOST,
    p.DB,
    p.COMMAND,
    p.TIME AS SLEEP_TIME_SEC,
    t.TRX_STATE,
    t.TRX_STARTED,
    TIMESTAMPDIFF(SECOND, t.TRX_STARTED, NOW()) AS TRX_AGE_SEC,
    t.TRX_TABLES_LOCKED,
    t.TRX_ROWS_LOCKED,
    t.TRX_LOCK_STRUCTS,
    t.TRX_QUERY
FROM information_schema.INNODB_TRX t
LEFT JOIN information_schema.PROCESSLIST p
    ON t.TRX_MYSQL_THREAD_ID = p.ID
WHERE p.COMMAND = 'Sleep'
ORDER BY TRX_AGE_SEC DESC;
```

`INNODB_TRX`는 현재 실행 중인 InnoDB 트랜잭션 정보를 저장하며, 트랜잭션 상태, 시작 시각, Lock 정보를 확인할 수 있습니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/information-schema/information-schema-tables/information-schema-innodb-tables/information-schema-innodb_trx-table?utm_source=chatgpt.com "Information Schema INNODB_TRX Table | Server - MariaDB"))  
해석:

|결과|판단|
|---|---|
|조회 결과 없음|Sleep 세션 중 열린 InnoDB 트랜잭션은 현재 없음|
|`TRX_AGE_SEC` 큼|장시간 열린 트랜잭션 가능|
|`TRX_ROWS_LOCKED > 0`|Lock을 잡고 있을 가능성|
|`TRX_QUERY` NULL|현재 실행 쿼리는 없지만 트랜잭션은 살아 있을 수 있음|

## 5. Lock을 유발하는 Sleep 세션 확인

다른 쿼리를 막고 있는 `Sleep` 세션을 찾으려면 `INNODB_LOCK_WAITS`, `INNODB_TRX`, `PROCESSLIST`를 조인합니다.

```sql
SELECT
    b.TRX_MYSQL_THREAD_ID AS BLOCKING_THREAD_ID,
    p_block.USER AS BLOCKING_USER,
    p_block.HOST AS BLOCKING_HOST,
    p_block.DB AS BLOCKING_DB,
    p_block.COMMAND AS BLOCKING_COMMAND,
    p_block.TIME AS BLOCKING_TIME,
    b.TRX_STATE AS BLOCKING_TRX_STATE,
    b.TRX_STARTED AS BLOCKING_TRX_STARTED,
    TIMESTAMPDIFF(SECOND, b.TRX_STARTED, NOW()) AS BLOCKING_TRX_AGE_SEC,
    b.TRX_ROWS_LOCKED AS BLOCKING_ROWS_LOCKED,
    b.TRX_QUERY AS BLOCKING_QUERY,
    r.TRX_MYSQL_THREAD_ID AS WAITING_THREAD_ID,
    p_wait.USER AS WAITING_USER,
    p_wait.HOST AS WAITING_HOST,
    p_wait.STATE AS WAITING_STATE,
    r.TRX_STATE AS WAITING_TRX_STATE,
    r.TRX_QUERY AS WAITING_QUERY
FROM information_schema.INNODB_LOCK_WAITS w
JOIN information_schema.INNODB_TRX r
    ON w.REQUESTING_TRX_ID = r.TRX_ID
JOIN information_schema.INNODB_TRX b
    ON w.BLOCKING_TRX_ID = b.TRX_ID
LEFT JOIN information_schema.PROCESSLIST p_wait
    ON r.TRX_MYSQL_THREAD_ID = p_wait.ID
LEFT JOIN information_schema.PROCESSLIST p_block
    ON b.TRX_MYSQL_THREAD_ID = p_block.ID
WHERE p_block.COMMAND = 'Sleep'
ORDER BY BLOCKING_TRX_AGE_SEC DESC;
```

`INNODB_LOCKS`는 요청했지만 아직 얻지 못한 Lock 또는 다른 트랜잭션을 막고 있는 Lock 정보를 제공하므로, Lock 분석에는 `INNODB_TRX`, `INNODB_LOCK_WAITS`, `INNODB_LOCKS` 계열을 함께 보는 방식이 적합합니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/information-schema/information-schema-tables/information-schema-innodb-tables/information-schema-innodb_locks-table?utm_source=chatgpt.com "Information Schema INNODB_LOCKS Table | Server - MariaDB"))

## 6. Sleep 세션의 “직전 SQL” 추적

중요한 한계가 있습니다.

```text
SHOW FULL PROCESSLIST만으로는 Sleep 세션이 직전에 실행한 SQL을 항상 알 수 없다.
```

`Sleep`은 현재 SQL 실행이 없는 상태이므로 `Info`가 비어 있을 수 있습니다. 직전 SQL을 추적하려면 아래 중 하나가 필요합니다.

### 6.1 Performance Schema history 확인

활성화되어 있다면 최근 완료 SQL을 볼 수 있습니다.

```sql
SELECT
    THREAD_ID,
    EVENT_ID,
    EVENT_NAME,
    SQL_TEXT,
    TIMER_WAIT,
    LOCK_TIME,
    ROWS_AFFECTED,
    ROWS_SENT,
    ROWS_EXAMINED
FROM performance_schema.events_statements_history
WHERE SQL_TEXT IS NOT NULL
ORDER BY THREAD_ID, EVENT_ID DESC;
```

MariaDB 문서 기준으로 `events_statements_history`는 thread별 최근 완료 statement event를 기록합니다. 기본적으로 thread당 최근 10개 완료 statement를 저장합니다. ([MariaDB](https://mariadb.com/docs/server/reference/system-tables/performance-schema/performance-schema-tables/performance-schema-events_statements_history-table?utm_source=chatgpt.com "Performance Schema events_statements_history Table | Server"))  
단, `PROCESSLIST.ID`와 Performance Schema `THREAD_ID`를 직접 매핑하려면 환경별로 thread 관련 테이블을 추가 확인해야 할 수 있습니다. 운영 DB에서 Performance Schema가 비활성화되어 있거나 history size가 작으면 원하는 직전 SQL이 없을 수 있습니다.

### 6.2 General Query Log로 사후 추적

앞으로 발생할 Sleep 원인을 추적하려면 General Query Log를 임시로 켤 수 있습니다.

```sql
SHOW VARIABLES LIKE 'general_log';
SHOW VARIABLES LIKE 'general_log_file';
SET GLOBAL general_log = 'ON';
```

중지:

```sql
SET GLOBAL general_log = 'OFF';
```

General Query Log는 클라이언트가 보낸 모든 SQL과 connect/disconnect를 기록하므로 원인 추적에는 강력하지만, 로그가 매우 빠르게 커질 수 있습니다. 운영에서는 짧은 시간만 제한적으로 켜야 합니다. ([MariaDB](https://mariadb.com/docs/server/server-management/server-monitoring-logs/general-query-log?utm_source=chatgpt.com "General Query Log | Server | MariaDB Documentation"))

### 6.3 Slow Query Log 확인

느린 쿼리로 인해 커넥션 점유가 길어지는 경우는 Slow Query Log가 더 적합합니다.

```sql
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'slow_query_log_file';
```

Slow Query Log는 오래 걸린 SQL과 영향을 받은 row 수 등을 기록합니다. 다만 민감한 값이 SQL에 포함될 수 있으므로 로그 보관에 주의해야 합니다. ([MariaDB](https://mariadb.com/docs/server/server-management/server-monitoring-logs/slow-query-log/slow-query-log-overview?utm_source=chatgpt.com "Slow Query Log Overview | Server | MariaDB Documentation"))

## 7. Spring JDBC + DBCP2 관점에서 같이 봐야 할 값

DB에서 `Sleep`으로 보여도 애플리케이션 Pool에서는 두 가지가 가능합니다.

|DB|DBCP|의미|
|---|---|---|
|Sleep|`numIdle`|Pool에 반납된 재사용 가능 커넥션|
|Sleep|`numActive`|애플리케이션이 빌려간 상태, 재사용 불가|
|Sleep + INNODB_TRX 존재|대부분 위험|트랜잭션 미종료 가능|
|Sleep + Lock blocking|위험|다른 쿼리를 막는 세션|
|애플리케이션에서 같은 시점에 확인:|||

```java
BasicDataSource ds = dataSource;
System.out.println("maxTotal=" + ds.getMaxTotal());
System.out.println("maxIdle=" + ds.getMaxIdle());
System.out.println("minIdle=" + ds.getMinIdle());
System.out.println("numActive=" + ds.getNumActive());
System.out.println("numIdle=" + ds.getNumIdle());
```

판단:

```text
DB Sleep 100, DBCP numIdle 10, numActive 90
→ 대부분은 Pool에 반납된 idle이 아니라 애플리케이션이 잡고 있는 커넥션 가능성

DB Sleep 100, WAS 10대, 각 WAS numIdle 10
→ minIdle 누적으로 정상 가능성
```

## 8. Linux/WAS 쪽 추적

특정 `HOST`가 확인되면 해당 WAS에서 Thread Dump를 잡습니다.

```bash
jps -l
jstack -l <PID> > /tmp/jstack_$(date +%Y%m%d_%H%M%S).log
```

3회 이상 반복:

```bash
for i in 1 2 3; do
  jstack -l <PID> > /tmp/jstack_${i}.log
  sleep 10
done
```

검색:

```bash
grep -nE "java.sql|org.springframework.jdbc|DataSourceTransactionManager|org.mybatis|jdbc|WAITING|BLOCKED" /tmp/jstack_*.log
```

소스 검색:

```bash
grep -RIn "getConnection()" /app/source
grep -RIn "@Transactional" /app/source
grep -RIn "REQUIRES_NEW" /app/source
grep -RIn "RestTemplate\|WebClient\|OkHttp\|HttpClient\|sendMail\|File" /app/source
```

특히 아래 구조를 의심해야 합니다.

```java
@Transactional
public void process() {
    mapper.updateSomething();   // Lock 또는 Connection 확보
    externalApi.call();         // 이 동안 DB에서는 Sleep처럼 보일 수 있음
    mapper.updateResult();
}
```

## 9. 긴급 조치: `KILL` 전 확인

특정 Sleep 세션이 Lock을 막고 있는 것이 확실하면 `KILL`을 검토할 수 있습니다.

```sql
KILL 12345;
```

다만 바로 실행하면 안 됩니다.

```text
1. BLOCKING_THREAD_ID가 맞는지 확인
2. HOST가 어느 WAS/배치인지 확인
3. 정산/주문/배치 등 중요한 트랜잭션인지 확인
4. Rollback 비용이 큰 대량 작업인지 확인
5. 동일 원인 재발 방지 조치 준비
```

`KILL`은 결과적으로 해당 세션의 작업을 중단시키므로, 장애 완화 수단이지 근본 해결책은 아닙니다.

## 10. 바로 적용 가능한 추적 순서

```text
1. PROCESSLIST에서 Sleep을 HOST별로 집계한다.
2. 오래된 Sleep 세션을 TIME 기준으로 확인한다.
3. INNODB_TRX와 조인해 열린 트랜잭션이 있는 Sleep을 찾는다.
4. INNODB_LOCK_WAITS와 조인해 다른 쿼리를 막는 Sleep을 찾는다.
5. 특정 HOST가 나오면 해당 WAS의 DBCP numActive/numIdle을 확인한다.
6. DB Sleep과 DBCP active/idle을 비교해 재사용 가능 커넥션인지 구분한다.
7. Performance Schema history 또는 General Query Log로 직전 SQL 추적 가능성을 확인한다.
8. WAS Thread Dump로 어떤 Java 로직이 커넥션을 오래 잡는지 찾는다.
9. 원인이 긴 트랜잭션이면 @Transactional 범위를 DB 처리 구간으로 축소한다.
10. 원인이 idle 과다이면 initialSize/minIdle/maxIdle을 WAS 수 기준으로 재산정한다.
```

## 최종 정리

```text
MariaDB의 Sleep 프로세스는 현재 SQL을 실행하지 않는 연결이다.
정상 Pool idle이면 재사용 가능하지만, 애플리케이션이 커넥션을 빌린 채 대기 중이어도 DB에서는 Sleep으로 보일 수 있다.
따라서 Sleep 추적은 PROCESSLIST만 보지 말고 INNODB_TRX, INNODB_LOCK_WAITS, DBCP numActive/numIdle, WAS Thread Dump를 함께 봐야 한다.
직전 SQL은 SHOW PROCESSLIST만으로는 한계가 있으므로 Performance Schema history, General Query Log, Slow Query Log, 애플리케이션 SQL 로그를 병행해야 한다.
```