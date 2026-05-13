---
layout: single
title: "[Spring Batch] Spring Batch"
excerpt: "Spring Batch 정리 #3"

categories:
  - tech
tags:
  - [tech, Spring Batch]

toc: false
toc_sticky: true

date: 2022-12-14
last_modified_at: 2022-12-14
---
# Spring Batch 정리 #3

![spring_batch_step](./../../images/tech/image_spring_batch.png)

- Jop : 빨래
  - Step : 설정
    - Tasklet : A사 세제를 넣는다
    - Tasklet : A사 섬유유연제를 넣는다
    - Tasklet : 세탁, 헹굼, 탈수 시간을 정한다
  - Step : 세탁
    - Tasklet : 일정시간동안 세탁을 진행한다
  - Step : 헹굼
    - Tasklet : 일정시간동안 헹굼을 진행한다
  - Step : 탈수
    - Takslet : 일정시간동안 탈수를 

### SimpleJob
- SimpleJob은 Step을 실행시키는 Job 구현체로서 SimpgitleJobBuilder에 의해 생성된다.
- 여러 단계의 Step을 구성할 수 있으면 Step을 순차적으로 실행시킨다.
- 모든 Step의 시랳ㅇ이 성공적으로 완료되어야 Job이 성공적으로 완료된다.
- 맨 마지막에 실행한 Step의 BatchStatus가 Job의 최종 BatchStatus가 된다.

```java
@Bean
public Job job1() {
    return jobBuilderFactory.get("job1")
      .start(step1())
      .next(step2())
      .next(step3())
      .incrementer(new RunIdIncrementer())
      .validator(new JobParametersValidator() {
        @Override
        public void validate(JobParameters parameters) throws JobParametersInvalidException {

        }
      })
      .preventRestart()
      .listener(new JobExecutionListener() {
        @Override
        public void beforeJob(JobExecution jobExecution) {

        }

        @Override
        public void afterJob(JobExecution jobExecution) {

        }
      })
      .build();
}
```

#### API
- get(String): Job을 생성한다.
- start(Step): Job에서 처음 실행할 Step을 설정한다.
  - 최초에 한번 설정하면 SimpleJobBuilder이 반환된다.
- next(Step): 다음에 실행 할 Step을 설정한다.
  - 횟수제한 X, 모든 next()의 Step이 종료되면 Job이 종료된다.
- incrementer(JobParametersIncrementer): JobParameter의 값을 자동으로 증가해주는 JobParametersIncrementer을 설정한다.
  - JobParameters에서 필요한 값을 증가시켜 다음에 사용될 JobParameters 오브젝트를 리턴한다.
  - 기존의 JobParameter 변경없이 Job을 여러 번 시작하고자 할 때 사용한다.
- RunIdIncrementer 구현체를 지원하고, 필요에 따라 인터페이스를 직접 구현할 수 있다.
- preventRestart(Boolean): Job의 재시작 가능 여부를 설정한다. 기본값 true
  - false는 재시작을 지원하지 않는다는 의미이다.
  - Job이 실패해도 재시작이 안되며 Job을 재시작하려고 하면 JobRestartException이 발생한다.
  - 재시작과 관련있는 기능으로 Job을 처음 실행하는 것과는 아무 상관이 없다.
- validator(JobParameterValidator): JobParameter을 실행하기 전에 올바른 구성이 되었는지 검증하는 JobParametersValidator을 설정한다.
  - Job 실행에 필요한 파라미터를 검증하는 용도로 사용한다.
  - DefaultJobParametersValidator 구현체를 지원하고, 필요에 따라 직접 인터페이스를 구현할 수 있다
- listener(JobExecutionListner): Job 라이프 사이클의 특정 시점에 콜백 제공도록 설정한다.
- build(): SimpleJob을 생성한다.

## Step

