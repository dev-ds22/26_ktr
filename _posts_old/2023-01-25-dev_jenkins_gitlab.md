---
layout: single
title: "[Jenkins] jenkins gitlab 연동 및 자동배포"
excerpt: "jenkins gitlab 연동 및 자동배포(CentOS)"

categories:
  - tech
tags:
  - [jenkins, gitlab]

toc: false
toc_sticky: true

date: 2023-01-25
last_modified_at: 2023-01-25
---

# jenkins gitlab 연동 및 자동배포

## 1. jenkins Web hook URL 확인 및 Token 생성

![inst gitlab](./../../images/tech/jenkins_gitlab_01.png)

- WebHook URL : http://pixxx.xxxx.com:8099/project/xxxxx-pub
- Trigger 이벤트 설정 : Push Events

![inst gitlab](./../../images/tech/jenkins_gitlab_02.png)

- Secret Token 생성 : b586c2195a13f8a94b91d1cfxxxxx

## 2. gitlab Web hook 설정

![inst gitlab](./../../images/tech/jenkins_gitlab_03.png)

- 하단 SSL verification 의 Enable SSL verification 체크해제

#### 참고.1 Jenkins 유저로 전환

```bash
  su - jenkins -s /bin/bash

  sudo chown -R jenkins /usr/local/lib/node_modules/
  sudo chown -R jenkins /usr/local/bin/
  sudo chown -R jenkins /usr/local/share/
```




```bash
  cp /apps/daiso-backend/build.gradle /apps/daiso-bo/build.gradle
  cp /apps/daiso-backend/settings.gradle /apps/daiso-bo/settings.gradle
```

#### 참고.2 Spring Boot 어플리케이션 실행시 pid 파일을 생성
- application.yaml

```bash
  spring: 
    pid:
      file: /apps/pilot/build/mo_boot.pid    
```

- Applcation 설정

```java
  @SpringBootApplication
  public class TestApplication {

      public static void main(String[] args) {
          SpringApplication application = new SpringApplication(TestApplication.class);
          application.addListeners(new ApplicationPidFileWriter());
          application.run(args);
      }
  }
```

- 실제 사용할 때 아래와 같이 실행하면 pid 파일에 숫자가 채워진다.

```bash
  BUILD_ID=dontKillMe nohup java -Dspring.profiles.active=dev -jar $JAR_PATH >> /var/lib/jenkins/workspace/nohup_daiso_mo.out &
```

$ cat mo_boot.pid # 다른 터미널에서 확인해보면 숫자가 채워진 것을 알 수 있다.

- 부트 어플리케이션 종료

```bash
  $ kill `cat test.pid` # WAS 종료 시
```

### Spring 내장 Tomcat 동작시 nohup 동작안될 때
- jenkins는 빌드 과정을 모두 마친 후, jenkins 사용자로 실행된 child process를 모두 kill 시키기 때문에 nohup같은 명령어가 계속 종료되는 현상이 발생.
- 이를 방지하기 위해 nohup 명령어 앞에 BUILD_ID=dontKillMe를 붙임.
- 젠킨스 build과정을 모두 마친 후, 종료되지 않기를 원하는 명령어에는 반드시 BUILD_ID=dontKillMe 를 추가.

```bash
#!/bin/bash
echo "DA-FO Backend PID Check..."

CURRENT_PID=`cat /apps/pilot/build/fo_boot.pid`
JAR_PATH=/apps/daiso-fo/daiso-fo/build/libs/daiso-fo.jar

echo "Running PID - DA-FO : {$CURRENT_PID}"

if [ -z "$CURRENT_PID" ]
then
  echo "DA-FO Project is not running"
else
  if ps -p $CURRENT_PID > /dev/null 2>&1 ; then
  	kill -15 $CURRENT_PID
  	sleep 10
  fi
fi

echo "Deploy DA-FO Project...."

BUILD_ID=dontKillMe nohup java -Dspring.profiles.active=dev -jar $JAR_PATH >> /var/lib/jenkins/workspace/nohup_daiso_fo.out &

echo "Done"
```

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

# FrontOffice
```bash
cd /usr/local/daiso-tomcat/vue/daiso-vue-pub
npm install --force &&
npm run build &&
pm2 stop -s daiso-pub || :
pm2 delete -s daiso-pub || :
set DEBUG=express:* & pm2 start npm --name daiso-pub -- run start
pm2 save || :
```

# BackOffice
```bash
#!/bin/bash
echo "PID Check..."

CURRENT_PID=$(ps -ef | grep java | grep PlatformCommon* | awk '{print $2}')
JAR_PATH=/var/jenkins_home/workspace/Dev-PlatformCommon/build/libs/PlatformCommon-0.0.1-SNAPSHOT.jar

echo "Running PID: {$CURRENT_PID}"

if [ -z "$CURRENT_PID" ]
then
  echo "Project is not running"
else
  kill -15 $CURRENT_PID
  sleep 10
fi

echo "Deploy Project...."

nohup java -Dspring.profiles.active=m2mdev -jar $JAR_PATH >> /var/jenkins_home/workspace/nohup.out &

echo "Done"
```

  </pre>
</details>