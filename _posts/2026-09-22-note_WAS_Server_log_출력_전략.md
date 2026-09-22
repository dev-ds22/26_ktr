---
layout: single
title: "WAS_Server_log_출력_전략"
excerpt: "WAS_Server_log_출력_전략"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-22"
last_modified_at: "2026-09-22 16:27:45 +0900"
mermaid: false
---
# 1. 결론

현재 구조에서는 **Application → Console → JBoss Logging → server.log/server.out** 경로가 열려 있기 때문에 같은 애플리케이션 로그가 3곳에 중복되는 것이 자연스러운 상태입니다.

운영 환경에서는 다음 구조를 권장합니다.

| 로그 | 주 역할 | 정상 INFO | WARN | ERROR/FATAL | 중복 허용 |
|---|---|---:|---:|---:|---|
| `bk-fo.log` | **Application 전용 원본** | O | O | O | 원칙적으로 없음 |
| `server.log` | **JBoss/EAP/WAS 전용 원본** | O | O | O | `server.out`과 ERROR 이상만 허용 |
| `server.out` | **기동/표준출력/비상진단용** | 최소화 | 선택 | O | `server.log`의 ERROR/FATAL 허용 |

가장 중요한 변경은 두 가지입니다.

```text
1. Application Log4j2에서 ConsoleAppender 제거
   → bk-fo.log만 기록

2. JBoss CONSOLE Handler를 INFO → ERROR로 변경
   → JBoss INFO/WARN은 server.log만
   → JBoss ERROR/FATAL만 server.log + server.out
```

즉 최종적으로:

```text
Application 정상 로그
        │
        └──────────────→ bk-fo.log


JBoss/EAP INFO/WARN
        │
        └──────────────→ server.log


JBoss/EAP ERROR/FATAL
        │
        ├──────────────→ server.log
        │
        └── CONSOLE ───→ server.out


JVM/기동 Script/stdout/stderr
        │
        └──────────────→ server.out
                         또는 EAP stdout/stderr 로깅 경로
```

이 구성이 세 로그의 목적을 가장 명확하게 나눕니다.

---

# 2. 현재 3중 로그가 발생하는 핵심 원인

현재 Application `log4j.xml`의 Root가 다음과 같습니다.

```xml
<Root level="INFO">
    <AppenderRef ref="console" />
    <AppenderRef ref="File_Appender"/>
</Root>
```

즉 Application 로그 하나가 발생하면 동시에:

```text
Application Logger
      │
      ├── File_Appender
      │       └── bk-fo.log
      │
      └── Console
              └── SYSTEM_OUT
                     │
                     └── JBoss/프로세스 표준출력
```

으로 갑니다.

또한 개별 Logger도 똑같습니다.

```xml
<Logger name="org.springframework" level="ERROR" additivity="false">
    <AppenderRef ref="console" />
    <AppenderRef ref="File_Appender"/>
</Logger>
```

따라서 Application이 의도적으로 **파일과 stdout에 같은 로그를 두 번 보내고 있습니다.**

Apache Log4j2도 Root Logger에 ConsoleAppender가 연결되어 있으면 해당 로그가 ConsoleAppender로 전달되는 구조라고 설명합니다. :chatgpt-content-reference{index="0"}

여기에 JBoss 측에서도:

```properties
logger.handlers=FILE,CONSOLE
```

이므로 JBoss가 처리하는 로그는 다시:

```text
JBoss Logger
   │
   ├── FILE
   │    └── server.log
   │
   └── CONSOLE
        └── SYSTEM_OUT
             └── server.out
```

이 됩니다.

Red Hat 문서에서도 EAP Root Logger가 기본적으로 Console Handler와 File Handler를 사용하고, File Handler가 `server.log`를 생성한다고 설명합니다. :chatgpt-content-reference{index="1"}

그래서 현재 현상은 설정상 충분히 설명됩니다.

---

# 3. 먼저 바로잡아야 할 중요한 부분: `logging.properties`

