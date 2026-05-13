---
layout: single
title: "[Spring Batch] Spring Batch"
excerpt: "Spring Batch 정리 #2"

categories:
  - tech
tags:
  - [tech, Spring Batch]

toc: false
toc_sticky: true

date: 2022-12-14
last_modified_at: 2022-12-14
---
# Spring Batch 정리 #2
## 쿼츠(Quartz) 사용법
- 스프링 부트에서의 스케줄과 관련된 대표적인 라이브러리인 쿼츠(Quartz)설정은 매우 간단.
- maven 또는 gradle에서 라이브러리를 추가하는 것 만으로도 쿼츠(Quartz)와 관련된 객체가 자동으로 어플리케이션 영역에 생성.
- 스케줄을 실행하는 대표적인 Job클래스의 형태.

```java
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;

public class JobScheduler extends QuartzJobBean {
  @Override
  protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
  
  }
}
```

- 추상 클래스인 QuartzJobBean을 상속 받으면 executeInternal이라는 메소드를 오버라이드.
- 메소드에 JobExecutionContext 라는 인터페이스를 사용가능.
- context라는 객체를 활용하면 스프링부트에 의해 이미 어플리케이션 영역에 등록된 쿼츠(Quartz)로부터 Bean 객체를 사용가능.

```java
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;

public class 동작해야되는스케줄클래스 extends QuartzJobBean {
	
  private 내가만든서비스 service;

  @Override
  protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
    if(service == null) { 
      ApplicationContext appCtx = (ApplicationContext)context.getJobDetail()
          .getJobDataMap().get("사용자가지정한이름");
      service = appCtx.getBean(내가만든서비스.class); 
    }        
  }
}
```

- context 객체로부터의 일(job)에 대한 정보를 "사용자가 지정한이름"으로부터 가져와서의 스프링부트의 어플리케이션(Application) 영역을 가져오게 되어 있습니다.
- getBean 메소드를 통해 본인이 등록한 서비스 또는 빈(Bean) 객체를 사용.
여기서 주의해야되는 점은 아래 어플리케이션(Application) 영역을 가져오는 코드입니다.

context.getJobDetail().getJobDataMap().get("사용자가지정한이름")
 
어플리케이션(Application)영역을 가져오는 코드는 반드시 별도의 설정을 통해서만 가능합니다.
만약 위 코드를 그냥 실행한다면 getJobDataMap() 메소드에서의 반환값이 아무것도 없는 상태(null 또는 size 0)로 나오게 될 것 입니다.
이러한 어플리케이션(Application)영역을 가져오기 위한 방법은 구글링하면 각종 샘플 코드가 존재하지만, 프로젝트의 구성과 환경이 서로 상이하기 때문에 잘 안되는 경우가 많습니다.
그래서 이를 위해서 아주 간단하게 어플리케이션(Application)영역을 등록하는 방법을 살펴 보겠습니다.

```java
import java.util.function.Function;

import org.quartz.CronScheduleBuilder;
import org.quartz.JobBuilder;
import org.quartz.JobDataMap;
import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.Trigger;
import org.quartz.TriggerBuilder;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Configuration;

import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class 웹설정클래스 implements WebMvcConfigurer{
  //쿼츠 스케줄을 위한 객체 입니다.
  private Scheduler scheduler;
  private ApplicationContext applicationContext;
  
  //어플리케이션 영역을 가져오기 위해 사용할 이름입니다.
  public static String APPLICATION_NAME = "appContext"; 

  //생성자를 통하여 아래 2개의 객체를 받습니다.
  public 웹설정클래스(Scheduler sch, ApplicationContext applicationContext) {
      this.scheduler = sch;
      this.applicationContext = applicationContext;
  }
}
```

- "웹설정클래스" 라는 이름의 클래스 입니다.
- WebMvcConfigurer 인터페이스를 상속 받은 뒤에 Configuration 에노테이션을 붙여 주었습니다.
- 이렇게 되면 해당 클래스는 스프링부트에 의해서 자동으로 빈(Bean) 주입을 받을 수 있게되며, 스프링부트에서의 설정 역할을 할 수 있게 됩니다.
- 이를 확인 한 이후에 동작을 시켜서 scheduler와 applicationContext 객체가 정상적으로 주입받았는지 확인 합니다.

