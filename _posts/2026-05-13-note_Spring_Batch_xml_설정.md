---
layout: single
title: "Spring Batch XML 방식에서 `faultTolerant` 설정하는 방법"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: 2026-05-13
last_modified_at: 2026-05-13 17:32:25
---

Spring Batch XML 설정에서는 Java 설정의 `.faultTolerant()`처럼 **별도 메서드를 직접 쓰지 않습니다.** 대신 `<chunk>`에 `skip-limit`, `retry-limit` 같은 속성과 예외 클래스 설정을 넣어서 **fault-tolerant step**으로 구성합니다. 이 동작은 공식 XML 예제들과 XML 파서용 `StepParserStepFactoryBean` API 설명으로 확인할 수 있습니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

## 1. 가장 기본 형태

```xml
<batch:step id="exampleStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="itemReader"
                     processor="itemProcessor"
                     writer="itemWriter"
                     commit-interval="100"
                     skip-limit="10"
                     retry-limit="3">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.batch.item.file.FlatFileParseException"/>
            </batch:skippable-exception-classes>
            <batch:retryable-exception-classes>
                <batch:include class="org.springframework.dao.DeadlockLoserDataAccessException"/>
            </batch:retryable-exception-classes>
        </batch:chunk>
        <batch:no-rollback-exception-classes>
            <batch:include class="org.springframework.batch.item.validator.ValidationException"/>
        </batch:no-rollback-exception-classes>
    </batch:tasklet>
</batch:step>
```

위처럼 설정하면 다음이 적용됩니다.

|설정|의미|
|---|---|
|`skip-limit="10"`|스킵 허용 한도|
|`<skippable-exception-classes>`|스킵 대상 예외|
|`retry-limit="3"`|재시도 횟수|
|`<retryable-exception-classes>`|재시도 대상 예외|
|`<no-rollback-exception-classes>`|롤백하지 않을 예외|
|공식 문서 기준으로 XML에서는 `skip-limit`과 `<skippable-exception-classes>`로 skip을, `retry-limit`과 `<retryable-exception-classes>`로 retry를, `<no-rollback-exception-classes>`로 rollback 제어를 설정합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))||

## 2. 핵심 포인트

### 2-1. `faultTolerant="true"` 같은 속성은 없다

XML에서는 보통 아래처럼 이해하면 됩니다.

- Java 설정: `.faultTolerant().skip(...).retry(...)`
    
- XML 설정: `<chunk skip-limit="..." retry-limit="..."> ... </chunk>`  
    즉, **skip/retry/rollback 정책을 선언하면 fault tolerance가 적용되는 구조**입니다. 이는 Spring Batch XML 파서가 설정된 속성에 따라 적절한 step builder를 선택한다는 API 설명과, 공식 XML 예제가 skip/retry를 직접 `<chunk>`에 선언하는 방식이라는 점에서 확인됩니다. 이 부분 중 “XML에 별도 `faultTolerant` 플래그가 없다”는 표현은 공식 예제 구조를 바탕으로 한 해석입니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))
    

### 2-2. skip-limit은 read/process/write 전체에 걸쳐 적용된다

스킵 카운트는 read/process/write별로 따로 집계되지만, **limit 자체는 전체 스킵 합계에 적용**됩니다. 예를 들어 `skip-limit="10"`이면 10번째까지는 허용되고, **11번째 스킵에서 step이 실패**합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

### 2-3. writer 예외는 기본적으로 rollback 된다

공식 문서 기준으로, retry/skip 여부와 상관없이 `ItemWriter`에서 발생한 예외는 기본적으로 트랜잭션을 rollback합니다. 반면 skip이 설정된 경우 `ItemReader` 예외는 rollback을 유발하지 않습니다. 필요하면 `<no-rollback-exception-classes>`로 예외별 롤백 제외를 걸 수 있습니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/controlling-rollback.html "Controlling Rollback :: Spring Batch Reference"))

### 2-4. `ItemProcessor`는 가능하면 idempotent하게 작성해야 한다

