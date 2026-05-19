---
layout: single
title: "JtaTransactionManager"
excerpt: "JtaTransactionManager"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-19"
last_modified_at: "2026-05-19 13:28:51 +0900"
---
## 이 기종 DB간 Transaction

Spring 5.3 이후에서도 **서로 다른 이기종 DB 간 트랜잭션 정합성**을 유지하는 대표 방법은 여러 가지가 있지만, 실무 신뢰도 기준으로는 아래 2가지만 우선 검토하는 것이 맞습니다.

|  순위 | 방법                                    | 정합성 수준      | 추천 상황                                  |
| --: | ------------------------------------- | ----------- | -------------------------------------- |
|   1 | `JTA/XA` 분산 트랜잭션                      | 강한 정합성, 2PC | 두 DB 모두 반드시 동시에 commit/rollback되어야 할 때 |
|   2 | `Transactional Outbox + Saga/보상 트랜잭션` | 결과적 정합성     | XA가 어렵거나 운영 안정성/확장성이 더 중요할 때           |

- `ChainedTransactionManager`처럼 여러 `PlatformTransactionManager`를 순서대로 묶는 방식은 Spring Data에서 deprecated 되었고, 부분 commit 상태가 발생할 수 있어 inconsistent state를 허용/복구할 수 있을 때만 사용하라고 명시되어 있습니다. 따라서 신뢰도 높은 방법 2가지에는 포함하지 않는 것이 맞습니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))
## 전제 정리

Spring의 일반 `DataSourceTransactionManager`는 **하나의 JDBC DataSource에 대한 로컬 트랜잭션**입니다. Spring 공식 문서도 local transaction은 특정 resource에 종속되며 여러 transactional resource에 걸쳐 동작할 수 없다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))  
즉 아래 구조는 엄격한 의미의 단일 트랜잭션이 아닙니다.

```text
Service
 ├─ dao  → Main DB → txManager
 └─ mDao → Mail DB → txManagerMail
```

`txManager`, `txManagerMail`을 각각 적용하면 예외 발생 시 rollback은 각각 수행될 수 있지만, 정상 종료 후 commit 단계에서 **한쪽 commit 성공 + 다른 쪽 commit 실패**가 발생하면 원자성을 보장할 수 없습니다.

## 방법 1. JTA/XA 분산 트랜잭션

### 개념

`JTA/XA`는 여러 DB, JMS 같은 여러 transactional resource를 하나의 global transaction으로 묶는 방식입니다. Spring Framework는 JTA, JDBC, Hibernate, JPA 등 서로 다른 트랜잭션 API에 대해 일관된 추상화를 제공하고, 여러 resource가 필요한 경우 JTA를 사용할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))  
Spring의 `JtaTransactionManager`는 여러 resource에 걸친 distributed transaction을 처리하는 용도에 적합합니다. 단일 JDBC DataSource만 쓰는 경우에는 `DataSourceTransactionManager`로 충분하지만, 다중 resource라면 `JtaTransactionManager`가 대상입니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html?utm_source=chatgpt.com "JtaTransactionManager (Spring Framework 7.0.7 API)"))

### 처리 흐름

```text
1. Service 메소드 진입
2. JtaTransactionManager가 global transaction 시작
3. Main XA DataSource 참여
4. Mail XA DataSource 참여
5. 비즈니스 로직 정상 종료
6. prepare phase
7. 모든 resource prepare 성공 시 commit phase
8. 하나라도 실패하면 rollback
```

이 방식은 일반적으로 **2 Phase Commit, 2PC** 기반입니다.

```text
Phase 1: prepare
 - 각 DB에게 commit 가능한지 확인
Phase 2: commit 또는 rollback
 - 모두 가능하면 commit
 - 하나라도 불가능하면 rollback
```

### Spring 5.3 XML 예시

JBoss/WildFly 같은 WAS에서 XA DataSource를 JNDI로 제공하는 경우가 가장 전통적인 구조입니다.

```xml
<jee:jndi-lookup id="mainDataSource" jndi-name="java:/jdbc/MainXaDS"/>
<jee:jndi-lookup id="mailDataSource" jndi-name="java:/jdbc/MailXaDS"/>
<bean id="transactionManager"
      class="org.springframework.transaction.jta.JtaTransactionManager"/>
<tx:annotation-driven transaction-manager="transactionManager"/>
```

Service는 하나의 JTA 트랜잭션만 사용합니다.

```java
@Service
public class TemplateService {
    @Transactional(rollbackFor = Exception.class)
    public void deleteTemplate(Map<String, Object> deleteMap,
                               Map<String, Object> loginUserInfo) {
        deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
        dao.deleteReplcInfo(deleteMap);
        mDao.deleteEmsReplcInfo(deleteMap);
        dao.deleteRcvTgt(deleteMap);
        dao.deleteTemplate(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    }
}
```

핵심은 `dao`, `mDao`가 각각 별도 로컬 TM을 쓰는 것이 아니라, **둘 다 JTA-aware XA DataSource를 통해 같은 global transaction에 참여**해야 한다는 점입니다.

### Spring Boot 이후 버전 관점

