## 결론

MariaDB 10.6에서 현재 DB 부하를 실무적으로 확인하려면 최소한 아래 **4단계**를 같이 보는 것이 좋습니다.

1. `Threads_running`, 연결 수 → **현재 동시 부하**
2. `SHOW FULL PROCESSLIST` → **어떤 SQL이 부하를 발생시키는지**
3. `SHOW GLOBAL STATUS`를 5~30초 간격으로 비교 → **QPS/TPS, Disk I/O, Temporary Table 등의 실제 발생률**
4. `SHOW ENGINE INNODB STATUS` / `INNODB_LOCK_WAITS` → **Lock, I/O, Buffer Pool, Transaction 병목**

특히 `Questions`, `Com_select`, `Innodb_buffer_pool_reads` 등은 **누적값이므로 숫자 자체가 크다고 현재 부하가 높은 것이 아닙니다.** 일정 시간 간격의 증가량을 봐야 합니다. MariaDB 공식 문서에서도 `SHOW GLOBAL STATUS`는 전체 연결에 대한 서버 상태 카운터를 제공한다고 명시합니다. 

---

## 1. 가장 먼저 실행할 현재 부하 확인 SQL

운영 장애나 응답지연이 발생했을 때 가장 먼저 보는 것을 추천합니다.

```sql
SHOW GLOBAL STATUS
WHERE Variable_name IN (
    'Threads_connected',
    'Threads_running',
    'Max_used_connections',
    'Questions',
    'Queries',
    'Slow_queries',
    'Created_tmp_tables',
    'Created_tmp_disk_tables',
    'Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_read_requests',
    'Innodb_data_reads',
    'Innodb_data_writes',
    'Innodb_row_lock_waits',
    'Innodb_row_lock_time',
    'Aborted_connects',
    'Aborted_clients'
);
```

### 주요 값 해석

|항목|의미|부하 판단|
|---|---|---|
|`Threads_running`|현재 실제 실행 중인 스레드|현재 DB 동시부하 판단에 매우 중요|
|`Threads_connected`|현재 DB 연결 수|Connection Pool 상태 확인|
|`Max_used_connections`|기동 이후 최대 연결 수|`max_connections` 대비 확인|
|`Questions`|클라이언트 요청 SQL 누적량|시간 차이를 이용해 QPS 계산|
|`Slow_queries`|Slow Query 누적 횟수|증가속도 확인|
|`Created_tmp_tables`|Memory Temporary Table|GROUP BY/SORT 등 부하 참고|
|`Created_tmp_disk_tables`|Disk Temporary Table|높으면 I/O 병목 가능성|
|`Innodb_buffer_pool_reads`|디스크에서 실제 읽은 횟수|Buffer Pool 효율 판단|
|`Innodb_buffer_pool_read_requests`|Buffer Pool 논리 read|Hit Ratio 계산|
|`Innodb_data_reads/writes`|InnoDB 물리 I/O|스토리지 부하 참고|
|`Innodb_row_lock_waits`|Row Lock 대기 누적|동시성 병목|
|`Innodb_row_lock_time`|Lock 대기시간|Transaction 병목|

`SHOW GLOBAL STATUS`는 서버 전체 상태를 보는 명령이며, `SHOW STATUS`만 실행하면 기본적으로 현재 세션 값이 나오므로 운영 부하 확인에서는 반드시 `GLOBAL`을 붙이는 것이 좋습니다. 

---

## 2. 현재 DB가 실제로 바쁜지 판단

가장 먼저 볼 값은 사실상 이것입니다.

```sql
SHOW GLOBAL STATUS LIKE 'Threads_running';
```