- 스프링 배치에서 Step이란 Job을 구성하는 하나의 단계로, 실제 배치가 실행되는 처리를 정의하고 컨트롤하는데 필요한 모든 정보를 가지고 있는 도메인 객체이다.
- 배치 작업을 어떻게 구성하고 실행할 것인지 Job의 세부 작업을 Task 기반으로 설정하고 명세해 놓은 객체이다.
- 단순한 단인 테스크부터 입력, 처리, 출력과 관련된 비즈니스 로직을 포함하는 모든 설정을 담고 있다.
- 모든 Job은 하나 이상의 Step으로 구성된다.

### Step의 기본 구현체

- TaskletStep: 가장 기본이 되는 클래스, Tasklet 타입의 구현체들을 제어한다.
- PartitionStep: 멀티 스레드 방식으로 Step을 여러 개로 분리해서 실행한다.
- JobStep: Step 내에서 Job을 실행하도록 한다.
- FlowStep: Step 내에서 Flow를 실행하도록 한다.

### StepExecution

- Step에 대한 한번의 시도를 의미한다.
- Step 실행 중에 발생한 정보들을 저장하고 있는 객체이다.
  - 시작시간, 종료시간, 상태, commit count, rollback count 등
- Step이 매번 시도될 때마다 생성되고, 각 Step별로 생성된다.
- Job이 재시작 하더라도 이미 성공적으로 완료된 Step은 재 실행되지 않고 실패한 Step만 실행된다.
- 이전 단계의 Step이 실패하였다면 다음 단계의 StepExecution은 생성되지 않는다. 즉, 실제로 실행된 Step만 StepExecution이 생성된다.

### StepContribution

- 청크 프로세스의 변경 사항을 버퍼링 한 후 StepExecution상태를 업데이트하는 도메인 객체
- 청크 커밋 직전에 StepExecution의 apply 메서드를 호출해 상태를 업데이트 한다.
- ExitStatus의 기본 종료코드 외 사용자 정의 종료코드를 생성해서 적용할 수 있다.

### ExecutionContext

- 프레임워크에서 유지, 관리하는 키/값으로 된 컬렉션으로 StepExecution 또는 JobExecution 객체의 상태를 젖아하는 공유 객체
- JOB_EXECUTION, STEP_EXECUTION
- DB에 직렬화 한 값으로 저장된다.
  - {"key", "value"}
- 공유범위
  - Step 범위: 각 Step의 StepExecution에 저장되며 Step간 서로 공유되지 않는다.
  - Job 범위: 각 Jbo의 JobExecution에 저장되며 Job간 서로 공유되지 않으며 해당 Job의 Step간 서로 공유된다.
- Job 재 시작시 이미 처리된 Row 데이터는 건너뛰고 이후로 수행하도록 할 때 상태 정보를 활용한다.

### JobRepository

- 배치 작업 중에 생성된 정보를 저장하는 저장소 역할을 한다.
- Job이 언제 수행되었고, 언제 끝나고, 몇 번 실행되었는지, 실행에 대한 결과 등의 배치 작업의 수행과 관련된 모든 데이터를 저장한다.
- JobLauncher, Job, Step 구현체 내부에서 CRUD 기능을 처리한다.

#### JobRepository 설정

- @EnableBatchProcessing 어노테이션 선언 시 JobRepository가 자동으로 빈으로 생성된다.
- BatchConfigurer 인터페이스를 구현하거나 BasicBatchConfigurer를 상속해서 JobRepository 설정을 커스터마이징 할 수 있다.
- JDBC 방식: JobRepositoryFactoryBean
  - 내부적으로 AOP 기술을 통해 트랜잭션 처리르 해준다.
  - 트랜잭션 isolation의 기본값은 SERIALIZEBLE로 최고 수준이며, 다른 레벨로 지정할 수 있다.
  - 메타테이블의 Table Prefix를 변경할 수 있다. (기본값 BATCH_)

- In Memory 방식: MapJobRepositoryFactoryBean
  - 성능 등의 이유로 도메인 오브젝트를 굳이 데이터베이스에 저장하고 싶지 않을 때 사용
  - 보통 Test나 프로토타입의 빠른 개발이 필요할 때 사용한다.

