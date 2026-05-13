---
layout: single
title: "[Spring Batch] Spring Batch"
excerpt: "Spring Batch 정리 #1"

categories:
  - tech
tags:
  - [tech, Spring Batch]

toc: false
toc_sticky: true

date: 2022-12-14
last_modified_at: 2022-12-14
---
# Spring Batch 정리 #1
## JOB 상태를 STARTED에서 FAILED 로 변경
- Spring Batch 기동 시 배치 상태를 STARTED에서 FAILED로 변경하는 방법

Spring Batch 를 개발하다 보면, 코드 검증 등을 위해서 WAS 재기동하는 경우가 많다. 
그런데 Spring Batch의 특정 Job이 실행 중일 때 재기동을 하게 되면 해당 Job은 STARTED 상태로 남아있게 되며, 만약 Job의 중복 실행을 방지하는 코드가 들어가 있게 되면, STARTED 상태인 Job은 실행이 되지 않게 된다. 
따라서 매 재기동 시마다 STARTED인 상태의 Job을 FAILED나 COMPLETED 상태로 변경한 후 Job을 실행시켜야 하는 번거로움이 발생한다.
이런 번거로움을 해결하기 위해서는 기동 시 배치 상태를 STARTED에서 FAILED로 변경하는 것이 필요하다(WAS 기동 시 배치 상태가 STARTED인 것은, WAS 내릴 때 해당 배치의 상태를 고려하지 않고 내린 것이 되므로, 상태를 FAILED처리 할 필요가 있다)

해당 작업을 수행하기 위해서는 아래 코드를 작성하면 된다. 

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.springframework.batch.core.BatchStatus;
import org.springframework.batch.core.ExitStatus;
import org.springframework.batch.core.JobExecution;
import org.springframework.batch.core.StepExecution;
import org.springframework.batch.core.explore.JobExplorer;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.stereotype.Component;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.Set;


@Component
public class ContextRefreshEventListener implements ApplicationListener<ContextRefreshedEvent> {
    private static final Logger log = LogManager.getLogger();
    private final JobExplorer jobExplorer;
    private final JobRepository jobRepository;

    public ContextRefreshEventListener(JobExplorer jobExplorer, JobRepository jobRepository) {
        this.jobExplorer = jobExplorer;
        this.jobRepository = jobRepository;
    }

