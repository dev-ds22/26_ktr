- 배포 시 `health check 파일`은 보통 2가지로 나눠서 작성합니다.

| 구분                      | 용도                                             | 권장도   |
| ----------------------- | ---------------------------------------------- | ----- |
| HTTP 기반 health endpoint | 애플리케이션이 실제로 요청을 받을 준비가 되었는지 확인                 | 가장 권장 |
| 파일 기반 health marker     | 배포 스크립트나 Kubernetes `exec probe`가 파일 존재 여부로 확인 | 보조 수단 |
- Spring Boot 기준으로는 Actuator의 `health` 엔드포인트를 기본으로 두고, 필요하면 readiness 상태를 파일로 내보내는 방식이 가장 안정적입니다. Spring Boot는 `/actuator/health`를 제공하고, liveness/readiness 상태를 별도 health group으로 노출할 수 있으며, 필요하면 readiness 상태를 파일로 export하는 방식도 공식 예시로 설명합니다. ([Home](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html "Endpoints :: Spring Boot"))

## 1. 작성 원칙

| 원칙                       | 설명                                          |
| ------------------------ | ------------------------------------------- |
| Liveness와 Readiness를 분리  | 프로세스가 살아 있는지와 트래픽을 받을 준비가 되었는지는 다름          |
| 배포 판정은 Readiness 기준      | 기동 직후 초기화가 끝나기 전에는 트래픽 차단 필요                |
| 외부 연동은 Readiness에 신중히 포함 | DB, Redis, 외부 API 장애가 전체 재기동으로 번지지 않게 설계 필요 |
| Health 응답은 빠르게           | 느린 health check는 오히려 장애 유발 가능               |
- Spring Boot 문서도 liveness는 외부 시스템 체크에 의존하지 않는 것이 좋다고 설명하고, readiness는 트래픽 수용 가능 여부를 나타내며, 느린 health indicator는 경고 대상이라고 안내합니다. ([Home](https://docs.spring.io/spring-boot/reference/features/spring-application.html "SpringApplication :: Spring Boot"))

## 2. 가장 추천하는 구조

```text
[Deploy Script]
  -> 애플리케이션 기동
  -> /readyz 또는 /actuator/health/readiness 반복 호출
  -> HTTP 200 + status=UP 확인
  -> 성공 시 서비스 연결
  -> 실패 시 롤백/중단
```

Spring Boot는 readiness/liveness를 health group으로 제공하고, `management.endpoint.health.probes.add-additional-paths=true` 설정 시 메인 포트에 `/livez`, `/readyz`로 추가 노출할 수 있습니다. 별도 management port만 쓰면 메인 앱이 실제로 비정상이어도 probe가 성공할 수 있어, 메인 포트에 추가 경로를 두는 구성이 실무적으로 더 안전합니다. ([Home](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html "Endpoints :: Spring Boot"))

## 3. Spring Boot 설정 파일 예제

### `build.gradle`

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```

Actuator는 운영용 health endpoint를 제공하는 공식 스타터입니다. ([Home](https://docs.spring.io/spring-boot/reference/actuator/index.html?utm_source=chatgpt.com "Production-ready Features :: Spring Boot"))

### `application.yml`

```yaml
server:
  port: 8080
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
      show-details: never
```

위 설정이면 일반적으로 아래 경로를 사용할 수 있습니다.

| 경로                           | 의미                    |
| ---------------------------- | --------------------- |
| `/actuator/health`           | 전체 health             |
| `/actuator/health/liveness`  | 프로세스 내부 생존 상태         |
| `/actuator/health/readiness` | 트래픽 수용 준비 상태          |
| `/livez`                     | 메인 포트 liveness 추가 경로  |
| `/readyz`                    | 메인 포트 readiness 추가 경로 |

- 이 경로 구성과 추가 path 동작은 Spring Boot Actuator 공식 문서 기준입니다. ([Home](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html "Endpoints :: Spring Boot"))
## 4. 배포용 health check 스크립트 파일 예제

### `health-check.sh`

```bash
#!/bin/bash
APP_URL="${1:-http://127.0.0.1:8080/readyz}"
MAX_RETRY=30
SLEEP_SEC=2
COUNT=1

echo "[INFO] health check start: ${APP_URL}"

while [ $COUNT -le $MAX_RETRY ]
do
  RESPONSE=$(curl -s -o /tmp/health_response.json -w "%{http_code}" "${APP_URL}")
  BODY=$(cat /tmp/health_response.json 2>/dev/null)

  if [ "${RESPONSE}" = "200" ] && echo "${BODY}" | grep -q '"status":"UP"'; then
    echo "[INFO] health check success"
    exit 0
  fi

  echo "[WARN] attempt=${COUNT}, http=${RESPONSE}, body=${BODY}"
  COUNT=$((COUNT+1))
  sleep ${SLEEP_SEC}
done

echo "[ERROR] health check failed"
exit 1
```

### 실행 예

```bash
chmod +x health-check.sh
./health-check.sh http://127.0.0.1:8080/readyz
```

## 5. 배포 스크립트에 연결하는 예

### `deploy.sh`

```bash
#!/bin/bash
APP_NAME="commerce-api"
JAR_PATH="/app/commerce/commerce-api.jar"
LOG_PATH="/app/commerce/logs/${APP_NAME}.log"

echo "[INFO] stop old process"
pkill -f "${APP_NAME}.jar" || true

echo "[INFO] start new process"
nohup java -jar "${JAR_PATH}" > "${LOG_PATH}" 2>&1 &

echo "[INFO] run health check"
./health-check.sh http://127.0.0.1:8080/readyz

if [ $? -ne 0 ]; then
  echo "[ERROR] deploy failed"
  exit 1
fi

echo "[INFO] deploy success"
```

## 6. 커스텀 health check 작성 예제

배포 판단을 더 정확히 하려면 DB, Redis, MQ, 외부 API 중 꼭 필요한 항목만 readiness에 반영하는 커스텀 `HealthIndicator`를 추가할 수 있습니다. Spring Boot는 `HealthIndicator` 구현체를 빈으로 등록해 custom health 정보를 추가하는 방식을 공식 지원합니다. ([Home](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html "Endpoints :: Spring Boot"))

### `CustomApiHealthIndicator.java`

```java
package com.example.health;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

@Component
public class CustomApiHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            boolean ok = checkDependency();
            if (ok) {
                return Health.up()
                        .withDetail("target", "custom-api")
                        .build();
            }
            return Health.down()
                    .withDetail("target", "custom-api")
                    .withDetail("reason", "dependency check failed")
                    .build();
        } catch (Exception e) {
            return Health.down(e)
                    .withDetail("target", "custom-api")
                    .build();
        }
    }

    private boolean checkDependency() {
        return true;
    }
}
```

### readiness 그룹에 포함 예

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: readinessState,customApi
```

