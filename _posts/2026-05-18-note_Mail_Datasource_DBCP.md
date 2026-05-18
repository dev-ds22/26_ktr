---
layout: single
title: "Mail_Datasource_DBCP"
excerpt: "Mail_Datasource_DBCP"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-18"
last_modified_at: "2026-05-18 11:51:52 +0900"
---
# Mail Datasource DBCP2 설정
```xml
<property name="initialSize" value="10"/>
<property name="maxTotal" value="10"/>
<property name="maxIdle" value="10"/>
<property name="minIdle" value="10"/>
<property name="maxWaitMillis" value="3000"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="false"/>
<property name="testOnReturn" value="false"/>
<property name="testWhileIdle" value="false"/>
<property name="timeBetweenEvictionRunsMillis" value="480000"/>
<property name="numTestsPerEvictionRun" value="4"/>
<property name="minEvictableIdleTimeMillis" value="-1"/>
<property name="poolPreparedStatements" value="true"/>
<property name="maxOpenPreparedStatements" value="30"/>****
```
# DBCP2 설정 위험도 분석

## 1. 최종 판단

현재 설정은 **소규모·저빈도 외부솔루션용 Datasource**라면 즉시 장애가 나지 않을 수 있지만, 운영 관점에서는 **중간~높음 위험**입니다. 특히 핵심 문제는 다음 3가지입니다.

```text
1. maxTotal=10으로 동시 커넥션 한계가 매우 낮음
2. testOnBorrow=false, testWhileIdle=false라서 죽은 커넥션 검증 없이 애플리케이션에 전달 가능
3. minIdle=maxIdle=maxTotal=10이라 DB 커넥션 10개를 항상 점유
```

Apache DBCP2 공식 설정 기준으로 `maxTotal`은 동시에 할당 가능한 최대 커넥션 수이고, `maxWaitMillis`는 커넥션이 없을 때 반환 대기 후 예외를 던지는 최대 대기 시간입니다. 현재 설정은 최대 10개까지만 할당하고, 모두 사용 중이면 3초 후 실패할 수 있습니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))

## 2. 설정별 위험도

|설정|현재값|위험도|판단|
|---|--:|--:|---|
|`maxTotal`|`10`|높음|외부솔루션 호출이 순간적으로 몰리면 10개 초과 요청은 대기 후 실패 가능|
|`maxWaitMillis`|`3000`|중간~높음|풀 고갈 시 3초 만에 `Cannot get a connection` 계열 오류 가능|
|`testOnBorrow`|`false`|높음|DB/L4/방화벽이 끊은 커넥션을 검증 없이 전달 가능|
|`testWhileIdle`|`false`|높음|Idle 상태 커넥션 사전 검증 없음|
|`timeBetweenEvictionRunsMillis`|`480000`|낮음~중간|8분마다 Evictor는 돌 수 있으나, 현재 검증/제거 설정상 효과 제한적|
|`minEvictableIdleTimeMillis`|`-1`|중간|Idle 커넥션을 시간 기준으로 제거하지 않음|
|`minIdle=maxIdle=maxTotal`|`10`|중간|항상 10개 DB 세션을 유지하므로 DB 자원 고정 점유|
|`poolPreparedStatements`|`true`|중간|SQL 종류가 많으면 커서/Statement 캐시 자원 증가|
|`maxOpenPreparedStatements`|`30`|낮음~중간|커넥션당 30개, 전체 최대 약 300개 캐시 가능|

## 3. 가장 심각한 문제: 죽은 커넥션 검증 부재

현재는 `validationQuery=SELECT 1`이 설정되어 있지만, 실제 검증 트리거가 모두 꺼져 있습니다.

```xml
<property name="testOnBorrow" value="false"/>
<property name="testOnReturn" value="false"/>
<property name="testWhileIdle" value="false"/>
```

DBCP2 공식 문서상 `validationQuery`는 커넥션을 호출자에게 반환하기 전에 검증할 때 사용되는 SELECT 쿼리이며, `testOnBorrow=true`이면 빌리기 전에 검증하고 실패한 커넥션은 풀에서 제거됩니다. 현재는 `testOnBorrow=false`라서 이 보호 동작이 없습니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))

### 발생 가능한 장애

```text
DB wait_timeout, L4 idle timeout, 방화벽 세션 정리
→ DBCP 풀에는 커넥션 객체가 남아 있음
→ 애플리케이션이 해당 커넥션을 borrow
→ 검증 없이 SQL 실행
→ Connection reset, Broken pipe, Communications link failure, No operations allowed after connection closed 등 발생
```