### JobLauncher

- 배치 Job을 실행시키는 역할을 한다.
- Job과 Job Parameters를 인자로 받으며 요청된 배치 작업을 수행한 후 최종 client에게 JobExecution을 반환한다.
- 스프링 부트 배치가 구동되면 JobLauncher 빈이 자동 생성된다.
- 스프링 부트 배치에서는 JobLauncherApplicationRunner가 자동적으로 JobLauncher을 실행시킨다.

### JobLauncher 동기적 실행

- taskExecutor를 SyncTaskExecutor로 설정할 경우 (기본값: SyncTaskExecutor)
- JobExecution을 획득하고 배치 처리를 최종 완료한 이후 Client에게 JobExecution을 반환한다.
- 스케줄러에 의한 배치처리에 적합하다. (배치처리시간이 상관 없는 경우)

### JobLauncher 비동기적 실행

- taskExecutor가 SimpleAsyncTaskExecutor로 설정할 경우
- JobExecution을 획득한 후 Client에게 바로 JobExecution을 반환하고 배치처리를 완료한다.
- HTTP 요청에 의한 배치처리에 적합하다.
  - 요청시 배치처리가 완료되기 전에 응답할 수 있다.

## StepBuilderFactory

```java
@Bean
public Step step1() {
  return stepBuilderFactory.get("step1")
    .tasklet((contribution, chunkContext) -> {
        System.out.println("step1 has executed");
        return RepeatStatus.FINISHED;
    })
    .build();
}
```

- StepBuilder를 생성하는 팩토리 클래스, get(String name) 메서드를 제공한다.

### StepBuidler

- Step을 구성하는 설정 조건에 따라 다섯개의 하위 빌더 클래스를 생성하고, Step 생성을 위임한다.
- TaskletStepBuilder
  - TaskletStep을 생성하는 기본 빌더 클래스
- SimpleStepBuilder
  - TaskletStep을 생성하며 내부적으로 청크기반의 작업을 처리하는 ChunkOrientedTasklet 클래스를 생성한다.
- PartitionStepBuilder
  - PartitionStep을 생성하며 멀티 스레드 방식으로 Job을 실행한다.
- JobStepBuilder
  - JobStep을 생성하여 Step 안에서 Job을 실행한다.
- FlowStepBuilder
  - FlowStep을 생성하여 Step 안에서 Flow를 실행한다.

### TaskletStep

```java
@Bean
public Step taskStep() {
  return stepBuilderFactory.get("taskStep")
    .tasklet(new Tasklet() {
        @Override
        public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
            System.out.println("step was executed");
            return RepeatStatus.FINISHED;
        }
    })
    .build();
}
```

- 스프링 배치에서 제공하는 Step의 구현체, Tasklet을 실행시키는 도메인 객체이다.
- RepeatTemplate를 사용해서 Tasklet의 구문을 트랜젝션 경계 내에서 반복 실행한다.
- Task기반과 Chunk기반으로 실행 단위가 나뉜다.

### Task 기반 Step

- ItemReader과 ItemWriter와 같은 청크 기반의 작업보다 단일 작업 기반으로 처리되는 것이 더 효율적인 경우 사용한다.
- 주로 Tasklet 구현체를 만들어서 사용한다.
- 대량 처리를 하는 경우 chunk기반에 비해 더 복잡한 구현이 필요하다.

### chunk 기반 Step

- 하나의 큰 덩어리를 n개씩 나눠서 실행한다는 의미
- 대량 처리를 하는 경우 효과적으로 설계된다.
- ItemReader, ItemProcessor, ItemWriter를 사용한다.
- 청크 기반 전용 Tasklet인 ChunkOrientedTasklet 구현체가 제공된다.

