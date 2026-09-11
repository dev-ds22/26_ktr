## 결론

현재 설정이 모든 Service 메소드에 대해 아래처럼 동일하게 적용되고 있다면,

```xml
<tx:method name="*" propagation="REQUIRED" rollback-for="Exception"/>
```

**CRUD별로 세분화하는 것은 실무적으로 가치가 있습니다.** 다만 `C=INSERT`, `R=SELECT`, `U=UPDATE`, `D=DELETE`처럼 기계적으로 4등분하기보다는 **실제 트랜잭션 성격에 따라 `조회 / 변경 / 잠금조회 / 독립처리 / 장시간 처리`로 분류하는 방식**이 더 안전합니다.

현재 Spring 5.3 + MariaDB 10.6 환경에서는 다음을 기본안으로 권장합니다.

| 기능 | 권장 기본 설정 | 판단 |
|---|---|---|
| 일반 R | `REQUIRED + read-only=true` | 가장 안전한 기본 |
| 단순 단건 R | 필요 시 `SUPPORTS + read-only=true` | 성능 최적화용, 선택적 |
| C | `REQUIRED + rollback-for=Exception` | 기존 정책 유지 |
| U | `REQUIRED + rollback-for=Exception` | 기존 정책 유지 |
| D | `REQUIRED + rollback-for=Exception` | 기존 정책 유지 |
| `SELECT FOR UPDATE` | `REQUIRED + read-only=false` | 일반 조회와 분리 필수 |
| 독립 로그/이력 저장 | 필요할 때만 `REQUIRES_NEW` | 남용 금지 |
| 대용량 배치 | `REQUIRED + timeout` 별도 | 일반 CRUD와 분리 |

Spring의 `<tx:method>`는 `propagation`, `isolation`, `timeout`, `read-only`, `rollback-for`, `no-rollback-for`를 개별 메소드 패턴별로 설정할 수 있습니다. 

---

## 1. 현재 설정의 특징

현재:

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>
    </tx:attributes>
</tx:advice>
```

이면 사실상 모든 대상 Service 메소드가:

```text
조회
등록
수정
삭제
단순 Validation 조회
count 조회
목록 조회
외부 연계 전 조회
```

전부 동일하게

```text
REQUIRED
readOnly=false
isolation=DEFAULT
timeout=기본값
rollback=Exception
```

으로 처리됩니다.

기능적으로는 안전한 편이지만 **조회까지 항상 read-write transaction으로 처리**한다는 점에서 최적화 여지가 있습니다.

---

## 2. CRUD별 세분화의 장점

| 장점 | 설명 |
|---|---|
| 조회 트랜잭션 의도 명확화 | `read-only=true`를 통해 해당 메소드가 조회 전용임을 선언 |
| 실수 조기 발견 가능 | read-only 강제 설정까지 적용하면 조회 메소드에서 잘못된 UPDATE/INSERT 검출 가능 |
| DB/Driver 최적화 가능 | read-only 정보를 TransactionManager/Driver/DB가 활용할 수 있음 |
| 코드 리뷰 용이 | 메소드 이름만으로도 transaction 목적을 추정 가능 |
| timeout 세분화 | 일반 CRUD와 장시간 배치/집계를 다르게 관리 가능 |
| 장애 영향 범위 감소 | 불필요하게 긴 transaction을 줄일 수 있음 |
| Lock 유지시간 개선 가능 | 단순 조회에서 transaction 범위를 최소화할 수 있음 |
| 운영 분석 용이 | 어떤 종류의 업무가 긴 transaction인지 파악하기 쉬움 |

특히 MariaDB 자체도 `READ ONLY` transaction mode를 지원하고, read-only transaction에서는 storage engine이 사용할 수 있는 최적화가 존재합니다. 

다만 Spring의 `readOnly=true` 자체는 **반드시 DB 쓰기를 차단한다는 의미가 아닙니다.** Spring 5.3에서도 기본적으로 read-only는 transaction subsystem에 제공되는 hint 성격입니다. 

---

## 3. 단점

세분화할수록 무조건 좋아지는 것은 아닙니다.

| 단점 | 실무 영향 |
|---|---|
| 메소드 명명 규칙 의존 | `get*`인데 UPDATE를 하는 기존 코드가 있으면 장애 가능 |
| 설정 복잡성 증가 | 패턴이 많아질수록 유지보수 난이도 상승 |
| 잘못된 read-only 적용 위험 | 조회 이름을 가진 변경 메소드에서 오류 발생 가능 |
| propagation 오설정 위험 | `SUPPORTS`, `REQUIRES_NEW` 남용 시 transaction 경계 변경 |
| 기존 rollback 동작 변경 가능 | `rollback-for="Exception"` 누락 시 checked Exception commit 가능 |
| timeout 예상과 실제 차이 | 기존 REQUIRED transaction에 참여하면 내부 timeout이 무시될 수 있음 |
| 기존 시스템 회귀 위험 | 전체 Service에 일괄 변경 시 숨어 있던 비정상 CRUD 구조가 노출 |

따라서 **전역 `REQUIRED` → CRUD 세분화를 한 번에 변경하는 것은 권하지 않습니다.**

---

## 4. 가장 중요한 주의사항: rollback-for를 잃으면 안 됨

현재 시스템은:

```xml
rollback-for="Exception"
```

이므로 다음 checked exception도 rollback 대상입니다.

```java
public void saveOrder() throws Exception {
    dao.insert(...);

    if (...) {
        throw new Exception("처리 실패");
    }
}
```

현재는:

```text
INSERT
 ↓
