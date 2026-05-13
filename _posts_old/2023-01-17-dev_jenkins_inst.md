---
layout: single
title: "[Jenkins] jenkins 설치 및 자동배포"
excerpt: "CentOs 환경 jenkins 설치 및 자동배포 연결과정 정리"

categories:
  - tech
tags:
  - [jenkins]

toc: false
toc_sticky: true

date: 2023-01-17
last_modified_at: 2023-01-17
---

# jenkins 설치 및 자동배포

## 1. jenkins 설치

### 1-1. jenkins repository를 설정파일 생성

```bash
  $ wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
```

### 1-2. /etc/yum.repos.d/jenkins.repo경로에 파일 존재 확인 후 아래 실행

```bash
  $ rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```
 
### 1-3. yum으로 jenkins 설치

``` bash
  $ yum install jenkins
```

> 이하오류 발생 시  
>   
> Errors during downloading metadata for repository 'gitlab_gitlab-ce':  
>   - Curl error (35): SSL connect error for https:~~~~  
>   
> /etc/yum.repos.d 에서해당하는 repo 파일을 지워준다.  

```bash
  설치되었습니다:
    jenkins-2.375.2-1.1.noarch

  완료되었습니다!
```

### 1-4. 기본 포트(8080)을 9090으로 변경

```bash
  $ vi /etc/sysconfig/jenkins

  ...
  #JENKINS_PORT="8080"
  ## 위 내용을 주석처리하고 아래와 같이 9099으로 변경
  JENKINS_PORT="8099"
  ...

```

> 현재 사용중인 포트 확인하기  
>   
> netstat -tulpn | grep LISTEN   

### 1-5. 방화벽 오픈

```bash
  firewall-cmd --permanent --zone=public --add-port=8099/tcp
  firewall-cmd --reload
```

### 1-6. 서비스 시작

- Jenkins를 systemctl에 등록해주고 실행을 시킵니다. 

```bash
  $ sudo systemctl daemon-reload
  $ sudo systemctl start jenkins
```

#### 자동시작할 경우

```bash
  $ sudo systemctl enable jenkins
```

#### 에러 CASE 1.
- systemctl status jenkins 명령을 이용하여 오류 부분을 상세히 확인.

```bash
  $ systemctl status jenkins

  /etc/init.d/functions : No such file or directory
```

- /etc/init.d/functions 모듈설치.

```bash
  $ yum install -y initscripts
```

#### 에러 CASE 2.

```bash
  The unit jenkins.service has entered the 'failed' state with result 'exit-code'.
  1월 17 13:40:36 localhost systemd[1]: Failed to start Jenkins Continuous Integration Server.
``` 

- 원인1. : 포트가 충돌나서 발생. 
  - /etc/sysconfig/jenkins 경로의 포트를 8080에서 8099으로 변경.
  - /usr/lib/systemd/system/jenkins.service 파일도 수정필요.

  - 해결책 : jenkins.service 파일의 Environment="JENKINS_PORT=9090"로 변경

- 원인2. : 포트가 충돌나서 발생. 

```bash
  localhost jenkins[3385779]: jenkins: invalid Java version: openjdk version "1.8.0_352"
  localhost jenkins[3385779]: OpenJDK Runtime Environment (build 1.8.0_352-b08)
  localhost jenkins[3385779]: OpenJDK 64-Bit Server VM (build 25.352-b08, mixed mode)
  localhost systemd[1]: jenkins.service: Main process exited, code=exited, status=1/FAILURE
```

  - https://www.jenkins.io/blog/2022/06/28/require-java-11/
  - 2022.6월 이후 jenkins 버전에서는 java 11필요.

- 기존에 설치 되어 있던 jenkins 를 삭제

> yum remove jenkins  

```log
  제거되었습니다:
    jenkins-2.375.2-1.1.noarch

  완료되었습니다!
```

- 이전버전 rpm 파일 다운로드

> wget https://get.jenkins.io/redhat-stable/jenkins-2.319.1-1.1.noarch.rpm  

- rpm 파일 다운 받은 파일을 설치

> yum install jenkins-2.319.1-1.1.noarch.rpm   


- demonnize 모듈이 없어 install 필요

> vi /etc/yum.repos.d/daemonize.repo  

  - 아래내용 입력, 저장