### 심각도

```text
높음
```

특히 외부솔루션이 배치성/연계성 작업을 수행한다면, 첫 SQL 실행 시점에 실패하므로 장애 원인이 애플리케이션 로직처럼 보일 수 있습니다.

## 4. `maxTotal=10`의 병목 위험

현재 설정은 풀 생성 시 10개를 만들고, 최대도 10개입니다.

```xml
<property name="initialSize" value="10"/>
<property name="maxTotal" value="10"/>
<property name="maxIdle" value="10"/>
<property name="minIdle" value="10"/>
```

DBCP2 공식 문서상 `initialSize`는 시작 시 생성되는 커넥션 수, `maxTotal`은 동시에 할당 가능한 최대 커넥션 수, `minIdle`은 유지하려는 최소 idle 커넥션 수입니다. 또한 `minIdle`은 Evictor가 양수 주기로 실행될 때 의미가 있습니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))

### 발생 가능한 장애

```text
동시 요청 10개가 모두 DB 사용
→ 11번째 요청은 최대 3초 대기
→ 3초 내 반환 커넥션 없으면 예외
→ 외부솔루션 화면/배치/연계 API 실패
```

### 예상 오류

```text
java.sql.SQLException: Cannot get a connection, pool error Timeout waiting for idle object
java.util.NoSuchElementException: Timeout waiting for idle object
```

### 심각도

```text
중간~높음
```

외부솔루션의 실제 동시 처리량이 낮으면 문제 없을 수 있지만, 누락된 close, 느린 쿼리, 일시 트래픽 증가가 있으면 장애 전파가 빠릅니다.

## 5. Evictor 설정이 사실상 큰 보호를 못 함

현재 Evictor는 480초, 즉 8분 주기로 설정되어 있습니다.

```xml
<property name="timeBetweenEvictionRunsMillis" value="480000"/>
<property name="numTestsPerEvictionRun" value="4"/>
<property name="minEvictableIdleTimeMillis" value="-1"/>
<property name="testWhileIdle" value="false"/>
```

DBCP2 공식 문서상 `timeBetweenEvictionRunsMillis`는 idle object evictor thread 실행 주기이고, `testWhileIdle=true`일 때 idle 커넥션 검증이 수행됩니다. `minEvictableIdleTimeMillis`는 idle 커넥션이 제거 대상이 되기 위한 최소 idle 시간입니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))  
현재는 다음과 같이 해석하는 것이 맞습니다.

```text
Evictor thread는 돌 수 있음
하지만 testWhileIdle=false → idle validation 없음
minEvictableIdleTimeMillis=-1 → 시간 기준 idle 제거도 사실상 없음
따라서 죽은 커넥션 정리 효과가 거의 없음
```

## 6. PreparedStatement Pool 위험

현재 설정:

```xml
<property name="poolPreparedStatements" value="true"/>
<property name="maxOpenPreparedStatements" value="30"/>
```

DBCP2 공식 문서상 `poolPreparedStatements=true`이면 커넥션별 PreparedStatement 풀이 생성됩니다. 또한 PreparedStatement Pool은 DB cursor를 열어둘 수 있어, 커넥션 리소스가 부족해질 수 있으므로 `maxOpenPreparedStatements`를 DB의 커서 한계보다 낮게 설정해야 한다고 설명합니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))  
현재 최대치는 대략 다음과 같습니다.

```text
maxTotal 10개 × maxOpenPreparedStatements 30개 = 최대 300개 수준의 Statement 캐시 가능
```

### 심각도

```text
중간
```

SQL 종류가 적고 반복 호출이 많으면 성능상 이점이 있습니다. 반대로 동적 SQL이 많고 SQL 패턴이 다양하면 커서/메모리 점유가 증가할 수 있습니다.

## 7. 장애 시나리오별 심각도

|시나리오|발생 가능성|영향도|심각도|
|---|--:|--:|--:|
|DB/L4가 idle 커넥션 종료|높음|첫 SQL 실패|높음|
|동시 요청 10개 초과|중간|3초 후 커넥션 획득 실패|중간~높음|
|커넥션 누수|낮음~중간|maxTotal 10 고갈|높음|
|느린 쿼리 다수 발생|중간|풀 점유 장기화|높음|
|PreparedStatement 캐시 과다|낮음~중간|DB cursor/메모리 증가|중간|
|DB 세션 10개 상시 점유|높음|DB 자원 고정 사용|낮음~중간|

