---
layout: single
title: "[IDE] Eclipse에서 PMD와 SpotBugs 사용하기"
excerpt: "개발환경 - 소스품질관리 #1"

categories:
  - tech
tags:
  - [tech, PMD, Spotbugs]

toc: false
toc_sticky: true

date: 2022-12-15
last_modified_at: 2022-12-15
---
# Eclipse에서 PMD와 SpotBugs 사용하기

## 1. PMD, SpotBugs 플러그인 설치
- Eclipse >> Help >> MarketPlace 에서 pmd-eclise-plugin과 SpotBugs Eclise-plugin 을 다운받고 재시작

![sts_plugin01](./../../images/tech/inst_pmd_01.png)

## 2. PMD 룰셋 적용하기

![sts_plugin02](./../../images/tech/inst_pmd_02.png)

- Language 3항목(Ecmascript, HTML, java) 룰셋 적용

## 3. 룰셋 Export (프로젝트 루트에 .ruleset으로 저장되면 xml임)

- 룰셋 export
- daiso_pmd_rulsset_221215.xml

## 4. 적용 후 코드 분석하기

![sts_plugin03](./../../images/tech/inst_pmd_03.png)

- PMD Check Code 실행

## 5. 해당프로젝트의 룰셋에 의해 식별된 항목 토글로 필터링 분석하기

![sts_plugin04](./../../images/tech/inst_pmd_04.png)

- 패키지 클래스별 레벨별 항목 분석

## 6. 아래 링크에서 SpotBugs(findsecbugs-plugin-1.10.1.jar) 파일 받아 external jar 등록

https://find-sec-bugs.github.io/download.htm
 
- Download - Find Security Bugs

![sts_plugin05](./../../images/tech/inst_pmd_05.png)

- 프로젝트 빌트 패스에 등록

![sts_plugin06](./../../images/tech/inst_pmd_06.png)

- external jar 등록

![sts_plugin07](./../../images/tech/inst_pmd_07.png)

- SpotBugs 실행하기

![sts_plugin08](./../../images/tech/inst_pmd_08.png)

- Bug Explorer 확인

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조


  </pre>
</details>