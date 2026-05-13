## 결론

가장 단순하고 운영하기 쉬운 방식은 **GitLab `dev` 브랜치 push → GitLab Webhook → Jenkins Pipeline Job 트리거 → `dev` 브랜치 checkout/build → SSH로 dev 서버 배포 → health check** 흐름입니다. Jenkins의 GitLab 플러그인은 GitLab webhook으로 Jenkins job을 트리거할 수 있고, webhook URL은 `https://JENKINS_URL/project/JOB_NAME` 형식을 사용해야 하며 `/job/.../build` 형태를 쓰면 플러그인을 우회하게 됩니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/ "GitLab | Jenkins plugin"))

## 전체 구성

```text
GitLab(dev push)
  -> Webhook(Push Events, branch=dev)
  -> Jenkins Pipeline Job
  -> Git clone(dev)
  -> Build
  -> SSH deploy to dev server
  -> health check
```

GitLab은 webhook의 `push` 이벤트를 외부 엔드포인트로 POST 할 수 있고, webhook 자체에서 브랜치 기준 필터링도 지원합니다. Jenkins GitLab 플러그인은 job 단에서도 브랜치 이름 또는 정규식 기준 필터링을 지원합니다. 따라서 **GitLab 쪽 1차 필터 + Jenkins 쪽 2차 필터**로 중복 방어를 거는 것이 가장 안전합니다. ([GitLab 문서](https://docs.gitlab.com/user/project/integrations/webhooks/ "Webhooks | GitLab Docs"))

## 1. Jenkins 플러그인 설치

다음 플러그인을 설치하는 구성이 가장 무난합니다.

| 플러그인             | 용도                               |
| ---------------- | -------------------------------- |
| GitLab Plugin    | GitLab webhook으로 Jenkins job 트리거 |
| Git Plugin       | GitLab 저장소 checkout              |
| Pipeline         | Jenkinsfile 실행                   |
| SSH Agent Plugin | dev 서버 SSH 배포                    |
- Jenkins의 GitLab 플러그인은 GitLab push나 merge request 이벤트로 빌드를 트리거할 수 있고, SSH Agent Plugin은 `sshagent` 스텝으로 Pipeline 안에서 SSH 자격증명을 사용할 수 있습니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/ "GitLab | Jenkins plugin"))

## 2. Jenkins 자격증명 준비

### 2-1. GitLab 저장소 clone 용

보통 `git@gitlab.example.com:group/project.git` 형태로 clone 하므로 Jenkins Credentials에 **SSH private key**를 등록합니다.

### 2-2. dev 서버 배포 용

dev 서버 접속용으로도 Jenkins Credentials에 **SSH username with private key**를 등록합니다.  
Jenkins 문서는 SSH 자격증명 생성 시 `Kind: SSH username with private key`를 사용하도록 안내합니다. 또한 GitLab 플러그인 문서상 **GitLab API 통신용 인증과 Git clone용 인증은 별개**입니다. 즉, GitLab connection/token 설정과 실제 repository clone용 SSH credentials는 따로 관리해야 합니다. ([Jenkins](https://www.jenkins.io/doc/book/using/using-agents/ "Using Jenkins agents"))

## 3. Jenkins Job 생성

### 3-1. Pipeline Job 생성

`New Item` → `Pipeline` 선택 후 예: `commerce-dev-deploy`

### 3-2. SCM 설정

Pipeline을 `Pipeline script from SCM`으로 두고 다음처럼 설정합니다.

|항목|값 예시|
|---|---|
|SCM|Git|
|Repository URL|`git@gitlab.example.com:group/project.git`|
|Credentials|GitLab clone용 SSH credential|
|Branch Specifier|`*/dev`|
|Script Path|`Jenkinsfile`|

### 3-3. Build Trigger 설정

`Build Triggers`에서 다음을 설정합니다.

|항목|값|
|---|---|
|Build when a change is pushed to GitLab|체크|
|Push Events|체크|
|Merge Request 관련 이벤트|해제|
|Branch Filter|`NameBasedFilter`|
|Include Branches|`dev`|
|GitLab 플러그인 문서상 Pipeline/Freestyle job에서는 `Build when a change is pushed to GitLab`를 선택하고, `Push Events` 등을 체크해 webhook으로 트리거할 수 있습니다. 또한 브랜치 필터는 이름 기반 또는 정규식 기반으로 줄 수 있습니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/ "GitLab \| Jenkins plugin"))||

## 4. Jenkins Webhook URL과 Secret Token 설정

Jenkins Job의 GitLab Trigger 설정에서 `Advanced`를 열어 `Secret Token`을 생성합니다. 그 다음 GitLab 프로젝트 Webhook에 아래 값을 넣습니다.