Exception
 ↓
ROLLBACK
```

입니다.

그런데 수정하면서:

```xml
<tx:method name="insert*" propagation="REQUIRED"/>
```

처럼 `rollback-for`를 빠뜨리면 Spring 기본 정책은 일반적으로:

```text
RuntimeException → rollback
Error            → rollback
checked Exception → commit
```

입니다. Spring 5.3 `@Transactional` 역시 별도 rollback rule이 없으면 checked exception을 기본 rollback 대상으로 하지 않습니다. 

따라서 기존 정책을 유지하려면 반드시:

```xml
rollback-for="Exception"
```

을 C/U/D에 그대로 넣는 것이 안전합니다.

---

## 5. 권장 1차 수정안

현재 시스템에 가장 현실적인 방식입니다.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>

        <!-- ==========================================
             조회
             ========================================== -->
        <tx:method name="get*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="find*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="select*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="search*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="list*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="count*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>


        <!-- ==========================================
             등록
             ========================================== -->
        <tx:method name="insert*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="create*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="add*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="save*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>


        <!-- ==========================================
             수정
             ========================================== -->
        <tx:method name="update*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="modify*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>


        <!-- ==========================================
             삭제
             ========================================== -->
        <tx:method name="delete*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="remove*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>


        <!-- ==========================================
             기존 메소드 보호용 fallback
             ========================================== -->
        <tx:method name="*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

    </tx:attributes>
</tx:advice>
```

이 방식의 핵심은 마지막:

```xml
<tx:method name="*" .../>
```

을 **없애지 않는 것**입니다.

기존에 이름 규칙을 따르지 않는:

```java
processOrder()
executeJoin()
handlePayment()
changeState()
approveSeller()
joinSeller()
```

같은 메소드도 계속 기존 `REQUIRED` 정책을 적용받습니다.

즉 처음부터:

```text
CRUD 이름에 안 맞으면 transaction 없음
```

으로 바꾸지 않습니다.

---

## 6. Spring은 어떤 패턴을 선택하는가?

예를 들어:

```xml
<tx:method name="get*" read-only="true"/>
<tx:method name="*" read-only="false"/>
```

가 있을 때:

```java
getOrder()
```

는 `get*`가 적용됩니다.

Spring 5.3.39의 `NameMatchTransactionAttributeSource` 구현을 보면 먼저 exact method name을 찾고, 없으면 wildcard 중에서 **더 구체적인 이름 패턴을 선택**합니다. 실제 구현에서도 가장 긴 matching name을 선택합니다. 

따라서:

```xml
<tx:method name="get*" .../>
<tx:method name="*" .../>
```

구조는 정상적인 사용 방식입니다.

---

## 7. 하지만 `get*`, `select*`를 무조건 read-only로 하면 위험

실무에서 가장 먼저 조사해야 하는 부분입니다.

예를 들어 기존 코드가:

```java
public OrderVO getOrder(long orderNo) throws Exception {

    OrderVO order = orderDAO.selectOrder(orderNo);

    orderDAO.updateLastAccessDate(orderNo);

    return order;
}
```

라면 메소드 이름은:

```text
getOrder
```

이지만 실제 작업은:

```text
SELECT
UPDATE
```

입니다.

이를:

```xml
<tx:method name="get*" read-only="true"/>
```

로 바꾸면 설계 의도와 충돌합니다.

따라서 적용 전에는 반드시:

```text
Service method
    ↓
호출 Service
    ↓
DAO
    ↓
실제 SQL
```

까지 조사해야 합니다.

특히 오래된 커머스 시스템에서는 메소드 이름만으로 CRUD를 판단하면 안 됩니다.

---

## 8. `SELECT FOR UPDATE`는 R로 보면 안 됨

예를 들어 재고를 처리한다고 하면:

```java
public StockVO getStockForUpdate(long itemNo) {

    StockVO stock = stockDAO.selectStockForUpdate(itemNo);

    ...

    return stock;
}
```

SQL:

```sql
SELECT *
  FROM STOCK
 WHERE ITEM_NO = ?
 FOR UPDATE;
```

이것은 SELECT지만 transaction 관점에서는 일반 조회가 아닙니다.

```text
row lock 획득
      ↓
동일 transaction 내 UPDATE 수행
      ↓
COMMIT/ROLLBACK에서 lock 해제
```

따라서:

```xml
<tx:method name="getStockForUpdate*"
           propagation="REQUIRED"
           read-only="false"
           rollback-for="Exception"/>
```

처럼 별도로 처리해야 합니다.

그리고 일반:

```xml
<tx:method name="get*" read-only="true"/>
```

보다 `getStockForUpdate*`가 더 구체적인 패턴이므로 해당 설정이 선택될 수 있습니다. 

실무에서는 이름도 아예:

```java
lockStock()
reserveStock()
updateStockQuantity()
```

같이 transaction 성격이 드러나도록 하는 편이 낫습니다.

---

## 9. R은 REQUIRED인가 SUPPORTS인가?

이 부분이 가장 많이 고민되는 부분입니다.

### `REQUIRED + read-only=true`

```xml
<tx:method name="select*"
           propagation="REQUIRED"
           read-only="true"/>
```

호출 시 transaction이 없다면 read-only transaction을 하나 생성합니다.

```text
Controller
  ↓
selectOrder()
  ↓
READ ONLY TX 시작
  ↓
SELECT A
SELECT B
SELECT C
  ↓
COMMIT
```

장점은 여러 SELECT가 **하나의 transaction 경계 안에서 일관된 상태를 유지하기 쉽다**는 것입니다.