Spring Boot 현재 문서도 JTA 환경이 감지되면 Spring의 `JtaTransactionManager`가 사용되고, auto-configured JMS, DataSource, JPA bean이 XA transaction을 지원하도록 구성된다고 설명합니다. 또한 current Boot 문서는 JNDI에서 가져온 transaction manager를 사용해 multiple XA resource 간 distributed JTA transaction을 지원한다고 설명합니다. ([Home](https://docs.spring.io/spring-boot/reference/io/jta.html "Distributed Transactions With JTA :: Spring Boot"))

### 장점

|항목|내용|
|---|---|
|정합성|두 DB를 하나의 global transaction으로 묶을 수 있음|
|코드 영향|Service 코드는 기존 `@Transactional` 방식과 유사|
|실패 처리|prepare 단계 실패 시 전체 rollback 가능|
|적합 업무|주문, 결제, 재고, 정산처럼 강한 정합성이 필요한 업무|

### 단점

|항목|내용|
|---|---|
|운영 복잡도|XA DataSource, Transaction Manager, recovery log 설정 필요|
|성능|2PC prepare/commit 과정 때문에 일반 로컬 트랜잭션보다 느림|
|락 유지|global transaction 시간이 길면 양쪽 DB lock 시간이 증가|
|장애 복구|in-doubt transaction, heuristic exception 대응 절차 필요|
|드라이버 의존|각 DB JDBC Driver의 XA 지원 품질에 영향 받음|

### 실무 주의점

|구분|주의점|
|---|---|
|XA 지원|MariaDB, Oracle, DB2, PostgreSQL 등 DB/Driver별 XA 지원 수준 확인 필요|
|DataSource|일반 `BasicDataSource`가 아니라 XA-aware DataSource 사용 필요|
|WAS|JBoss/WildFly 사용 시 서버의 transaction subsystem과 XA datasource 설정을 같이 봐야 함|
|Timeout|JTA timeout, DB lock timeout, query timeout을 일관되게 설계해야 함|
|Connection Pool|일반 DBCP2 설정만으로는 XA enlistment/recovery가 완전하지 않을 수 있음|
|Recovery|장애 후 transaction recovery log 위치, 권한, 백업 정책 필요|
|테스트|commit 직전 DB kill, 네트워크 단절, 한쪽 DB prepare 실패 테스트 필요|

### 구현 난이도

|구분|난이도|
|---|--:|
|개발 코드 수정|중|
|Spring 설정|중~상|
|WAS/JTA 설정|상|
|DB/Driver 검증|상|
|운영 장애 대응|상|

### 적용 판단

`dao`, `mDao`가 반드시 동시에 성공/실패해야 하고, 불일치가 발생하면 업무적으로 허용되지 않는다면 **JTA/XA가 가장 신뢰도 높은 정공법**입니다.

## 방법 2. Transactional Outbox + Saga/보상 트랜잭션

### 개념

`Transactional Outbox`는 하나의 DB 트랜잭션 안에서 **업무 데이터 변경 + 후속 처리 이벤트 저장**을 같이 commit하고, 별도 relay/batch/consumer가 outbox 이벤트를 읽어 다른 DB나 외부 시스템에 반영하는 방식입니다. Microservices.io의 패턴 설명도, 메시지를 먼저 DB에 저장하고 별도 프로세스가 message broker로 전송하는 방식이라고 설명합니다. ([microservices.io](https://microservices.io/patterns/data/transactional-outbox.html "Pattern: Transactional outbox"))  
이 방식은 두 DB를 동시에 commit하는 것이 아닙니다. 대신 아래를 보장합니다.

```text
Main DB 변경이 commit되면 outbox 이벤트도 반드시 남는다.
Main DB 변경이 rollback되면 outbox 이벤트도 남지 않는다.
outbox 이벤트를 기준으로 mDao 쪽 처리를 재시도/보상한다.
```

Spring Modulith도 Event Publication Registry를 통해 원래 비즈니스 트랜잭션 안에서 이벤트 publication log를 기록하고, listener 실패 시 log를 남겨 재시도할 수 있는 구조를 제공합니다. ([Home](https://docs.spring.io/spring-modulith/reference/events.html "Working with Application Events :: Spring Modulith"))

### 처리 흐름

```text
1. Service에서 Main DB 트랜잭션 시작
2. dao 삭제 처리
3. outbox 테이블에 Mail DB 삭제 요청 이벤트 저장
4. Main DB commit
5. 별도 relay 또는 batch가 outbox 이벤트 조회
6. mDao 삭제 처리
7. 성공 시 outbox 상태 DONE
8. 실패 시 RETRY 또는 보상 대상 기록
```

### 테이블 예시

```sql
CREATE TABLE TX_OUTBOX (
    OUTBOX_ID        BIGINT PRIMARY KEY,
    EVENT_TYPE       VARCHAR(100) NOT NULL,
    AGGREGATE_ID     VARCHAR(100) NOT NULL,
    PAYLOAD          CLOB NOT NULL,
    STATUS           VARCHAR(20) NOT NULL,
    RETRY_COUNT      INT NOT NULL,
    CREATED_AT       TIMESTAMP NOT NULL,
    UPDATED_AT       TIMESTAMP
);
```

### Service 예시

```java
@Service
public class TemplateService {
    @Transactional(
        transactionManager = "txManager",
        rollbackFor = Exception.class
    )
    public void deleteTemplate(Map<String, Object> deleteMap,
                               Map<String, Object> loginUserInfo) {
        deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
        dao.deleteReplcInfo(deleteMap);
        dao.deleteRcvTgt(deleteMap);
        dao.deleteTemplate(deleteMap);
        outboxDao.insertOutboxEvent(
            "MAIL_TEMPLATE_DELETE",
            String.valueOf(deleteMap.get("TEMP_ID")),
            toPayload(deleteMap)
        );
    }
}
```

### Relay 예시

```java
@Component
public class MailTemplateDeleteRelay {
    @Scheduled(fixedDelay = 10000)
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxDao.selectPendingEvents("MAIL_TEMPLATE_DELETE", 100);
        for (OutboxEvent event : events) {
            try {
                process(event);
                outboxDao.markDone(event.getOutboxId());
            } catch (Exception e) {
                outboxDao.increaseRetryCount(event.getOutboxId(), e.getMessage());
            }
        }
    }
    @Transactional(
        transactionManager = "txManagerMail",
        rollbackFor = Exception.class
    )
    public void process(OutboxEvent event) {
        Map<String, Object> param = fromPayload(event.getPayload());
        mDao.deleteEmsReplcInfo(param);
        mDao.deleteEmsTemp(param);
    }
}
```

### 장점

|항목|내용|
|---|---|
|운영 안정성|XA보다 장애 복구가 단순함|
|확장성|DB 간 강결합을 줄일 수 있음|
|재처리|실패 이벤트를 재시도 가능|
|장애 추적|outbox 상태로 누락/실패 건을 추적 가능|
|적합 업무|메일, 알림, 로그, 통계, 외부 연계, 후속 반영성 업무|

### 단점

|항목|내용|
|---|---|
|정합성|즉시 일관성이 아니라 결과적 일관성|
|지연|Main DB commit 후 Mail DB 반영까지 시간차 발생|
|중복 처리|relay 재시도 때문에 중복 실행 가능성 있음|
|구현량|outbox 테이블, relay, 재시도, 모니터링 필요|
|업무 설계|보상/재처리 정책을 업무적으로 정의해야 함|

### 실무 주의점

|구분|주의점|
|---|---|
|Idempotency|`mDao` 처리는 같은 이벤트가 2번 실행되어도 결과가 같아야 함|
|상태값|`READY`, `PROCESSING`, `DONE`, `FAIL`, `DEAD` 같은 상태 관리 필요|
|락 처리|여러 WAS가 relay를 돌리면 `SELECT FOR UPDATE SKIP LOCKED` 또는 상태 선점 필요|
|순서 보장|동일 template/order/customer 단위 순서가 필요하면 aggregate key 기준 직렬화 필요|
|보상 정책|삭제 실패 시 Main DB 복원인지, Mail DB 재시도인지, 운영자 수동 처리인지 결정 필요|
|모니터링|미처리 건수, 실패 건수, retry 초과 건수 알림 필요|
|중복 방지|outbox event id 또는 business key 기준 unique 처리 권장|

### 구현 난이도

|구분|난이도|
|---|--:|
|개발 코드 수정|중|
|DB 테이블 추가|중|
|재시도/배치 구현|중|
|모니터링 구현|중~상|
|운영 장애 대응|중|

### 적용 판단

`mDao`가 메일/EMS/외부 연계성 DB이고, Main DB 삭제가 기준 데이터이며, Mail DB 반영은 재처리 가능하다면 **Outbox + Saga/보상 방식이 실무적으로 더 안전할 수 있습니다.**

## 두 방법 비교

|항목|JTA/XA|Outbox + Saga|
|---|---|---|
|정합성|강한 정합성|결과적 정합성|
|commit 원자성|높음|없음|
|장애 복구|복잡|상대적으로 명확|
|성능|낮아질 수 있음|상대적으로 유리|
|DB 락 시간|길어질 수 있음|짧음|
|구현 난이도|상|중~상|
|운영 난이도|상|중|
|DB/Driver 의존|높음|낮음|
|재처리 구조|JTA recovery 중심|업무 outbox/retry 중심|
|추천 업무|결제/정산/재고처럼 즉시 정합성 필수|메일/알림/연계/부가 데이터 동기화|

## `Propagation`, `Isolation`과의 관계

|항목|관계|
|---|---|
|`Propagation`|트랜잭션 참여/분리 방식을 결정하지만, 로컬 TM 2개를 하나의 원자적 commit으로 만들지는 못함|
|`Isolation`|각 DB 내부의 동시성 제어 수준이며, 서로 다른 DB 간 commit 원자성을 해결하지 못함|
|`rollbackFor`|예외 발생 시 rollback 범위에 영향 있음|
|`timeout`|JTA/XA에서는 global transaction timeout 설계가 중요|
|Spring `TransactionDefinition`은 propagation, isolation, timeout, readOnly 같은 표준 트랜잭션 속성을 제공하지만, 이것은 트랜잭션 범위와 동시성 제어 속성이지 다중 DB commit 원자성을 자동으로 만들어주는 설정은 아닙니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))||

## 현재 `dao`/`mDao` 구조 기준 권장 판단

### Case 1. `mDao`도 반드시 즉시 일관성 필요

예:

```text
Main DB 삭제 성공 + Mail DB 삭제 실패가 절대 허용되지 않음
```

권장:

```text
JTA/XA
```

구조:

```text
TemplateService
 └─ @Transactional
     ├─ dao  → Main XA DataSource
     └─ mDao → Mail XA DataSource
```

### Case 2. `mDao`가 EMS/메일/부가 시스템 성격

예:

```text
Main DB가 기준 데이터이고, mDao는 후속 동기화/삭제 대상
실패 시 재처리 가능
```

권장:

```text
Transactional Outbox + 재처리/보상
```

구조:

```text
TemplateService
 ├─ Main DB 삭제
 └─ Outbox 이벤트 저장
Relay/Batch
 └─ Mail DB 삭제 재시도
```

## 피해야 할 방식

### 1. Service 하나에 `txManager`, `txManagerMail`을 둘 다 걸고 해결했다고 보는 방식

예외 발생 시 rollback은 둘 다 될 수 있지만, commit 단계에서 한쪽만 성공할 수 있습니다.

```text
txManagerMail commit 성공
txManager commit 실패
→ 불일치
```

### 2. `ChainedTransactionManager`

Spring Data 문서상 deprecated이며, 첫 번째 TransactionManager는 commit됐고 이후 TransactionManager commit이 실패하는 partial commit 상태가 발생할 수 있다고 명시되어 있습니다. 문서에서도 inconsistent state를 허용하거나 복구할 수 있을 때만 사용하라고 설명합니다. ([Home](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/transaction/ChainedTransactionManager.html "ChainedTransactionManager (Spring Data Core 4.0.5 API)"))

### 3. `REQUIRES_NEW` 남용

`REQUIRES_NEW`는 독립 트랜잭션을 만들기 때문에 전체 정합성을 강화하는 것이 아니라 오히려 한쪽 선commit 구조를 만들 수 있습니다.

## 최종 추천

현재 질문의 `dao`/`mDao` 구조가 **Main DB + Mail/EMS DB** 성격이라면 1차 추천은 아래입니다.

```text
1순위: Outbox + 재처리/보상
2순위: 정말 강한 정합성이 필요할 때만 JTA/XA
```

반대로 두 DB가 모두 주문/결제/재고처럼 **동시 commit/rollback이 업무적으로 필수**라면 아래가 맞습니다.

```text
1순위: JTA/XA
2순위: 업무 재설계 후 Outbox/Saga
```

Spring 5.3 자체는 JTA 추상화를 지원하지만, Spring Framework 5.x의 OSS 지원은 2024년 8월 31일 종료되었으므로 신규 구성이라면 Spring 6.x 또는 지원 가능한 상용 지원 정책도 함께 검토하는 것이 안전합니다. ([github.com](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-5.3-Release-Notes "Spring Framework 5.3 Release Notes · spring-projects/spring-framework Wiki · GitHub"))

# JtaTransaction Manager

## 결론

Spring 5.3에서 `dao`, `mDao`가 **서로 다른 DB**를 사용하면서도 하나의 트랜잭션처럼 commit/rollback되어야 한다면, 실무 정공법은 **`JtaTransactionManager` + WAS/JTA Transaction Manager + XA DataSource 2개** 구조입니다. Spring 공식 문서도 로컬 트랜잭션은 특정 JDBC Connection 같은 단일 리소스에 종속되며 여러 transactional resource에 걸쳐 동작할 수 없고, 여러 리소스를 다뤄야 할 때 Application Server의 JTA 기능이 필요하다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))

## 1. 목표 구조

```text
TemplateService.deleteTemplate()
 └─ JTA Global Transaction
     ├─ dao  → Main XA DataSource → Main DB
     └─ mDao → Mail XA DataSource → Mail/EMS DB
```

핵심은 `dao`는 `txManager`, `mDao`는 `txManagerMail`로 각각 묶는 구조가 아니라, **둘 다 하나의 `JtaTransactionManager`가 관리하는 global transaction에 참여**하게 만드는 것입니다.

## 2. 적용 전제

|구분|필수 조건|
|---|---|
|DB|두 DB 모두 XA Transaction 지원 필요|
|JDBC Driver|XA 지원 Driver 필요|
|DataSource|일반 DBCP2 `BasicDataSource`가 아니라 WAS에 등록된 XA DataSource 권장|
|Spring TM|`DataSourceTransactionManager`가 아니라 `JtaTransactionManager` 사용|
|DAO|`DriverManager.getConnection()` 직접 사용 금지|
|MyBatis|Spring-managed transaction에 참여하도록 `SqlSessionTemplate`/`SqlSessionFactoryBean` 사용|
|예외|rollback 대상 예외가 Service 밖으로 전파되어야 함|
|JBoss/WildFly 계열에서는 XA DataSource를 Management CLI 또는 Console로 생성할 수 있고, Red Hat JBoss EAP 문서도 XA datasource 생성 절차와 XA recovery 구성을 별도 항목으로 설명합니다. 특히 JDBC/JMS XA resource는 recovery module이 필요하며, 일반적인 JDBC/JMS resource는 recovery module이 자동 연결될 수 있지만 recovery 접속 정보는 설정해야 합니다. ([레드햇 문서](https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/6.4/html/administration_and_configuration_guide/sect-xa_datasources "6.4. XA Datasources \| Administration and Configuration Guide \| Red Hat JBoss Enterprise Application Platform \| 6.4 \| Red Hat Documentation"))||

## 3. JBoss/WildFly XA DataSource 예시

아래는 개념 예시입니다. 실제 `driver-name`, `xa-datasource-class`, XA property 명칭은 DB/Driver 버전에 맞춰 확인해야 합니다.

```bash
# Main DB XA DataSource 생성 예시
/subsystem=datasources/xa-data-source=MainXaDS:add(
    jndi-name=java:/jdbc/MainXaDS,
    driver-name=mariadb-java-client.jar,
    xa-datasource-class=VENDOR_XA_DATASOURCE_CLASS,
    user-name=main_user,
    password=main_password
)
# XA property 예시
/subsystem=datasources/xa-data-source=MainXaDS/xa-datasource-properties=ServerName:add(value=192.168.100.130)
/subsystem=datasources/xa-data-source=MainXaDS/xa-datasource-properties=PortNumber:add(value=3306)
/subsystem=datasources/xa-data-source=MainXaDS/xa-datasource-properties=DatabaseName:add(value=MAIN_DB)
# 활성화
/subsystem=datasources/xa-data-source=MainXaDS:enable
```

```bash
# Mail/EMS DB XA DataSource 생성 예시
/subsystem=datasources/xa-data-source=MailXaDS:add(
    jndi-name=java:/jdbc/MailXaDS,
    driver-name=mariadb-java-client.jar,
    xa-datasource-class=VENDOR_XA_DATASOURCE_CLASS,
    user-name=mail_user,
    password=mail_password
)
/subsystem=datasources/xa-data-source=MailXaDS/xa-datasource-properties=ServerName:add(value=192.168.100.131)
/subsystem=datasources/xa-data-source=MailXaDS/xa-datasource-properties=PortNumber:add(value=3306)
/subsystem=datasources/xa-data-source=MailXaDS/xa-datasource-properties=DatabaseName:add(value=MAIL_DB)
/subsystem=datasources/xa-data-source=MailXaDS:enable
```

Red Hat 문서의 XA datasource 생성 예시도 `xa-data-source add --name=... --jndi-name=... --driver-name=... --xa-datasource-class=...` 형식으로 생성한 뒤, `ServerName`, `DatabaseName` 같은 XA datasource property를 추가하고 datasource를 enable하는 흐름을 제시합니다. ([레드햇 문서](https://docs.redhat.com/en/documentation/red_hat_jboss_enterprise_application_platform/6.4/html/administration_and_configuration_guide/sect-xa_datasources "6.4. XA Datasources | Administration and Configuration Guide | Red Hat JBoss Enterprise Application Platform | 6.4 | Red Hat Documentation"))

## 4. Spring 5.3 XML 설정 예시

### 4.1 JNDI XA DataSource 조회

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:jee="http://www.springframework.org/schema/jee"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans https://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/jee https://www.springframework.org/schema/jee/spring-jee.xsd
           http://www.springframework.org/schema/tx https://www.springframework.org/schema/tx/spring-tx.xsd">
    <!-- Main DB XA DataSource -->
    <jee:jndi-lookup id="mainDataSource"
                     jndi-name="java:/jdbc/MainXaDS"
                     resource-ref="false"/>
    <!-- Mail/EMS DB XA DataSource -->
    <jee:jndi-lookup id="mailDataSource"
                     jndi-name="java:/jdbc/MailXaDS"
                     resource-ref="false"/>
</beans>
```

### 4.2 `JtaTransactionManager` 단일 등록

```xml
<bean id="transactionManager"
      class="org.springframework.transaction.jta.JtaTransactionManager">
    <!-- 초 단위. 업무 특성에 맞게 조정 -->
    <property name="defaultTimeout" value="30"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager"
                      proxy-target-class="true"/>
```

Spring 5.3 문서 기준으로 `JtaTransactionManager`와 `DataSourceTransactionManager`는 설정만 바꾸면 같은 코드 모델을 유지할 수 있지만, JTA는 savepoint/custom isolation을 지원하지 않고 timeout 메커니즘도 다릅니다. 따라서 JTA 구성에서는 `PROPAGATION_NESTED`나 메소드별 isolation 조정을 전제로 설계하면 안 됩니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html?utm_source=chatgpt.com "Data Access"))

## 5. MyBatis 기반 `dao`, `mDao` 설정 예시

### 5.1 Main DB SqlSessionFactory

```xml
<bean id="mainSqlSessionFactory"
      class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="mainDataSource"/>
    <property name="configLocation" value="classpath:/mybatis/mybatis-config.xml"/>
    <property name="mapperLocations" value="classpath:/mybatis/main/**/*.xml"/>
</bean>
<bean id="mainSqlSessionTemplate"
      class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg ref="mainSqlSessionFactory"/>
</bean>
```

### 5.2 Mail DB SqlSessionFactory

```xml
<bean id="mailSqlSessionFactory"
      class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource" ref="mailDataSource"/>
    <property name="configLocation" value="classpath:/mybatis/mybatis-config.xml"/>
    <property name="mapperLocations" value="classpath:/mybatis/mail/**/*.xml"/>
</bean>
<bean id="mailSqlSessionTemplate"
      class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg ref="mailSqlSessionFactory"/>
</bean>
```

MyBatis-Spring은 Spring Transaction Manager가 설정되면 트랜잭션을 투명하게 관리하고, 트랜잭션 동안 하나의 `SqlSession`을 사용하며 트랜잭션 완료 시 commit 또는 rollback한다고 설명합니다. 또한 DAO 클래스에 별도 트랜잭션 코드를 넣을 필요가 없다고 설명합니다. ([MyBatis](https://mybatis.org/spring/transactions.html?utm_source=chatgpt.com "Transactions - mybatis-spring"))

### 5.3 DAO Bean 예시

```xml
<bean id="dao" class="com.example.template.dao.TemplateDao">
    <property name="sqlSessionTemplate" ref="mainSqlSessionTemplate"/>
</bean>
<bean id="mDao" class="com.example.template.dao.MailTemplateDao">
    <property name="sqlSessionTemplate" ref="mailSqlSessionTemplate"/>
</bean>
```

```java
public class TemplateDao {
    private SqlSessionTemplate sqlSessionTemplate;
    public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
        this.sqlSessionTemplate = sqlSessionTemplate;
    }
    public int deleteReplcInfo(Map<String, Object> param) {
        return sqlSessionTemplate.delete("template.deleteReplcInfo", param);
    }
    public int deleteRcvTgt(Map<String, Object> param) {
        return sqlSessionTemplate.delete("template.deleteRcvTgt", param);
    }
    public int deleteTemplate(Map<String, Object> param) {
        return sqlSessionTemplate.delete("template.deleteTemplate", param);
    }
}
```

```java
public class MailTemplateDao {
    private SqlSessionTemplate sqlSessionTemplate;
    public void setSqlSessionTemplate(SqlSessionTemplate sqlSessionTemplate) {
        this.sqlSessionTemplate = sqlSessionTemplate;
    }
    public int deleteEmsReplcInfo(Map<String, Object> param) {
        return sqlSessionTemplate.delete("mailTemplate.deleteEmsReplcInfo", param);
    }
    public int deleteEmsTemp(Map<String, Object> param) {
        return sqlSessionTemplate.delete("mailTemplate.deleteEmsTemp", param);
    }
}
```

## 6. Service 예시

```java
@Service
public class TemplateService {
    private final TemplateDao dao;
    private final MailTemplateDao mDao;
    public TemplateService(TemplateDao dao, MailTemplateDao mDao) {
        this.dao = dao;
        this.mDao = mDao;
    }
    @Transactional(
        rollbackFor = Exception.class,
        timeout = 30
    )
    public void deleteTemplate(Map<String, Object> deleteMap,
                               Map<String, Object> loginUserInfo) throws Exception {
        deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
        // Main DB
        dao.deleteReplcInfo(deleteMap);
        dao.deleteRcvTgt(deleteMap);
        dao.deleteTemplate(deleteMap);
        // Mail/EMS DB
        mDao.deleteEmsReplcInfo(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    }
}
```

이 코드에서 `@Transactional`은 `dao`와 `mDao` 각각의 로컬 트랜잭션을 시작하는 것이 아니라, Spring의 단일 `JtaTransactionManager`를 통해 **JTA global transaction**을 시작합니다. Spring은 `PlatformTransactionManager` 추상화를 통해 JTA/JDBC 등 실제 트랜잭션 기술이 바뀌어도 선언적 트랜잭션 모델을 유지할 수 있게 합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))

## 7. Rollback 테스트 예시

### 7.1 `mDao` 이후 강제 예외

```java
@Transactional(rollbackFor = Exception.class, timeout = 30)
public void deleteTemplateForTest(Map<String, Object> deleteMap,
                                  Map<String, Object> loginUserInfo) throws Exception {
    deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
    dao.deleteReplcInfo(deleteMap);
    dao.deleteRcvTgt(deleteMap);
    dao.deleteTemplate(deleteMap);
    mDao.deleteEmsReplcInfo(deleteMap);
    mDao.deleteEmsTemp(deleteMap);
    if ("Y".equals(deleteMap.get("FORCE_ROLLBACK"))) {
        throw new RuntimeException("JTA rollback test");
    }
}
```

기대 결과:

```text
1. dao 삭제 실행
2. mDao 삭제 실행
3. RuntimeException 발생
4. JTA global transaction rollback
5. Main DB, Mail DB 모두 삭제 전 상태로 복구
```

### 7.2 `dao` 이후 예외

```java
@Transactional(rollbackFor = Exception.class, timeout = 30)
public void deleteTemplateForDaoRollbackTest(Map<String, Object> deleteMap,
                                             Map<String, Object> loginUserInfo) throws Exception {
    deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
    dao.deleteReplcInfo(deleteMap);
    dao.deleteRcvTgt(deleteMap);
    dao.deleteTemplate(deleteMap);
    throw new RuntimeException("rollback after main dao");
}
```

기대 결과:

```text
Main DB 삭제 rollback
Mail DB는 호출되지 않음
```

## 8. 예외 처리 주의

### 8.1 예외를 삼키면 안 됨

```java
@Transactional(rollbackFor = Exception.class)
public void deleteTemplate(Map<String, Object> deleteMap,
                           Map<String, Object> loginUserInfo) {
    try {
        dao.deleteTemplate(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    } catch (Exception e) {
        // 나쁜 예: 로그만 남기고 정상 종료
        log.error("delete failed", e);
    }
}
```

위 코드는 Service가 정상 종료되므로 commit될 수 있습니다. 반드시 다시 던져야 합니다.

```java
@Transactional(rollbackFor = Exception.class)
public void deleteTemplate(Map<String, Object> deleteMap,
                           Map<String, Object> loginUserInfo) {
    try {
        dao.deleteTemplate(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    } catch (Exception e) {
        log.error("delete failed", e);
        throw new TemplateDeleteException("template delete failed", e);
    }
}
```

```java
public class TemplateDeleteException extends RuntimeException {
    public TemplateDeleteException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## 9. Propagation/Isolation 설정 기준

### 9.1 권장 기본값

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class,
    timeout = 30
)
```

|항목|권장|이유|
|---|---|---|
|`Propagation`|`REQUIRED`|Service 진입 시 하나의 JTA global transaction 생성|
|`Isolation`|명시하지 않음|JTA 표준에서 custom isolation 제약 있음|
|`rollbackFor`|`Exception.class`|checked exception까지 rollback 대상 포함|
|`timeout`|업무별 지정|장기 lock 방지|
|`JtaTransactionManager`는 timeout은 지원하지만 per-transaction isolation level은 지원하지 않는다고 Spring Javadoc에 명시되어 있습니다. Spring 5.3 문서도 JTA는 savepoint/custom isolation을 지원하지 않는다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html?utm_source=chatgpt.com "JtaTransactionManager (Spring Framework 7.0.7 API)"))|||

### 9.2 피해야 할 설정

```java
@Transactional(
    propagation = Propagation.NESTED,
    isolation = Isolation.SERIALIZABLE
)
```

JTA에서는 `NESTED`/savepoint 기반 설계를 기대하면 안 되고, isolation을 업무 메소드별로 바꾸는 구조도 일반적이지 않습니다.

## 10. JTA 방식에서 DBCP2 설정은 어떻게 되는가

기존처럼 Spring Bean으로 아래를 쓰는 구조는 JTA XA 구성과 맞지 않습니다.

```xml
<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
    ...
</bean>
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

JTA/XA 구성에서는 보통 WAS의 XA DataSource pool을 사용하고 Spring은 JNDI로 그 DataSource를 조회합니다.

```xml
<jee:jndi-lookup id="mainDataSource" jndi-name="java:/jdbc/MainXaDS"/>
```

따라서 기존 DBCP2의 `validationQuery`, `validationQueryTimeout`, `testWhileIdle` 같은 설정은 Spring DBCP2 Bean이 아니라 **WAS datasource pool 설정**에서 대응해야 합니다. WildFly datasource 모델에도 `check-valid-connection-sql` 같은 커넥션 검증 SQL 속성이 존재합니다. ([WildFly Documentation](https://docs.wildfly.org/20/wildscribe/subsystem/datasources/xa-data-source/index.html "WildFly Full 20 Model Reference"))

## 11. 실무 체크리스트

|구분|확인 내용|
|---|---|
|JNDI|`java:/jdbc/MainXaDS`, `java:/jdbc/MailXaDS`가 서버에서 정상 lookup되는지|
|XA|두 DataSource가 일반 datasource가 아니라 XA datasource인지|
|Driver|사용 중인 DB Driver가 XA를 지원하는지|
|Recovery|XA recovery username/password 또는 보안 설정이 되어 있는지|
|Spring|`JtaTransactionManager` 하나만 사용하는지|
|DAO|`DriverManager` 직접 연결이 없는지|
|MyBatis|`SqlSessionTemplate` 사용 여부|
|예외|Service 예외가 밖으로 전파되는지|
|Timeout|JTA timeout, query timeout, DB lock timeout이 과도하게 길지 않은지|
|테스트|한쪽 DB 장애, commit 직전 장애, timeout, WAS 재기동 복구 테스트 수행|

## 12. 장애 시나리오별 기대 동작

|시나리오|기대 동작|
|---|---|
|`dao` 실행 중 예외|Global transaction rollback|
|`mDao` 실행 중 예외|Main DB + Mail DB 모두 rollback|
|Service 마지막에서 RuntimeException|Main DB + Mail DB 모두 rollback|
|Checked Exception 발생|`rollbackFor=Exception.class`가 있으면 rollback|
|예외 catch 후 정상 종료|commit될 수 있음|
|WAS 장애로 in-doubt 발생|JTA recovery 대상|
|한쪽 DB prepare 실패|전체 rollback 기대|
|한쪽 DB commit 단계 장애|JTA recovery/heuristic 상태 확인 필요|

## 13. 구현 난이도와 운영상 주의점

|항목|난이도|주의점|
|---|--:|---|
|Spring 설정|중|`DataSourceTransactionManager` 제거, `JtaTransactionManager` 단일화|
|WAS 설정|상|XA datasource, recovery, transaction timeout 필요|
|DB 설정|중~상|XA 지원, 권한, lock timeout 확인|
|코드 수정|중|Service 트랜잭션 경계 정리, 예외 처리 수정|
|테스트|상|부분 장애/복구 시나리오 필수|
|운영|상|in-doubt, heuristic exception, recovery log 확인 절차 필요|

## 14. 현재 `deleteTemplate()`에 적용한 최종 예시

```java
@Service
public class TemplateService {
    private final TemplateDao dao;
    private final MailTemplateDao mDao;
    public TemplateService(TemplateDao dao, MailTemplateDao mDao) {
        this.dao = dao;
        this.mDao = mDao;
    }
    @Transactional(
        propagation = Propagation.REQUIRED,
        rollbackFor = Exception.class,
        timeout = 30
    )
    public void deleteTemplate(Map<String, Object> deleteMap,
                               Map<String, Object> loginUserInfo) throws Exception {
        deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
        // 1. Main DB 삭제
        dao.deleteReplcInfo(deleteMap);
        dao.deleteRcvTgt(deleteMap);
        dao.deleteTemplate(deleteMap);
        // 2. Mail/EMS DB 삭제
        mDao.deleteEmsReplcInfo(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    }
}
```

이 구조에서 `dao`, `mDao` 모두 XA DataSource를 사용하고, Spring의 트랜잭션 매니저가 `JtaTransactionManager` 하나로 통일되어 있다면, Service 메소드에서 예외가 발생했을 때 두 DB 작업은 같은 global transaction의 rollback 대상이 됩니다.

## 15. 최종 판단

JTA 방식의 핵심은 아래 3가지입니다.

```text
1. DataSource 2개를 모두 XA DataSource로 WAS에 등록
2. Spring에서는 JNDI로 XA DataSource를 조회
3. TransactionManager는 JtaTransactionManager 하나만 사용
```

현재 `dao`, `mDao` 구조에 JTA를 적용하려면 기존처럼 `txManager`, `txManagerMail`을 따로 두는 방식에서 벗어나야 합니다. 실무적으로는 **JTA 설정 자체보다 XA datasource/recovery/장애 테스트가 더 중요**하며, 이 부분이 검증되지 않으면 코드상 `@Transactional`이 있어도 운영에서 완전한 신뢰성을 보장하기 어렵습니다.


# Propagation 과 Isolation

## 결론

`JtaTransactionManager` 사용 시 `Propagation`은 **실제로 중요**하지만, `Isolation`은 **대부분 `DEFAULT`로 두는 것이 실무 표준에 가깝습니다.**  
Spring JTA에서는 여러 XA 리소스를 하나의 Global Transaction으로 묶을 수 있지만, 표준 `JtaTransactionManager`는 **per-transaction isolation level을 기본 지원하지 않습니다.** Spring 공식 Javadoc도 표준 `JtaTransactionManager`는 timeout은 지원하지만 per-transaction isolation은 지원하지 않는다고 명시합니다. 또한 기본 구현은 `ISOLATION_DEFAULT` 외의 isolation이 지정되면 예외를 던질 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html "JtaTransactionManager (Spring Framework 7.0.7 API)"))

## 1. JTA에서 권장 기본 설정

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT,
    rollbackFor = Exception.class,
    timeout = 30
)
public void deleteTemplate(Map<String, Object> deleteMap,
                           Map<String, Object> loginUserInfo) throws Exception {
    deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
    dao.deleteReplcInfo(deleteMap);
    mDao.deleteEmsReplcInfo(deleteMap);
    dao.deleteRcvTgt(deleteMap);
    dao.deleteTemplate(deleteMap);
    mDao.deleteEmsTemp(deleteMap);
}
```

실무 기본값은 아래와 같이 보는 것이 안전합니다.

|항목|권장값|이유|
|---|---|---|
|`transactionManager`|JTA 단일 TM|`dao`, `mDao`를 하나의 global transaction으로 묶기 위함|
|`propagation`|`REQUIRED`|Service 진입 시 기존 JTA 트랜잭션 참여 또는 신규 생성|
|`isolation`|`DEFAULT`|표준 JTA에서 per-transaction isolation 미지원|
|`rollbackFor`|`Exception.class`|checked exception까지 rollback 대상 포함|
|`timeout`|업무 기준 초 단위|장기 lock/2PC 지연 방지|

## 2. Propagation 사용 기준

Spring `@Transactional`의 기본 propagation은 `REQUIRED`이고, isolation 기본값은 `DEFAULT`입니다. 또한 기본 rollback은 `RuntimeException`/`Error` 대상이며 checked exception은 기본 rollback 대상이 아닙니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html "Using @Transactional :: Spring Framework"))

### 2.1 `REQUIRED`

JTA에서 가장 일반적인 설정입니다.

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    rollbackFor = Exception.class
)
public void deleteTemplate(...) {
    dao.deleteTemplate(...);
    mDao.deleteEmsTemp(...);
}
```

동작:

```text
기존 JTA 트랜잭션이 있으면 참여
없으면 새 JTA Global Transaction 생성
dao, mDao가 XA DataSource를 사용하면 같은 Global Transaction에 enlist
```

`PROPAGATION_REQUIRED`는 각 메소드에 논리적 트랜잭션 범위를 만들지만, 표준 동작에서는 같은 물리 트랜잭션에 매핑됩니다. 내부 scope가 rollback-only를 설정하면 외부 commit 시점에 `UnexpectedRollbackException`이 발생할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))

### 2.2 `REQUIRES_NEW`

`REQUIRES_NEW`는 기존 트랜잭션에 참여하지 않고 **독립적인 물리 트랜잭션**을 새로 만듭니다. 내부 트랜잭션은 외부 트랜잭션과 별도로 commit/rollback될 수 있습니다. Spring 공식 문서는 `REQUIRES_NEW` 사용 시 외부 트랜잭션의 리소스가 유지된 상태에서 내부 트랜잭션이 새 리소스를 획득하므로 커넥션 풀 고갈 및 deadlock 위험이 있고, 동시 스레드 수보다 최소 1개 이상 크게 pool을 잡으라고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW,
    rollbackFor = Exception.class
)
public void saveAuditLog(...) {
    auditDao.insert(...);
}
```

적합한 경우:

```text
업무 트랜잭션이 rollback되어도 감사 로그, 실패 이력, 보상 요청 이력은 남겨야 하는 경우
```

주의:

```text
REQUIRES_NEW는 전체 원자성을 강화하는 설정이 아님
오히려 외부 트랜잭션 rollback과 무관하게 내부 트랜잭션이 commit될 수 있음
```

### 2.3 `MANDATORY`

기존 트랜잭션이 반드시 있어야 할 때 사용합니다.

```java
@Transactional(
    propagation = Propagation.MANDATORY,
    rollbackFor = Exception.class
)
public void deleteMailTemplate(...) {
    mDao.deleteEmsTemp(...);
}
```

적합한 경우:

```text
이 메소드는 반드시 상위 JTA 트랜잭션 안에서만 호출되어야 한다는 제약을 강제할 때
```

실무적으로 `dao`, `mDao` 하위 서비스가 단독 호출되면 위험한 경우에 사용할 수 있습니다.

### 2.4 `SUPPORTS`

트랜잭션이 있으면 참여하고, 없으면 non-transactional로 실행됩니다.

```java
@Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
public Template getTemplate(...) {
    return dao.selectTemplate(...);
}
```

적합한 경우:

```text
조회성 메소드
상위 트랜잭션이 있으면 일관성 있게 읽고, 없으면 단순 조회로 처리해도 되는 경우
```

주의:

```text
수정/삭제 메소드에는 부적합
상위 트랜잭션 없이 호출되면 rollback 대상이 아님
```

### 2.5 `NOT_SUPPORTED`

기존 트랜잭션을 중단하고 non-transactional로 실행합니다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void callExternalApi(...) {
    externalClient.call(...);
}
```

적합한 경우:

```text
외부 API 호출, 장시간 파일 I/O, 메일 발송처럼 DB lock을 잡은 채 수행하면 위험한 작업
```

주의:

```text
JTA transaction suspend/resume 동작이 필요
표준 JTA 환경에서 suspend 지원 여부와 WAS 구현을 확인해야 함
```

Spring `JtaTransactionManager` Javadoc도 transaction suspension, 즉 `REQUIRES_NEW`, `NOT_SUPPORTED`는 JTA `TransactionManager`가 등록되어 있어야 가능하다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html "JtaTransactionManager (Spring Framework 7.0.7 API)"))

