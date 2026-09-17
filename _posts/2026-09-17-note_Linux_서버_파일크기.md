---
layout: single
title: "Linux_서버_파일크기"
excerpt: "Linux_서버_파일크기"
categories:
  - "tech"
tags:
  - "tech"
  - "memo"
toc: false
toc_sticky: true
date: "2026-09-17"
last_modified_at: "2026-09-17 18:03:01 +0900"
mermaid: false
---
가능합니다. 운영 서버에서 **부하를 최소화하려면 “크기 조회”와 “파일 수 조회”를 분리**하는 편이 좋습니다. 두 작업 모두 정확한 값을 얻으려면 결국 하위 파일시스템을 순회해야 하지만, `nice + ionice + -x/-xdev`를 사용하면 서비스에 미치는 영향을 줄일 수 있습니다.

아래에서는 조회 대상 디렉토리를 다음처럼 가정하겠습니다.

```bash
DIR="/app/data"
```

## 1. 가장 낮은 부하 우선: 대상 디렉토리 전체 크기만 조회

```bash
ionice -c 3 nice -n 19 du -shx "$DIR" 2>/dev/null
```

예상 결과:

```text
18G     /app/data
```

옵션 설명:

| 옵션            | 의미                               |
| ------------- | -------------------------------- |
| `ionice -c 3` | I/O 우선순위를 Idle 등급으로 낮춤           |
| `nice -n 19`  | CPU 스케줄링 우선순위를 가장 낮게 설정          |
| `du -s`       | 전체 합계만 출력                        |
| `-h`          | KB/MB/GB 단위 표시                   |
| `-x`          | 다른 파일시스템/NFS 등의 마운트 영역으로 넘어가지 않음 |
| `2>/dev/null` | 권한 오류 메시지 제거                     |

정확한 디렉토리 크기를 구하려면 내부 파일을 순회해야 하므로 **부하가 0은 아닙니다.** 다만 운영 중 실행한다면 이 형태를 가장 먼저 권장합니다.

---

# 2. 대상 디렉토리 + 바로 아래 하위 디렉토리 크기

다음 명령은 **한 번의 `du` 탐색으로** 대상 디렉토리와 1단계 하위 디렉토리의 크기를 모두 구합니다.

```bash
ionice -c 3 nice -n 19 du -xh --max-depth=1 "$DIR" 2>/dev/null
```

예:

```text
2.1G    /app/data/log
350M    /app/data/temp
12G     /app/data/upload
3.5G    /app/data/archive
18G     /app/data
```

큰 디렉토리부터 보고 싶다면:

```bash
ionice -c 3 nice -n 19 du -xh --max-depth=1 "$DIR" 2>/dev/null | sort -hr
```

### 2-1-1. 중요한 점

```bash
--max-depth=1
```

은 **탐색 깊이를 1단계로 제한하는 옵션이 아닙니다.**

출력만 1단계로 제한합니다.

예를 들어:

```text
/app/data
 ├─ log
 │   ├─ 2024
 │   │   └─ 수십만 파일
 │   └─ 2025
 └─ upload
     └─ 수백만 파일
```

라고 되어 있다면 `du`는 정확한 크기를 구하기 위해 저 내부까지 모두 확인합니다.

---

# 3. 대상 디렉토리 전체 파일 수만 조회

크기 조회와 **별도로 실행하는 것을 권장**합니다.

```bash
ionice -c 3 nice -n 19 find "$DIR" -xdev -type f -printf '1\n' 2>/dev/null | wc -l
```

예:

```text
128543
```

즉:

```text
/app/data 전체 파일 수 = 128,543개
```

옵션 설명:

| 옵션              | 의미                 |
| --------------- | ------------------ |
| `find "$DIR"`   | 지정한 디렉토리 이하 탐색     |
| `-xdev`         | 다른 파일시스템으로 넘어가지 않음 |
| `-type f`       | 일반 파일만 계산          |
| `-printf '1\n'` | 파일 이름 대신 1줄씩 출력    |
| `wc -l`         | 줄 개수 = 파일 개수       |

단순히:

```bash
find "$DIR" -type f | wc -l
```

도 가능하지만, 운영 환경에서는 위 명령을 더 권장합니다.

`-printf '1\n'`을 사용하는 이유는 파일명이 길어도 불필요한 문자열을 파이프로 전달하지 않기 때문입니다.

---

# 4. 바로 아래 하위 디렉토리별 파일 수

이 작업은 크기 조회보다 조금 더 주의해야 합니다.

**하위 디렉토리마다 `find`를 반복 실행하지 않고, 전체를 한 번만 순회해서 집계하는 방식**이 상대적으로 효율적입니다.

```bash
ionice -c 3 nice -n 19 find "$DIR" -xdev -type f -printf '%P\n' 2>/dev/null | awk -F/ '
{
    total++
    if (NF == 1)
        root++
    else
        count[$1]++
}
END {
    printf "%-40s %12d\n", "[ROOT 직접 파일]", root
    for (d in count)
        printf "%-40s %12d\n", d, count[d]
    printf "%-40s %12d\n", "[TOTAL]", total
}'
```

예:

```text
[ROOT 직접 파일]                                  32
log                                           15231
temp                                            853
upload                                       104527
archive                                        7900
[TOTAL]                                      128543
```

이 방식의 장점은:

```text
find 1회
      ↓
전체 파일 확인
      ↓
awk가 첫 번째 하위 디렉토리 기준으로 집계
```