```bash
[daemonize]
baseurl=https://download-ib01.fedoraproject.org/pub/epel/7/x86_64/
gpgcheck=no
enabled=yes
```

  - 저장이 완료후 demonize 모듈을 설치한다

  > yum install daemonize -y    
 

  - jenkins를 설치.

#### JAVA 11설치 시 

- amazon-corretto-11 설치 시 

```bash
  sudo rpm --import https://yum.corretto.aws/corretto.key 
  sudo curl -L -o /etc/yum.repos.d/corretto.repo https://yum.corretto.aws/corretto.repo
  sudo yum install -y java-11-amazon-corretto-devel
```

```bash
  $ alternatives --config java

  [root@localhost /]# alternatives --config java

  There are 2 programs which provide 'java'.

    Selection    Command
  -----------------------------------------------
  + 1           java-1.8.0-openjdk.x86_64 (/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.352.b08-2.el9.x86_64/jre/bin/java)
  *  2           /usr/lib/jvm/java-11-amazon-corretto/bin/java
```

- port 수정

> vi /usr/lib/systemd/system/jenkins.service  
>  
> #해당 부분을 찾아 원하는 포트로 수정  
> Environment="JENKINS_PORT=8099"   
>  
> # JAVA 설치 버전이 2개라면 아래 부분의 주석을 제거하고 위에서 복사한 경로로 수정  
> #Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64"  
> Environment="JAVA_HOME=/usr/lib/jvm/java-11-amazon-corretto/bin/java"  


### 1-7. 초기 어드민 패스워드 입력 

  - /var/lib/jenkins/secrets/initialAdminPassword 에서 확인

### 1-8. 플러그인 설치

  - 추천 플러그인 선택

### 1-9. 어드민 계정 정보 입력
  계정명 : admin
  암호 : m2~~~~~
  암호확인 : m2~~~~~
  이름 : m2m
  이메일주소 : m2m

## 2. 빌드 및 배포설정  

### 2-1. JDK 환경설정

  - jenkins에 admin 계정으로 로그인 후 'Jenkins 관리' 탭 클릭

### 2-2. GITLAB 환경설정
#### GITLAB Access Token 생성
- Gitlab 접속
- 연동대상 프로젝트 선택 - 설정 - 액세스 토큰 선택
  - Token name : token 의 용도.
  - 만료일 : token의 사용기한. 미입력시 무한.
  - 역할 : 유지보수 > 개발자 > 보고자 > 손님 > 비멤버 순 권한할당.
  - select scopes : 권한 부여, 모두 선택.
  - create project access token 으로 token이 생성.

  > glpat-5qGmHru-CNDs-q1-****  

![inst gitlab](./../../images/tech/git_lab_access_token_011.png)

#### jenkins 에 GITLAB 플러그인 설치
- jenkins에 admin 계정으로 로그인 후 'Jenkins 관리' 탭 클릭  
- '플러그인 관리' 클릭
- 'Available plugins' 탭 선택
- GitLabVersion, 1.6.0 - Build Triggers 선택 후 install without restart 클릭
- Download progress 에서 Success 확인

#### gitlab 계정 연결
- Jenkins 관리 - Manage Credentials 선택
- Domains의 (global) 클릭
- Global credentials (unrestricted) 화면에서 adding some credentials 클릭
- New credentials 에서 credentials 생성
  - UserName : gitlab 계정 아이디
  - Password : gitlab 계정 비밀번호
  - id : Credential을 식별하는 아이디
  - Description : Credential에 대한 설명

#### Build Server SSH키 생성