### 2.6 `NESTED`

JTA에서는 일반적으로 권장하지 않습니다.

```java
// JTA Global Transaction에서는 실무 권장하지 않음
@Transactional(propagation = Propagation.NESTED)
```

Spring 공식 문서는 `PROPAGATION_NESTED`가 하나의 물리 트랜잭션 안에서 savepoint를 사용하며, 보통 JDBC resource transaction에서 `DataSourceTransactionManager`에 매핑된다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))  
즉 JTA/XA 기반 `dao + mDao` 분산 트랜잭션에서는 `NESTED`를 전제로 설계하지 않는 것이 안전합니다.

## 3. Isolation 사용 기준

### 3.1 JTA에서는 `Isolation.DEFAULT` 권장

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT,
    rollbackFor = Exception.class
)
```

Spring `@Transactional` Javadoc 기준으로 isolation은 새로 시작되는 트랜잭션에만 적용되며, 주로 `REQUIRED` 또는 `REQUIRES_NEW`와 함께 사용하도록 설계되어 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Transactional.html "Transactional (Spring Framework 7.0.7 API)"))  
하지만 표준 `JtaTransactionManager`는 per-transaction isolation을 기본 지원하지 않고, 기본 구현은 `ISOLATION_DEFAULT` 외의 isolation level에 대해 예외를 던질 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html "JtaTransactionManager (Spring Framework 7.0.7 API)"))

### 3.2 피해야 하는 예

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.SERIALIZABLE,
    rollbackFor = Exception.class
)
public void deleteTemplate(...) {
    dao.deleteTemplate(...);
    mDao.deleteEmsTemp(...);
}
```

