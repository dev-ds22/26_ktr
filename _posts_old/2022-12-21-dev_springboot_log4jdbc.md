---
layout: single
title: "[LOGGING] Spring Log4jdbc-log4j2"
excerpt: "Spring Boot Log4jdbc-log4j2 적용법"

categories:
  - tech
tags:
  - [tech, spring boot, Log4jdbc]

toc: false
toc_sticky: true

date: 2022-12-21
last_modified_at: 2022-12-21
---
# Log4jdbc-log4j2 적용법
 

## 1. 의존성 설정

- gradle

```bash
  implementation 'org.bgee.log4jdbc-log4j2:log4jdbc-log4j2-jdbc4.1:1.16'
```
  
- maven

```xml
  <dependency>
    <groupId>org.bgee.log4jdbc-log4j2</groupId>
    <artifactId>log4jdbc-log4j2-jdbcXX</artifactId>
    <version>1.16</version>
  </dependency>
```

## 2. log4jdbc.log4j2.properties 파일 추가

```bash
  log4jdbc.spylogdelegator.name=net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator
  log4jdbc.dump.sql.maxlinelength=0
```

- log4jdbc.spylogdelegator.name = net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator
  - **log4j2 대신 slf4j를 쓰고 싶을때 사용.**

>  
> First, you need to tell log4jdbc-log4j2 that you want to use  
> the SLF4J logger. You need to configure the option log4jdbc.  
> spylogdelegator.name to the value net.sf.log4jdbc.log.slf4j.Slf4jSpyLogDelegator.  
> This is done either via the log4jdbc.log4j2.properties file stored  
> in your classpath, or via system properties.
>

- log4j2로만 쓰거나 log4j2/slf4j를 같이 쓰면, 로그가 복잡해짐
  - log4j2로 출력되는 로그는 분류도 제대로 안되어있고, 양도 많음
  - log4j2로 쓰는 로그는 error 이상만, additivity는 false로 해서 콘솔 미출력 권장.

- log4jdbc.dump.sql.maxlinelength는 최대 sql 출력 길이. 
  - 0 설정시 다 출력.
  - 0 이외엔 설정한 길이까지만 출력.
  - default 값은 90L.

## 3. DB 관련 정보를 수정.

- application.yml 수정.

### 수정 전

```yaml
  spring:
    datasource:
      driver-class-name: oracle.jdbc.driver.OracleDriver
      hikari:
        jdbc-url: jdbc:oracle:thin:@localhost:1521:xe
```
  
### 수정 후

```yaml
  spring:
    datasource:
      driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy
      hikari:
        jdbc-url: jdbc:log4jdbc:oracle:thin:@localhost:1521:xe
```

## 4. LOGGING 설정

- Slf4j로 찍히는 log4jdbc 로그는 다음과 같다.
- jdbc.sqlonly	
  - SQL문만을 로그로 출력. 가독성을 위해, preparedStatement는 관련된 arguments 값으로 대체.
- jdbc.sqltiming
  - SQL문과 해당 SQL을 실행하는데 걸린 시간(millisecond) 포함.
- jdbc.audit
  - ResultSets를 제외한 모든 JBDC 호출 정보. JDBC 문제를 추적할 때 외엔 OFF.
- jdbc.resultset
  - JDBC 결과. ResultSet 오브젝트에 대한 모든 호출이 로깅.
- jdbc.resultsettable
  - JDBC 결과를 테이블로 기록. 
- jdbc.connection	
  - 수행 도중 열리고 닫히는 연결 내용. 연결 누수 문제를 찾는데에 유용.

### logback-spring.xml 수정
- **Slf4j 로깅은 logback으로 로그가 출력**

```xml
  <!--Log4jdbc-->
  <logger name="jdbc" level="OFF"/>
  <logger name="jdbc.connection" level="OFF"/>
  <logger name="jdbc.sqlonly" level="OFF"/>
  <logger name="jdbc.sqltiming" level="DEBUG" additivity="false">
    <appender-ref ref="DATABASE_FILE_APPENDER"/>
  </logger>
  <logger name="jdbc.audit" level="OFF"/>
  <logger name="jdbc.resultset" level="OFF"/>
  <logger name="jdbc.resultsettable" level="OFF"/>
```

### 로깅시 중복제거
- log4j 혹은 logback 설정을 통해 로그를 찍는 경우에 별다른 설정이 없으면 중복 로그가 발생.
- logger는 기본적으로 정의한 패키지의 상위로부터 모든 appender를 상속
- 이런 기본설정 때문에 중복로그 출력.

- **additivity="false"로 지정.**
  - 해당 attribute로 지정하면 로거는 상위로부터 내려오는 appender를 상속안함.

```xml
<logger name="jdbc" level="OFF" additivity="false" />

<logger name="com.tistory.reference" level="DEBUG" additivity="false">
	<appender-ref ref="STDOUT" />
</logger>

<logger name="jdbc.sqlonly" level="DEBUG" additivity="false">
	<appender-ref ref="STDOUT" />
</logger>

<logger name="jdbc.resultsettable" level="DEBUG" additivity="false">
	<appender-ref ref="STDOUT" />
</logger>

<root level="INFO">
	<appender-ref ref="STDOUT" />
</root>
```
  


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- logback-spring.xml

```bash
<?xml version="1.0" encoding="UTF-8"?  
<configuration
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender"
    <encoder
      <pattern>[%d{yy/MM/dd HH:mm:ss}] [%-4level] %logger.%method:%line - %msg%n </pattern
    </encoder
  </appender

  <property name="LOG_PATH" value="log-bo/logs"/

  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender"
    <file>${LOG_PATH}/logback.log</file
    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder"
      <Pattern>[%d{yy/MM/dd HH:mm:ss}] [%-4level] %logger.%method:%line - %msg%n</Pattern
    </encoder

    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy"
      <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd-HH}.log</fileNamePattern
      <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd-HH}.%i.log</fileNamePattern
      <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP"
        <maxFileSize>10MB</maxFileSize
      </timeBasedFileNamingAndTriggeringPolicy
    </rollingPolicy
  </appender

  <logger name="jdbc" level="OFF"/
  <logger name="jdbc.sqlonly" level="OFF"/
  <logger name="jdbc.sqltiming" level="DEBUG"/
  <logger name="jdbc.audit" level="OFF"/
  <logger name="jdbc.resultset" level="OFF"/
  <logger name="jdbc.resultsettable" level="OFF"/
  <logger name="jdbc.connection" level="OFF"/
  <logger name="com.ulisesbocchio.jasyptspringboot" level="WARN"/
  
  <root level="INFO"
    <appender-ref ref="CONSOLE" /
    <appender-ref ref="FILE" /
  </root
</configuration
```

- application.yml 로 대체

```yaml
logging:
  pattern:
    file: "[%d{yy/MM/dd HH:mm:ss}] [%-4level] %logger.%method:%line - %msg%n"
    rolling-file-name: "/logs/log-bo/%d{yyyy-MM-dd-HH}_%i.log"
  file:
    name: /logs/log-bo/logback.log
    max-history: 30
  level:
    root: INFO
    org:
      springframework: INFO 
    jdbc: 
      root: ERROR
      sqlonly: ERROR
      sqltiming: DEBUG
      audit: ERROR
      resultset: ERROR
      resultsettable: DEBUG
      connection: ERROR
    com:
      ulisesbocchio:
        jasyptspringboot: WARN
```

  </pre>
</details>