#### API
- stepBuilderFactory.get(): StepBuilder를 생성하는 팩토리, Step의 이름을 매개변수로 받는다.
- tasklet(Tasklet): Tasklet 클래스 설정, TaskletStepBuidler를 반환한다.
- startLimit(Integer): Step의 실행 횟수를 설정한다. 설정한 만큼 실행되고 초과시 오류를 발생한다.
  - 기본값: INTEGER.MAX_VALUE
  - Step의 실행 횟수를 조정할 수 있다.
  - Step마다 설정할 수 있다. 
  - 설정 값을 초과해서 다시 실행하려고 하면 StartLimitExceededException이 발생한다.
  - startLimit의 디폴트 값으 Integer.MAX_VALUE
- allowStartIfComplete(Boolean): Step의 성공, 실패와 상관없이 Step을 실행하기 위한 설정
  - 재시작 가능한 Job에서 Step의 이전 성공 여부와 상관없이 항상 step을 실행하기 위한 설정이다.
  - 실행마다 유효성을 검증하는 Step이나 사전 작업이 꼭 필요한 Step 등에 사용한다.
  - 기본적으로 COMPLETED상태를 가진 Step은 Job 재시작시 실행하지 않고 스킵한다.
  - allowStartIfComplete가 true로 설정된 Step은 항상 실행한다.
- listener(StepExecutionListner): Step 라이프사이클의 특정 시점에 콜백 제공받도록 StepExcutionListener을 설정한다.
- build(): TaskletStep 생성

### Tesklet

```java
public Step step1() {
  return stepBuilderFactory.get("step1")
    .tasklet((contribution, chunkContext) -> {
        return RepeatStatus.FINISHED;
    })
    .build();
}
```

- Step 내에서 구성되고 실행되는 도메인 객체로, 주로 단일 테스크를 수행하기 위해 사용한다.
- TaskletStep에 의해 반복적으로 수행되며 반환값에 따라 계속 수행하거나 종료한다.
- RepeatStatus: Tasklet의 반복 여부 상태 값
  - RepeatStatus.FINISHED: Tasklet 종료, RepeatStatus를 null로 반환하면 RepeatStatus.FINISHED으로 해석된다.
  - RepeatStatus.CONTINUABLE: Tasklet 반복
  - RepeatStatus.FINISHED가 리턴되거나 실패 예외가 던져지기 전까지 TaskletStep에 의해 while문 안에서 반복적으로 호출된다.
- 익명 클래스 또는 구현 클래스를 만들어서 사용한다.
- Tasklet()메서드를 실행하게 되면 TaskletStepBuilder가 반환되어 관련된 API를 설정할 수 있다.
- Step에 오직 하나의 Tasklet 설정이 가능하며, 두개 이상을 설정할 경우 마지막에 설정한 객체가 실행된다.

### JobStep

```java
@Bean
public Step jobStep(JobLauncher jobLauncher) {
  return stepBuilderFactory.get("jobStep")
    .job(childJob())
    .launcher(jobLauncher)
    .parametersExtractor(jobParametersExtractor())
    .build();
}
```

- Job에 속하는 Step중 외부의 Job을 포함하고 있는 Step
- 외부의 Job이 실패하면 해당 Step이 실패하므로 결국 최종 기본 Job도 실패하게 된다.
- 모든 메타데이터는 기본 Job과 외부 Job별로 각각 저장된다.
- 커다란 시스템을 모듈로 쪼개고 흐름관리를 위해 사용한다.

#### API
- job(Job): JobStep 내에서 실행 될 Job 설정, JobStepBuilder를 반환한다.
- launcher(JobLauncher): Job을 실행할 JobLauncher를 설정한다.
- parametersExtractor(JobParameterExtractor): Step의 ExecutionContext를 Job이 실행되는데 피룡한 JobParameters로 변환한다.
- build(): JobStep을 생성한다.

### FlowJob

```java
@Bean
public Job batchJob() {
  return jobBuilderFactory.get("batchJob")
    .start(step1())
    // step1이 성공한다면 step3 실행
    .on("COMPLETED").to(step3())
    // step1이 실패한다면 step2 실행
    .from(step1())
    .on("FAILED").to(step2())
    .end()
    .build();
}
```