JTA 환경에서는 위 설정이 기대대로 적용되지 않거나, `InvalidIsolationLevelException` 계열 예외로 기동/실행 문제가 발생할 수 있습니다. 표준 JTA에서 isolation을 메소드별로 바꾸는 설계는 피하는 것이 좋습니다.

### 3.3 Isolation을 꼭 제어해야 한다면

JTA 트랜잭션 메소드에서 isolation을 바꾸기보다 아래 방식이 현실적입니다.

|방법|설명|권장도|
|---|---|--:|
|DB 기본 isolation 설정|DB 또는 datasource 기본값으로 통일|상|
|SQL lock 사용|`SELECT ... FOR UPDATE` 등 업무 단위 잠금|상|
|낙관적 락|version column 사용|상|
|전용 로컬 트랜잭션 분리|강한 isolation이 필요한 업무를 단일 DB 로컬 트랜잭션으로 분리|중|
|JTA custom isolation 허용|`setAllowCustomIsolationLevels(true)` 등 특수 구성|하|

`JtaTransactionManager#setAllowCustomIsolationLevels`는 non-default isolation 지정 허용 여부를 설정할 수 있지만, 기본값은 `false`이며 non-default isolation 지정 시 예외를 던집니다. 이 옵션은 resource adapter가 thread-bound transaction context를 보고 isolation을 개별 적용하는 특수 환경에서만 검토해야 합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/jta/JtaTransactionManager.html "JtaTransactionManager (Spring Framework 7.0.7 API)"))

