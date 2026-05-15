---
layout: single
title: "Git_gitignore_오류_및_Pulling_is_not_possible"
excerpt: "Git_gitignore_오류_및_Pulling_is_not_possible"

categories:
  - tech
tags:
  - [tech, memo]

toc: false
toc_sticky: true

date: "2026-05-15"
last_modified_at: "2026-05-15 11:47:48 +0900"
---

정확도: 97%

## 결론

`.gitignore`에 아래 라인을 추가했는데도 동작하지 않는 가장 흔한 이유는 **해당 파일이 이미 Git에 추적되고 있기 때문**입니다.

```gitignore
/note/.obsidian/plugins/obsidian-git/data.json
```

Git 공식 문서 기준으로 `.gitignore`는 **아직 Git이 추적하지 않는 파일을 untracked 상태로 유지하기 위한 것**이며, 이미 tracked 상태인 파일은 `.gitignore`에 추가해도 계속 변경사항이 잡힙니다. 이미 추적 중인 파일을 중지하려면 `git rm --cached`로 index에서 제거해야 합니다. ([Git](https://git-scm.com/docs/gitignore?utm_source=chatgpt.com "gitignore Documentation"))

## 1. 가장 가능성 높은 원인: 이미 tracked 상태

먼저 확인:

```bash
git ls-files note/.obsidian/plugins/obsidian-git/data.json
```

결과가 나오면 이미 Git이 추적 중입니다.

```text
note/.obsidian/plugins/obsidian-git/data.json
```

이 경우 `.gitignore`는 정상이어도 무시되지 않습니다.

### 해결

```bash
git rm --cached note/.obsidian/plugins/obsidian-git/data.json
git add .gitignore
git commit -m "Ignore Obsidian Git plugin data"
```

이 명령은 **로컬 파일은 삭제하지 않고**, Git 추적 목록에서만 제거합니다.

## 2. `.gitignore` 위치 문제

패턴이 `/note/...`로 시작하면, 이 의미는 **해당 `.gitignore` 파일이 있는 디렉토리를 기준으로 루트 anchoring**입니다.  
즉, `.gitignore`가 Git 저장소 루트에 있다면 아래 경로만 매칭됩니다.

```text
<repository-root>/note/.obsidian/plugins/obsidian-git/data.json
```

그런데 실제 경로가 아래처럼 다르면 동작하지 않습니다.

```text
<repository-root>/docs/note/.obsidian/plugins/obsidian-git/data.json
<repository-root>/obsidian/note/.obsidian/plugins/obsidian-git/data.json
<repository-root>/Note/.obsidian/plugins/obsidian-git/data.json
```

### 실제 경로 확인

```bash
git status --short
```

또는:

```bash
find . -path "*obsidian-git/data.json"
```

## 3. 경로 대소문자 문제

GitHub는 Linux 기반으로 동작하고, Linux 파일 시스템은 일반적으로 대소문자를 구분합니다.  
예를 들어 실제 폴더가 `Note`인데 `.gitignore`에는 `note`라고 쓰면 매칭되지 않을 수 있습니다.

```gitignore
/note/.obsidian/plugins/obsidian-git/data.json
```

실제 경로가 아래라면 다릅니다.

```text
/Note/.obsidian/plugins/obsidian-git/data.json
```

## 4. 더 안전한 ignore 패턴

저장소 안 어디에 있든 Obsidian Git 플러그인의 `data.json`을 무시하려면 아래처럼 쓰는 편이 안전합니다.

```gitignore
**/.obsidian/plugins/obsidian-git/data.json
```

`note` 폴더 하위만 명확히 제한하려면:

```gitignore
note/.obsidian/plugins/obsidian-git/data.json
```

`/` 없이 써도 일반적으로 루트의 `note` 경로와 매칭됩니다. 다만 정확히 루트 기준으로만 제한하려면 현재 작성한 방식도 맞습니다.

## 5. 어떤 ignore 규칙이 적용되는지 확인

Git 공식 문서 기준으로 `git check-ignore`는 특정 경로가 어떤 ignore 규칙에 의해 무시되는지 디버깅할 때 사용하는 명령입니다. ([Git](https://git-scm.com/docs/git-check-ignore?utm_source=chatgpt.com "git-check-ignore Documentation"))

```bash
git check-ignore -v note/.obsidian/plugins/obsidian-git/data.json
```

아무 출력도 없으면 `.gitignore` 규칙이 매칭되지 않는 것입니다.  
이미 tracked 파일까지 무시 규칙을 확인하려면:

```bash
git check-ignore -v --no-index note/.obsidian/plugins/obsidian-git/data.json
```

## 권장 조치 순서

```bash
# 1. 실제 Git 추적 여부 확인
git ls-files note/.obsidian/plugins/obsidian-git/data.json

# 2. 이미 추적 중이면 index에서만 제거
git rm --cached note/.obsidian/plugins/obsidian-git/data.json

# 3. .gitignore에 규칙 추가 또는 수정
echo "**/.obsidian/plugins/obsidian-git/data.json" >> .gitignore

# 4. 변경사항 커밋
git add .gitignore
git commit -m "Ignore Obsidian Git plugin data"
```

## error: Pulling is not possible because you have unmerged files.

현재 오류는 **이전 `pull` 또는 `merge` 중 충돌 파일이 아직 해결되지 않아서 추가 `pull`이 불가능한 상태**입니다.  
요구사항이 **현재 로컬 파일 기준으로 엎어쓰기**라면, 충돌 파일에 대해 `ours`를 선택하면 됩니다.

## 1. 현재 상태 확인

```bash
git status
```

아래처럼 나오면 충돌 상태입니다.

```text
You have unmerged paths.
fix conflicts and run "git commit"
```

## 2. 로컬 파일 기준으로 모든 충돌 파일 덮어쓰기

```bash
git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --
```

의미:

```text
충돌 중인 파일 목록을 찾고,
각 파일을 현재 로컬 브랜치 버전으로 선택
```

## 3. 충돌 해결 처리

```bash
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

이렇게 하면 현재 merge/pull 충돌이 **로컬 파일 기준으로 해결**됩니다.

## 4. 원격에 반영

```bash
git push origin 현재브랜치명
```

예:

```bash
git push origin dev
```

## 특정 파일만 로컬 기준으로 덮어쓰기

전체가 아니라 특정 파일만 로컬 기준으로 선택하려면:

```bash
git checkout --ours -- 경로/파일명
git add 경로/파일명
git commit -m "Resolve conflict using local file"
```

## 주의사항

`git checkout --ours`는 현재 merge 충돌 상황에서 **내 로컬 브랜치 쪽 변경사항을 선택**합니다.

다만 `pull --rebase` 중 충돌이면 `ours`와 `theirs` 의미가 일반 merge와 다르게 느껴질 수 있습니다. 현재 상태가 rebase인지 확인하려면:

```bash
git status
```

`rebase in progress`가 보이면 아래처럼 처리하는 편이 맞습니다.

```bash
git checkout --ours -- 충돌파일
git add -A
git rebase --continue
```

## 가장 안전한 실행 순서

```bash
git status

git branch backup-before-resolve

git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --

git add -A
git commit -m "Resolve merge conflicts using local changes"

git push origin 현재브랜치명
```

## 만약 merge 자체를 취소하고 싶다면

로컬 파일 기준 해결이 아니라, 방금 진행 중인 merge/pull을 취소하고 싶으면:

```bash
git merge --abort
```

rebase 중이면:

```bash
git rebase --abort
```

## 최종 요약

```bash
git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

이게 현재 충돌 파일을 **내 로컬 파일 기준으로 선택해서 pull 충돌을 마무리하는 명령**입니다.

## Your branch and 'origin/main' have diverged,
## 의미

```text
Your branch and 'origin/main' have diverged,
and have 4 and 2 different commits each, respectively.
```

현재 로컬 `main`과 원격 `origin/main`이 서로 다른 커밋을 가지고 있다는 뜻입니다.

```text
로컬 main     : 원격에는 없는 커밋 4개 보유
origin/main  : 로컬에는 없는 커밋 2개 보유
```

즉, 단순히 `git pull`만 하면 자동으로 fast-forward가 안 되고, **merge/rebase/force push 중 하나를 선택해야 하는 상태**입니다.

## 현재 로컬 기준으로 원격을 엎어쓰고 싶을 때

기존 질문 흐름처럼 **내 로컬 파일/커밋을 기준으로 GitHub `main`을 덮어쓰기**라면 아래 순서가 맞습니다.

### 1. 현재 브랜치 확인

```bash
git branch
```

`main`에 있는지 확인합니다.

```bash
git switch main
```

### 2. 현재 상태 확인

```bash
git status
```

만약 아직 충돌 상태라면 먼저 해결해야 합니다.

```bash
git status
```

결과에 아래가 있으면:

```text
You have unmerged paths.
```

로컬 기준으로 충돌 해결:

```bash
git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

### 3. 로컬 기준으로 원격 main 덮어쓰기

```bash
git push --force-with-lease origin main
```

## `--force-with-lease`를 쓰는 이유

|명령|설명|
|---|---|
|`git push --force`|원격을 강제로 덮어씀. 다른 사람 작업까지 날릴 위험 큼|
|`git push --force-with-lease`|내가 마지막으로 알고 있던 원격 상태와 다르면 강제 push를 막음|
|따라서 실무에서는 단순 `--force`보다 아래가 안전합니다.||

```bash
git push --force-with-lease origin main
```

## 원격 변경 2개를 버리게 되는 구조

현재 상태에서 위 명령을 실행하면:

```text
로컬 main의 4개 커밋은 유지
origin/main의 2개 커밋은 원격 main에서 사라짐
```

즉, **GitHub main을 내 로컬 main 상태로 맞추는 작업**입니다.

## 실행 전 백업 권장

강제 push 전에는 백업 브랜치를 하나 만들어두는 것이 안전합니다.

```bash
git branch backup-before-force-main
```

원격에도 백업을 남기려면:

```bash
git push origin backup-before-force-main
```

그 다음:

```bash
git push --force-with-lease origin main
```

## 최종 명령 요약

```bash
git switch main
git status
git branch backup-before-force-main
git push origin backup-before-force-main
git push --force-with-lease origin main
```

충돌 상태가 남아 있으면 먼저:

```bash
git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --
git add -A
git commit -m "Resolve merge conflicts using local changes"
git push --force-with-lease origin main
```

## 주의

이 방식은 **원격 `origin/main`의 2개 커밋을 버리는 방식**입니다. 혼자 쓰는 저장소면 괜찮지만, 다른 사람이 같이 쓰는 저장소라면 원격 2개 커밋에 필요한 변경이 있는지 먼저 확인하는 것이 안전합니다.


### powershell 에서 xargs 오류

## 원인

`xargs`는 Linux/Git Bash 계열 명령어입니다.  
현재 실행 환경이 **Windows PowerShell**이라서 아래 오류가 발생한 것입니다.

```powershell
xargs : 'xargs' 용어가 cmdlet, 함수, 스크립트 파일 또는 실행할 수 있는 프로그램 이름으로 인식되지 않습니다.
```

## PowerShell에서 동일하게 실행하는 방법

기존 명령:

```bash
git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --
```

PowerShell에서는 아래처럼 실행하세요.

```powershell
git diff --name-only --diff-filter=U | ForEach-Object { git checkout --ours -- $_ }
```

그 다음:

```powershell
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

## 전체 명령 순서

현재 충돌 파일을 **내 로컬 파일 기준으로 선택**하려면:

```powershell
git status
git diff --name-only --diff-filter=U | ForEach-Object { git checkout --ours -- $_ }
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

이후 원격 `main`을 로컬 기준으로 덮어쓰려면:

```powershell
git push --force-with-lease origin main
```

## Git Bash를 쓰는 경우

Windows라도 **Git Bash**에서 실행하면 기존 `xargs` 명령을 그대로 사용할 수 있습니다.

```bash
git diff --name-only --diff-filter=U -z | xargs -0 git checkout --ours --
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

## 더 안전한 PowerShell 방식

파일명에 공백이 있을 수 있으면 아래 방식이 더 안전합니다.

```powershell
$files = git diff --name-only --diff-filter=U
foreach ($file in $files) {
    git checkout --ours -- "$file"
}
git add -A
git commit -m "Resolve merge conflicts using local changes"
```

## 주의

현재 `git status`에서 `rebase in progress`가 보이면 마지막은 `commit`이 아니라 아래를 써야 합니다.

```powershell
git rebase --continue
```

일반 `merge` 충돌이면:

```powershell
git commit -m "Resolve merge conflicts using local changes"
```

정리하면, PowerShell에서는 `xargs` 대신 **`ForEach-Object`**를 사용하면 됩니다.

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