현재 확인하신:

```text
/configuration/logging.properties
```

는 EAP 7.2에서는 원칙적으로 **부팅 초기 Logging 설정**입니다.

Red Hat 공식 설명은:

> `logging.properties`는 서버 부팅 시 사용되고 Logging Subsystem이 시작되면 Logging Subsystem이 설정을 인계한다.

라는 구조입니다. 또한 수동 수정한 `logging.properties`가 시작 과정에서 다시 생성/덮어써질 수 있으므로 일반적인 운영 설정은 Logging Subsystem에서 관리하는 것이 맞습니다. :chatgpt-content-reference{index="2"}

따라서 실제 운영 중 `server.log` 설정의 본체를 먼저 확인하십시오.

```bash
$JBOSS_HOME/bin/jboss-cli.sh --connect
```

그리고:

```text
/subsystem=logging/root-logger=ROOT:read-resource(recursive=true)
```

Console Handler:

```text
/subsystem=logging/console-handler=CONSOLE:read-resource(recursive=true)
```

File Handler:

```text
/subsystem=logging/periodic-rotating-file-handler=FILE:read-resource(recursive=true)
```

전체를 한 번에 보면:

```text
/subsystem=logging:read-resource(recursive=true)
```

가장 중요합니다.

---

# 4. 권장하는 3개 로그의 역할

## 4-1. `bk-fo.log` — Application의 원본 로그

여기에는 다음 계층에서 발생한 업무/애플리케이션 로그를 기록합니다.

```text
Controller
Service
DAO
Application Service
Batch
외부 API 호출
업무 예외
Application Exception
Spring Application 오류
DB 호출 오류
```

예:

```text
INFO  주문 처리 시작
INFO  회원 상태 변경
WARN  외부 API 응답 지연
ERROR 주문 저장 실패
ERROR SQLException 발생
```

반대로 이런 것은 가능하면 넣지 않습니다.

```text
JBoss 기동
Undertow listener
Infinispan
JGroups
JBoss transaction manager 자체 상태
JBoss deployment infrastructure
XNIO
WFLYSRVxxxx
WFLYUTxxxx
```

즉:

> 애플리케이션 개발자가 확인하는 로그 = `bk-fo.log`

로 만드는 것이 좋습니다.

---

# 5. `server.log` — JBoss/EAP 인프라 원본

`server.log`는 WAS 운영 로그로 정의합니다.

대표적으로:

```text
JBoss boot
deployment / undeployment
datasource
connection pool
Undertow
XNIO
transaction manager
JGroups
Infinispan
EJB
thread pool
socket/listener
server shutdown
WFLYxxxx
JBASxxxx
```

등입니다.

Red Hat 역시 `server.log`를 EAP의 기본 server log로 정의하고 있습니다. :chatgpt-content-reference{index="3"}

즉:

> WAS 운영자가 보는 로그 = `server.log`

입니다.

여기에 일반적인 Application INFO가 들어오는 것은 가급적 없애는 것이 좋습니다.

---

# 6. `server.out` — 정상 운영 로그가 아니라 비상 진단 로그

`server.out`은 가장 중요한 관점 변화가 필요합니다.

`server.out`을 세 번째 정상 로그 파일로 취급하지 않는 것을 권장합니다.

용도는:

```text
JBoss 시작 전/초기 stdout
기동 shell 출력
JVM stdout/stderr
치명적인 console error
logging subsystem 자체 문제 발생 시 fallback
System.out/System.err을 직접 사용하는 비정상 코드
```

정도로 제한합니다.

따라서 정상 운영 상태에서:

```text
server.out 크기 << server.log 크기
```

가 되는 것이 바람직합니다.

특히 JBoss Console Handler가 기본적으로 `System.out`을 대상으로 할 수 있다는 것은 Red Hat 공식 설정에도 명시되어 있습니다. :chatgpt-content-reference{index="4"}

---