fault-tolerant step에서는 rollback 후 캐시된 아이템이 다시 처리될 수 있으므로, 공식 문서는 `ItemProcessor`를 **idempotent**하게 구현할 것을 권장합니다. 즉, 입력 객체 자체를 바꾸기보다 결과 객체를 만들어 반환하는 형태가 안전합니다. ([Home](https://docs.spring.io/spring-batch/reference/processor.html "Item processing :: Spring Batch Reference"))

## 3. 실무에서 가장 많이 쓰는 XML 예제

### 3-1. 파일 읽기 중 일부 라인 오류는 건너뛰기

```xml
<batch:step id="importStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="flatFileItemReader"
                     processor="itemProcessor"
                     writer="jdbcItemWriter"
                     commit-interval="500"
                     skip-limit="50">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.batch.item.file.FlatFileParseException"/>
            </batch:skippable-exception-classes>
        </batch:chunk>
    </batch:tasklet>
</batch:step>
```

적합한 경우:

- 대량 파일 적재
    
- 일부 불량 레코드는 로그 남기고 계속 진행
    

### 3-2. DB deadlock은 재시도

```xml
<batch:step id="syncStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="jdbcPagingItemReader"
                     writer="jdbcBatchItemWriter"
                     commit-interval="100"
                     retry-limit="3">
            <batch:retryable-exception-classes>
                <batch:include class="org.springframework.dao.DeadlockLoserDataAccessException"/>
            </batch:retryable-exception-classes>
        </batch:chunk>
    </batch:tasklet>
</batch:step>
```

적합한 경우:

- 일시적 DB lock 충돌
    
- 재시도 시 성공 가능성이 있는 오류
    

### 3-3. 검증 예외는 실패시키지 않고 롤백도 막기

```xml
<batch:step id="validateStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="itemReader"
                     processor="validatingProcessor"
                     writer="itemWriter"
                     commit-interval="50"
                     skip-limit="100">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.batch.item.validator.ValidationException"/>
            </batch:skippable-exception-classes>
        </batch:chunk>
        <batch:no-rollback-exception-classes>
            <batch:include class="org.springframework.batch.item.validator.ValidationException"/>
        </batch:no-rollback-exception-classes>
    </batch:tasklet>
</batch:step>
```

적합한 경우:

- 유효성 오류 데이터만 제외하고 계속 처리
    
- writer 동작 전 검증 예외를 굳이 rollback시키고 싶지 않을 때
    

## 4. JMS 같은 트랜잭션 큐 Reader를 쓸 때

Reader가 JMS queue처럼 **트랜잭션 자원** 위에 있으면 rollback 시 메시지가 다시 큐로 돌아갈 수 있습니다. 이 경우 Spring Batch는 XML에서 `is-reader-transactional-queue="true"` 설정을 제공합니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/controlling-rollback.html "Controlling Rollback :: Spring Batch Reference"))

```xml
<batch:step id="queueStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="jmsItemReader"
                     writer="itemWriter"
                     commit-interval="10"
                     is-reader-transactional-queue="true"/>
    </batch:tasklet>
</batch:step>
```

## 5. 권장 설정 순서

|순서|권장 내용|
|---|---|
|1|먼저 어떤 예외를 `skip`할지 결정|
|2|재시도 가능한 예외만 `retry` 대상으로 분리|
|3|`writer` 예외 중 rollback 제외가 필요한 것만 `no-rollback` 설정|
|4|`processor`는 재처리 안전하게 구현|
|5|skip/retry 발생 건은 listener나 별도 로그로 남김|

## 6. 실무 판단 기준

|상황|추천|
|---|---|
|입력 파일의 일부 데이터 불량 허용|`skip-limit` + `skippable-exception-classes`|
|DB deadlock, 일시 네트워크 오류|`retry-limit` + `retryable-exception-classes`|
|검증 예외는 실패로 보지 않음|`skip` + 필요 시 `no-rollback`|
|JMS/큐 기반 reader|`is-reader-transactional-queue="true"` 검토|

## 7. 한 줄 정리

Spring Batch XML에서는 `faultTolerant()`를 직접 쓰는 것이 아니라, **`<chunk>`에 skip/retry/no-rollback 설정을 선언하는 방식으로 fault tolerance를 구성**하면 됩니다. 공식적으로는 `skip-limit`, `retry-limit`, 각 예외 클래스 블록, 그리고 필요 시 `is-reader-transactional-queue`를 조합해서 설정합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))  
다음 답변에서는 원하시는 방식으로 `FlatFileItemReader + JdbcBatchItemWriter` 기준의 **완전한 XML 샘플**을 바로 작성해드릴 수 있습니다.


정확도: 96%

# 대상 설정의 의미

아래 설정은 **chunk 기반 Step**에서 `itemReader`로 읽고, `validatingProcessor`로 검증한 뒤, `itemWriter`로 쓰는 구조입니다. `transaction-manager`는 처리 중 트랜잭션 시작/커밋에 사용할 `PlatformTransactionManager`를 지정하고, `commit-interval="50"`은 **50건 단위로 하나의 chunk/트랜잭션 경계**를 만든다는 의미입니다. `skip-limit="100"`과 `skippable-exception-classes`가 있으면 skip 기반의 fault-tolerant 동작이 활성화되며, skip은 read/process/write 전체에 대해 합산됩니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/configuring.html?utm_source=chatgpt.com "Configuring a Step :: Spring Batch Reference"))

```xml
<batch:tasklet transaction-manager="transactionManager">
    <batch:chunk reader="itemReader"
                 processor="validatingProcessor"
                 writer="itemWriter"
                 commit-interval="50"
                 skip-limit="100">
        <batch:skippable-exception-classes>
            <batch:include class="org.springframework.batch.item.validator.ValidationException"/>
        </batch:skippable-exception-classes>
    </batch:chunk>
    <batch:no-rollback-exception-classes>
        <batch:include class="org.springframework.batch.item.validator.ValidationException"/>
    </batch:no-rollback-exception-classes>
</batch:tasklet>
```

# 이 설정이 실제로 하는 일

`ValidatingItemProcessor`는 입력을 수정하지 않고 검증만 수행하는 processor이며, 검증 실패 시 기본적으로 `ValidationException`을 다시 던져 **해당 item을 skip 대상으로 표시**합니다. 다만 `setFilter(true)`를 주면 예외를 던지지 않고 `null`을 반환하여 **skip이 아니라 filter**로 처리됩니다. Spring Batch는 `ItemProcessor`가 `null`을 반환하면 해당 item을 writer 전달 목록에 넣지 않습니다. ([Home](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/validator/ValidatingItemProcessor.html "ValidatingItemProcessor (Spring Batch 6.0.3 API)"))  
즉, 현재 XML은 다음 의미입니다.