그리고:

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
```

설정값도 같이 확인합니다.

```sql
SHOW VARIABLES LIKE 'max_connections';
```

예를 들어:

```text
max_connections     = 200
Threads_connected   = 145
Threads_running     = 47
```

라면 연결 수도 많고 실제 실행 중인 Query도 상당히 많은 상태일 가능성이 있습니다.

반면:

```text
Threads_connected   = 145
Threads_running     = 3
```

이라면 연결은 많아도 대부분 Connection Pool에서 idle 상태일 수 있으므로 **145개 연결 = DB 부하 145**라고 판단하면 안 됩니다.

Spring + DBCP2 환경에서는 특히 `Threads_connected`가 많다는 이유만으로 DB 병목으로 판정하면 안 됩니다. Pool은 연결을 유지하는 것이 정상적인 동작이기 때문입니다.

---

## 3. 누가 현재 DB 부하를 발생시키는지

가장 중요한 명령 중 하나입니다.

```sql
SHOW FULL PROCESSLIST;
```

MariaDB 공식 문서 기준 `FULL`을 사용하지 않으면 실행 SQL의 `Info`가 앞부분 100자까지만 표시됩니다. `PROCESS` 권한이 있으면 전체 사용자 Thread를 볼 수 있고, 없으면 자신의 Thread만 볼 수 있습니다. 

실무에서는 다음 방법이 더 편합니다.

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
WHERE COMMAND <> 'Sleep'
ORDER BY TIME DESC;
```

MariaDB는 `TIME_MS`도 제공합니다.

```sql
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    TIME_MS,
    STATE,
    INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND <> 'Sleep'
ORDER BY TIME_MS DESC;
```

`TIME_MS`는 현재 상태가 지속된 시간을 밀리초 단위로 제공하므로 짧은 Query가 많은 서비스에서는 상당히 유용합니다. 

### 특히 주의할 상태

```text
Waiting for table metadata lock
Locked
Waiting for...
Sending data
Creating sort index
Copying to tmp table
```

다만 `Sending data`가 오래됐다고 반드시 네트워크 문제라는 뜻은 아닙니다. 실제로는 데이터 탐색·조인·결과 생성 과정까지 포함할 수 있으므로 실행계획까지 추가 확인해야 합니다.

---

## 4. 5~10초 간격으로 측정해야 진짜 부하가 보임

예를 들어 현재:

```sql
SHOW GLOBAL STATUS LIKE 'Questions';
```

결과가:

```text
Questions = 123456789
```

라고 해도 이것만으로 현재 부하를 전혀 판단할 수 없습니다.

10초 후:

```text
123461789
```

라면:

```text
(123461789 - 123456789) / 10
= 500 QPS
```

즉 약:

```text
500 queries/sec
```

입니다.

### SQL만으로 간단하게 측정

테스트/진단 환경이라면 다음과 같이 할 수도 있습니다.

```sql
SELECT VARIABLE_VALUE INTO @Q1
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'QUESTIONS';

DO SLEEP(10);

SELECT
    (
        CAST(VARIABLE_VALUE AS UNSIGNED) - CAST(@Q1 AS UNSIGNED)
    ) / 10 AS QPS
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME = 'QUESTIONS';
```

단, 운영 모니터링 프로그램에서 `SLEEP()`을 넣기보다는 외부 프로그램이 10초 간격으로 두 번 조회하는 것이 더 낫습니다.

---

## 5. Connection 사용률

다음 SQL도 유용합니다.

```sql
SELECT
    @@max_connections AS max_connections,
    MAX(
        CASE
            WHEN VARIABLE_NAME = 'THREADS_CONNECTED'
            THEN CAST(VARIABLE_VALUE AS UNSIGNED)
        END
    ) AS threads_connected,
    MAX(
        CASE
            WHEN VARIABLE_NAME = 'THREADS_RUNNING'
            THEN CAST(VARIABLE_VALUE AS UNSIGNED)
        END
    ) AS threads_running
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'THREADS_CONNECTED',
    'THREADS_RUNNING'
);
```

Connection 사용률은 대략:

```text
Threads_connected / max_connections × 100
```

으로 계산할 수 있습니다.

예:

```text
max_connections    = 200
Threads_connected  = 160

connection usage = 80%
```

그러나 다시 강조하면 이것은 **Connection 포화도**이지 CPU/DB 처리 부하 자체는 아닙니다.

---

## 6. InnoDB Buffer Pool 상태

MariaDB 10.6 + InnoDB에서는 매우 중요합니다.