# 7. Application `log4j.xml`에서 가장 먼저 할 변경

현재:

```xml
<Console name="console" target="SYSTEM_OUT">
    <PatternLayout pattern="%d %5p [%c] %m%n" />
</Console>
```

를 운영에서는 제거합니다.

그리고 모든:

```xml
<AppenderRef ref="console" />
```

도 제거합니다.

## 7-1. 운영용 권장 `log4j.xml`

현재 구조를 최대한 유지하면서 수정하면 다음 정도가 안전합니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Properties>
        <Property name="logNm">bk-fo</Property>
        <Property name="layoutPattern">%d{yyyy/MM/dd HH:mm:ss,SSS} [%-5p] [%c] [%t] %m%throwable%n</Property>
    </Properties>

    <Appenders>
        <RollingFile name="File_Appender"
                     fileName="/data/app_log/${logNm}/${logNm}.log"
                     filePattern="/data/app_log/${logNm}/${logNm}_%d{yyyy-MM-dd}_%i.log.gz">

            <PatternLayout pattern="${layoutPattern}"/>

            <Policies>
                <SizeBasedTriggeringPolicy size="100000KB"/>
                <TimeBasedTriggeringPolicy interval="1"/>
            </Policies>

            <DefaultRolloverStrategy max="10" fileIndex="min"/>
        </RollingFile>
    </Appenders>

    <Loggers>

        <Logger name="java.sql" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="egovframework" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="org.egovframe" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <!-- 운영환경에서는 SQL 전체 출력 금지 권장 -->
        <Logger name="jdbc.sqlonly" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="jdbc.sqltiming" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="jdbc.audit" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="jdbc.resultset" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="jdbc.resultsettable" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Logger name="org.springframework" level="ERROR" additivity="false">
            <AppenderRef ref="File_Appender"/>
        </Logger>

        <Root level="INFO">
            <AppenderRef ref="File_Appender"/>
        </Root>

    </Loggers>
</Configuration>
```

이것만으로도 가장 큰 변화가 생깁니다.

기존:

```text
Application
 ├─ bk-fo.log
 └─ stdout
      ├─ server.log
      └─ server.out
```

변경:

```text
Application
 └──────────→ bk-fo.log
```

가 됩니다.

---

# 8. 위 `log4j.xml`에서 같이 개선한 부분

현재 설정에는 `java.sql`이 **두 번 선언되어 있습니다.**

현재:

```xml
<Logger name="java.sql" ...>
```

가 앞부분과 SQL 부분에 중복되어 있습니다.

하나는 제거해야 합니다.

또 현재:

```xml
<Logger name="jdbc.sqlonly" level="DEBUG">
```

가 운영 서버에 적용되어 있다면 좋지 않습니다.

`jdbc.sqlonly=DEBUG`는 SQL을 상당히 많이 발생시킬 수 있으므로:

```xml
level="ERROR"
```

또는 SQL 출력이 전혀 필요 없다면:

```xml
level="OFF"
```

를 권장합니다.

특히 SQL에는 개인정보, 검색조건, ID 등의 값이 포함될 가능성이 있으므로 운영에서 DEBUG SQL 로그를 상시 켜는 것은 디스크뿐 아니라 보안 측면에서도 불리합니다.

---

# 9. 기존 파일 로그에서 색상 Pattern도 제거 권장

현재:

```xml
%style{...}{cyan}
%highlight{...}
```

를 사용하고 있습니다.

이런 ANSI 색상 처리는 콘솔용입니다.

`bk-fo.log` 같은 파일 로그에는:

```text
%d{yyyy/MM/dd HH:mm:ss,SSS} [%-5p] [%c] [%t] %m%throwable%n
```

정도의 plain format이 낫습니다.

예:

```text
2026/09/22 15:45:13,201 [INFO ] [com.xxx.OrderService] [http-nio-8080-exec-3] order start
```

로그 수집, grep, ELK/OpenSearch, 장애 분석에도 이쪽이 더 안정적입니다.

---

# 10. JBoss는 `FILE=INFO`, `CONSOLE=ERROR` 전략

현재 JBoss bootstrap 설정은:

```properties
logger.level=INFO
logger.handlers=FILE,CONSOLE
```

이고:

```properties
handler.CONSOLE.level=INFO
```

입니다.

따라서 INFO 이상의 JBoss 로그가:

```text
FILE      → server.log
CONSOLE   → server.out
```

양쪽에 모두 들어갑니다.

이것이 JBoss 자체 로그의 `server.log ↔ server.out` 중복 원인입니다.

### 10-1-1. 권장

```properties
handler.CONSOLE.level=ERROR
```

로 변경합니다.

즉 bootstrap 기준으로:

```properties
loggers=sun.rmi,org.jboss.as.config,com.arjuna