![spring_quartz](./../../images/tech/spring_batch_quartz.png)

여기까지 왔다면 이제 남은 작업은 어플리케이션 영역을 넣어주고, Job 클래스를 상속받은(QuartzJobBean) 스케줄을 동작해야 될 객체를 등록하여 줍니다.

아래 샘플 소스코드가 이번 내용의 핵심 입니다!!!

```java
import java.util.function.Function;
import javax.annotation.PostConstruct;
import org.quartz.CronScheduleBuilder;
import org.quartz.JobBuilder;
import org.quartz.JobDataMap;
import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.Trigger;
import org.quartz.TriggerBuilder;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import 동작해야되는스케줄클래스;  //사용자가 만든 스케줄클래스 입니다.

@Configuration
public class 웹설정클래스 implements WebMvcConfigurer{

  //쿼츠 스케줄을 위한 객체 입니다.
  private Scheduler scheduler;
  private ApplicationContext applicationContext;
  
  //어플리케이션 영역을 가져오기 위해 사용할 이름입니다.
  public static String APPLICATION_NAME = "appContext"; 

  //생성자를 통하여 아래 2개의 객체를 받습니다.
  public 웹설정클래스(Scheduler sch, ApplicationContext applicationContext) {
      this.scheduler = sch;
      this.applicationContext = applicationContext;
  }

  @PostConstruct
  public void schInint() throws SchedulerException {
    //크론스케줄을 쓰겠다는 함수
    final Function<String, Trigger> trigger = (exp)-> TriggerBuilder.newTrigger() 
        .withSchedule(CronScheduleBuilder.cronSchedule(exp)).build();

    JobDataMap ctx = new JobDataMap();  //스케줄러에게 어플리케이션(Application)영역을 넣어 줍니다.  
    ctx.put(APPLICATION_NAME, applicationContext);  //넣어줄 때 이름은 "appContext" 입니다.
    JobDetail jobDetail = JobBuilder.newJob(동작해야되는스케줄클래스.class).setJobData(ctx).build();  //스케줄을 생성해서
    scheduler.scheduleJob(jobDetail, trigger.apply("0/59 * * * * ?"));  //크론형식을 더해 시작합니다
  }

}
```

- PostConstruct 에노테이션을 활용하여 Scheduler 와 ApplicationContext 객체가 "웹설정클래스"가 의존성을 다 받고 나서 동작하게 하였습니다.

- 이렇게 간단한 웹 설정을 하면 처음 소개한 "동작해야되는스케줄클래스" 가 정상적으로 동작하며 기존에 등록한 Bean 객체에 접근을 할 수 있게 됩니다.

```java
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;

public class 동작해야되는스케줄클래스 extends QuartzJobBean {
	
  private 내가만든서비스 service;

  @Override
  protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
    if(service == null) { 
      ApplicationContext appCtx = (ApplicationContext)context.getJobDetail()
          .getJobDataMap().get("사용자가지정한이름");
      service = appCtx.getBean(내가만든서비스.class); 
      System.out.println(service);  //null이 안나오면 성공입니다!! 이제 원하는작업 ㄱㄱ!
    }        
  }
}
```

- 다른 내용을 검색하면 SchedulerFactoryBean 를 받아서 Bean으로 등록해야한다..AutoWired를 해야한다 등등..
스케줄러의 객체를 관리하고 선언하며, 직접적인 접근을 통해 세부적인 설정을 하는 예시가 많습니다.
이러한 방법은 프로젝트의 구성, 라이브러리의 버전 및 기타 환경등에 의해서 잘 안되는 경우가 많습니다.


## @Scheduled 이용한 Batch 작업
예전에 Quartz를 이용해서 작업할 때 보다는 @Scheduled annotation을 사용하면 배치 작업을 무척 쉽게 만들 수 있습니다.