1. 1건을 읽는다.
    
2. `validatingProcessor`가 검증한다.
    
3. 검증 실패로 `ValidationException`이 발생하면, 그 건은 skip한다.
    
4. 이런 skip이 누적 100건까지는 허용된다.
    
5. 101번째 skip이 발생하면 Step이 실패한다. ([Home](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/validator/ValidatingItemProcessor.html "ValidatingItemProcessor (Spring Batch 6.0.3 API)"))
    

# 태그/옵션별 상세 설명

|구성|의미|실무 해석|
|---|---|---|
|`<batch:tasklet transaction-manager="transactionManager">`|이 Step 처리에 사용할 트랜잭션 매니저를 지정|DB write가 있는 chunk step에서는 보통 명시적으로 사용하는 편이 안전|
|`<batch:chunk ...>`|reader → processor → writer로 이어지는 chunk 처리 정의|가장 일반적인 Spring Batch step 형태|
|`reader="itemReader"`|입력 데이터를 1건씩 공급하는 `ItemReader` bean 참조|파일/DB/API reader 등|
|`processor="validatingProcessor"`|각 item을 검증/가공하는 `ItemProcessor` bean 참조|여기서는 “검증 전용 processor” 성격|
|`writer="itemWriter"`|가공 완료 item들을 쓰는 `ItemWriter` bean 참조|DB insert/update, 파일 write 등|
|`commit-interval="50"`|50건 단위로 write 후 commit|성능과 rollback 영향 범위를 함께 결정|
|`skip-limit="100"`|허용 가능한 총 skip 수|read/process/write skip의 합계 기준|
|`<skippable-exception-classes>`|skip 대상으로 인정할 예외 목록|업무상 “버리고 계속 가도 되는 예외”만 넣어야 함|
|`<include class="...ValidationException"/>`|`ValidationException`과 그 하위 예외를 skip 대상으로 포함|검증 실패 레코드를 실패가 아닌 제외 대상으로 처리|
|`<no-rollback-exception-classes>`|해당 예외 발생 시 rollback하지 않도록 분류|불필요한 chunk rollback을 줄이는 용도|
|`<include class="...ValidationException"/>`|`ValidationException` 발생 시 rollback 제외|검증 실패가 “시스템 오류”가 아니라 “불량 데이터”라는 전제에 적합|
|위 의미 중 `transaction-manager`, `reader`, `writer`, `processor`의 기본 역할과 XML 속성 의미는 Spring Batch step 구성 문서에, `commit-interval`은 chunk 단위 커밋 문서에, `skip-limit`과 skippable exception은 skip 문서에, `no-rollback-exception-classes`는 rollback 제어 문서에 나와 있습니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring.html?utm_source=chatgpt.com "Configuring a Step :: Spring Batch Reference"))|||

# 특히 중요한 옵션 3개

## 1. `commit-interval="50"`

50건을 모아 한 번에 writer 호출 후 commit합니다. commit interval이 작으면 transaction 오버헤드는 늘고, 너무 크면 rollback 시 되돌려야 할 범위가 커집니다. Spring Batch는 chunk를 트랜잭션 범위 안에서 처리하고, commit interval마다 커밋합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/commit-interval.html?utm_source=chatgpt.com "The Commit Interval :: Spring Batch Reference"))

### 실무 주의점

|항목|주의점|
|---|---|
|너무 큼|writer 실패 시 재처리/rollback 범위가 커짐|
|너무 작음|DB round-trip, commit 비용 증가|
|권장 접근|성능만 보지 말고 “재처리 비용”과 “장애 시 복구 편의성”까지 같이 봐야 함|

## 2. `skip-limit="100"`

skip은 read/process/write 각각에서 별도로 집계되지만, **limit 적용은 전체 합계** 기준입니다. 그리고 **100번째까지 허용**, **101번째에서 Step 실패**입니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

### 실무 주의점

|항목|주의점|
|---|---|
|숫자를 크게만 잡는 경우|실패 데이터가 대량인데도 배치가 성공처럼 보일 수 있음|
|숫자를 너무 작게 잡는 경우|현실적인 데이터 오염을 감당하지 못하고 자주 실패함|
|권장 접근|총 건수 대비 허용 불량률 기준으로 산정하는 것이 좋음|

## 3. `no-rollback-exception-classes`

Spring Batch 문서는 기본적으로 **`ItemWriter` 예외는 rollback**, skip이 설정된 경우 **`ItemReader` 예외는 rollback하지 않음**이라고 설명합니다. 그리고 rollback하지 않을 예외 목록을 따로 설정할 수 있다고 안내합니다. 현재 설정은 `ValidationException`을 그 목록에 포함한 것입니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/controlling-rollback.html "Controlling Rollback :: Spring Batch Reference"))

### 실무 주의점

