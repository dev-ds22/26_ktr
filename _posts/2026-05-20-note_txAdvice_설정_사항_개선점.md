---
layout: single
title: "txAdvice_설정_사항_개선점"
excerpt: "txAdvice_설정_사항_개선점"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-20"
last_modified_at: "2026-05-20 17:24:04 +0900"
---
## 1. 원본

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">  
<tx:attributes>  
	<tx:method name="save*" propagation="REQUIRED" rollback-for="java.lang.Exception" />  
	<tx:method name="insert*" propagation="REQUIRED" rollback-for="java.lang.Exception" />  
	<tx:method name="update*" propagation="REQUIRED" rollback-for="java.lang.Exception" />  
	<tx:method name="delete*" propagation="REQUIRED" rollback-for="java.lang.Exception" />  
	<tx:method name= "*"   propagation="SUPPORTS" read-only="true" />  
</tx:attributes>  
</tx:advice>  
  
<aop:config>  
	<aop:pointcut id="serviceTx"  expression="execution(* bb.aaa..service..*Service.*(..))"/>  
	<aop:advisor advice-ref="txAdvice" pointcut-ref="serviceTx" />  
</aop:config>
```

## 2. Propagation 과 read-only
**서비스 메서드 명명 규칙을 완전히 준수하더라도 `SUPPORTS + read-only=true` 조합에는 피하기 어려운 구조적 한계**가 있습니다. Spring 5.3 문서 기준으로 `<tx:method>`의 `read-only`는 `REQUIRED` 또는 `REQUIRES_NEW` propagation에만 적용되는 속성으로 설명됩니다. 따라서 `SUPPORTS read-only=true`는 실무에서 기대보다 약하게 동작하거나 오해를 만들 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.30/reference/html/data-access.html "Data Access"))

## 3. REQUIRED와 SUPPORTS 차이

|                    구분 | REQUIRED          | SUPPORTS             |
| --------------------: | ----------------- | -------------------- |
|            기존 트랜잭션 있음 | 기존 트랜잭션 참여        | 기존 트랜잭션 참여           |
|            기존 트랜잭션 없음 | 새 트랜잭션 시작         | 트랜잭션 없이 실행           |
| DB Connection 트랜잭션 제어 | 수행                | 기존 트랜잭션 없으면 수행 안 함   |
|          read-only 적용 | 새 트랜잭션 시작 시 의미 있음 | 기존 트랜잭션 없으면 실질 효과 약함 |
|             쓰기 작업 적합성 | 적합                | 부적합                  |
|             단순 조회 적합성 | 안정성 높음            | 성능상 가벼울 수 있음         |

- Spring의 propagation 정의에서도 `REQUIRED`는 현재 트랜잭션을 지원하고 없으면 새로 생성하며, `SUPPORTS`는 현재 트랜잭션을 지원하지만 없으면 비트랜잭션으로 실행한다고 정의합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html "Propagation (Spring Framework 7.0.7 API)"))

### 3-1. REQUIRED와 SUPPORTS

`SUPPORTS`는 **조회용 기본값으로도 신중해야 하며, 범용 메서드 `*`에 무조건 적용하면 위험**합니다.  
특히 아래 설정은 겉으로는 “나머지는 조회 전용”처럼 보이지만, 실제로는 **트랜잭션을 새로 만들지 않기 때문에 read-only의 DB 적용 효과도 약하고, 서비스 단위 정합성도 약해질 수 있습니다.**

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

Spring 5.3 기준으로 `REQUIRED`는 기존 트랜잭션이 있으면 참여하고 없으면 물리 트랜잭션을 새로 만듭니다. 반면 `SUPPORTS`는 기존 트랜잭션이 있으면 참여하지만, 없으면 리소스 레벨에서 비트랜잭션 방식으로 동작할 수 있습니다. Spring 공식 문서도 `PROPAGATION_REQUIRED`는 일반적인 서비스 계층 호출에서 좋은 기본값이라고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))

### 3-2. REQUIRED와 SUPPORTS 차이

#### 3-2-1. Propagation 5개 비교
Spring 공식 API 기준으로 `REQUIRED`, `SUPPORTS`, `REQUIRES_NEW`, `NOT_SUPPORTED`, `NESTED`는 각각 트랜잭션 참여/생성/중단/중첩 방식이 다릅니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html "Propagation (Spring Framework 7.0.7 API)"))

|               구분 | REQUIRED           | REQUIRES_NEW                       | SUPPORTS                           | NOT_SUPPORTED                      | NESTED                                  |
| ---------------: | ------------------ | ---------------------------------- | ---------------------------------- | ---------------------------------- | --------------------------------------- |
|       기존 트랜잭션 있음 | 기존 트랜잭션 참여         | 기존 트랜잭션 일시 중단 후 새 트랜잭션 시작          | 기존 트랜잭션 참여                         | 기존 트랜잭션 일시 중단 후 비트랜잭션 실행           | 기존 트랜잭션 안에서 중첩 트랜잭션 실행                  |
|       기존 트랜잭션 없음 | 새 트랜잭션 시작          | 새 트랜잭션 시작                          | 트랜잭션 없이 실행                         | 트랜잭션 없이 실행                         | `REQUIRED`처럼 새 트랜잭션 시작                  |
|          물리 트랜잭션 | 기존 또는 신규 1개 사용     | 항상 독립 물리 트랜잭션 사용                   | 기존 트랜잭션 없으면 물리 트랜잭션 없음             | 항상 물리 트랜잭션 없이 실행                   | 하나의 물리 트랜잭션 안에서 savepoint 사용            |
|       외부 트랜잭션 영향 | 내부 실패 시 전체 롤백 가능   | 내부 트랜잭션은 외부와 독립 commit/rollback 가능 | 기존 트랜잭션 있으면 외부에 영향                 | 외부 트랜잭션과 분리되어 영향 적음                | 내부 rollback을 savepoint까지 되돌리고 외부는 계속 가능 |
|       서비스 단위 원자성 | 보장 가능              | 독립 단위로 보장 가능                       | 기존 트랜잭션 없으면 보장 약함                  | 보장하지 않음                            | 부분 rollback 구조 가능                       |
| 예외 발생 시 rollback | 가능                 | 새 트랜잭션만 rollback 가능                | 기존 트랜잭션 없으면 rollback 대상 없음         | rollback 대상 트랜잭션 없음                | 중첩 범위만 rollback 가능                      |
|     read-only 적용 | 실제 트랜잭션에 적용 가능     | 새 독립 트랜잭션에 적용 가능                   | 기존 트랜잭션 없으면 DB read-only 적용 기대 어려움 | 트랜잭션이 없으므로 read-only 의미 약함         | savepoint 기반 트랜잭션 범위에서 의미 가능            |
|           조회 일관성 | 트랜잭션 범위에서 확보 가능    | 독립 트랜잭션 범위에서 확보 가능                 | 호출 시점/커넥션 사용 방식에 따라 약함             | 조회 일관성 보장 약함                       | 외부 트랜잭션 스냅샷 안에서 부분 rollback 가능          |
|           커넥션 사용 | 보통 1개              | 외부 트랜잭션 중 호출 시 추가 커넥션 필요 가능        | 기존 트랜잭션 없으면 일반 커넥션 사용              | 외부 트랜잭션 중단 후 별도 비트랜잭션 리소스 사용 가능    | 보통 같은 JDBC 커넥션의 savepoint 사용            |
|            대표 용도 | 대부분의 서비스 쓰기/조회 기본값 | 감사 로그, 이력 저장, 외부 트랜잭션과 독립 커밋 필요 시  | 단순 조회, 트랜잭션 필수 아님                  | 외부 API 호출, 긴 파일 처리, 트랜잭션 밖 실행 필요 시 | 일부 단계 실패만 되돌리고 전체 작업은 유지할 때             |
|           범용 기본값 | 비교적 안전             | 비추천                                | 위험                                 | 비추천                                | 특수 목적                                   |

#### 3-2-2. 각 Propagation의 실무 해석

##### 3-2-2-1. REQUIRED

```xml
<tx:method name="save*" propagation="REQUIRED" rollback-for="java.lang.Exception"/>
```

가장 일반적인 서비스 계층 기본값입니다. 기존 트랜잭션이 있으면 참여하고, 없으면 새 트랜잭션을 시작하므로 메서드 단위 원자성, rollback, read-only 적용을 비교적 명확하게 기대할 수 있습니다. Spring 문서도 `PROPAGATION_REQUIRED`를 일반적인 트랜잭션 전파 방식으로 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.30/reference/html/data-access.html "Data Access"))

##### 3-2-2-2. REQUIRES_NEW

```xml
<tx:method name="saveLog*" propagation="REQUIRES_NEW" rollback-for="java.lang.Exception"/>
```

항상 새 트랜잭션을 시작하고 기존 트랜잭션이 있으면 중단합니다. 따라서 외부 트랜잭션이 rollback되어도 내부 `REQUIRES_NEW` 트랜잭션은 이미 commit될 수 있습니다. Spring 공식 문서는 `REQUIRES_NEW`가 독립 물리 트랜잭션을 사용하며, 외부 트랜잭션과 독립적으로 commit/rollback될 수 있다고 설명합니다. 단, 외부 트랜잭션의 커넥션이 묶인 상태에서 내부 트랜잭션이 새 커넥션을 요구할 수 있으므로 커넥션 풀 고갈 위험이 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))

##### 3-2-2-3. SUPPORTS

```xml
<tx:method name="select*" propagation="SUPPORTS" read-only="true"/>
```

기존 트랜잭션이 있으면 참여하지만 없으면 트랜잭션 없이 실행합니다. 그래서 단순 조회에는 사용할 수 있지만, 범용 `*` 기본값으로 두면 위험합니다. 특히 기존 트랜잭션이 없을 때는 rollback 대상 트랜잭션이 없고, `read-only=true`도 DB 트랜잭션 레벨로 적용된다고 보기 어렵습니다. Spring API도 `SUPPORTS`를 “현재 트랜잭션을 지원하되 없으면 비트랜잭션으로 실행”한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html "Propagation (Spring Framework 7.0.7 API)"))

##### 3-2-2-4. NOT_SUPPORTED

```xml
<tx:method name="callExternal*" propagation="NOT_SUPPORTED"/>
```

항상 트랜잭션 없이 실행하고, 기존 트랜잭션이 있으면 일시 중단합니다. 외부 API 호출, 파일 처리, 오래 걸리는 비DB 작업처럼 DB 트랜잭션을 길게 잡고 있으면 안 되는 구간에 제한적으로 사용합니다. Spring API도 `NOT_SUPPORTED`를 비트랜잭션 실행 및 기존 트랜잭션 중단으로 정의합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/annotation/Propagation.html "Propagation (Spring Framework 7.0.7 API)"))

##### 3-2-2-5. NESTED

```xml
<tx:method name="partialProcess*" propagation="NESTED" rollback-for="java.lang.Exception"/>
```

기존 트랜잭션이 있으면 중첩 트랜잭션처럼 동작하고, 없으면 `REQUIRED`처럼 새 트랜잭션을 시작합니다. Spring 공식 문서는 `NESTED`가 하나의 물리 트랜잭션 안에서 여러 savepoint를 사용하며, 내부 범위만 savepoint까지 rollback할 수 있다고 설명합니다. 단, 이 방식은 주로 JDBC `DataSourceTransactionManager`의 savepoint 기반 동작에 적합합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))

#### 3-2-3. 실무 권장 기준

|상황|권장 Propagation|이유|
|--:|---|---|
|일반 저장/수정/삭제|`REQUIRED`|서비스 단위 원자성 확보|
|일반 복합 조회|`REQUIRED read-only=true`|여러 SELECT의 트랜잭션 일관성 확보 가능|
|단순 단건 조회|`SUPPORTS read-only=true` 가능|트랜잭션 생성 비용을 줄일 수 있으나 read-only 강제 효과는 약함|
|감사 로그/이력 독립 저장|`REQUIRES_NEW`|본 트랜잭션 rollback과 무관하게 기록 가능|
|외부 API/파일 처리|`NOT_SUPPORTED`|DB 트랜잭션 점유 시간 감소|
|일부 단계만 되돌리기|`NESTED`|savepoint 기반 부분 rollback 가능|

#### 3-2-4. 참고

범용 기본값으로는 **`REQUIRED`가 가장 안전**합니다. `SUPPORTS`는 “있으면 참여, 없으면 비트랜잭션”이므로 범용 `*`에 사용하면 rollback, read-only, 조회 일관성이 모두 약해질 수 있습니다. `REQUIRES_NEW`는 독립 commit이 필요한 로그/이력에 유용하지만 커넥션 풀 고갈 위험이 있고, `NOT_SUPPORTED`는 트랜잭션 밖에서 오래 걸리는 작업을 수행할 때, `NESTED`는 JDBC savepoint 기반 부분 rollback이 필요한 경우에만 제한적으로 사용하는 것이 좋습니다.

## 4. REQUIRED 동작 상세

`REQUIRED`는 서비스 계층에서 가장 일반적인 기본 전파 방식입니다.

```xml
<tx:method name="update*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
```

동작은 다음과 같습니다.

```text
외부 트랜잭션 있음 → 그 트랜잭션에 참여
외부 트랜잭션 없음 → 새 트랜잭션 시작
```

예:

```java
public void updateOrder() {
	orderDao.updateOrder();
	orderHistoryDao.insertHistory();
}
```

`REQUIRED`이면 `updateOrder()` 전체가 하나의 트랜잭션 범위가 됩니다.

```text
order update 성공
history insert 실패
→ 전체 rollback 가능
```

Spring 문서에 따르면 `PROPAGATION_REQUIRED`는 물리 트랜잭션을 강제하며, 외부 트랜잭션이 없으면 현재 scope에 대해 새 트랜잭션을 만들고, 외부 트랜잭션이 있으면 그 트랜잭션에 참여합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))

## 5. SUPPORTS 동작 상세

`SUPPORTS`는 이름 그대로 “있으면 지원”입니다.

```xml
<tx:method name="select*" propagation="SUPPORTS" read-only="true" />
```

동작은 다음과 같습니다.

```text
외부 트랜잭션 있음 → 그 트랜잭션에 참여
외부 트랜잭션 없음 → 새 트랜잭션을 만들지 않음
```

즉, `SUPPORTS`는 **항상 트랜잭션이 있는 조회**가 아닙니다.  
예:

```java
public User selectUser(String userId) {
	User user = userDao.selectUser(userId);
	List<Role> roles = roleDao.selectRoles(userId);
	return merge(user, roles);
}
```

이 메서드가 단독 호출되면 `SUPPORTS`는 트랜잭션을 만들지 않습니다.  
따라서 `userDao.selectUser()`와 `roleDao.selectRoles()`가 반드시 하나의 트랜잭션 스냅샷 안에서 수행된다고 보기 어렵습니다.  
MariaDB InnoDB의 기본 격리 수준은 `REPEATABLE READ`이고, 하나의 트랜잭션 안에서는 첫 consistent read 기준 스냅샷을 공유합니다. 그러나 `SUPPORTS`로 트랜잭션을 만들지 않으면 이런 서비스 메서드 단위의 일관성을 기대하기 어렵습니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction "SET TRANSACTION | Server | MariaDB Documentation"))

## 6. SUPPORTS를 범용으로 쓰면 위험한 이유

### 6-1. 메서드 실행 결과가 호출 위치에 따라 달라짐

같은 메서드라도 호출자가 트랜잭션을 가지고 있는지에 따라 동작이 달라집니다.

```java
public List<User> selectUsers() {
	return userDao.selectUsers();
}
```

호출 1:

```java
userService.selectUsers();
```

```text
외부 트랜잭션 없음
→ 트랜잭션 없이 실행
```

호출 2:

```java
@Transactional
public void process() {
	userService.selectUsers();
}
```

```text
외부 트랜잭션 있음
→ 기존 트랜잭션에 참여
```

즉, `SUPPORTS`는 **메서드 자체의 트랜잭션 정책이 독립적으로 고정되지 않습니다.**  
이 점 때문에 운영 장애 분석 시 “이 메서드는 항상 read-only 조회로 돈다”고 단정하기 어렵습니다.

### 6-2. 쓰기 작업이 누락되면 원자성이 깨짐

예를 들어 명명 규칙상 쓰기 메서드는 `save*`, `insert*`, `update*`, `delete*`만 사용한다고 해도 실무에서는 다음과 같은 메서드가 생깁니다.

```text
processOrder()
approveUser()
cancelOrder()
syncData()
executeBatch()
modifyStatus()
```

이들이 `* SUPPORTS read-only=true`에 걸리면 기존 트랜잭션이 없는 경우 트랜잭션 없이 실행될 수 있습니다.

```java
public void processOrder() {
	orderDao.updateOrder();
	paymentDao.insertPaymentHistory();
}
```

`processOrder()`가 `SUPPORTS`로 실행되면:

```text
order update 성공
payment history insert 실패
→ 앞의 update만 반영될 수 있음
```

이것이 `SUPPORTS`를 범용 fallback으로 쓰면 위험한 가장 큰 이유입니다.

### 6-3. rollback-for가 사실상 의미 없어질 수 있음

`rollback-for`는 롤백할 **트랜잭션이 있을 때** 의미가 있습니다.

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

이 상태에서 기존 트랜잭션 없이 DML이 실행되면, Spring이 시작한 트랜잭션이 없기 때문에 메서드 전체 rollback을 기대하기 어렵습니다.  
즉, `SUPPORTS`는 아래와 같은 위험을 만듭니다.

```text
트랜잭션이 있으면 rollback 가능
트랜잭션이 없으면 각 SQL/커넥션/autocommit 동작에 의존
```

### 6-4. read-only가 DB 쓰기 차단 장치로 동작하지 않을 수 있음

Spring의 read-only는 기본적으로 “힌트”입니다. Spring 5.3 `TransactionDefinition` 문서도 read-only flag는 트랜잭션 서브시스템에 대한 힌트이며, 쓰기 접근 실패를 반드시 유발하지 않는다고 설명합니다. 또한 `SUPPORTS`처럼 리소스 레벨에서 비트랜잭션으로 동작하는 경우에는 Hibernate `Session` 같은 애플리케이션 관리 리소스에만 적용될 수 있다고 설명합니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/transaction/TransactionDefinition.html "TransactionDefinition (Spring Framework 5.3.46 API)"))  
따라서 아래 설정은:

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

다음 의미로 봐야 합니다.

```text
기존 트랜잭션이 있으면 read-only 속성을 참고할 수 있다.
기존 트랜잭션이 없으면 DB에 read-only transaction을 시작하지 않는다.
쓰기 SQL을 반드시 막는 설정은 아니다.
```

### 6-5. 복합 조회의 일관성 보장이 약함

조회도 단순 단건 SELECT라면 문제가 작을 수 있습니다.  
하지만 다음처럼 여러 DAO를 조합하면 다릅니다.

```java
public OrderDetail selectOrderDetail(String orderNo) {
	Order order = orderDao.selectOrder(orderNo);
	List<Item> items = itemDao.selectItems(orderNo);
	List<Coupon> coupons = couponDao.selectCoupons(orderNo);
	return new OrderDetail(order, items, coupons);
}
```

`REQUIRED read-only=true`이면 메서드 전체가 하나의 읽기 트랜잭션으로 묶일 수 있습니다.  
반면 `SUPPORTS read-only=true`는 외부 트랜잭션이 없으면 새 트랜잭션을 만들지 않으므로, 각 SELECT 사이의 데이터 변경에 대해 서비스 단위 일관성이 약해질 수 있습니다.  
MariaDB InnoDB `REPEATABLE READ`에서는 하나의 트랜잭션 안의 consistent read들이 첫 read 기준 스냅샷을 공유하지만, 그 전제는 “하나의 트랜잭션 안”입니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction "SET TRANSACTION | Server | MariaDB Documentation"))

## 7. read-only는 실제 동작하는가?

결론부터 말하면 **일부는 동작하지만, 기대한 만큼 강제되지는 않을 수 있습니다.**  
Spring의 read-only 동작은 크게 3단계로 나누어 봐야 합니다.

|단계|동작|설명|
|--:|---|---|
|Spring 트랜잭션 속성|동작|`TransactionDefinition.isReadOnly()`로 전달됨|
|JDBC 기본 처리|일부 동작|보통 `Connection.setReadOnly(true)` 힌트 적용|
|DB 강제 read-only|기본은 약함|`DataSourceTransactionManager.enforceReadOnly=true` 필요|
|Spring `DataSourceTransactionManager`는 `enforceReadOnly=true` 설정 시 트랜잭션 커넥션에 `SET TRANSACTION READ ONLY`를 명시적으로 실행할 수 있습니다. 이 방식은 기본 JDBC `Connection.setReadOnly(true)` 힌트보다 강하며, MySQL 계열에서도 이해되는 구문으로 설명됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html "DataSourceTransactionManager (Spring Framework 7.0.7 API)"))|||

## 8. read-only가 실제로 안 먹는 것처럼 보이는 이유

### 8-1. read-only는 기본적으로 힌트라서

Spring 공식 API 문서에 따르면 read-only는 트랜잭션 최적화를 위한 힌트이며, 실제 트랜잭션 매니저가 해석하지 못하더라도 예외를 던지지 않고 무시할 수 있습니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/transaction/TransactionDefinition.html "TransactionDefinition (Spring Framework 5.3.46 API)"))  
즉, 아래 설정만으로:

```xml
<tx:method name="select*" propagation="REQUIRED" read-only="true" />
```

항상 아래가 보장되는 것은 아닙니다.

```text
UPDATE 실행 시 반드시 DB 에러 발생
```

### 8-2. SUPPORTS는 새 트랜잭션을 만들지 않기 때문에

`SUPPORTS read-only=true`는 외부 트랜잭션이 없으면 실제 DB 트랜잭션을 시작하지 않습니다.  
따라서 MariaDB에 `SET TRANSACTION READ ONLY` 같은 구문을 적용할 기회 자체가 없습니다.  
이 때문에 `SUPPORTS + read-only=true`는 다음 용도로 부적합합니다.

```text
DB 쓰기 차단
조회 트랜잭션 보장
DB read-only 최적화 기대
복합 조회 일관성 보장
```

### 8-3. 외부 트랜잭션에 참여하면 내부 read-only 선언이 무시될 수 있음

`REQUIRED`도 기존 트랜잭션에 참여할 때는 내부 메서드의 read-only, isolation, timeout 선언이 외부 트랜잭션 속성에 흡수될 수 있습니다. Spring 문서도 기본적으로 참여 트랜잭션은 외부 scope의 특성에 합류하며, local isolation/timeout/read-only 선언이 조용히 무시될 수 있다고 설명합니다. 이 불일치를 잡으려면 `validateExistingTransactions` 설정을 고려해야 합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))

## 9. MariaDB 10.6.15에서 read-only 관점

MariaDB는 트랜잭션 접근 모드로 `READ WRITE`와 `READ ONLY`를 지원합니다. MySQL 문서 기준으로 `READ ONLY` 모드에서는 테이블 변경이 금지되고, 스토리지 엔진이 쓰기가 허용되지 않는 상황에서 가능한 최적화를 할 수 있다고 설명합니다. MariaDB도 `SET TRANSACTION` 문서에서 트랜잭션 격리 수준과 접근 모드를 다룹니다. ([MySQL](https://dev.mysql.com/doc/en/set-transaction.html "MySQL :: MySQL 9.7 Reference Manual :: 15.3.7 SET TRANSACTION Statement"))  
다만 Spring에서 MariaDB에 이 효과를 기대하려면 조건이 있습니다.

```text
1. 실제 트랜잭션이 시작되어야 함
2. read-only=true가 해당 트랜잭션에 적용되어야 함
3. DataSourceTransactionManager에서 enforceReadOnly=true 설정을 검토해야 함
```

권장 설정 예:

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
	<property name="enforceReadOnly" value="true"/>
</bean>
```