커머스 핵심 Service 조회에는 이쪽을 우선 권장합니다.

---

### `SUPPORTS + read-only=true`

```xml
<tx:method name="select*"
           propagation="SUPPORTS"
           read-only="true"/>
```

의 의미는:

```text
기존 TX 존재 → 기존 TX 참여
기존 TX 없음 → 새 TX 생성하지 않음
```

입니다.

간단한:

```java
selectCode()
countNotice()
findSimpleConfig()
```

같은 단일 조회에는 transaction 시작 비용을 줄일 수 있습니다.

하지만:

```java
OrderVO getOrderDetail() {

    selectOrder();
    selectPayment();
    selectDelivery();
    selectClaim();

}
```

처럼 여러 데이터를 조합한다면 그 사이 데이터가 변경될 가능성과 조회 일관성을 고려해야 합니다.

### 제 권장

| 조회 종류 | 권장 |
|---|---|
| 핵심 주문/결제/재고/회원 조회 | `REQUIRED + readOnly=true` |
| 여러 테이블 조합 조회 | `REQUIRED + readOnly=true` |
| 단순 코드/설정 조회 | `SUPPORTS + readOnly=true` 검토 가능 |
| 단순 count | `SUPPORTS` 검토 가능 |
| SELECT FOR UPDATE | `REQUIRED + readOnly=false` |
| C/U/D Transaction 내부의 조회 | 외부 REQUIRED transaction 참여 |

Spring 문서에서도 `REQUIRED`는 기존 transaction이 있으면 참여하고, 없으면 새 physical transaction을 만드는 일반적인 Service 계층 기본값으로 설명합니다. 

---

## 10. 현재 환경에서는 1차적으로 `SUPPORTS`까지 적용하지 않는 것을 권장

이번 리팩터링 목적이:

> 전역 `REQUIRED`를 CRUD별로 정밀화

라면 한 번에:

```text
R: REQUIRED → SUPPORTS
R: readOnly false → true
CUD: 기존 유지
timeout 변경
isolation 변경
```

까지 모두 바꾸는 것은 회귀 원인을 찾기 어렵게 만듭니다.

따라서 **1차 변경은 다음 정도가 적절합니다.**

```text
기존
모든 Service
REQUIRED + read-write
        ↓
1차
R   : REQUIRED + readOnly
CUD : REQUIRED + read-write

        ↓ 검증

2차
일부 단순 R
REQUIRED → SUPPORTS 검토

        ↓ 성능 측정

3차
batch / external interface / 독립 transaction
별도 정책
```

이렇게 해야 장애 발생 시 원인을 분리할 수 있습니다.

---

## 11. timeout도 CRUD 공통으로 넣지 않는 것이 좋음

예를 들어:

```xml
<tx:method name="*" timeout="30"/>
```

을 무작정 넣으면 장시간 정상 업무가 rollback될 가능성이 있습니다.

대신:

```xml
<tx:method name="batch*"
           propagation="REQUIRED"
           timeout="300"
           rollback-for="Exception"/>

<tx:method name="update*"
           propagation="REQUIRED"
           timeout="30"
           rollback-for="Exception"/>
```

처럼 실제 SLA와 SQL 수행시간을 조사한 후 적용해야 합니다.

Spring 5.3에서 `timeout`은 `REQUIRED` 또는 `REQUIRES_NEW`에서 **새 transaction을 시작할 때 의미가 있습니다.** 

또한 다음 구조라면:

```text
AService.process() REQUIRED
        ↓ T1 생성

BService.update() REQUIRED timeout=5
        ↓
T1 참여
```

내부 `BService`의 timeout=5가 기존 T1을 새롭게 5초 transaction으로 바꾸는 것이 아닙니다. `REQUIRED`로 기존 transaction에 참여하면 outer transaction 특성이 우선되며, Spring 문서도 기존 transaction에 참여하는 내부 scope의 isolation/timeout/read-only 속성이 기본적으로 무시될 수 있음을 명시합니다. 

---

## 12. isolation은 CRUD별로 건드리지 않는 것이 기본

예를 들어:

```xml
<tx:method name="select*"
           isolation="READ_COMMITTED"/>

<tx:method name="update*"
           isolation="SERIALIZABLE"/>
```

같은 설정은 특별한 근거가 없다면 추천하지 않습니다.

`SERIALIZABLE` 같은 높은 isolation을 광범위하게 적용하면:

```text
lock 증가
→ transaction 대기 증가
→ connection 점유 증가
→ 응답시간 증가
→ deadlock/lock timeout 증가 가능
```

가 발생할 수 있습니다.

CRUD 세분화의 1차 목적은:

```text
R → readOnly
CUD → readWrite
```

정도로 잡고 isolation 변경은 **실제 동시성 문제를 확인한 업무만 별도 적용**하는 것이 좋습니다.

---

## 13. readOnly=true에 대한 중요한 오해

```xml
read-only="true"
```

를 설정했다고 해서 Spring 자체가 항상:

```text
UPDATE 발견
→ 무조건 Exception
```

을 발생시키는 것은 아닙니다.

Spring 문서에서도 기본 read-only flag는 underlying transaction subsystem에 전달되는 hint라고 설명합니다. 

`DataSourceTransactionManager`의 경우 `enforceReadOnly=true`를 사용하면 JDBC `Connection.setReadOnly()` 수준보다 강하게 DB connection에 read-only transaction 명령을 적용하도록 할 수 있습니다. Spring 5.3에 해당 기능이 존재합니다. 

MariaDB도:

```sql
START TRANSACTION READ ONLY;
```

