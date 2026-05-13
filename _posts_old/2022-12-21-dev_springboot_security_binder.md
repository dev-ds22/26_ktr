---
layout: single
title: "[SECURE] Spring Java 프레임워크 보안 업데이트 권고"
excerpt: "보안 업데이트 권고 (CVE-2022-22965, CVE-2022-22963)"

categories:
  - tech
tags:
  - [tech, spring boot, logback]

toc: false
toc_sticky: true

date: 2022-12-21
last_modified_at: 2022-12-21
---
# Spring Java 프레임워크 보안 업데이트 권고

## 개요

- Spring 보안팀에서 Spring 프레임워크 및 Spring Cloud Function 관련 원격코드 실행 취약점을 해결한 임시조치 방안 및 보안업데이트 권고
- 공격자는 해당 취약점을 이용하여 정상 서비스에 피해를 발생시킬 수 있으므로, 최신 버전으로 업데이트 권고
 
## 주요 내용

- Spring Core에서 발생하는 원격코드실행 취약점(CVE-2022-22965)[1]
- Spring Cloud Function에서 발생하는 원격코드실행 취약점 (CVE-2022-22963)[2]
 
## 영향을 받는 버전

- CVE-2022-22965(Spring4Shell)
  - 1) JDK 9 이상의 2) Spring 프레임워크 사용하는 경우
  - Spring Framework 5.3.0 ~ 5.3.17, 5.2.0 ~ 5.2.19 및 이전 버전
  - ※ JDK 8 이하의 경우 취약점의 영향을 받지 않음

- CVE-2022-22963
  - Spring Cloud Function 3.1.6 ~ 3.2.2 버전
  - ※ 취약점이 해결된 버전 제외(3.1.7, 3.2.3 업데이트 버전 제외)
 
## Spring4Shell 버전 확인 방법

- JDK 버전 확인
  - “java -version” 명령 입력

- Spring 프레임워크 사용 유무 확인
  - 프로젝트가 jar, war 패키지로 돼 있는 경우 zip 확장자로 변경하여 압축풀기
  - 이후 아래와 같이 “spring-beans-.jar”, “spring.jar”, “CachedIntrospectionResuLts.class” 로 검색
    - find . -name spring-beans*.jar 

## 대응방안

- 제조사 홈페이지를 통해 최신버전으로 업데이트 적용
- ※ 제조사 홈페이지에 신규버전이 계속 업데이트되고 있어 확인 후 업데이트 적용 필요
- CVE-2022-22965(Spring4Shell)
  - Spring Framework 5.3.18, 5.2.20 버전으로 업데이트[4]

- CVE-2022-22963
  - Spring Cloud Function 3.1.7, 3.2.3 버전으로 업데이트[3]

- 신규 업데이트가 불가능할 경우 아래와 같이 조치 적용
  - CVE-2022-22965(Spring4Shell)
    - 프로젝트 패키지 아래 해당 전역 클래스 생성 후 재컴파일(테스트 필요)

```java
  import org.springwork.core.Ordered;
  import org.springwork.core.annotation.Order;
  import org.springwork.web.bind.WebDataBinder;
  import org.springwork.web.bind.annotation.ControllerAdvice;
  import org.springwork.web.bind.annotation.InitBinder;
  
  @ControllerAdvice
  @Order(10000)
  public class BinderControllerAdvice {

    @InitBinder
    public setAllowedFields(WebDataBinder dataBinder) {
      String[] denylist = new String[]{"class.*", "Class.*", "*.class.*", "*.Class.*"};
      dataBinder.setDisallowedFields(denylist);
    }
  }
```
## 기타 문의사항

- 한국인터넷진흥원 사이버민원센터: 국번없이 118
 
### 참고사이트

- [1] 취약점 정보 : https://tanzu.vmware.com/security/cve-2022-22963
- [2] 신규버전 다운로드 : https://repo.maven.apache.org/maven2/org/springframework/cloud/spring-cloud-function-context/
- [3] 취약점 업데이트 정보 : https://spring.io/blog/2022/03/31/spring-framework-rce-early-announcement
- [4] 제조사별 현황 : https://github.com/NCSC-NL/spring4shell/tree/main/software



<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- BinderControllerAdvice.java

```java
  @ControllerAdvice
  @Order(Ordered.LOWEST_PRECEDENCE)
  public class BinderControllerAdvice {

      @InitBinder
      public void setAllowedFields(WebDataBinder dataBinder) {
          String[] denylist = new String[]{"class.*", "Class.*", "*.class.*", "*.Class.*"};
          dataBinder.setDisallowedFields(denylist);
      }
  }
```

  </pre>
</details>