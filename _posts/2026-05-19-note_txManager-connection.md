---
layout: single
title: "txManager-connection"
excerpt: "txManager-connection"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-19"
last_modified_at: "2026-05-19 14:54:44 +0900"
---
## txManager - txManagerMail 분리
```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSource"/> </bean> 
  <!-- 
       <tx:advice id="txAdvice" transaction-manager="txManager"> 
         <tx:attributes> 
           <tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
         </tx:attributes> 
       </tx:advice> 
    --> 
<tx:advice id="txAdvice" transaction-manager="txManager">
  <tx:attributes> 
    <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception" />
    <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception" />
    <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception" />
    <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception" />
    <tx:method name="*" propagation="SUPPORTS" read-only="true" />
  </tx:attributes>
</tx:advice>

<aop:config>
  <aop:pointcut id="requiredTx" expression="execution(* bk.app..service.*Service.*(..))"/>
  <aop:advisor advice-ref="txAdvice" pointcut-ref="requiredTx" />
</aop:config>

<bean id="txManagerMail" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <property name="dataSource" ref="dataSourceMail"/>
</bean>
```


현재 XML 설정 기준으로는 **`txManagerMail`에는 선언적 트랜잭션이 기본적으로 적용되지 않습니다.**  
이유는 `txManagerMail` 빈은 선언되어 있지만, 실제 AOP 트랜잭션 Advice에 연결되어 있지 않기 때문입니다.  
현재 적용되는 트랜잭션 Advice는 아래 1개뿐입니다.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
```

그리고 이 Advice는 아래 Pointcut에만 적용됩니다.

```xml
<aop:pointcut id="requiredTx" expression="execution(* bk.app..service.*Service.*(..))"/>
<aop:advisor advice-ref="txAdvice" pointcut-ref="requiredTx" />
```

즉, 현재 AOP 트랜잭션은 **`txManager` → `dataSource`** 에 대해서만 동작합니다.  
`txManagerMail` → `dataSourceMail`은 **빈으로만 존재하고, AOP 트랜잭션 처리에는 참여하지 않습니다.**  
Spring 공식 문서에서도 `<tx:advice>`의 `transaction-manager` 속성은 해당 트랜잭션을 구동할 `PlatformTransactionManager` 빈 이름을 지정한다고 설명합니다. 또한 `<aop:advisor>`로 Pointcut과 Advice를 연결해야 해당 지점에서 트랜잭션 Advice가 실행됩니다. ([Home](https://docs.spring.io/spring-framework/docs/4.1.x/spring-framework-reference/html/transaction.html "12. Transaction Management"))

## 현재 설정의 적용 범위

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
```

위 설정으로 인해 트랜잭션 대상은 `dataSource`입니다.  
반면 아래 설정은 존재하지만 사용되지 않습니다.

```xml
<bean id="txManagerMail" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSourceMail"/>
</bean>
```

## 메서드명별 트랜잭션 동작

현재 `bk.app..service.*Service.*(..)`에 매칭되는 Service 메서드만 아래 규칙을 탑니다.

|메서드명|적용 트랜잭션|대상 DataSource|
|---|---|---|
|`save*`|`REQUIRED`, rollback-for `Exception`|`dataSource`|
|`insert*`|`REQUIRED`, rollback-for `Exception`|`dataSource`|
|`update*`|`REQUIRED`, rollback-for `Exception`|`dataSource`|
|`delete*`|`REQUIRED`, rollback-for `Exception`|`dataSource`|
|그 외 `*`|`SUPPORTS`, read-only|`dataSource` 기준|
|`dataSourceMail` 사용 DAO|별도 설정 없으면 트랜잭션 미적용|`dataSourceMail`|
|`SUPPORTS`는 현재 트랜잭션이 있으면 참여하고, 없으면 비트랜잭션으로 실행됩니다. Spring 공식 Javadoc에서도 `PROPAGATION_SUPPORTS`는 현재 트랜잭션을 지원하되, 트랜잭션이 없으면 non-transactional로 실행된다고 설명합니다. 다만 transaction synchronization이 있는 경우 단순 “트랜잭션 없음”과 완전히 같지는 않을 수 있고, 같은 JDBC Connection 같은 리소스가 해당 scope 동안 공유될 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/TransactionDefinition.html?utm_source=chatgpt.com "TransactionDefinition (Spring Framework 7.0.7 API)"))|||

## `txManagerMail`에 트랜잭션이 적용되지 않는다는 의미

### 적용되지 않는 것

`dataSourceMail`에 대해서는 기본적으로 아래가 적용되지 않습니다.

```text
1. 트랜잭션 시작
2. autoCommit=false 전환
3. commit/rollback 제어
4. rollback-for="Exception" 규칙
5. read-only 트랜잭션 설정
6. timeout/isolation 설정
```