## 4. `dao`, `mDao` 구조 기준 권장 패턴

### 4.1 전체 삭제를 하나의 JTA 트랜잭션으로 묶는 경우

```java
@Service
public class TemplateService {
    private final TemplateDao dao;
    private final MailTemplateDao mDao;
    public TemplateService(TemplateDao dao, MailTemplateDao mDao) {
        this.dao = dao;
        this.mDao = mDao;
    }
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.DEFAULT,
        rollbackFor = Exception.class,
        timeout = 30
    )
    public void deleteTemplate(Map<String, Object> deleteMap,
                               Map<String, Object> loginUserInfo) throws Exception {
        deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
        dao.deleteReplcInfo(deleteMap);
        mDao.deleteEmsReplcInfo(deleteMap);
        dao.deleteRcvTgt(deleteMap);
        dao.deleteTemplate(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    }
}
```

전제:

```text
dao  → Main XA DataSource
mDao → Mail XA DataSource
Spring TransactionManager → JtaTransactionManager 단일 사용
```

### 4.2 실패 이력은 별도 commit하고 싶은 경우

```java
@Service
public class TemplateService {
    private final TemplateDeleteService templateDeleteService;
    private final DeleteFailLogService deleteFailLogService;
    public void deleteTemplate(Map<String, Object> deleteMap,
                               Map<String, Object> loginUserInfo) throws Exception {
        try {
            templateDeleteService.deleteTemplateByJta(deleteMap, loginUserInfo);
        } catch (Exception e) {
            deleteFailLogService.saveFailLog(deleteMap, e);
            throw e;
        }
    }
}
```