|항목|주의점|
|---|---|
|남용 금지|업무 데이터 정합성에 영향을 줄 수 있는 예외까지 no-rollback 처리하면 위험|
|적합한 경우|정말 “해당 건만 버리면 되고 현재 트랜잭션 전체를 되돌릴 이유가 없는 경우”|
|부적합한 경우|DB 제약조건 위반, writer partial failure, 외부 연계 반영 불일치 같은 정합성 이슈|

# 현재 설정의 실무적 의미

이 설정은 본질적으로 `ValidationException`을 **시스템 장애가 아니라 데이터 품질 문제**로 취급하는 설계입니다. 즉 “잘못된 데이터 몇 건 때문에 전체 배치를 실패시키지 말고, 일정 한도까지는 건너뛰고 진행하자”는 정책입니다. Spring Batch 문서도 skip은 데이터 의미를 이해하는 사람이 결정해야 하며, 금융 데이터처럼 정확성이 절대적인 경우에는 skippable하지 않을 수 있다고 설명합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

# 실무에서 가장 많이 하는 오해

## 오해 1. `skip-limit=100`이면 process에서만 100건까지 허용된다

아닙니다. read/process/write 전체 skip 합계 기준입니다. 예를 들어 read 30, process 40, write 30이면 총 100이고, 다음 skip에서 실패합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

## 오해 2. `ValidatingItemProcessor`는 무조건 skip만 한다

아닙니다. `setFilter(true)`면 예외를 던지지 않고 `null`을 반환하여 **filter**로 처리합니다. 이 경우 skip count를 쓰지 않고 writer 대상에서만 제외됩니다. ([Home](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/validator/ValidatingItemProcessor.html "ValidatingItemProcessor (Spring Batch 6.0.3 API)"))

## 오해 3. `no-rollback-exception-classes`만 넣으면 다 안전하다

아닙니다. rollback을 막는 것은 성능이나 재처리 효율에는 도움이 될 수 있지만, 잘못 쓰면 **부분 반영/정합성 문제를 감추는 방향**으로 갈 수 있습니다. 이 부분은 공식 문서의 rollback 제어 원칙을 바탕으로 한 실무 판단입니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/controlling-rollback.html "Controlling Rollback :: Spring Batch Reference"))

# 이 설정에서 특히 조심해야 할 점

## 1. `ValidationException`의 의미를 엄격히 제한해야 함

`ValidationException`은 “업무상 버려도 되는 불량 데이터”에만 써야 합니다. DB 연결 실패, SQL 오류, 네트워크 오류, 매퍼 오류 같은 **시스템 예외**를 여기에 섞으면 안 됩니다. 현재 구성이 그 예외를 skip/no-rollback 대상으로 보기 때문입니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

## 2. `processor`는 idempotent하게 작성해야 함

