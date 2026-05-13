---
layout: single
title: "[MSSQL] Docker 환경에서 MSSQL 설치 및 기본설정"
excerpt: "Docker 환경에서 MSSQL 설치 및 기본설정"

categories:
  - tech
tags:
  - [tech, mssql, docker]

toc: false
toc_sticky: true

date: 2022-12-19
last_modified_at: 2022-12-19
---
# Docker 환경에서 MSSQL 설치 및 기본설정하기

- 출처 : https://oingdaddy.tistory.com/285

## Docker 설치는 각자

## Docker MSSQL 설치

- Docker설치가 완료되었다면 windows terminal에서 다음 명령어를 순서대로 입력. 

### 1. Docker MSSQL 설치 - pull

```powershell
  > docker pull mcr.microsoft.com/mssql/server:2019-latest
```
  

```bash
  PS C:\Users\M2M-NB-131> docker pull mcr.microsoft.com/mssql/server:2019-latest
    2019-latest: Pulling from mssql/server
    210a236fbb96: Pull complete
    4d3b5ee6a318: Pull complete
    b97468a53f24: Pull complete
    Digest: sha256:f57d743a99a4003a085d0fd67dbb5ecf98812c08a616697a065082cad68d77ce
    Status: Downloaded newer image for mcr.microsoft.com/mssql/server:2019-latest
    mcr.microsoft.com/mssql/server:2019-latest
  PS C:\Users\M2M-NB-131>
```

### 2. Docker MSSQL 설치 - run

```bash
  docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=m2makstp' -p 1433:1433 --name local-dev-mssql-server -d mcr.microsoft.com/mssql/server:2019-latest
```
  

```bash
  PS C:\Users\M2M-NB-131> docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=m2makstp!' -p 1433:1433 --name local-dev-mssql-server -d mcr.microsoft.com/mssql/server:2019-latest
  78b8b1912f0b1fc0acfdffe149c24f274418b7a12235dce680da80dea16ede4a

  PS C:\Users\M2M-NB-131> docker container ls -a
  CONTAINER ID   IMAGE                                        COMMAND                  CREATED          STATUS          PORTS                    NAMES
  16a0be4882c1   mcr.microsoft.com/mssql/server:2019-latest   "/opt/mssql/bin/perm…"   35 seconds ago   Up 35 seconds   0.0.0.0:1433->1433/tcp   local-dev-mssql-server
  PS C:\Users\M2M-NB-131>
```

### 3. SA 암호 변경

```powershell
  > docker exec -it local-dev-mssql-server /opt/mssql-tools/bin/sqlcmd -S > localhost -U SA -P 'm2makstp!' -Q 'ALTER LOGIN SA WITH PASSWORD="m2makstp!@"'
```

### 4. Container 진입
```powershell
  > docker exec -it local-dev-mssql-server "bash"
```

### 5. MSSQL 접속

```powershell
  > /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'm2makstp!'
```

## MSSQL에 들어갔을때 DATABASE 생성 및 사용자 계정 생성, 권한 부여 등에 대한 내용. 

### 6. DB 생성

```powershell
  1> CREATE DATABASE localTestDB
  2> GO
 
  1> USE localTestDB
  2> GO
```  


### 7. 사용자 계정 생성

```powershell
  1> CREATE LOGIN local_test WITH PASSWORD='m2makstp'
  2> GO
  
  1> CREATE USER local_test FOR LOGIN local_test;
  2> GO  
```

### 8. 계정 권한 할당

```powershell
  1> exec sp_addrolemember 'db_owner', local_test;
```
  

```bash
  mssql@16a0be4882c1:/$ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'm2makstp!'
  1> CREATE DATABASE localTestDB
  2> GO

  1> USE localTestDB
  2> GO
  Changed database context to 'localTestDB'.
  1> CREATE LOGIN local_test WITH PASSWORD='m2makstp'
  2> GO
  Msg 33064, Level 16, State 2, Server 16a0be4882c1, Line 1
  Password validation failed. The password does not meet SQL Server password policy requirements because it is not complex enough. The password must be at least 8 characters long and contain characters from three of the following four sets: Uppercase letters, Lowercase letters, Base 10 digits, and Symbols.

  1> CREATE LOGIN local_test WITH PASSWORD='m2makstp!'
  2> GO

  1> CREATE USER local_test FOR LOGIN local_test;
  2> GO

  1> exec sp_addrolemember 'db_owner', local_test;
  2> GO
```

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>