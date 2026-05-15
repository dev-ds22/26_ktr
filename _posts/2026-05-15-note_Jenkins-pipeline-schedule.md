---
layout: single
title: "Jenkins-pipeline-schedule"
excerpt: "Jenkins-pipeline-schedule"
categories:
  - "cicd"
tags:
  - "cicd"
  - "jenkins"
  - "batch"
toc: false
toc_sticky: true
date: "2026-05-15"
last_modified_at: "2026-05-15 15:14:06 +0900"
---
## Scheduling

![Pasted_image_20260515140939](./../../images/2026-05-15-note_Jenkins-pipeline-schedul/Pasted_image_20260515140939.png)

## 테스트용 원본
```bash
pipeline {
        agent any
        environment {
            // Shell target 경로

            TARGET_SHELL = "/sample/test_app/sh/testJob.sh"

            // SSH 접근 계정
            SSH_CREDENTIALS_ID = "sshagent"        

            // HOST 접근 변수 (계정 변경 필요)
            TARGET_HOST = "jboss@111.222.120.70"
        }
        stages {
            stage("JobExecute") {
                steps {
                    script {
                        echo '[JOB Execute] jobExecute'
                        sshagent (credentials: ["${SSH_CREDENTIALS_ID}"]){
                        sh """
                            ssh -o StrictHostKeyChecking=no -p 22222 ${TARGET_HOST} '
                            sh chmod 776 ${TARGET_SHELL}
                            sh ${TARGET_SHELL}
                            '
                            """                                              
                        }
                    }
                }
            }
        }
    }
```


정확도: 96%

## 1. 스크립트 전체 목적

이 Jenkins Pipeline은 **Jenkins 서버에서 원격 서버 `111.222.120.70`에 SSH로 접속한 뒤, 원격 서버에 있는 Shell Script `/sample/test_app/sh/testJob.sh`를 실행**하는 구조입니다.

```text
Jenkins Job 실행
→ Jenkins Agent에서 Pipeline 수행
→ sshagent로 SSH Key 로딩
→ ssh -p 22222 jboss@111.222.120.70 접속
→ 원격 서버의 /sample/test_app/sh/testJob.sh 실행
```

Jenkins Declarative Pipeline은 `pipeline { agent ... stages ... steps ... }` 구조를 사용하며, `stage`는 작업 단계를, `steps`는 실제 실행 작업을 정의합니다. Jenkins 공식 문서에서도 `stage`, `steps`, `sh`를 Declarative Pipeline의 핵심 구성으로 설명합니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com "Pipeline Syntax"))

## 2. 원본 스크립트

```groovy
pipeline {
    agent any
    environment {
        // Shell target 경로
        TARGET_SHELL = "/sample/test_app/sh/testJob.sh"
        // SSH 접근 계정
        SSH_CREDENTIALS_ID = "sshagent"
        // HOST 접근 변수 (계정 변경 필요)
        TARGET_HOST = "jboss@111.222.120.70"
    }
    stages {
        stage("JobExecute") {
            steps {
                script {
                    echo '[JOB Execute] jobExecute'
                    sshagent (credentials: ["${SSH_CREDENTIALS_ID}"]){
                        sh """
                            ssh -o StrictHostKeyChecking=no -p 22222 ${TARGET_HOST} '
                            sh chmod 776 ${TARGET_SHELL}
                            sh ${TARGET_SHELL}
                            '
                        """
                    }
                }
            }
        }
    }
}
```

## 3. 구성 요소별 설명

|구분|설명|
|---|---|
|`pipeline`|Declarative Pipeline 최상위 블록|
|`agent any`|사용 가능한 Jenkins Agent 아무 곳에서나 실행|
|`environment`|Pipeline 전체에서 사용할 환경변수 선언|
|`stages`|실행 단계 묶음|
|`stage("JobExecute")`|원격 Shell 실행 단계|
|`steps`|해당 stage에서 수행할 작업|
|`script`|Groovy 스크립트 로직을 사용할 때 사용|
|`sshagent`|Jenkins Credential에 등록된 SSH Key를 임시 ssh-agent에 로딩|
|`sh`|Jenkins Agent OS에서 Shell 명령 실행|
|`ssh`|원격 서버 접속 후 명령 실행|

## 4. environment 블록 설명

```groovy
environment {
    TARGET_SHELL = "/sample/test_app/sh/testJob.sh"
    SSH_CREDENTIALS_ID = "sshagent"
    TARGET_HOST = "jboss@111.222.120.70"
}
```

### `TARGET_SHELL`

```text
/sample/test_app/sh/testJob.sh
```

원격 서버에서 실행할 Shell Script 경로입니다.  
주의할 점은 **Jenkins 서버의 경로가 아니라 원격 서버 `111.222.120.70` 내부 경로**라는 점입니다.