    // Spring 이 기동될 때마다 수행됨. 상태가 STARTED인 Job 전체 조회 후 상태를 FAILD 처리
    @Override
    public void onApplicationEvent(ContextRefreshedEvent contextRefreshedEvent) {
        log.info("Container Start. find STARTED STATUS Job and Change to FAILED");
        List<String> jobs = jobExplorer.getJobNames();
        for (String job : jobs) {
            Set<JobExecution> runningJobs = jobExplorer.findRunningJobExecutions(job);

            for (JobExecution runningJob : runningJobs) {
                try {
                    if (!runningJob.getStepExecutions().isEmpty()) {
                        Iterator<StepExecution> iter = runningJob.getStepExecutions().iterator();
                        while (iter.hasNext()) {
                            StepExecution runningStep = (StepExecution)iter.next();
                            if (runningStep.getStatus().isRunning()) {
                                runningStep.setEndTime(new Date());
                                runningStep.setStatus(BatchStatus.FAILED);
                                runningStep.setExitStatus(new ExitStatus("FAILED", "BATCH FAILED"));
                                jobRepository.update(runningStep);
                            }
                        }
                    }
                    runningJob.setEndTime(new Date());
                    runningJob.setStatus(BatchStatus.FAILED);
                    runningJob.setExitStatus(new ExitStatus("FAILED", "BATCH FAILED"));
                    jobRepository.update(runningJob);
                } catch (Exception e) {
                    log.error(e.getMessage(), e);
                }
            }
        }
    }
}
```

- 위와 같이 코드를 작성하게 되면, 스프링부트 WAS 기동 시, 상태가 STARTED인 JOB과 해당 JOB의 STEP 중 상태가 STARTED인 STEP을 찾아서 모두 FAILED 처리. 

### 한계점 
- Spring Batch 서버가 한 개인 경우는 유용하지만, 2개 이상의 서버로 관리하는 경우는 유용하지 않을 수 있다(예를 들면 배치 1번 서버를 재기동한 후 배치 2번 서버를 재기동 하기 전, 그 찰나의 순간에 스케쥴링 되어 있는 배치가 배치 1번 서버에서 돌게 되면, 배치 2번 서버 재기동 시 배치 1번 서버에서 실행된 배치 Job의 상태를 FAILED로 바꿀 수 있기 때문..)
그렇기 때문에 위와 같은 문제를 해결하기 위해서는 재기동 시에는 스케쥴을 중지해 놓고, 모든 서버 재기동 후 스케쥴을 원상복구 해 놓거나, 스케쥴을 동적으로 변경할 수 없는 경우라면 STARTED->FAILED로 변경하는 로직을 수동으로 호출할 수 있게 개발해야 할 것 같다)


## 배치 인프라스트럭처 구성하기
- @EnableBatchProcessing 적용 시 별도의 작업을 하지 않고 Job Repository를 사용할가능. 
- Job Repository를 Custom.

### BatchConfigurer 인터페이스
- 1. @EnableBatchProcessing 적용 후
- 2. BatchConfigurer 구현체에서 빈 생성
- 3. SimpleBatchConfiguration에서 ApplicationContext에 생성한 빈 등록

- 이 과정에서 보통 노출되는 컴포넌트를 커스텀하기 위해서 BatchConfigurer을 커스텀.

![spring_batch_custom1](./../../images/tech/spring_batch_custom01.png)

- PlatformTransactionManager : 프레임 워크가 제공하는 모든 트랜잭션 관리 시에 스프링 배치가 사용하는 컴포넌트
- JobExplorer : JobRepository의 데이터를 읽기 전용으로 볼 수 있는 기능
- BatchConfigurer을 상속받아 모든 메서드를 재정의하기보다는 DefaultBatchConfigurer을 상속해 필요한 메서드만 재정의하는 것이 더 쉽다.

### JobRepository 커스텀
- JobRepository는 JobRepositoryFactoryBean이라는 빈을 통해 생성된다.

![spring_batch_custom2](./../../images/tech/spring_batch_custom02.png)

![spring_batch_custom3](./../../images/tech/spring_batch_custom03.png)

위의 있는 필드들을 수정하여 커스텀할 수 있다.

DefaultBatchConfigurer를 상속해 createJobRepository() 를 재정의해야 하는 가장 일반적인 경우는 ApplicationContext에 두 개 이상의 데이터 소스가 존재한느 경우이다.

```java
    @Override
    protected JobRepository createJobRepository() throws Exception
    {
        JobRepositoryFactoryBean factoryBean = new JobRepositoryFactoryBean();
        factoryBean.setDatabaseType(DatabaseType.MYSQL.getProductName());
        
 	// 테이블 접두어 기본 값을 BSATCH_가 아닌 FOO_로 설정
    	factoryBean.setTablePrefix("FOO_");
        
   	// 데이터 생성 시 트랜잭션 격리 레벨 설정 factoryBean.setIsolcationLevelForCreate("ISOLATION_REPEATABLE_READ");
        factoryBean.setDataSource(dataSource);
        
        // 스프링 컨테이너가 빈 정의로 직접 호출하지 않음. 개발자가 직접 호출해야 한다.
        factoryBean.afterPropertiesSet();
        return factoryBean.getObject();
    }
```

### TransactionManager 커스텀
DefaultBatchConfigurer의 getTransactionManager를 호출하면 배치 처리에 사용할 목적으로 정의해둔 PlatformTransactionManager가 리턴된다. 이 때 정의해둔 TransactionManager가 없을 경우 DefaultConfigurer가 자동으로 생성한다.

![spring_batch_custom4](./../../images/tech/spring_batch_custom04.png)

getTransactionManager 메서드를 오버라이드하여 재정의하면 어떤 TransactionManager를 리턴할지 커스텀할 수 있다.

### JobExplorer 커스텀
JobExplorer는 JobRepository가 다루는 데이터를 읽기 전용으로 보는 뷰
-> 기본적 데이터 접근 계층은 JobRepository, JobExplorer 가 공유하는 공통 DAO 집합
-> 데이터를 읽을 때 사용하는 애트리뷰트 JobRepository와 동일

![spring_batch_custom5](./../../images/tech/spring_batch_custom05.png)

```java
    @Override
    protected JobExplorer createJobExplorer() throws Exception
    {
        JobExplorerFactoryBean factory = new JobExplorerFactoryBean();
        factory.setDataSource(this.dataSource);
        factory.setTablePrefix("FOO_");
        // BatchConfigurer 메소드는 스프링 컨테이너에 직접 노출되지 않으므로 직접 호출
        factory.afterPropertiesSet();
        return factory.getObject();
    }