```java
package com.copycoding.batch;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class BatchApplication {
    public static void main(String[] args) {
        SpringApplication.run(BatchApplication.class, args);
    }
}
```

- @EnableScheduling 어노테이션을 추가.

### 실제 스케쥴 작업할 class 파일.

```java
package com.copycoding.batch;
import java.time.LocalDateTime;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class ScheduleTask {
  @Scheduled(fixedDelay = 2000)
  public void task1() {
      System.out.println("The current date (1) : " + LocalDateTime.now());
  }

  @Scheduled(fixedDelayString = "${spring.task.fixedDelay}")
  public void task2() {
      System.out.println("The current date (2) : " + LocalDateTime.now());
  }
}
```

- 중요한건 class 파일에 @Component를 설정해야 합니다.  
- 이렇게 두개의 어노테이션을 적어주기만 하면 설정 끝.
- 이제 메소드를 만들고 @Scheduled를 이용해서 메소드의 작동 시간을 입력하고 코딩 

### @Scheduled() 어노테이션 설정 정리.
- fixedDelay
  - @Scheduled(fixedDelay = 1000)
  - 이전 작업이 종료된 후 설정시간(밀리터리세컨드) 이후에 다시 시작

- fixedDelayString
  - @Scheduled(fixedDelay = “1000”)
  - fixedDelay와 동일 하고 지연시간(the delay in milliseconds)을 문자로 입력

- fixedRate
  - @Scheduled(fixedRate = 1000)
  - 설정된 시간마다 시작을 한다. 즉 이전 작업이 종료되지 않아도 시작.

- fixedRateString
  - @Scheduled(fixedRateString = “1000”)
  - fixedRate와 동일 하고 지연시간(the delay in milliseconds)을 문자로 입력

- initialDelay
  - @Scheduled(fixedRate = 5000, initialDelay = 3000)
  - 프로그램이 시작하자마자 작업하는게 아닌 시작을 설정된 시간만큼 지연하여 작동을 시작 한다.(예제는 3초 후 부터 5초 간격으로 작업)

- initialDelayString
  - @Scheduled(fixedRate = 5000, initialDelay = “3000”)
  - initialDelay와 동일 하고 지연시간(the delay in milliseconds)을 문자로 입력

- cron
  - @Scheduled(cron = "* * * * * *")
  - 첫번째 부터 위치별 설정 값은 초(0-59), 분(0-59), 시간(0-23), 일(1-31), 월(1-12), 요일(0-7)
    - * : all
    - ? : none
    - m : array
    - a-b : a부터 b까지
    - a/b : a부터 b마다. a, a+b, a+b+b, ...

- zone
  - @Scheduled(cron = "0 0 14 * * *" , zone = "Asia/Seoul")
  - 미설정시 local 시간대를 사용한다. oracle에서 제공하는 문서를 참조하여 입력 한다.
  https://docs.oracle.com/cd/B13866_04/webconf.904/b10877/timezone.htm


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- CustomQuartzJob

```java
@EqualsAndHashCode(callSuper=false)
@Data
@Slf4j
@NonNullApi
public class CustomQuartzJob extends QuartzJobBean {
  @Override
  protected void executeInternal(JobExecutionContext context) {
    try{
      PThemeExhbtMgtVO pthemeExhbtMgtVO = (PThemeExhbtMgtVO) context.getMergedJobDataMap().get("themeExhbtMgtVO");
      DsThemeExhbtMstVo dsthemeExhbtMstVO = new DsThemeExhbtMstVo();
      BeanUtils.copyProperties(pthemeExhbtMgtVO, dsthemeExhbtMstVO);
      ApplicationContext ctx = (ApplicationContext) context.getMergedJobDataMap().get("applicationContext");
      ThemeExbtService themeExbtService = ctx.getBean(ThemeExbtService.class);

      log.info("themeExhbtMgtVo : {}", pthemeExhbtMgtVO);
      log.info("themeExbtService:{} ", themeExbtService);

      themeExbtService.themeExbtServiceProc(dsthemeExhbtMstVO, pthemeExhbtMgtVO.getUserId());
    } catch (Exception e){
        e.printStackTrace();
    }
  }
}
```