조회 메서드:

```xml
<tx:method name="select*" propagation="REQUIRED" read-only="true" />
<tx:method name="find*" propagation="REQUIRED" read-only="true" />
<tx:method name="get*" propagation="REQUIRED" read-only="true" />
```

이 조합이어야 “Spring read-only”가 MariaDB의 read-only transaction에 가까워집니다.

## 10. read-only로 DB 부하 회피가 가능한가?

가능성은 있지만, **일반적인 성능 개선 수단으로 과대평가하면 안 됩니다.**

|항목|read-only 효과|판단|
|--:|---|---|
|DML 방지|`enforceReadOnly=true`면 기대 가능|안정성 효과 큼|
|불필요한 flush 방지|JPA/Hibernate에서 효과 큼|MyBatis/JDBC만 쓰면 제한적|
|DB 락 감소|일반 SELECT 중심이면 애초에 락이 작음|효과 제한적|
|스냅샷 일관성|`REQUIRED read-only` 트랜잭션이면 가능|복합 조회에 유리|
|CPU/IO 절감|쿼리 자체가 무거우면 거의 해결 안 됨|인덱스/SQL 튜닝이 우선|
|Connection 부하 감소|read-only 자체로 해결 안 됨|커넥션 사용 시간/쿼리 수가 핵심|
|즉, read-only의 주효과는 보통 아래에 가깝습니다.|||