| 항목           | 값 예시                                                      |
| ------------ | --------------------------------------------------------- |
| URL          | `https://jenkins.example.com/project/commerce-dev-deploy` |
| Secret Token | Jenkins에서 생성한 토큰                                          |
| Event        | Push events                                               |
- GitLab 플러그인 문서에는 webhook URL 형식이 `https://JENKINS_URL/project/PROJECT_NAME`라고 명시되어 있고, job 설정의 `Advanced`에서 `Secret Token`을 생성한 뒤 GitLab Webhook의 `Secret Token` 칸에 같은 값을 넣는 절차가 나와 있습니다. 문서에는 `USERID:APITOKEN@...` 방식도 있지만, 일반적으로는 **URL에 자격증명을 직접 넣지 않고 Secret Token 방식**이 더 깔끔합니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/ "GitLab | Jenkins plugin"))
## 5. GitLab Webhook에서 `dev`만 오도록 제한

GitLab 프로젝트에서 `Settings -> Webhooks`로 들어가서:

| 항목                 | 값                  |
| ------------------ | ------------------ |
| Trigger            | Push events        |
| Branch filter type | Regular expression |
| Pattern            | `^dev$`            |

- GitLab 문서상 webhook의 push 이벤트는 브랜치 기준으로 필터링할 수 있고, wildcard pattern 또는 regular expression(RE2)을 사용할 수 있습니다. `dev` 브랜치만 보내려면 `^dev$`가 가장 명확합니다. ([GitLab 문서](https://docs.gitlab.com/user/project/integrations/webhooks/ "Webhooks | GitLab Docs"))

## 6. Jenkinsfile 예제

아래는 **`dev` 브랜치가 push되면 해당 소스를 build 후 dev 서버에 배포**하는 가장 기본적인 예시입니다.

```groovy
pipeline {
    agent any
    environment {
        APP_NAME   = 'commerce-api'
        DEPLOY_HOST = '10.10.10.21'
        DEPLOY_USER = 'deploy'
        DEPLOY_DIR  = '/app/commerce'
        HEALTH_URL  = 'http://10.10.10.21:8080/health'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git branch --show-current || true'
                sh 'git rev-parse --short HEAD'
            }
        }
        stage('Build') {
            steps {
                sh './gradlew clean bootJar -x test'
            }
        }
        stage('Deploy to DEV') {
            steps {
                sshagent(credentials: ['dev-server-ssh']) {
                    sh '''
                    set -e
                    ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} "mkdir -p ${DEPLOY_DIR}"
                    scp -o StrictHostKeyChecking=no build/libs/*.jar ${DEPLOY_USER}@${DEPLOY_HOST}:${DEPLOY_DIR}/${APP_NAME}.jar
                    ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} '
                        set -e
                        pkill -f ${APP_NAME}.jar || true
                        nohup java -jar ${DEPLOY_DIR}/${APP_NAME}.jar > ${DEPLOY_DIR}/${APP_NAME}.log 2>&1 &
                    '
                    '''
                }
            }
        }
        stage('Health Check') {
            steps {
                sh '''
                set +e
                for i in $(seq 1 30); do
                  CODE=$(curl -s -o /tmp/dev-health.json -w "%{http_code}" ${HEALTH_URL})
                  if [ "$CODE" = "200" ] && grep -q '"status":"UP"' /tmp/dev-health.json; then
                    echo "Health check success"
                    exit 0
                  fi
                  sleep 2
                done
                echo "Health check failed"
                cat /tmp/dev-health.json || true
                exit 1
                '''
            }
        }
    }
    post {
        success {
            echo 'DEV deploy success'
        }
        failure {
            echo 'DEV deploy failed'
        }
    }
}
```

Jenkins의 SSH Agent Plugin은 `sshagent(credentials: ['...'])` 형태로 SSH 자격증명을 Pipeline 단계에서 사용할 수 있습니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/ssh-agent/ "SSH Agent Plugin"))

## 7. dev 서버 측 권장 배포 스크립트 분리 방식

실무에서는 Jenkinsfile 안에 긴 운영 명령을 다 넣기보다, dev 서버에 별도 스크립트를 두는 편이 더 관리하기 쉽습니다.

### dev 서버 예시: `/app/commerce/deploy-dev.sh`

```bash
#!/bin/bash
set -e
APP_NAME="commerce-api"
DEPLOY_DIR="/app/commerce"
JAR_FILE="${DEPLOY_DIR}/${APP_NAME}.jar"
LOG_FILE="${DEPLOY_DIR}/${APP_NAME}.log"

pkill -f ${APP_NAME}.jar || true
nohup java -jar ${JAR_FILE} > ${LOG_FILE} 2>&1 &
```

