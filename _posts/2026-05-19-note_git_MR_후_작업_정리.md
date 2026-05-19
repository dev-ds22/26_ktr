---
layout: single
title: "git_MR_후_작업_정리"
excerpt: "git_MR_후_작업_정리"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-05-19"
last_modified_at: "2026-05-19 18:05:41 +0900"
---
## MR 후 작업

GitLab에서 `feature/aaa → dev` Merge Request 승인/머지 시 **`Delete source branch`를 선택하면 `origin/feature/aaa`는 자동 삭제 대상**입니다. 단, 삭제되는 것은 **source branch인 `feature/aaa`**이고, **target branch인 `dev`는 삭제되지 않습니다.** GitLab 공식 문서도 MR 생성 또는 머지 시 `Delete source branch`를 선택하면 source branch를 삭제할 수 있으며, 실제 삭제는 머지한 사용자 또는 auto-merge 설정 사용자의 권한에 따라 수행된다고 설명합니다. ([GitLab Docs](https://docs.gitlab.com/user/project/merge_requests/ "Merge requests | GitLab Docs"))  
로컬 PC의 `feature/aaa`는 GitLab이 자동으로 지우지 않습니다. 원격 branch가 삭제되어도 내 PC의 local branch는 그대로 남아 있으므로 직접 정리해야 합니다.

## 2. 용어 정리

| 구분            | 예시                   | 삭제 여부                                  |
| ------------- | -------------------- | -------------------------------------- |
| Source branch | `feature/aaa`        | `Delete source branch` 선택 시 원격에서 삭제 가능 |
| Target branch | `dev`                | MR 대상 branch이며 삭제되지 않음                 |
| Remote branch | `origin/feature/aaa` | GitLab 서버의 branch                      |
| Local branch  | 내 PC의 `feature/aaa`  | 자동 삭제 안 됨                              |

- GitLab 문서의 “Delete source branch”는 MR의 source branch 삭제를 의미합니다. `feature/aaa → dev` MR에서는 `feature/aaa`가 source branch이고 `dev`가 target branch입니다. ([GitLab Docs](https://docs.gitlab.com/user/project/merge_requests/ "Merge requests | GitLab Docs"))
## 3. MR 머지 후 local `feature/aaa` 처리

### 3.1 원격 branch 삭제 상태 반영

#### `remote`의 `origin`에 `feature/aaa` 브랜치가 존재하는지 확인하려면 아래 명령을 사용하면 됩니다.
## 1. 가장 정확한 방법

```
git ls-remote --heads origin feature/aaa
```

### 존재하는 경우

```
abc123456789refs/heads/feature/aaa
```

또는 실제 출력은 아래처럼 나옵니다.

```
abc123456789abcdef123456789abcdef12345678	refs/heads/feature/aaa
```

### 존재하지 않는 경우

```
아무것도 출력되지 않음
```


MR 머지 후 GitLab에서 `origin/feature/aaa`가 삭제되었다면 local에서 먼저 원격 branch 정보를 정리합니다.

```bash
git fetch --prune origin
```

확인:

```bash
git branch -vv
```

예상 출력 예:

```text
feature/aaa  abc1234 [origin/feature/aaa: gone] 작업 내용
```

`gone`으로 보이면 remote tracking branch가 사라진 상태입니다.

### 3.2 local branch 삭제

작업이 `dev`에 정상 반영되었고 더 이상 필요 없으면 삭제합니다.

```bash
git switch dev
git pull --ff-only origin dev
git branch -d feature/aaa
```

만약 Git이 “완전히 merge되지 않았다”고 판단해서 삭제를 거부하지만, 이미 GitLab MR로 반영된 것을 확인했다면 강제 삭제할 수 있습니다.

```bash
git branch -D feature/aaa
```

- 이 명령은 **내 PC의 로컬 브랜치 `feature/aaa`만 삭제**합니다. `origin/feature/aaa` 원격 브랜치나 다른 개발자 PC의 브랜치에는 **영향 없습니다.**
### `-d`와 `-D` 차이

| 명령                          | 의미                   | 안전성   |
| --------------------------- | -------------------- | ----- |
| `git branch -d feature/aaa` | merge된 브랜치만 삭제       | 안전    |
| `git branch -D feature/aaa` | merge 여부와 관계없이 강제 삭제 | 주의 필요 |

- `-D`는 강제 삭제입니다. 아직 `dev`나 `origin/feature/aaa`에 push/merge되지 않은 commit이 있으면, 브랜치를 삭제한 뒤 해당 commit을 찾기 어려워질 수 있습니다.
## 삭제 전 확인 권장

```
git switch feature/aaa
git statusgit log --oneline --decorate -5
git branch -vv
```

또는 `dev`에 반영됐는지 확인:

```
git switch dev
git pull --ff-only origin dev
git branch --merged
```

`feature/aaa`가 목록에 있으면 현재 기준으로 merge된 브랜치입니다.
## 4. MR 완료 후 새 작업을 시작하는 표준 흐름

MR이 끝난 `feature/aaa`를 계속 재사용하지 말고, 보통은 **최신 `dev`에서 새 feature branch를 다시 따는 방식**이 안전합니다.

```bash
git switch dev
git fetch --prune origin
git pull --ff-only origin dev
git switch -c feature/bbb
```

작업 후:

```bash
git status
git add .
git commit -m "작업 내용"
git push -u origin feature/bbb
```

그 다음 GitLab에서:

```text
source: feature/bbb
target: dev
```

로 새 Merge Request를 생성합니다.

## 5. 다른 개발자가 `dev`에 push한 내용을 현재 작업 중인 `feature/aaa`에 반영하는 방법

현재 `feature/aaa`에서 아직 작업 중이고, 다른 개발자가 `dev`에 먼저 merge/push한 내용을 내 branch에 가져와야 한다면 아래 두 방식 중 하나를 사용합니다.

## 6. 방법 A: `merge` 방식 — 실무에서 가장 안전

공유 branch이거나, 강제 push를 피하고 싶으면 이 방식이 좋습니다.

```bash
git fetch origin
git switch feature/aaa
git merge origin/dev
```

충돌 발생 시:

```bash
# 충돌 파일 수정
git status
git add .
git commit
```

그 다음 remote feature branch에 반영:

```bash
git push origin feature/aaa
```

이후 GitLab MR 화면에서 변경 사항과 충돌 여부를 확인합니다.

### merge 방식 장단점

|항목|내용|
|---|---|
|장점|강제 push가 필요 없어 안전함|
|장점|다른 사람이 같은 feature branch를 보고 있어도 영향이 적음|
|단점|merge commit이 생길 수 있음|
|추천|운영/협업 프로젝트, GitLab MR 중심 개발|

## 7. 방법 B: `rebase` 방식 — commit history 정리용

혼자 작업하는 branch이고 commit history를 깔끔하게 유지하고 싶으면 사용할 수 있습니다.

```bash
git fetch origin
git switch feature/aaa
git rebase origin/dev
```

충돌 발생 시:

```bash
# 충돌 파일 수정
git add .
git rebase --continue
```

취소하려면:

```bash
git rebase --abort
```

rebase 후 remote branch에 반영:

```bash
git push --force-with-lease origin feature/aaa
```

### rebase 방식 장단점

|항목|내용|
|---|---|
|장점|commit history가 깔끔함|
|장점|`dev` 최신 기준 위에 내 작업 commit이 다시 쌓임|
|단점|강제 push가 필요할 수 있음|
|주의|다른 개발자가 같은 branch를 쓰면 위험|
|추천|혼자 사용하는 feature branch|

## 8. “작업 후 다시 dev에 push”의 권장 흐름

실무 GitLab 프로젝트에서는 일반적으로 `dev`에 직접 push하지 않고 MR로 반영합니다. GitLab 문서도 merge request는 branch에서 작업하고 리뷰 후 병합하는 협업 흐름으로 설명합니다. ([GitLab Docs](https://docs.gitlab.com/user/project/merge_requests/ "Merge requests | GitLab Docs"))  
권장 흐름:

```bash
git fetch origin
git switch feature/aaa
git merge origin/dev
# 또는 git rebase origin/dev

# 작업
git add .
git commit -m "작업 내용"
git push origin feature/aaa
```

그 후 GitLab에서:

```text
feature/aaa → dev Merge Request 생성 또는 기존 MR 갱신
```

MR 승인 후 머지합니다.

## 9. 직접 `dev`에 push해야 하는 경우

권한/정책상 직접 push가 허용된 경우에만 사용합니다.

```bash
git fetch origin
git switch dev
git pull --ff-only origin dev
git merge --no-ff feature/aaa
git push origin dev
```

단, `dev`가 protected branch라면 직접 push가 막힐 수 있습니다. GitLab은 기본 branch나 protected branch에 대해 역할/보호 설정에 따라 merge 권한을 제한합니다. ([GitLab Docs](https://docs.gitlab.com/user/project/merge_requests/ "Merge requests | GitLab Docs"))

## 10. 전체 작업 시나리오 정리

### 시나리오 1: MR 머지 완료 + source branch 삭제 선택

```bash
git fetch --prune origin
git switch dev
git pull --ff-only origin dev
git branch -d feature/aaa
```

삭제가 안 되면:

```bash
git branch -D feature/aaa
```

### 시나리오 2: MR 완료 후 새 작업 시작

```bash
git switch dev
git pull --ff-only origin dev
git switch -c feature/bbb
# 작업
git add .
git commit -m "새 작업 내용"
git push -u origin feature/bbb
```

### 시나리오 3: 현재 `feature/aaa` 작업 중인데 `dev` 최신 내용 반영

안전한 merge 방식:

```bash
git fetch origin
git switch feature/aaa
git merge origin/dev
git push origin feature/aaa
```

깔끔한 rebase 방식:

```bash
git fetch origin
git switch feature/aaa
git rebase origin/dev
git push --force-with-lease origin feature/aaa
```

## 11. 실무 권장 기준

|상황|권장 작업|
|---|---|
|MR 머지 완료|`origin/feature/aaa` 삭제 가능|
|로컬 branch|직접 `git branch -d feature/aaa`로 삭제|
|같은 branch 재사용|비권장|
|새 작업|최신 `dev`에서 새 `feature/*` 생성|
|다른 개발자 변경 반영|`merge origin/dev` 권장|
|혼자 쓰는 branch|`rebase origin/dev` 가능|
|공유 branch|`rebase` 후 force push 비권장|
|dev 반영|직접 push보다 MR 권장|

## 12. 최종 권장 명령 세트

가장 안전한 운영 흐름은 아래입니다.

```bash
# 1. dev 최신화
git fetch --prune origin
git switch dev
git pull --ff-only origin dev

# 2. 새 feature branch 생성
git switch -c feature/aaa

# 3. 작업 후 commit
git add .
git commit -m "작업 내용"

# 4. 원격 feature branch push
git push -u origin feature/aaa

# 5. GitLab에서 MR 생성
# source: feature/aaa
# target: dev
```

MR 머지 후:

```bash
# 6. 원격 삭제 상태 반영
git fetch --prune origin

# 7. dev 최신화
git switch dev
git pull --ff-only origin dev

# 8. local feature branch 삭제
git branch -d feature/aaa
```

핵심은 **GitLab의 `Delete source branch`는 원격 `origin/feature/aaa` 삭제 옵션이고, 내 PC의 local `feature/aaa`는 직접 삭제해야 한다는 점**입니다.

## 내 작업 후 확인 절차

`dev`에서 `feature/aaa`를 만든 뒤 작업하는 동안 다른 개발자가 `dev`에 push/merge한 내용이 있다면, **MR 신청 전 `origin/dev` 최신 내용을 내 `feature/aaa`에 먼저 반영**해야 합니다.  
실무에서는 보통 아래 순서를 권장합니다.

```bash
git fetch origin
git switch feature/aaa
git merge origin/dev
# 충돌 해결
git push origin feature/aaa
```

그 후 GitLab에서 `feature/aaa → dev` Merge Request를 신청합니다.

## 권장 작업 흐름

### 1. 처음 feature branch 생성

```bash
git fetch origin
git switch dev
git pull --ff-only origin dev
git switch -c feature/aaa
```

작업 후 commit:

```bash
git status
git add .
git commit -m "작업 내용"
```

원격 feature branch 생성:

```bash
git push -u origin feature/aaa
```

## 2. MR 신청 전 dev 최신 내용 반영

MR 신청 직전에 반드시 최신 `dev`를 가져옵니다.

```bash
git fetch origin
```

현재 작업 branch 확인:

```bash
git branch
```

`feature/aaa`로 이동:

```bash
git switch feature/aaa
```

`origin/dev` 내용을 내 branch에 병합:

```bash
git merge origin/dev
```

충돌이 없으면 바로 push:

```bash
git push origin feature/aaa
```

이후 GitLab에서 MR 생성:

```text
source branch: feature/aaa
target branch: dev
```

## 3. 충돌 발생 시 처리

`git merge origin/dev` 중 충돌이 발생하면:

```bash
git status
```

충돌 파일을 수정한 뒤:

```bash
git add .
git commit
git push origin feature/aaa
```

GitLab MR 화면에서 conflict가 사라졌는지 확인합니다.

## 4. 전체 명령어 요약

```bash
# dev 최신화 후 feature 생성
git fetch origin
git switch dev
git pull --ff-only origin dev
git switch -c feature/aaa

# 작업 후 commit
git add .
git commit -m "작업 내용"
git push -u origin feature/aaa

# MR 신청 직전 dev 최신 변경 반영
git fetch origin
git switch feature/aaa
git merge origin/dev

# 충돌 해결 후 또는 충돌 없으면 push
git push origin feature/aaa
```

## 5. merge 방식 권장 이유

|항목|merge 방식|
|---|---|
|명령|`git merge origin/dev`|
|장점|강제 push가 필요 없음|
|장점|다른 사람이 내 feature branch를 보고 있어도 안전함|
|단점|merge commit이 생길 수 있음|
|추천|GitLab MR 기반 협업 프로젝트|

## 6. rebase 방식도 가능한가?

가능합니다. 다만 **혼자 쓰는 feature branch일 때만 권장**합니다.

```bash
git fetch origin
git switch feature/aaa
git rebase origin/dev
```

충돌 해결:

```bash
git add .
git rebase --continue
```

push:

```bash
git push --force-with-lease origin feature/aaa
```

주의:

```text
rebase는 commit 이력을 다시 쓰므로 이미 origin에 push한 branch라면 force push가 필요할 수 있음.
다른 개발자가 같은 feature/aaa를 기준으로 작업 중이면 rebase는 피하는 것이 안전함.
```

## 7. 실무 권장 기준

|상황|권장|
|---|---|
|일반 협업 branch|`merge origin/dev`|
|혼자 쓰는 개인 feature branch|`rebase origin/dev` 가능|
|이미 MR 생성한 branch|`merge origin/dev` 권장|
|강제 push 금지 프로젝트|`merge origin/dev` 사용|
|commit history 정리 우선|`rebase` 검토|

## 8. MR 전 최종 확인

```bash
git status
```

정상 상태:

```text
nothing to commit, working tree clean
```

최신 dev 반영 여부 확인:

```bash
git log --oneline --graph --decorate --all -20
```

원격 push 상태 확인:

```bash
git branch -vv
```

## 최종 추천 절차

MR 신청 전에는 아래만 기억하면 됩니다.

```bash
git fetch origin
git switch feature/aaa
git merge origin/dev
git push origin feature/aaa
```

이후 GitLab에서 `feature/aaa → dev`로 MR을 생성하면 됩니다.  
운영/협업 프로젝트에서는 `rebase`보다 `merge origin/dev` 방식이 충돌 처리와 이력 추적 측면에서 더 안전합니다.

## git merge origin/dev 에[서 충돌 발생 시

`feature/aaa`에서 `git merge origin/dev` 중 충돌이 났고, **충돌 파일은 일단 `origin/dev` 최신 파일 기준으로 가져온 뒤 내 작업 내용을 다시 추가**하려면 아래 방식으로 처리하면 됩니다.

```bash
git checkout --theirs -- 충돌파일
# 또는 최신 Git
git restore --source=MERGE_HEAD -- 충돌파일
```

`merge` 충돌 상황에서:

|표현|의미|
|---|---|
|`--ours`|현재 작업 브랜치 `feature/aaa`의 파일|
|`--theirs`|병합해오는 브랜치 `origin/dev`의 파일|
|즉, `feature/aaa`에서 `git merge origin/dev`를 실행했다면 `--theirs`가 `origin/dev` 최신 파일입니다.||

## 1. 충돌 발생 후 현재 상태 확인

```bash
git status
```

예:

```text
both modified: src/main/java/com/example/TestService.java
```

## 2. 충돌 파일을 `origin/dev` 기준으로 덮어쓰기

특정 파일 하나만 `origin/dev` 버전으로 가져오기:

```bash
git checkout --theirs -- src/main/java/com/example/TestService.java
```

최신 Git 명령 방식:

```bash
git restore --source=MERGE_HEAD -- src/main/java/com/example/TestService.java
```

이 시점의 의미:

```text
충돌 파일 내용이 origin/dev 기준 최신 파일로 바뀜
내 feature/aaa 쪽 변경 내용은 해당 파일에서는 일단 제거됨
```

## 3. 그 위에 내 작업 내용 다시 추가

파일을 열어서 `origin/dev` 최신 내용 위에 내가 필요한 수정사항을 다시 반영합니다.

```text
src/main/java/com/example/TestService.java 수정
```

수정 후 확인:

```bash
git diff
```

## 4. 충돌 해결 처리

수정이 끝났으면 Git에 충돌 해결 완료를 표시합니다.

```bash
git add src/main/java/com/example/TestService.java
```

충돌 파일이 여러 개면 각각 처리 후:

```bash
git status
```

모든 충돌이 해결되면 merge commit 생성:

```bash
git commit
```

Git이 자동으로 merge commit 메시지를 띄우면 그대로 저장해도 됩니다.

## 5. 원격 feature branch에 push

```bash
git push origin feature/aaa
```

이후 GitLab에서 `feature/aaa → dev` MR을 생성하거나, 이미 MR이 있다면 자동으로 갱신됩니다.

## 전체 명령 예시

```bash
# 1. dev 최신 내용 가져오기
git fetch origin
# 2. feature 브랜치에서 dev 병합
git switch feature/aaa
git merge origin/dev
# 3. 충돌 발생 파일 확인
git status
# 4. 충돌 파일을 origin/dev 버전으로 선택
git checkout --theirs -- src/main/java/com/example/TestService.java
# 5. 파일 열어서 내 작업 내용 다시 추가
# 6. 수정 확인
git diff
# 7. 충돌 해결 표시
git add src/main/java/com/example/TestService.java
# 8. merge commit
git commit
# 9. 원격 feature branch push
git push origin feature/aaa
```

## 여러 충돌 파일을 전부 `origin/dev` 기준으로 가져올 때

주의해서 사용해야 합니다.

```bash
git checkout --theirs -- .
```

또는:

```bash
git restore --source=MERGE_HEAD -- .
```

이 명령은 충돌 파일뿐 아니라 merge 대상 파일을 넓게 바꿀 수 있으므로, 실무에서는 파일 단위로 처리하는 것을 권장합니다.

```bash
git checkout --theirs -- path/to/file1
git checkout --theirs -- path/to/file2
```

## 내 변경 내용을 보존해두고 싶을 때

충돌 상태에서 `origin/dev` 파일로 덮기 전에 내 브랜치 쪽 파일을 백업하려면:

```bash
git show :2:src/main/java/com/example/TestService.java > /tmp/TestService.feature.java
```

여기서 `:2:`는 merge 충돌 상태에서 **ours**, 즉 `feature/aaa` 쪽 파일입니다.  
`origin/dev` 쪽 파일은:

```bash
git show :3:src/main/java/com/example/TestService.java > /tmp/TestService.dev.java
```

그 후:

```bash
git checkout --theirs -- src/main/java/com/example/TestService.java
```

그리고 `/tmp/TestService.feature.java`를 참고해서 필요한 내 변경만 다시 반영하면 됩니다.

## 충돌 처리가 꼬였을 때 취소

merge 자체를 취소하고 처음으로 돌아가려면:

```bash
git merge --abort
```

그 후 다시:

```bash
git fetch origin
git merge origin/dev
```

## 실무 권장 절차

```bash
git fetch origin
git switch feature/aaa
git merge origin/dev
git status
git checkout --theirs -- 충돌파일
# 파일 열어서 내 변경 재반영
git add 충돌파일
git commit
git push origin feature/aaa
```

핵심은 **충돌 상태에서 `--theirs`가 `origin/dev` 최신 파일**이라는 점입니다. 그 파일을 기준으로 가져온 뒤, 내 작업 변경사항만 다시 추가해서 `git add → git commit → git push` 하면 됩니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
