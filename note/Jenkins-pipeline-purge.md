---
categories:
  - cicd
tags:
  - cicd
  - jenkins
  - cdn
---
## Purge 용 스크립트
```bash
pipeline {
    agent any

    environment {
        method = "POST"
        url = "https://g-openapi.samsungsdscloud.com/cdn~~~~~~~~~~~~~~~~~~~~/purge"
        clientType = "OpenApi"
        timestamp = sh(script: 'date +%s%3N', returnStdout: true).trim() // milliseconds since epoch
        customUrl = "/"
        targetContent = "INVALIDATED_CONTENT"
        targetUrl = "DOMAIN"
    }

    stages {
        stage('Install npm packages') {
            steps {
                script {
                    // npm 패키지 설치
                    sh 'npm install crypto-js'
                }
            }
        }

        stage('Make Signature') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'accessYek', variable: 'accessYek'),
                        string(credentialsId: 'secretYek', variable: 'secretYek'),
                        string(credentialsId: 'projectId', variable: 'projectId')
                    ]) {
                        // 메시지 생성
                        def message = "${method}${url}${timestamp}${accessYek}${projectId}${clientType}"

                        // 서명 생성
                        def signature = sh(
                            script: """
                                node -e "const CryptoJS = require('crypto-js'); const message = '${message}'; const secretYek = '${secretYek}'; console.log(CryptoJS.HmacSHA256(message, secretYek).toString(CryptoJS.enc.Base64));"
                            """,
                            returnStdout: true
                        ).trim()

                        env.signature = signature
                    }
                }
            }
        }

        stage('HTTP Request') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'accessYek', variable: 'accessYek'),
                        string(credentialsId: 'secretYek', variable: 'secretYek'),
                        string(credentialsId: 'projectId', variable: 'projectId')
                    ]) {
                        // HTTP POST 요청
                        def response = httpRequest(
                            acceptType: 'APPLICATION_JSON',
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            url: url,
                            customHeaders: [
                                ["name": "X-Cmp-AccessYek", "value": accessYek],
                                ["name": "X-Cmp-Signature", "value": signature],
                                ["name": "X-Cmp-Timestamp", "value": timestamp],
                                ["name": "X-Cmp-ClientType", "value": clientType],
                                ["name": "X-Cmp-ProjectId", "value": projectId]
                            ],
                            requestBody: """
                            {
                                "customUrl" : "${customUrl}",
                                "targetContent" : "${targetContent}",
                                "targetUrl" : "${targetUrl}"
                            }
                            """
                        )
                    }
                }
            }
        }
    }
}
```

## 1. 전체 목적

이 Jenkins Pipeline은 **Samsung SDS Cloud CDN OpenAPI의 purge API를 호출하여 CDN 캐시를 무효화**하는 자동화 스크립트입니다. 전체 흐름은 아래입니다.

```text
Jenkins Job 실행
→ npm crypto-js 설치
→ Access Key / Secret Key / Project ID를 Jenkins Credentials에서 조회
→ HMAC-SHA256 Signature 생성
→ CDN Purge API로 HTTP POST 요청
→ CDN 캐시 무효화 수행
```

Samsung Cloud Platform OpenAPI 문서 기준 인증 헤더는 `X-Cmp-AccessKey`, `X-Cmp-Signature`, `X-Cmp-Timestamp`이며, Signature는 요청 문자열을 Access Secret Key로 HMAC-SHA256 처리 후 Base64 인코딩하는 방식입니다.

## 2. 스크립트 구성 분석

### 2.1 `agent any`

```groovy
agent any
```

사용 가능한 Jenkins Agent 아무 곳에서나 실행됩니다. Jenkins Declarative Pipeline에서 `agent any`는 사용 가능한 임의의 Agent에서 Pipeline 또는 Stage를 실행한다는 의미입니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/ "Pipeline Syntax"))  
실무에서는 purge API 호출이 가능한 Agent를 고정하는 것이 안전합니다.