```bash
[root@localhost .ssh]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:diE22f79m2O9LuH/ouFsNAJTQduxqweT1yuyWQLH8P4 root@localhost
The key's randomart image is:
+---[RSA 3072]----+
|         .o..    |
|         o.o o   |
|        *.+ o    |
|       .oB o o   |
|        SoX o .  |
|       . =.*oo . |
|          =oB.+ .|
|           @.=.+o|
|          o.E.=BB|
+----[SHA256]-----+
[root@localhost .ssh]# ls
id_rsa  id_rsa.pub
[root@localhost .ssh]# ls -lt
합계 8
-rw------- 1 root root 2602  1월 18 10:29 id_rsa
-rw-r--r-- 1 root root  568  1월 18 10:29 id_rsa.pub
[root@localhost .ssh]# cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCzcftAu9g6jI/0Pq8wPsOMUcc2b7E46nRGUGny/yODD7qqH39k0Lr3OKEa877Lytff4oxDzuAK+HNM+crs3n0QJCh0O/nwvGkSvR15eK2WLEzuaF4g5ACjvmL3IJs/aqJMRrJ7Q3xtrITrdG41YN/hSC8Bm8qdrcxi+GeVUnYNTjfyLQsZpiovfBb0jG51oWZ2MsXZA8jQHsg9CchjWU9XiZrsXC1rE702k57Hnfy5HwwqVXYodtTrWCvfsfdrdRoBb5JFDoVtzN1VNoIy03EoyAXX5nZ6j/GPPWEpW/+1b5OzTF77aho3b2N4gwkSZ0mJ4RUPfCEOQVaPiBMzjhW2zsucjSY1X68VL7tyRy6guRqtEeLhb+PjkQHX1mMPRlniT24f7c5DiQJB/+HLZYrB79S+8dkcD++V/TWcFRPTJV2UpxInLFAEf8n38DMunw1zXrlLf9HnnE2JJEGrKz0CyNbLmzbhvqmh7ltyQBcYjpBB0gD2sIzFEJjof9TOfD8= root@localhost
```

#### Git Server에 인증키 파일생성.
- Git 서버에서 만들어야합니다.

```bash
cd
mkdir .ssh
vim authorized_keys
```

- jenkins의 키값을 붙여 넣기 하고 파일을 닫기.

> 참고 : Publish over SSH  
> - Publish over SSH 플러그인이 2022.01.12. 이후 젠킨스 보안 정책으로 인해 배포중단.  
>   - https://www.jenkins.io/security/advisory/2022-01-12/  
> - 필요할 경우 직접 .hpi 확장자를 다운로드 하여 설치필요.  
>   - https://archives.jenkins-ci.org/plugins/publish-over-ssh/latest/ 에 접속.  
>   - publish-over-ssh.hpi 파일 다운로드.  
>   - Jenkins 관리 -> 플러그인 관리 -> 고급 -> 플러그인 올리기 으로 이동.  
>   - 다운로드 파일첨부 후 올리기 버튼 클릭, 수동설치 진행가능.  

#### Jenkins GitLab Connection 설정

- Jenkins 관리 > 시스템 설정 > GitLab에서 GitLab connections를 설정.

- Connection name : 식별할 수 있는 값 입력
- GitLab Host URL : Gitlab 서버 IP입력.(상용 서비스일 경우 Gitlab의 도메인 주소).
- Credentials : 위에서 생성한 Credentials를 등록
- Test Connection > Success 확인되면 SAVE