```java
@Service
public class TemplateDeleteService {
    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.DEFAULT,
        rollbackFor = Exception.class,
        timeout = 30
    )
    public void deleteTemplateByJta(Map<String, Object> deleteMap,
                                    Map<String, Object> loginUserInfo) throws Exception {
        deleteMap.put("UPDUSR_ID", loginUserInfo.get("USER_ID"));
        dao.deleteReplcInfo(deleteMap);
        mDao.deleteEmsReplcInfo(deleteMap);
        dao.deleteRcvTgt(deleteMap);
        dao.deleteTemplate(deleteMap);
        mDao.deleteEmsTemp(deleteMap);
    }
}
```

```java
@Service
public class DeleteFailLogService {
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.DEFAULT,
        rollbackFor = Exception.class,
        timeout = 10
    )
    public void saveFailLog(Map<String, Object> deleteMap, Exception e) {
        failLogDao.insertFailLog(deleteMap, e.getMessage());
    }
}
```

주의:

```text
REQUIRES_NEW는 실패 이력 보존에는 적합
하지만 업무 데이터 dao/mDao를 REQUIRES_NEW로 쪼개면 전체 원자성이 깨질 수 있음
```

## 5. 실무 주의점

|구분|주의점|
|---|---|
|TM 구성|`txManager`, `txManagerMail` 2개 로컬 TM이 아니라 `JtaTransactionManager` 하나로 통일|
|DataSource|일반 DBCP2 `BasicDataSource`가 아니라 WAS/컨테이너 XA DataSource 사용 권장|
|Propagation|업무 본 트랜잭션은 대부분 `REQUIRED` 사용|
|`REQUIRES_NEW`|감사 로그/실패 이력/보상 요청 저장에 제한적으로 사용|
|`NESTED`|JTA/XA 설계에서는 사용 전제 금지|
|Isolation|`DEFAULT` 권장, `READ_COMMITTED`, `SERIALIZABLE` 명시 지양|
|Timeout|반드시 설정 권장. 2PC는 장애 시 lock 유지 시간이 길어질 수 있음|
|예외 처리|catch 후 삼키면 commit될 수 있으므로 반드시 재throw|
|Checked Exception|`rollbackFor=Exception.class` 명시 권장|
|Self-invocation|같은 클래스 내부 메소드 호출은 Spring AOP 프록시를 거치지 않아 `@Transactional`이 적용되지 않을 수 있음|
|외부 호출|DB 트랜잭션 안에서 장시간 API/파일/메일 발송을 수행하지 말 것|
|Pool sizing|`REQUIRES_NEW` 사용 시 외부 트랜잭션 리소스 + 내부 트랜잭션 리소스가 동시에 필요하므로 pool 고갈 주의|

