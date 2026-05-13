---
layout: single
title: "GitLab 서버 설치 시 Nexus 연계 설정"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: 2026-05-13
last_modified_at: 2026-05-13 17:32:53
---

# GitLab 서버 설치 시 Nexus 연계 설정

정확도: 95%

# GitLab 서버 설치 시 Nexus 연계 설정

## 1. 핵심 결론

GitLab과 Nexus 연계는 보통 **GitLab 서버 설정 파일에 Nexus를 직접 등록하는 방식이 아닙니다.** 실무에서는 **GitLab Runner가 실행하는 CI/CD Job에서 Maven/Gradle이 Nexus를 바라보도록 설정**합니다.

```text
GitLab Push
→ GitLab Pipeline
→ GitLab Runner
→ Maven/Gradle Build
→ Nexus에서 dependency 다운로드
→ 빌드 산출물을 Nexus hosted repository에 배포
```

Nexus Repository는 `hosted`, `proxy`, `group` 저장소 유형을 제공하며, Maven Repository 포맷을 지원합니다. `group` repository는 여러 repository를 하나의 URL로 묶어 빌드 도구에서 단일 URL로 사용하기 좋습니다. ([help.sonatype.com](https://help.sonatype.com/en/repository-types.html?utm_source=chatgpt.com "Repository Types"))

## 2. Nexus 쪽 구성

### 2.1 Maven Repository 구성

권장 Repository는 다음 4개입니다.

1. `maven-central`  
    외부 Maven Central을 캐시하는 `proxy` repository입니다.
    
2. `maven-releases`  
    사내 Release 산출물을 배포하는 `hosted` repository입니다.
    
3. `maven-snapshots`  
    사내 Snapshot 산출물을 배포하는 `hosted` repository입니다.
    
4. `maven-public`  
    GitLab Runner, Jenkins, 개발자 PC가 dependency 조회 시 사용하는 `group` repository입니다.  
    권장 Group 순서는 다음과 같습니다.
    

```text
1. maven-releases
2. maven-snapshots
3. maven-central
```

내부 산출물을 먼저 조회하고, 없을 때 외부 dependency를 조회하도록 하는 구조입니다.

## 3. Nexus 계정/권한 구성

CI/CD에서는 `admin` 계정을 사용하지 않는 것이 좋습니다.  
권장 계정은 다음과 같습니다.

1. `nexus-readonly`  
    dependency 다운로드 전용 계정입니다.  
    `maven-public`에 대한 read 권한만 부여합니다.
    
2. `nexus-deploy`  
    GitLab Runner가 산출물을 배포할 때 사용하는 계정입니다.  
    `maven-releases`, `maven-snapshots`에 대한 read/write 권한을 부여합니다.
    
3. `admin`  
    Nexus 운영 관리 전용 계정입니다.  
    CI/CD Job에서는 사용하지 않습니다.
    

## 4. GitLab CI/CD Variables 설정

GitLab Project 또는 Group에서 다음 위치에 변수를 등록합니다.

```text
Settings
→ CI/CD
→ Variables
```

등록할 변수:

```text
NEXUS_MAVEN_PUBLIC=https://nexus.aaa.com/repository/maven-public/
NEXUS_RELEASES_URL=https://nexus.aaa.com/repository/maven-releases/
NEXUS_SNAPSHOTS_URL=https://nexus.aaa.com/repository/maven-snapshots/
NEXUS_USERNAME=nexus-deploy
NEXUS_PASSWORD=********
```

`NEXUS_USERNAME`, `NEXUS_PASSWORD`는 `Masked`로 설정하고, 운영 배포용이면 `Protected`도 함께 적용하는 것이 좋습니다. GitLab CI/CD Variables는 Project, Group, Instance 단위로 정의할 수 있고, Masked 변수는 Job 로그에서 값 노출을 줄이며, Protected 변수는 보호 브랜치/태그 Pipeline에서만 사용할 수 있습니다. ([GitLab 문서](https://docs.gitlab.com/ci/variables/?utm_source=chatgpt.com "CI/CD variables"))

## 5. Maven `settings.xml` 설정

프로젝트 루트에 CI 전용 설정 파일을 둡니다.  
파일명 예:

```text
ci_settings.xml
```

내용:

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <mirrors>
    <mirror>
      <id>nexus-maven-public</id>
      <mirrorOf>*</mirrorOf>
      <url>${env.NEXUS_MAVEN_PUBLIC}</url>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${env.NEXUS_USERNAME}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>${env.NEXUS_USERNAME}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>
</settings>
```

`settings.xml`은 Maven 실행 환경 설정 파일이며, 인증 정보나 서버 접속 정보처럼 프로젝트 소스에 직접 고정하기 어려운 값을 관리하는 데 사용됩니다. Maven 설정에는 `servers`, `mirrors`, `proxies`, `profiles` 등을 둘 수 있습니다. ([Apache Maven](https://maven.apache.org/xsd/settings-1.2.0.xsd?utm_source=chatgpt.com "https://maven.apache.org/xsd/settings-1.2.0.xsd"))

## 6. `pom.xml` 배포 저장소 설정

### 방식 A: `pom.xml`에 `distributionManagement` 사용

가장 일반적인 방식입니다.

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <name>Nexus Releases</name>
    <url>https://nexus.aaa.com/repository/maven-releases/</url>
  </repository>
  <snapshotRepository>
    <id>nexus-snapshots</id>
    <name>Nexus Snapshots</name>
    <url>https://nexus.aaa.com/repository/maven-snapshots/</url>
  </snapshotRepository>
</distributionManagement>
```

중요한 점은 `pom.xml`의 repository `id`와 `ci_settings.xml`의 `server id`가 반드시 일치해야 한다는 것입니다.

```text
pom.xml id                 settings.xml server id
-------------------------------------------------
nexus-releases       ↔     nexus-releases
nexus-snapshots      ↔     nexus-snapshots
```

Maven Deploy Plugin은 `deploy` 단계에서 빌드 산출물을 원격 repository에 배포하는 역할을 합니다. ([Apache Maven](https://maven.apache.org/plugins/maven-deploy-plugin/usage.html?utm_source=chatgpt.com "Apache Maven Deploy Plugin – Usage"))

### 방식 B: CI에서 배포 대상 URL을 직접 지정

`pom.xml`에 Nexus URL을 넣고 싶지 않다면 GitLab CI에서 `altDeploymentRepository`를 사용할 수 있습니다.  
Release 배포 예:

```bash
mvn -s ci_settings.xml -B clean deploy \
  -DaltDeploymentRepository=nexus-releases::default::${NEXUS_RELEASES_URL}
```

Snapshot 배포 예:

```bash
mvn -s ci_settings.xml -B clean deploy \
  -DaltDeploymentRepository=nexus-snapshots::default::${NEXUS_SNAPSHOTS_URL}
```

운영에서는 **방식 A**가 단순하고 표준적입니다. 여러 환경별로 배포 대상이 자주 바뀌는 경우에는 **방식 B**도 검토할 수 있습니다.

## 7. `.gitlab-ci.yml` 예시

### 7.1 Build + Deploy 기본 예시

```yaml
stages:
  - build
  - deploy
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
cache:
  key: "$CI_PROJECT_NAME"
  paths:
    - .m2/repository
build:
  stage: build
  image: maven:3.9.9-eclipse-temurin-17
  script:
    - mvn -s ci_settings.xml -B clean package
deploy:
  stage: deploy
  image: maven:3.9.9-eclipse-temurin-17
  script:
    - mvn -s ci_settings.xml -B clean deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "dev"'
    - if: '$CI_COMMIT_TAG'
```

핵심 의미:

1. `mvn -s ci_settings.xml`  
    CI용 Maven 설정 파일을 사용합니다.
    
2. `clean package`  
    빌드 산출물을 생성합니다.
    
3. `clean deploy`  
    산출물을 Nexus에 배포합니다.
    
4. `.m2/repository` cache  
    GitLab Runner의 Maven dependency 다운로드 시간을 줄입니다.
    
5. `rules`  
    `dev` 브랜치 또는 tag에서만 배포되도록 제한합니다.
    

## 8. GitLab Runner 서버 점검

중요한 것은 **GitLab 서버가 아니라 Runner가 실행되는 서버 또는 컨테이너에서 Nexus 접근이 가능해야 한다는 점**입니다.  
Runner 서버에서 확인:

```bash
curl -I https://nexus.aaa.com/repository/maven-public/
curl -I https://nexus.aaa.com/repository/maven-releases/
curl -I https://nexus.aaa.com/repository/maven-snapshots/
```

응답 판단:

```text
200  → 접근 정상
401  → 인증 필요
403  → 권한 부족
404  → Repository URL 오류
502  → Reverse Proxy 또는 Nexus 문제
503  → Nexus 서비스 문제 가능
504  → Gateway timeout 또는 네트워크 지연
```

Docker Runner를 사용한다면 **Runner Host에서 curl이 성공하는 것만으로는 부족**합니다. 실제 Maven 빌드가 실행되는 Docker 컨테이너 내부에서도 Nexus 도메인, 포트, 인증서 접근이 가능해야 합니다.

## 9. 사설 인증서 사용 시

Nexus가 사내 인증서 HTTPS를 사용하면 Maven 빌드에서 다음과 같은 오류가 날 수 있습니다.

```text
PKIX path building failed
unable to find valid certification path to requested target
```

확인:

```bash
curl -Iv https://nexus.aaa.com/repository/maven-public/
```

대응 방법:

1. Runner 서버 OS trust store에 사내 CA 등록
    
2. Maven 빌드 Docker 이미지에 사내 CA 포함
    
3. JDK truststore에 사내 CA 등록
    
4. 임시로 HTTP 사용은 가능하지만 운영에서는 HTTPS 권장
    

## 10. Maven 실행 검증

Runner 환경에서 직접 테스트합니다.  
Dependency 조회:

```bash
mvn -s ci_settings.xml -B dependency:resolve
```

빌드:

```bash
mvn -s ci_settings.xml -B clean package
```

배포:

```bash
mvn -s ci_settings.xml -B clean deploy
```

오류별 주요 원인:

```text
401 Unauthorized
→ GitLab 변수 누락, 계정/비밀번호 오류, server id 불일치

403 Forbidden
→ Nexus 계정 권한 부족

404 Not Found
→ Nexus Repository URL 오류

PKIX path building failed
→ 사설 인증서 신뢰 문제

Could not transfer artifact
→ Nexus 접근 불가, Proxy, 방화벽, DNS 문제

Return code is: 401
→ pom.xml distributionManagement id와 settings.xml server id 불일치 가능
```

## 11. 설치 시 최종 체크리스트

1. Nexus에 `maven-public`, `maven-releases`, `maven-snapshots`가 있는가
    
2. `maven-public` group에 내부 hosted와 외부 proxy가 포함되어 있는가
    
3. CI 전용 계정 `nexus-deploy`가 있는가
    
4. `nexus-deploy`에 필요한 read/write 권한이 있는가
    
5. GitLab Variables에 Nexus URL과 계정 정보가 등록되어 있는가
    
6. 비밀번호 변수는 `Masked` 처리했는가
    
7. 운영 배포 변수는 `Protected` 처리했는가
    
8. Runner 서버에서 Nexus URL로 `curl`이 되는가
    
9. Docker Runner라면 컨테이너 내부에서도 Nexus 접근이 되는가
    
10. `ci_settings.xml`의 `server id`와 `pom.xml`의 repository `id`가 일치하는가
    
11. `mvn -s ci_settings.xml clean package`가 성공하는가
    
12. `mvn -s ci_settings.xml clean deploy`가 성공하는가
    
13. Nexus UI에서 artifact가 정상 생성되는가
    

## 12. 권장 운영 구조

```text
조회:
GitLab Runner
→ maven-public
→ maven-releases
→ maven-snapshots
→ maven-central

배포:
GitLab Runner
→ maven-releases 또는 maven-snapshots
```

Release 버전:

```text
1.0.0
1.0.1
2.0.0
```

Snapshot 버전:

```text
1.0.0-SNAPSHOT
2.0.0-SNAPSHOT
```

Maven은 버전에 `-SNAPSHOT`이 붙어 있으면 Snapshot 배포 대상으로 처리하고, 그렇지 않으면 Release 배포 대상으로 처리합니다.

## 결론

GitLab 서버 설치 시 Nexus 연계의 핵심은 **GitLab Runner의 빌드 환경이 Nexus를 Maven 저장소로 사용하도록 구성하는 것**입니다. Nexus에는 `maven-public`, `maven-releases`, `maven-snapshots`를 구성하고, GitLab에는 Nexus URL과 계정 정보를 CI/CD Variables로 등록합니다. 이후 `ci_settings.xml`, `pom.xml`, `.gitlab-ci.yml`을 연결하면 GitLab Pipeline에서 dependency 다운로드와 artifact 배포를 Nexus 중심으로 처리할 수 있습니다.

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
