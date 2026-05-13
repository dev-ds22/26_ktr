---
layout: single
title: "[GIT] GitLab 기반 Workflow 가이드"
excerpt: "GitLab 기반시스템을 이용한 Workflow 전반"

categories:
  - tech
tags:
  - [gitlab, workflow]

toc: false
toc_sticky: true

date: 2023-01-03
last_modified_at: 2023-01-03
---
# GitLab 기반시스템을 이용한 Workflow 가이드

프로젝트 수행에 필요한 업무를 GitLab 기반시스템을 통해 진행하는 절차를 설명.

## 1. 팀을 구성하고 업무를 배정

![GitLab 프로젝트 구성](./../../images/tech/gitlab-group.png)

GitLab을 이용하여 팀을 구성(Group, Member, Permission)하고 업무를 계획(Milestone)하고 업무 세분화하여 팀원에게 배정(Issue)하는 단계로 프로세스는 다음과 같다.

![GitLab 프로젝트 업무흐름](./../../images/tech/gitlab-workflow-main.png)

- Group 등록
- Member 등록 및 Permission(접근권한) 설정
- Project 등록
- Milestone 등록
- Issue 등록 및 Assign
- 업무진행 및 Issue 종료

수행 주체는 프로젝트 매니저(PM)이다. 상황에 따라 PL에게 권한을 이관할 수 있다.

### 1.1. Group 등록

프로젝트를 수행하는 팀 단위로 Group을 만든다. 유지보수를 수행하는 팀이라면 운영하는 시스템 단위로 Group을 생성한다.

![Create Group](./../../images/tech/gitlab-workflow-001.png)

### 1.2. Member 등록 및 Permission 설정

프로젝트 팀원을 추가하고 팀원별로 Permission을 설정한다.
Permission은 Project Action과 Group Action에 대한 권한을 말한다.

- Owner 
  - Group 등록한 팀원으로 모든 권한을 가지고 있다. 
  - PM이 이에 해당한다.
- Master 
  - Group 수정/삭제와 Project 이동/삭제를 제외하고 Owner와 같은 권한을 가지고 있다.
  - 업무를 할당하고 소스코드에 대한 Merge Reqeust를 승인하는 팀원으로 PL이 이에 해당한다.
- Developer
  - 프로젝트를 수행하는 팀원이 이에 해당한다.
  - 새로운 브랜치를 만들고 push가 가능하지만 protected 브랜치에는 push가 불가능하다.
  - 기본적으로 master 브랜치가 protected 브랜치로 생성되는데 Developer는 master 브랜치로 push가 불가하다.
  - master 브랜치로 Merge하기 위해서는 Merge Reqeust 작성해서 Asignee의 승인이 필요하다.
- Reporter
  - Project 조회가 가능하고 Issue와 Label에 대한 관리도 가능하다.
  - QA나 빌드/배포 팀원이 이에 해당한다.
- Guest
  - Issue 등록과 comment 남기는 것, Group & Project 조회 가능하다.

![Add Member](./../../images/tech/gitlab-workflow-002.png)

### 1.3. Project 등록

소스코드 관리 단위로 Project를 만든다. 소스코드 관리 단위라는 것은 IDE(통합개발도구)에서 git을 통해 Project를 checkout해서 개발이 가능한 형태를 얘기한다.

예를 들어, 쇼핑몰을 구축하는 프로젝트이고 안드로드와 iOS 앱을 개발해야 한다면 Project는 최소 4개(서버 웹어플리케이션과 안드로이드 App, iOS App, 프로젝트산출물 관리용)를 등록한다.
 
![Create Project](./../../images/tech/gitlab-workflow-010.png)

### 1.4. Milestone 등록

Milestone은 릴리즈 단위나 프로젝트 단계별 단위로 만든다. 업무 목표와 완료일을 작성하고 Milestone별로 Issue를 등록하여 Issue와 Merge Reqeust의 완료여부에 따라 업무의 진척도를 모니터링할 수 있다.

예를 들어, 전통적인 프로젝트 진행 방식이라면 요구분석, 설계, 구현, 검수, 상용화 단계별로 Milestone을 등록하고, 애자일 방식의 경우는 Sprint 단위로 등록을 한다. 유지보수의 경우에는 릴리즈 단위로 Milestone을 등록한다.

![Create Milestone](./../../images/tech/gitlab-workflow-011.png)

### 1.5. Label 등록

Label은 Issues와 Merge Requests 등록 시에 태그로 사용한다. 태그는 업무를 구분하는 카테고리 성격으로 보면 된다. 최초 Label 생성 시에는 기본 셋으로 선택해서 만들 수 있다.

기본적으로 기본 셋을 생성해서 사용하고 필요한 경우에 Label을 추가/수정/삭제한다.

![Generate default set](./../../images/tech/gitlab-workflow-012.png)

![Default Labels](./../../images/tech/gitlab-workflow-013.png)

### 1.6. Issue 등록 및 할당

업무 단위로 Issue를 등록하고 작업내용, 작업자, Milestone, Label 을 설정한다.

![Create Issue](./../../images/tech/gitlab-workflow-014.png)

## 2. 개발업무절차 (workflow)

PM이 배정한 업무를 팀원이 Issue를 통해서 할당 받으면, 업무 결과인 코드 또는 문서를 git 저장소에 올리고 해당 Issue를 종료하는 절차이다.