## 6. 권장/비권장 정리

|상황|권장 설정|판단|
|---|---|---|
|`dao`, `mDao` 동시 commit/rollback|`REQUIRED + DEFAULT`|가장 표준|
|실패 로그는 rollback되면 안 됨|로그 저장만 `REQUIRES_NEW + DEFAULT`|제한적 사용|
|조회만 수행|`SUPPORTS + readOnly=true`|가능|
|외부 API 호출|`NOT_SUPPORTED` 또는 트랜잭션 밖으로 분리|권장|
|부분 rollback 필요|JTA `NESTED` 대신 업무 보상/분리 설계|권장|
|메소드별 isolation 강제|JTA에서는 지양|비권장|
|강한 잠금 필요|isolation 변경보다 `SELECT FOR UPDATE`/version lock 검토|권장|

## 7. 최종 판단

JTA 방식에서 `Propagation`과 `Isolation`은 다음 기준으로 사용하면 됩니다.

```text
Propagation:
- 업무 본 트랜잭션: REQUIRED
- 실패 로그/감사 로그: REQUIRES_NEW
- 반드시 상위 트랜잭션 필요: MANDATORY
- 단순 조회: SUPPORTS
- 장시간 외부 작업: NOT_SUPPORTED
- NESTED는 JTA에서 지양
```

```text
Isolation:
- 기본은 DEFAULT
- JTA 메소드에 READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE 명시 지양
- 격리 수준 제어가 필요하면 DB 기본값, SQL lock, 낙관적 락으로 해결
```

현재 `deleteTemplate()` 같은 `dao + mDao` 동시 삭제 업무는 아래 설정이 가장 안전합니다.

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.DEFAULT,
    rollbackFor = Exception.class,
    timeout = 30
)
```

이 설정을 쓰되, 반드시 **XA DataSource 2개 + `JtaTransactionManager` 단일화 + 예외 재전파 + timeout 설정 + 장애 복구 테스트**까지 같이 검증해야 실무 적용이 가능합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