및

```sql
SET TRANSACTION READ ONLY;
```

을 지원하며 read-only transaction에서 쓰기를 제한할 수 있습니다. 

다만 현재처럼 기존 운영 시스템이라면 **처음부터 enforceReadOnly까지 켜지 않는 것을 권장**합니다.

먼저:

```text
readOnly=true를 metadata/hint 용도로 적용
→ 회귀 테스트
→ 잘못된 조회 메소드 내 CUD 제거
→ 별도 테스트 환경에서 enforceReadOnly 검증
```

순서가 안전합니다.

특히 여러 TransactionManager를 묶어 사용하는 구조라면 각 underlying TransactionManager가 read-only를 어떻게 처리하는지도 별도로 확인해야 합니다. 

---

## 14. 실무 권장 최종 XML 예시

현재 시스템을 크게 흔들지 않는다면 이 정도를 추천합니다.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>

        <!-- Locking query -->
        <tx:method name="*ForUpdate*"
                   propagation="REQUIRED"
                   read-only="false"
                   rollback-for="Exception"/>

        <tx:method name="lock*"
                   propagation="REQUIRED"
                   read-only="false"
                   rollback-for="Exception"/>


        <!-- Read -->
        <tx:method name="get*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="find*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="select*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="search*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="list*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="count*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>


        <!-- Create -->
        <tx:method name="insert*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="create*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="add*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="save*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>


        <!-- Update -->
        <tx:method name="update*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="modify*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>


        <!-- Delete -->
        <tx:method name="delete*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="remove*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>


        <!-- Fallback : 기존 동작 보장 -->
        <tx:method name="*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

    </tx:attributes>
</tx:advice>

<aop:config>
    <aop:pointcut id="requiredTx"
        expression="execution(* app..service.*Service.*(..))"/>

    <aop:advisor
        advice-ref="txAdvice"
        pointcut-ref="requiredTx"/>
</aop:config>
```

### 이 설정의 핵심

```text
기존 transaction 경계는 거의 그대로 유지
             +
조회에만 readOnly 속성 추가
             +
잠금조회 예외처리
             +
이름 규칙에서 빠진 기존 메소드는 fallback REQUIRED
```

따라서 **첫 단계 리팩터링으로는 상당히 안전한 편**입니다.

---

## 15. 실제 Service 예시

```java
@Service
public class OrderService {

    /**
     * 주문 조회
     * → REQUIRED + readOnly=true
     */
    public OrderVO getOrder(long orderNo) throws Exception {
        return orderDAO.selectOrder(orderNo);
    }

    /**
     * 주문 등록
     * → REQUIRED + readWrite
     */
    public long insertOrder(OrderVO order) throws Exception {

        orderDAO.insertOrder(order);
        orderDAO.insertOrderHistory(order);

        return order.getOrderNo();
    }

    /**
     * 주문 변경
     * → REQUIRED + readWrite
     */
    public void updateOrder(OrderVO order) throws Exception {

        orderDAO.updateOrder(order);
        orderDAO.insertOrderHistory(order);
    }

    /**
     * 주문 삭제
     * → REQUIRED + readWrite
     */
    public void deleteOrder(long orderNo) throws Exception {

        orderDAO.deleteOrderDetail(orderNo);
        orderDAO.deleteOrder(orderNo);
    }
}
```

실행 결과는 다음과 같습니다.

```text
getOrder()
   → get*
   → REQUIRED
   → readOnly=true

insertOrder()
   → insert*
   → REQUIRED
   → readOnly=false
   → Exception 발생 시 rollback

updateOrder()
   → update*
   → REQUIRED
   → readOnly=false
   → Exception 발생 시 rollback

deleteOrder()
   → delete*
   → REQUIRED
   → readOnly=false
   → Exception 발생 시 rollback
```

---

## 16. 실무 수정 시 반드시 선행 조사할 항목

이번 변경에서는 코드 수정보다 **사전 분류 작업이 더 중요합니다.**

| 확인 대상 | 반드시 확인할 내용 |
|---|---|
| Service 메소드명 | get/find/select/list/count 등 naming 현황 |
| 조회 메소드 | 내부에 INSERT/UPDATE/DELETE가 숨어 있는지 |
| 변경 메소드 | 여러 DAO가 하나의 transaction을 구성하는지 |
| 잠금 조회 | `FOR UPDATE`, pessimistic lock 존재 여부 |
| 내부 Service 호출 | 기존 transaction 참여 관계 |
| Self invocation | 같은 Service 내부 호출 여부 |
| 예외 | checked Exception 사용 여부 |
| rollback | 기존 `rollback-for="Exception"` 의존 여부 |
| 외부 API | transaction을 잡은 상태에서 HTTP/socket 호출하는지 |
| 대량 처리 | transaction 수행시간 |
| 다중 DB | 동일 transaction에서 DB 2개 이상 접근 여부 |
| 비동기 | `@Async`, 별도 Thread 사용 여부 |

특히:

```java
@Transactional
public void process() {
    externalApi.call();   // 10초
    dao.update();
}
```

처럼 **DB transaction을 잡은 상태에서 외부 HTTP/socket을 기다리는 구조**가 있다면 CRUD 세분화보다 이 문제를 먼저 또는 함께 개선하는 것이 DB connection pool과 lock 관점에서 더 큰 효과를 낼 수 있습니다.

---

## 17. 적용 순서

실무적으로는 다음 순서가 가장 안전합니다.

1. **현재 Service 메소드 전수 목록 작성**
   `get/find/select/insert/update/delete/process/save/...`별로 실제 SQL 동작을 분류합니다.

2. **1차 변경**
   `R = REQUIRED + readOnly=true`, `C/U/D = 기존 REQUIRED`, `* = 기존 fallback`만 적용합니다.

3. **회귀 테스트**
   회원가입, 로그인, 주문, 결제, 취소, 환불, 재고, 판매자 등록 등 핵심 transaction을 테스트합니다.

4. **운영 성능 비교**
   DB connection 사용량, active transaction, lock wait, deadlock, 평균 SQL 수행시간을 변경 전후 비교합니다.

5. **2차 최적화**
   매우 단순한 조회만 `SUPPORTS + readOnly=true`, 장시간 업무에 timeout, 독립성이 정말 필요한 업무만 `REQUIRES_NEW`를 검토합니다.

---

## 최종 권고

현재 구조에서는 처음부터 CRUD마다 서로 다른 propagation을 주는 것보다 다음 수준으로 시작하는 것이 가장 적절합니다.

```text
                현재
                  │
                  ▼
       모든 Service = REQUIRED
                  │
                  ▼
