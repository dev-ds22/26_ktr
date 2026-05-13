정확도: 96%

# Spring Framework 5.3 CGLIB 기반 클래스 프록시 설명

## 1. 검증 결과

|항목|검증 내용|
|---|---|
|공식 근거|Spring AOP는 프록시 생성 시 **JDK Dynamic Proxy** 또는 **CGLIB**을 사용한다. CGLIB은 `spring-core`에 재패키징되어 제공된다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html "Proxying Mechanisms :: Spring Framework"))|
|5.3 동작|대상 Bean이 Interface를 구현하면 기본적으로 JDK Proxy, Interface가 없으면 CGLIB Proxy가 생성된다. CGLIB 강제 사용은 `proxy-target-class="true"`로 설정한다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html "Proxying Mechanisms :: Spring Framework"))|
|운영 위험|`final class`, `final method`, `private method`, self-invocation, ApplicationContext 위치 오류가 대표 위험이다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html "Proxying Mechanisms :: Spring Framework"))|
|대안|일반 트랜잭션은 Proxy 방식 유지가 실무적으로 무난하다. 같은 클래스 내부 호출까지 트랜잭션이 필요하면 Bean 분리 또는 AspectJ mode 검토가 필요하다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))|
|버전 주의|Spring Framework 5.x OSS 지원은 2024-08-31 종료되었으므로 신규 구축/장기 운영은 Spring 6.x 계열 검토가 필요하다. ([GitHub](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-5.3-Release-Notes "Spring Framework 5.3 Release Notes · spring-projects/spring-framework Wiki · GitHub"))|

## 2. CGLIB 프록시란?

**CGLIB 기반 클래스 프록시**는 Spring이 원본 클래스를 상속한 하위 클래스를 런타임에 만들어서, 메서드 호출 전후에 AOP 부가기능을 끼워 넣는 방식입니다.

```md
원본 Service 클래스
↓
Spring이 런타임에 하위 Proxy 클래스 생성
↓
외부 Bean은 원본 Service가 아니라 Proxy Bean을 호출
↓
Proxy가 TransactionInterceptor, Advice 등을 실행
↓
실제 Service 메서드 실행
```

예를 들어 아래 Service가 있을 때:

```java
@Service
public class OrderService {
    @Transactional
    public void saveOrder() {
        // DB insert/update
    }
}
```

Spring은 내부적으로 개념상 다음과 유사한 프록시를 만듭니다.

```java
public class OrderService$$SpringCGLIB extends OrderService {
    @Override
    public void saveOrder() {
        // 1. 트랜잭션 시작
        // 2. super.saveOrder() 또는 target method 호출
        // 3. 정상 종료 시 commit
        // 4. 예외 발생 시 rollback
    }
}
```

실제 생성 클래스명은 환경에 따라 다르지만 보통 다음과 유사하게 보입니다.

```text
com.example.order.OrderService$$EnhancerBySpringCGLIB$$...
```

## 3. JDK Dynamic Proxy와 CGLIB Proxy 차이

|구분|JDK Dynamic Proxy|CGLIB Proxy|
|---|---|---|
|기준|Interface 기반|Class 상속 기반|
|Interface 필요|필요|필수 아님|
|프록시 대상|Interface에 선언된 메서드 중심|클래스의 오버라이드 가능한 메서드|
|Service class 직접 주입|불리할 수 있음|상대적으로 유리|
|`final` class/method|Interface 호출이면 영향 적음|프록시/Advice 적용 불가|
|Spring 5.3 기본 판단|Interface 있으면 JDK Proxy|Interface 없으면 CGLIB|
|Spring 공식 문서 기준으로 대상 객체가 하나 이상의 Interface를 구현하면 JDK Dynamic Proxy가 사용되고, Interface가 없으면 CGLIB Proxy가 생성됩니다. 또한 Interface 메서드뿐 아니라 대상 클래스의 메서드까지 프록시하고 싶으면 CGLIB 사용을 강제할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html "Proxying Mechanisms :: Spring Framework"))|||

## 4. CGLIB 프록시가 만들어지는 조건

### 4.1 Interface가 없는 Service

```java
@Service
public class OrderService {
    @Transactional
    public void saveOrder() {
    }
}
```