- Step을 순차적으로만 구성하는 것이 아닌 상태에 따라 흐름을 전환하도록 구성할 수 있으며 FlowJobBuilder에 의해 생성된다.
  - Step이 실패하더라도 Job은 실패로 끝나지 않도록 해야 하는 경우
  - Step이 성공했을 때 다음에 실행해야 할 Step을 구분해서 실행해야 하는 경우
  - 특정 Step은 전혀 실행되지 않게 구성해야 하는 경우
- Flow와 Job의 흐름을 구성하는 데만 관여하고 실제 비즈니스 로직은 Step에서 이루어진다.
- 내부적으로 SimpleFlow 객체를 포함하고 있으며 Job 실행 시 호출한다.
- 단순한 Step으로 생성하는 SimpleJob보다 다양한 Flow로 구성하는 FlowJob의 생성 구조가 더 복잡하고 많은 API를 제공한다.

#### API
- start(Step): Flow를 시작하는 Step을 설정한다.
- on(String pattern): Step의 실행 결과로 돌려받는 ExitStatus를 캐치하여 매칭하는 패턴, TansitionBuilder을 반환한다.
  - on 메서드를 호출하게 되면 TransitionBuilder가 작동하며 Step간에 조건부 전환을 구성할 수 있게 된다.
  - to(Step): 다음으로 이동할 Step을 지정한다.
  - stop(), fail(), end(), stopAndRestart(): Flow를 중지, 실패, 종료하도록 Flow를 종료, FlowBuilder 반환
- from(Step): 이전 단계에서 정의한 Step의 Flow를 추가적으로 정의한다.
- next(Step): 다음으로 이동할 Step을 지정한다.
- end(): build()앞에 위치하면 FlowBuilder를 종료하고 SimpleFlow 객체를 생성한다.
- build(): FlowJob을 생성하고 Flow 필드에 SimpleFlow를 저장한다.

### Transition

```java
@Bean
public Job batchJob() {
  return jobBuilderFactory.get("batchJob")
    .start(step1())
        .on("FAILED")
        .to(step2())
        .on("FAILED")
        .stop()
    .from(step1())
        .on("*")
        .to(step3())
        .next(step4())
    .from(step2())
        .on("*")
        .to(step5())
        .end()
    .build();
}
```

- Flow 내의 Step의 조건부 전환을 정의한다.
- Job의 API 설정에서 on(String pattern) 메서드를 호출하면 TransitionBuilder가 반환되어 Transition Flow를 구성할 수 있다.
- Step의 ExitStatus가 어떤 pattern과도 매칭되지 않으면 스프링 배치는 예외를 발생시키고 Job은 실패한다.
- transition은 구체적인 것부터 그렇지 않은 순서로 적용된다.

#### API
- on(String pattern)
  - Step의 실행 결과로 돌려받는 ExitStatus와 매칭하는 패턴 스키마, BatchStatus와 매칭하는 것이 아니다.
  - pattern이 ExitStatus와 매칭되면 다음으로 실행할 Step을 지정할 수 있다.
  - *: 0개 이상의 문자와 매칭, 모든 ExitStatus와 매칭
  - ?: 정확히 1개의 문자와 매칭
- to(): 다음으로 실행할 단계 지정
- from(): 이전 단계에서 정의한 Transition을 새롭게 추가 정의
- stop(): FlowExecutionStatus가 STOPPED 상태로 종료되는 transition
  - Job의 batchStatus와 ExitStatus가 STOPPED로 종료된다.
- fail(): flowExecutionStatus가 FAILED 상태로 종료되는 transition
  - Job의 batchStatus와 ExitStatus가 FAILED로 종료된다.
- end(): flowExecutionStatus가 COMPLETED 상태로 종료되는 transition
  - Job의 BatchStatus와 ExitStatus가 COMPLETED으로 종료된다.
  - Step의 ExitStatus가 FAILED이더라도 Job의 BatchStatus가 COMPLETED로 종료하도록 가능하며 이 때 Job의 재시작은 불가능하다.