```sql
SHOW GLOBAL STATUS
WHERE Variable_name IN (
    'Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_read_requests'
);
```

대략적인 Buffer Pool Hit Ratio는:

```text
1 -
Innodb_buffer_pool_reads
--------------------------
Innodb_buffer_pool_read_requests
```

입니다.

SQL로 계산하면:

```sql
SELECT
    ROUND(
        (
            1 -
            (
                MAX(CASE
                        WHEN VARIABLE_NAME = 'INNODB_BUFFER_POOL_READS'
                        THEN CAST(VARIABLE_VALUE AS UNSIGNED)
                    END)
                /
                NULLIF(
                    MAX(CASE
                            WHEN VARIABLE_NAME = 'INNODB_BUFFER_POOL_READ_REQUESTS'
                            THEN CAST(VARIABLE_VALUE AS UNSIGNED)
                        END),
                    0
                )
            )
        ) * 100,
        4
    ) AS buffer_pool_hit_ratio
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'INNODB_BUFFER_POOL_READS',
    'INNODB_BUFFER_POOL_READ_REQUESTS'
);
```

단 이것도 서버 기동 이후 누적값이므로 **현재 순간의 성능 문제 분석에서는 10~60초 delta 기반 Hit Ratio를 계산하는 것이 훨씬 정확합니다.**

---

## 7. InnoDB 자체가 병목인지 확인

```sql
SHOW ENGINE INNODB STATUS\G
```

매우 중요한 진단 명령입니다.

여기서는 다음을 확인할 수 있습니다.

```text
TRANSACTIONS
SEMAPHORES
FILE I/O
LOG
BUFFER POOL AND MEMORY
ROW OPERATIONS
LATEST DETECTED DEADLOCK
```

MariaDB 공식 문서에서도 이 명령을 Deadlock, Buffer Pool, I/O, semaphore contention 등을 진단하기 위한 주요 도구로 설명합니다. 

특히 다음 부분을 확인합니다.

```text
queries inside InnoDB
queries in queue

Pending reads
Pending writes

reads/s
writes/s

Log sequence number
Last checkpoint at
```

예를 들어:

```text
20 queries inside InnoDB
35 queries in queue
```

상태가 지속된다면 상당히 중요한 병목 신호입니다.

---

## 8. Lock 때문에 부하가 생긴 것인지 확인

현재 Lock Wait:

```sql
SELECT *
FROM information_schema.INNODB_LOCK_WAITS;
```

MariaDB 공식 문서상 이 테이블은 대기 Transaction과 Blocking Transaction을 연결해서 볼 수 있으며 `PROCESS` 권한이 필요합니다. 

Transaction도 확인합니다.

```sql
SELECT
    trx_id,
    trx_state,
    trx_started,
    trx_wait_started,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;
```

실무적으로는 **오래 열린 Transaction**이 상당히 중요합니다.

예:

```text
TRX_STARTED    10분 전
TRX_STATE      RUNNING
TRX_QUERY      NULL
```

이라고 해서 안전한 것이 아닙니다.

Spring Transaction에서 SQL은 끝났지만 Commit/Rollback되지 않은 채 Transaction이 유지되고 있을 수도 있습니다.

이 경우 다음 문제가 발생할 수 있습니다.

```text
Undo 증가
Purge 지연
Row Lock 장기 보유
MVCC old version 증가
다른 Transaction 차단
```

---

## 9. MariaDB 10.6의 Performance Schema

먼저 확인합니다.

```sql
SHOW VARIABLES LIKE 'performance_schema';
```

결과:

```text
performance_schema ON
```

이라면 보다 정밀한 분석이 가능합니다.

MariaDB에서는 Performance Schema가 기본적으로 비활성화될 수 있고, 활성화 여부는 `performance_schema` 변수로 확인합니다. 런타임에서 단순 ON 전환할 수 있는 기능이 아니므로 서버 시작 설정이 필요합니다. 

예를 들어 SQL 유형별 누적 부하:

```sql
SELECT
    SCHEMA_NAME,
    DIGEST_TEXT,
    COUNT_STAR,
    ROUND(SUM_TIMER_WAIT / 1000000000000, 2) AS total_sec,
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS avg_ms,
    SUM_ROWS_EXAMINED,
    SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

이를 통해 단순히 현재 오래 실행되는 SQL뿐 아니라 **누적 DB 시간을 가장 많이 소비한 SQL 유형**을 찾을 수 있습니다.

Performance Schema digest는 literal 값을 제거하여 동일 구조 SQL을 하나의 패턴으로 묶어 통계화합니다. 

예:

```sql
SELECT * FROM order WHERE order_id = 100;
SELECT * FROM order WHERE order_id = 200;
```

를 개념적으로:

```sql
SELECT * FROM order WHERE order_id = ?;
```

처럼 묶어서 집계합니다.

운영 튜닝에서는 매우 유용합니다.

---

## 10. MariaDB 10.6에서 사용할 수 있는 sys.metrics

MariaDB **10.6부터 sys schema view가 제공**되므로 설치/환경 상태에 따라 다음도 상당히 편리합니다.

```sql
SELECT *
FROM sys.metrics;
```

`sys.metrics`는 Global Status, InnoDB Metrics, Performance Schema 메모리 정보 등을 하나의 View에서 제공하도록 구성되어 있습니다. 

다만 기존 운영 DB에서 Performance Schema를 OFF로 사용하고 있다면 이를 켜기 위해 DB 재기동까지 강제할 필요는 없습니다. 먼저 `SHOW GLOBAL STATUS`, `PROCESSLIST`, `INNODB STATUS`만으로 진단하는 것이 안전합니다.

---

## 11. 필요한 권한

실무에서는 다음처럼 정리할 수 있습니다.

| 측정                                  | 명령                                     | 권한                              |
| ----------------------------------- | -------------------------------------- | ------------------------------- |
| 기본 서버 부하                            | `SHOW GLOBAL STATUS`                   | 일반 계정으로 조회 가능                   |
| 설정 확인                               | `SHOW VARIABLES`                       | 일반적인 변수 조회 가능                   |
| 자기 Connection                       | `SHOW PROCESSLIST`                     | 별도 `PROCESS` 불필요                |
| 전체 Connection                       | `SHOW FULL PROCESSLIST`                | `PROCESS` 필요                    |
| 전체 `information_schema.PROCESSLIST` | SELECT                                 | 전체 Thread 확인에는 `PROCESS` 필요     |
| InnoDB Lock/Transaction 진단          | `INNODB_LOCK_WAITS`, 일부 InnoDB I_S 테이블 | `PROCESS` 필요                    |
| Buffer Pool 상세 정보                   | `INNODB_BUFFER_POOL_STATS`             | `PROCESS` 필요                    |
| Performance Schema                  | P_S 테이블 SELECT                         | 해당 P_S 조회 권한 필요                 |
| Performance Schema 설정 변경            | `setup_* UPDATE`                       | 추가 UPDATE/관리 권한 필요              |
| SQL 종료                              | `KILL`                                 | 다른 사용자의 Thread 종료에는 추가 관리 권한 필요 |

MariaDB 공식 문서에서도 `PROCESS` 권한은 현재 실행 중인 Query의 평문을 볼 수 있도록 하는 서버 관리 권한으로 정의합니다. 

---

## 12. 운영 모니터링 계정 권한 추천

운영 Application 계정에 `PROCESS`를 추가하는 것은 권장하지 않습니다.

별도 계정:

```text
db_monitor
```

를 만드는 것이 좋습니다.

예:

```sql
CREATE USER 'db_monitor'@'10.10.10.%'
IDENTIFIED BY '강력한비밀번호';