logger.level=INFO
logger.handlers=FILE,CONSOLE

logger.sun.rmi.level=WARN
logger.sun.rmi.useParentHandlers=true

logger.org.jboss.as.config.level=DEBUG
logger.org.jboss.as.config.useParentHandlers=true

logger.com.arjuna.level=WARN
logger.com.arjuna.useParentHandlers=true

handler.CONSOLE=org.jboss.logmanager.handlers.ConsoleHandler
handler.CONSOLE.level=ERROR
handler.CONSOLE.formatter=COLOR-PATTERN
handler.CONSOLE.properties=autoFlush,target,enabled
handler.CONSOLE.autoFlush=true
handler.CONSOLE.target=SYSTEM_OUT
handler.CONSOLE.enabled=true

handler.FILE=org.jboss.logmanager.handlers.PeriodicRotatingFileHandler
handler.FILE.level=ALL
handler.FILE.formatter=PATTERN
handler.FILE.properties=append,autoFlush,enabled,suffix,fileName
handler.FILE.constructorProperties=fileName,append
handler.FILE.append=true
handler.FILE.autoFlush=true
handler.FILE.enabled=true
handler.FILE.suffix=.yyyy-MM-dd
handler.FILE.fileName=/log/fo-01/server.log

formatter.PATTERN=org.jboss.logmanager.formatters.PatternFormatter
formatter.PATTERN.properties=pattern
formatter.PATTERN.pattern=%d{yyyy-MM-dd HH\:mm\:ss,SSS} %-5p [%c] (%t) %s%e%n

formatter.COLOR-PATTERN=org.jboss.logmanager.formatters.PatternFormatter
formatter.COLOR-PATTERN.properties=pattern
formatter.COLOR-PATTERN.pattern=%K{level}%d{HH\:mm\:ss,SSS} %-5p [%c] (%t) %s%e%n
```

실질적인 변경점은 딱 이것입니다.

```diff
-handler.CONSOLE.level=INFO
+handler.CONSOLE.level=ERROR
```

그러나 앞에서 설명했듯 **이 파일만 수정해서는 안 됩니다.**

정상 기동 후 Logging Subsystem 설정도 동일하게 맞춰야 합니다.

---

# 11. 실제 JBoss 운영 설정은 CLI에서 변경

먼저 현재 상태 확인:

```text
/subsystem=logging/console-handler=CONSOLE:read-resource(recursive=true)
```

예상:

```text
level => INFO
target => System.out
```

이면 다음 변경을 권장합니다.

```text
/subsystem=logging/console-handler=CONSOLE:write-attribute(name=level,value=ERROR)
```

Root는 그대로:

```text
INFO
```

를 유지합니다.

즉:

```text
ROOT INFO
   │
   ├── FILE ALL/INFO
   │      └── server.log : INFO/WARN/ERROR/FATAL
   │
   └── CONSOLE ERROR
          └── server.out : ERROR/FATAL
