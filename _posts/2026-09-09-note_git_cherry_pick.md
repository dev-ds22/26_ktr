---
layout: single
title: "git_cherry_pick"
excerpt: "git_cherry_pick"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-09"
last_modified_at: "2026-09-09 10:34:23 +0900"
mermaid: false
---
## 1. `origin/feature/test1`의 일부 변경만 `origin/dev`에 반영하는 Cherry-pick 방법

`cherry-pick`은 **브랜치 전체를 merge하지 않고 원하는 Commit만 골라서 현재 브랜치에 반영**하는 기능입니다.
예를 들어:

```text
origin/dev
A --- B --- C
             \
origin/feature/test1
              D --- E --- F
```

여기서 `E`만 `dev`에 반영하고 싶다면:

```text
A --- B --- C --- E'
```

형태로 만드는 것이 `cherry-pick`입니다.

## 2. 가장 일반적인 작업 절차

### 2-1. ① Remote 최신 정보 가져오기

```bash
git fetch origin
```

`git fetch`는 실제 작업 브랜치를 변경하지 않고 `origin/dev`, `origin/feature/test1` 정보를 최신화합니다.

### 2-2. ② `dev` 브랜치로 이동

```bash
git switch dev
```

구버전 Git에서는:

```bash
git checkout dev
```

### 2-3. ③ 로컬 `dev`를 `origin/dev`와 동일하게 맞춤

일반적인 경우:

```bash
git pull origin dev
```

로컬 `dev`에 개인 작업이 없고 **무조건 origin/dev 기준으로 정확히 맞추려는 경우**:

```bash
git fetch origin
git reset --hard origin/dev
```

> `reset --hard`는 로컬 변경사항을 삭제하므로 수정 중인 파일이 있을 때는 사용하면 안 됩니다.

## 3. `feature/test1`에서 반영할 Commit 찾기

다음 명령으로 `dev`에는 없고 `feature/test1`에만 있는 Commit을 확인합니다.

```bash
git log --oneline origin/dev..origin/feature/test1
```

예:

```text
f82b617 회원가입 트랜잭션 개선
34fc910 판매자 세션 체크 수정
74a1962 공통 로그 처리 변경
```

여기서 `34fc910`만 `dev`에 넣고 싶다고 가정합니다.

## 4. 원하는 Commit 하나만 Cherry-pick

현재 브랜치가 `dev`인지 먼저 확인:

```bash
git branch --show-current
```

결과:

```text
dev
```

그 다음:

```bash
git cherry-pick 34fc910
```

성공하면:

```text
origin/feature/test1의 34fc910
        ↓
local dev에 새로운 commit으로 생성
```

원래 Commit hash가 `34fc910`이었다고 해도 `dev`에 생성되는 Commit hash는 보통 달라집니다.

## 5. 여러 Commit을 선택적으로 반영

예를 들어:

```text
f82b617  → 반영
34fc910  → 반영 안 함
74a1962  → 반영
```

이라면:

```bash
git cherry-pick f82b617
git cherry-pick 74a1962
```

또는 한 번에:

```bash
git cherry-pick f82b617 74a1962
```

### 5-1. 순서 중요

Commit 간 의존성이 있으면 **오래된 Commit부터** 적용하는 것이 안전합니다.
예:

```text
A → B → C
```

라면:

```bash
git cherry-pick A B C
```

순으로 진행합니다.

## 6. 연속된 Commit 여러 개를 전부 반영

예를 들어:

```text
A
B
C
D
```

중 `B ~ D`까지 모두 적용하려면:

```bash
git cherry-pick B^..D
```

예:

```bash
git cherry-pick 34fc910^..f82b617
```

다만 실무에서는 대상 Commit 수가 적다면 명시적으로:

```bash
git cherry-pick 34fc910 abc1234 f82b617
```

처럼 작성하는 편이 실수 방지에 좋습니다.

## 7. Cherry-pick 후 확인

### 7-1. 적용된 Commit 확인

```bash
git log --oneline -10
```

### 7-2. `origin/dev` 대비 변경 확인

```bash
git diff origin/dev
```

### 7-3. 변경된 파일만 확인

```bash
git diff --name-status origin/dev
```

예:

```text
M src/main/java/.../JoinSellerService.java
M src/main/java/.../JoinSellerController.java
```

문제가 없다면:

```bash
git push origin dev
```

다만 현재 프로젝트처럼 **`dev → prd`는 MR 기반이고 `feature → dev`도 MR 정책**이라면 직접 `origin/dev`에 push하지 말고 프로젝트 정책에 맞춰 MR을 사용하는 것이 안전합니다.

## 8. Cherry-pick 중 Conflict 발생 시

예:

```text
CONFLICT (content): Merge conflict in JoinSellerService.java
error: could not apply 34fc910...
```

### 8-1. ① 충돌 파일 확인

```bash
git status
```

### 8-2. ② 충돌 파일 직접 수정

파일에 다음 형태가 나타납니다.

```text
<<<<<<< HEAD
dev의 기존 코드
=======
feature/test1의 코드
>>>>>>> 34fc910
```

원하는 코드로 수정하고 위 표시들을 제거합니다.

### 8-3. ③ 수정 완료 등록

```bash
git add 수정한파일
```

예:

```bash
git add src/main/java/.../JoinSellerService.java
```

모든 충돌 해결 후:

```bash
git cherry-pick --continue
```

### 8-4. Cherry-pick 자체를 취소하고 원상복구

```bash
git cherry-pick --abort
```

그러면 cherry-pick 실행 전 상태로 돌아갑니다.

## 9. 중요한 차이: Commit 일부만 반영하고 싶은 경우

여기서 주의해야 합니다.
`cherry-pick`의 기본 단위는 **Commit**입니다.
예를 들어 Commit `34fc910`에:

```text
A.java 수정
B.java 수정
C.java 수정
```

이 모두 들어 있는데 `A.java`만 `dev`에 넣고 싶다면 단순:

```bash
git cherry-pick 34fc910
```

을 하면 안 됩니다.

### 9-1. 방법 1: Cherry-pick 후 Commit하지 않고 필요한 파일만 선택

```bash
git cherry-pick -n 34fc910
```

`-n` 또는 `--no-commit`은 변경사항만 가져오고 Commit은 하지 않습니다.
현재 상태 확인:

```bash
git status
```

예:

```text
modified: A.java
modified: B.java
modified: C.java
```

A.java만 필요하다면 B/C 변경을 되돌립니다.

```bash
git restore B.java
git restore C.java
```

그 다음:

```bash
git add A.java
git commit -m "A.java 변경사항 반영"
```

이 방식이 매우 유용합니다.

## 10. 한 Commit 안에서 특정 코드 라인만 가져오려면

예를 들어 `A.java` 안에서도 일부 수정만 반영하려면:

```bash
git cherry-pick -n 34fc910
```

후:

```bash
git restore --staged .
```

그리고:

```bash
git add -p
```

실행합니다.
Git이 변경 내용을 작은 단위(hunk)로 보여줍니다.

```text
Stage this hunk [y,n,q,a,d,s,e,?]?
```

주요 입력:

| 입력  | 의미              |
| --- | --------------- |
| `y` | 이 변경 반영         |
| `n` | 이 변경 제외         |
| `s` | 변경 덩어리를 더 작게 나눔 |
| `q` | 종료              |
| `a` | 이후 변경 모두 반영     |
| `d` | 이후 변경 모두 제외     |

필요한 부분만 선택한 후:

```bash
git commit -m "필요 변경사항만 dev 반영"
```

나머지 작업트리 변경을 제거:

```bash
git restore .
```

## 11. 실무에서 가장 추천하는 절차

현재 상황이:

> `origin/feature/test1`에는 이미 Push 완료되어 있고, 그중 필요한 변경만 `origin/dev`에 반영
> 이라면 다음 순서가 가장 안전합니다.

```bash
# 1. Remote 최신화
git fetch origin
# 2. dev 이동
git switch dev
# 3. origin/dev 최신화
git pull origin dev
# 4. feature/test1에만 있는 commit 확인
git log --oneline origin/dev..origin/feature/test1
# 5. 필요한 commit 선택
git cherry-pick <commit-hash>
# 6. 변경 확인
git status
git diff origin/dev
git log --oneline -10
# 7. 테스트
mvn test
# 또는 프로젝트에 맞는 테스트 수행
# 8. push
git push origin dev
```

## 12. "일부 변경"의 유형별 명령

| 반영하려는 범위            | 추천 방법                                          |
| ------------------- | ---------------------------------------------- |
| 특정 Commit 하나        | `git cherry-pick <hash>`                       |
| 특정 Commit 여러 개      | `git cherry-pick <hash1> <hash2>`              |
| 연속된 여러 Commit       | `git cherry-pick A^..D`                        |
| Commit 안의 특정 파일만    | `git cherry-pick -n <hash>` 후 불필요 파일 `restore` |
| Commit 안의 특정 코드만    | `git cherry-pick -n <hash>` + `git add -p`     |
| 잘못 cherry-pick 진행 중 | `git cherry-pick --abort`                      |
| 완료한 cherry-pick 취소  | 해당 Commit에 대해 `git revert <hash>` 권장           |

## 13. 현재 상황에서 핵심 명령 5개

Commit 단위로 가져오는 경우 이것만 기억해도 됩니다.

```bash
git fetch origin
git switch dev
git pull origin dev
git log --oneline origin/dev..origin/feature/test1
git cherry-pick <가져올_commit_hash>
```

확인 후:

```bash
git push origin dev
```

### 13-1. 예시

```bash
git fetch origin
git switch dev
git pull origin dev
git log --oneline origin/dev..origin/feature/test1
```

결과:

```text
853f123 세션 체크 개선
782a991 회원정보 수정
11ce733 공통 Controller 수정
```

`회원정보 수정`만 필요:

```bash
git cherry-pick 782a991
```

확인:

```bash
git diff origin/dev
git log --oneline -5
```

정상이라면:

```bash
git push origin dev
```

**중요한 판단 기준은 하나입니다.**

```text
Commit 전체 필요
    ↓
git cherry-pick <hash>
Commit 중 일부 파일/코드만 필요
    ↓
git cherry-pick -n <hash>
```

특히 여러 개발자가 사용하는 `dev`라면 **`origin/dev` 최신화 → cherry-pick → diff 확인 → 빌드/테스트 → push/MR** 순서를 지키는 것이 안전합니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