##### Jenkins 서버 SSL설정 
```bash
## 개인키 생성
## 2048bit RSA 암호화 인증서
[root@localhost ~]# openssl genrsa -des3 -out server.key 2048
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:

## 인증요청서 생성(CSR)
[root@localhost ~]# openssl req -new -key server.key -out server.csr
Enter pass phrase for server.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:KR
State or Province Name (full name) []:seoul
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:m2makstp
An optional company name []:m2m

## 개인키패스워드 제거
[root@localhost ~]# cp server.key server.key.origin
[root@localhost ~]# openssl rsa -in server.key.origin -out server.key
Enter pass phrase for server.key.origin:
writing RSA key

##인증서 생성(개인키와 서버요청서 이용하여 인증서 생성)
[root@localhost ~]# openssl x509 -req -days 3650 -in server.csr -signkey server.key -out server.crt
Certificate request self-signature ok
subject=C = KR, ST = seoul, L = Default City, O = Default Company Ltd

## server.crt(인증 요청서)와 server.key(개인키)를 이용하여 jenkins.pfx 생성
[root@localhost ~]# openssl pkcs12 -export -in server.crt -inkey server.key -out jenkins.pfx
Enter Export Password:
Verifying - Enter Export Password:

## jenkins.pfx 확인방법
[root@localhost ~]# openssl pkcs12 -info -in jenkins.pfx
Enter Import Password:
MAC: sha256, Iteration 2048
MAC length: 32, salt length: 8
PKCS7 Encrypted data: PBES2, PBKDF2, AES-256-CBC, Iteration 2048, PRF hmacWithSHA256
Certificate bag
Bag Attributes
    localKeyID: 1F C3 8E AE 4A 1E CD 8D E3 6E 9F 2E 78 73 37 E0 1D 4B BF 8C
subject=C = KR, ST = seoul, L = Default City, O = Default Company Ltd
issuer=C = KR, ST = seoul, L = Default City, O = Default Company Ltd
-----BEGIN CERTIFICATE-----
MIIDKzCCAhMCFCIB6GGlrHjnofVYjetaS9e+mmT7MA0GCSqGSIb3DQEBCwUAMFIx
CzAJBgNVBAYTAktSMQ4wDAYDVQQIDAVzZW91bDEVMBMGA1UEBwwMRGVmYXVsdCBD
aXR5MRwwGgYDVQQKDBNEZWZhdWx0IENvbXBhbnkgTHRkMB4XDTIzMDExODA0Mzgx
MVoXDTMzMDExNTA0MzgxMVowUjELMAkGA1UEBhMCS1IxDjAMBgNVBAgMBXNlb3Vs
MRUwEwYDVQQHDAxEZWZhdWx0IENpdHkxHDAaBgNVBAoME0RlZmF1bHQgQ29tcGFu
eSBMdGQwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCli2hSBoNBOCia
ZV1fMBFVLt7RvuJSIV8iwsN/aEnY+dGk760odesBX0wFGj4H7Yn+2WnW+UyzzmRn
8tvWOO9cwjpeBK2JwsV1zelo45NhAOseruNvJuFSQFfHIhyJEOjZdBG3NjWh6Vg9
Toefj8pukXWflOiBp4t+ph8paDp7usS/oXMnSWC5G3egmtxB3f3IphyzaWUHoXnG
B6WO4vRoi1IbRmrnjWH1/HQGf8UGGP2jQcCpqDzpueTIfbgMiLrWJKQfVkPuvPFx
ldDdcRETjWkxmclPg6nur2KM1nKQ+/50PSwQTy7YsU8mALHLDm4KJTsT6cuvOWEv
k7E+8fqZAgMBAAEwDQYJKoZIhvcNAQELBQADggEBAHzvRqHLlXWYfO8QX9pKlqvD
FsK4tp9fOBPkLCwsUf/n/Hp/fh15bet5ta7ydFQleZxsZp7VZ32WoKyfwfEOZD37
ZVGkuSPq16IqvthzsGYZDlUEP9eHl7T8NFYwq2M6IeSGeG1hg2aXvLplPSI9UhMl
APkjYfaR5aTGJrFYxbmWwdf8+IryGEH5dSa6JIX4f1y7tOILjjvECB4pGgT7Qgky
k6BR+smqx4wTmyvuckyQJ/6WUDWczXSTifZ2I+g1IpY7MzUm8PqqXpo1MFeh9Y09
geyBTfUhTSgBubgjOp0WQtpyH+87ZNY1e8o1xIhNT/dqdQocrUJ2gc1f6rOcio4=
-----END CERTIFICATE-----
PKCS7 Data
Shrouded Keybag: PBES2, PBKDF2, AES-256-CBC, Iteration 2048, PRF hmacWithSHA256
Bag Attributes
    localKeyID: 1F C3 8E AE 4A 1E CD 8D E3 6E 9F 2E 78 73 37 E0 1D 4B BF 8C
Key Attributes: <No Attributes>
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIFLTBXBgkqhkiG9w0BBQ0wSjApBgkqhkiG9w0BBQwwHAQIkNmF8H/HTvYCAggA
MAwGCCqGSIb3DQIJBQAwHQYJYIZIAWUDBAEqBBB2SsvroWvBo3tJT6TDyU8bBIIE
0MQZMs8XaRMjObtdOvVFs9bX3dMKBscUYPKwhnEEZFs+96cs/DlcSB87bEz9Sh3L
kZ1JyCHpYeIQDmWGjCsBuHwwL/HxVWkzd2MX1N8xzEX1e8011QRtmmNwNtZYiJKE
u5aHw8e+DZOo/wQ3PQxc6kS3CPJEszixsARsSl7d/BTzfDnSiA7Rux7wcfh6pQ6j
Bq/i9CwfNpJQuukDqX11uExTvYxknm7CBIoQkOFL6pIp/OSnOK4AgJWo05Fhc6Wv
2PXYUqM+s77urEGazWONxxM34prAslrTsgoDmvPMxPlHlFhgKsGlQLVHWqNxtj2Q
dN6YWWsKAo3EP2nVMevpCAVU8hj31IEjqTXLNrlCiGp0dQfa8XFftJ3ND1FUnkd1
o1DDKiOQvFBn+7zSIFkNf8/JW3+IxnnWWD3fIflxnRfsnw6HCGfKX+IB8bAYqhbw
sYZhussOpZiQL+3qL0AkKMD4evXJ0KbRVIUcET/AW6KSy5g2tUY8lkuCl5Qb+JbT
+AQ1Yumz/pvUIrSKleyBbkD2xwXxTTAEIo1j/KuE4uEivepThEAkFcmGHsgTw1oX
t7ZX4uRmR91g7C59UXS+27QbnGPdmiWw1MGc+hxREEuXTABgT3ak0/zF4e0lGAps
0duHXTi95+iqmyWdwLJlI73GYQcZbxMfqeWrOYXd3VzyGT5wqIcAcapiwyxV9XvV
OSqQa1ntHZexmuck1ngA2dl1e16biSPOGuU2nDm0EEGUCwvfIbwtjmlTCN6dNgOk
PmnHyQDIKJEGvsMI0/xo50S7/7rmT+Mzm9Rj9Fam6T996k2vHspkjVAlQNG9x1Mt
Xadj3go5C9QfJdLeB8FD2kL0+JcuKFbI05e7ZOsHf04RzlAJwHNwvZztc/ffdObO
HDYz/JZT5VW5IyIKPqjr5snJ8KoNsBXQy3E66MxT24sXHWCNqouJEqAQB0jEzU+K
gIgGKiLzn/J94oobIQSYwnw+DLN1Nyc9h6E1X1cm0YjAdSp8pq70IO1HRknfDU/O
FXravJcUh8DI7KGxPBbHIvSlATDc2IO5h6W0qHssuuK13O8Ofo0ybnrmWevvfbwp
vW+QSYU16AbYLXNPxOdItWEnukXddMR1yAbiXH0u4YAQdzp2slAGk77iw99el3Yv
z5KT/t3IWjFIOsBvFtnsbToxfAPh85/ozBrCCCpdyl8cM3QBo3W+5E6Ks8oHEBhn
LFNs8IAtBWixP8Rd4lCEFBBeUN8hMtR+18qkCztcGi9V0GP29+AcLROC+8yt6dQO
/wAENno0IJBEZ7kO07Xf8ikFAMeHSmaZHREISjSag7o0z7EZtxPjNmNlXdkV4Z1y
9Yg/IYy9ps/bubJSRGOaN2Gap54z7rYt1ENWxHgNHdy7fj13NEF9wN2M3VHhddNd
jkUE+K7pUm6JfTeZxBL2HGN0wA0FJ9h9IqhenJmBDhvou1Chu85Wluwi+uHXLf9a
nzyDpRqL/nlZsT60dOr5iyj0n0fQrVHDuwzcKYcty6GPX51EIUftC3f2t7Ay2kLw
anlsavAMZ8GKLRQYLvbzn7E7/VL2IplCpHZa+m1kFToMy5mLbAktoPJsMIIkwWwu
fCYxoT/idBeIOUfQlpO7p3ioWfrfPPm0GsRNx6PFJzZi
-----END ENCRYPTED PRIVATE KEY-----

## jenkins.pfx > jenkins.jks로 변환
[root@localhost ~]# keytool -importkeystore -srckeystore jenkins.pfx -srcstoretype pkcs12 -destkeystore jenkins.jks                                                  -deststoretype jks
키 저장소 jenkins.pfx을(를) jenkins.jks(으)로 임포트하는 중...
대상 키 저장소 비밀번호 입력:
새 비밀번호 다시 입력:
소스 키 저장소 비밀번호 입력:
1 별칭에 대한 항목이 성공적으로 임포트되었습니다.
임포트 명령 완료: 성공적으로 임포트된 항목은 1개, 실패하거나 취소된 항목은 0개입니다.

Warning:
JKS 키 저장소는 고유 형식을 사용합니다. "keytool -importkeystore -srckeystore jenkins.jks -destkeystore jenkins.jks                                                  -deststoretype pkcs12"를 사용하는 산업 표준 형식인 PKCS12로 이전하는 것이 좋습니다.

## https 포트추가 및 인증서(jks) 설정
[root@localhost ~]# vi /usr/lib/systemd/system/jenkins.service

## 주석 해제하고 아래와 같이 설정
Environment="JENKINS_HTTPS_PORT=8443"
Environment="JENKINS_HTTPS_KEYSTORE=/data/jenkins.jks"
Environment="JENKINS_HTTPS_KEYSTORE_PASSWORD=비밀번호"

## 젠킨스 서비스 재시작
[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl restart jenkins
```