- stopAndRestart(Step or Flow or jobExecutionDecider)
  - stop() transition과 흐름은 동일하다.
  - 특정 Step에서 작업을 중단하도록 설정하면 중단 이전의 Step만 COMPLETED로 저장되고 이후의 Step은 실행되지 않고 STOPPED상태로 Job을 종료한다.
  - Job이 다시 실행됐을 때 실행해야 할 step을 restart인자로 넘기면 이전에 COMPLETED로 저장된 step은 건너뛰고 중단 이후 step부터 시작한다.

### SimpleFlow

```java
@Bean
public Job batchJob() {
  return jobBuilderFactory.get("batchJob")
    .start(flow())
    .next(step3())
    .end()
    .build();
}
```

- 스프링 배치에서 제공하는 Flow의 구현체로서 각 요소(Step, Flow, JobExecutionDecider)들을 담고 있는 State를 실행시키는 도메인 객체
- FlowBuilder를 사용해서 생성하며 Transition과 조합하여 여러 개의 Flow 및 중첩 Flow를 만들어 Job을 구성할 수 있다.

#### API
- start(Flow): Flow를 정의해서 설정한다.
- on("COMPLETED").to(Flow()): Flow를 Transition과 함께 구성한다.
- end(): SimpleFlow 객체를 생성한다.
- build(): FlowJob 객체를 생성한다.

### FlowStep

```java
private Step flowStep() {
    return stepBuilderFactory.get("flowStep")
            .flow(flow())
            .build();
}

private Flow flow() {
    FlowBuilder<Flow> flowBuilder = new FlowBuilder<>("flow");
    flowBuilder.start(step1())
            .end();
    return flowBuilder.build();
}
```

- Step 내에서 Flow를 할당하여 실행시키는 도메인 객체
- flowStep의 BatchStatus와 ExitStatus는 Flow의 최종 상태값에 따라 결정된다.


## @JobScope와 @StepScope

- Scope란? : 스프링 컨테이너에서 빈이 관리되는 범위
- singleton, prototype, request, session, application이 있으며 기본은 singleton으로 생성된다.

### 스프링 배치 스코프

- @JobScope, @StepScope

```java
@Bean
@JobScope
public Step step1(@Value("#{jobParameters['message']}") String message) {
  return stepBuilderFactory.get("step1")
    .tasklet(tasklet1(null))
    .build();
}

@Bean
@StepScope
public Tasklet tasklet1(
    @Value("#{jobExecutionContext['name']}") String name
) {
  return (contribution, chunkContext) -> {
    log.warn(">>tasklet1 has executed");
    return RepeatStatus.FINISHED;
  };
}

@Bean
@StepScope
public Tasklet tasklet2(
    @Value("#{stepExecutionContext['name2']}") String name2
) {
  return (contribution, chunkContext) -> {
    log.warn(">>tasklet2 has executed");
    return RepeatStatus.FINISHED;
  };
}
```