```

JBoss Handler는 자신의 level 이상의 메시지만 처리하기 때문에 `CONSOLE=ERROR`로 설정하면 INFO/WARN은 Console Handler에서 버려지고 ERROR/FATAL만 출력됩니다. :chatgpt-content-reference{index="5"}

이 방식이 `CONSOLE` 자체를 완전히 삭제하는 것보다 낫습니다.

왜냐하면 Logging/File Handler 쪽에 문제가 생겼을 때 `server.out`이 **비상 진단 경로** 역할을 할 수 있기 때문입니다.

---

# 12. 왜 CONSOLE을 완전히 제거하지 않고 ERROR를 남기는가

제가 권장하는 의도적인 중복입니다.

예를 들어 JBoss 자체에 심각한 문제가 발생했다고 가정하면:

```text
ERROR [org.jboss...] datasource failure
```

이를:

```text
server.log O
server.out O
```

두 군데 남기는 것은 운영상 의미가 있습니다.

반대로:

```text
INFO deploy success
INFO listener started
INFO transaction subsystem started
```

같은 정상 정보는:

```text
server.log O
server.out X
```

이면 충분합니다.

따라서 중복 허용 정책은 다음이 가장 적절합니다.

| 이벤트 | bk-fo.log | server.log | server.out |
|---|---:|---:|---:|
| Application INFO | O | X | X |
| Application WARN | O | X | X |
| Application ERROR | O | 원칙 X | 원칙 X |
| JBoss INFO | X | O | X |
| JBoss WARN | X | O | X |
| JBoss ERROR/FATAL | X | O | **O** |
| JBoss startup infrastructure | X | O | 필요 시 O |
| JVM/native/boot 문제 | X | 경우에 따라 O | O |
| 직접 `System.out` | 사용 금지 | 환경에 따라 | 환경에 따라 |
| 직접 `System.err` | 사용 금지 | 환경에 따라 | 환경에 따라 |

이것이 제가 생각하는 가장 실무적인 중복 정책입니다.

---

# 13. Application ERROR를 server.log에도 일부러 넣을 필요는 없음

여기서는 **굳이 넣지 않는 것을 권장**합니다.

예를 들어:

```java
log.error("주문 처리 실패", e);
```

라면:

```text
bk-fo.log
```

하나만으로 충분합니다.

JBoss까지 같은 Exception을 다시 보내면 다시:

```text
bk-fo.log
server.log
server.out
```

3중화가 시작됩니다.

다만 다음과 같이 Application 오류가 WAS 경계를 넘어간 경우는 다릅니다.

```text
Deployment 실패
Servlet 초기화 실패
Spring Context startup 실패
ClassNotFoundException
NoClassDefFoundError
OutOfMemoryError
Datasource startup 실패
Transaction subsystem 실패
```

이런 오류는 JBoss가 직접 감지해 `server.log`에 남기는 것이 정상입니다.

이것은 **의도적인 중복이 아니라 서로 다른 계층에서 같은 장애를 관찰한 것**이므로 허용하는 편이 좋습니다.

---

# 14. `System.out.println()`과 `printStackTrace()`는 운영 코드에서 제거

특히 다음 코드를 찾아보는 것을 권장합니다.

```bash
grep -R "System.out.print" <SOURCE_PATH>
```

```bash
grep -R "System.err.print" <SOURCE_PATH>
```

```bash
grep -R "printStackTrace" <SOURCE_PATH>
```

운영 코드에:

```java
System.out.println(...)
```

```java
e.printStackTrace()
```

가 있다면 모두 Log4j2 Logger로 전환하는 것이 좋습니다.

예:

```java
log.error("외부 API 호출 실패", e);
```

EAP에서는 stdout/stderr 자체가 JBoss LogManager category로 처리되어 `server.log`에 들어가는 구성이 가능하며 Red Hat도 `stdout`, `stderr` category를 별도로 다루는 방법을 문서화하고 있습니다. :chatgpt-content-reference{index="6"}

따라서:

> `System.out`을 server.out 전용으로 쓰자

라는 설계 자체를 하지 않는 것이 좋습니다.

`server.out`은 어디까지나 fallback으로 봐야 합니다.

---

# 15. 한 단계 더 좋은 Application Logger 구조

현재 설정에는 약간의 구조적 문제가 하나 더 있습니다.

```xml
<Root level="INFO">
    <AppenderRef ref="File_Appender"/>
