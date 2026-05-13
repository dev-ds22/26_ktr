---
layout: single
title: "[Spring Boot] Spring Mybatis Config (feat. HikariCP)"
excerpt: "Spring Boot DB Config & HikariCP"

categories:
  - tech
tags:
  - [tech, spring boot, mybatis, hikaricp]

toc: false
toc_sticky: true

date: 2022-12-21
last_modified_at: 2022-12-21
---
# [Spring Boot] Mybatis config

- Spring Boot에서 xml설정대신 config를 이용하여 설정. 
- xml과의 연결은 @Mapper를 이용.

- @Mapper를 사용시 주의점.
  - 1. xml의 id와 Mapper의 메소드명 일치.
  - 2. xml의 namespace에 실제 Mapper의 경로 지정.
  - 3. Mapper가 xml을 찾을 수 있도록 경로 설정필요.

## MyBatis 개요

- Spring 프레임워크등에서 웹 어플리케이션 개발시 사용할 수 있는 추가 프레임워크.
- iBatis라는 곳에서 제공하였었는데 구글이 인수후 MyBatis라는 이름으로 지원.

## Maven 라이브러리 추가

- gradle

```xml
 implementation('org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0')
```
  

- Maven

```xml
  <dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
  </dependency>
```

## DB Connection 정보 (application.yml)

- application.yml 파일에 DB 커넥션 정보를 작성.

```yaml
  datasource:
    oracle:
      driver-class-name: oracle.jdbc.OracleDriver
      jdbc-url: jdbc:oracle:thin:@localhost:1521/xe
      username: system
      password: oracle
      validationQuery: select 1 from dual
      test-on-borrow: true
```


## Config

```java
  @Configuration
  @MapperScan(basePackages = {"mapper들이 있는 패키지경로"})
  public class DBConfiguration {

    @Value("${spring.datasource.driver-class-name}")
    private String driverClassName;
    
    @Value("${spring.datasource.url}")
    private String url;
    
    @Value("${spring.datasource.username}")
    private String username;
    
    @Value("${spring.datasource.password}")
    private String password;

    @Autowired
    ApplicationContext applicationContext;
    
    @Bean
    public DataSource dataSource() {
      DriverManagerDataSource dataSource = new DriverManagerDataSource();
      dataSource.setDriverClassName(driverClassName);
      dataSource.setUrl(url);
      dataSource.setUsername(username);
      dataSource.setPassword(password);

      return dataSource;
    }

    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) throws IOException {
      SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();
      factoryBean.setDataSource(dataSource);
      factoryBean.setConfigLocation(applicationContext.getResource("classpath:/mybatis/mybatis-config.xml"));
      factoryBean.setMapperLocations(applicationContext.getResources("classpath:/mapper/**/*Mapper.xml"));
      return factoryBean;
    }

    @Bean
    public SqlSessionTemplate sqlSession(SqlSessionFactory sqlSessionFactory) {
      return new SqlSessionTemplate(sqlSessionFactory);
    }

  }
```

- @MapperScan
  - @Mapper 어노테이션이 있는 클래스를 찾아 Bean 등록.
- DataSource
  - application.yml의 설정값을 읽어 dataSource 빈 생성.
- SqlSessionFactory
  - DataSource를 이용하여 mysql서버와 mybatis를 연결해준다.
  - SqlSessionFactory는 데이터베이스와의 연결과 SQL의 실행에 대한 모든 것을 가진 가장 중요한 객체.

## mybatis설정 파일 등록. (mybatis-config.xml)

```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
  <configuration>
    <settings>
      <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
  </configuration>
```

- mapUnderScoreToCamelCase는 USER_ID -> userId처럼 변환하여 돌려준다.

## SqlSessionTemplate

- SqlSessionTemplate는 mybatis의 쿼리문을 수행하는 역할을 한다.

## Mapper 인터페이스

```java
  @Mapper
  public interface UserMapper {
    List<UserVO> test() throws Exception;
  }
```

## XML 파일

```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  <mapper namespace="패키지경로.UserMapper">
    
    <select id="test" resultType="패키지경로.UserVO">
      SELECT *
      FROM USER
    </select>

  </mapper>
```

- 1. Mapper 인터페이스의 메소드명과 XML의 id가 같아함.
- 2. Mapper 인터페이스의 풀 경로를 XML 파일의 namespace에 선언필요.
  

# HikariCP

## DBCP (DataBase Connection Pool)
- DBCP(DataBase Connection Pool) : DB와 커넥션을 맺고 있는 객체를 관리하는 역할.
- WAS 실행 시 미리 일정량의 DB Connection 객체를 생성하고 Pool에 저장. 
- DB 연결 요청이 있으면, 이 Pool에서 Connection 객체를 가져다 사용후 반환.
- DBCP를 사용하는 가장큰 이유는 효율성.

## HikariCP
- 스프링 부트 2.0부터 설정된 default JDBC Connection Pool.
- 기존의 DBCP들 보다 빠른 작업 처리 및 경량.

- application.yml

```yaml
spring: 
  datasource: 
    hikari: 
      driver-class-name: com.mysql.cj.jdbc.Driver 
      username: {id} 
      password: {password} 
      jdbc-url: jdbc:mysql://{url}:{port}/{db스키마} 
      minimum-idle: 5 
      maximum-pool-size: 10
      idle-timeout: 30000 
      pool-name: DevLogHikariCP 
      max-lifetime: 200000 
      connection-timeout: 30000 
      connection-test-query: /*CONNECTION TEST QUERY*/SELECT NOW() FROM DUAL

#    hikari:
#      driver-class-name: oracle.jdbc.OracleDriver
#      jdbc-url: jdbc:oracle:thin:@localhost:1521/xe
#      username: system
#      password: oracle
#      connectionTimeout: 30000
#      maximumPoolSize: 10
#      maxLifetime: 1800000
#      poolName: HikariCP
#      readOnly: false
#      connectionTestQuery: SELECT 1 from dual      
```
  