### `SSH_CREDENTIALS_ID`

```text
sshagent
```

Jenkins Credentials에 등록된 SSH 인증 정보의 ID입니다. Jenkins SSH Agent Plugin은 Pipeline에서 SSH Credential을 `ssh-agent` 형태로 사용할 수 있게 해주는 step을 제공합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/ssh-agent/?utm_source=chatgpt.com "SSH Agent Plugin"))

### `TARGET_HOST`

```text
jboss@111.222.120.70
```

원격 접속 대상입니다.

```text
계정: jboss
IP: 111.222.120.70
```

실무에서는 IP를 직접 넣기보다 Jenkins Parameter 또는 환경별 변수로 분리하는 것이 좋습니다.

## 5. sshagent 블록 설명

```groovy
sshagent (credentials: ["${SSH_CREDENTIALS_ID}"]){
    ...
}
```

이 블록 안에서 실행되는 `ssh`, `scp` 명령은 Jenkins Credential에 등록된 SSH Key를 사용할 수 있습니다.  
즉, 아래처럼 직접 private key 파일 경로를 노출하지 않아도 됩니다.

```bash
ssh -i /path/to/private_key jboss@111.222.120.70
```

실무적으로는 `sshagent` 방식이 Jenkins Credential을 통해 비밀키를 관리하므로, 스크립트에 key 파일 경로나 비밀번호를 직접 쓰는 방식보다 안전합니다.

## 6. sh 블록 설명

```groovy
sh """
    ssh -o StrictHostKeyChecking=no -p 22222 ${TARGET_HOST} '
    sh chmod 776 ${TARGET_SHELL}
    sh ${TARGET_SHELL}
    '
"""
```

Jenkins Agent에서 Linux Shell을 실행합니다. Jenkins 문서에서도 `sh` step은 Shell Script를 실행하는 Pipeline step으로 설명됩니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/?utm_source=chatgpt.com "Pipeline"))  
이 안에서 실제로는 다음 SSH 명령이 실행됩니다.

```bash
ssh -o StrictHostKeyChecking=no -p 22222 jboss@111.222.120.70 '
sh chmod 776 /sample/test_app/sh/testJob.sh
sh /sample/test_app/sh/testJob.sh
'
```

## 7. 관련 커맨드 상세 설명

### 7.1 `ssh`

```bash
ssh -o StrictHostKeyChecking=no -p 22222 jboss@111.222.120.70 '원격명령'
```

|옵션|의미|
|---|---|
|`ssh`|원격 서버에 SSH 접속|
|`-o StrictHostKeyChecking=no`|Host Key 확인을 자동 수락|
|`-p 22222`|SSH 포트 22222 사용|
|`jboss@111.222.120.70`|`jboss` 계정으로 해당 IP 접속|
|`'원격명령'`|접속 후 원격 서버에서 실행할 명령|

### 7.2 `StrictHostKeyChecking=no`

```bash
-o StrictHostKeyChecking=no
```