## 8. 권장 설정안

### 안정성 우선 권장안

```xml
<property name="initialSize" value="5"/>
<property name="maxTotal" value="20"/>
<property name="maxIdle" value="10"/>
<property name="minIdle" value="5"/>
<property name="maxWaitMillis" value="5000"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testOnReturn" value="false"/>
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="5"/>
<property name="minEvictableIdleTimeMillis" value="300000"/>
<property name="softMinEvictableIdleTimeMillis" value="180000"/>
<property name="poolPreparedStatements" value="true"/>
<property name="maxOpenPreparedStatements" value="30"/>
```

### 왜 이렇게 바꾸는가

```text
1. maxTotal 10 → 20
   - 순간 동시 요청 대응력 확보
2. minIdle 10 → 5
   - DB 세션 상시 점유 감소
3. testOnBorrow false → true
   - 죽은 커넥션을 애플리케이션에 넘기기 전 제거
4. testWhileIdle false → true
   - idle 상태에서 사전 검증
5. Evictor 480초 → 60초
   - 죽은 idle 커넥션 발견 지연 감소
6. minEvictableIdleTimeMillis -1 → 300000
   - 오래된 idle 커넥션 제거 가능
```

## 9. 부하가 매우 낮은 외부솔루션이면 보수적 수정안

외부솔루션 트래픽이 낮고 DB max_connections 여유가 작다면 `maxTotal`을 크게 늘리지 말고 검증만 켜는 방식이 안전합니다.

```xml
<property name="initialSize" value="5"/>
<property name="maxTotal" value="10"/>
<property name="maxIdle" value="10"/>
<property name="minIdle" value="5"/>
<property name="maxWaitMillis" value="5000"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testOnReturn" value="false"/>
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="5"/>
<property name="minEvictableIdleTimeMillis" value="300000"/>
<property name="poolPreparedStatements" value="true"/>
<property name="maxOpenPreparedStatements" value="30"/>
```