- JobController

```java
@Api(tags={"4. 테마기획전"})
@RestController
@Slf4j
public class JobController {
  @Autowired
  DynamicScheduleConfig dynamicScheduleConfig;

//    @Autowired
//    ThemeExhbtMgtMapper themeExhbtMgtMapper;

  @Autowired
  @Qualifier("themeExbt")
  private Job job;

  @ApiOperation(value = "테마기획전 자동 업데이트 List", notes = "테마기획전 listThemeJob")
  @PostMapping("/listThemeJob")
  public ResponseEntity<Object> listThemeJob() throws SchedulerException {
    List<Map<String, Object>> out = dynamicScheduleConfig.listJobTrigger();
    log.info("out: {}", out);
    return ResponseEntity.ok(new ReturnVo("true", out));
  }

  @PostConstruct
  public void initThemeAuto() {
    log.info("===================================================================");

//        DsThemeExhbtMstVo dsThemeExhbtMstVo = new DsThemeExhbtMstVo();
//        List<DsThemeExhbtMstVo> dsThemeExhbtMstVoList = themeExhbtMgtMapper.selectThemeAutoUpdate(dsThemeExhbtMstVo);
//
//        String userId = "SYSTEM";
//        for(DsThemeExhbtMstVo vo : dsThemeExhbtMstVoList) {
//            log.info("{}, {}, {}, {}, {}", vo.getThemeExhbtSq(), vo.getThemeExhbtTitl(), vo.getCarRnewDcd(), vo.getRnewCyclCd(), userId);
//            PThemeExhbtMgtVO pThemeExhbtMgtVO = new PThemeExhbtMgtVO();
//            BeanUtils.copyProperties(vo, pThemeExhbtMgtVO);
//            pThemeExhbtMgtVO.setUserId(userId);
//
//            String jobName = String.format("job_%s", pThemeExhbtMgtVO.getThemeExhbtSq());
//
//            try {
//                String cronStr = getCrontabStr(pThemeExhbtMgtVO);
//                dynamicScheduleConfig.addJob(jobName, cronStr, pThemeExhbtMgtVO);
//            }catch(MyException ex){
//                log.info("에러가 발생했습니다. {}", ex.getMessage());
//            }
//        }
    log.info("===================================================================");
  }

  private String getCrontabStr(PThemeExhbtMgtVO pThemeExhbtMgtVO) throws MyException {
    String cronStr ;
    if ("01".equals(pThemeExhbtMgtVO.getRnewCyclCd())) {    // 30분
        cronStr = "0 0/30 * 1/1 * ? *";
        //"0/30 * * * * ?"
    } else if ("02".equals(pThemeExhbtMgtVO.getRnewCyclCd())) {    // 1시간
        cronStr = "0 0 0/1 1/1 * ? *";
    } else if ("03".equals(pThemeExhbtMgtVO.getRnewCyclCd())) {    // 3시간
        cronStr = "0 0 0/3 1/1 * ? *";
    } else if ("04".equals(pThemeExhbtMgtVO.getRnewCyclCd())) {    // 6시간
        cronStr = "0 0 0/6 1/1 * ? *";
    } else if ("05".equals(pThemeExhbtMgtVO.getRnewCyclCd())) {    // 12시간
        cronStr = "0 0 0/12 1/1 * ? *";
    } else if ("06".equals(pThemeExhbtMgtVO.getRnewCyclCd())) {    // 24시간
        cronStr = "0 0 12 1/1 * ? *";
    } else {
        throw new MyException("허용 하지 않는 갱신 주기입니다.");
    }
    return cronStr;
  }

  @ApiOperation(value = "테마 자동 업데이트 추가", notes = "테마 addThemeJob")
  @PostMapping("/addThemeJob")
  public ResponseEntity<Object> addThemeJob(@ApiParam("테마 자동") @Valid PThemeExhbtMgtVO pThemeExhbtMgtVO){
    log.info("addThemeJob : {}", pThemeExhbtMgtVO);

    String jobName = String.format("job_%s", pThemeExhbtMgtVO.getThemeExhbtSq());
    if (dynamicScheduleConfig.findJobByJobName(jobName) != null) {
      return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(String.format("%s이 이미 존재 합니다.", jobName));
    }

    try {
      String cronStr = getCrontabStr(pThemeExhbtMgtVO);
      dynamicScheduleConfig.addJob(jobName, cronStr, pThemeExhbtMgtVO);
    }catch(MyException myException ){
      return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("허용하지 않는 갱신 주기 코드 입니다.");
    }
    return ResponseEntity.ok(new ReturnVo("true", ""));
  }

  @ApiOperation(value = "테마 자동 업데이트 수정", notes = "테마 editThemeJob")
  @PostMapping("/editThemeJob")
  public ResponseEntity<Object> editThemeJob(@ApiParam("테마기획전 자동") @Valid PThemeExhbtMgtVO pThemeExhbtMgtVO){
    log.info("editThemeJob : {}", pThemeExhbtMgtVO);

    String jobName = String.format("job_%s", pThemeExhbtMgtVO.getThemeExhbtSq());
    if (dynamicScheduleConfig.findJobByJobName(jobName) == null) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(String.format("%s은 존재 하지 않는 job입니다.", jobName));
    }

    try {
      String cronStr = getCrontabStr(pThemeExhbtMgtVO);
      dynamicScheduleConfig.updateJob(jobName, cronStr, pThemeExhbtMgtVO);
    }catch(MyException myException ){
      return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("허용하지 않는 갱신 주기 코드 입니다.");
    }

    return ResponseEntity.ok(new ReturnVo("true", ""));
  }

  @ApiOperation(value = "자동 업데이트 삭제", notes = "테마 stopThemeJob")
  @PostMapping("/stopThemeJob")
  public ResponseEntity<Object> stopThemeJob(@ApiParam("테마 자동") @Valid DThemeExhbtMgtVO dThemeExhbtMgtVO){
    log.info("stopThemeJob : {}", dThemeExhbtMgtVO);

    String jobName = String.format("job_%s", dThemeExhbtMgtVO.getThemeExhbtSq());
    if (dynamicScheduleConfig.findJobByJobName(jobName) == null) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(String.format("%s은 존재 하지 않는 job입니다.", jobName));
    }
    dynamicScheduleConfig.deleteJob(jobName);

    return ResponseEntity.ok(new ReturnVo("true", ""));
  }
}
```