Spring `DataSourceTransactionManager`는 단일 JDBC `DataSource`에 대한 `PlatformTransactionManager`이며, 지정된 `DataSource`의 JDBC Connection을 현재 Thread에 bind해서 트랜잭션을 관리합니다. 즉, `txManager`는 `dataSource`를 관리하고, `txManagerMail`은 `dataSourceMail`을 관리하도록 선언되어 있지만, 실제 Advice가 `txManagerMail`을 사용하지 않으면 `dataSourceMail`에 대한 트랜잭션 경계는 생기지 않습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html "DataSourceTransactionManager (Spring Framework 7.0.7 API)"))

## Connection 사용 영향

## 1. `dataSource`는 트랜잭션 관리 대상

예를 들어 아래 Service 메서드가 있다고 가정합니다.

```java
public void saveOrder() {
    orderDao.insertOrder();       // dataSource 사용
    mailDao.insertMailQueue();    // dataSourceMail 사용
}
```

`saveOrder()`가 `save*`에 해당하고 Pointcut에 매칭되면:

```text
saveOrder()
→ txManager가 dataSource Connection을 Thread에 bind
→ dataSource 작업은 하나의 트랜잭션으로 묶임
→ 정상 종료 시 commit
→ Exception 발생 시 rollback
```

## 2. `dataSourceMail`은 별도 트랜잭션 없이 동작

같은 메서드 안에서 `mailDao.insertMailQueue()`가 `dataSourceMail`을 사용하더라도, 현재 설정만으로는 `txManagerMail`이 개입하지 않습니다.  
결과:

```text
mailDao.insertMailQueue()
→ dataSourceMail에서 Connection 획득
→ 별도 트랜잭션 시작 없음
→ 일반적으로 autoCommit=true 상태에서 SQL 단위 commit
→ Service 전체 rollback이 발생해도 mail DB 작업은 이미 commit될 수 있음
```

즉, `dataSource` 작업은 rollback되는데 `dataSourceMail` 작업은 commit되어 **데이터 불일치**가 생길 수 있습니다.

## 3. 여러 SQL이 하나의 mail 트랜잭션으로 묶이지 않음

`dataSourceMail`로 아래처럼 여러 작업을 수행한다고 가정합니다.

```java
mailDao.insertMailMaster();
mailDao.insertMailDetail();
mailDao.updateMailStatus();
```

`txManagerMail`이 적용되지 않으면 일반적으로 각 `JdbcTemplate` 호출마다 Connection을 빌리고 반납합니다.

```text
insertMailMaster()
→ Connection borrow
→ SQL 실행
→ autoCommit commit
→ Connection return
insertMailDetail()
→ Connection borrow
→ SQL 실행
→ autoCommit commit
→ Connection return
updateMailStatus()
→ Connection borrow
→ SQL 실행
→ autoCommit commit
→ Connection return
```

중간에 예외가 나면 앞에서 실행된 SQL은 이미 commit되었을 수 있습니다.

```text
insertMailMaster() 성공 commit
insertMailDetail() 성공 commit
updateMailStatus() 실패
→ 앞의 2건 rollback 불가
```

## 4. Connection Pool 관점 영향

트랜잭션이 없으면 보통 Connection 보유 시간이 짧아지는 장점은 있습니다.

```text
SQL 실행 시점에 borrow
SQL 종료 후 return
```

하지만 단점도 있습니다.

|구분|영향|
|---|---|
|Connection 재사용|하나의 업무 메서드 안에서도 여러 Connection을 사용할 수 있음|
|일관성|여러 SQL을 하나의 원자적 작업으로 보장하지 못함|
|rollback|Service 실패 시 mail DB 작업 rollback 불가|
|autoCommit|SQL 단위로 즉시 commit 가능성 높음|
|Pool 부하|짧은 borrow/return이 반복될 수 있음|
|장애 분석|메인 DB rollback, mail DB commit 같은 불일치 발생 가능|

## 5. 중요한 예외: 기존 Spring transaction synchronization이 있는 경우

주의할 점이 하나 있습니다.  
`save*`, `insert*`, `update*`, `delete*` 메서드에서 `txManager`가 이미 트랜잭션을 시작하면 Spring의 transaction synchronization이 활성화됩니다. 이때 `JdbcTemplate` 또는 `DataSourceUtils`를 통해 `dataSourceMail` Connection을 얻으면, Spring이 해당 Connection을 현재 Thread에 묶어 **같은 scope 동안 재사용**할 수 있습니다.  
Spring 공식 문서는 `DataSourceUtils.getConnection()`이 현재 Thread에 bind된 Connection을 인식하고, transaction synchronization이 활성화되어 있으면 Connection을 Thread에 bind할 수 있다고 설명합니다. 또한 기존 트랜잭션에 synchronized된 Connection이 있으면 그 인스턴스를 반환하고, 없으면 새 Connection을 만들고 기존 트랜잭션에 선택적으로 synchronize하여 같은 transaction 안에서 재사용할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/4.0.0.M1_to_4.2.0.M2/Spring%20Framework%204.2.0.M2/org/springframework/jdbc/datasource/DataSourceUtils.html "DataSourceUtils"))  
다만 이 말은 **`dataSourceMail`이 `txManagerMail`로 commit/rollback 관리된다는 뜻은 아닙니다.**  
정리하면:

```text
Spring synchronization 때문에 dataSourceMail Connection이 Thread에 묶일 수는 있음
하지만 txManagerMail이 시작한 트랜잭션은 아님
따라서 mail DB SQL은 autoCommit 기준으로 개별 commit될 가능성이 높음
```

## 장애 시나리오

### 시나리오 1. 메인 DB rollback, Mail DB commit

```java
public void saveOrder() {
    orderDao.insertOrder();       // dataSource, txManager 관리
    mailDao.insertMailQueue();    // dataSourceMail, 트랜잭션 미관리
    throw new RuntimeException();
}
```

결과:

```text
orderDao.insertOrder()
→ txManager rollback

mailDao.insertMailQueue()
→ 이미 autoCommit으로 commit되었을 가능성

최종 결과:
주문 데이터 없음
메일 발송 큐 데이터 있음
```

이 구조는 운영에서 흔히 문제가 됩니다.

## 시나리오 2. Mail 관련 다건 insert 중 일부만 commit

```java
public void insertMailData() {
    mailDao.insertHeader();
    mailDao.insertReceiver();
    mailDao.insertBody();
}
```

`insertMailData()`가 Pointcut에 걸리더라도 `txManager`는 `dataSource`만 관리합니다. `mailDao`가 `dataSourceMail`을 쓰면 `txManagerMail`이 없으므로 다건 SQL의 원자성이 보장되지 않습니다.

```text
Header insert commit
Receiver insert commit
Body insert 실패
→ Header/Receiver만 남음
```

## 현재 설정에서 `txManagerMail`을 적용하려면

## 방법 1. Mail 전용 Advice 추가

Mail Service가 별도 패키지나 클래스명으로 분리되어 있다면 가장 명확합니다.

```xml
<tx:advice id="txAdviceMail" transaction-manager="txManagerMail">
    <tx:attributes>
        <tx:method name="save*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="insert*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="update*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="delete*" propagation="REQUIRED" rollback-for="Exception"/>
        <tx:method name="*" propagation="SUPPORTS" read-only="true"/>
    </tx:attributes>
</tx:advice>
<aop:config>
    <aop:pointcut id="requiredTx"
                  expression="execution(* bk.app..service.*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="requiredTx"/>
    <aop:pointcut id="requiredTxMail"
                  expression="execution(* bk.app..mail..service.*Service.*(..))"/>
    <aop:advisor advice-ref="txAdviceMail" pointcut-ref="requiredTxMail"/>
</aop:config>
```

Spring 문서에서도 서로 다른 Bean에 다른 트랜잭션 설정을 적용하려면 서로 다른 `<aop:advisor>`를 정의하고, 다른 pointcut과 advice-ref를 연결하는 방식을 제시합니다. ([Home](https://docs.spring.io/spring-framework/docs/4.1.x/spring-framework-reference/html/transaction.html "12. Transaction Management"))

## 방법 2. Annotation 방식 사용

XML AOP 방식 대신 annotation을 쓸 경우:

```xml
<tx:annotation-driven transaction-manager="txManager"/>
```

그리고 mail 서비스에:

```java
@Transactional("txManagerMail")
public void insertMailQueue() {
    ...
}
```

Spring 공식 문서는 여러 TransactionManager가 있는 경우 `@Transactional`의 value/qualifier로 사용할 `PlatformTransactionManager`를 지정할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/4.1.x/spring-framework-reference/html/transaction.html "12. Transaction Management"))  
다만 현재 XML에는 `<tx:annotation-driven>`이 없습니다. 따라서 `@Transactional("txManagerMail")`을 붙여도 annotation-driven 설정이 없으면 적용되지 않습니다.

## 방법 3. 메인 DB와 Mail DB를 하나의 트랜잭션으로 묶어야 한다면

`txManager`와 `txManagerMail`은 각각 **로컬 트랜잭션 매니저**입니다. Spring 공식 문서도 로컬 트랜잭션은 특정 리소스에 한정되며, 여러 transactional resource를 함께 처리하는 데 한계가 있다고 설명합니다. 여러 DB를 하나의 원자적 트랜잭션으로 묶으려면 일반적으로 JTA 같은 글로벌 트랜잭션을 검토해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/4.1.x/spring-framework-reference/html/transaction.html "12. Transaction Management"))  
하지만 실무에서는 메일/알림/로그성 데이터는 메인 업무 트랜잭션과 강하게 묶기보다 아래 방식이 더 많이 사용됩니다.