────────────────────────────────
1차 개선
────────────────────────────────
R
REQUIRED
readOnly=true

C/U/D
REQUIRED
readOnly=false
rollback-for=Exception

기타
REQUIRED
rollback-for=Exception
(fallback 유지)
────────────────────────────────
                  │
                  ▼
       회귀 + 성능 + Lock 검증
                  │
                  ▼
────────────────────────────────
2차 개선
────────────────────────────────
단순 R       → SUPPORTS 검토
잠금 R       → REQUIRED read-write
장시간 처리   → timeout 개별 설정
독립 처리     → REQUIRES_NEW 제한 사용
특수 동시성   → isolation 개별 검토
────────────────────────────────
```

**가장 중요한 것은 `R=readOnly`, `C/U/D=REQUIRED` 자체가 아니라, 기존 `rollback-for="Exception"`과 transaction 경계를 보존하면서 점진적으로 변경하는 것입니다.**

현재 시스템에서는 우선 **조회 메소드에 `read-only="true"`만 추가하고 C/U/D의 `REQUIRED + rollback-for="Exception"`은 그대로 유지하는 1차 변경**이 위험 대비 효과가 가장 좋습니다. 그 이후 실제 DB 부하와 transaction 수행시간을 측정한 후 `SUPPORTS`, `timeout`, `REQUIRES_NEW`까지 확장하는 것이 적절합니다.

---
## 결론

질문의 메소드가 **현재 기존 설정 그대로**라면:

```xml
<tx:method name="*"
           propagation="REQUIRED"
           rollback-for="Exception"/>
```

`getOrder()`도 **일반 read-write `REQUIRED` 트랜잭션**이므로 `UPDATE`는 정상적으로 수행됩니다.

```java
public OrderVO getOrder(long orderNo) throws Exception {
    OrderVO order = orderDAO.selectOrder(orderNo);
    orderDAO.updateLastAccessDate(orderNo);
    return order;
}
```

실행 구조는 다음과 같습니다.

```text
getOrder() 진입
    ↓
REQUIRED Transaction 시작(T1)
readOnly = false
    ↓
SELECT ─────────┐
                │ 같은 T1
UPDATE ─────────┘
    ↓
정상 종료
    ↓
COMMIT
```

따라서 `SELECT`와 `UPDATE`는 하나의 트랜잭션에서 실행되고, 중간 또는 이후에 `Exception`이 발생하면 현재 `rollback-for="Exception"` 정책에 의해 UPDATE도 같이 rollback됩니다.

문제는 앞서 검토한 것처럼 `get*`를 다음처럼 변경했을 때입니다.

```xml
<tx:method name="get*"
           propagation="REQUIRED"
           read-only="true"
           rollback-for="Exception"/>
```

이 경우 **UPDATE가 반드시 실패한다고 볼 수 없습니다.** 이 점이 이번 리팩터링에서 가장 중요한 위험요소입니다.

---

## 1. `getOrder()`가 `readOnly=true`일 때 실제 동작

다음 설정을 적용했다고 가정하겠습니다.

```xml
<tx:method name="get*"
           propagation="REQUIRED"
           read-only="true"
           rollback-for="Exception"/>

<tx:method name="*"
           propagation="REQUIRED"
           rollback-for="Exception"/>
```

메소드:

```java
public OrderVO getOrder(long orderNo) throws Exception {

    OrderVO order = orderDAO.selectOrder(orderNo);

    orderDAO.updateLastAccessDate(orderNo);

    return order;
}
```

외부 transaction이 없다면 Spring이 생성하는 트랜잭션의 논리적 속성은:

```text
Propagation : REQUIRED
readOnly    : true
Isolation   : DEFAULT
Rollback    : Exception 포함
```

입니다.

중요한 것은:

> `readOnly=true`가 SELECT 문에만 적용되고 UPDATE 때 자동으로 read-write로 바뀌는 것이 아닙니다.

전체 Service 메소드가 하나의 트랜잭션입니다.

```text
BEGIN Transaction
    readOnly = true

    SELECT order

    UPDATE last_access_date