Spring Boot 문서상 health group은 include/exclude 설정이 가능하고, readiness 그룹에 custom indicator를 포함시킬 수 있습니다. 다만 외부 시스템을 readiness에 넣을지는 서비스 특성에 따라 신중히 판단해야 합니다. ([Home](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html "Endpoints :: Spring Boot"))

## 7. 파일 기반 health check 예제

플랫폼이 HTTP 대신 파일 존재 여부를 검사해야 하면, readiness 상태를 파일로 export하는 방식이 가능합니다. Spring Boot는 `AvailabilityChangeEvent<ReadinessState>`를 받아 readiness 상태에 따라 파일을 생성/삭제하는 예시를 제공합니다. ([Home](https://docs.spring.io/spring-boot/reference/features/spring-application.html "SpringApplication :: Spring Boot"))

### `ReadinessFileExporter.java`

```java
package com.example.health;

import org.springframework.boot.availability.AvailabilityChangeEvent;
import org.springframework.boot.availability.ReadinessState;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

@Component
public class ReadinessFileExporter {

    private static final Path HEALTH_FILE = Path.of("/tmp/healthy");

    @EventListener
    public void onStateChange(AvailabilityChangeEvent<ReadinessState> event) throws IOException {
        if (event.getState() == ReadinessState.ACCEPTING_TRAFFIC) {
            if (Files.notExists(HEALTH_FILE)) {
                Files.createFile(HEALTH_FILE);
            }
        } else if (event.getState() == ReadinessState.REFUSING_TRAFFIC) {
            Files.deleteIfExists(HEALTH_FILE);
        }
    }
}
```

### 파일 확인 스크립트

```bash
#!/bin/bash
if [ -f /tmp/healthy ]; then
  echo "UP"
  exit 0
else
  echo "DOWN"
  exit 1
fi
```

## 8. 실무 권장안

|상황|권장 방식|
|---|---|
|일반 Spring Boot 서버 배포|`/readyz`를 배포 스크립트에서 polling|
|LB/Nginx/Ingress 연동|`/livez`, `/readyz` 분리|
|Kubernetes HTTP Probe|`/actuator/health/liveness`, `/actuator/health/readiness` 또는 `/livez`, `/readyz`|
|Kubernetes Exec Probe|`/tmp/healthy` 파일 방식|
|복잡한 연동 시스템|커스텀 `HealthIndicator` 추가|

## 9. 주의사항

| 주의사항                       | 이유                         |
| -------------------------- | -------------------------- |
| 정적 파일만 두고 끝내지 말 것          | 웹서버는 살아 있어도 앱 초기화 실패일 수 있음 |
| liveness에 DB/API 체크 넣지 말 것 | 외부 장애가 전체 재기동 연쇄로 번질 수 있음  |
| readiness는 너무 무겁게 만들지 말 것  | 응답 지연으로 배포 실패나 오탐 가능       |
| 별도 management port만 믿지 말 것 | 메인 애플리케이션 포트 이상을 놓칠 수 있음   |
- 이 주의사항들은 Spring Boot의 liveness/readiness 분리 원칙과 메인 포트 추가 path 권장 내용에 부합합니다. ([Home](https://docs.spring.io/spring-boot/reference/features/spring-application.html "SpringApplication :: Spring Boot")

## 10. 바로 쓸 최소 구성

가장 간단하게는 아래 3개만 있으면 됩니다.

### `application.yml`

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health
  endpoint:
    health:
      probes:
        enabled: true
        add-additional-paths: true
```

### `health-check.sh`

```bash
#!/bin/bash
curl -fsS http://127.0.0.1:8080/readyz | grep -q '"status":"UP"'
```

### 배포 후 실행

```bash
./health-check.sh
```

원하시면 다음 답변에서 `Jenkins`, `Nginx`, `Kubernetes`, `Systemd` 중 하나를 기준으로 배포용 health check 파일을 환경별로 바로 맞춰드리겠습니다.