```text
1. 메인 DB 트랜잭션 commit
2. Outbox 테이블에 발송 요청 저장
3. 별도 배치/큐/스케줄러가 메일 발송
4. 실패 시 재처리
```

메일 큐 DB가 별도라면 JTA보다 Outbox/재처리 구조가 운영 안정성이 더 좋은 경우가 많습니다.

## 현재 설정에 대한 최종 판단

```text
txManagerMail은 현재 선언만 되어 있고 AOP 트랜잭션에는 연결되어 있지 않다.
따라서 dataSourceMail에 대한 선언적 트랜잭션은 기본적으로 적용되지 않는다.
```

Connection 관점에서는 아래처럼 봐야 합니다.

```text
1. dataSource는 txManager가 관리한다.
2. dataSourceMail은 txManagerMail이 관리하지 않는다.
3. dataSourceMail SQL은 일반적으로 autoCommit 단위로 commit된다.
4. 메인 트랜잭션 rollback과 mail DB rollback은 연동되지 않는다.
5. Spring transaction synchronization이 활성화된 경우 dataSourceMail Connection이 scope 동안 Thread에 bind될 수는 있으나, 이것이 txManagerMail 트랜잭션을 의미하지는 않는다.
6. mail DB에 원자성이 필요하면 txAdviceMail 또는 @Transactional("txManagerMail") 구성이 필요하다.
```

운영 기준으로는 **메일/알림성 데이터가 메인 업무 데이터와 반드시 함께 rollback되어야 하는지**부터 결정해야 합니다. 함께 rollback되어야 한다면 `txManagerMail` 적용 또는 JTA 검토가 필요하고, 함께 rollback될 필요가 없다면 Outbox/비동기 재처리 구조가 더 안정적입니다.


## SQLNonTransientConnectionException: Software caused connection abort