```groovy
agent { label 'linux-deploy' }
```

이유:

```text
1. date +%s%3N 명령이 Linux 기준
2. node/npm 설치 여부가 Agent마다 다를 수 있음
3. 외부 OpenAPI endpoint 접근 가능 여부가 Agent마다 다를 수 있음
4. 운영 purge 권한을 가진 Job은 실행 노드를 제한하는 것이 안전
```

## 2.2 `environment`

```groovy
environment {
    method = "POST"
    url = "https://g-openapi.samsungsdscloud.com/cdn~~~~~~~~~~~~~~~~~~~~/purge"
    clientType = "OpenApi"
    timestamp = sh(script: 'date +%s%3N', returnStdout: true).trim()
    customUrl = "/"
    targetContent = "INVALIDATED_CONTENT"
    targetUrl = "DOMAIN"
}
```

Jenkins Declarative Pipeline은 `environment` directive를 통해 전체 Pipeline에서 사용할 환경변수를 지정할 수 있고, `sh(returnStdout: true)`를 이용한 동적 환경변수 설정도 가능합니다. 단, `returnStdout` 결과에는 trailing whitespace가 붙을 수 있어 `.trim()` 처리가 필요합니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/ "Using a Jenkinsfile"))

|변수|의미|실무 주의|
|---|---|---|
|`method`|HTTP Method|Signature 문자열과 실제 요청 method가 반드시 같아야 함|
|`url`|Purge API URL|Signature 생성 URL과 실제 호출 URL이 byte 단위로 같아야 함|
|`clientType`|OpenAPI client type|API 스펙 대소문자 확인 필요|
|`timestamp`|Epoch milliseconds|요청 직전에 생성하는 것이 안전|
|`customUrl`|Purge 대상 경로|`/`는 전체 purge 가능성이 있어 위험|
|`targetContent`|purge 대상 콘텐츠|SDS CDN API 스펙 값 확인 필요|
|`targetUrl`|purge 대상 URL 타입|`DOMAIN`이면 도메인 단위 purge 가능성|

## 3. Stage별 상세 분석

## 3.1 `Install npm packages`

```groovy
stage('Install npm packages') {
    steps {
        script {
            sh 'npm install crypto-js'
        }
    }
}
```

`crypto-js`를 설치해서 HMAC-SHA256 서명을 만들기 위한 단계입니다.

### 실무 문제점

```text
1. 매번 npm install 수행 → purge job 지연
2. npm registry 장애 시 purge 불가
3. package-lock.json 없이 설치 → crypto-js 버전 재현성 약함
4. 외부 패키지 supply-chain 위험
5. Jenkins workspace에 node_modules 누적 가능
```

### 권장 대안

Node.js에는 내장 `crypto` 모듈이 있으므로 별도 npm 설치 없이 HMAC-SHA256 생성이 가능합니다.

```bash
node -e "const crypto=require('crypto'); console.log(crypto.createHmac('sha256', process.env.SECRET_KEY).update(process.env.MESSAGE, 'utf8').digest('base64'));"
```

운영 purge Job에서는 **npm install 제거**를 권장합니다.

실무 주의점:

```
1. 매 빌드마다 npm install을 수행하면 느림
2. 외부 npm registry 장애 시 purge job 실패 가능
3. package-lock 없이 설치하면 매번 다른 버전이 설치될 수 있음
4. supply-chain 보안 위험 존재
5. Jenkins workspace에 node_modules가 누적될 수 있음
```

실무에서는 아래 중 하나가 더 낫습니다.

```
1. Jenkins Agent 이미지에 crypto-js 사전 설치
2. package.json/package-lock.json을 SCM에 고정
3. Node.js 대신 openssl 또는 Python 표준 라이브러리로 HMAC 생성
4. Jenkins Shared Library로 Signature 생성 로직 공통화
```
## 3.2 `Make Signature`