</Root>
```

이면 Application뿐 아니라 별도로 제어하지 않은 모든 library의 INFO 로그가 `bk-fo.log`에 들어갈 수 있습니다.

가장 이상적인 구조는 **자사 Application package만 INFO**, 나머지는 WARN 또는 ERROR입니다.

예를 들어 실제 Application base package가:

```text
kr.co.xxx
```

라면:

```xml
<Logger name="kr.co.xxx" level="INFO" additivity="false">
    <AppenderRef ref="File_Appender"/>
</Logger>

<Root level="WARN">
    <AppenderRef ref="File_Appender"/>
</Root>
```

가 더 좋습니다.

그러면:

```text
kr.co.xxx.*        INFO 이상
Spring             ERROR
eGov               ERROR
기타 라이브러리    WARN 이상
```

이 됩니다.

다만 **현재 Application base package를 제가 알 수 없기 때문에 이 부분은 바로 적용하지 말고**, 먼저 ConsoleAppender 제거만 하는 것이 안전합니다.

---

# 16. 변경은 3단계로 하는 것이 가장 안전

운영 시스템이라면 한 번에 많은 것을 바꾸는 것보다 다음 순서가 좋습니다.

| 단계 | 변경 | 위험도 | 효과 |
|---|---|---:|---|
| 1 | Application Log4j2 `ConsoleAppender` 제거 | 낮음 | 3중 로그의 핵심 원인 제거 |
| 2 | JBoss `CONSOLE` INFO → ERROR | 낮음 | server.log/server.out 중복 대폭 감소 |
| 3 | Application Root INFO → Application package INFO + Root WARN | 중간 | bk-fo.log 노이즈 추가 감소 |

특히 **1단계만으로도 상당한 중복이 제거될 가능성이 높습니다.**

---

# 17. 변경 후 반드시 검증할 테스트

운영 적용 전에 DEV에서 다음 네 개의 문자열을 각각 발생시키는 것이 가장 확실합니다.

```java
log.info("LOG_TEST_APP_INFO");
log.error("LOG_TEST_APP_ERROR");
System.out.println("LOG_TEST_STDOUT");
System.err.println("LOG_TEST_STDERR");
```

그리고:

```bash
grep "LOG_TEST_" /data/app_log/bk-fo/bk-fo.log
```

```bash
grep "LOG_TEST_" /log/fo-01/server.log
```

```bash
grep "LOG_TEST_" <server.out 경로>
```

정상적인 목표 결과는:

| 테스트 | bk-fo.log | server.log | server.out |
|---|---:|---:|---:|
| `APP_INFO` | O | X | X |
| `APP_ERROR` | O | X* | X* |
| `STDOUT` | X | EAP 설정에 따라 가능 | EAP/실행환경에 따라 가능 |
| `STDERR` | X | EAP 설정에 따라 가능 | EAP/실행환경에 따라 가능 |

`*` Application Exception이 container까지 전파된 경우에는 JBoss가 자체 오류로 다시 기록할 수 있으므로 server.log에 나타나는 것은 정상일 수 있습니다.

---

# 18. ConsoleAppender를 제거했는데도 Application 로그가 server.log에 남는다면

이 경우에는 **두 번째 로깅 경로가 존재하는 것**입니다.

JBoss EAP는 Application logging을 Logging Subsystem에서 처리할 수도 있고 Deployment별 logging configuration을 사용할 수도 있습니다. Red Hat 문서에서도 per-deployment logging이 없으면 Logging Subsystem 설정이 Application에도 적용될 수 있다고 설명합니다. :chatgpt-content-reference{index="7"}

이 경우 다음을 확인합니다.

```text
/subsystem=logging:read-attribute(name=use-deployment-logging-config)
```

그리고 deployment logging 상태:

```text
/deployment=<배포WAR명>/subsystem=logging:read-resource(recursive=true,include-runtime=true)
```

또는 deployment logging configuration을 확인한 후:

```text
/deployment=<배포WAR명>/subsystem=logging/configuration=<CONFIG>:read-resource(recursive=true,include-runtime=true)
```

를 사용할 수 있습니다.

Red Hat EAP 7.2에서도 공식적으로 이 방식으로 deployment가 `default`, logging profile 또는 deployment별 logging configuration 중 어느 것을 사용하고 있는지 확인하도록 안내합니다. :chatgpt-content-reference{index="8"}

여기까지 확인한 뒤에야 JBoss 측 Application category를 차단하거나 deployment logging 분리를 하는 것이 안전합니다.

**처음부터 `use-deployment-logging-config=false`로 바꾸는 것은 권장하지 않습니다.** 다른 WAR/EAR의 로깅에도 영향을 줄 수 있기 때문입니다.

---

# 19. `server.out` 자체의 Rotation도 별도로 확인

`server.log`:

```text
JBoss PeriodicRotatingFileHandler
```

가 관리합니다.

`bk-fo.log`:

```text
Log4j2 RollingFile
```

가 관리합니다.

하지만 `server.out`:

```text
shell redirect
systemd
init script
nohup
```

등이 생성하는 파일이라면 JBoss Logging의 rotation과 별개입니다.

따라서:

```bash
ls -l /proc/<JBOSS_PID>/fd/1
ls -l /proc/<JBOSS_PID>/fd/2
```

로 stdout/stderr가 어디로 연결되는지 확인하고:

```bash
grep -R "server.out" /etc/logrotate.d 2>/dev/null
```

도 확인하는 것이 좋습니다.

`server.out`을 ERROR 중심으로 줄이더라도 **rotation이 전혀 없는 구조는 피하는 것이 좋습니다.**

---

# 20. 최종 권장 구성

제가 현재 시스템이라면 최종적으로 이렇게 구성합니다.

```text
                     ┌──────────────────────┐
                     │   Java Application   │
                     │ Controller/Service   │
                     │ DAO / Business Logic │
                     └──────────┬───────────┘
                                │
                           Log4j2 File
                                │
                                ▼
                       /bk-fo/bk-fo.log
                        INFO/WARN/ERROR
                                │
                       Application 원본


        ┌────────────────────────────────────────┐
        │                JBoss EAP               │
        │ Undertow / XNIO / Tx / DS / JGroups   │
        │ Infinispan / Deployment / WFLY...     │
        └──────────────────┬─────────────────────┘
                           │
                    JBoss Root INFO
                           │
                ┌──────────┴───────────┐
                │                      │
           FILE Handler          CONSOLE Handler
             INFO+                   ERROR+
                │                      │
                ▼                      ▼
          server.log               SYSTEM_OUT
        INFO/WARN/ERROR                 │
                                        ▼
                                    server.out
                                   ERROR/FATAL
                                  + boot/fallback
```

핵심 정책을 한 문장으로 정리하면:

> **`bk-fo.log`는 Application의 사실상 유일한 운영 로그, `server.log`는 JBoss/EAP의 유일한 정상 운영 로그, `server.out`은 JBoss ERROR/FATAL과 기동·표준출력 이상을 위한 비상 로그로 사용한다.**

그리고 현재 설정에서 **가장 먼저 실제 변경할 부분은 Application `ConsoleAppender` 제거와 JBoss `CONSOLE level=ERROR` 두 가지**입니다. 이 두 가지만 먼저 적용하고 세 로그에 테스트 문자열이 어떻게 분배되는지 확인한 뒤, Application 로그가 여전히 `server.log`에 남는 경우에만 JBoss Deployment Logging 경로를 추가로 분리하는 것이 운영 리스크가 가장 낮습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