Spring Batch는 fault-tolerant step에서 rollback이 발생하면 읽어 둔 item이 다시 처리될 수 있으므로, `ItemProcessor`는 입력 객체 자체를 변경하지 않는 등 **idempotent**하게 구현할 것을 권장합니다. ([Home](https://docs.spring.io/spring-batch/reference/processor.html "Item processing :: Spring Batch Reference"))

### 실무 예

|나쁜 예|좋은 예|
|---|---|
|processor 내부에서 입력 객체 상태를 누적 변경|새 결과 객체를 만들어 반환|
|processor에서 외부 API 호출 후 상태 저장|검증/변환만 수행하고 side effect는 writer 또는 별도 단계로 분리|

## 3. skip된 건은 반드시 별도 기록 필요

Spring Batch 문서는 bad record를 보통 로그로 남기며, `SkipListener`의 대표적인 용도가 skip된 item 기록이라고 설명합니다. 또한 `SkipListener`는 item당 1번만 호출되고, 커밋 직전에 호출됩니다. 운영 배치에서는 skip 데이터를 별도 테이블이나 파일로 남겨야 사후 정정이 가능합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

## 4. “검증 실패”라면 skip보다 filter가 더 맞는 경우가 많음

정말 오류가 아니라 “대상 제외”라면 `ValidatingItemProcessor#setFilter(true)` 또는 사용자 정의 processor에서 `null` 반환이 더 자연스럽습니다. skip은 보통 “문제가 있는 데이터” 의미이고, filter는 “정상적으로 읽었지만 이번 배치 대상은 아님”에 가깝습니다. ([Home](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/validator/ValidatingItemProcessor.html "ValidatingItemProcessor (Spring Batch 6.0.3 API)"))

# 현재 XML을 실무 관점에서 평가

|평가 항목|판단|
|---|---|
|구조 자체|정상적인 fault-tolerant XML 설정|
|적합한 상황|파일/인터페이스 배치에서 일부 불량 데이터 허용 시 적합|
|위험 포인트|`ValidationException` 범위를 넓게 쓰면 장애 은폐 가능|
|추가 권장|`SkipListener` 등록, skip 전용 로그/에러 테이블, skip 허용률 모니터링|
|재검토 포인트|정말 skip이 맞는지, 아니면 filter가 맞는지 구분 필요|

# 권장 운영 기준

|상황|권장 방식|
|---|---|
|검증 실패가 “오류 데이터”|현재 방식 유지 가능|
|검증 실패가 “단순 제외 조건”|`setFilter(true)` 또는 `null` 반환 방식 검토|
|writer가 DB batch insert 수행|`no-rollback` 남용 금지, writer 예외는 보수적으로 처리|
|대량 운영 배치|skip 건수, skip률, 상위 오류 유형을 모니터링 지표로 관리|

# 한 줄 결론

이 XML은 **“검증 실패 데이터는 최대 100건까지 건너뛰고, 그 예외로 chunk 전체를 불필요하게 rollback하지 않겠다”**는 정책입니다. 다만 실무에서는 `ValidationException`을 **정말 버려도 되는 데이터 오류에만 제한**하고, skip된 건은 반드시 별도 기록하며, 단순 제외라면 skip보다 **filter 방식**이 더 적합한지 먼저 판단하는 것이 안전합니다. ([Home](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/validator/ValidatingItemProcessor.html "ValidatingItemProcessor (Spring Batch 6.0.3 API)"))  
원하시면 다음 답변에서 이 설정을 기준으로 **Spring Batch XML 전체 Step 예제 + SkipListener + 에러파일 적재 예제**까지 바로 정리하겠습니다.

정확도: 95%

# Spring Batch XML에서 `writer` insert 시 `DuplicateKeyException` 무시 설정

`DuplicateKeyException`은 Spring에서 **PK/UNIQUE 제약 위반** 시 발생하는 예외입니다. Spring Batch XML에서는 이를 “완전 무시”라기보다 **skip 처리**로 설정합니다. 즉, 해당 건은 건너뛰고 Step은 계속 진행하되, `skip-limit`를 초과하면 Step이 실패합니다. 또한 Spring Batch는 기본적으로 `ItemWriter`에서 발생한 예외에 대해 **트랜잭션 rollback**을 수행합니다. ([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/dao/DuplicateKeyException.html "DuplicateKeyException (Spring Framework 7.0.7 API)"))

## 1. 기본 XML 설정

```xml
<batch:step id="importStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="itemReader"
                     processor="itemProcessor"
                     writer="itemWriter"
                     commit-interval="50"
                     skip-limit="100">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.dao.DuplicateKeyException"/>
            </batch:skippable-exception-classes>
        </batch:chunk>
    </batch:tasklet>
</batch:step>
```

위 설정의 의미는 아래와 같습니다. `writer`에서 `org.springframework.dao.DuplicateKeyException`이 발생하면 해당 item은 skip 대상이 되고, 전체 skip 누적이 `100`건을 넘기기 전까지는 Step이 계속 진행됩니다. Spring Batch XML의 skip 설정은 `<chunk ... skip-limit="...">`와 `<skippable-exception-classes>`로 구성합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))

## 2. 옵션 의미

|설정|의미|
|---|---|
|`commit-interval="50"`|50건 단위로 chunk 트랜잭션을 구성|
|`skip-limit="100"`|read/process/write 전체에서 누적 skip 100건까지 허용|
|`<skippable-exception-classes>`|skip 허용 예외 목록|
|`DuplicateKeyException`|중복 키 예외 발생 시 해당 건을 skip|
|`skip-limit`는 writer만 따로 세는 값이 아니라 **read/process/write 전체 skip 합계 기준**입니다. Spring Batch 문서 예제도 XML에서 동일한 방식으로 skip 예외를 선언합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html "Configuring Skip Logic :: Spring Batch Reference"))||

## 3. `no-rollback`까지 같이 넣을지 여부

설정은 가능하지만, `DuplicateKeyException`에 대해서는 **기본적으로 바로 넣지 않는 편이 안전**합니다.

```xml
<batch:step id="importStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="itemReader"
                     processor="itemProcessor"
                     writer="itemWriter"
                     commit-interval="50"
                     skip-limit="100">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.dao.DuplicateKeyException"/>
            </batch:skippable-exception-classes>
        </batch:chunk>
        <batch:no-rollback-exception-classes>
            <batch:include class="org.springframework.dao.DuplicateKeyException"/>
        </batch:no-rollback-exception-classes>
    </batch:tasklet>
</batch:step>
```

Spring Batch는 기본적으로 `ItemWriter` 예외가 발생하면 rollback합니다. 문서도 `ItemWriter` 예외는 기본 rollback 대상이라고 설명하며, `no-rollback-exception-classes`는 정말 rollback이 불필요한 경우에만 별도로 지정하도록 안내합니다. `DuplicateKeyException`은 Spring에서 `NonTransientDataAccessException` 계열로 분류되므로, 일반적으로는 “재시도보다 skip 또는 실패 처리” 쪽이 더 자연스럽습니다. ([Home](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/step.html "Configuring a Step"))

## 4. 실무 권장안

|상황|권장|
|---|---|
|중복이 드물고, 들어오면 해당 건만 버리면 됨|`skippable-exception-classes`만 설정|
|중복이 자주 발생함|배치 skip보다 SQL 자체를 중복 허용 형태로 재설계 검토|
|writer가 JDBC batch insert이고 정합성이 중요함|`no-rollback`은 신중하게 검토|
|중복 건을 사후 확인해야 함|`SkipListener`로 별도 로그/에러 테이블 적재|
|핵심은 `DuplicateKeyException`을 **retry 대상**으로 보기보다는 **skip 대상**으로 두는 것입니다. Spring Batch 문서도 retry는 deadlock 같은 일시적 예외에 더 적합하다고 설명합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/retry-logic.html "Configuring Retry Logic :: Spring Batch Reference"))||