- BatchScheduler

```java
@Slf4j
@Component
@EnableScheduling
public class BatchScheduler {

  @Autowired
  private JobLauncher jobLauncher;

  @Autowired
  protected JobRegistry jobRegistry;

  @Autowired
  JobExplorer jobExplorer;

  @Autowired
  private Job userUpdateBatch;

  @Autowired
  @Qualifier("UpdateGoogleContentApiBatch")
  private Job UpdateGoogleContentApiJob;

  /**
    * 실행주기 0 0 2 * * * (1일 / 1회 / 02시)
    */
  @Scheduled(cron="0 0 2 * * *")
  public void DeleteSearchCarBatch() {
    try {
      JobParameters jobParameters = new JobParametersBuilder()
              .addDate("date", new Date())
              .toJobParameters();

      JobExecution jobExecution = jobLauncher.run(deleteSearchCarBatchJob, jobParameters);

      log.info("name : {} ", jobExecution.getJobInstance().getJobName());
      log.info("status : {}", jobExecution.getStatus());
    } catch (JobExecutionAlreadyRunningException | JobRestartException | JobInstanceAlreadyCompleteException | JobParametersInvalidException e) {
      e.printStackTrace();
    }
  }

  /**
    * 실행주기 1일 / 1회 / 10시
    */
  @Scheduled(cron="0 0 10 * * *")

  /**
    * 실행주기 1일 / 1회 / 00시
    */
  @Scheduled(cron="0 0 0 * * *")

/**
    * 실행주기 1일 / 1회 / 22시
    */
  @Scheduled(cron="0 0 22 * * *")

  /**
    * 실행주기 1일 / 1시간마다
    */
  @Scheduled(cron="0 0 */1 * * *")

  /**
    * 실행주기 1일 / 1회 / 21시48분
    */
  @Scheduled(cron="0 48 21 * * *")

  /**
    * 실행주기 1일 / 1회 / 21시48분
    */
  @Scheduled(cron="0 48 12 * * *")

  /**
    * 실행주기 매시 10분마다 배치 동작
    */
  @Scheduled(cron="0 */10 * * * *")

  /**
    * 실행주기 1일 / 8-20시 / 3분
    */
  @Scheduled(cron="0 */3 8-20 * * *")

  /**
    * 실행주기 1일 / 매시간마다
    */
  @Scheduled(cron="0 0 */1 * * *")

  /**
    * 실행주기 1일 / 8-20시 / 05분, 35분마다
    */
  @Scheduled(cron="0 05,35 8-20 * * *")

  /**
    * 실행주기 1일 / 1회 / 00시 10분
    */
  @Scheduled(cron="0 10 0 * * *")

  /**
    * 실행주기 매일 / 1시간마다
    */
  @Scheduled(cron="0 0 */1 * * *")

  /**
    * 실행주기 1일 / 8-23시 매시간마다
    */
  @Scheduled(cron="0 0 08-23/1 * * *")

  /*
    * @실행주기 1일 / 10분마다
    */
  @Scheduled(cron="0 */10 * * * *")

  /*
    * @실행주기 1일 / 1회 / 00시 10분
    */
  @Scheduled(cron="0 10 0 * * *")

  /*
    * @실행주기 1일 / 2시간마다
    */
  @Scheduled(cron="0 0 */2 * * *")

  /*
    * @실행주기    1일 / 9-18시 / 30분
    */
  @Scheduled(cron="0 */30 9,10,11,12,13,14,15,16,17,18 * * *")

  /*
    * @실행주기    1일 / 1회 / 2시
    */
  @Scheduled(cron="0 0 2 * * *")

  /*
    * @실행주기    1일 / 1회 / 5시
    */
  @Scheduled(cron="0 0 5 * * *")

  /*
    * @실행주기    1일 / 20분마다
    */
  @Scheduled(cron="0 */20 * * * *")

  /*
    * @실행주기    1일 / 1회 / 2시
    */
  @Scheduled(cron="0 0 2 * * *")

  /*
    * @실행주기    1일 / 1회 / 10시
    */
  @Scheduled(cron="0 0 10 * * *")

  /*
    * @실행주기 1일 / 09시 30분
    */
  @Scheduled(cron="0 30 9 * * ?")

  /*
    * @실행주기 1일 / 00시 00분
    */
  @Scheduled(cron="0 0 0 * * ?")
  public void updateMemeberLeaveBatchJob(){
    JobParameters jobParameters = new JobParametersBuilder().addLong("time", System.currentTimeMillis()).toJobParameters();
    try {
      JobExecution jobExecution = jobLauncher.run(updateMemberLeaveJob, jobParameters);
      log.info("JOB STATUS:::{}", jobExecution.getStatus());
      log.info("JOB IS:::{}", jobExecution);
    } catch (JobExecutionAlreadyRunningException | JobRestartException | JobInstanceAlreadyCompleteException | JobParametersInvalidException e) {
      e.printStackTrace();
    }
  }

  /*
    * @실행주기 30분마다 실행
    */
  @Scheduled(cron="0 */30 * * * *")

  /*
    * @실행주기 12월 28일 10시
    */
  @Scheduled(cron="0 0 10 28 12 *")

  /*
    * @실행주기    1일 / 1시간마다
    */
  @Scheduled(cron="0 0 */1 * * *")

  /*
    * @실행주기    1일 / 3시
    */
  @Scheduled(cron="0 0 3 * * *")
}
```


  </pre>
</details>