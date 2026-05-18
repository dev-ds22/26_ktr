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