### Jenkinsfile의 deploy 단계 단순화

```groovy
stage('Deploy to DEV') {
    steps {
        sshagent(credentials: ['dev-server-ssh']) {
            sh '''
            set -e
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} "mkdir -p ${DEPLOY_DIR}"
            scp -o StrictHostKeyChecking=no build/libs/*.jar ${DEPLOY_USER}@${DEPLOY_HOST}:${DEPLOY_DIR}/${APP_NAME}.jar
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${DEPLOY_HOST} "bash ${DEPLOY_DIR}/deploy-dev.sh"
            '''
        }
    }
}
```

## 8. Git 명령어 예시

`dev` 브랜치에 Jenkinsfile을 추가하고 push하는 기본 명령은 아래와 같습니다.

```bash
git checkout dev
git pull origin dev
vi Jenkinsfile
git add Jenkinsfile
git commit -m "chore: add jenkins pipeline for dev deploy"
git push origin dev
```

이후부터는 `dev` 브랜치에 push가 발생할 때마다 GitLab의 push webhook이 Jenkins를 호출하고, Jenkins job이 `dev` 브랜치를 checkout 해서 build/deploy 하게 됩니다. GitLab webhook의 push 이벤트와 Jenkins GitLab trigger의 Push Events가 이 연결의 핵심입니다. ([GitLab 문서](https://docs.gitlab.com/user/project/integrations/webhooks/ "Webhooks | GitLab Docs"))

## 9. 추천 운영 설정

| 항목                           | 권장값                       |
| ---------------------------- | ------------------------- |
| GitLab webhook branch filter | `^dev$`                   |
| Jenkins branch filter        | `NameBasedFilter` + `dev` |
| Git clone credential         | SSH key                   |
| dev 서버 배포 credential         | 별도 SSH key                |
| 배포 후 확인                      | health check 필수           |
| Jenkins job 권한               | 최소 권한                     |
| 배포 스크립트                      | 서버에 분리                    |
- 브랜치 필터는 GitLab과 Jenkins 양쪽에 둘 수 있고, Jenkins GitLab 플러그인은 branch 이름 기반 필터와 regex 기반 필터를 둘 다 지원합니다. ([GitLab 문서](https://docs.gitlab.com/user/project/integrations/webhooks/ "Webhooks | GitLab Docs"))

## 10. 자주 틀리는 부분

| 실수                                                | 문제                               |
| ------------------------------------------------- | -------------------------------- |
| Webhook URL을 `/job/.../build`로 입력                 | GitLab plugin 트리거를 우회해서 동작 이상 가능 |
| GitLab API token과 Git clone credential을 같은 것으로 착각 | clone 실패 또는 status 연동 실패         |
| MR 이벤트까지 같이 체크                                    | 원하지 않는 배포 발생 가능                  |
| 브랜치 필터 미설정                                        | feature 브랜치 push도 dev 배포될 수 있음   |
| Jenkinsfile만 만들고 webhook 테스트 안 함                  | 권한/URL/토큰 오류를 늦게 발견              |
- Webhook URL은 `/project/JOB_NAME` 형식을 써야 하고, GitLab plugin 문서상 GitLab API용 인증과 Git clone용 인증은 분리되어 있습니다. 또한 Push Events 외의 MR 관련 이벤트를 같이 켜면 의도치 않게 build/deploy가 늘어날 수 있습니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/ "GitLab | Jenkins plugin"))

## 11. 가장 권장하는 적용 순서

1. Jenkins에 `GitLab Plugin`, `Git Plugin`, `Pipeline`, `SSH Agent Plugin` 설치
    
2. GitLab clone용 SSH credential 등록
    
3. dev 서버 배포용 SSH credential 등록
    
4. Jenkins Pipeline Job 생성
    
5. SCM을 GitLab repo + `*/dev`로 설정
    
6. Build Trigger에서 `Build when a change is pushed to GitLab` + `Push Events` 설정
    
7. Jenkins에서 Secret Token 생성
    
8. GitLab Webhook에 Jenkins URL/Secret Token 등록
    
9. GitLab Webhook branch regex를 `^dev$`로 설정
    
10. Jenkinsfile push
    
11. GitLab Webhook `Test` 실행
    
12. 실제 `dev` 브랜치에 commit/push 하여 자동배포 검증  
    원하시면 다음 답변에서 **Jenkins UI 기준 실제 클릭 경로별로 화면 메뉴 순서까지** 아주 세세하게 풀어서 정리해드리겠습니다.