```groovy
withCredentials([
    string(credentialsId: 'accessKey', variable: 'accessKey'),
    string(credentialsId: 'secretKey', variable: 'secretKey'),
    string(credentialsId: 'projectId', variable: 'projectId')
]) {
    def message = "${method}${url}${timestamp}${accessKey}${projectId}${clientType}"
```

Jenkins `withCredentials`는 credential을 step scope 안의 환경변수로 바인딩합니다. Jenkins 문서도 secret은 해당 scope에서 환경변수로 사용 가능하다고 설명합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/ "Credentials Binding Plugin"))

### 현재 Signature 문자열

```text
method + url + timestamp + accessKey + projectId + clientType
```

하지만 Samsung Cloud Platform OpenAPI 샘플 Java code는 아래 순서로 서명 문자열을 만들고, `MULTIPART_FORM_DATA`가 아니면 `requestBody`도 append합니다.

```text
method
+ url
+ timestamp
+ accessKey
+ projectId
+ clientType
+ requestBody
```

현재 스크립트는 `POST` 요청인데 **requestBody를 Signature 대상에 포함하지 않습니다.**  
이 API가 공식 샘플 방식대로 검증한다면 다음 오류 가능성이 있습니다.

```text
401 Unauthorized
Invalid signature
Signature mismatch
Authentication failed
```

## 3.3 Node 서명 생성 방식

현재:

```groovy
def signature = sh(
    script: """
        node -e "const CryptoJS = require('crypto-js'); const message = '${message}'; const secretKey = '${secretKey}'; console.log(CryptoJS.HmacSHA256(message, secretKey).toString(CryptoJS.enc.Base64));"
    """,
    returnStdout: true
).trim()
```

### 문제점

문제점:

```
1. Secret Key가 shell command line에 들어감
2. Secret에 ', ", \, $, ` 문자가 있으면 스크립트 깨질 수 있음
3. Jenkins 로그 masking이 완전한 보안 대책은 아님
4. 같은 Agent의 다른 프로세스가 환경/프로세스 정보를 볼 가능성
```

Jenkins Credentials Binding 문서도 secret은 로그에서 masking되지만, masking은 우회 가능하며 같은 노드에서 신뢰할 수 없는 Job과 함께 실행하지 말아야 한다고 설명합니다.

| 항목                   | 문제                                                 |
| -------------------- | -------------------------------------------------- |
| Groovy interpolation | `${secretKey}`가 shell command에 직접 삽입됨              |
| 로그/프로세스 노출           | masking이 되더라도 프로세스 명령행 노출 위험                       |
| 특수문자 취약              | secret에 `'`, `"`, `\`, `$`, newline 등이 있으면 깨질 수 있음 |
| 외부 패키지 의존            | `crypto-js` 설치 필요                                  |

- Jenkins Credentials Binding 문서는 secret을 Groovy interpolation으로 넘기는 방식은 덜 안전하며, shell 환경변수로 확장되도록 single quote script를 쓰는 방식을 권장합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/credentials-binding/ "Credentials Binding Plugin"))