GRANT PROCESS ON *.* TO 'db_monitor'@'10.10.10.%';
```

Performance Schema를 사용한다면 환경에 맞추어 필요한 조회권한만 별도로 부여합니다.

중요한 점은 모니터링 목적으로:

```sql
GRANT ALL PRIVILEGES ON *.* ...
```

를 주면 안 된다는 것입니다.

`PROCESS` 자체도 전체 사용자의 실행 SQL을 노출할 수 있는 강한 권한이기 때문에 접속 IP를 제한하는 것이 좋습니다.

예:

```text
'db_monitor'@'%'
```

보다는:

```text
'db_monitor'@'10.20.30.15'
```

같은 방식이 안전합니다.

---

## 13. OS 부하까지 반드시 함께 봐야 함

여기서 중요한 한계가 하나 있습니다.

**MariaDB SQL만으로 서버의 실제 CPU%, Disk util%, iowait를 완전히 판단할 수 없습니다.**

DB 서버 Linux라면 같이 봐야 합니다.

```bash
top
```

또는:

```bash
pidstat -p $(pidof mariadbd) 1
```

CPU:

```bash
mpstat -P ALL 1
```

Disk:

```bash
iostat -xz 1
```

Memory:

```bash
free -m
```

특히:

```bash
iostat -xz 1
```

에서 다음이 중요합니다.

```text
%util
await
r/s
w/s
rkB/s
wkB/s
```

따라서 다음과 같은 상황을 구분할 수 있습니다.

```text
Threads_running ↑
CPU 95%
Disk util 낮음
```

→ CPU/SQL 연산 병목 가능성

반면:

```text
Threads_running ↑
CPU 낮음
iowait ↑
Disk await ↑
Disk %util ↑
```

→ Storage I/O 병목 가능성

그리고:

```text
Threads_running ↑
CPU 낮음
Disk 낮음
INNODB_LOCK_WAITS ↑
```

→ Lock/Transaction contention 가능성이 큽니다.

---

## 14. 우선 사용할 진단 순서

현재처럼 **MariaDB 10.6 + Spring 5.3 + DBCP2 + 커머스 시스템**이라면 장애 시 저는 다음 순서를 권합니다. 

```sql
-- ① 현재 실행 부하
SHOW GLOBAL STATUS
WHERE Variable_name IN (
    'Threads_connected',
    'Threads_running',
    'Max_used_connections',
    'Questions',
    'Slow_queries'
);

-- ② 현재 실행 SQL
SELECT
    ID,
    USER,
    HOST,
    DB,
    COMMAND,
    TIME,
    TIME_MS,
    STATE,
    INFO
FROM information_schema.PROCESSLIST
WHERE COMMAND <> 'Sleep'
ORDER BY TIME_MS DESC;

-- ③ Lock Wait
SELECT *
FROM information_schema.INNODB_LOCK_WAITS;

-- ④ 장기 Transaction
SELECT
    trx_id,
    trx_state,
    trx_started,
    trx_wait_started,
    trx_mysql_thread_id,
    trx_query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;

-- ⑤ InnoDB 내부 병목
SHOW ENGINE INNODB STATUS;

-- ⑥ Performance Schema
SHOW VARIABLES LIKE 'performance_schema';
```

그리고 서버에서 동시에:

```bash
top
iostat -xz 1
vmstat 1
```

을 확인하는 방식이 가장 효율적입니다.

### 최종적으로 보는 관계

```text
                MariaDB 부하
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   SQL 실행량    동시성/Lock     자원
        │           │           │
      QPS       Threads       CPU
      TPS       Lock Wait     Memory
   Slow SQL     Long TRX      Disk I/O
        │           │           │
        └───────────┼───────────┘
                    ▼
                 병목 원인
```


**단순 DB 부하 측정 목적이라면 우선 `PROCESS` 권한을 가진 읽기 전용 모니터링 계정 + `SHOW GLOBAL STATUS` + `PROCESSLIST` + InnoDB 진단 정도면 충분합니다.**

원하시면 다음 단계로 **MariaDB 10.6에서 SQL 하나만 실행하면 `현재 Connection 사용률 / QPS / TPS / Slow Query / Buffer Pool Hit Ratio / Lock Wait / Disk Temporary Table 비율`을 한 화면에 보여주는 “DB 부하 점검용 SQL”**을 실제 운영에서 사용 가능하도록 만들어드릴 수 있습니다.