git config --global http.sslCAInfo /root/server.crt



6. 인증 키 생성

   ssh-keygen -t rsa -f id_rsa

   id_rsa, id_rsa.pub 두 파일 생성

- id_rsa

```bash
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

- id_rsa.pub

```bash
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDRTvgmGKsuPIcGtbZd9CsdutNyPRkw6+urNDLPeddYXF/mee3wjKkwaLHaBZKOZdLYnShr8ltlh9VJiRwDQ/1dBThp4l4r4PlpUQtsTPwwsZSaw6etq99EB/VFLTOS7tellRsnNl6XvrBHCzer8bwK0ER/TeaU3cpywJGfSQc0vFopoRD9wX1QZhSXLbRbwffaF013YOJkrolVa8xF6axJnKdHn/xr+7ioV4oMzqoLU2cEAW4PnUQOij+UbaLS7XxTM5S69xFH1oOH+1k5IRbfv7qkv/0bHQDCd1cEPRGUMhzN34i2NV8FGljj1rWgvw+muVo9e1IiPqrA5gjAmbkR/5YFQw1oXvSRNOh2t1XvI5xx0CSvkvvfl4MZ+pRMp4YV+pWmoS895apxMWIIGDB8WWouHHmAHFqwgnkMM+pG6U+csHu0UJPieuikYSEJMxN8ys98cRHYqltAahR/hbjQJiY4TwVtjUy/op0nlRsw9Y2qMSwWayB2+OlJNHaRK7CRkhGRxbuJLpDkkgGAyyBko9j5jOjIBO7fVku1cnuxTPVAJt3TS/1TiIPOClLCFWs5+1nT7TkOtYLBFrLwpkZeSNYxr0odDAVXVxSB+C2SBJSxBQW8MWC2OZo/2QnnnJYRwTbM7zK/JKSofGlQbmnNKKTiOPAi9hYjF4H2sNU62Q== wontae_k@m2mglobal.co.kr
```

7. id_rsa 파일 젠킨스 설정

   cat id_rsa 로 키 복사

   credentials -> system ->Global credentials -> Add credentials -> Kind : SSH Username with private key

8. id_rsa.pub 파일 깃랩 설정

   사용자 -> Settings -> SSH Keys에 설정


##### gitlab 웹훅(webhook) Internal Error 500

최신 버전 GitLab 에서 같은 서버에있는 CI(Jenkins) 에 webhook 을 요청시에는
Internal Server Error 500 이 나타납니다.
일단 해당 이슈는 ssrf 관련 이슈이기 때문에 gitlab ssrf 라고 검색 하여도 충분한 결과를 얻을수는 있을 겁니다.
ssrf이슈인지 파악이 안됐었.. ㅜ



해당 내용 설명에 앞서 ssrf에 대해서 간단하게 작성하자면

SSRF ( Server - Side - Request - Forgery ) 서버 요청 위조 
CSRF ( Client - Side - Request - Forgery ) - 클라이언트 요청 위조

SSRF는 CSRF 와 달리 서버가 직접 호출해서 발생하는 문제입니다.  서버가 직접 요청하는 바람에 외부에서 내부에 있는 리소소스 등 접근해서 서버 직접 제어가 가능해지기 때문에 해당 의 경우 SSRF의 위험성이 있기 때문에
Jenkins 서버와 GitLab이 같은 서버에 위치할때 루프백 (자기자신) 으로 요청하게 되면 SSRF 가 발생하게됩니다.
gitlab webhook 은 기본적으로 내부 서버 요청을 서버가 직접 하는것들을 SSRF 이슈로 인해 허용하지 않기 때문입니다.
그래서 기본적으로 gitlab 생성후 webhook 을 내부 같은 서버에 있는 jenkins(CI)에 호출하게되면 500 Internal Server Error 가 발생합니다.


#### [Jenkins] Gradle 을 이용한 submodule update 이슈
- Jenkins 에서 Gradle 을 이용한 multi project(submodule) build 시 참고사항
  - 젠킨스 소스 관리영역에서 Additional Behaviours 설정을 추가 해줘야 합니다.
	- Advanced sub-modules behaviours

		여기서 총 세 가지 항목을 선택 하게 됩니다.
			- Recursively update submodules
				이 설정은 --recursive 옵션을 사용합니다.

			- Update tracking submodules to tip of branch
				이 설정은 --remote 옵션을 사용합니다.

			- Use credentials from default remote of parent repository
				이 설정은 parent project 의 credential 을 사용 한다는 의미 입니다. 

- submodule 이 업데이트 되지 않는 문제발생 시 . 2번째 옵션을 선택필요.

	- Update tracking submodules to tip of branch
		이 옵션의 의미는 submodules 의 변경된 내용을 업데이트 하기 위한 것.

#### gradle 명령어 및 Gradle build 문제 해결
- build시 --warning-mode=all (-Dorg.gradle.warning.mode=all)옵션(Program Argument) 사용시 상세 설명 출력 (option 끝에 넣어야 동작)

###### 멀티프로젝트 빌드 옵션

- gradle :api:build : api와 api가 의존하는 모든 프로젝트에 대해 컴파일과 jar를 수행하고 api 프로젝트의 build를 수행한다.
- gradle -a :api:build : api 프로젝트의 build만 수행한다.
- gradle :api:buildNeeded : api와 api가 의존하는 모든 프로젝트의 build를 수행한다.
- gradle :api:buildDependents : api와 api에 의존하는 모든 프로젝트에 대해 build를 수행한다.
- gradle build : 모든 프로젝트에 대해 build한다.

###### Gradle build 문제 해결

- gradle build시 lombok 에러 발생
  compileOnly 'org.projectlombok:lombok'
  annotationProcessor 'org.projectlombok:lombok'
  testCompile 'org.projectlombok:lombok'
  testAnnotationProcessor 'org.projectlombok:lombok'

- main class 못찾는다는 에러 발생
Execution failed for task ':bootJar'.
> Main class name has not been configured and it could not be resolved
bootJar {
  mainClassName = 'org.syaku.blog.Application'
}

- 실행할 메인 클래스가 없을때는 아래와 같이 한다
bootJar.enabled = false

- Gitlab Webhook으로 Jenkins 빌드 유발하기
  - https://zunoxi.tistory.com/106

- webhook 사용 자동빌딩(배포) 시, 특정 브런치명만 실행하기(CI/CD 옵션)
  - https://lemontia.tistory.com/1054  


##### Jenkins 유저로 전환

```bash
 su - jenkins -s /bin/bash