```text
쓰기 방지 의도 명확화
ORM flush 최소화
조회 트랜잭션 의도 표현
복합 조회 일관성 확보
실수로 DML 수행하는 코드 조기 발견
```

반대로 아래를 기대하면 안 됩니다.

```text
느린 SELECT가 자동으로 빨라짐
DB CPU 부하가 크게 감소함
커넥션 사용량이 자동으로 줄어듦
락 문제가 자동으로 해결됨
```

DB 부하를 줄이려면 read-only보다 다음이 더 직접적입니다.

```text
SQL 실행 계획 개선
인덱스 추가/정리
페이징 처리
N+1 쿼리 제거
불필요한 반복 조회 제거
캐시 적용
조회 전용 Replica 분리
Connection Pool 적정화
```

## 11. read-only 사용 장점

### 11-1. 코드 의도가 명확해짐

```xml
<tx:method name="select*" propagation="REQUIRED" read-only="true" />
```

이 설정은 “이 메서드는 조회용”이라는 아키텍처 규칙을 명확히 보여줍니다.

### 11-2. ORM 사용 시 flush 비용 감소 가능

JPA/Hibernate 환경에서는 read-only 트랜잭션이 flush mode와 연계되어 변경 감지/flush 비용을 줄이는 데 도움이 될 수 있습니다. Spring 문서는 read-only flag가 Hibernate `Session` 같은 애플리케이션 관리 리소스에 적용될 수 있다고 설명합니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/transaction/TransactionDefinition.html "TransactionDefinition (Spring Framework 5.3.46 API)"))