COMMIT
```

---

## 2. 그러면 UPDATE는 성공하는가?

### 답: 환경에 따라 성공할 수도 있고 실패할 수도 있습니다.

Spring 5.3의 `readOnly=true`는 기본적으로 **쓰기 금지를 보장하는 제약조건이 아니라 transaction subsystem에 전달되는 hint**입니다. Spring 공식 문서도 read-only flag가 반드시 write 접근을 실패시키는 것은 아니라고 명시하고 있습니다. 

따라서 크게 세 가지 경우가 있습니다.

| 환경 | `UPDATE` 결과 | 비고 |
|---|---|---|
| 기존 전역 `REQUIRED`, readOnly=false | **성공** | 정상 write transaction |
| `readOnly=true`, 기본적인 JDBC hint 수준 | **성공할 수도 있음** | readOnly가 강제되지 않음 |
| DB transaction 자체를 `READ ONLY`로 강제 | **실패** | MariaDB가 DML 거부 |

MariaDB 자체는 `READ ONLY` transaction에서는 쓰기를 허용하지 않습니다. 

---

## 3. Spring `DataSourceTransactionManager`라면 더 구체적으로

Spring `DataSourceTransactionManager`의 기본 read-only 처리는 JDBC:

```java
connection.setReadOnly(true);
```

수준의 hint입니다.

반면:

```java
setEnforceReadOnly(true)
```

를 설정하면 Spring은 transaction 시작 시 DB에 명시적인 read-only transaction을 적용하도록 동작합니다. Spring 문서에서는 이것이 기본 `Connection.setReadOnly()`보다 강한 방식이며 DML을 엄격하게 금지하는 방식이라고 설명합니다. 

따라서:

### 기본 상태

```text
@Transactional(readOnly=true)
        ↓
Connection.setReadOnly(true)
        ↓
UPDATE
        ↓
Driver/DB 설정에 따라 실행 가능성이 있음
```

### enforceReadOnly=true

```text
@Transactional(readOnly=true)
        ↓
DB READ ONLY transaction
        ↓
UPDATE
        ↓
DB ERROR
        ↓
Exception
        ↓
ROLLBACK
```

입니다.

그래서 **현재 코드에 이런 혼합 메소드가 많다면 바로 `enforceReadOnly=true`를 적용하면 안 됩니다.**

운영 장애를 만들 가능성이 있습니다.

---

## 4. MariaDB Connector/J에서는 추가 위험도 있음

MariaDB Connector/J에서 `Connection.setReadOnly(true)`는 구성에 따라 단순한 hint 이상의 의미를 가질 수 있습니다.

특히 primary/replica 구성에서 MariaDB Connector/J는 `setReadOnly(true)`를 replica connection 선택에 이용할 수 있습니다. 공식 문서에서도 `setReadOnly(true)`를 replica 선택에 사용한다고 설명합니다. 

따라서 향후 DB가:

```text
Primary
   ↑
 C/U/D

Replica
   ↑
 SELECT
```

구조로 확장된다면,

```java
getOrder()
    SELECT
    UPDATE
```

같은 메소드는 단순한 코드 품질 문제가 아니라 **DB Routing 장애의 원인**이 될 수 있습니다.

예를 들어:

```text
getOrder()
   ↓
readOnly=true
   ↓
Replica connection 선택
   ↓
SELECT 성공
   ↓
UPDATE
   ↓
실패
```

할 수 있습니다.

현재 단일 Primary 환경에서 UPDATE가 우연히 성공한다고 해서 안전한 코드라고 판단하면 안 됩니다.

---

## 5. 더 중요한 예외: 외부 write transaction이 이미 존재하는 경우

다음 구조를 보겠습니다.

```java
public void updateOrderProcess(...) {

    orderDAO.updateOrder(...);

    orderService.getOrder(orderNo);
}
```

외부 메소드가:

```text
REQUIRED
readOnly=false
```

로 T1을 먼저 생성했다면:

```text
updateOrderProcess()
REQUIRED / read-write
       │
       │ T1 생성
       ↓
getOrder()
REQUIRED / readOnly=true
       │
       └── 기존 T1 참여
```

Spring의 기본 `REQUIRED` 정책에서는 내부 transaction의 `readOnly`, isolation, timeout 같은 속성이 기존 transaction에 참여할 때 **외부 physical transaction의 특성으로 대체될 수 있습니다.**

즉 기본값에서는 실제 transaction은:

```text
T1
REQUIRED
readOnly=false
```

이고,

```text
getOrder()
 ├─ SELECT
 └─ UPDATE
```

모두 T1에서 실행됩니다.

그래서 UPDATE가 정상 성공합니다.

Spring 문서도 `REQUIRED`가 기존 transaction에 참여할 때 기본적으로 outer transaction의 특성을 따르고, 내부의 read-only/isolation 등의 로컬 특성은 무시될 수 있다고 설명합니다. 

---

## 6. 호출 관계에 따라 결과가 달라지는 것이 가장 위험

동일한 `getOrder()`인데 호출 위치에 따라 결과가 달라질 수 있습니다.

### Case A — Controller에서 직접 호출

```text
Controller
    ↓
getOrder()
REQUIRED + readOnly=true
    ↓
새로운 read-only T1
    ↓
SELECT
UPDATE ← 환경에 따라 실패 가능
```

### Case B — 기존 write transaction에서 호출

```text
Controller
    ↓
OrderApplicationService
REQUIRED + readOnly=false
    ↓ T1
update()
    ↓
getOrder()
REQUIRED + readOnly=true
    ↓
기존 T1 참여
    ↓
SELECT
UPDATE ← 정상 실행 가능
```

즉 같은 메소드가:

```text
직접 호출 → UPDATE 실패