`SQLNonTransientConnectionException: Software caused connection abort`는 **SQL 문법/트랜잭션 오류가 아니라 DB와 연결된 TCP Socket이 비정상 종료된 상태에서 JDBC Driver가 DB 통신을 시도했을 때 발생하는 Connection 계층 오류**입니다. `SQLNonTransientConnectionException`은 JDBC에서 SQLState class `08` 계열의 연결 오류에 사용되며, 원인을 고치지 않으면 같은 작업을 재시도해도 성공하지 않는 연결 실패를 의미합니다. ([Oracle Docs](https://docs.oracle.com/en/java/javase/17/docs/api/java.sql/java/sql/SQLNonTransientConnectionException.html?utm_source=chatgpt.com "java.sql.SQLNonTransientConnectionException"))

## 의미 분해

```text
SQLNonTransientConnectionException
= DB Connection 자체가 정상 상태가 아님

Software caused connection abort
= OS/JVM Socket 계층에서 연결이 abort됨

Write failed / recv failed
= 이미 끊겼거나 끊기는 중인 Socket에 쓰기/읽기 시도
```

즉, 가장 흔한 해석은 다음입니다.

```text
DB Connection 객체는 애플리케이션/Pool에 남아 있었지만,
실제 TCP 연결은 DB 서버, L4, 방화벽, NAT, OS, 네트워크 장비 중 하나에 의해 이미 끊겼다.
그 상태에서 업무 SQL, SELECT 1, commit, rollback 중 통신을 시도하다가 오류가 발생했다.
```

## 주요 발생 원인

|구분|원인|설명|
|---|---|---|
|1|Idle Timeout|MariaDB `wait_timeout`, L4/방화벽 idle timeout으로 미사용 connection 종료|
|2|죽은 Connection 재사용|DBCP2 Pool에 남아 있던 stale connection을 업무 Thread나 Evictor가 사용|
|3|DB 재기동|MariaDB 재시작, 장애조치, VIP 전환, 세션 강제 종료|
|4|네트워크 장비 Reset|L4, 방화벽, NAT 장비가 세션을 강제로 정리|
|5|Socket Timeout/Abort|OS 또는 JVM socket 계층에서 read/write 중 연결 abort|
|6|Connection 검증 실패|`SELECT 1` 또는 `Connection.isValid()` 중 이미 끊긴 socket 확인|
|7|Pool 설정 부적절|`testOnBorrow=false`, `testWhileIdle=false`, idle 유지 시간이 DB/L4 timeout보다 김|
|8|DB 접속 한계|`max_connections` 초과, DB 부하, 서버 응답 지연|

MariaDB Connector/J는 Java 애플리케이션이 표준 JDBC API로 MariaDB/MySQL에 연결하기 위한 드라이버이므로, 이 오류는 MariaDB 서버가 그대로 던진 오류라기보다 **드라이버가 네트워크/연결 이상을 JDBC 예외로 변환한 결과**로 보는 것이 맞습니다. ([MariaDB](https://mariadb.com/docs/connectors/mariadb-connector-j/about-mariadb-connector-j?utm_source=chatgpt.com "About MariaDB Connector/J Guide"))

## DBCP2 환경에서 가장 흔한 흐름

```text
1. 애플리케이션이 DB Connection 사용 후 Pool에 반환
2. Connection이 idle 상태로 유지됨
3. DB wait_timeout 또는 L4/방화벽 idle timeout 도달
4. 실제 TCP 세션이 종료됨
5. DBCP2 Pool은 아직 Connection 객체를 보유
6. 이후 업무 Thread 또는 commons-pool-evictor가 해당 Connection 사용
7. SELECT 1 / 업무 SQL / commit / rollback 수행
8. Software caused connection abort 발생
```

DBCP2의 `validationQuery`는 Pool에서 Connection을 호출자에게 반환하기 전 검증하는 SQL이고, 지정하지 않으면 JDBC `Connection.isValid()`로 검증할 수 있습니다. 또한 `testOnBorrow`, `testWhileIdle` 같은 옵션이 검증 수행 시점을 결정합니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html?utm_source=chatgpt.com "BasicDataSource Configuration – Apache Commons DBCP"))

## 로그 위치별 판단

|발생 위치|해석|심각도|
|---|---|--:|
|`commons-pool-evictor`|Idle Connection 검증 중 죽은 Connection 발견|중간|
|`default task-*`|사용자 요청 처리 중 죽은 Connection 사용|높음|
|`BizExecutor-*`|배치/비동기 업무 중 연결 단절|높음|
|`commit()` / `rollback()`|트랜잭션 종료 시점에 연결 단절|높음|
|`SELECT 1`|Connection validation 중 이미 끊긴 socket 확인|중간|

## `Connection reset by peer`와의 차이

둘은 원인이 매우 유사합니다.

```text
Connection reset by peer
= 상대편이 TCP 연결을 reset

Software caused connection abort
= 로컬 OS/JVM socket 계층에서 연결 abort 감지
```

운영 분석에서는 둘 다 **DB Connection이 정상 유지되지 못한 네트워크/세션 단절 문제**로 묶어 봐야 합니다.

## 가장 의심해야 할 설정 조합

아래 조합이면 해당 오류가 업무 Thread에서 발생할 가능성이 커집니다.

```xml
<property name="testOnBorrow" value="false"/>
<property name="testWhileIdle" value="false"/>
<property name="minEvictableIdleTimeMillis" value="-1"/>
<property name="timeBetweenEvictionRunsMillis" value="480000"/>
```

위 설정에서는 Pool 내부에 오래된 idle connection이 남을 수 있고, DB/L4가 먼저 세션을 끊으면 애플리케이션이 stale connection을 잡을 수 있습니다.

## 대응책

### 1. 업무 Thread 보호: `testOnBorrow=true`

```xml
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
```

효과:

```text
Connection을 업무 로직에 넘기기 직전에 검증
죽은 Connection이면 폐기 후 다른 Connection 확보 시도
```

주의:

```text
borrow마다 SELECT 1 부하 발생 가능
요청량이 많으면 DB에 validation query TPS가 증가
```

### 2. Idle 단계에서 사전 정리: `testWhileIdle=true`

```xml
<property name="testWhileIdle" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="5"/>
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
```

효과:

```text
업무 요청 전에 Evictor가 idle connection을 미리 검증/폐기
```

### 3. DB/L4 timeout보다 Pool 제거 시간을 짧게 설정

DB 또는 L4 idle timeout이 300초라면 Pool은 그보다 먼저 정리해야 합니다.

```xml
<property name="minEvictableIdleTimeMillis" value="240000"/>
<property name="softMinEvictableIdleTimeMillis" value="180000"/>
```

기준:

```text
DB/L4 idle timeout > DBCP idle 제거/검증 시간
```

### 4. `SELECT 1` 로그가 부담이면 `Connection.isValid()` 검토

`validationQuery`를 제거하면 DBCP2는 JDBC `Connection.isValid()` 기반 검증을 사용할 수 있습니다. ([commons.apache.org](https://commons.apache.org/dbcp/configuration.html?utm_source=chatgpt.com "BasicDataSource Configuration – Apache Commons DBCP"))

```xml
<!-- validationQuery 미설정 -->
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testWhileIdle" value="true"/>
```

주의:

```text
SELECT 1 로그는 줄 수 있으나 네트워크 검증 자체가 사라지는 것은 아님
이미 끊긴 socket이면 validation 실패는 여전히 발생 가능
```

### 5. JDBC URL timeout 명시

MariaDB Connector/J 사용 시 연결/소켓 timeout을 명시해 장시간 대기를 줄입니다.

```properties
jdbc:mariadb://192.168.100.130:3306/DB_BK?connectTimeout=3000&socketTimeout=30000&tcpKeepAlive=true
```

단, `tcpKeepAlive`는 OS keepalive 주기의 영향을 받으므로 L4 idle timeout이 짧으면 단독 해결책이 아닙니다.

## 확인 쿼리

```sql
SHOW VARIABLES LIKE 'wait_timeout';
SHOW VARIABLES LIKE 'interactive_timeout';
SHOW VARIABLES LIKE 'max_connections';
SHOW FULL PROCESSLIST;
```

## 로그 확인 기준

```bash
grep -i "Software caused connection abort\|Connection reset by peer\|Broken pipe\|Communications link failure" server.log
grep -i "commons-pool-evictor\|default task\|BizExecutor" server.log
grep -i "Timeout waiting for idle object" server.log
```

판단:

```text
commons-pool-evictor에서만 발생
→ Evictor가 죽은 idle connection을 발견한 상황일 가능성

default task 또는 BizExecutor에서 발생
→ 실제 업무 요청/배치가 죽은 connection을 사용한 상황일 가능성

동일 시간대 DB 재시작/네트워크 장비 로그 존재
→ DB/L4/방화벽 세션 종료 가능성

DB wait_timeout보다 DBCP idle 유지 시간이 길다
→ stale connection 재사용 가능성 높음
```

## 권장 설정 예시

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
```

## 최종 요약

```text
SQLNonTransientConnectionException: Software caused connection abort는 DB Connection의 TCP 세션이 비정상 종료된 상태에서 JDBC 통신을 시도했을 때 발생한다.
MariaDB + DBCP2 환경에서는 DB/L4 idle timeout, stale connection 재사용, DB 재기동, 네트워크 reset이 가장 흔한 원인이다.
업무 Thread에서 발생하면 실제 장애 가능성이 높고, Evictor Thread에서만 발생하면 죽은 idle connection을 사전 발견한 로그일 수 있다.
대응은 testOnBorrow=true, testWhileIdle=true, timeout 정렬, JDBC timeout 명시 순서로 진행하는 것이 안전하다.
```


## isValid 사용 방법

## 핵심 결론

DBCP2에서 `Connection.isValid()` 방식으로 바뀌었는지 확인하려면 **`validationQuery`가 null/미설정인지**, **`SELECT 1` 로그가 사라지는지**, **evictor/borrow 검증 실패 시 끊긴 커넥션이 폐기되는지**를 같이 봐야 합니다.  
DBCP2 공식 문서 기준으로 `validationQuery`가 지정되지 않으면 connection 검증 시 `Connection.isValid()`가 사용됩니다. `testWhileIdle=true`이면 idle evictor가 검증을 수행하고, 실패한 객체는 pool에서 drop됩니다. ([Apache Commons](https://commons.apache.org/dbcp/configuration.html "BasicDataSource Configuration – Apache Commons DBCP"))

## 1. 설정 변경 방법

기존 설정:

```xml
<property name="validationQuery" value="SELECT 1"/>
<property name="validationQueryTimeout" value="3"/>
<property name="testWhileIdle" value="true"/>
<property name="testOnBorrow" value="false"/>
```

변경 후:

```xml
<!-- validationQuery 제거 -->
<property name="validationQueryTimeout" value="3"/>
<property name="testWhileIdle" value="true"/>
<property name="testOnBorrow" value="false"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="20"/>
```

가장 중요한 점은 `validationQuery`를 **빈 문자열로 애매하게 남기지 말고 property 자체를 제거**하는 것입니다. DBCP2 `BasicDataSource` 구현상 `validationQuery`가 지정되지 않으면 `Connection.isValid(int)`가 사용됩니다. 단, `validationQuery` 같은 설정은 pool 초기화 이후 변경해도 효과가 없으므로 서버 재기동 후 확인해야 합니다. ([Apache Git Repositories](https://apache.googlesource.com/commons-dbcp/%2B/6d45f3c5f1f55c7368a0b31c3ae243dfb65a386b/src/main/java/org/apache/commons/dbcp2/BasicDataSource.java "src/main/java/org/apache/commons/dbcp2/BasicDataSource.java - commons-dbcp - Git at Google"))

## 2. 정상 적용 여부 확인 포인트

|확인|정상 기준|
|---|---|
|DBCP 설정|`validationQuery=null` 또는 미설정|
|SQL 로그|`SELECT 1`, `SELECT 2` 같은 validation SQL이 더 이상 보이지 않음|
|Evictor 로그|`commons-pool-evictor` thread는 존재하지만 validation SQL 로그는 안 보일 수 있음|
|MariaDB processlist|주기적 `SELECT 1` thread가 보이지 않을 수 있음|
|장애 주입|끊긴 connection이 borrow/idle validation 시 폐기되고 새 connection으로 대체됨|
|업무 영향|사용자 요청에서 `Connection reset by peer` 빈도가 감소해야 함|

## 3. 왜 SQL 로그로만 확인하면 안 되는가

`Connection.isValid()`는 JDBC 표준 메소드입니다. Java 공식 문서 기준으로 driver는 이 메소드 호출 시 query를 제출하거나 다른 메커니즘을 사용해 connection 유효성을 검증할 수 있습니다. timeout은 초 단위이며, `0`이면 timeout 미적용입니다. ([Oracle Documentation](https://docs.oracle.com/javase/8/docs/api/java/sql/Connection.html "Connection (Java Platform SE 8 )"))  
MariaDB Connector/J는 `Connection.isValid()`가 MySQL protocol ping을 수행한다고 설명합니다. 즉 일반 SQL `SELECT 1`이 아니라 **프로토콜 ping**이므로 SQL 로그, log4jdbc, MyBatis 로그에 안 찍히는 것이 정상일 수 있습니다. ([MariaDB](https://mariadb.com/docs/connectors/mariadb-connector-j/about-mariadb-connector-j "About MariaDB Connector/J Guide | Connectors | MariaDB Documentation"))

## 4. 애플리케이션에서 설정값 직접 확인

서버 기동 후 `BasicDataSource` 값을 로그로 출력합니다.

```java
import org.apache.commons.dbcp2.BasicDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import javax.annotation.PostConstruct;
public class DataSourceCheckBean {
    private static final Logger log = LoggerFactory.getLogger(DataSourceCheckBean.class);
    private BasicDataSource dataSource;
    public void setDataSource(BasicDataSource dataSource) {
        this.dataSource = dataSource;
    }
    @PostConstruct
    public void checkDbcpValidationConfig() {
        log.info("DBCP validationQuery={}", dataSource.getValidationQuery());
        log.info("DBCP validationQueryTimeout={}", dataSource.getValidationQueryTimeout());
        log.info("DBCP testWhileIdle={}", dataSource.getTestWhileIdle());
        log.info("DBCP testOnBorrow={}", dataSource.getTestOnBorrow());
        log.info("DBCP timeBetweenEvictionRunsMillis={}", dataSource.getTimeBetweenEvictionRunsMillis());
        log.info("DBCP numTestsPerEvictionRun={}", dataSource.getNumTestsPerEvictionRun());
    }
}
```

Spring XML 예시:

```xml
<bean id="dataSourceCheckBean" class="com.example.DataSourceCheckBean">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

정상 기대값:

```text
DBCP validationQuery=null
DBCP validationQueryTimeout=3
DBCP testWhileIdle=true
DBCP testOnBorrow=false
```

`validationQuery=SELECT 1`이 찍히면 아직 `isValid()` 방식으로 바뀐 것이 아닙니다.

## 5. JMX로 확인

DBCP2 `BasicDataSource`는 JMX 속성으로 pool 상태를 볼 수 있습니다. 운영에서 확인할 값은 아래입니다.

|JMX 항목|확인 의미|
|---|---|
|`NumActive`|현재 사용 중 connection 수|
|`NumIdle`|idle connection 수|
|`ValidationQuery`|null이어야 `isValid()` 사용 기대|
|`TestWhileIdle`|true인지 확인|
|`TestOnBorrow`|요청 시 검증 여부 확인|
|`TimeBetweenEvictionRunsMillis`|evictor 주기 확인|
|`NumTestsPerEvictionRun`|1회 검사 수 확인|
|JConsole, VisualVM, Java Mission Control 또는 WAS JMX 콘솔에서 확인합니다.||

## 6. 로그로 확인하는 방법

### 변경 전

```text
[commons-pool-evictor] SELECT 1
[default task-15] SELECT 1
```

### 변경 후 정상

```text
[commons-pool-evictor] SELECT 1 로그 없음
[default task-15] SELECT 1 로그 없음
```

단, 이것만으로 evictor가 안 돈다고 판단하면 안 됩니다. `isValid()`가 프로토콜 ping을 쓰면 SQL 로그에 안 보입니다.  
추가로 pool 관련 로그를 일시적으로 DEBUG로 올립니다.

```xml
<logger name="org.apache.commons.dbcp2" level="DEBUG"/>
<logger name="org.apache.commons.pool2" level="DEBUG"/>
```

운영에서는 DEBUG를 장시간 유지하지 말고, 검증 시간 동안만 적용하는 것이 좋습니다.

## 7. 장애 주입 테스트

가장 확실한 확인은 **끊긴 idle connection을 DBCP2가 폐기하는지** 보는 것입니다.

### 테스트 절차

```text
1. 서버 기동
2. DB 요청을 여러 번 발생시켜 pool에 idle connection 생성
3. MariaDB에서 SHOW FULL PROCESSLIST로 connection id 확인
4. 특정 idle connection을 KILL
5. timeBetweenEvictionRunsMillis + 여유 시간 대기
6. 업무 요청 또는 borrow 발생
7. Connection reset by peer가 업무 SQL까지 노출되는지 확인
8. NumIdle/NumActive, 신규 connection 생성 여부 확인
```

MariaDB 확인:

```sql
SHOW FULL PROCESSLIST;
```

특정 connection 종료:

```sql
KILL 12345;
```

정상 기대:

```text
끊긴 connection이 validation 과정에서 invalid 처리됨
해당 connection이 pool에서 drop됨
필요 시 새 connection 생성됨
업무 SQL에는 끊긴 connection이 전달되지 않거나 빈도가 감소함
```

`testOnBorrow=false`이고 `testWhileIdle=true`만 켠 경우, evictor가 해당 idle connection을 검사하기 전에는 다음 업무 요청에서 끊긴 connection을 만날 수 있습니다. 이 경우 `numTestsPerEvictionRun`을 늘리거나 임시로 `testOnBorrow=true`를 켜서 확인합니다.

## 8. `testOnBorrow=true`로 단기 확인

검증 효과를 빠르게 확인하려면 임시로 아래처럼 설정합니다.

```xml
<property name="validationQueryTimeout" value="3"/>
<property name="testOnBorrow" value="true"/>
<property name="testWhileIdle" value="true"/>
```

정상 적용 시:

```text
업무 요청 시 getConnection()
→ validationQuery가 없으므로 Connection.isValid(3) 호출
→ MariaDB Connector/J가 protocol ping 수행
→ 끊긴 connection이면 false 또는 SQLException
→ DBCP2가 해당 connection drop
→ 다른 connection 또는 신규 connection 반환
```

DBCP2 `PoolableConnection` API 문서도 validation SQL이 null이면 driver의 `isValid(timeout)`을 호출하고, false를 반환하면 SQLException을 발생시키는 알고리즘을 설명합니다. ([Apache Commons](https://commons.apache.org/proper/commons-dbcp/apidocs/org/apache/commons/dbcp2/PoolableConnection.html "PoolableConnection (Apache Commons DBCP 2.14.0 API)"))

## 9. 운영 적용 권장 설정

`isValid()` 방식으로 전환하되, 요청 지연을 줄이고 idle 사전 정리를 우선하면:

```xml
<!-- validationQuery 제거 -->
<property name="validationQueryTimeout" value="3"/>
<property name="testWhileIdle" value="true"/>
<property name="testOnBorrow" value="false"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="20"/>
<property name="minEvictableIdleTimeMillis" value="-1"/>
<property name="softMinEvictableIdleTimeMillis" value="240000"/>
```

운영 중 `Connection reset by peer`가 사용자 요청에 계속 노출되면:

```xml
<!-- 안정성 우선 임시 설정 -->
<property name="validationQueryTimeout" value="3"/>
<property name="testWhileIdle" value="true"/>
<property name="testOnBorrow" value="true"/>
<property name="timeBetweenEvictionRunsMillis" value="60000"/>
<property name="numTestsPerEvictionRun" value="20"/>
<property name="softMinEvictableIdleTimeMillis" value="240000"/>
```

## 10. 주의 사항

|항목|주의점|
|---|---|
|SQL 로그|`SELECT 1`이 안 보이는 것이 정상일 수 있음|
|timeout|`validationQueryTimeout`은 양수 권장. `-1`/`0`은 timeout 없음으로 장애 감지가 늦을 수 있음|
|드라이버|MariaDB Connector/J 버전에 따라 `isValid()` 동작 차이 가능|
|pool 재기동|`validationQuery` 변경은 pool 초기화 후 반영되지 않을 수 있으므로 서버 재기동 필요|
|관측 방식|log4jdbc SQL 로그만으로 판단하지 말고 JMX/pool 로그/장애 주입을 같이 봐야 함|
|testWhileIdle|evictor가 모든 idle connection을 매번 검사하지 않으므로 `numTestsPerEvictionRun` 조정 필요|

## 최종 판단

정상 전환 여부는 아래 4개가 맞으면 됩니다.

```text
1. dataSource.getValidationQuery() == null
2. validation SQL 로그에서 SELECT 1이 사라짐
3. testWhileIdle 또는 testOnBorrow 시 끊긴 connection이 pool에서 제거됨
4. 업무 요청에서 Connection reset by peer 발생 빈도가 감소하거나 사라짐
```

가장 확실한 검증은 **`validationQuery=null` 확인 + `testOnBorrow=true` 임시 적용 + MariaDB connection KILL 테스트**입니다. 이 테스트에서 끊긴 connection이 업무 SQL까지 전달되지 않으면 `Connection.isValid()` 방식이 정상 동작한다고 판단할 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