### 11-3. DB 레벨 강제와 결합하면 실수 방지 가능

`DataSourceTransactionManager.enforceReadOnly=true`를 사용하면 `SET TRANSACTION READ ONLY`를 실행해 DML을 더 강하게 차단할 수 있습니다. Spring 문서상 이 방식은 일반적인 `Connection.setReadOnly(true)` 힌트보다 강하며, 데이터 조작문을 엄격히 금지하는 모드로 설명됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html "DataSourceTransactionManager (Spring Framework 7.0.7 API)"))

### 11-4. 복합 조회 일관성 확보 가능

`REQUIRED read-only=true`로 실제 트랜잭션을 만들면 여러 SELECT가 하나의 트랜잭션 안에서 수행될 수 있습니다.  
MariaDB InnoDB의 `REPEATABLE READ`에서는 하나의 트랜잭션 안의 consistent read들이 첫 read 기준 스냅샷을 공유합니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction "SET TRANSACTION | Server | MariaDB Documentation"))

## 12. read-only 사용 단점

### 12-1. 무조건 성능이 좋아지는 설정은 아님

MyBatis/JDBC 단순 조회에서는 read-only 설정만으로 체감 성능이 크게 좋아진다고 단정하기 어렵습니다. 느린 조회의 대부분은 실행 계획, 인덱스, 데이터량, 조인 구조, 네트워크 왕복, 애플리케이션 반복 호출 문제에서 발생합니다.

