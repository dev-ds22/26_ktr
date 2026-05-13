---
layout: single
title: "[Tech Stack] ETC - 공개 SW를 활용한 코드품질관리"
excerpt: "오픈소스를 통한 코드품질관리"

categories:
  - tech
tags:
  - [tech]

toc: false
toc_sticky: true

date: 2022-11-30
last_modified_at: 2022-11-30
---
# 공개 SW를 활용한 소프트웨어 개발보안 점검가이드(2019.6)'
- 오픈소스 :소스코드가 공개되어 있는 소프트웨어로 누구나 자유롭게 수정 및 이용이 가능
- 코드품질관리 및 시큐어코딩을 위해 Spotbugs, FindSecurityBugs, PMD, Jenkins 사용 


## 1. Spotbugs
- 자바 바이트 코드를 분석해서 버그 패턴을 발견하는 정적분석 공개소프트웨어. 
- 자바 프로그램에서 발생 가능한 100여개의 잠재적인 에러에 대해 scariest scary, trobuling, concern의 4등급으로 구분하고 그 결과를 XML으로 저장.  
- [다운로드](https://spotbugs.github.io/)

## 2. FindSecurityBugs
- 자바 웹 어플리케이션에 대한 보안 감사를 지원하는 Spotbugs의 플러그인
- 200개 이상의 시그니처를 활용하여 OWASP TOP 10과 CWE를 커버하는 78개의 버그패턴(Bug Pattern)탐지.  
- [다운로드](http://find-sec-bugs.github.io/)

## 3. PMD 
- 자바 프로그램의 소스코드를 분석하여 프로그램의 부적절한 부분을 찾아내고 성능을 높이도록 도와주는 도구
- 시스템 개발공정의 구현 및 테스트 단계에서 정적분석 활용가능.
- [내용 확인](https://pmd.github.io/pmd-6.9.0/pmd_rules_java.html)

## 4. Jenkis
- CI(Continuous Integration) 도구
- 보안 취약점을 진단하고 결과를 보고하는 용도로 사용가능. 

## 구성도 

![ELK Stack](./../../images/tech/code_quality01.png)

- 1~4 부터 개발자는 IDE에 플러그인 된 도구를 활용하여 취약점을 진단하고 서버에 반영
- 4~8 Jenkins 서버는 주기적으로 플러그인 된 도구를 통해 정적분석(코드를 실행시키지 않는 분석)을 실시 후 분석된 결과를 보고 
- 9 결과를 담당자에게 전달

- 위 내용은 '공개 SW를 활용한 소프트웨어 개발보안 점검가이드(2019.6)'에서 확인 가능. 

https://www.mois.go.kr/frt/bbs/type001/commonSelectBoardArticle.do?bbsId=BBSMSTR_000000000012&nttId=73814 

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조


  </pre>
</details>