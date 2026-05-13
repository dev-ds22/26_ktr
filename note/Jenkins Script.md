## 핵심 차이
`agent any` 는 **파이프라인(또는 해당 stage)을 실행할 agent를 하나 할당해서 그 위에서 작업하도록 하는 설정**이고, `agent none` 는 **파이프라인 전체에 대해 공통 agent를 미리 잡지 않는 설정**입니다. 특히 `agent none` 을 top-level `pipeline` 에 두면 **각 stage마다 반드시 별도의 `agent` 를 선언해야 합니다.** :contentReference[oaicite:0]{index=0}
	
| 항목                 | `agent any`           | `agent none`               |
| ------------------ | --------------------- | -------------------------- |
| 의미                 | 사용 가능한 아무 agent에서 실행  | 전역 agent를 할당하지 않음          |
| top-level 사용 시     | 파이프라인 전체 실행용 agent 확보 | 공통 agent 없음                |
| stage별 agent 선언    | 선택                    | 사실상 필수                     |
| workspace/executor | 바로 확보됨                | 해당 stage에서 agent 지정 전까지 없음 |
| 주 용도               | 단순 빌드/배포 파이프라인        | stage별 서로 다른 실행 환경 분리      |

## `agent any`
가장 단순한 방식입니다. Jenkins 공식 문서상 `any` 는 **사용 가능한 아무 agent에서 Pipeline 또는 stage를 실행**합니다. 즉, 별도 label 지정 없이 Jenkins가 실행 가능한 agent를 잡습니다. :contentReference[oaicite:1]{index=1}

### 예시
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo '빌드 실행'
            }
        }
        stage('Test') {
            steps {
                echo '테스트 실행'
            }
        }
    }
}
````

### 해석

- Pipeline 시작 시 Jenkins가 실행 가능한 agent 하나를 잡습니다.
    
- 보통 그 agent의 workspace에서 stage들이 이어서 실행됩니다.
    
- 구조가 단순해서 **테스트용 Job**, **단일 서버 빌드**, **간단한 배포 파이프라인**에 적합합니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com "Pipeline Syntax"))
    

## `agent none`

`agent none` 을 top-level 에 두면 Jenkins는 **파이프라인 전체용 executor/agent를 불필요하게 먼저 잡지 않습니다.** 공식 문서도 이것이 **불필요한 executor 할당을 피하게 해 준다**고 설명합니다. 대신 각 stage에 `agent` 가 있어야 실제 step을 실행할 수 있습니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com "Pipeline Syntax"))

### 예시

```groovy
pipeline {
    agent none
    stages {
        stage('Build') {
            agent any
            steps {
                echo '빌드 실행'
            }
        }
        stage('Test') {
            agent { label 'linux' }
            steps {
                echo '리눅스 테스트 실행'
            }
        }
    }
}
```

### 해석

- Pipeline 전체는 agent를 점유하지 않습니다.
    
- `Build` stage가 시작될 때만 agent를 잡습니다.
    
- `Test` stage는 `linux` label이 붙은 agent에서 따로 실행됩니다.
    
- 따라서 **stage별로 서로 다른 node/docker/container 환경을 써야 할 때** 매우 유용합니다. Jenkins 공식 예제도 `agent none` + stage별 다른 agent 조합을 보여줍니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com "Pipeline Syntax"))
    

## 실무 기준으로 언제 무엇을 쓰는가

|상황|권장|
|---|---|
|빌드/배포가 한 서버에서 단순하게 끝남|`agent any`|
|빌드는 Linux, 테스트는 Windows, 배포는 별도 node|`agent none`|
|불필요하게 executor 오래 점유하면 안 됨|`agent none`|
|Jenkins 학습/간단 테스트용 Job|`agent any`|

## 주의할 점

`agent none` 상태에서는 workspace나 node context가 필요한 step을 바로 실행할 수 없습니다. Jenkins step 문서도 이런 step들은 **node context**, 즉 Declarative에서는 **agent가 정의된 stage** 안에서 실행되어야 한다고 설명합니다. 예를 들어 `fileExists` 같은 step은 `agent none` 상태의 전역 영역에서 바로 쓸 수 없습니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/workflow-basic-steps/?utm_source=chatgpt.com "Pipeline: Basic Steps"))

## 한 줄 정리

- `agent any` = **아무 agent 하나 잡고 전체 또는 해당 stage 실행**
    
- `agent none` = **전역 agent는 잡지 않고, stage마다 필요한 agent를 따로 지정**
    

실무에서는 **단순하면 `agent any`**, **환경 분리·리소스 최적화가 필요하면 `agent none`** 으로 보면 거의 맞습니다. ([Jenkins](https://www.jenkins.io/doc/book/pipeline/syntax/?utm_source=chatgpt.com "Pipeline Syntax"))
