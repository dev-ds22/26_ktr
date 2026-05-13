---
layout: single
title: "[LOGGING] banner.txt 설정과 logback application.yml 설정"
excerpt: "banner.txt 설정과 logback application.yml 설정"

categories:
  - tech
tags:
  - [tech, spring boot, logback]

toc: false
toc_sticky: true

date: 2022-12-20
last_modified_at: 2022-12-20
---
# banner.txt 설정과 logback application.yml 설정

## Spring Boot Banner.txt (프로젝트 타이틀 콘솔 출력)

- src/main/resources 경로에 banner.txt 파일을 생성해주고 내용을 입력.
- 배너생성 1 : https://devops.datenkollektiv.de/banner.txt
- 배너생성 2 : http://patorjk.com/software/taag

- banner.txt 생성

```bash                                                                                
  ,--.  ,--.         ,--. ,--.           ,--.   ,--.                 ,--.    ,--. 
  |  '--'  |  ,---.  |  | |  |  ,---.    |  |   |  |  ,---.  ,--.--. |  |  ,-|  | 
  |  .--.  | | .-. : |  | |  | | .-. |   |  |.'.|  | | .-. | |  .--' |  | ' .-. | 
  |  |  |  | \   --. |  | |  | ' '-' '   |   ,'.   | ' '-' ' |  |    |  | \ `-' | 
  `--'  `--'  `----' `--' `--'  `---'    '--'   '--'  `---'  `--'    `--'  `---'  

  ${AnsiColor.BRIGHT_GREEN}:: Spring Boot ::${AnsiColor.DEFAULT}${spring-boot.formatted-version}
```

- banner.txt 파일에서 자동완성 단축키(ctrl + space) 사용가능.
- 글자색, 배경색도 지정가능.
- 그외 애플리케이션, 부트 관련 속성도 사용가능.

```bash
  ${application.formatted-version} : (v1.0.0)
  ${application.title} : My application
  ${application.version} : 1.0.0
  ${spring-boot.formatted-version} : (v2.2.5.RELEASE)
  ${spring-boot.version} : 2.2.5.RELEASE
```

## 참고 

- 이미지도 사용가능한 듯...
- application.yml 

```yaml
spring:
  banner:
    charset: UTF-8
    location: classpath:banner.txt
    image:
      location: classpath:banner.gif
      bitdepth: 1
      invert: false
      margin: 2
      pixelmode: text
      width: 5
      height: 5
```

# 로깅설정 logback-spring.xml 대신 application.yml 사용

## logback-spring.xml

```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <configuration>
      <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
          <encoder>
              <pattern>[%d{yy/MM/dd HH:mm:ss}] [%-4level] %logger.%method:%line - %msg%n </pattern>
          </encoder>
      </appender>

      <property name="LOG_PATH" value="prj-bo/logs"/>

      <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
          <file>${LOG_PATH}/logback.log</file>
          <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
              <Pattern>[%d{yy/MM/dd HH:mm:ss}] [%-4level] %logger.%method:%line - %msg%n</Pattern>
          </encoder>

          <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
              <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd-HH}.log</fileNamePattern>
  <!--            <fileNamePattern>${LOG_PATH}/%d{yyyy-MM-dd-HH}.%i.log</fileNamePattern>-->
  <!--            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">-->
  <!--                <maxFileSize>10MB</maxFileSize>-->
  <!--            </timeBasedFileNamingAndTriggeringPolicy>-->
          </rollingPolicy>
      </appender>

      <logger name="jdbc" level="OFF"/>
      <logger name="jdbc.sqlonly" level="OFF"/>
      <logger name="jdbc.sqltiming" level="DEBUG"/>
      <logger name="jdbc.audit" level="OFF"/>
      <logger name="jdbc.resultset" level="OFF"/>
      <logger name="jdbc.resultsettable" level="OFF"/>
      <logger name="jdbc.connection" level="OFF"/>
      
      <logger name="com.ulisesbocchio.jasyptspringboot" level="WARN"/>

      <root level="INFO">
          <appender-ref ref="CONSOLE" />
          <appender-ref ref="FILE" />
      </root>
  </configuration>
```

## application.yml

```yaml
  logging:
    pattern:
      file: "[%d{yy/MM/dd HH:mm:ss}] [%-4level] %logger.%method:%line - %msg%n"
      rolling-file-name: "/logs/daiso-bo/%d{yyyy-MM-dd-HH}_%i.log"
    file:
      name: /logs/daiso-bo/logback.log
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

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>