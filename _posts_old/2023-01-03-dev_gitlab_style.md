---
layout: single
title: "[GIT] GitLab Style Guide"
excerpt: "GitLab & Git 스타일 가이드 (GitLab Style Guide)"

categories:
  - tech
tags:
  - [gitlab, git style]

toc: false
toc_sticky: true

date: 2023-01-03
last_modified_at: 2023-01-03
---
# GitLab & Git 스타일 가이드 (GitLab Style Guide)

- GitLab과 Git 사용 시에 네이밍과 문장 스타일에 대한 가이드문서.

## 1. Group name

### 1.1. 규칙

- 단어(알파벳 소문자)와 구분자(-) 간략하게 구성
- {발주기관에 대한 약어}-{프로젝트 또는 시스템 약어}-{기타 약어} 3단어를 초과하지 않는다.
- 발주기관에 대한 약어는 다음과 같다.
  - LG유플러스: uplus
  - kt: kt
  - SK텔레콤: skt
  - 우리생협: wooricoop
  - SK C&C: skcc
  - 국립공원관리공단: knps
  - 지어소프트: gaeasoft
  - 사내 프로젝트의 경우: internal
- 프로젝트 또는 시스템 약어
  - 고객상황인지: cca
  - 탐방객이동경로분석시스템: vras
  - 미디어컨텐츠관리시스템: mecs
- 기타 약어
  - 앞의 두 단어까지 같다면 구분할 수 있는 약어를 추가해서 Group을 구분한다.

### 1.2. 샘플

> LG유플러스 고객상황인지시스템 프로젝트 그룹명: uplus-cca  
> 국립공원관리공단 탐방객이동경로분석시스템 프로젝트 그룹명: knps-vras  

## 2. Project name

### 2.1. 규칙

- 단어(알파벳 소문자)와 구분자(-) 간략하게 구성
- prefix로는 프로젝트 또는 시스템 약어를 사용하고 가능하면 5단어를 초과하지 않는다.

### 2.2. 샘플

> 고객상황인지 지오펜싱 클라이언트 서비스: sico-zone-client-service  
> 고객상황인지 유틸 라이브러리: sico-util  

## 3. Commit message

### 3.1. 규칙

- 타이틀 + 빈줄 + 적용항목으로 구성한다.
- 타이틀은 커밋메시지를 요약해서 한줄로 작성한다.
- 이슈와 연관된 커밋인 경우 closed #이슈번호를 타이틀 맨 앞에 표기한다.
- 적용항목은 -를 앞에 붙여서 리스트 형식으로 작성한다.

### 3.2. 샘플

> closed #1, Android 6.0 대응 예외처리  
>   - Andorid 6.0 타겟 빌드  
>   - Android 6.0 런타임 권한 모델 정책에 따라 권한이 Off된 API 접근 시 앱이 종료되는 증상에 대한 예외처리  
>   - 폰번호가 null이거나 빈 값일 경우에 서버 연동하지 않도록 처리, 업데이트 알람 설정하지 않음  

## 4. Merge request message

### 4.1. 규칙 (요청 시)

- 병합 요청 시에 Title과 Description은 커밋 메시지의 타이틀과 적용항목을 적용한다.
- Title에는 커밋 메시지의 타이틀을 입력한다.
- Description에는 커밋 메시지의 적용항목을 적용한다.
- 병합 요청 시에 커밋이 여러 개인 경우에는 Title은 주요 커밋 중에 하나를 선택하고 Description은 적용항목을 모두 적어주면 된다.

### 4.2. 규칙 (승인 시)

- master 브랜치는 커밋 이력을 깔끔하게 관리해야 한다.
- 승인자가 병합 시에는 커밋 메시지를 확인하고 수정이 필요한 부분은 내용을 수정한 후에 병합을 진행한다.

## 5. Git branch name

### 5.1. 규칙

- Git 브랜치명은 브랜치 용도에 따라 master, develop, feature, release, hotfix 브랜치별로 네이밍 규칙을 따른다.
- feature, release, hotfix 브랜치는 아래와 같이 접두어를 붙여준다. feature는 기능명을 표현하는 단어로 단어(알파벳 소문자)와 구분자(-)를 사용하고 release, hotfix에는 버전명을 사용한다.
- 브랜치별로 다음과 같은 규칙을 사용한다.

- Production branch name
  - master
- Development branch name
  - develop
- Feature 브랜치 접두어
  - feature/
- Release 브랜치 접두어
  - release/
- Hotfix 브랜치 접두어
  - hotfix/

### 5.2. 샘플

> Feature branch  
>  - feature/this-is-awesome  
> Release branch  
>  - release/v1.0.0  
> Hotfix branch  
>  - hotfix/v1.1.1  

## 6. Tag Name

### 6.1. 규칙

- 제품을 릴리즈 하는 시점에 해당 commit에 Tag를 만든다.
- 네이밍은 제품 버전명과 동일하게 사용한다. 예를 들어 안드로이드의 경우에는 매니패스트에 android:versionName을 사용하면 된다.
- 혹시 버전명이 없다면 릴리즈한 일자(YYYYMMDD)을 표기해도 된다.

### 6.2. 샘플

> 버전 1.1.2 Tag Name: v1.1.1 2015년 10월 25일에 반영한 버전의 Tag Name: 20151025


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>