하기 때문에 다음처럼 **각 디렉토리마다 `find`를 반복하는 것보다 효율적**입니다.

---

# 5. 운영 서버에서는 가능하면 피할 방식

다음 방식은 이해하기는 쉽지만 파일이 많은 서버에서는 권장하지 않습니다.

```bash
for d in "$DIR"/*/; do
    echo "$d"
    find "$d" -type f | wc -l
done
```

왜냐하면 하위 디렉토리별로 `find` 프로세스를 반복 실행하기 때문입니다.

하위 디렉토리가:

```text
100개
```

라면 최대 100회의 `find`가 실행됩니다.

각 디렉토리는 서로 다른 영역이므로 모든 파일을 중복해서 읽는 것은 아니지만, 프로세스 생성과 파일시스템 탐색이 반복되기 때문에 운영 서버에서는 앞의 **단일 `find + awk` 방식이 더 적절**합니다.

---

# 6. 크기와 파일 수를 동시에 조회하지 않는 것을 권장

원하는 최종 결과는 이런 형태일 것입니다.

```text
디렉토리                    크기        파일수
------------------------------------------------
/app/data                   18G         128543
/app/data/log               2.1G         15231
/app/data/temp              350M           853
/app/data/upload             12G        104527
/app/data/archive           3.5G          7900
```

하지만 운영 서버에서 이를 한 번에 만드는 명령은 권장하지 않습니다.

예를 들어 이런 방식:

```bash
for d in "$DIR" "$DIR"/*/; do
    du -sh "$d"
    find "$d" -type f | wc -l
done
```

은 **상위 디렉토리를 먼저 전체 탐색한 뒤 각 하위 디렉토리를 다시 탐색**합니다.

즉 파일 수가 많을수록 불필요한 재탐색이 증가합니다.

따라서 아래처럼 **2개 작업으로 분리**하는 것이 좋습니다.

### 6-1-1. 1차: 크기 조회

```bash
ionice -c 3 nice -n 19 du -xh --max-depth=1 "$DIR" 2>/dev/null | sort -hr
```

### 6-1-2. 2차: 파일 수 조회

```bash
ionice -c 3 nice -n 19 find "$DIR" -xdev -type f -printf '%P\n' 2>/dev/null | awk -F/ '
{
    total++
    if (NF == 1)
        root++
    else
        count[$1]++
}
END {
    printf "%-40s %12d\n", "[ROOT 직접 파일]", root
    for (d in count)
        printf "%-40s %12d\n", d, count[d]
    printf "%-40s %12d\n", "[TOTAL]", total
}'
```

이렇게 하면 전체 파일시스템 순회가 사실상 **크기 계산 1회 + 파일 개수 계산 1회**로 제한됩니다.

---

# 7. 일반적인 간단한 명령

운영 부하를 크게 신경 쓰지 않아도 되는 서버라면 훨씬 간단하게 사용할 수 있습니다.

### 7-1-1. 전체 크기

```bash
du -sh /app/data
```

### 7-1-2. 하위 디렉토리 포함 크기

```bash
du -h --max-depth=1 /app/data
```

### 7-1-3. 크기순 정렬

```bash
du -h --max-depth=1 /app/data | sort -hr
```

### 7-1-4. 전체 파일 수

```bash
find /app/data -type f | wc -l
```

### 7-1-5. 바로 아래 디렉토리만 확인

```bash
find /app/data -mindepth 1 -maxdepth 1 -type d
```

---

# 8. 권장 실행 순서

현재처럼 **운영 서버의 상태를 확인하는 목적**이라면 저는 다음 순서를 권장합니다.

### 8-1-1. ① 파일시스템 전체 상황 확인 — 부하 매우 낮음

```bash
df -h "$DIR"
```

`df`는 파일들을 일일이 순회하지 않기 때문에 부담이 매우 작습니다.

### 8-1-2. ② 지정 디렉토리 전체 크기

```bash
ionice -c 3 nice -n 19 du -shx "$DIR"
```

### 8-1-3. ③ 문제가 있다면 하위 디렉토리별 크기

```bash
ionice -c 3 nice -n 19 du -xh --max-depth=1 "$DIR" 2>/dev/null | sort -hr
```

### 8-1-4. ④ 파일 수가 필요한 경우에만 전체 파일 수

```bash
ionice -c 3 nice -n 19 find "$DIR" -xdev -type f -printf '1\n' 2>/dev/null | wc -l
```

### 8-1-5. ⑤ 마지막으로 하위 디렉토리별 파일 수

```bash
ionice -c 3 nice -n 19 find "$DIR" -xdev -type f -printf '%P\n' 2>/dev/null | awk -F/ '
{
    total++
    if (NF == 1)
        root++
    else
        count[$1]++
}
END {
    printf "%-40s %12d\n", "[ROOT 직접 파일]", root
    for (d in count)
        printf "%-40s %12d\n", d, count[d]
    printf "%-40s %12d\n", "[TOTAL]", total
}'
```

**핵심은 `du`와 `find`를 하나의 반복문 안에 묶지 않는 것입니다.** 파일이 수십만~수백만 개라면 `du` 자체보다 오히려 디렉토리별로 `du + find`를 반복하는 스크립트가 운영 서버에 더 부담을 줄 수 있습니다.
<details>
  <summary>참고</summary>  
  <pre>
  </pre>
</details>