### 12-2. 잘못 쓰면 쓰기 로직을 숨김

아래처럼 범용 메서드에 read-only를 걸면 위험합니다.

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

이 설정은 개발자에게 “나머지는 조회용”이라는 착시를 줍니다. 하지만 실제로는 다음 문제가 남습니다.

```text
트랜잭션이 없을 수 있음
DB read-only가 적용되지 않을 수 있음
쓰기 SQL이 실행될 수 있음
rollback이 안 될 수 있음
```

### 12-3. 외부 트랜잭션과 속성 충돌 가능

외부 read-write 트랜잭션 안에서 내부 read-only 메서드를 호출하면 내부 read-only 속성이 기대대로 적용되지 않을 수 있습니다. 반대로 외부 read-only 트랜잭션 안에서 내부 쓰기 메서드가 `REQUIRED`로 참여하면 정책 충돌이 생길 수 있습니다. 이 경우 기본 설정에서는 조용히 흡수될 수 있어 `validateExistingTransaction` 설정 검토가 필요합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html "Transaction Propagation :: Spring Framework"))

## 13. 범용 메서드 `*`에 read-only를 쓰면 위험한 이유

위험한 설정:

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

### 13-1. 문제 1. 명명 규칙 누락 시 쓰기 메서드가 비트랜잭션으로 실행