```

### JobLauncher 커스텀
JobLauncher는 스프링 배치 잡을 실행하는 진입점으로 대부분 SimpleJobLauncher를 사용한다. 그렇기에 커스터마이징할 일이 거의 없지만 어떤 잡이 MVC 애플리케이션의 일부분으로 존재하며 컨트롤러를 통해 해당 잡을 실행할 때 별도의 방식으로 잡을 구동하는 방법을 외부에 공개하고 싶을 수 있다. 이 때 SimpleJobLauncher의 구동 방식을 조정한다.

![spring_batch_custom6](./../../images/tech/spring_batch_custom06.png)

SimpleJobLauncher 클래스에서 job repository, task executor(보통 SyncTaskExecutor)를 커스텀할 수 있는 메서드를 확인할 수 있다.

### 잡 메타 데이터 사용하기
보통 JobExplorer를 사용해서 JobRepository의 데이터를 가져온다.

![spring_batch_custom7](./../../images/tech/spring_batch_custom07.png)

jobExplorer 인터페이스에 선언된 메소드들이다. 위의 메소드들을 이용하여 메타 데이터를 읽어올 수 있다.

```java
public class ExploringTasklet implements Tasklet {

	private JobExplorer explorer;

	public ExploringTasklet(JobExplorer explorer) {
		this.explorer = explorer;
	}

	public RepeatStatus execute(StepContribution stepContribution,
			ChunkContext chunkContext) {

		// 현재 job의 이름을 가져온다.
		String jobName = chunkContext.getStepContext().getJobName();

		// jobName인 job의 첫번째 인덱스부터 MAX_VALUE만큼 가져온다.
		List<JobInstance> instances =
				explorer.getJobInstances(jobName,
						0,
						Integer.MAX_VALUE);

		System.out.println(
				String.format("There are %d job instances for the job %s",
				instances.size(),
				jobName));

		System.out.println("They have had the following results");
		System.out.println("************************************");

		for (JobInstance instance : instances) {
        		// instance의 job execution 리스트를 가져온다.
			List<JobExecution> jobExecutions =
					this.explorer.getJobExecutions(instance);

			System.out.println(
					String.format("Instance %d had %d executions",
							instance.getInstanceId(),
							jobExecutions.size()));

			for (JobExecution jobExecution : jobExecutions) {
            			// job execution 정보를 가져온다.
				System.out.println(
						String.format("\tExecution %d resulted in Exit Status %s",
								jobExecution.getId(),
								jobExecution.getExitStatus()));
			}
		}

		return RepeatStatus.FINISHED;
	}
}
```

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- BatchApplicationRefreshListener

```java
@Slf4j
@Component
public class BatchApplicationRefreshListener implements ApplicationListener<ContextRefreshedEvent> {

    @Autowired
    JobExplorer jobExplorer;

    @Autowired
    JobOperator jobOperator;