##### 문제점:
```
1. Secret Key가 shell command line에 들어감
2. Secret에 ', ", \, $, ` 문자가 있으면 스크립트 깨질 수 있음
3. Jenkins 로그 masking이 완전한 보안 대책은 아님
4. 같은 Agent의 다른 프로세스가 환경/프로세스 정보를 볼 가능성
```

Jenkins Credentials Binding 문서도 secret은 로그에서 masking되지만, masking은 우회 가능하며 같은 노드에서 신뢰할 수 없는 Job과 함께 실행하지 말아야 한다고 설명합니다.
## 3.4 `HTTP Request`

```groovy
def response = httpRequest(
    acceptType: 'APPLICATION_JSON',
    contentType: 'APPLICATION_JSON',
    httpMode: 'POST',
    url: url,
    customHeaders: [
        ["name": "X-Cmp-AccessKey", "value": accessKey],
        ["name": "X-Cmp-Signature", "value": signature],
        ["name": "X-Cmp-Timestamp", "value": timestamp],
        ["name": "X-Cmp-ClientType", "value": clientType],
        ["name": "X-Cmp-ProjectId", "value": projectId]
    ],
    requestBody: """
    {
        "customUrl" : "${customUrl}",
        "targetContent" : "${targetContent}",
        "targetUrl" : "${targetUrl}"
    }
    """
)
```

Jenkins `httpRequest` step은 HTTP 요청을 수행하고 `response.status`, `response.content`를 가진 response object를 반환합니다. 또한 `customHeaders`는 `maskValue`, `timeout`, `validResponseCodes`, `consoleLogResponseBody`, `quiet`, `requestBody` 등을 지원합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/http_request/ "HTTP Request Plugin"))  
현재 스크립트는 response를 받지만 검증하지 않습니다.

```groovy
def response = httpRequest(...)
```

운영에서는 최소한 아래 설정이 필요합니다.
```
echo "Purge API status: ${response.status}"
```

단, 응답 본문에 민감정보가 포함될 가능성이 있으면 body 출력은 제한해야 합니다.
```groovy
validResponseCodes: '200:299',
timeout: 30,
consoleLogResponseBody: false,
quiet: true
```

## 4. 현재 스크립트의 핵심 위험

|구분|현재 상태|위험|
|---|---|---|
|Header명|`X-Cmp-AccessKey` 사용|이전 오타는 수정됨|
|Signature|requestBody 미포함|API 스펙에 따라 인증 실패 가능|
|Timestamp|environment에서 Job 초기에 생성|npm install 등으로 지연되면 만료 가능|
|npm|매번 `npm install crypto-js`|장애·속도·보안·재현성 문제|
|Secret 처리|Groovy interpolation|secret 노출·깨짐 위험|
|Purge 범위|`customUrl="/"`|전체 purge 가능성|
|응답 검증|없음|실패 원인 추적 어려움|
|Header masking|없음|AccessKey/Signature 노출 위험|
|동시 실행|제한 없음|중복 purge, origin 부하|
|승인 절차|없음|운영 전체 purge 실수 위험|

## 5. purge 용도 실무 주의사항

## 5.1 `customUrl="/"`는 가장 위험한 설정

```groovy
customUrl = "/"
targetUrl = "DOMAIN"
```

이 조합은 CDN API 스펙에 따라 **도메인 전체 캐시 무효화**가 될 수 있습니다.  
전체 purge의 영향:

```text
1. CDN Edge 캐시 대량 삭제
2. 이후 요청이 Origin으로 집중
3. Origin Nginx/WAS/Object Storage 부하 증가
4. TTFB 증가
5. 사용자는 화면 로딩 지연 경험
6. 장애처럼 보이는 현상 발생 가능
```

운영 기준:

```text
전체 purge는 장애 대응, 긴급 롤백, 보안 사고 대응 등 예외 상황으로 제한
평상시 배포는 변경 URL만 purge
정적 리소스는 파일명 hash + long cache 전략으로 purge 의존도 축소
```

## 5.2 Purge 후 캐시 미스 폭증

Purge는 캐시를 삭제하거나 stale 처리하는 작업입니다.

```text
Purge
→ CDN Edge 캐시 제거
→ 사용자 요청 유입
→ CDN MISS
→ Origin Fetch
→ Origin 부하 증가
→ TTFB 증가
```

대책:

```text
1. URL 단위 purge
2. purge 후 preload/warm-up
3. 배포 직후 트래픽이 낮은 시간대 수행
4. CDN HIT ratio, MISS ratio, Origin request count 모니터링
5. 전체 purge는 승인 기반으로만 수행
```

## 5.3 QueryString purge 주의

이전에 확인한 것처럼 `selectDigtlKbcGoodsListMng.js?ver=5.035555555` 같은 URL은 CDN Cache Key 정책에 따라 별도 객체가 될 수 있습니다.  
확인 필요:

```text
1. CDN Cache Key에 QueryString이 포함되는가?
2. purge 대상에 QueryString까지 포함해야 하는가?
3. Path purge가 query variant 전체를 제거하는가?
4. `?ver=...` 값별 객체가 따로 남는가?
```

## 5-4. Purge 용도로 실무 사용 시 가장 중요한 주의점

### 5-4-1 `/` 전체 Purge는 매우 위험

```
customUrl = "/"targetUrl = "DOMAIN"
```

이 조합은 API 스펙에 따라 **도메인 전체 캐시 무효화**로 해석될 수 있습니다.  
운영에서 전체 purge를 수행하면 아래 문제가 생깁니다.

```
1. CDN Edge 캐시 대량 삭제2. 사용자 요청이 Origin으로 집중3. Origin 서버, Nginx, WAS, Object Storage 부하 급증4. TTFB 증가5. 장애처럼 보이는 느림 발생6. CDN 비용 증가 가능
```

실무 원칙:

```
전체 Purge는 장애 대응, 긴급 롤백, 보안 사고 대응 시에만 제한적으로 사용평소 배포에서는 변경된 URL만 Purge정적 파일은 파일명 hash + long cache 전략을 우선 적용
```

정적 자산은 URL에 해시/버전을 넣고 `Cache-Control: max-age=31536000, immutable` 같은 장기 캐시를 적용하는 추세입니다. 파일이 바뀌면 새 URL을 쓰는 cache-busting 패턴을 사용합니다.

### 5-4-2 Purge 후 Origin 부하 대비 필요

Purge는 캐시 삭제입니다. 삭제 후 첫 요청은 대부분 Origin으로 갑니다.

```
Purge 실행→ CDN Edge 캐시 제거→ 사용자 요청 유입→ CDN MISS→ Origin Fetch→ TTFB 증가 가능
```

대책:

```
1. purge 후 주요 URL preload/warm-up 수행2. 전체 purge 대신 URL 단위 purge3. 배포 직후 트래픽 낮은 시간에 수행4. Origin 서버 부하 지표 모니터링5. CDN HIT ratio, MISS ratio 확인
```

### 5-4-3 Purge와 Invalidate의 의미 구분

CDN마다 용어가 다릅니다.

|용어|일반적 의미|
|---|---|
|Purge/Delete|캐시 객체 삭제|
|Invalidate|캐시 객체를 stale 처리하고 재검증 유도|
|Refresh|Origin에서 다시 가져와 캐시 갱신|
|Preload/Prefetch|Edge에 미리 적재|
|현재 값:||

```
targetContent = "INVALIDATED_CONTENT"
```

이 값이 실제로 **삭제인지, TTL 0 처리인지, stale 처리인지**는 Samsung SDS CDN API 스펙으로 확인해야 합니다.
## 6. 관련 커맨드 설명

### 6.1 timestamp 생성

```bash
date +%s%3N
```

의미:

```text
%s  : Unix epoch seconds
%3N : nanoseconds 앞 3자리, milliseconds
```

결과 예:

```text
1715750000123
```

- Linux GNU date 기준으로 동작합니다. Windows Agent나 일부 Unix 환경에서는 동일하게 동작하지 않을 수 있습니다.
- Windows Agent에서는 동작하지 않을 수 있으므로 `agent any`보다 Linux Agent를 지정하는 것이 안전합니다.
```
agent { label 'linux' }
```
## 6.2 Node 내장 crypto로 HMAC 생성

```
node -e "const CryptoJS = require('crypto-js'); ..."
```

`crypto-js`를 사용해 HMAC-SHA256 후 Base64로 변환합니다.  
단, 실무에서는 npm install 의존도를 줄이기 위해 Node 내장 `crypto` 사용을 권장합니다.

```
node -e "const crypto=require('crypto'); console.log(crypto.createHmac('sha256', process.env.SECRET_KEY).update(process.env.MESSAGE, 'utf8').digest('base64'));"
```

이 방식은 별도 `crypto-js` 설치가 필요 없습니다.

```bash
MESSAGE="..." SECRET_KEY="..." node -e "const crypto=require('crypto'); console.log(crypto.createHmac('sha256', process.env.SECRET_KEY).update(process.env.MESSAGE, 'utf8').digest('base64'));"
```

장점:

```text
1. npm install 불필요
2. 실행 속도 빠름
3. 외부 패키지 보안 위험 감소
4. 버전 재현성 개선
```

## 6.3 curl 테스트 구조
- Jenkins 밖에서 검증할 때는 아래 구조로 테스트할 수 있습니다.
```bash
curl -i -X POST "$URL" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Cmp-AccessKey: $ACCESS_KEY" \
  -H "X-Cmp-Signature: $SIGNATURE" \
  -H "X-Cmp-Timestamp: $TIMESTAMP" \
  -H "X-Cmp-ClientType: OpenApi" \
  -H "X-Cmp-ProjectId: $PROJECT_ID" \
  --data "$REQUEST_BODY"