이 경우 Interface가 없으므로 Spring AOP는 CGLIB 기반 클래스 프록시를 생성할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html "Proxying Mechanisms :: Spring Framework"))

### 4.2 Interface가 있어도 CGLIB을 강제한 경우

```xml
<tx:annotation-driven
    transaction-manager="transactionManager"
    proxy-target-class="true"/>
```

또는 Java Config:

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = true)
public class TxConfig {
}
```

`proxy-target-class="true"` 또는 `proxyTargetClass = true`는 Interface 기반 프록시 대신 클래스 기반 프록시를 사용하도록 하는 설정입니다. Spring 문서에서도 이 속성이 `true`이면 class-based proxy가 생성되고, `false` 또는 생략 시 표준 JDK interface-based proxy가 사용된다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))

## 5. 트랜잭션에서 CGLIB 프록시가 동작하는 흐름

```md
Controller
→ OrderService Proxy Bean 호출
→ TransactionInterceptor 진입
→ PlatformTransactionManager로 트랜잭션 시작
→ 실제 OrderService.saveOrder() 실행
→ 정상 종료: commit
→ RuntimeException/Error 등 rollback 대상 예외: rollback
```

핵심은 **호출 대상이 원본 객체가 아니라 Spring Container가 등록한 Proxy Bean이어야 한다는 점**입니다. Spring 문서도 기본 proxy mode에서는 Proxy를 통해 들어오는 외부 메서드 호출만 인터셉트되며, 같은 클래스 내부에서 자기 자신의 메서드를 호출하는 self-invocation은 트랜잭션이 적용되지 않는다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))

## 6. 설정 예시

### 6.1 XML 기반 트랜잭션 설정

```xml
<bean id="transactionManager"
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<tx:annotation-driven
    transaction-manager="transactionManager"
    proxy-target-class="true"/>
```

### 6.2 XML AOP 설정에서 CGLIB 강제

```xml
<aop:config proxy-target-class="true">
    <aop:pointcut id="serviceOperation"
        expression="execution(* com.example..service..*Service.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="serviceOperation"/>
</aop:config>
```

### 6.3 Java Config 방식

```java
@Configuration
@EnableTransactionManagement(proxyTargetClass = true)
public class TransactionConfig {
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

## 7. CGLIB 사용 시 반드시 알아야 할 제한

|제한|이유|결과|
|---|---|---|
|`final class`|상속 불가|프록시 생성 불가|
|`final method`|오버라이드 불가|Advice/Transaction 적용 불가|
|`private method`|오버라이드 불가|Advice/Transaction 적용 불가|
|self-invocation|Proxy를 거치지 않음|트랜잭션 미적용|
|`@PostConstruct` 내부 호출|Proxy 초기화 전일 수 있음|기대 동작 불가|
|다른 Context에 설정|Service Bean을 못 찾을 수 있음|트랜잭션 미적용|
|Spring 공식 문서는 CGLIB이 대상 클래스를 상속해야 하므로 `final class`는 프록시할 수 없고, `final`/`private` 메서드는 오버라이드할 수 없기 때문에 Advice를 적용할 수 없다고 설명합니다. 또한 Java Module System 환경에서는 CGLIB 프록시에 추가 제한이 있을 수 있습니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html "Proxying Mechanisms :: Spring Framework"))|||

## 8. `@Transactional`과 메서드 접근제어자

Spring Framework 5.3 표준 설정에서는 `@Transactional`은 기본적으로 **public 메서드에 적용하는 것이 안전합니다.**

```java
@Service
public class OrderService {
    @Transactional
    public void saveOrder() {
        // 적용 가능
    }
    @Transactional
    protected void saveInternal() {
        // 표준 설정에서는 기대대로 동작하지 않을 수 있음
    }
    @Transactional
    private void savePrivate() {
        // 적용 안 됨
    }
}
```

Spring 5.3 문서는 표준 설정의 transactional proxy에서는 `@Transactional`을 public 메서드에 적용해야 하며, `protected`, `private`, package-visible 메서드에 붙여도 오류는 발생하지 않지만 트랜잭션 설정이 적용되지 않는다고 설명합니다. 다만 class-based proxy에서 별도 `AnnotationTransactionAttributeSource(false)` 설정을 등록하면 protected/package-private 메서드도 일부 지원할 수 있습니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html "Data Access"))

## 9. self-invocation 문제