    @Autowired
    JobRepository jobRepository;

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        List<String> jobs = jobExplorer.getJobNames();
        for (String job : jobs) {
            Set<JobExecution> runningJobs = jobExplorer.findRunningJobExecutions(job);

            for (JobExecution runningJob : runningJobs) {
                try {
                    runningJob.setStatus(BatchStatus.FAILED);
                    runningJob.setEndTime(new Date());
                    jobRepository.update(runningJob);
                    jobOperator.restart(runningJob.getId());
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

- CustomDefaultBatchConfigurer

```java
@Configuration
public class CustomDefaultBatchConfigurer extends DefaultBatchConfigurer {

    @Autowired
    private DataSource dataSource;

    @Override protected JobRepository createJobRepository() throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTablePrefix("BATCH_");
        factory.setTransactionManager(new DataSourceTransactionManager(dataSource));
        factory.setIsolationLevelForCreate("ISOLATION_READ_COMMITTED");
        factory.afterPropertiesSet();
        return factory.getObject();
    }


//    @Override
//    public void setDataSource(DataSource dataSource) {
//        // 여기를 비워놓는다
//    }
}
```


- DuplicateJobCheckListener

```java
@Slf4j
@Component
public class DuplicateJobCheckListener extends JobExecutionListenerSupport{

    @Autowired
    JobExplorer jobExplorer;

    @Autowired
    JobOperator jobOperator;

    @Bean
    public JobRegistryBeanPostProcessor jobRegistryBeanPostProcessor(JobRegistry jobRegistry) {
        JobRegistryBeanPostProcessor jobRegistryBeanPostProcessor = new JobRegistryBeanPostProcessor();
        jobRegistryBeanPostProcessor.setJobRegistry(jobRegistry);
        return jobRegistryBeanPostProcessor;
    }


    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("******************************** beforeJob :"+jobExecution.getJobInstance().getJobName());
        Set<JobExecution> jobExecutionSet = jobExplorer.findRunningJobExecutions(jobExecution.getJobInstance().getJobName());
        if (jobExecutionSet.size() > 1) {
            log.info("**********************Exist********** beforeJob :"+jobExecution.getJobInstance().getJobName());
            log.info("**********************StopId********** beforeJob :"+jobExecution.getId());
            try {
                jobOperator.stop(jobExecution.getId());
            } catch (NoSuchJobExecutionException e) {
                e.printStackTrace();
            } catch (JobExecutionNotRunningException e) {
                e.printStackTrace();
            }
        }
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        log.info("******************************** afterJob :"+jobExecution.getJobInstance().getJobName());
        log.info("******************************** afterJobId :"+jobExecution.getId());
    }
}
```

- DynamicScheduleConfig

```java
@Slf4j
@Component
public class DynamicScheduleConfig {

    public  Scheduler scheduler;
    private SchedulerFactory schedulerFactory;
    private SchedulerFactoryBean schedulerFactoryBean;

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private JobLocator jobLocator;

    @Qualifier("themeExbt")
    private Job job;

    @Autowired
    ApplicationContext applicationContext;

    @PostConstruct
    public void dynamicSchedulerStart() throws SchedulerException{

        schedulerFactory = new StdSchedulerFactory();
        this.scheduler = schedulerFactory.getScheduler();
        this.scheduler.start();
    }


    public JobDetail makeJobDetail(String jobName,  PThemeExhbtMgtVO themeExhbtMgtVO){

        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("jobName", jobName);
        jobDataMap.put("jobLauncher", jobLauncher);
        jobDataMap.put("jobLocator", jobLocator);
        jobDataMap.put("triggerName", jobName);
        jobDataMap.put("themeExhbtMgtVO", themeExhbtMgtVO);
        jobDataMap.put("applicationContext", applicationContext);
        log.info("applicationContext : {}", applicationContext);

        return JobBuilder.newJob(DaisoQuartzJob.class)
                .setJobData(jobDataMap)
                .withIdentity(jobName)
                .build();
    }

        public JobDetail makeJobDetail(Job job){
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("jobName", job.getName());

        jobDataMap.put("jobLauncher", jobLauncher);
        jobDataMap.put("jobLocator", jobLocator);

        return JobBuilder.newJob(DaisoQuartzJob.class)
                .setJobData(jobDataMap)
                .withIdentity(job.getName())
                .build();
    }

    public Trigger makeJobTrigger(JobDetail jobDetail, String jobName, String cronStr){
        return TriggerBuilder.newTrigger()
                .forJob(jobDetail)
                .withIdentity(jobName)
                .withDescription(cronStr)
                .withSchedule(CronScheduleBuilder.cronSchedule(cronStr))
                .build();
    }

    public JobKey findJobByJobName(String jobName){
        try {
            for (JobKey jobKey : this.scheduler.getJobKeys(null)) {
                if (StringUtils.equalsIgnoreCase(jobKey.getName(), jobName)){
                    return jobKey;
                }
            }
        }catch (SchedulerException se){
            log.error("[[ERROR]] findJobByJobName !!! | jobId: {} | msg: {}",jobName,  se.getMessage());
        }
        return null;
    }

    public List<Map<String, Object>> listJobTrigger() throws SchedulerException {
        List<Map<String, Object>> out = new ArrayList<>() ;

        for (String name : this.scheduler.getTriggerGroupNames())
        {
            log.info("name:{}", name);
            Set<TriggerKey> keys = this.scheduler.getTriggerKeys(GroupMatcher.triggerGroupEquals(name));

            Trigger trigger;
            for(TriggerKey triggerkey : keys){
                trigger = this.scheduler.getTrigger(triggerkey);
                log.info("trigger:{}", trigger);
                log.info("getJobKey:{}", trigger.getJobKey());
                log.info("getPreviousFireTime:{}", trigger.getPreviousFireTime());
                log.info("trigger starttime:{}", trigger.getStartTime());
                log.info("trigger nexttime:{}", trigger.getNextFireTime());
                log.info("getDescription:{}", trigger.getDescription());

                Map<String, Object> map = new HashMap<>();
                map.put("name", trigger.getJobKey());
                map.put("starttime", trigger.getStartTime());
                map.put("prevtime", trigger.getPreviousFireTime());
                map.put("nexttime", trigger.getNextFireTime());
                map.put("desc", trigger.getDescription());
                out.add(map);
            }
        }
        return out;
    }


    public List<String> listJobName() {
        List<String> out = new ArrayList<>();

        try {
            for (JobKey jobKey : this.scheduler.getJobKeys(null)) {
                out.add(jobKey.getName());
            }
            return out;
        }catch (SchedulerException se){
            log.error("[[ERROR]] findJobByJobName !!! | msg: {}",se.getMessage());
        }
        return null;
    }


    public boolean addJob(String jobName, String cronStr, PThemeExhbtMgtVO themeExhbtMgtVO){
        try {
            JobDetail jobDetail = makeJobDetail(jobName, themeExhbtMgtVO);
            Trigger trigger = makeJobTrigger(jobDetail, jobName, cronStr);

            this.scheduler.scheduleJob(jobDetail, trigger);
        }
        catch (SchedulerException se){
            log.error("[[ERROR]] addJob !!! | jobId: {} | msg: {}", jobName,  se.getMessage());
            return false;
        }
        return true;
    }

    public boolean updateJob(String jobName,String cronStr, PThemeExhbtMgtVO themeExhbtMgtVO) {
        try {
            JobDetail jobDetail = makeJobDetail(jobName, themeExhbtMgtVO);
            Trigger trigger = makeJobTrigger(jobDetail, jobName,cronStr);
            scheduler.rescheduleJob(new TriggerKey(jobName), trigger);
        } catch (Exception se) {
            log.error("[[ERROR]] updateJob !!! | jobId: {} | msg: {}", jobName,  se.getMessage());
            return false;
        }
        return true;
    }

    public boolean deleteJob(String jobName){
        try {
            this.scheduler.deleteJob(findJobByJobName(jobName));
            return true;
        } catch (SchedulerException se){
            log.error("[[ERROR]] deleteJob !!! | jobId: {} | msg: {}", jobName,  se.getMessage());
            return false;
        }
    }

}
```

- MybatisMySqlConfig

```java
@Configuration
@MapperScan(basePackages = "kr.co.amugerna.batch.**.mapper.mysql", sqlSessionFactoryRef = "mySqlSessionFactory")
public class MybatisMySqlConfig {

    @Primary
    @Bean("mySqlDataSource")
    @ConfigurationProperties(prefix ="spring.datasource.mysql")
    public DataSource mysqlDataSource(){
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "mySqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(
            @Qualifier("mySqlDataSource") DataSource dataSource, ApplicationContext applicationContext) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setConfigLocation(applicationContext.getResource("classpath:mybatis-config.xml"));
        sqlSessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:mapper/oracle/**/*.xml"));

        return sqlSessionFactoryBean.getObject();
    }

    @Primary
    @Bean(name = "mySqlSessionTemplate")
    public SqlSessionTemplate sqlSessionTemplate(
            @Qualifier("mySqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

- ParameterAop

```java
//@Aspect
//@Component
@Slf4j
public class ParameterAop {

    @Pointcut("execution(* kr.co.amugerna.batch.controller.procedure.*Controller.*(..))")
    public void onRequest() { }

    @Pointcut("@annotation(org.springframework.web.bind.annotation.PostMapping)")
    public void postMapping() {}

    @Around("postMapping()")
    public Object ParameteAction(ProceedingJoinPoint joinPoint) throws Throwable {

        StopWatch stopWatch = new StopWatch();
	    stopWatch.start();

        Object result = null;
        try {
            result = joinPoint.proceed(joinPoint.getArgs());
            return result;
        } finally {
            ObjectMapper objectMapper = new ObjectMapper();
            stopWatch.stop();
            log.info("====================================================================");
            log.info("info: {}.{}",  joinPoint.getSignature().getDeclaringTypeName(), joinPoint.getSignature().getName());
            log.info("parameters : {}", objectMapper.writerWithDefaultPrettyPrinter().writeValueAsString(params(joinPoint)));
            log.info("response : {}", objectMapper.writerWithDefaultPrettyPrinter().writeValueAsString(result));
            log.info("controller time: {}", stopWatch.getTotalTimeMillis());
            log.info("controller time: {}", stopWatch.prettyPrint());
            log.info("====================================================================");
        }
    }

    private Map<String, Object> params(JoinPoint joinPoint) {
      CodeSignature codeSignature = (CodeSignature) joinPoint.getSignature();
      String[] parameterNames = codeSignature.getParameterNames();
      Object[] args = joinPoint.getArgs();
      Map<String, Object> params = new HashMap<>();
      for (int i = 0; i < parameterNames.length; i++) {
          params.put(parameterNames[i], args[i]);
      }
      return params;
    }
}
```

  </pre>
</details>