## 5. 가장 실무적인 결론

중복 insert를 XML 설정만으로 넘기려면 아래 방식이 가장 보수적입니다.

```xml
<batch:step id="importStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="itemReader"
                     writer="itemWriter"
                     commit-interval="50"
                     skip-limit="100">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.dao.DuplicateKeyException"/>
            </batch:skippable-exception-classes>
        </batch:chunk>
    </batch:tasklet>
</batch:step>
```

즉, **우선은 skip만 설정**하고, `no-rollback`은 writer의 실제 동작과 데이터 정합성을 충분히 확인한 뒤에만 추가하는 것이 안전합니다. Spring Batch 기준으로 writer 예외는 기본 rollback이기 때문입니다. ([Home](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/step.html "Configuring a Step"))  
원하시면 다음 답변에서 `JdbcBatchItemWriter` 기준으로 **완전한 XML bean 설정 예제**까지 이어서 정리하겠습니다.

정확도: 95%

# `DuplicateKeyException` 오류 진단과 실무 처리 방안

현재 오류는 `goodsCtgryRelMvDtStep`에서 `MyBatisBatchItemWriter.write()`가 `flushStatements()`를 수행하는 시점에 `PK_GOODS_CTGRY_REL` 기본키 중복으로 실패한 케이스입니다. 즉, **reader/process 단계가 아니라 writer의 batch flush 단계에서 중복 키가 터졌고**, 그 결과 Step 전체가 실패했습니다. 로그상 최상위 예외도 `org.springframework.dao.DuplicateKeyException`으로 번역되어 올라왔습니다.  
Spring Batch는 skip을 설정할 수 있지만, 기본적으로 `ItemWriter`에서 발생한 예외는 Step 트랜잭션을 rollback합니다. 또 `MyBatisBatchItemWriter`는 `SqlSessionTemplate`의 batch 기능을 이용해 chunk 단위로 묶어 실행하므로, writer 단계에서의 중복키 오류는 **한 건만 조용히 무시되는 구조가 아니라 chunk flush 실패로 표면화**되는 것이 기본 동작입니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/controlling-rollback.html?utm_source=chatgpt.com "Controlling Rollback :: Spring Batch Reference"))

## 1. 현재 케이스에 대한 가장 현실적인 판단

|항목|판단|
|---|---|
|오류 위치|`writer` 단계|
|오류 성격|`PK_GOODS_CTGRY_REL` 중복 insert|
|현재 동작|batch flush 시 Step 실패|
|skip 적용 가능 여부|가능|
|skip만으로 충분한가|대부분의 경우 **임시 대응**에는 가능하지만 **근본 해결은 아님**|
|핵심은 이것입니다.||
|**중복이 “정상 재실행으로 인해 생기는 허용 가능한 중복”인지**, 아니면 **소스 데이터/조인/전처리 오류로 인해 생긴 비정상 중복인지**를 먼저 구분해야 합니다. Spring Batch 공식 문서도 skip 여부는 단순 기술 설정이 아니라 **데이터 의미를 이해한 뒤 결정해야 한다**고 설명합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html?utm_source=chatgpt.com "Configuring Skip Logic :: Spring Batch Reference"))||

## 2. 실무용 skip 처리 예제(XML)

### 2-1. Step XML

```xml
<batch:step id="goodsCtgryRelMvDtStep">
    <batch:tasklet transaction-manager="transactionManager">
        <batch:chunk reader="goodsCtgryRelReader"
                     processor="goodsCtgryRelProcessor"
                     writer="goodsCtgryRelWriter"
                     commit-interval="100"
                     skip-limit="1000">
            <batch:skippable-exception-classes>
                <batch:include class="org.springframework.dao.DuplicateKeyException"/>
            </batch:skippable-exception-classes>
        </batch:chunk>
        <batch:listeners>
            <batch:listener ref="goodsCtgryRelSkipListener"/>
        </batch:listeners>
    </batch:tasklet>
</batch:step>
```

이 설정은 `DuplicateKeyException`이 발생한 item을 skip 대상으로 분류합니다. 다만 writer 예외는 기본 rollback 대상이므로, 실제로는 chunk rollback 후 Spring Batch가 skip 정책에 따라 재처리를 진행하는 흐름으로 이해하는 것이 맞습니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html?utm_source=chatgpt.com "Configuring Skip Logic :: Spring Batch Reference"))

### 2-2. SkipListener 예제

```java
public class GoodsCtgryRelSkipListener implements SkipListener<GoodsCtgryRel, GoodsCtgryRel> {
    private static final Logger log = LoggerFactory.getLogger(GoodsCtgryRelSkipListener.class);
    @Override
    public void onSkipInRead(Throwable t) {
        log.warn("READ SKIP: {}", t.getMessage(), t);
    }
    @Override
    public void onSkipInProcess(GoodsCtgryRel item, Throwable t) {
        log.warn("PROCESS SKIP: goodsNo={}, ctgryId={}, reason={}",
                item.getGoodsNo(), item.getCtgryId(), t.getMessage(), t);
    }
    @Override
    public void onSkipInWrite(GoodsCtgryRel item, Throwable t) {
        log.warn("WRITE SKIP: goodsNo={}, ctgryId={}, reason={}",
                item.getGoodsNo(), item.getCtgryId(), t.getMessage(), t);
    }
}
```