아래 코드는 CGLIB이어도 `innerSave()` 트랜잭션이 기대대로 적용되지 않습니다.

```java
@Service
public class OrderService {
    public void outerSave() {
        innerSave(); // this.innerSave()와 같음. Proxy를 거치지 않음
    }
    @Transactional
    public void innerSave() {
        // 트랜잭션 미적용 가능
    }
}
```

해결은 Service를 분리하는 방식이 가장 안전합니다.

```java
@Service
public class OrderFacade {
    private final OrderSaveService orderSaveService;
    public OrderFacade(OrderSaveService orderSaveService) {
        this.orderSaveService = orderSaveService;
    }
    public void outerSave() {
        orderSaveService.innerSave(); // 다른 Bean의 Proxy를 통해 호출
    }
}
@Service
public class OrderSaveService {
    @Transactional
    public void innerSave() {
    }
}
```

## 10. 실제 CGLIB 프록시 여부 확인 방법

```java
import org.springframework.aop.support.AopUtils;
import org.springframework.aop.framework.AopProxyUtils;
@Component
public class ProxyChecker {
    public ProxyChecker(OrderService orderService) {
        System.out.println("bean class = " + orderService.getClass());
        System.out.println("isAopProxy = " + AopUtils.isAopProxy(orderService));
        System.out.println("isCglibProxy = " + AopUtils.isCglibProxy(orderService));
        System.out.println("targetClass = " + AopProxyUtils.ultimateTargetClass(orderService));
    }
}
```

예상 출력:

```text
bean class = class com.example.OrderService$$EnhancerBySpringCGLIB$$...
isAopProxy = true
isCglibProxy = true
targetClass = class com.example.OrderService
```

## 11. 운영 적용 기준

### 권장 구조

```java
@Service
public class OrderService {
    @Transactional
    public void saveOrder() {
        // DB 변경 작업
    }
}
```

```xml
<tx:annotation-driven
    transaction-manager="transactionManager"
    proxy-target-class="true"/>
```

### 피해야 할 구조

```java
@Service
public final class OrderService {
    @Transactional
    public void saveOrder() {
    }
}
```

```java
@Service
public class OrderService {
    @Transactional
    private void saveOrder() {
    }
}
```

```java
@Service
public class OrderService {
    public void outer() {
        this.saveOrder();
    }
    @Transactional
    public void saveOrder() {
    }
}
```

## 12. 바로 적용 절차

```md
1. Service 클래스가 Spring Bean으로 등록되어 있는지 확인한다.
2. <tx:annotation-driven proxy-target-class="true"/> 또는 @EnableTransactionManagement(proxyTargetClass = true)를 설정한다.
3. 트랜잭션 대상 메서드는 public으로 작성한다.
4. 클래스와 메서드에 final을 사용하지 않는다.
5. 같은 클래스 내부에서 @Transactional 메서드를 직접 호출하지 않는다.
6. AopUtils.isCglibProxy(bean)로 실제 CGLIB 프록시 여부를 확인한다.
7. RuntimeException을 발생시키는 테스트 메서드를 만들어 rollback 여부를 검증한다.
8. 신규 구축 또는 장기 운영이면 Spring 6.x + Java 17 전환 가능성을 별도 검토한다.
```

## 최종 결론

Spring Framework 5.3에서 CGLIB 기반 클래스 프록시는 **Interface 없이도 Service 클래스에 트랜잭션/AOP를 적용하기 위한 핵심 방식**입니다. 다만 CGLIB은 원본 클래스를 상속해 프록시를 만들기 때문에 **final, private, self-invocation, public 메서드 여부**가 실제 동작을 좌우합니다. 운영에서는 `proxy-target-class="true"`를 설정하더라도 반드시 **프록시 여부 확인 + rollback 테스트**까지 수행해야 안전합니다.

# 문2. Spring Framework 에서 CGLIB 방식의 Service 이용 시 해당 Service 의 메소드 처리 시에는 하나의 Transaction 에 종속되는가?

# 결론