git 저장소를 활용하는 workflow 모델은 매우 다양하기 때문에 프로젝트 성격에 따라서 적합한 workflow 방식을 설계할 필요가 있다. 그리고 git이 익숙하지 않는 팀에서 복잡한 방식을 먼저 도입하게 된다면 오히려 독이 될 수 있으니 간단한 방법부터 적용하고, 사용하면서 보완이 필요한 부분들을 개선해 나가는 것이 좋다.

여기서는 master-develop 브랜치 workflow 모델과 Feature 브랜치 workflow 모델에 대해서 설명한다. 전자는 신규 프로젝트에 적합한 모델이고 후자는 기존 시스템을 유지보수하는 프로젝트에 적합하다.

이 외에, 팀에 맞는 workflow 모델을 설계해서 사용해 보길 바라고 적용 사례를 공유하길 바란다. 하지만, 아래 2.1. 브랜치 규칙에 대해서는 필히 준수해서 사용하길 바란다.

### 2.1. 브랜치 규칙

기본적인 브랜치를 관리하는 규칙으로 업무 프로세스를 설계하는 경우에 필히 준수해야할 사항이다.

- master 브랜치는 protected 브랜치이다.
- protected 브랜치는 Master 이상 권한만 push가 가능하다.
- Developer 권한 사용자는 master 브랜치에서 신규 브랜치를 추가한다.
- 신규 브랜치에서 소스를 commit하고 push 한다.
- Merge request를 생성하여 master 브랜치로 Merge 요청을 한다.
- Master 권한 사용자는 Developer 사용자와 함께 리뷰 진행 후 master 브랜치로 Merge 한다.

### 2.2. Feature 브랜치 workflow

![feature workflow](./../../images/tech/gitlab-workflow-014.png)

Feature 브랜치 workflow 모델은 전통적인 git-flow 모델의 복잡함을 간편화한 모델로 메인 브랜치인 master 브랜치와 다수의 feature 브랜치로 구성된다.

Issue가 할당되면 master 브랜치에서 feature 브랜치를 생성하고 생성한 브랜치에서 작업한 후에 commit & push 하고 master 브랜치로 Merge 하는 방식이다.

기본 절차는 다음과 같다.

- Step 1. (Developer) Clone Project (최초 Project 체크아웃 시에):

```bash
  git clone git@example.com:project-name.git
```

- Step 2. (Developer) feature 브랜치 생성 및 체크아웃:

```bash
  git checkout -b $feature_name
```

- Step 3. (Developer) 코드 작성 후 코드 commit:

```bash
  git commit -am "My feature is ready"
```

- Step 4. (Developer) 코드를 GitLab으로 push:

```bash
  git push origin $feature_name
```

- Step 5. (Developer) Commits 페이지에서 자체 코드 리뷰. 동료 리뷰가 필요한 경우에는 팀 동료에게 함께 리뷰해 달라고 한다.

- Step 6. (Developer) Merge request 생성.
- Step 7. (Master) Developer와 함께 코드 리뷰를 하고 master 브랜치로 Merge 한다.
- Step 8. (Master) issue를 close(완료) 한다.
- Step 9. (Master) 제품이 Release(상용반영)가 되면 해당 commit 시점으로 Tag를 생성한다.

> 참고문서 http://doc.gitlab.com/ce/workflow/workflow.html  

### 2.3. master-develop workflow

master-develop 브랜치 workflow 모델은 svn과 같은 중앙집중식 모델에서 코드리뷰 절차를 추가하기 위해 master 브랜치에 develop 브랜치를 추가한 구성이다.

Developer는 develop 브랜치에서 작성한 후에 commit & push 하고 master 브랜치로 Merge 하는 방식이다.

- Step 1. Clone Project (최초 Project 체크아웃 시에):

```bash
  git clone git@example.com:project-name.git
```

- Step 2. feature 브랜치 생성 및 체크아웃:

```bash
  git checkout -b develop origin/develop
```

- Step 3. 코드 작성 후 코드 commit:

```bash
  git commit -am "develop is ready"
```

- Step 4. 코드를 GitLab으로 push:

```bash
  git push origin develop
```

- Step 5. Commits 페이지에서 자체 코드 리뷰. 동료 리뷰가 필요한 경우에는 팀 동료에게 함께 리뷰해 달라고 한다.
- Step 6. Merge request 생성.
- Step 7. Master는 Developer와 함께 코드 리뷰를 하고 master 브랜치로 Merge 한다.
- Step 8. issue를 close(완료) 한다.

### 2.4. 로컬에서 Merge하는 방법

GitLab에서는 Merge Request를 승인하면 GitLab에서 알아서 Merge를 진행한다. 하지만 로컬에서 Master가 직접 Merge를 하고 master 브랜치에 push하는 방법도 있다. GitLab에서 Merge 시에 conflict가 발생한 경우에 이 방법을 사용한다.

로컬로 Check out하고 Review와 Merge를 하는 방법이다.

- Step 1. Fetch and check out the branch for this merge request

```bash
  git fetch origin
  git checkout -b this-is-new-feature origin/this-is-new-feature
```

- Step 2. 변경사항 리뷰

- Step 3. conflict를 해결하고 master에 Merge

```bash
  git checkout master
  git merge --no-ff this-is-new-feature
```

- Step 4. master 브랜치를 GitLab에 push

```bash
  git push origin master
```


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>