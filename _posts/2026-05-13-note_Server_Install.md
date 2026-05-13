---
layout: single
title: "Nexus + Maven + GitLab 16.5.0 + Jenkins 2.414.3 서버 구축 순서"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: 2026-05-13
last_modified_at: 2026-05-13 17:29:08
---

# Nexus + Maven + GitLab 16.5.0 + Jenkins 2.414.3 서버 구축 순서

정확도: 95%

# Nexus + Maven + GitLab 16.5.0 + Jenkins 2.414.3 서버 구축 순서

## 1. 전체 개략

목표 구조는 다음과 같습니다.

```text
개발자
→ GitLab 16.5.0에 소스 Push
→ GitLab Webhook 또는 Jenkins GitLab Integration으로 Jenkins 빌드 트리거
→ Jenkins 2.414.3에서 Git Checkout
→ Maven 빌드 수행
→ Maven dependency는 Nexus Repository에서 다운로드
→ 빌드 산출물은 Nexus Repository에 배포
```

GitLab 공식 문서 기준 Jenkins Integration은 GitLab에 Push가 발생했을 때 Jenkins Build를 트리거할 수 있습니다. 단, 이 통합은 Jenkins에서 GitLab CI/CD Pipeline을 직접 트리거하는 용도가 아니라 GitLab 변경 사항으로 Jenkins Build를 실행하는 용도입니다. ([GitLab 문서](https://docs.gitlab.com/integration/jenkins/?utm_source=chatgpt.com "Jenkins"))

## 2. 권장 구축 순서

|순서|대상|작업|이유|
|--:|---|---|---|
|1|공통 인프라|OS, DNS, 방화벽, 계정, 디스크, NTP, 백업 경로 준비|이후 모든 서버의 공통 기반|
|2|Nexus|Repository, 계정, 권한, Blob Store, Backup 설정|Maven dependency 조회와 산출물 배포의 중심|
|3|GitLab|소스 저장소, 그룹/프로젝트, 권한, Webhook 준비|소스 관리 기준점|
|4|Jenkins|JDK, Maven, Git, 플러그인, Credential, Job/Pipeline 설정|자동 빌드 실행 주체|
|5|Maven 연계|`settings.xml`, `pom.xml`, 배포 Repository 설정|Jenkins 빌드가 Nexus를 사용하게 함|
|6|GitLab-Jenkins 연계|Webhook, Jenkins Token, GitLab API Token 설정|Push/Merge 시 자동 빌드|
|7|검증|Commit → Jenkins Build → Maven Build → Nexus Deploy 확인|전체 흐름 기동 검증|
|8|운영 설정|로그, 백업, 보안, Cleanup, 모니터링, 업그레이드 정책|장기 운영 안정화|

## 3. 버전 관련 주의사항

| 구분      |      지정 버전 | 주의사항                                                                  |
| ------- | ---------: | --------------------------------------------------------------------- |
| GitLab  |     16.5.0 | 2023년 릴리즈 계열이므로 신규 구축 시 최신 패치/지원 버전 검토 필요                             |
| Jenkins |    2.414.3 | 2023년 LTS 계열이며 보안 공지가 포함된 릴리즈                                         |
| Java    | Jenkins 기준 | 2.414.3은 Java 11 이상 환경에서 운용 가능하나, Docker 이미지 태그에 따라 Java 17이 기본일 수 있음 |
| Maven   |     3.x 권장 | HTTP 저장소 차단, TLS, mirror 정책 등 Maven 버전에 따라 동작 차이 가능                   |
| Nexus   |     3.x 기준 | Nexus Repository 3 기준으로 hosted/proxy/group repository 구성              |
- GitLab 16.5는 2023년 10월 릴리즈 문서가 존재하며, Jenkins 2.414.3도 2023년 10월 LTS 릴리즈입니다. Jenkins 2.414.3 changelog에는 중요한 보안 수정이 포함되어 있으며, 해당 시점 Docker 이미지에서는 Java 17 기본 태그와 Java 11 태그 구분이 존재했습니다. ([GitLab 문서](https://docs.gitlab.com/releases/16/gitlab-16-5-released/?utm_source=chatgpt.com "GitLab 16.5 release notes"))

- 현재 Jenkins 공식 Java 지원 정책은 최신 Jenkins 라인에서 Java 17/21 테스트를 안내하므로, 신규 구축이나 업그레이드 계획이 있다면 Jenkins Core와 Plugin 호환성 기준으로 Java 버전을 다시 검토해야 합니다. 

## 4. 1단계: 공통 인프라 준비

### 4.1 서버 구성 예시

| 서버               | 역할             | 권장 구성                                           |
| ---------------- | -------------- | ----------------------------------------------- |
| GitLab 서버        | 소스 저장소         | CPU/Memory/Disk 충분히 확보, `/var/opt/gitlab` 분리 권장 |
| Jenkins 서버       | 자동 빌드          | JDK, Maven, Git 설치, Workspace 디스크 확보            |
| Nexus 서버         | Repository 관리  | Blob Store 디스크 별도 확보 권장                         |
| Reverse Proxy 서버 | HTTPS, 도메인 라우팅 | Nginx 또는 L4/L7 LB                               |

- 운영 규모가 작으면 한 서버에 같이 둘 수는 있지만, 실무에서는 GitLab, Jenkins, Nexus를 분리하는 것이 장애 영향도와 디스크 관리를 분리할 수 있어 유리합니다.

### 4.2 공통 준비 항목

|항목|확인 내용|주의사항|
|---|---|---|
|DNS|`gitlab.aaa.com`, `jenkins.aaa.com`, `nexus.aaa.com`|내부망/외부망 해석 일관성 확인|
|방화벽|80, 443, 22 또는 Git SSH 포트, Jenkins/Nexus 내부 포트|Runner/Jenkins → GitLab/Nexus 접근 허용|
|NTP|시간 동기화|Webhook, Token, 인증서 문제 방지|
|디스크|GitLab repo, Jenkins workspace, Nexus blob|Nexus와 GitLab은 디스크 증가가 빠름|
|계정|서비스 계정 분리|root 직접 실행 지양|
|인증서|HTTPS 적용|Maven/Jenkins에서 사설 CA 신뢰 필요 가능|
|백업|GitLab/Nexus/Jenkins 각각 백업 경로|Blob Store와 설정 파일 포함 필요|

## 5. 2단계: Nexus Repository 구축

### 5.1 Repository 구성

Nexus Repository는 `hosted`, `proxy`, `group` Repository 유형을 지원합니다. `hosted`는 내부 산출물을 저장하고, `proxy`는 외부 저장소를 프록시/캐시하며, `group`은 여러 Repository를 하나의 URL로 묶어 제공합니다. ([Sonatype Help](https://help.sonatype.com/en/repository-types.html?utm_source=chatgpt.com "Repository Types"))

| Repository        | 유형     | 용도                  | 예시                   |
| ----------------- | ------ | ------------------- | -------------------- |
| `maven-central`   | proxy  | 외부 Maven Central 캐시 | 외부 dependency 다운로드   |
| `maven-releases`  | hosted | Release 산출물 저장      | `1.0.0`, `1.0.1`     |
| `maven-snapshots` | hosted | Snapshot 산출물 저장     | `1.0.0-SNAPSHOT`     |
| `maven-public`    | group  | 빌드에서 사용하는 통합 조회 URL | Jenkins/개발자 PC 공통 사용 |
|                   |        |                     |                      |
- Nexus Maven Repository 문서에서는 기본 설치에 Maven Central proxy repository가 포함될 수 있고, CI 서버와 개발자의 중복 다운로드를 줄이기 위해 원격 저장소를 proxy repository로 구성하는 방식을 설명합니다. ([Sonatype Help](https://help.sonatype.com/en/maven-repositories.html?utm_source=chatgpt.com "Maven Repositories"))

### 5.2 `maven-public` Group 순서

|순서|Repository|이유|
|--:|---|---|
|1|`maven-releases`|내부 Release 라이브러리 우선 조회|
|2|`maven-snapshots`|내부 Snapshot 라이브러리 조회|
|3|`maven-central`|내부에 없을 경우 외부 dependency 조회|

### 5.3 Nexus 계정/권한

|계정|권한|사용처|
|---|---|---|
|`nexus-readonly`|`maven-public` read|개발자 PC, 단순 빌드 조회|
|`nexus-deploy`|`maven-releases`, `maven-snapshots` read/write|Jenkins 배포용|
|`admin`|관리자 권한|운영자 전용, CI/CD 사용 금지|
|주의사항:|||
|항목|주의 내용||
|---|---||
|Admin 계정 사용 금지|Jenkins에 `admin` 계정을 저장하지 말 것||
|Release 재배포 정책|운영에서는 Release Repository의 redeploy 허용 여부를 엄격히 결정||
|Snapshot 정리|Snapshot은 증가가 빠르므로 Cleanup Policy 필요||
|Blob Store|OS에서 직접 파일 삭제 금지, Nexus Cleanup/Compact Task 사용||
|백업|DB/설정 백업과 Blob Store 백업을 함께 관리||

## 6. 3단계: GitLab 16.5.0 구축

### 6.1 GitLab 기본 구성

|항목|설정 내용|주의사항|
|---|---|---|
|외부 URL|`https://gitlab.aaa.com`|인증서와 일치 필요|
|SSH URL|기본 22 또는 별도 포트|방화벽, 개발자 PC 접근 확인|
|그룹|서비스/시스템 단위 그룹 생성|권한 상속 구조 설계|
|프로젝트|소스 Repository 생성|브랜치 전략과 연계|
|권한|Maintainer/Developer 분리|운영 브랜치 권한 제한|
|Protected Branch|`main`, `master`, `prd`, `release/*`|강제 Push 제한|
|Webhook|Jenkins 트리거 URL 등록|Secret Token 사용 권장|

### 6.2 브랜치 전략 예시

|브랜치|용도|Jenkins 동작|
|---|---|---|
|`feature/*`|개발자 기능 개발|선택적으로 빌드|
|`dev`|개발 통합|자동 빌드/개발 배포|
|`stg` 또는 `qa`|검증|검증 환경 배포|
|`main` 또는 `prd`|운영 반영 기준|승인 후 운영 배포|

### 6.3 GitLab Webhook 설정 위치

```text
Project
→ Settings
→ Webhooks
```

주요 설정:

| 항목               | 설정                                       |
| ---------------- | ---------------------------------------- |
| URL              | Jenkins Job 또는 GitLab Plugin Webhook URL |
| Secret Token     | Jenkins와 동일한 Token                       |
| Trigger          | Push events, Merge request events 등      |
| SSL verification | 사설 인증서면 신뢰 설정 확인                         |

GitLab 공식 Webhook 문서는 Project 또는 Group Webhook으로 외부 시스템과 연계할 수 있고, Push/Merge Request 등 이벤트를 기준으로 호출할 수 있음을 설명합니다. ([GitLab 문서](https://docs.gitlab.com/user/project/integrations/webhooks/?utm_source=chatgpt.com "Webhooks"))
## 7. 4단계: Jenkins 2.414.3 구축

### 7.1 Jenkins 기본 설치 후 설정

| 항목            | 설정 위치                  | 설정 내용                                        |
| ------------- | ---------------------- | -------------------------------------------- |
| JDK           | Manage Jenkins → Tools | Java 11 또는 17 경로                             |
| Maven         | Manage Jenkins → Tools | Maven 3.x 설치 경로                              |
| Git           | Manage Jenkins → Tools | Git CLI 경로                                   |
| Credential    | Manage Credentials     | GitLab Token, Git SSH Key, Nexus 계정          |
| Global Config | System 설정              | GitLab URL, Jenkins URL, Proxy 등             |
| Plugin        | Plugin Manager         | GitLab, Pipeline, Git, Credentials, Maven 관련 |

Jenkins 2.414.3은 보안 수정이 포함된 LTS 릴리즈이며, Jenkins 공식 보안 공지에서는 해당 릴리즈 라인과 관련된 보안 수정 내역을 확인할 수 있습니다. 운영 서버는 설치 후 Plugin까지 포함해 보안 공지를 주기적으로 확인해야 합니다. ([Jenkins](https://www.jenkins.io/security/advisory/2023-10-18/?utm_source=chatgpt.com "Jenkins Security Advisory 2023-10-18"))
### 7.2 Jenkins 플러그인

| 플러그인                                | 용도                          | 주의사항         |
| ----------------------------------- | --------------------------- | ------------ |
| Git Plugin                          | GitLab Repository Checkout  | Git CLI 필요   |
| Pipeline                            | Jenkinsfile 기반 빌드           | 권장 방식        |
| Credentials Plugin                  | 비밀번호/Token/SSH Key 관리       | 평문 저장 금지     |
| GitLab Plugin                       | GitLab Webhook 연계, 빌드 상태 전달 | 버전 호환성 확인 필요 |
| Maven Integration 또는 Pipeline Maven | Maven 빌드 편의                 | 필수는 아님       |
| Workspace Cleanup                   | 빌드 후 workspace 정리           | 디스크 관리       |

Jenkins GitLab Plugin은 GitLab commit 또는 Merge Request 이벤트로 Jenkins Build를 트리거하고 Build Status를 GitLab에 전달할 수 있습니다. 다만 해당 플러그인은 GitLab Inc. 또는 CloudBees가 공식 지원하는 플러그인은 아니므로, Jenkins 2.414.3과 설치할 플러그인 버전의 호환성 검증이 필요합니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/?utm_source=chatgpt.com "GitLab | Jenkins plugin"))
## 8. 5단계: Maven과 Nexus 연계

### 8.1 Jenkins 서버의 Maven `settings.xml`

Jenkins 서버에 CI용 Maven 설정 파일을 둡니다.  
예시 경로:

```text
/var/lib/jenkins/.m2/settings.xml
```

또는 Jenkins Credential로 관리 후 Pipeline에서 파일로 생성할 수 있습니다.

```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <mirrors>
    <mirror>
      <id>nexus-maven-public</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.aaa.com/repository/maven-public/</url>
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

Maven `settings.xml`은 Maven 실행 환경을 구성하는 파일이며, `servers`, `mirrors`, `proxies`, `profiles` 같은 설정을 담을 수 있습니다. Repository Manager를 사용하는 조직에서는 mirror 설정으로 Maven 요청을 내부 Repository Manager로 모으는 구성이 일반적입니다. ([docs.oracle.com](https://docs.oracle.com/en/middleware/fusion-middleware/12.2.1.3/maven/installing-and-configuring-maven-build-automation-and-dependency-management.html?source=%3Aso%3Afb%3Aor%3Aawr%3Aosec%3A%2C%3Aso%3Afb%3Aor%3Aawr%3Aosec%3A%2C%3Aso%3Afb%3Aor%3Aawr%3Aosec%3A%2C%3Aso%3Afb%3Aor%3Aawr%3Aosec%3A&utm_source=chatgpt.com "Installing and Configuring Maven for Build Automation and ..."))

### 8.2 프로젝트 `pom.xml`

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

`pom.xml`의 `repository.id`, `snapshotRepository.id`는 `settings.xml`의 `<server><id>`와 반드시 일치해야 합니다. Maven Deploy Plugin은 `deploy` phase에서 원격 Repository로 artifact를 배포하는 역할을 합니다. ([maven.apache.org](https://maven.apache.org/xsd/settings-1.2.0.xsd?utm_source=chatgpt.com "https://maven.apache.org/xsd/settings-1.2.0.xsd"))  
정확한 매핑:

|구분|`pom.xml`|`settings.xml`|
|---|---|---|
|Release 배포|`nexus-releases`|`nexus-releases`|
|Snapshot 배포|`nexus-snapshots`|`nexus-snapshots`|

## 9. 6단계: Jenkins Pipeline 구성

### 9.1 Jenkins Credential 구성

|Credential|종류|용도|
|---|---|---|
|`gitlab-ssh-key`|SSH Username with private key|GitLab 소스 Checkout|
|`gitlab-api-token`|Secret text|GitLab Build Status 전달 또는 API 연계|
|`nexus-deploy-user`|Username/Password|Nexus Deploy|
|`maven-settings`|Secret file 또는 Config File|Maven settings.xml 관리|

### 9.2 Jenkinsfile 예시

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK11'
        maven 'Maven3'
    }

    environment {
        NEXUS_USERNAME = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'dev',
                    credentialsId: 'gitlab-ssh-key',
                    url: 'git@gitlab.aaa.com:group/project.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -s /var/lib/jenkins/.m2/settings.xml -B clean package'
            }
        }

        stage('Deploy Artifact to Nexus') {
            when {
                branch 'dev'
            }
            steps {
                sh 'mvn -s /var/lib/jenkins/.m2/settings.xml -B clean deploy'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
```

주의:

|항목|주의사항|
|---|---|
|Credential ID|Jenkinsfile의 ID와 Jenkins Credential ID가 일치해야 함|
|JDK 이름|`tools { jdk 'JDK11' }`의 이름은 Jenkins Tool 설정명과 일치해야 함|
|Maven 이름|`tools { maven 'Maven3' }`의 이름도 Jenkins Tool 설정명과 일치해야 함|
|Nexus 비밀번호|Jenkinsfile에 평문 작성 금지|
|Workspace|빌드 산출물과 dependency cache로 디스크 증가 가능|
|Branch 조건|운영 배포는 `dev`가 아니라 승인된 브랜치/태그로 제한 권장|

## 10. 7단계: GitLab → Jenkins 자동 빌드 연계

### 10.1 방식 A: GitLab Jenkins Integration 사용

흐름:

```text
GitLab Project
→ Settings
→ Integrations
→ Jenkins
→ Jenkins URL, Project name, Token 설정
```

장점:

| 항목            | 내용                              |
| ------------- | ------------------------------- |
| GitLab UI 통합  | GitLab에서 Jenkins 결과 확인 가능       |
| Push 기반 자동 빌드 | GitLab 변경 사항으로 Jenkins Build 실행 |
| 운영 익숙도        | 기존 Jenkins 중심 조직에 적합            |

- GitLab 공식 문서는 Jenkins Integration을 GitLab에서 Jenkins Build를 트리거하는 방식으로 설명합니다. Jenkins 중심 CI를 유지하면서 향후 GitLab CI/CD로 이전할 수 있는 중간 단계로도 사용할 수 있습니다. ([GitLab 문서](https://docs.gitlab.com/integration/jenkins/?utm_source=chatgpt.com "Jenkins"))
### 10.2 방식 B: GitLab Webhook 직접 사용

흐름:

```text
GitLab Webhook
→ Jenkins Job Webhook URL
→ Jenkins Token 검증
→ Pipeline 실행
```

점검 항목:

|항목|확인|
|---|---|
|Webhook URL|Jenkins에서 외부 접근 가능한 URL인지|
|Token|GitLab Secret Token과 Jenkins Token 일치|
|SSL|GitLab이 Jenkins 인증서를 신뢰하는지|
|방화벽|GitLab 서버 → Jenkins 서버 접근 허용|
|Job 권한|익명 Trigger 차단, Token 기반 Trigger 권장|

## 11. 8단계: 전체 기동 검증 절차

### 11.1 Nexus 단독 검증

```bash
curl -I https://nexus.aaa.com/repository/maven-public/
curl -I https://nexus.aaa.com/repository/maven-releases/
curl -I https://nexus.aaa.com/repository/maven-snapshots/
```

판단:

|응답|의미|
|---|---|
|200|접근 가능|
|401|인증 필요, 정책상 정상일 수 있음|
|403|권한 부족|
|404|Repository URL 오류|
|502/503/504|Nexus, Reverse Proxy, 네트워크 문제|

### 11.2 Jenkins 서버에서 Maven-Nexus 검증

```bash
mvn -s /var/lib/jenkins/.m2/settings.xml -B dependency:resolve
```

배포 검증:

```bash
mvn -s /var/lib/jenkins/.m2/settings.xml -B clean deploy
```

### 11.3 GitLab-Jenkins Webhook 검증

```text
1. GitLab 프로젝트에 테스트 commit push
2. GitLab Webhook Recent Deliveries 확인
3. Jenkins Job 자동 실행 확인
4. Jenkins Console Output 확인
5. Maven dependency가 Nexus에서 내려오는지 확인
6. Nexus UI에서 artifact 배포 결과 확인
```

### 11.4 Jenkins Console에서 확인할 로그

```text
Started by GitLab push by ...
Cloning repository ...
Using credentials ...
Downloaded from nexus-maven-public ...
Uploading to nexus-releases ...
BUILD SUCCESS
```

## 12. 주요 장애 원인과 조치

|증상|주요 원인|확인/조치|
|---|---|---|
|GitLab Push 후 Jenkins가 안 뜸|Webhook URL 오류, Token 불일치, 방화벽|GitLab Webhook Recent Deliveries 확인|
|Jenkins Checkout 실패|GitLab Credential 오류, SSH Key 미등록|Jenkins Credential, GitLab Deploy Key 확인|
|Maven dependency 실패|Nexus mirror 설정 오류, Repository Group 오류|`settings.xml`, `maven-public` 확인|
|401 Unauthorized|Nexus 계정/비밀번호 오류 또는 server id 불일치|Jenkins Credential, `settings.xml`, `pom.xml` ID 확인|
|403 Forbidden|Nexus 권한 부족|`nexus-deploy` 권한 확인|
|404 Not Found|Nexus Repository URL 오류|URL 경로, Repository 이름 확인|
|PKIX path building failed|사설 인증서 신뢰 문제|JDK truststore에 사내 CA 등록|
|Deploy가 Snapshot으로 감|버전이 `-SNAPSHOT`|`pom.xml` version 확인|
|Release 재배포 실패|Release Repository redeploy 금지|버전 증가 후 재배포|
|Jenkins 디스크 부족|workspace, `.m2`, build archive 증가|Workspace Cleanup, 보관 정책 적용|

## 13. 운영 시 주의사항

|영역|주의사항|
|---|---|
|보안|GitLab/Jenkins/Nexus 관리자 계정 분리|
|Credential|토큰/비밀번호 Jenkinsfile, pom.xml에 평문 저장 금지|
|권한|Jenkins 배포 계정은 필요한 Repository에만 write 권한|
|백업|GitLab repo, Jenkins home, Nexus DB/Blob Store 모두 백업|
|로그|GitLab, Jenkins, Nexus, Reverse Proxy 로그 위치 표준화|
|디스크|Nexus Blob, Jenkins workspace, GitLab repository 증가 모니터링|
|업그레이드|GitLab/Jenkins/Nexus는 보안 공지 기준으로 정기 패치|
|플러그인|Jenkins Plugin은 Core 버전 호환성 확인 후 설치|
|인증서|사설 CA 사용 시 GitLab/Jenkins/Maven/JDK truststore 반영|

## 14. 개선 여지와 최신 대안

| 현재 방식                   | 개선 방향                                       | 설명                                               |
| ----------------------- | ------------------------------------------- | ------------------------------------------------ |
| Jenkins 중심 빌드           | GitLab CI/CD 검토                             | GitLab Runner로 빌드/배포를 통합하면 Webhook/Jenkins 의존 감소 |
| Jenkins 단일 서버 빌드        | Agent 분리                                    | 빌드 부하를 Agent로 분산                                 |
| Jenkins 서버에 Maven 직접 설치 | Containerized Build                         | Maven/JDK 버전을 Docker 이미지로 고정                     |
| 평문 settings.xml 관리      | Jenkins Config File Provider 또는 Secret File | Credential 관리 강화                                 |
| Nexus 수동 정리             | Cleanup Policy + Compact Task               | 디스크 증가 자동 관리                                     |
| Release 재배포 허용          | 불변 Release 정책                               | 운영 artifact 추적성 강화                               |
| 수동 배포                   | Tag 기반 배포 승인                                | `v1.0.0` 태그 생성 시 Release Deploy                  |

- 신규 구축이라면 Jenkins 2.414.3과 GitLab 16.5.0을 그대로 고정하기보다, 사내 호환성 검증 후 보안 패치가 반영된 최신 지원 버전 사용을 검토하는 것이 좋습니다. 특히 Jenkins는 Core와 Plugin 호환성이 중요하므로 Jenkins만 업그레이드하거나 Plugin만 최신화하는 방식은 장애를 유발할 수 있습니다. Jenkins의 보안 공지는 지속적으로 발행되므로 운영 중인 Core/Plugin 조합을 기준으로 정기 점검해야 합니다. ([Jenkins](https://www.jenkins.io/security/advisories/?utm_source=chatgpt.com "Security Advisories"))
## 15. 최종 구축 절차 요약

|순서|작업|완료 기준|
|--:|---|---|
|1|OS/DNS/방화벽/인증서/디스크 준비|각 서버 접근 가능|
|2|Nexus 설치|Nexus UI 접속 가능|
|3|Nexus Maven Repository 생성|`maven-public`, `maven-releases`, `maven-snapshots` 구성|
|4|Nexus CI 계정 생성|`nexus-deploy` 권한 확인|
|5|GitLab 설치|Project 생성 및 Git Push 가능|
|6|Jenkins 설치|JDK/Maven/Git Tool 설정 완료|
|7|Jenkins Credential 등록|GitLab/Nexus 인증 정보 등록|
|8|Maven `settings.xml` 구성|Nexus mirror 및 server 설정 완료|
|9|프로젝트 `pom.xml` 구성|`distributionManagement` 설정 완료|
|10|Jenkins Pipeline 생성|`mvn clean package` 성공|
|11|Nexus Deploy 검증|`mvn clean deploy` 성공, Nexus에 artifact 생성|
|12|GitLab Webhook 연결|Push 시 Jenkins 자동 빌드|
|13|운영 정책 적용|백업, Cleanup, 보안, 로그, 모니터링 설정|

## 결론

이 서버 구성의 핵심은 **Nexus를 Maven Repository 표준 진입점으로 먼저 안정화한 뒤, GitLab은 소스 기준점으로, Jenkins는 자동 빌드 실행 주체로 연결하는 것**입니다. 구축 순서는 `공통 인프라 → Nexus → GitLab → Jenkins → Maven 설정 → Webhook 연계 → 전체 검증`이 가장 안전합니다. 운영 안정성을 높이려면 Jenkins와 Nexus Credential 분리, GitLab Protected Branch, Nexus Release/Snapshot 정책, Jenkins Plugin 호환성, 백업/디스크/Cleanup 정책을 반드시 함께 설계해야 합니다.

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