- minimum-idle : Connection Pool에 유지 가능한 최소 커넥션 개수
- maximum-pool-size : Connection Pool에 유지 가능한 최대 커넥션 개수
- idle-timeout : Connection이 Poll에서 유휴상태(사용하지 않는 상태)로 남을 수 있는 최대 시간
- pool-name : Connction Pool 이름
- max-lifetime : Connection의 최대 유지 가능 시간
- connection-timeout : Pool에서 Connection을 구할 때 대기시간, 대기시간안에 구하지 못하면 Exception
- connection-test-query : Connection이 잘 되었는지 확인하는 TEST SQL

## Config

```java
@Configuration 
@MapperScan(basePackages = {"mapper들이 있는 패키지경로"}) 
public class DBConfiguration { 
  @Autowired 
  ApplicationContext applicationContext; 
  
  @Bean 
  @ConfigurationProperties(prefix="spring.datasource.hikari") 
  public HikariConfig hikariConfig() { 
    return new HikariConfig(); 
  }
  
  @Bean 
  public DataSource dataSource() { 
    DataSource dataSource = new HikariDataSource(hikariConfig()); 
      System.out.println("DataSource connection : " + dataSource.toString()); 
      return dataSource;
  } 
  
  @Bean 
  public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource) throws IOException { 
    SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean(); 
      factoryBean.setDataSource(dataSource); 
      factoryBean.setConfigLocation(applicationContext.getResource("classpath:/mybatis/mybatis-config.xml")); 
      factoryBean.setMapperLocations(applicationContext.getResources("classpath:/mapper/**/*Mapper.xml")); 
      return factoryBean; 
  } 
  
  @Bean 
  public SqlSessionTemplate sqlSession(SqlSessionFactory sqlSessionFactory) {
    return new SqlSessionTemplate(sqlSessionFactory); 
  }
}
```
  
- @ConfigurationProperties : application.yml에서 spring.datasource.hikari.* 에 해당하는 값을 바인딩.


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- application.yml

```yaml
  datasource:
#    hikari:
#      driver-class-name: oracle.jdbc.OracleDriver
#      jdbc-url: ENC(FgTGXjDOkIoVK4MxGDcB5ifrGJW3JiAgPBI1uwiL+HJ3mPXUzXbky/utkvGNvcM87LcDKYN8H7c=)
#      username: ENC(dNkCYMuu6fhfb1VPU1/HCodku40TpQou)
#      password: ENC(ASBMixcTxKCAWQSlPA9kN6I7IV9eRqzT)
#      connectionTimeout: 30000
#      maximumPoolSize: 10
#      maxLifetime: 1800000
#      poolName: HikariCP
#      readOnly: false
#      connectionTestQuery: SELECT 1 from dual
    oracle:
      driver-class-name: oracle.jdbc.OracleDriver
#      jdbc-url: ENC(FgTGXjDOkIoVK4MxGDcB5ifrGJW3JiAgPBI1uwiL+HJ3mPXUzXbky/utkvGNvcM87LcDKYN8H7c=)
#      username: ENC(dNkCYMuu6fhfb1VPU1/HCodku40TpQou)
#      password: ENC(ASBMixcTxKCAWQSlPA9kN6I7IV9eRqzT)
      jdbc-url: jdbc:oracle:thin:@localhost:1521/xe
      username: system
      password: oracle
      validationQuery: select 1 from dual
      test-on-borrow: true
```
  

- MybatisOracleConfig
  
```java
  @Configuration
  @MapperScan(basePackages = "kr.co.daiso.bo.**.mapper.oracle")
  public class MybatisOracleConfig {

    @Bean("oracleDataSource")
    @ConfigurationProperties(prefix ="spring.datasource.oracle")

    public DataSource oracleDataSource(){
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "oracleSqlSessionFactory")
    public SqlSessionFactory sqlSessionFactory(
            @Qualifier("oracleDataSource") DataSource dataSource, ApplicationContext applicationContext) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        sqlSessionFactoryBean.setConfigLocation(applicationContext.getResource("classpath:/mybatis-config.xml"));
        sqlSessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:/mapper/oracle/**/*.xml"));

        return sqlSessionFactoryBean.getObject();
    }

    @Bean(name = "oracleSqlSessionTemplate")
    public SqlSessionTemplate sqlSessionTemplate(
            @Qualifier("oracleSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```
  

- mybatis-config.xml
  
```bash
  <?xml version="1.0" encoding="UTF-8"?
  <!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "HTTP://mybatis.org/dtd/mybatis-3-config.dtd"

  <configuration 
      <settings 
          <setting name="cacheEnabled" value="true" /
          <setting name="lazyLoadingEnabled" value="true" /
          <setting name="aggressiveLazyLoading" value="true" /
          <setting name="defaultExecutorType" value="REUSE" /
          <setting name="defaultStatementTimeout" value="300" /
          <setting name="localCacheScope" value="SESSION" /
          <setting name="callSettersOnNulls" value="true" /
          <setting name="jdbcTypeForNull" value="NULL" /
          <setting name="useGeneratedKeys" value="true" /
          <setting name="mapUnderscoreToCamelCase" value="true"/
      </settings 

      <plugins
          <plugin interceptor="kr.co.common.interceptor.MybatisInterceptor" /
          <plugin interceptor="kr.co.common.interceptor.MaskingInterceptor" /
      </plugins
  </configuration
```

  </pre>
</details>