```

중요:

```text
Signature 생성에 사용한 requestBody와 실제 --data로 보낸 requestBody가 byte 단위로 같아야 할 수 있음
공백, 줄바꿈, key 순서 차이로 signature mismatch가 날 수 있음
```

## 7. 보안 사항

## 7.1 `withCredentials` 사용은 맞지만 shell 전달 방식은 개선 필요

현재 `withCredentials` 사용 자체는 맞습니다. 하지만 secret을 `""" ... ${secretKey} ... """` 형태로 shell command에 직접 삽입하는 방식은 피해야 합니다.  
권장:

```groovy
sh(
  script: '''
    set +x
    node -e "const crypto=require('crypto'); console.log(crypto.createHmac('sha256', process.env.SECRET_KEY).update(process.env.MESSAGE, 'utf8').digest('base64'));"
  ''',
  returnStdout: true
)
```

단, `MESSAGE`, `SECRET_KEY`를 안전하게 환경변수로 넘기는 구조가 필요합니다.

## 7.2 `maskValue` 적용

`httpRequest`의 `customHeaders`는 `maskValue`를 지원합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/http_request/ "HTTP Request Plugin"))

```groovy
customHeaders: [
    [name: "X-Cmp-AccessKey", value: accessKey, maskValue: true],
    [name: "X-Cmp-Signature", value: env.signature, maskValue: true],
    [name: "X-Cmp-Timestamp", value: timestamp],
    [name: "X-Cmp-ClientType", value: clientType],
    [name: "X-Cmp-ProjectId", value: projectId, maskValue: true]
]
```

## 7.3 로그 출력 제한

권장:

```
consoleLogResponseBody: false
quiet: true
```

API 실패 분석이 필요하면 status와 제한된 메시지만 출력합니다.
## 7.3 API Key 운영 정책

Samsung Cloud Platform OpenAPI 문서에는 프로젝트별 인증키, HMAC SHA256, IP access control 개념이 설명되어 있습니다.  
운영 권장:

```text
1. 개발/운영 AccessKey 분리
2. purge 전용 최소권한 Key 사용
3. Jenkins Folder 단위 Credential 권한 제한
4. Jenkins Agent outbound IP를 OpenAPI 허용 IP로 제한
5. SecretKey 주기적 교체
6. Job 실행 권한을 운영자/배포관리자에게만 부여
```

## 8. 개선 Jenkinsfile 예시

아래는 현재 스크립트를 운영 purge에 맞게 개선한 예시입니다.

```groovy
pipeline {
    agent { label 'linux-deploy' }

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    parameters {
        string(name: 'CUSTOM_URL', defaultValue: '/js/app.js', description: 'Purge 대상 URL path. 전체 / 사용 주의')
        booleanParam(name: 'APPROVE_DOMAIN_PURGE', defaultValue: false, description: 'customUrl=/ 전체 purge 승인')
    }

    environment {
        METHOD = "POST"
        CDN_PURGE_URL = "https://g-openapi.samsungsdscloud.com/cdn~~~~~~~~~~~~~~~~~~~~/purge"
        CLIENT_TYPE = "OpenApi"
        TARGET_CONTENT = "INVALIDATED_CONTENT"
        TARGET_URL = "DOMAIN"
    }

    stages {
        stage('Validate') {
            steps {
                script {
                    if (params.CUSTOM_URL == "/" && !params.APPROVE_DOMAIN_PURGE) {
                        error "전체 purge('/')는 APPROVE_DOMAIN_PURGE=true일 때만 허용합니다."
                    }
                }
            }
        }

        stage('Make Signature And Purge') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'accessKey', variable: 'ACCESS_KEY'),
                        string(credentialsId: 'secretKey', variable: 'SECRET_KEY'),
                        string(credentialsId: 'projectId', variable: 'PROJECT_ID')
                    ]) {
                        def timestamp = sh(script: 'date +%s%3N', returnStdout: true).trim()

                        def requestBody = groovy.json.JsonOutput.toJson([
                            customUrl    : params.CUSTOM_URL,
                            targetContent: env.TARGET_CONTENT,
                            targetUrl    : env.TARGET_URL
                        ])

                        def message = "${env.METHOD}${env.CDN_PURGE_URL}${timestamp}${ACCESS_KEY}${PROJECT_ID}${env.CLIENT_TYPE}${requestBody}"

                        withEnv([
                            "MESSAGE=${message}",
                            "SECRET_KEY=${SECRET_KEY}"
                        ]) {
                            env.SIGNATURE = sh(
                                script: '''
                                    set +x
                                    node -e "const crypto=require('crypto'); console.log(crypto.createHmac('sha256', process.env.SECRET_KEY).update(process.env.MESSAGE, 'utf8').digest('base64'));"
                                ''',
                                returnStdout: true
                            ).trim()
                        }

                        def response = httpRequest(
                            acceptType: 'APPLICATION_JSON',
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            url: env.CDN_PURGE_URL,
                            requestBody: requestBody,
                            timeout: 30,
                            validResponseCodes: '200:299',
                            consoleLogResponseBody: false,
                            quiet: true,
                            customHeaders: [
                                [name: "X-Cmp-AccessKey", value: ACCESS_KEY, maskValue: true],
                                [name: "X-Cmp-Signature", value: env.SIGNATURE, maskValue: true],
                                [name: "X-Cmp-Timestamp", value: timestamp],
                                [name: "X-Cmp-ClientType", value: env.CLIENT_TYPE],
                                [name: "X-Cmp-ProjectId", value: PROJECT_ID, maskValue: true]
                            ]
                        )

                        echo "CDN purge request completed. status=${response.status}, customUrl=${params.CUSTOM_URL}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "CDN purge success"
        }
        failure {
            echo "CDN purge failed"
        }
    }
}
```

### 개선 포인트

|항목|기존|개선|
|---|---|---|
|Agent|`any`|`linux-deploy` 고정|
|npm|매번 `npm install crypto-js`|Node 내장 `crypto` 사용|
|timestamp|전역 environment에서 생성|요청 직전 생성|
|requestBody|Signature 미포함|Signature에 포함|
|전체 purge|항상 `/` 가능|승인 파라미터 필요|
|Header masking|없음|`maskValue: true`|
|timeout|없음|`timeout: 30`|
|응답 검증|없음|`validResponseCodes: '200:299'`|
|동시 실행|가능|`disableConcurrentBuilds()`|

## 9. 현재 추세

CDN 운영의 최근 추세는 **purge를 자주 실행하는 구조보다 purge 필요성을 줄이는 구조**입니다.

```text
1. JS/CSS 파일명에 content hash 적용
2. Cache-Control: public, max-age=31536000, immutable 적용
3. HTML만 짧은 TTL 또는 no-cache/revalidate
4. 정적 파일은 변경 시 새 URL로 배포
5. purge는 HTML, sitemap, 긴급 롤백, 보안 사고 시 제한적으로 사용
6. 전체 purge보다 URL 단위 purge
7. purge 후 preload/warm-up 자동화
```

정적 리소스가 버전/hash 기반 URL이면 장기 캐시와 `immutable`을 적용할 수 있으며, MDN은 `max-age`, `immutable` 같은 Cache-Control 지시어를 설명합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/http_request/ "HTTP Request Plugin"))

최근 CDN 운영에서는 **배포 때마다 전체 purge를 반복하는 방식보다, 정적 자산의 파일명에 content hash를 붙이고 장기 캐시를 적용하는 방식**이 더 일반적입니다.  
권장 구조:

```
app.js→ app.8f3a91c2.js
```

응답 헤더:

```
Cache-Control: public, max-age=31536000, immutable
```

이 방식은 CDN purge 의존도를 낮추고, 브라우저와 CDN 캐시 효율을 높입니다. MDN과 web.dev 모두 versioned/static asset에 장기 `max-age`와 `immutable` 또는 public cache 전략을 설명하고 있습니다.  
실무 추세:

```
1. 전체 purge 최소화
2. URL 단위 purge
3. content-hash 파일명 사용
4. long TTL + immutable
5. 배포 자동화에서 purge/preload 분리
6. API Key는 Vault/Jenkins Credential/Cloud Secret Manager로 관리
7. purge Job은 승인 기반 수동 실행 또는 배포 파이프라인의 제한 단계로 운영
8. purge 후 synthetic check와 CDN HIT ratio 확인 자동화
```


## 10. 최종 판단

이 버전은 이전 스크립트의 `AccessYek` 오타는 수정되어 **인증 헤더명은 정상 방향**입니다. 다만 운영 purge용으로는 아직 아래 문제가 큽니다.

```text
1. POST requestBody가 Signature 대상에서 빠져 있을 가능성
2. timestamp가 너무 이른 시점에 생성됨
3. npm install crypto-js를 매번 수행함
4. secretKey가 Groovy interpolation으로 shell command에 직접 들어감
5. customUrl="/" 전체 purge 위험
6. httpRequest timeout/validResponseCodes/maskValue 미설정
7. purge 결과 response.status 검증 부족
```

따라서 운영 적용 전에는 **requestBody 포함 Signature 검증**, **전체 purge 방지 로직**, **secret masking/전달 방식 개선**, **URL 단위 purge 전환**을 우선 반영하는 것이 안전합니다.


## Purge 운영 체크리스트

```
1. 이 Job이 운영 CDN에 연결되는지 개발 CDN에 연결되는지 명확한가?
2. Access Key / Secret Key / Project ID가 운영 전용인지 확인했는가?
3. Jenkins Agent outbound IP가 SCP OpenAPI IP 허용 목록에 등록되어 있는가?
4. 전체 purge('/')를 실수로 실행하지 못하게 막았는가?
5. purge 대상 URL이 query string 포함 여부까지 정확한가?
6. CDN Cache Key가 query string을 포함하는지 확인했는가?
7. purge 후 HIT/MISS/TTFB 변화를 모니터링하는가?
8. purge 후 주요 정적 파일 preload를 수행하는가?
9. 동일 purge job 동시 실행이 차단되어 있는가?
10. API 실패 시 재시도 정책이 과도하지 않은가?
11. 응답 로그에 Secret, Signature, Access Key가 출력되지 않는가?
12. Secret Key 교체 절차가 있는가?
```