### 2-3. bean 등록

```xml
<bean id="goodsCtgryRelSkipListener"
      class="com.example.batch.listener.GoodsCtgryRelSkipListener"/>
```

## 3. skip 방식의 장단점

|구분|장점|단점|
|---|---|---|
|운영 즉시성|배치 중단을 막고 계속 진행 가능|중복 원인을 가린 채 지나갈 수 있음|
|개발 난이도|XML 설정만으로 빠르게 적용 가능|writer batch 구조에서는 rollback/재처리 비용이 남음|
|가시성|SkipListener로 남기면 추적 가능|로그 안 남기면 “성공했지만 일부 누락” 상태가 됨|
|정합성|허용 가능한 중복에는 유효|중복이 비정상 데이터라면 근본 문제 미해결|

## 4. `no-rollback-exception-classes`를 같이 넣는 것은 권장하는가

현재 케이스에서는 **기본적으로 비권장**입니다. Spring Batch는 writer 예외를 기본 rollback 대상으로 보고 있으며, 특히 지금처럼 `MyBatisBatchItemWriter`가 batch flush로 묶어서 쓰는 구조에서는 어느 시점까지 실제 반영되었는지에 대한 보수적 접근이 필요합니다. 따라서 `DuplicateKeyException`에 대해 바로 `no-rollback`을 넣기보다, **일단 rollback + skip 정책**으로 운영하는 편이 안전합니다. ([Home](https://docs.spring.io/spring-batch/reference/5.1/step/chunk-oriented-processing/controlling-rollback.html?utm_source=chatgpt.com "Controlling Rollback :: Spring Batch Reference"))

## 5. 이 경우의 더 좋은 방안들

### 5-1. 1순위: SQL 자체를 idempotent하게 변경

중복이 “재실행 시 다시 들어올 수 있는 정상 중복”이라면, 가장 좋은 방식은 **writer에서 예외를 발생시키지 않도록 SQL을 바꾸는 것**입니다. MariaDB는 `INSERT ... ON DUPLICATE KEY UPDATE`를 공식적으로 지원합니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/data-manipulation/inserting-loading-data/insert-on-duplicate-key-update?utm_source=chatgpt.com "INSERT ON DUPLICATE KEY UPDATE | Server - MariaDB"))

#### 예시

```xml
<insert id="insertGoodsCtgryRel">
    INSERT INTO GOODS_CTGRY_REL (
        GOODS_NO,
        CTGRY_ID,
        REG_ID,
        REG_DTM
    ) VALUES (
        #{goodsNo},
        #{ctgryId},
        #{regId},
        NOW()
    )
    ON DUPLICATE KEY UPDATE
        REG_ID = VALUES(REG_ID),
        REG_DTM = VALUES(REG_DTM)
</insert>
```

#### 장점

|장점|설명|
|---|---|
|배치 안정성|writer 예외 자체를 줄임|
|재실행 안전성|같은 배치를 다시 돌려도 실패 확률이 낮음|
|운영 단순화|skip 건수 관리보다 운영이 단순|

#### 주의점