```java
public void approveOrder() {
	orderDao.updateStatus();
	orderDao.insertApprovalHistory();
}
```

`approve*`가 별도 규칙에 없으면 `SUPPORTS read-only=true`로 빠집니다.

### 13-2. 문제 2. read-only가 쓰기 차단을 보장하지 않음

Spring read-only는 기본적으로 힌트이며, 트랜잭션 매니저가 해석하지 못하면 무시될 수 있습니다. ([docs.enterprise.spring.io](https://docs.enterprise.spring.io/spring-framework/docs/5.3.46/javadoc-api/org/springframework/transaction/TransactionDefinition.html "TransactionDefinition (Spring Framework 5.3.46 API)"))

### 13-3. 문제 3. 트랜잭션이 없으면 rollback도 없음

`SUPPORTS`는 기존 트랜잭션이 없으면 새 트랜잭션을 만들지 않습니다. 따라서 중간 실패 시 메서드 전체 rollback이 아니라 SQL별 처리 결과에 의존할 수 있습니다.

### 13-4. 문제 4. 운영 장애 분석이 어려움

같은 메서드가 어떤 호출 경로에서는 트랜잭션 참여, 다른 호출 경로에서는 비트랜잭션으로 실행됩니다.

```text
Controller → selectA() : 비트랜잭션
Batch REQUIRED → selectA() : 기존 트랜잭션 참여
다른 Service REQUIRED → selectA() : 기존 트랜잭션 참여
```

동일 코드라도 실행 맥락에 따라 커넥션/트랜잭션/read-only 적용이 달라집니다.

## 14. 실무 권장 기준

### 14-1. 안정성 우선 권장안

```xml
<tx:attributes>
	<!-- 조회 메서드: 실제 read-only 트랜잭션 생성 -->
	<tx:method name="get*" propagation="REQUIRED" read-only="true" />
	<tx:method name="find*" propagation="REQUIRED" read-only="true" />
	<tx:method name="select*" propagation="REQUIRED" read-only="true" />
	<tx:method name="search*" propagation="REQUIRED" read-only="true" />
	<tx:method name="count*" propagation="REQUIRED" read-only="true" />
	<tx:method name="exists*" propagation="REQUIRED" read-only="true" />

	<!-- 쓰기 메서드 -->
	<tx:method name="save*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<tx:method name="insert*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<tx:method name="update*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<tx:method name="delete*" propagation="REQUIRED" rollback-for="java.lang.Exception" />

	<!-- 미분류 메서드는 안전하게 쓰기 가능 트랜잭션 -->
	<tx:method name="*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
</tx:attributes>
```

### 14-2. DB read-only 강제 검토

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
	<property name="enforceReadOnly" value="true"/>
	<property name="validateExistingTransaction" value="true"/>
</bean>
```

`enforceReadOnly=true`는 read-only 트랜잭션에 대해 `SET TRANSACTION READ ONLY`를 실행하는 방향이고, `validateExistingTransaction=true`는 기존 트랜잭션 참여 시 read-only/isolation 불일치를 더 엄격히 잡는 용도로 검토할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html "DataSourceTransactionManager (Spring Framework 7.0.7 API)"))

## 15. 언제 SUPPORTS를 써도 되는가

`SUPPORTS`는 완전히 잘못된 설정은 아닙니다. 다만 범용 기본값보다는 제한적으로 써야 합니다.  
적합한 경우:

```text
단건 조회
트랜잭션 스냅샷 일관성이 중요하지 않음
내부에서 DML을 절대 수행하지 않음
DB read-only 강제 효과를 기대하지 않음
성능상 트랜잭션 시작 비용을 아주 민감하게 보는 조회
```

부적합한 경우:

```text
복합 조회
여러 DAO를 조합하는 조회
조회 중 계산/검증 결과가 정합성에 중요함
쓰기 가능성이 조금이라도 있는 메서드
범용 fallback 메서드
운영 장애 시 재현성과 추적성이 중요한 서비스
```

## 16. 최종 정리

`REQUIRED`는 **서비스 메서드 단위의 트랜잭션 경계**를 만들어 원자성, rollback, 조회 일관성, read-only 적용 가능성을 확보하는 방식입니다. `SUPPORTS`는 **이미 트랜잭션이 있으면 참여하지만 없으면 만들지 않는 방식**이므로, 범용으로 쓰면 메서드 동작이 호출 경로에 따라 달라지고 rollback/read-only/조회 일관성이 약해집니다. `read-only=true`는 기본적으로 힌트이며, Spring JDBC + MariaDB 환경에서 DB 레벨 쓰기 차단까지 기대하려면 `REQUIRED read-only=true`와 `DataSourceTransactionManager.enforceReadOnly=true` 조합을 검토해야 합니다. DB 부하 감소 목적만으로 read-only를 과신하기보다는, **조회 의도 명확화, DML 실수 방지, 복합 조회 일관성 확보** 목적으로 사용하는 것이 실무적으로 더 정확합니다.


## 17. 현재 설정에서 피할 수 없는 문제점

### 17-1. `SUPPORTS + read-only=true`는 조회 트랜잭션을 보장하지 않음

현재 fallback 설정:

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

이 설정은 `get*`, `find*`, `select*`, `search*` 같은 조회 메서드가 기존 트랜잭션 없이 호출될 경우 **트랜잭션 없이 실행**될 수 있습니다.  
예:

```java
public List<User> getUsers() {
	List<User> users = userDao.selectUsers();
	List<Role> roles = roleDao.selectRoles();
	return merge(users, roles);
}
```

이 메서드가 단독 호출되면 `SUPPORTS`는 새 트랜잭션을 만들지 않습니다. 따라서 두 번의 SELECT가 하나의 트랜잭션 스냅샷 안에서 실행된다고 보장하기 어렵습니다. MariaDB InnoDB의 기본 격리 수준은 `REPEATABLE READ`이고, 하나의 트랜잭션 안에서는 첫 consistent read 기준 스냅샷을 공유하지만, 트랜잭션이 없으면 이런 서비스 단위 일관성을 기대하기 어렵습니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction "SET TRANSACTION | Server | MariaDB Documentation"))

### 17-2. `read-only=true`가 쓰기 SQL 차단 장치가 아님

`read-only=true`는 Spring 설정상 읽기 전용 트랜잭션 힌트입니다. 특히 `SUPPORTS`에서는 기존 트랜잭션이 없으면 새 트랜잭션을 만들지 않으므로 DB에 명확한 read-only transaction mode가 적용되지 않을 수 있습니다. Spring `DataSourceTransactionManager`는 기본적으로 JDBC `Connection.setReadOnly(true)` 성격의 힌트를 적용하고, 더 강한 DB 레벨 강제를 원할 때 `enforceReadOnly=true`를 통해 `SET TRANSACTION READ ONLY`를 실행할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html "DataSourceTransactionManager (Spring Framework 7.0.7 API)"))  
즉 아래처럼 이해하면 안 됩니다.

```text
SUPPORTS + read-only=true 이므로 update/delete가 반드시 DB에서 차단된다
```

정확히는 다음에 가깝습니다.

```text
트랜잭션이 새로 시작되는 read-only REQUIRED/REQUIRES_NEW 상황에서 의미가 크고,
SUPPORTS 단독 호출에서는 read-only 효과를 기대하기 어렵다.
```

### 17-3. 조회 메서드 내부에서 실수로 DML이 호출되어도 구조적으로 막기 어려움

명명 규칙을 지켜도 아래 같은 코드는 발생할 수 있습니다.

```java
public User getUser(String userId) {
	userDao.updateLastAccessTime(userId);
	return userDao.selectUser(userId);
}
```

메서드명은 `getUser`이므로 현재 설정에서는 `SUPPORTS + read-only=true`입니다. 그런데 내부에서 `updateLastAccessTime()` 같은 DML이 실행될 수 있습니다. 기존 트랜잭션이 없으면 비트랜잭션 또는 autocommit 기반으로 update가 실행될 수 있고, `read-only=true`만으로는 이를 안정적으로 차단한다고 보기 어렵습니다.

### 17-4. 외부 read-only 트랜잭션 안에서 쓰기 메서드를 호출하는 문제

예:

```java
public User getAndSaveSomething() {
	saveUser(...);
	return getUser(...);
}
```

또는 다른 서비스에서:

```java
public void getUserAndUpdate() {
	userService.getUser(...);
	userService.updateUser(...);
}
```

Spring의 `REQUIRED`는 기존 트랜잭션이 있으면 같은 물리 트랜잭션에 참여합니다. 기본 설정에서는 내부 메서드의 isolation, timeout, read-only 선언이 외부 트랜잭션 속성과 다르더라도 조용히 무시될 수 있고, 이를 엄격하게 검증하려면 `validateExistingTransactions=true` 설정을 고려해야 합니다. Spring 문서도 기존 트랜잭션 참여 시 내부 선언의 read-only flag가 기본적으로 무시될 수 있으며, 엄격 모드에서는 read-only mismatch를 거부할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.30/reference/html/data-access.html "Data Access"))

### 17-5. 같은 클래스 내부 호출은 AOP 트랜잭션이 적용되지 않을 수 있음

예:

```java
public void processUser() {
	saveUser(); // 같은 클래스 내부 호출
}

public void saveUser() {
	userDao.insertUser(...);
}
```

XML AOP 기반 선언적 트랜잭션은 일반적으로 프록시를 통한 외부 호출에 적용됩니다. 같은 클래스 내부에서 `this.saveUser()` 형태로 호출되면 프록시를 거치지 않아 `save* REQUIRED`가 기대대로 적용되지 않을 수 있습니다. 이 문제는 propagation/read-only 설정만으로 해결되지 않습니다.

## 18. MariaDB 10.6.15에서 read-only 관련 동작

MariaDB는 `SET TRANSACTION` 문으로 transaction isolation level과 access mode를 설정할 수 있고, access mode에는 `READ WRITE`, `READ ONLY`가 포함됩니다. `SESSION` 또는 `GLOBAL` 없이 사용하면 다음 트랜잭션에만 적용되고 이후에는 세션 값으로 되돌아갑니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/administrative-sql-statements/set-commands/set-transaction "SET TRANSACTION | Server | MariaDB Documentation"))  
Spring `DataSourceTransactionManager`에서 `enforceReadOnly=true`를 사용하면 read-only 트랜잭션 시작 시 `SET TRANSACTION READ ONLY`를 실행할 수 있습니다. Spring 문서상 이 방식은 단순 JDBC `Connection.setReadOnly(true)` 힌트보다 강하며, MySQL 계열에서도 이해되는 SQL로 설명됩니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html "DataSourceTransactionManager (Spring Framework 7.0.7 API)"))  
하지만 현재 설정의 핵심 문제는 이것입니다.

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

`SUPPORTS`는 기존 트랜잭션이 없으면 새 트랜잭션을 만들지 않습니다. 따라서 MariaDB의 `SET TRANSACTION READ ONLY`를 기대하려면 조회 메서드도 `REQUIRED read-only=true`로 트랜잭션을 시작해야 합니다.

## 19. 현재 설정의 실무 적합성

|항목|현재 설정|실무 판단|
|--:|---|---|
|쓰기 메서드|`save/insert/update/delete REQUIRED`|대체로 적합|
|조회 메서드|`* SUPPORTS read-only=true`|단순 조회만 있으면 가능|
|복합 조회|`SUPPORTS`|일관성 보장 약함|
|read-only 강제|`SUPPORTS read-only`|실효성 약함|
|DML 차단|read-only에 의존|위험|
|명명 규칙 의존|높음|코드 리뷰/테스트 없이는 한계|
|장애 예방|보통|fallback을 REQUIRED로 두는 편이 안전|

## 20. 개선안 1: 조회 메서드를 명시하고 `REQUIRED read-only=true` 사용

서비스 계층에서 조회 일관성과 read-only 의미를 살리고 싶다면 아래가 더 적합합니다.

```xml
<tx:attributes>
	<!-- 조회성 메서드 -->
	<tx:method name="get*" propagation="REQUIRED" read-only="true" />
	<tx:method name="find*" propagation="REQUIRED" read-only="true" />
	<tx:method name="select*" propagation="REQUIRED" read-only="true" />
	<tx:method name="search*" propagation="REQUIRED" read-only="true" />
	<tx:method name="count*" propagation="REQUIRED" read-only="true" />
	<tx:method name="exists*" propagation="REQUIRED" read-only="true" />
	<!-- 쓰기성 메서드 -->
	<tx:method name="save*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<tx:method name="insert*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<tx:method name="update*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<tx:method name="delete*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	<!-- 미분류 메서드는 안전하게 쓰기 트랜잭션 -->
	<tx:method name="*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
</tx:attributes>
```

### 20-1. 이유

`read-only`는 Spring 5.3 문서 기준으로 `REQUIRED` 또는 `REQUIRES_NEW`에 적용되는 속성이므로, 조회 메서드에 read-only 의미를 주려면 `SUPPORTS`보다 `REQUIRED read-only=true`가 더 명확합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.30/reference/html/data-access.html "Data Access"))

## 21. 개선안 2: DB 레벨 read-only 강제가 필요하면 `enforceReadOnly=true`

`DataSourceTransactionManager`에 아래 설정을 추가할 수 있습니다.

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
	<property name="enforceReadOnly" value="true"/>
</bean>
```

### 21-1. 기대 효과

```text
REQUIRED read-only=true 조회 트랜잭션 시작
→ Spring이 SET TRANSACTION READ ONLY 실행
→ MariaDB에서 DML 차단 가능성 증가
```

### 21-2. 주의

이 설정은 `SUPPORTS` 단독 호출에는 기대 효과가 약합니다. 새 트랜잭션이 시작되어야 `SET TRANSACTION READ ONLY`를 실행할 수 있기 때문입니다.

## 22. 개선안 3: read-only mismatch 검출

외부 read-only 트랜잭션 안에서 내부 쓰기 메서드가 `REQUIRED`로 참여하는 상황을 조기에 잡고 싶다면 다음 설정을 검토할 수 있습니다.

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
	<property name="validateExistingTransaction" value="true"/>
</bean>
```

### 22-1. 효과

```text
외부 read-only 트랜잭션
→ 내부 read-write REQUIRED 메서드 참여 시도
→ read-only mismatch를 예외로 감지 가능
```

Spring 문서에서는 기본적으로 기존 트랜잭션에 참여할 때 내부 선언의 read-only flag가 조용히 무시될 수 있고, `validateExistingTransactions`를 true로 바꾸면 read-only mismatch를 거부할 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.30/reference/html/data-access.html "Data Access"))

## 23. 개선안 4: 단순 조회 성능 우선이면 SUPPORTS 유지 가능

아래 조건이면 현재 방식도 사용할 수 있습니다.

```text
조회 메서드가 단일 SELECT 중심이다.
조회 중 정합성 스냅샷이 크게 중요하지 않다.
조회 메서드 내부에서 절대 DML을 수행하지 않는다.
read-only를 DB 차단 장치로 기대하지 않는다.
```

이 경우 현재 fallback은 다음 의미로만 받아들이는 것이 안전합니다.

```xml
<tx:method name="*" propagation="SUPPORTS" read-only="true" />
```

실무적 해석:

```text
기존 트랜잭션이 있으면 참여한다.
기존 트랜잭션이 없으면 트랜잭션 없이 조회한다.
read-only는 강제 차단보다 힌트에 가깝다.
```

## 24. 최종 권장안

Spring 5.3 + MariaDB 10.6.15 + 서비스 계층 트랜잭션 기준으로는 아래 구성을 권장합니다.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
	<tx:attributes>
		<tx:method name="get*" propagation="REQUIRED" read-only="true" />
		<tx:method name="find*" propagation="REQUIRED" read-only="true" />
		<tx:method name="select*" propagation="REQUIRED" read-only="true" />
		<tx:method name="search*" propagation="REQUIRED" read-only="true" />
		<tx:method name="count*" propagation="REQUIRED" read-only="true" />
		<tx:method name="exists*" propagation="REQUIRED" read-only="true" />
		<tx:method name="save*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
		<tx:method name="insert*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
		<tx:method name="update*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
		<tx:method name="delete*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
		<tx:method name="*" propagation="REQUIRED" rollback-for="java.lang.Exception" />
	</tx:attributes>
</tx:advice>
```

DB 레벨 read-only 강제까지 원하면:

```xml
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
	<property name="enforceReadOnly" value="true"/>
	<property name="validateExistingTransaction" value="true"/>
</bean>
```

## 25. 결론

현재 수정안은 `save*`, `insert*`, `update*`, `delete*` 쓰기 메서드를 `REQUIRED`로 잡았기 때문에 이전보다 안전합니다. 그러나 `* SUPPORTS read-only=true`는 **조회 트랜잭션을 새로 만들지 않고, read-only 적용도 제한적이며, MariaDB의 `SET TRANSACTION READ ONLY` 같은 DB 레벨 강제를 기대하기 어렵다**는 한계가 있습니다. 실무 안정성을 우선하면 **조회 메서드는 `REQUIRED read-only=true`, 쓰기 메서드는 `REQUIRED`, 미분류 메서드는 안전하게 `REQUIRED`**로 두는 구성이 더 적합합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