**조건부로 “하나의 Transaction에 종속된다”고 볼 수 있습니다.**  
정확히는 **CGLIB Service의 메서드가 Spring Proxy를 통해 호출되고, 그 메서드에 `@Transactional` 또는 XML 트랜잭션 Advice가 적용되면 해당 메서드 실행 구간은 하나의 트랜잭션 경계 안에서 처리됩니다.** 다만 이것은 **CGLIB이라서가 아니라 Spring Transaction AOP가 메서드 호출을 가로채기 때문**입니다. Spring AOP는 대상 객체에 대해 JDK Dynamic Proxy 또는 CGLIB Proxy를 만들고, CGLIB은 대상 클래스의 런타임 하위 클래스를 만들어 프록시합니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))

# 핵심 판단

|구분|답변|
|---|---|
|Service 자체|하나의 Transaction에 고정되지 않음|
|Service의 `@Transactional` 메서드 1회 호출|일반적으로 하나의 트랜잭션 경계가 됨|
|기본 전파 옵션|`REQUIRED`|
|외부 Transaction 없음|새 Transaction 생성|
|외부 Transaction 있음|기존 Transaction에 참여|
|CGLIB의 역할|Transaction을 만드는 것이 아니라 메서드 호출을 가로채는 Proxy 역할|
|예외|`REQUIRES_NEW`, `NESTED`, `NOT_SUPPORTED`, self-invocation 등|

# 1. 일반적인 경우: 메서드 1회 호출 = 하나의 Transaction 경계