다른 CUD Service에서 호출 → UPDATE 성공
```

처럼 보일 수도 있습니다.

이런 코드는 운영에서 재현하기 상당히 어려운 장애를 만들기 때문에 정리하는 것이 좋습니다.

---

## 7. 현재 소스에 이런 코드가 많다면 `get* = readOnly`를 바로 적용하면 안 됨

이번 변경에서 가장 중요한 결론입니다.

현재 코드베이스에:

```java
getXXX() {
    selectXXX();
    updateXXX();
}
```

패턴이 많이 존재한다면 제가 권장한:

```xml
<tx:method name="get*" read-only="true"/>
```

를 **바로 운영에 일괄 적용해서는 안 됩니다.**

먼저 기존 Service를 분류해야 합니다.

| 분류 | 예 | 처리 |
|---|---|---|
| 순수 조회 | SELECT만 수행 | `readOnly=true` |
| 조회 + 통계 기록 | SELECT + UPDATE count | 별도 검토 |
| 조회 + 최근접속 갱신 | SELECT + UPDATE | 구조 개선 |
| 조회 + 상태 변경 | SELECT + UPDATE status | write transaction |
| 조회 + 이력 INSERT | SELECT + INSERT history | write transaction |
| 조회 + `FOR UPDATE` | locking SELECT | write transaction 성격 |
| 여러 조회만 수행 | SELECT + SELECT | readOnly 가능 |

---

## 8. 가장 좋은 개선 방법: Query와 Command 분리

질문의 코드는:

```java
public OrderVO getOrder(long orderNo) throws Exception {

    OrderVO order = orderDAO.selectOrder(orderNo);

    orderDAO.updateLastAccessDate(orderNo);

    return order;
}
```

메소드 이름만 보면 순수 Query인데 실제로는:

```text
Query + Command
```

입니다.

이것부터 분리하는 것이 가장 좋습니다.

```java
public OrderVO getOrder(long orderNo) throws Exception {
    return orderDAO.selectOrder(orderNo);
}

public void updateLastAccessDate(long orderNo) throws Exception {
    orderDAO.updateLastAccessDate(orderNo);
}
```

하지만 호출자가 단순히 두 개를 연속 호출하면 transaction 경계가 달라질 수 있으므로, 실제 업무 단위가 둘을 묶어야 한다면 Application Service를 둡니다.

```java
public OrderVO accessOrder(long orderNo) throws Exception {

    OrderVO order = orderService.getOrder(orderNo);

    orderService.updateLastAccessDate(orderNo);

    return order;
}
```

구조:

```text
OrderApplicationService.accessOrder()
            │
            ├── OrderService.getOrder()
            │       └── SELECT
            │
            └── OrderService.updateLastAccessDate()
                    └── UPDATE
```

`accessOrder()`가 하나의 비즈니스 단위라면 outer transaction을:

```text
REQUIRED / read-write
```

로 설정합니다.

이 구조가 가장 명확합니다.

---

## 9. 꼭 하나의 메소드로 유지해야 한다면

기존 영향 때문에 리팩터링이 어렵다면 최소한 이 메소드를 **read-write transaction으로 분류**해야 합니다.

예를 들어 이름을:

```java
public OrderVO accessOrder(long orderNo) throws Exception {
    OrderVO order = orderDAO.selectOrder(orderNo);
    orderDAO.updateLastAccessDate(orderNo);
    return order;
}
```

로 변경합니다.

XML에서는 `access*`가 일반 fallback:

```xml
<tx:method name="*"
           propagation="REQUIRED"
           rollback-for="Exception"/>
```

으로 들어가므로:

```text
REQUIRED + readOnly=false
```

가 됩니다.

또는 기존 메소드명을 유지해야 하면 구체적인 예외 패턴을 둘 수 있습니다.

```xml
<tx:method name="getOrder"
           propagation="REQUIRED"
           read-only="false"
           rollback-for="Exception"/>

<tx:method name="get*"
           propagation="REQUIRED"
           read-only="true"
           rollback-for="Exception"/>
```

그러면:

```text
getOrder       → read-write
getOrderList   → read-only
getOrderCount  → read-only
```

로 구분할 수 있습니다.

다만 이런 예외가 수십·수백 개로 늘어나면 XML 관리 방식 자체가 복잡해지므로 구조 개선이 우선입니다.

---

## 10. `REQUIRES_NEW`로 update를 분리하는 방법은 제한적으로만

`lastAccessDate`가 본 업무 transaction 성공 여부와 무관하게 기록해야 하는 데이터라면 별도의 Service Bean으로 분리할 수도 있습니다.

```java
public OrderVO getOrder(long orderNo) throws Exception {

    OrderVO order = orderDAO.selectOrder(orderNo);

    orderAccessService.updateLastAccessDate(orderNo);

    return order;
}
```

그리고:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void updateLastAccessDate(long orderNo) {
    orderDAO.updateLastAccessDate(orderNo);
}
```

실행:

```text
getOrder()
READ ONLY T1
    │
    ├── SELECT
    │
    └── updateLastAccessDate()
             ↓
          T1 suspend
             ↓
          WRITE T2
             ↓
          UPDATE + COMMIT
             ↓
          T1 resume
```

그러나 이것을 모든 접근일시 갱신 등에 남용하는 것은 추천하지 않습니다.

이유는:

- 추가 DB connection 사용 가능
- transaction 생성 비용 증가
- connection pool 압박
- 외부 transaction과 atomicity 분리
- T2는 commit됐는데 T1은 rollback될 수 있음
- 동시 접근 시 UPDATE 경쟁 증가

입니다.

즉 **업무적으로 독립 commit이 꼭 필요한 데이터만** 사용해야 합니다.

---

## 11. 최근 접근일시 같은 정보라면 더 좋은 설계도 검토

질문의:

```java
updateLastAccessDate(orderNo);
```

가 정말 단순한 조회 부가정보라면 매 조회마다 본 transaction 안에서 UPDATE할 필요가 있는지도 확인해야 합니다.

예:

```text
주문 조회
↓
SELECT ORDER
↓
UPDATE LAST_ACCESS_DATE
```

조회가 많아지면:

```text
SELECT 부하
+
UPDATE 부하
+
row lock
+
redo/undo
+
replication traffic
+
transaction log
```

까지 발생합니다.

즉 read-only 최적화를 적용하려는데 모든 조회에서 접근일자를 UPDATE한다면 최적화 효과가 크게 줄어듭니다.

업무 요구사항에 따라 다음을 고려할 수 있습니다.

| 데이터 성격 | 권장 방식 |
|---|---|
| 반드시 즉시 정확해야 함 | write transaction 유지 |
| 본 업무와 atomic해야 함 | 같은 `REQUIRED` |
| 실패해도 업무 영향 없음 | 별도 transaction/비동기 검토 |
| 초 단위 정확성 불필요 | 일정 시간 단위 갱신 검토 |
| 단순 조회 통계 | 로그/이벤트 기반 집계 검토 |

---

## 12. Migration 과정에서 `validateExistingTransaction` 활용

Spring `AbstractPlatformTransactionManager`에는:

```java
setValidateExistingTransaction(true)
```

가 있습니다.

기본값은 `false`이므로 기존 transaction에 참여할 때 내부 transaction의 read-only/isolation 불일치를 관대하게 처리합니다.

`true`를 사용하면 예를 들어:

```text
Outer
readOnly=true

        ↓

Inner
readOnly=false
REQUIRED
```

처럼 서로 맞지 않는 transaction 정의를 발견했을 때 참여를 거부하고 예외를 발생시키는 식으로 잘못된 transaction 설계를 조기에 발견하는 데 도움이 됩니다. 

다만 중요한 한계가 있습니다.

현재 코드처럼:

```java
getOrder()                  // Service
    ↓
orderDAO.updateXXX()        // DAO
```

이고 DAO가 별도의 transaction interceptor 대상이 아니라면 이것만으로 SQL UPDATE를 검출하는 것은 아닙니다.

즉:

> `validateExistingTransaction=true`는 SQL 검사기가 아니라 **서로 다른 transaction definition의 불일치를 검사하는 옵션**입니다.

---

## 13. 현재 시스템에서는 다음 순서로 변경하는 것이 가장 안전

제가 실제 운영 리팩터링을 한다면 다음 순서로 진행하겠습니다.

```text
[1] Service 메소드 전수 조사
        ↓
get/find/select/search/list/count
        ↓
내부 DAO 호출 확인

[2] 순수 R 분류
        ↓
SELECT만 존재
        ↓
readOnly=true 후보

[3] Mixed R/CUD 분류
        ↓
SELECT + INSERT/UPDATE/DELETE
        ↓
read-write 유지

[4] Mixed 메소드 구조 개선
        ↓
Query / Command 분리

[5] XML 적용
        ↓
순수 R만 readOnly=true

[6] 테스트
        ↓
Controller 직접 호출
다른 Service 내부 호출
예외/rollback
동시접근

[7] 이후 선택적으로
        ↓
enforceReadOnly
validateExistingTransaction
Replica routing
검토
```

---

## 권장 XML 1차 적용안

현재 코드에 혼합 메소드가 많다는 전제라면 **처음에는 안전한 메소드만 명시적으로 read-only화**하는 방법도 좋습니다.

```xml
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>

        <!-- 검증된 순수 조회 -->
        <tx:method name="select*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <tx:method name="find*"
                   propagation="REQUIRED"
                   read-only="true"
                   rollback-for="Exception"/>

        <!-- get*은 기존 코드 조사 완료 전에는 readOnly 적용 보류 -->
        <tx:method name="get*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <!-- C/U/D -->
        <tx:method name="insert*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="update*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <tx:method name="delete*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

        <!-- 안전장치 -->
        <tx:method name="*"
                   propagation="REQUIRED"
                   rollback-for="Exception"/>

    </tx:attributes>
</tx:advice>
```

다만 `select*`, `find*`에서도 CUD를 수행하는 기존 코드가 있다면 동일한 문제가 발생하므로 **이 역시 실제 코드 조사 후 적용해야 합니다.**

## 최종 판단

질문의 코드 자체는 현재 전역 `REQUIRED` 아래에서는 문제가 없습니다.

```java
getOrder()
  ├ SELECT
  └ UPDATE
```

둘 다 하나의 정상적인 **read-write REQUIRED transaction**으로 실행됩니다.

하지만 CRUD별로 transaction을 세분화해서:

```text
get* → REQUIRED + readOnly=true
```

로 바꾸는 순간 이 코드는 **리팩터링 대상**이 됩니다.

가장 권장하는 방향은:

```text
순수 Query
getOrder()
    → SELECT만

Command
updateLastAccessDate()
    → UPDATE

업무 조합이 필요하면
ApplicationService
    → 두 작업의 transaction 경계 결정
```

입니다.

특히 현재 코드베이스에 `get/find/select/list` 내부의 INSERT/UPDATE/DELETE가 많이 존재한다면 **CRUD transaction XML 변경보다 먼저 "조회형 이름을 가진 Service의 CUD 호출 전수조사"를 수행하는 것이 필수**입니다. 그렇지 않고 곧바로 `readOnly=true` 또는 `enforceReadOnly=true`를 적용하는 것은 운영 장애 가능성이 있습니다.