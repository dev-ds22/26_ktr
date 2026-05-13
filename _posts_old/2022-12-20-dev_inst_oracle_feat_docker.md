---
layout: single
title: "[ORACLE] Docker 환경에서 ORACLE 설치 및 기본설정"
excerpt: "Docker 환경에서 ORACLE 설치 및 기본설정"

categories:
  - tech
tags:
  - [tech, oracle, docker]

toc: false
toc_sticky: true

date: 2022-12-20
last_modified_at: 2022-12-20
---
# Docker 환경에서 ORACLE 설치 및 기본설정하기

## Docker ORACLE 설치

- Docker설치가 완료되었다면 windows terminal에서 다음 명령어를 순서대로 입력. 

### 1. Docker ORACLE 설치 - pull

- 오라클 설치에 사용할 이미지를 당겨옵니다.
- oracle-xe로 검색
  - 이미지 검색하기

```powershell
$  docker search oracle-xe 
```
  

```bash
PS C:\daiso\kafka\kafka_2.13-3.3.1> docker search oracle-xe
NAME                              DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
  oracleinanutshell/oracle-xe-11g                                                   247
  gvenzl/oracle-xe                  Oracle Database XE (21c, 18c, 11g) for every…   140
  wnameless/oracle-xe-11g-r2        Oracle Express Edition 11g Release 2 on Ubun…   91
  orangehrm/oracle-xe-11g            docker container with Oracle Express Editio…   17                   [OK]
  pvargacl/oracle-xe-18.4.0         Oracle Express Edition 18.4.0 on Oracle Linu…   8
  christophesurmont/oracle-xe-11g   Clone of the wnameless/oracle-xe-11g.           7
  jaspeen/oracle-xe-11g             Fork from sath89/docker-oracle-xe-11g - smal…   6                    [OK]
  thebookpeople/oracle-xe-11g                                                       4
  PS C:\daiso\kafka\kafka_2.13-3.3.1>
```


- 이미지 당겨오기

```
  $ docker pull wnameless/oracle-xe-11g-r2
```
  

```bash
  PS C:\daiso\kafka\kafka_2.13-3.3.1> docker pull wnameless/oracle-xe-11g-r2
  Using default tag: latest
  latest: Pulling from wnameless/oracle-xe-11g-r2
  5667fdb72017: Pull complete
  d83811f270d5: Pull complete
  ee671aafb583: Pull complete
  7fc152dfb3a6: Pull complete
  51896e240a72: Pull complete
  64e16b57ee0a: Pull complete
  Digest: sha256:e8cfa3733a2c11b415fb94a9632424025d69165fb4903a22206f6073be30eeb9
  Status: Downloaded newer image for wnameless/oracle-xe-11g-r2:latest
  docker.io/wnameless/oracle-xe-11g-r2:latest
  PS C:\daiso\kafka\kafka_2.13-3.3.1>
```

### 2. 컨테이너 띄우기
- 당겨온 이미지로 컨테이너 실행하기

```powershell
  $ docker run --name oracle11g -d -p 8080:8080 -p 1521:1521 wnameless/oracle-xe-11g-r2 
```
  

```bash
  PS C:\daiso\kafka\kafka_2.13-3.3.1> docker run --name oracle11g -d -p 8080:8080 -p 1521:1521 wnameless/oracle-xe-11g-r2
  9fcfdc9f8d804a2f1720864bba8eea8f0c92e18a3ba11ce35947f17b9add5729
```

- 컨테이너 뜬 것 확인

```powershell
  docker ps
```
  

```bash
  PS C:\daiso\kafka\kafka_2.13-3.3.1> docker ps
  CONTAINER ID   IMAGE                                        COMMAND                  CREATED          STATUS          PORTS                                                    NAMES
  9fcfdc9f8d80   wnameless/oracle-xe-11g-r2                   "/bin/sh -c '/usr/sb…"   43 seconds ago   Up 41 seconds   0.0.0.0:1521->1521/tcp, 22/tcp, 0.0.0.0:8080->8080/tcp   oracle11g
  55edd822a653   redis                                        "docker-entrypoint.s…"   3 hours ago      Up 3 hours      6379/tcp                                                 determined_northcutt
  5c0225a3f88d   redis                                        "docker-entrypoint.s…"   3 hours ago      Up 3 hours      0.0.0.0:6379->6379/tcp                                   myredis
  16a0be4882c1   mcr.microsoft.com/mssql/server:2019-latest   "/opt/mssql/bin/perm…"   26 hours ago     Up 3 hours      0.0.0.0:1433->1433/tcp                                   local-dev-mssql-server
```

### 3. SQLPLUS 접속
- 이미지에서 제공되는 기본 접속 정보는 system/oracle 입니다.
- 컨테이너 내부 sqlplus로 디비 접속

```powershell
  $ docker exec -u 0 -it oracle11g /bin/bash  
```
  

```bash
  PS C:\daiso\kafka\kafka_2.13-3.3.1> docker exec -u 0 -it oracle11g /bin/bash
  root@9fcfdc9f8d80:/# su oracle
  oracle@9fcfdc9f8d80:/$ sqlplus /nolog

  SQL*Plus: Release 11.2.0.2.0 Production on Tue Dec 20 07:53:55 2022

  Copyright (c) 1982, 2011, Oracle.  All rights reserved.

  SQL> conn / as sysdba
  Connected.
  SQL> SELECT 1 FROM dual;

          1
  ----------
          1

  SQL> select instance_name, version, status from v$instance;

  INSTANCE_NAME    VERSION           STATUS
  ---------------- ----------------- ------------
  XE               11.2.0.2.0        OPEN

  SQL>
```

- http://127.0.0.1:8080/apex/ 로 접속가능
  - 초기계정 : ADMIN / admin
  - M2makstp! 로 변경

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>