예를 들어 다음 Service가 있다고 가정합니다.

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        orderMapper.insertOrder();
        stockMapper.decreaseStock();
        paymentMapper.insertPaymentRequest();
    }
}
```

외부 Bean에서 다음처럼 호출하면:

```java
orderService.createOrder();
```

처리 흐름은 다음과 같습니다.

```md
Controller
→ OrderService CGLIB Proxy
→ TransactionInterceptor
→ Transaction 시작 또는 기존 Transaction 참여
→ createOrder() 실제 실행
→ 정상 종료 시 commit
→ rollback 대상 예외 발생 시 rollback
```

이 경우 `createOrder()` 안의 여러 DB 작업은 **같은 트랜잭션 컨텍스트** 안에서 처리됩니다. Spring 문서 기준으로 선언적 트랜잭션은 Proxy 기반 AOP와 `TransactionInterceptor`를 통해 메서드 호출 전후에 트랜잭션 처리를 수행합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html?utm_source=chatgpt.com "Data Access - Spring"))

# 2. 단, “항상 새 Transaction 1개”라는 뜻은 아님

Spring의 기본 전파 옵션인 `PROPAGATION_REQUIRED`에서는 트랜잭션이 없으면 새 물리 트랜잭션을 만들고, 이미 트랜잭션이 있으면 기존 트랜잭션에 참여합니다. Spring 공식 문서는 `PROPAGATION_REQUIRED`에서 각 메서드마다 논리 트랜잭션 범위가 만들어지지만, 표준 동작에서는 이 논리 범위들이 같은 물리 트랜잭션에 매핑된다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-propagation.html?utm_source=chatgpt.com "Transaction Propagation :: Spring Framework"))

## 외부 Transaction이 없는 경우

```java
controller.orderCreate()
→ orderService.createOrder()
```

결과:

```md
Transaction 없음
→ createOrder() 진입 시 새 Transaction 생성
→ createOrder() 종료 시 commit/rollback
```

이 경우에는 **메서드 호출 1회가 하나의 물리 Transaction**이라고 볼 수 있습니다.

## 외부 Transaction이 있는 경우

```java
@Service
public class OrderFacade {
    @Transactional
    public void order() {
        orderService.createOrder();
    }
}
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
    }
}
```

결과:

```md
OrderFacade.order()에서 Transaction 시작
→ OrderService.createOrder()는 기존 Transaction에 참여
→ 전체가 하나의 물리 Transaction으로 처리
```

따라서 `createOrder()`도 트랜잭션 안에서 실행되지만, **createOrder()만의 독립 Transaction이 새로 생기는 것은 아닙니다.**

# 3. 전파 옵션에 따라 하나의 Transaction이 아닐 수 있음

|전파 옵션|메서드 처리 시 Transaction 관계|
|---|---|
|`REQUIRED`|기존 Transaction 있으면 참여, 없으면 생성|
|`REQUIRES_NEW`|항상 새 Transaction 생성, 기존 Transaction은 일시 중단|
|`NESTED`|기존 Transaction 안에서 savepoint 기반 중첩 처리 가능|
|`SUPPORTS`|Transaction 있으면 참여, 없으면 없이 실행|
|`NOT_SUPPORTED`|Transaction 없이 실행, 기존 Transaction은 일시 중단|
|`NEVER`|Transaction이 있으면 예외|
|`MANDATORY`|기존 Transaction이 반드시 있어야 함|
|특히 `REQUIRES_NEW`는 외부 트랜잭션과 독립적인 물리 트랜잭션을 사용합니다. 이 경우 외부 트랜잭션과 내부 트랜잭션이 분리되므로, “메서드들이 모두 하나의 Transaction에 종속된다”고 보면 안 됩니다. Spring 문서도 `REQUIRES_NEW`는 기존 외부 트랜잭션에 참여하지 않고 독립적인 물리 트랜잭션을 사용한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/data-access.html?utm_source=chatgpt.com "Data Access - Spring"))||

# 4. CGLIB이어도 내부 메서드 호출은 주의

아래 코드는 많이 착각하는 구조입니다.

```java
@Service
public class OrderService {
    public void outer() {
        inner(); // this.inner() 호출
    }
    @Transactional
    public void inner() {
        // 기대와 달리 Transaction 미적용 가능
    }
}
```

CGLIB Proxy를 사용하더라도 같은 클래스 내부의 `this.inner()` 호출은 Proxy를 거치지 않습니다. Spring 공식 문서는 self-invocation, 즉 대상 객체 내부에서 자기 자신의 메서드를 호출하는 경우 Advice가 적용될 기회를 얻지 못한다고 설명합니다. ([Home](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html?utm_source=chatgpt.com "Proxying Mechanisms :: Spring Framework"))  
안전한 구조는 Bean을 분리하는 것입니다.

```java
@Service
public class OrderService {
    private final OrderTxService orderTxService;
    public OrderService(OrderTxService orderTxService) {
        this.orderTxService = orderTxService;
    }
    public void outer() {
        orderTxService.inner();
    }
}
@Service
public class OrderTxService {
    @Transactional
    public void inner() {
    }
}
```

# 5. 운영 관점에서의 정확한 표현

아래 표현이 가장 정확합니다.

```md
CGLIB 방식의 Service 메서드가 Proxy를 통해 호출되고 Transaction Advice 대상이면,
그 메서드 실행 구간은 TransactionInterceptor가 관리하는 트랜잭션 경계 안에서 처리된다.
기본 REQUIRED에서는 기존 트랜잭션이 없으면 새 Transaction을 만들고,
기존 트랜잭션이 있으면 그 Transaction에 참여한다.
따라서 “메서드 처리 시 하나의 Transaction에 종속된다”는 표현은 일반적으로 맞지만,
전파 옵션과 호출 경로에 따라 달라진다.
```

# 6. 바로 적용 가능한 점검 절차

```md
1. Service가 Spring Bean인지 확인한다.
2. 실제 주입 객체가 CGLIB Proxy인지 확인한다.
   - AopUtils.isCglibProxy(orderService)
3. 대상 메서드가 public인지 확인한다.
4. 대상 메서드에 @Transactional 또는 XML tx:method가 적용되는지 확인한다.
5. 외부 Bean에서 Proxy를 통해 호출되는지 확인한다.
6. propagation 값을 확인한다.
   - 기본값: REQUIRED
7. 같은 클래스 내부 this 호출이 아닌지 확인한다.
8. RuntimeException 발생 테스트로 rollback 여부를 검증한다.
9. REQUIRES_NEW 사용 시 Connection Pool 여유와 Lock 위험을 확인한다.
```

# 최종 답

**CGLIB 방식의 Service 메서드 처리 시, 해당 메서드가 트랜잭션 대상이고 Proxy를 통해 호출되면 하나의 트랜잭션 경계 안에서 처리됩니다.**  
다만 **항상 독립적인 새 Transaction 하나가 생긴다는 뜻은 아닙니다.** 기본값 `REQUIRED`에서는 외부 Transaction이 없으면 새로 만들고, 있으면 기존 Transaction에 참여합니다. 따라서 실무적으로는 **“Service가 Transaction에 종속된다”가 아니라 “Proxy를 통한 메서드 호출이 Transaction 경계가 된다”**고 이해하는 것이 가장 정확합니다.