## 10. 운영 적용 전 확인 항목

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW VARIABLES LIKE 'wait_timeout';
SHOW VARIABLES LIKE 'interactive_timeout';
SHOW FULL PROCESSLIST;
```

확인 기준:

```text
1. 외부솔루션 전용 DB 계정의 평균/최대 Sleep 세션 수
2. Active 쿼리 수행 시간이 긴 SQL 존재 여부
3. wait_timeout이 300초~600초처럼 짧은지 여부
4. L4/방화벽 idle timeout 존재 여부
5. WAS 로그에 Timeout waiting for idle object 존재 여부
6. Connection reset by peer, Broken pipe, Communications link failure 존재 여부
```

## 11. 최종 판단

현재 설정의 가장 큰 문제는 **커넥션 수 10개 제한**보다 **검증 없는 커넥션 재사용**입니다.

```text
우선순위 1: testOnBorrow=true 적용
우선순위 2: testWhileIdle=true + Evictor 60초 적용
우선순위 3: minIdle 10 → 5로 완화
우선순위 4: 실제 동시 처리량에 따라 maxTotal 10 유지 또는 20 확장
우선순위 5: PreparedStatement Pool은 SQL 패턴 확인 후 유지/해제 판단
```

운영 장애 가능성을 등급으로 보면 다음과 같습니다.

```text
현재 설정 위험도: 중간~높음
즉시 장애 가능성: 낮음~중간
트래픽 증가/DB idle timeout/L4 timeout 존재 시 장애 가능성: 높음
운영 권장 조치: testOnBorrow와 testWhileIdle은 최소한 반드시 켜는 방향 권장
```

정확도: 94%

# 결론

`testWhileIdle=false`여도 **Evictor Thread 자체는 동작할 수 있습니다.**  
이유는 `testWhileIdle`은 **Evictor 실행 여부를 결정하는 옵션이 아니라**, Evictor가 실행될 때 **Idle 커넥션 유효성 검사를 할지 여부**를 결정하는 옵션이기 때문입니다.  
Evictor 실행 여부는 주로 아래 설정이 결정합니다.

```xml
<property name="timeBetweenEvictionRunsMillis" value="480000"/>
```

Apache DBCP2 공식 문서 기준으로 `timeBetweenEvictionRunsMillis`는 Idle Object Evictor Thread 실행 간격이며, 값이 `0 이하`이면 Evictor Thread가 실행되지 않습니다. 현재 값은 `480000`으로 양수이므로 Evictor가 실행됩니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html?utm_source=chatgpt.com "BasicDataSource Configuration – Apache Commons DBCP"))

# 설정별 역할

| 설정                              |      현재값 | 의미                                                |
| ------------------------------- | -------: | ------------------------------------------------- |
| `timeBetweenEvictionRunsMillis` | `480000` | Evictor Thread를 480초, 즉 8분 주기로 실행                 |
| `numTestsPerEvictionRun`        |      `4` | Evictor 1회 실행 시 최대 4개 Idle Object 검사              |
| `testWhileIdle`                 |  `false` | Evictor 실행 시 `validationQuery`로 커넥션 유효성 검사는 하지 않음 |
| `minEvictableIdleTimeMillis`    |     `-1` | 시간 기준 Idle 커넥션 제거 조건 비활성에 가까움                     |
| `minIdle`                       |     `10` | 최소 Idle 10개 유지 시도                                 |

# 핵심 오해

```text
testWhileIdle=false
= Evictor 미동작
```

이 해석은 맞지 않습니다.  
정확히는 다음입니다.

```text
testWhileIdle=false
= Evictor는 돌 수 있지만, validationQuery 기반 유효성 검사는 하지 않음
```

# 현재 설정에서 실제 동작

현재 설정:

```xml
<property name="testWhileIdle" value="false"/>
<property name="timeBetweenEvictionRunsMillis" value="480000"/>
<property name="numTestsPerEvictionRun" value="4"/>
<property name="minEvictableIdleTimeMillis" value="-1"/>
<property name="minIdle" value="10"/>
```

동작 흐름은 대략 다음과 같습니다.

```text
1. 480초마다 Evictor Thread 실행
2. Idle Pool에서 일부 커넥션 후보 확인
3. testWhileIdle=false이므로 SELECT 1 검증은 수행하지 않음
4. minEvictableIdleTimeMillis=-1이므로 일반적인 idle 시간 초과 제거도 사실상 수행하지 않음
5. minIdle=10 조건 때문에 idle 수 유지 로직은 고려될 수 있음
```

즉, 로그에 `[commons-pool-evictor]` 같은 Thread명이 보이는 것은 정상입니다. 다만 그 Thread가 반드시 `SELECT 1`을 실행했다는 뜻은 아닙니다.

# Evictor가 하는 일과 `testWhileIdle`의 차이

|구분|설명|
|---|---|
|Evictor 실행|Idle 객체를 주기적으로 순회/검사하는 백그라운드 작업|
|Eviction|Idle 시간이 오래된 객체를 제거할지 판단|
|Validation|커넥션이 살아 있는지 `SELECT 1` 또는 `Connection.isValid()`로 확인|
|`testWhileIdle`|Evictor 실행 중 Validation을 할지 결정|

DBCP2 공식 문서에서도 `testWhileIdle`은 Evictor가 Idle 커넥션을 검사할 때 validation을 수행할지 여부로 설명됩니다. 반면 Evictor Thread 자체는 `timeBetweenEvictionRunsMillis`가 양수일 때 실행됩니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html?utm_source=chatgpt.com "BasicDataSource Configuration – Apache Commons DBCP"))

# 그런데 왜 `PoolableConnection.validate` 로그가 보일 수 있는가?

`testWhileIdle=false`인데도 `PoolableConnection.validate` 또는 `SELECT 1` 로그가 보인다면, 아래 가능성을 봐야 합니다.

## 1. 실제 적용 설정이 다를 가능성

설정 파일에는 `false`지만, 실제 런타임 Bean에는 다르게 적용되었을 수 있습니다.

```text
- 다른 datasource 설정 파일이 사용됨
- 외부솔루션 내부 기본값이 덮어씀
- JNDI Resource 설정이 별도로 존재
- Spring profile 또는 WAS 설정이 우선 적용
- 운영 배포 파일과 확인한 설정 파일이 다름
```

확인 방법:

```java
BasicDataSource ds = ...;
System.out.println(ds.getTestWhileIdle());
System.out.println(ds.getTestOnBorrow());
System.out.println(ds.getTimeBetweenEvictionRunsMillis());
System.out.println(ds.getNumTestsPerEvictionRun());
```

## 2. `testOnBorrow=true`가 다른 경로에서 적용된 경우

현재 제시 설정은 `testOnBorrow=false`지만, 실제 런타임에서 `true`이면 커넥션을 빌릴 때 validate가 수행됩니다.

```text
요청 Thread
→ getConnection()
→ testOnBorrow=true
→ validate()
→ SELECT 1
```

이 경우 Evictor가 아니라 업무 Thread에서 검증합니다.

## 3. 다른 DataSource의 Evictor 로그일 가능성

WAS에 DataSource가 여러 개라면 `[commons-pool-evictor]` 로그가 해당 외부솔루션 DataSource의 로그가 아닐 수 있습니다.

```text
- 업무용 datasource
- 외부솔루션 datasource
- 배치 datasource
- 모니터링 datasource
```

로그에 DataSource 식별자가 없다면 오인 가능성이 큽니다.

## 4. `maxConnLifetimeMillis` 또는 abandoned 설정 영향

현재 제시 설정에는 없지만, 외부 설정에 아래 값이 있으면 Evictor/Pool 관리 로직에서 별도 정리 작업이 발생할 수 있습니다.

```xml
<property name="maxConnLifetimeMillis" value="..."/>
<property name="removeAbandonedOnMaintenance" value="true"/>
```

이 경우 `testWhileIdle`과 무관하게 Evictor 계열 Thread가 관여할 수 있습니다.

# 현재 설정에서 가장 중요한 판단

현재 설정은 Evictor Thread는 돌지만, 실질적인 보호 효과는 낮습니다.

```text
Evictor 실행됨: Yes
Idle 커넥션 SELECT 1 검증: No
Idle 시간 초과 제거: 거의 No
죽은 커넥션 사전 제거: 거의 No
```

따라서 DB/L4/방화벽이 idle connection을 끊으면, 풀 안에는 죽은 커넥션 객체가 남아 있을 수 있고, 이후 애플리케이션이 그 커넥션을 빌려서 첫 SQL 실행 시점에 실패할 수 있습니다.

# 운영 권장

장애 방어 목적이면 아래처럼 가는 것이 안전합니다.

```xml
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="5"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
```

그리고 더 확실한 방어가 필요하면:

```xml
<property name="testOnBorrow" value="true"/>
```

단, `testOnBorrow=true`는 커넥션을 빌릴 때마다 검증하므로 요청량이 많은 서비스에서는 DB에 `SELECT 1` 부하가 추가됩니다.

# 요약

```text
testWhileIdle=false여도 Evictor는 동작 가능하다.
Evictor 실행 여부는 timeBetweenEvictionRunsMillis > 0 여부가 결정한다.
testWhileIdle=false는 Evictor가 SELECT 1 검증을 하지 않는다는 의미다.
현재 설정에서는 Evictor 로그가 보여도 죽은 커넥션 방어 효과는 거의 없다.
운영 안정성을 원하면 testWhileIdle=true 또는 testOnBorrow=true 적용을 검토해야 한다.
```

정확도: 93%

## 결론

첨부 로그의 핵심은 **`SELECT 1`이 장애 원인이라기보다, 이미 끊긴 커넥션을 Evictor가 검증하다가 끊김을 발견한 것**입니다. 로그상 실행 주체는 업무 Thread가 아니라 `commons-pool-evictor`이고, `PoolableConnection.validate()` → `PreparedStatement.executeQuery()` → `SELECT 1` 수행 중 `Connection reset by peer (Write failed)`가 발생했습니다. 즉, DBCP2가 idle 커넥션 검증을 수행하는 순간, TCP 소켓이 이미 DB/L4/방화벽/네트워크 쪽에서 종료된 상태였다고 보는 것이 가장 타당합니다.

## 로그 해석

|구분|해석|
|---|---|
|Thread|`commons-pool-evictor` → DBCP2 Idle Evictor Thread|
|실행 SQL|`PreparedStatement.executeQuery() SELECT 1`|
|발생 위치|`PoolableConnection.validate()`|
|예외|`SQLNonTransientConnectionException: Connection reset by peer (Write failed)`|
|직접 원인|검증 쿼리를 쓰는 순간 TCP 연결이 reset됨|
|근본 원인 후보|DB `wait_timeout`, L4/방화벽 idle timeout, DB 재기동, 네트워크 세션 정리, 오래된 idle connection|
|Apache DBCP2 공식 문서 기준으로 `validationQuery`는 커넥션을 호출자에게 반환하기 전 유효성 검사용 SELECT이며, `testWhileIdle=true`이면 idle object evictor가 검증을 수행하고 실패한 객체는 pool에서 제거합니다. 또한 `timeBetweenEvictionRunsMillis`가 양수이면 Evictor Thread가 실행됩니다. ([Apache Commons](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))||

## 중요한 판단: DB 서버 로그가 아니라 WAS/JDBC 로그일 가능성이 높음

첨부 로그는 다음 형식입니다.

```text
INFO  [stdout] (commons-pool-evictor) ... ERROR [jdbc.audit] ... SELECT 1
```

따라서 이것은 MariaDB의 `mysqld error.log`라기보다 **JBoss/WAS의 stdout 또는 server.log에 출력된 log4jdbc audit 로그**로 보는 것이 맞습니다. log4jdbc는 JDBC 호출과 SQL을 로깅하는 프록시 드라이버이고, `jdbc.audit`는 ResultSet을 제외한 거의 모든 JDBC 호출을 기록하는 매우 많은 양의 로그입니다. SQL 예외가 발생하면 ERROR 레벨에서 stack trace를 출력합니다. ([GitHub](https://github.com/arthurblake/log4jdbc "GitHub - arthurblake/log4jdbc: log4jdbc is a Java JDBC driver that can log SQL and/or JDBC calls (and optionally SQL timing information) for other JDBC drivers using the Simple Logging Facade For Java (SLF4J) logging system. · GitHub"))

## 왜 `SELECT 1`에서 오류가 발생하는가

흐름은 다음과 같습니다.

```text
1. 커넥션이 idle 상태로 pool에 남아 있음
2. DB, L4, 방화벽, NAT, 네트워크 장비 중 하나가 idle TCP 세션을 먼저 종료
3. DBCP2 pool은 아직 해당 커넥션 객체를 보유
4. Evictor가 검증을 위해 SELECT 1 전송
5. 이미 닫힌 socket에 write 시도
6. Connection reset by peer (Write failed)
7. DBCP2는 해당 커넥션을 invalid로 보고 제거
```

MariaDB의 `wait_timeout`은 서버가 비활성 connection이 다시 active가 되기를 기다리는 시간이며, 시간이 지나면 서버가 연결을 닫을 수 있습니다. 따라서 DB 또는 중간 네트워크 장비의 idle timeout보다 DBCP2의 idle 커넥션 유지 시간이 길면 이런 현상이 반복될 수 있습니다. ([MariaDB](https://mariadb.com/docs/server/server-management/variables-and-modes/server-system-variables?utm_source=chatgpt.com "Server System Variables | Server | MariaDB Documentation"))

## 심각도

|항목|심각도|판단|
|---|--:|---|
|Evictor 로그 자체|낮음~중간|업무 SQL 실패가 아니라 idle 검증 실패라면 즉시 장애는 아닐 수 있음|
|반복 발생|중간|pool이 오래된 죽은 커넥션을 계속 보유하거나 timeout 정렬이 안 맞는 상태|
|업무 Thread에서 발생|높음|`default task-*`, `BizExecutor-*` 등에서 동일 오류가 나면 실제 요청 실패|
|pool 고갈 동반|높음|검증 실패/재생성 반복 + `maxTotal` 부족 시 서비스 영향 가능|

## 대응 원칙

핵심은 **검증 쿼리를 없애는 것**이 아니라, **죽은 커넥션이 검증 시점에 ERROR 로그로 폭발하지 않도록 timeout과 logging을 정렬**하는 것입니다.

## 대응책 1: Evictor 검증은 유지하되, DB/L4 timeout보다 먼저 정리

가장 권장하는 방식입니다. 업무 요청이 죽은 커넥션을 잡기 전에 Evictor가 미리 제거하도록 합니다.

```xml
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="5"/>
<property name="minEvictableIdleTimeMillis" value="240000"/>
<property name="softMinEvictableIdleTimeMillis" value="180000"/>
```

단, 이 설정은 DB/L4 idle timeout이 예를 들어 300초 수준일 때의 예시입니다. 실제 값은 반드시 아래를 확인한 뒤 맞춰야 합니다.

```sql
SHOW VARIABLES LIKE 'wait_timeout';
SHOW VARIABLES LIKE 'interactive_timeout';
```

권장 기준:

```text
DB/L4 idle timeout = 300초라면
DBCP idle 제거/검증 주기 = 60초
minEvictableIdleTimeMillis = 180~240초
```

`minEvictableIdleTimeMillis`는 Evictor가 idle 객체를 제거 대상으로 판단하기 위한 최소 idle 시간이고, `softMinEvictableIdleTimeMillis`는 `minIdle` 조건을 고려해 idle connection을 제거할 수 있게 하는 설정입니다. ([Apache Commons](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))

## 대응책 2: `validationQuery` 제거 후 JDBC `Connection.isValid()` 사용

`SELECT 1` 로그 자체를 줄이고 싶다면 다음 방식이 가능합니다.

```xml
<!-- validationQuery 제거 또는 미설정 -->
<!-- <property name="validationQuery" value="SELECT 1"/> -->
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testWhileIdle" value="true"/>
```

DBCP2 공식 문서상 `validationQuery`가 지정되지 않으면 connection 검증은 JDBC `Connection.isValid()` 호출로 수행됩니다. 따라서 log4jdbc의 `PreparedStatement.executeQuery() SELECT 1` 형태의 audit 로그는 줄어들 수 있습니다. ([Apache Commons](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))  
다만 주의할 점이 있습니다.

```text
1. isValid()도 내부적으로 driver ping 또는 네트워크 I/O를 수행할 수 있음
2. 이미 끊긴 TCP connection이면 예외나 validation 실패는 여전히 발생 가능
3. SELECT 1 로그는 줄어도 죽은 커넥션 문제 자체가 사라지는 것은 아님
4. MariaDB Connector/J 버전에 따라 내부 구현 차이가 있을 수 있음
```

즉, **로그 회피 목적에는 효과가 있을 수 있지만 근본 대응은 timeout 정렬**입니다.

## 대응책 3: log4jdbc `jdbc.audit` 로그 비활성화

현재 로그가 `jdbc.audit`에서 ERROR로 찍히는 구조이므로, 운영에서는 `jdbc.audit`를 끄는 것이 가장 직접적인 로그 소음 회피책입니다. log4jdbc 문서에서도 `jdbc.audit`는 거의 모든 JDBC 호출을 기록하므로 특정 JDBC 문제 추적 때가 아니면 일반적으로 필요하지 않으며, 로그가 매우 빠르게 커질 수 있다고 설명합니다. ([GitHub](https://github.com/arthurblake/log4jdbc "GitHub - arthurblake/log4jdbc: log4jdbc is a Java JDBC driver that can log SQL and/or JDBC calls (and optionally SQL timing information) for other JDBC drivers using the Simple Logging Facade For Java (SLF4J) logging system. · GitHub"))

### Log4j2 예시

```xml
<Logger name="jdbc.audit" level="off" additivity="false"/>
<Logger name="jdbc.resultset" level="off" additivity="false"/>
<Logger name="jdbc.resultsettable" level="off" additivity="false"/>
<Logger name="jdbc.connection" level="warn" additivity="false"/>
<Logger name="jdbc.sqlonly" level="info" additivity="false"/>
<Logger name="jdbc.sqltiming" level="warn" additivity="false"/>
```

운영 기준 권장:

```text
jdbc.audit = OFF
jdbc.resultset = OFF
jdbc.resultsettable = OFF
jdbc.connection = WARN 또는 OFF
jdbc.sqlonly = 필요한 경우만 INFO
jdbc.sqltiming = 느린 SQL 추적 시 WARN/INFO
```

이 조치는 **실제 connection reset을 해결하지는 않지만**, `SELECT 1` 검증 실패가 server.log를 오염시키는 문제는 줄일 수 있습니다.

## 대응책 4: 검증 실패 로그를 줄이려면 `testWhileIdle`만 끄는 방법은 비권장

아래처럼 하면 Evictor의 `SELECT 1` 오류 로그는 줄 수 있습니다.

```xml
<property name="testWhileIdle" value="false"/>
```

하지만 이 경우 죽은 idle connection이 pool 안에 남아 있다가 업무 요청이 `getConnection()`으로 가져갈 수 있습니다. 특히 `testOnBorrow=false`까지 같이 꺼져 있으면 업무 SQL에서 오류가 발생합니다.

```text
testWhileIdle=false
+ testOnBorrow=false
= 죽은 커넥션이 업무 Thread로 전달될 가능성 증가
```

따라서 로그만 없애려고 검증을 끄는 것은 운영 안정성 측면에서 좋지 않습니다.

## 대응책 5: `testOnBorrow=true`를 최소 안전장치로 적용

업무 장애 방어 목적이라면 최소한 아래는 권장합니다.

```xml
<property name="testOnBorrow" value="true"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
```

DBCP2 공식 문서상 `testOnBorrow=true`이면 pool에서 빌리기 전에 객체를 검증하고, 실패하면 해당 객체를 버린 뒤 다른 객체를 빌리려고 시도합니다. ([Apache Commons](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))  
단점:

```text
커넥션 borrow 시점마다 검증 비용 발생
요청량이 매우 많은 서비스에서는 SELECT 1 부하 증가
```

대안:

```text
testWhileIdle=true로 사전 정리
testOnBorrow=true는 안정성 우선 환경에서 적용
부하가 크면 testOnBorrow=true 적용 전후 SELECT 1 TPS 확인
```

## 대응책 6: MariaDB Connector/J 네트워크 옵션 검토

MariaDB Connector/J에는 socket timeout과 TCP keepalive 관련 옵션이 있습니다. 공식 문서상 `socketTimeout`은 네트워크 socket read timeout이고, `tcpKeepAlive`는 socket의 TCP keepalive 옵션을 설정합니다. Connector/J 3.0.0부터 `tcpKeepAlive` 기본값은 true지만, 현재 로그의 드라이버는 `mariadb-java-client-2.7.12`이므로 기본값/지원 옵션 차이를 확인해야 합니다. ([MariaDB](https://mariadb.com/docs/connectors/mariadb-connector-j/about-mariadb-connector-j "About MariaDB Connector/J Guide | Connectors | MariaDB Documentation"))  
예시:

```properties
jdbc:mariadb://host:3306/db?tcpKeepAlive=true&connectTimeout=3000&socketTimeout=30000
```

주의:

```text
tcpKeepAlive는 OS TCP keepalive 주기 영향을 받음
기본 OS keepalive 주기가 길면 L4 idle timeout 회피에 충분하지 않을 수 있음
socketTimeout은 장시간 read 대기 방지용이지 idle connection reset의 직접 해결책은 아님
```

## 추천 최종 설정안

DB/L4 idle timeout이 5~10분 수준이라고 가정한 안정성 우선 예시입니다.

```xml
<property name="initialSize" value="5"/>
<property name="maxTotal" value="20"/>
<property name="maxIdle" value="10"/>
<property name="minIdle" value="5"/>
<property name="maxWaitMillis" value="5000"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testOnReturn" value="false"/>
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="5"/>
<property name="minEvictableIdleTimeMillis" value="240000"/>
<property name="softMinEvictableIdleTimeMillis" value="180000"/>
<property name="poolPreparedStatements" value="true"/>
<property name="maxOpenPreparedStatements" value="30"/>
```

`SELECT 1` audit 로그 자체를 줄이는 방향이면 다음 변형을 검토합니다.

```xml
<!-- validationQuery를 제거해서 JDBC Connection.isValid() 사용 유도 -->
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
```

## 운영 점검 순서

```sql
SHOW VARIABLES LIKE 'wait_timeout';
SHOW VARIABLES LIKE 'interactive_timeout';
SHOW FULL PROCESSLIST;
```

```bash
grep -i "Connection reset by peer\|Broken pipe\|Communications link failure\|Timeout waiting for idle object" server.log
grep -i "commons-pool-evictor" server.log
grep -i "default task\|BizExecutor" server.log
```

판단 기준:

```text
commons-pool-evictor에서만 발생 → 사전 검증 중 죽은 커넥션 제거 로그일 가능성
업무 Thread에서도 발생 → 실제 요청 장애 가능성 높음
동일 conn 번호 반복 → pool/driver/log4jdbc wrapper 상태 추가 확인
특정 시간대 집중 → DB/L4 idle timeout, 방화벽 세션 정리, DB 백업/재기동 의심
```

## 최종 권고

우선순위는 다음입니다.

```text
1. 이 로그가 MariaDB error.log인지 WAS server.log인지 분리
2. log4jdbc jdbc.audit OFF 적용
3. DB wait_timeout/L4 idle timeout 확인
4. DBCP2 idle 제거 시간이 DB/L4 timeout보다 먼저 오도록 조정
5. testOnBorrow=true 적용으로 업무 Thread 보호
6. SELECT 1 로그 회피가 필요하면 validationQuery 제거 후 isValid() 방식 검토
```

가장 안전한 해석은 다음입니다.

```text
SELECT 1이 DB 장애를 만든 것이 아니라, 이미 죽은 connection을 발견하면서 로그가 발생한 것이다.
로그를 없애려면 jdbc.audit를 끄고, 근본적으로는 DB/L4 timeout보다 먼저 DBCP2가 connection을 검증·폐기하게 해야 한다.
```
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
