---
layout: single
title: "[GIT] Git Submodule 가이드"
excerpt: "Gradle 멀티 프로젝트, Git Sub Module"

categories:
  - tech
tags:
  - [git, gradle]

toc: false
toc_sticky: true

date: 2023-01-05
last_modified_at: 2023-01-05
---
# Gradle 멀티 프로젝트 Git Submodule 하기

## 1. STS에서 Gradle 멀티프로젝트 생성

### 1-1. Gradle 멀티프로젝트 생성

```bash
buildscript {
    apply plugin: 'eclipse'
    ext {
        springBootVersion = '2.5.12'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}"
        classpath "io.spring.gradle:dependency-management-plugin:1.0.11.RELEASE"
    }
}

allprojects {

}

subprojects {
   repositories {
        mavenCentral()
   }

    apply plugin: 'java'
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'

    sourceCompatibility = 11

    dependencies {

        implementation 'org.projectlombok:lombok:1.18.16'
        implementation 'org.bgee.log4jdbc-log4j2:log4jdbc-log4j2-jdbc4.1:1.16'
        implementation 'com.integralblue:log4jdbc-spring-boot-starter:1.0.2'
        annotationProcessor 'org.projectlombok:lombok:1.18.16'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
        implementation 'org.springframework.boot:spring-boot-starter-web'

        //lucy-xss
        implementation 'com.navercorp.lucy:lucy-xss-servlet:2.0.1'
        implementation ('org.apache.commons:commons-lang3:3.12.0')
        implementation 'org.apache.commons:commons-text:1.9'

        implementation 'org.springframework.boot:spring-boot-starter-actuator'
        implementation 'commons-io:commons-io:2.11.0'

        implementation ('com.googlecode.json-simple:json-simple:1.1')
        implementation 'com.sun.mail:javax.mail:1.6.2'
        
        implementation 'org.springframework.kafka:spring-kafka'  			// kafka 추가
    	testImplementation 'org.springframework.kafka:spring-kafka-test'  	// kafka 추가
    }
}

project(':ds-bo'){
    dependencies {
        //swagger
        implementation("io.springfox:springfox-boot-starter:3.0.0")
        implementation("io.springfox:springfox-swagger-ui:3.0.0")
    }
}

project(':ds-fo'){
    dependencies {
        //swagger
        implementation("io.springfox:springfox-boot-starter:3.0.0")
        implementation("io.springfox:springfox-swagger-ui:3.0.0")
    }
}

project(':ds-mo'){
    dependencies {
        //swagger
        implementation("io.springfox:springfox-boot-starter:3.0.0")
        implementation("io.springfox:springfox-swagger-ui:3.0.0")
    }
}

project(':ds-batch'){
    dependencies {
        //swagger
        implementation("io.springfox:springfox-boot-starter:3.0.0")
        implementation("io.springfox:springfox-swagger-ui:3.0.0")
    }
}
```

```bash
  rootProject.name = 'ds-backend'
  include 'ds-common'
  include 'ds-bo'
  include 'ds-fo'
  include 'ds-mo'
  include 'ds-batch'
```

## 2. 멀티프로젝트 Git SubModule로 등록

### 2-1. GItlab 에 레포지토리 생성

- 최상단 프로젝트인 Main(ds-backend.git) 및 SubModle 레포지토리를 모두 생성
- 생성시는 Blank 프로젝트로 생성

### 2-2. 최상단 프로젝트(ds-backend.git)에 접속

- Git Config 유저정보 설정
- Git Remote 설정

```bash
git init
git config user.name "user"
git config user.email user@company.co.kr
git config http.sslVerify false

git remote add origin https://gitlab.ds.com/pilot/ds-backend.git
```

### 2-3. 최상단 프로젝트(ds-backend.git)에서 SubModule 설정

- git submodule add 를 이용 SubModule 설정

```bash
git submodule add https://gitlab.ds.com/pilot/ds-common.git
git submodule add https://gitlab.ds.com/pilot/ds-bo.git
git submodule add https://gitlab.ds.com/pilot/ds-fo.git
git submodule add https://gitlab.ds.com/pilot/ds-mo.git
git submodule add https://gitlab.ds.com/pilot/ds-batch.git
```

### 2-4. 소스 Merge 및 반영

- 해당 리포지토리에 작업된 소스를 Copy 후 add 및 commit, push 실행

```bash
// 소스 복사 후 이하작업 실행

git branch -M main
git add .
git commit -m 'first commit'
git push -uf origin main
```

### 2-5. Gitlab에서 소스반영 확인

- Sub Module 프로젝트는 ds-batch @ 9628356f 와 같이 '@ + 숫자'로 표시 확인


## 3. Git SubModule를 STS에서 Import

### 3-1. Git Repositoory 생성

- 1. 빈 workspaces 로 이동 (git repository를 받는 로컬의 위치가 workspaces가 됨)

![multi_project_import](./../../images/tech/multiproject_11.png)

- 2. 개발환경을 git.

![multi_project_import](./../../images/tech/multiproject_12.png)

- 3. Clone URI 를 선택.

![multi_project_import](./../../images/tech/multiproject_13.png)

- 4. Repository URI입력 후, next 를 클릭.

![multi_project_import](./../../images/tech/multiproject_14.png)
 
- 5. 로컬 git repository 위치를 선택,  
  - **Clone submodules 체크** 후 finish 를 클릭.

![multi_project_import](./../../images/tech/multiproject_15.png)

- 6. Git Repositoory 생성 확인

![multi_project_import](./../../images/tech/multiproject_16.png)

### 3-2. Repositoory Import

- 1. import projects를 클릭.

![multi_project_import](./../../images/tech/multiproject_17.png)

- 2. import source에 repository 위치를 선택 
  - **Detect and configure project natures의 선택을 해제** 후 Finish 를 클릭. 

![multi_project_import](./../../images/tech/multiproject_18.png)

- 2-1. 위의 import로 프로젝트가 생성되지 않았을때

  - Project -> import -> General -> Existing Projects into Workspace

![multi_project_import](./../../images/tech/multiproject_19.png)

![multi_project_import](./../../images/tech/multiproject_20.png)

![multi_project_import](./../../images/tech/multiproject_21.png)

- 3. project 생성확인 후 개발환경을 JAVA로 전환.

- 4. Gradle Nature 여부에 따라 이하 분기

- 4-1. project 우클릭 > Configure > Add Gradle Nature 를릭

- 4-2. project 우클릭 > Gradle > Refresh Gradle Project

![multi_project_import](./../../images/tech/multiproject_22.png)

  - Sub Module 들이 Multi Project로 설정되있는것 확인

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>