원격 서버의 host key를 엄격하게 검증하지 않겠다는 옵션입니다.  
보안상 편리하지만, 실무 운영환경에서는 권장되지 않습니다. SSH host key 검증은 접속 대상 서버가 내가 알고 있는 서버가 맞는지 확인하는 역할을 하며, 이를 비활성화하면 중간자 공격 위험이 커질 수 있습니다. OpenSSH 계열 SSH에서는 이 옵션으로 host key checking 동작을 제어합니다. ([eukhost](https://www.eukhost.com/kb/how-to-enable-stricthostkeychecking-in-ssh/?utm_source=chatgpt.com "Enable StrictHostKeyChecking in SSH"))  
운영 권장 방식:

```bash
ssh-keyscan -p 22222 111.222.120.70 >> ~/.ssh/known_hosts
ssh -o StrictHostKeyChecking=yes -p 22222 jboss@111.222.120.70 'sh /sample/test_app/sh/testJob.sh'
```

### 7.3 `chmod`

원본:

```bash
sh chmod 776 /sample/test_app/sh/testJob.sh
```

이 부분은 **잘못된 명령일 가능성이 높습니다.**  
`chmod`는 보통 아래처럼 실행합니다.

```bash
chmod 776 /sample/test_app/sh/testJob.sh
```

또는 실행 권한만 주려면:

```bash
chmod 750 /sample/test_app/sh/testJob.sh
```

`sh chmod 776 ...`는 `chmod`라는 파일을 `sh`로 실행하려는 의미가 될 수 있어 정상 동작하지 않을 가능성이 큽니다.  
정상:

```bash
chmod 750 /sample/test_app/sh/testJob.sh
```

비정상 가능:

```bash
sh chmod 776 /sample/test_app/sh/testJob.sh
```

### 7.4 `chmod 776`

```bash
chmod 776 testJob.sh
```

권한 의미:

```text
소유자: rwx = 7
그룹: rwx = 7
기타: rw- = 6
```

즉, 기타 사용자에게도 쓰기 권한이 있습니다. 운영 Shell Script에 `776`은 과도한 권한입니다.  
권장:

```bash
chmod 750 /sample/test_app/sh/testJob.sh
```

또는 실행만 필요하다면:

```bash
chmod 755 /sample/test_app/sh/testJob.sh
```

보안상 더 엄격하게는:

```bash
chmod 750 /sample/test_app/sh/testJob.sh
chown jboss:jboss /sample/test_app/sh/testJob.sh
```

### 7.5 `sh ${TARGET_SHELL}`

```bash
sh /sample/test_app/sh/testJob.sh
```

Shell Script를 `sh`로 실행합니다.  
단, 스크립트가 Bash 문법을 사용한다면 `sh`가 아니라 `bash`로 실행해야 합니다.

```bash
bash /sample/test_app/sh/testJob.sh
```

또는 파일 첫 줄에 shebang이 있다면:

```bash
#!/bin/bash
```

아래처럼 직접 실행할 수도 있습니다.

```bash
/sample/test_app/sh/testJob.sh
```

## 8. 현재 스크립트의 주요 문제점

|항목|문제|영향|
|---|---|---|
|`sh chmod`|명령 형식 오류 가능성|권한 변경 실패 가능|
|`chmod 776`|과도한 권한|보안 취약|
|`StrictHostKeyChecking=no`|Host Key 검증 비활성화|MITM 위험|
|원격 명령 에러 처리 부족|Shell 실패 시 원인 파악 어려움|장애 대응 어려움|
|`agent any`|어느 Jenkins Agent에서 실행될지 불명확|SSH 환경 차이 발생|
|절대 IP 하드코딩|환경 변경에 취약|운영/개발 분리 어려움|
|원격 Shell 중복 실행 방지 없음|동시 실행 시 충돌 가능|배치/파일 lock 문제|
|로그 관리 없음|원격 실행 결과 추적 어려움|장애 분석 어려움|

## 9. 현재 스크립트에서 특히 위험한 부분

### 9.1 `StrictHostKeyChecking=no`

개발/테스트에서는 편할 수 있지만 운영에서는 위험합니다.

```bash
ssh -o StrictHostKeyChecking=no ...
```

대책:

```bash
ssh-keyscan -p 22222 111.222.120.70 >> ~/.ssh/known_hosts
ssh -o StrictHostKeyChecking=yes -p 22222 ...
```

### 9.2 `chmod 776`

운영 스크립트에 기타 사용자 쓰기 권한을 줄 필요는 거의 없습니다.

```bash
chmod 776 testJob.sh
```

대책:

```bash
chmod 750 testJob.sh
```

### 9.3 원격 명령 실패 감지 부족

현재 구조는 원격에서 중간 명령이 실패해도 의도대로 빌드 실패가 잡히지 않을 수 있습니다.  
원격 Shell 안에서 아래 옵션을 사용하는 것이 좋습니다.

```bash
set -e
```

더 엄격하게:

```bash
set -euo pipefail
```

단, `sh`에서는 `pipefail`이 지원되지 않을 수 있으므로 Bash 실행이 필요합니다.

## 10. 권장 개선 스크립트

### 10.1 최소 수정 버전

```groovy
pipeline {
    agent any

    environment {
        TARGET_SHELL = "/sample/test_app/sh/testJob.sh"
        SSH_CREDENTIALS_ID = "sshagent"
        TARGET_HOST = "jboss@111.222.120.70"
        TARGET_PORT = "22222"
    }

    stages {
        stage("JobExecute") {
            steps {
                script {
                    echo "[JOB Execute] jobExecute"

                    sshagent(credentials: ["${SSH_CREDENTIALS_ID}"]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no -p ${TARGET_PORT} ${TARGET_HOST} '
                                set -e
                                chmod 750 ${TARGET_SHELL}
                                sh ${TARGET_SHELL}
                            '
                        """
                    }
                }
            }
        }
    }
}
```

변경점:

```text
1. sh chmod → chmod 수정
2. chmod 776 → chmod 750 수정
3. TARGET_PORT 분리
4. set -e 추가
```

## 10.2 운영 권장 버전

```groovy
pipeline {
    agent { label 'linux-deploy' }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        TARGET_SHELL = "/sample/test_app/sh/testJob.sh"
        SSH_CREDENTIALS_ID = "sshagent"
        TARGET_HOST = "jboss@111.222.120.70"
        TARGET_PORT = "22222"
    }

    stages {
        stage("Check Remote Host") {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS_ID}"]) {
                    sh """
                        ssh -o BatchMode=yes -o StrictHostKeyChecking=yes -p ${TARGET_PORT} ${TARGET_HOST} 'hostname && whoami'
                    """
                }
            }
        }

        stage("JobExecute") {
            steps {
                sshagent(credentials: ["${SSH_CREDENTIALS_ID}"]) {
                    sh """
                        ssh -o BatchMode=yes -o StrictHostKeyChecking=yes -p ${TARGET_PORT} ${TARGET_HOST} '
                            set -e
                            test -f ${TARGET_SHELL}
                            test -r ${TARGET_SHELL}
                            chmod 750 ${TARGET_SHELL}
                            bash ${TARGET_SHELL}
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "[JOB Execute] success"
        }
        failure {
            echo "[JOB Execute] failed"
        }
        always {
            echo "[JOB Execute] finished"
        }
    }
}
```

## 11. 운영 권장 버전 설명

|항목|설명|
|---|---|
|`agent { label 'linux-deploy' }`|SSH 실행 가능한 Agent로 제한|
|`timestamps()`|Jenkins 로그에 시간 표시|
|`disableConcurrentBuilds()`|동일 Job 동시 실행 방지|
|`timeout()`|원격 Shell 무한 대기 방지|
|`BatchMode=yes`|인증 실패 시 비밀번호 입력 대기 없이 즉시 실패|
|`StrictHostKeyChecking=yes`|Host Key 검증 활성화|
|`test -f`|원격 Shell 파일 존재 확인|
|`test -r`|읽기 권한 확인|
|`chmod 750`|실행 권한 보정|
|`bash`|Bash 문법 사용하는 Shell 대응|
|`post`|성공/실패/항상 처리|

## 12. 관련 커맨드 정리

### Jenkins 서버에서 SSH 접속 테스트

```bash
ssh -p 22222 jboss@111.222.120.70
```

### Jenkins 서버에서 원격 명령 테스트

```bash
ssh -p 22222 jboss@111.222.120.70 'hostname && whoami'
```

### Host Key 등록

```bash
ssh-keyscan -p 22222 111.222.120.70 >> ~/.ssh/known_hosts
```

### 원격 파일 존재 확인

```bash
ssh -p 22222 jboss@111.222.120.70 'test -f /sample/test_app/sh/testJob.sh && echo exists'
```

### 원격 파일 권한 확인

```bash
ssh -p 22222 jboss@111.222.120.70 'ls -l /sample/test_app/sh/testJob.sh'
```

### 원격 Shell 문법 검사

```bash
ssh -p 22222 jboss@111.222.120.70 'bash -n /sample/test_app/sh/testJob.sh'
```

### 원격 Shell 실행

```bash
ssh -p 22222 jboss@111.222.120.70 'bash /sample/test_app/sh/testJob.sh'
```

### 원격 Shell 실행 로그 저장

```bash
ssh -p 22222 jboss@111.222.120.70 'bash /sample/test_app/sh/testJob.sh >> /sample/test_app/log/testJob.log 2>&1'
```

## 13. 실무 체크리스트

```text
1. Jenkins Credential ID가 실제로 존재하는가?
2. Credential의 SSH Key가 원격 서버 jboss 계정의 authorized_keys에 등록되어 있는가?
3. Jenkins Agent에서 111.222.120.70:22222로 네트워크 접근 가능한가?
4. 원격 서버의 testJob.sh 파일이 존재하는가?
5. testJob.sh의 소유자와 권한이 적절한가?
6. testJob.sh가 sh 문법인지 bash 문법인지 확인했는가?
7. 동일 Job이 동시에 실행되어도 문제없는가?
8. 원격 Shell 실패 시 Jenkins Build가 실패로 처리되는가?
9. 운영환경에서 StrictHostKeyChecking=no를 제거했는가?
10. Shell 실행 로그가 남는가?
```

## 14. 최종 판단

현재 Pipeline은 구조적으로는 **Jenkins에서 SSH로 원격 서버 Shell을 실행하는 단순 Job 실행용 Pipeline**입니다.  
다만 실무 운영 기준으로는 아래 4가지는 반드시 수정하는 것이 좋습니다.

```text
1. sh chmod 776 → chmod 750 수정
2. StrictHostKeyChecking=no 제거 또는 known_hosts 사전 등록
3. disableConcurrentBuilds(), timeout() 추가
4. 원격 Shell 실행 전 test -f, test -r, bash -n 검증 추가
```

특히 `chmod 776`과 `StrictHostKeyChecking=no`는 운영 보안 관점에서 위험하므로, 개발 테스트용 임시 설정으로만 보는 것이 안전합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