sudo chown -R jenkins /usr/local/lib/node_modules/
sudo chown -R jenkins /usr/local/bin/
sudo chown -R jenkins /usr/local/share/
```

#### .gitmodules 파일에 서브모듈 정보 확인

- ssh(git@~~~~) 이용 안하고 https (https://~~~~) 이용시 jenkins에서 접속안될수 있음

```
[submodule "diso-common"]
	path = diso-common
	url = git@gitlab.diso.com:pilot/diso-common.git
[submodule "diso-bo"]
	path = diso-bo
	url = git@gitlab.diso.com:pilot/diso-bo.git
[submodule "diso-fo"]
	path = diso-fo
	url = git@gitlab.diso.com:pilot/diso-fo.git
[submodule "diso-mo"]
	path = diso-mo
	url = git@gitlab.diso.com:pilot/diso-mo.git
[submodule "diso-batch"]
	path = diso-batch
	url = git@gitlab.diso.com:pilot/diso-batch.git
```


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

# FrontOffice
```bash
npm install --force &&
npm run build &&
pm2 stop -s front-end || :
pm2 delete -s front-end || :
set DEBUG=express:* & pm2 start npm --name front-end -- run start
```

# BackOffice
```bash
#!/bin/bash
echo "PID Check..."

CURRENT_PID=$(ps -ef | grep java | grep PlatformCommon* | awk '{print $2}')
JAR_PATH=/var/jenkins_home/workspace/Dev-PlatformCommon/build/libs/PlatformCommon-0.0.1-SNAPSHOT.jar

echo "Running PID: {$CURRENT_PID}"

if [ -z "$CURRENT_PID" ]
then
  echo "Project is not running"
else
  kill -15 $CURRENT_PID
  sleep 10
fi

echo "Deploy Project...."

nohup java -Dspring.profiles.active=m2mdev -jar $JAR_PATH >> /var/jenkins_home/workspace/nohup.out &

echo "Done"
```

  </pre>
</details>