- Job과 Step의 빈 생성과 실행에 관여하는 스코프
- 프록시 모드를 기본값으로 하는 스코프: @Scope(value = "job", proxyMode = ScopedProxyMode.TARGET_CLASS)
- @JobScope나 @StepScope가 선언되면 빈의 생성이 어플리케이션 구동시점이 아닌 빈의 실행시점에 이루어진다.
  - @Values를 주잉ㅂ해서 빈의 실행 시점에 값을 참조할 수 있으며 일종의 Lazy Binding이 가능해진다.
  - @Value(”#{jobParameters[파라미터명]}”), @Value(”#{jobExecutionContext[파라미터명]”}), @Value(”#{stepExecutionContext[파라미터명]”})
  - @Value를 사용할 경우 빈 선언문에 @JobScope, @StepScope를 정의하지 않으면 오류를 발생한다. 때문에 반드시 선언해야 한다.
- 프록시 모드로 빈이 선어되기 때문에 어플리케이션 구동시점에는 빈의 프록시 객체가 생성되어 실행 시점에 실제 빈을 호출해준다.
- 병렬처리 시 각 스레드마다 생성된 스코프 빈이 할당되기 때문에 스레드에 안전하게 실행이 가능하다.

### @JobScope

```java
@Bean
@JobScope
public Step step1(@Value("#{jobParameters['message']}") String message) {
  return stepBuilderFactory.get("step1")
      .tasklet(tasklet1(null))
      .build();
}
```

- Step 선언문에 정의한다.
- @Value: jobParameter, jobExecutionContext만 사용가능하다.

### @StepScope

```java
@Bean
@StepScope
public Tasklet tasklet1(
    @Value("#{jobExecutionContext['name']}") String name
) {
  return (contribution, chunkContext) -> {
      log.warn(">>tasklet1 has executed");
      return RepeatStatus.FINISHED;
  };
}

@Bean
@StepScope
public Tasklet tasklet2(
    @Value("#{stepExecutionContext['name2']}") String name2
) {
  return (contribution, chunkContext) -> {
      log.warn(">>tasklet2 has executed");
      return RepeatStatus.FINISHED;
  };
}
```

- Tasklet이나 ItemReader, ItemWriter, ItemProcessor선언문에 정의한다.
- @Value: jobParameter, jobExecutionContext, stepExecutionContext만 사용가능하다.

## 멀티쓰레드 Step
- 대용량 배치 처리에 사용
- 수정 전 기존 배치
  ```java
  private Step useDeckStatsStep() {
    return stepBuilderFactory.get("useDeckStatsStep")
      .<MatchInfo, MatchInfo>chunk(chunkSize)
      .reader(useDeckStatsReader())
      .processor(useDeckStatsProcessor)
      .writer(useDeckStatsWriter())
      .build();
  }
  ```

- 위와 같은 Step을 10개의 쓰레드를 이용해 멀티쓰레드 환경으로 실행.

- 수정 후
  ```java
  private final int poolSize = 10;

  ...

  private Step useDeckStatsStep() {
    return stepBuilderFactory.get("useDeckStatsStep")
      .<MatchInfo, MatchInfo>chunk(chunkSize)
      .reader(useDeckStatsReader())
      .processor(useDeckStatsProcessor)
      .writer(useDeckStatsWriter())
      .taskExecutor(executor())
      .throttleLimit(poolSize)
      .build();
  }

  ...

  private TaskExecutor executor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(poolSize);
    executor.setMaxPoolSize(poolSize);
    executor.setThreadNamePrefix("multi-thread-");
    executor.setWaitForTasksToCompleteOnShutdown(Boolean.TRUE);
    executor.initialize();
    return executor;
  }

  ...

  ```

- ThreadPoolTaskExecutor: 쓰레드 풀을 이용해 쓰레드를 관리하는 방식
- corePoolSize: Pool의 기본 사이즈
- maxPoolSize: Pool의 최대 사이즈
- threadNamePrefix: 쓰레드 앞에 붙을 이름
- throttleLimit: 생성된 쓰레드 중 작업에 사용할 쓰레드 수를 설정한다.
- 위와 같이 taskExecutor과 throttleLimit을 사용해 멀티쓰레드 환경을 설정한다.

- 주의할 점은 ItemReader의 Thread Safe 유무이다.

```java
private ItemReader<MatchInfo> useDeckStatsReader() {
  return new JpaPagingItemReaderBuilder<MatchInfo>()
    .name("useDeckStatsReader")
    .entityManagerFactory(emf)
    .pageSize(chunkSize)
    .queryString("select m from MatchInfo m where m.isDeckCollected = FALSE")
    .saveState(false)
    .build();
}
```
- JpaPagingItemReader은 saveState를 false로 설정했을 경우 thread-safe를 지원.
- saveState를 false로 설정.
- 서로 다른쓰레드를 사용하여 동일 테이블을 입력, 조회 시에는 문제발생 가능.

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조


  </pre>
</details>