---
layout: single
title: "[REDIS] Docker 환경에서 Redis 설치 및 기본설정"
excerpt: "Docker 환경에서 Redis 설치 및 기본설정"

categories:
  - tech
tags:
  - [tech, redis, docker]

toc: false
toc_sticky: true

date: 2022-12-19
last_modified_at: 2022-12-19
---
# Docker 환경에서 Redis 설치 및 기본설정하기

- 출처 : https://oingdaddy.tistory.com/285

## Docker 설치는 각자

## Docker MSSQL 설치

- Docker설치가 완료되었다면 windows terminal에서 다음 명령어를 순서대로 입력. 

- 1. 도커 이미지 pull

```powershell
  $ docker pull redis
```
  

```bash
  PS C:\Users\M2M-NB-131> docker pull redis
  Using default tag: latest
  latest: Pulling from library/redis
  025c56f98b67: Pull complete
  060e65aed679: Pull complete
  b95291e865b7: Pull complete
  7b6050af44d2: Pull complete
  e64c0623c4eb: Pull complete
  85500bdb8386: Pull complete
  Digest: sha256:fdaa0102e0c66802845aa5c961cb89a091a188056811802383660cd9e10889da
  Status: Downloaded newer image for redis:latest
  docker.io/library/redis:latest
  PS C:\Users\M2M-NB-131>
```

# 설정파일 디렉토리 생성 및 설정파일 다운로드

```powershell
  $ mkdir -p /etc/redis
  $ wget http://download.redis.io/redis-stable/redis.conf -O /etc/redis/redis.conf
```
  

```bash
  PS C:\Users\M2M-NB-131> mkdir -p /etc/redis
      디렉터리: C:\etc
  Mode                 LastWriteTime         Length Name
  ----                 -------------         ------ ----
  d-----      2022-12-20   오후 1:21                redis

  PS C:\Users\M2M-NB-131> wget http://download.redis.io/redis-stable/redis.conf -O /etc/redis/redis.conf
```

# redis-net이라는 브리지를 사용

```powershell
  $ docker network create redis-net
```
  

```bash
  PS C:\Users\M2M-NB-131> docker network create redis-net
  70cce1404f850812dd47f2dc35b41b9fb28835a2f9da9086df5e476a9f6e1070
```

# 컨테이너 실행

```powershell
  $ docker run --name myredis -p 6379:6379 --network redis-net -d -v /etc/redis/redis.conf:/usr/local/etc/redis/redis.conf redis redis-server --appendonly yes
```
  

```bash
  PS C:\Users\M2M-NB-131> docker run --name myredis -p 6379:6379 --network redis-net -d -v /etc/redis/redis.conf:/usr/local/etc/redis/redis.conf redis redis-server --appendonly yes
  5c0225a3f88d1bbb335a88d6c971249183146fc22a0b0837597d53ad6d52c24b
  PS C:\Users\M2M-NB-131>
```

# redis-cli 로 실행

```powershell
  $ docker run -it --network redis-net --rm redis redis-cli -h myredis
```
  

```bash
  PS C:\Users\M2M-NB-131> docker run -it --network redis-net --rm redis redis-cli -h myredis
  myredis:6379> keys *
  (empty array)
  myredis:6379> set k_one v_one
  OK
  myredis:6379> mset k_two v_two k_tree v_tree
  OK
  myredis:6379> keys *
  1) "k_tree"
  2) "k_two"
  3) "k_one"
  myredis:6379>
```

# 컨테이너 접속

```powershell
  $ docker exec -it myredis /bin/bash
```

## 추가

### Redis 엔터프라이즈 Active-Active 이중화(Duplexing)

#### 커뮤니티 버전

- Active-Standby 구조로만 사용가능.
- HA(High Availability)를 구성하려면 추가로 3대의 센티널(Sentinel)이 필요.
- Active 마스터에 접속하기 위한 애플리케이션 개발이 어려움.
- Active 서버가 바뀔 수 있기 때문에 항상 Active 서버 IP와 포트를 센티널에 문의 후 접속필요.

#### 엔터프라이즈 버전

- Active-Active 마스터간 데이터를 실시간으로 동기화.
- VIP(Virtual IP) 하나로 레디스 서버에 접속.
  - 애플리케이션 개발이 쉬워짐.
  - Active 서버간에 부하 분산(Load Balancing)도 됩니다.
- 서버 대수가 줄어들어 비용이 절감.
- 센티널이 없으므로 운영단순.

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>