|주의점|설명|
|---|---|
|업데이트 의미 확인|기존값을 덮어써도 되는지 업무 확인 필요|
|트리거/감사 컬럼 영향|update 발생으로 `upd_dtm` 등 변경 가능|
|다중 unique 인덱스|MariaDB는 둘 이상의 unique 인덱스가 있는 경우 첫 번째 매칭 인덱스만 update 대상으로 동작할 수 있어 주의 필요|
|즉, **“이미 있으면 그대로 두거나 필요한 값만 갱신”이 업무적으로 맞다면 가장 실무적**입니다. ([MariaDB](https://mariadb.com/docs/server/reference/sql-statements/data-manipulation/inserting-loading-data/insert-on-duplicate-key-update?utm_source=chatgpt.com "INSERT ON DUPLICATE KEY UPDATE \| Server - MariaDB"))||

### 5-2. 2순위: `NOT EXISTS` / `LEFT JOIN` 방식으로 선제 차단

중복 시 update조차 원하지 않고, **없는 것만 insert**해야 한다면 `INSERT ... SELECT ... WHERE NOT EXISTS` 또는 `LEFT JOIN ... WHERE target.pk IS NULL` 방식이 더 명확할 수 있습니다.

#### 예시

```xml
<insert id="insertGoodsCtgryRel">
    INSERT INTO GOODS_CTGRY_REL (GOODS_NO, CTGRY_ID, REG_ID, REG_DTM)
    SELECT
        #{goodsNo},
        #{ctgryId},
        #{regId},
        NOW()
    FROM DUAL
    WHERE NOT EXISTS (
        SELECT 1
        FROM GOODS_CTGRY_REL
        WHERE GOODS_NO = #{goodsNo}
          AND CTGRY_ID = #{ctgryId}
    )
</insert>
```

#### 장점/단점

|구분|내용|
|---|---|
|장점|기존 데이터 수정 없이 신규만 넣음|
|단점|동시성 높은 환경에서는 최종 방어선으로 PK/UNIQUE는 여전히 필요|
|배치가 단일 Step/단일 인스턴스로만 돌고 동시 insert 경쟁이 적다면 꽤 안정적입니다. 다만 다중 배치 동시 실행 가능성이 있으면 PK는 계속 필요합니다.||

### 5-3. 3순위: Reader/Processor에서 사전 중복 제거

중복 원인이 target 테이블이 아니라 **입력 데이터 자체**라면, writer skip보다 앞단 제거가 맞습니다. 특히 현재 키 값이 `3723968-003001002002` 같은 복합 업무키라면, 소스 조인 결과 중복 가능성을 먼저 의심해야 합니다. 실제 로그도 특정 PK 하나가 중복으로 반복 충돌하는 형태입니다.

#### 실무 방식

|방식|설명|
|---|---|
|Reader SQL에 `DISTINCT`|가장 간단|
|Reader SQL에 `GROUP BY`|조인 중복 제거에 유효|
|Processor에서 `Set` dedupe|당일 입력량이 작을 때 유효|
|Staging 테이블 적재 후 dedupe|대량 배치에 가장 안정적|

#### 추천 SQL 점검 예

```sql
SELECT GOODS_NO, CTGRY_ID, COUNT(*)
FROM 소스조회결과
GROUP BY GOODS_NO, CTGRY_ID
HAVING COUNT(*) > 1;
```

이 쿼리로 **소스에서 이미 중복이 만들어지는지** 먼저 확인하는 것이 좋습니다.

## 6. 무엇이 “최선”인가

### 케이스별 권장 순위

|상황|최선의 방안|
|---|---|
|배치 재실행 시 같은 데이터가 다시 들어오는 것이 정상|`ON DUPLICATE KEY UPDATE`|
|기존 데이터는 건드리면 안 되고 신규만 넣어야 함|`NOT EXISTS` 기반 insert|
|중복이 소스 조인 오류/원천 데이터 문제|Reader SQL 정제 또는 staging dedupe|
|운영 장애를 우선 막아야 하는 긴급 상황|Spring Batch skip + SkipListener|
|즉, **현재 장애를 오늘 막는 임시 대응은 skip**, **장기적으로 가장 좋은 해법은 SQL/입력 구조를 idempotent하게 바꾸는 것**입니다.||

## 7. 현재 환경에서의 추천안

현재 스택은 `Spring Batch 4.3.6 + MyBatisBatchItemWriter + MariaDB`이고, 오류가 writer flush 시점에서 발생하고 있습니다. 이런 구조에서는 아래 순서를 권장합니다. ([MyBatis](https://mybatis.org/spring/apidocs/org/mybatis/spring/batch/MyBatisBatchItemWriter.html?utm_source=chatgpt.com "MyBatisBatchItemWriter (mybatis-spring 4.0.0 API)"))

|우선순위|권장 조치|
|---|---|
|1|원천 SQL에서 `(GOODS_NO, CTGRY_ID)` 중복 발생 여부 확인|
|2|업무적으로 허용되면 mapper를 `ON DUPLICATE KEY UPDATE`로 변경|
|3|그 전까지는 XML skip 설정으로 배치 중단 방지|
|4|SkipListener로 skip 건 별도 로그/테이블 적재|
|5|`no-rollback`은 보류|

## 8. 바로 적용 가능한 결론

### 임시 대응

```xml
<batch:chunk reader="goodsCtgryRelReader"
             processor="goodsCtgryRelProcessor"
             writer="goodsCtgryRelWriter"
             commit-interval="100"
             skip-limit="1000">
    <batch:skippable-exception-classes>
        <batch:include class="org.springframework.dao.DuplicateKeyException"/>
    </batch:skippable-exception-classes>
</batch:chunk>
```

### 근본 대응

- **재실행 중복 허용**이면: `ON DUPLICATE KEY UPDATE`
    
- **신규만 허용**이면: `NOT EXISTS`
    
- **소스 중복**이면: reader SQL/staging 정제
    

# 결론

이번 오류는 단순히 “중복 한 건 무시” 문제가 아니라, **writer batch flush 구조에서 PK 중복으로 chunk 전체가 실패하는 전형적인 케이스**입니다.  
운영을 빨리 안정화하려면 **Spring Batch skip 설정**을 먼저 넣는 것이 맞지만, 가장 좋은 해결은 **SQL을 idempotent하게 변경하거나 source 중복 자체를 제거하는 것**입니다. Spring Batch 공식 문서 기준으로 skip은 데이터 의미를 이해한 뒤 결정해야 하며, writer 예외는 기본 rollback 대상이므로 무조건 `no-rollback`으로 푸는 방식은 보수적으로 봐야 합니다. ([Home](https://docs.spring.io/spring-batch/reference/step/chunk-oriented-processing/configuring-skip.html?utm_source=chatgpt.com "Configuring Skip Logic :: Spring Batch Reference"))  
다음 답변에서는 원하시는 기준으로 **현재 `MyBatis mapper XML`을 `ON DUPLICATE KEY UPDATE` 또는 `NOT EXISTS` 방식으로 실제 변환한 예제**까지 바로 작성해드릴 수 있습니다.

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
