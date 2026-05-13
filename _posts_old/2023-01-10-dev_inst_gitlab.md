---
layout: single
title: "[GITLAB] Gitlab 설치"
excerpt: "GITLAB 서버설치 가이드"

categories:
  - tech
tags:
  - [git, gradle]

toc: false
toc_sticky: true

date: 2023-01-10
last_modified_at: 2023-01-10
---
# gitlab 설치 구축
 
 - 참조1 : https://blog.lael.be/post/5476

## 1. GIT 의 정의

- https://en.wikipedia.org/wiki/Git => 소스 버전 관리 시스템
- 모든 시점에 대해 저장을 하고, 코드의 병합(merge)이나 변경취소, 시점 복원등의 기능을 이용.
- 문제의 원인을 파악하기 쉬우며, 예상치 못한 소스 망실에 대해서 안전.
- GIT = 수동 백업 + 백업 시점 설명

### 권장하는 GIT 도입 시점 (다음 중 하나 조건이 충족하면 됨)

- 구성원이 GIT 사용에 능숙
- 프로젝트 착수 환경 준비에 1주일 이상의 기간이 할당된 프로젝트
- 구성원이 작업하는 파트가 독립된(겹치지 않는) 프로젝트
- 급하게 변하지 않으며, 장기적인 유지보수가 필요한(진행중인) 프로젝트 - 회사 상용솔루션
- 아이디어가 샘솟는 개인 프로젝트


## 2. GIT 서버 소프트웨어 선택
- GIT 을 사용하려면 소스를 제어하는 GIT 서버 설치.

### 이하 서버의 구축에는 심각한 주의 필요

>   
> 1) 메일서버 구축  
> 메일서버는 중단이 되면 안되는 서버.   
> 일반적으로 메일은 5대 이상의 서버가 계층을 이루며 서비스.  
> 대부분의 경우 메일서버 구축은 손해.  
> 도메인 업체에서 무료로 제공하는 메일이나  
> 구글앱스 같은 유료 서비스 사용을 권장.  
>   
> 2) 네임서버 구축  
> 네임서버는 메일서버보다 더 중요. 
> 무료 네임서버가 많으니 가능하다면 무료네임서버 이용 권장.
>  
> 3) 로컬환경에 GIT서버 구축  
> GIT 서버는 소스관리를 하는 것이기 때문에 중요.  
> git 프로젝트에서 git 서버가 죽으면 작업이 중단.   
> “로컬환경에 git 서버 구축”은 피하여야 함  
>  

### GIT 서버 소프트웨어의 종류 및 가격(*최저 옵션 기준)

- GITHUB Enterprise - https://enterprise.github.com - 10유저당 매년 250만원
- GITLAB - https://about.gitlab.com - 무제한 사용자당 무료
- Bitbucket - https://bitbucket.org - 10유저당 1만원(1회성 - one time fee)
 

## 3. GIT 서버 도메인 준비

- GIT 서버 소프트웨어가 결정 후, GIT 서버 도메인을 준비.
  - 보통 회사의 도메인이 example.com 이라면, git.example.com, dev.example.com, git-dev.example.com 사용

- 웹서버의 보안 통신은 https 를 통해서 이루어지며, https 의 필요조건은 도메인의 보유.
 
## 4. GIT 서버 소프트웨어를 설치할 서버 준비

- 서버 프로그램을 구동하기 위한 최소사양.
  - https://docs.gitlab.com/ce/install/requirements.html

- CPU 는 동시 접속자 (concurrent activation) 에 영향을 준다.
- RAM 은 동시 프로세스 실행에 영향을 준다.
- swap 이 설정된 2코어 2GB 서버를 선택하거나, 2코어 4GB 서버를 선택.

## 5. 생성한 서버와 준비한 도메인 연결

- 3번에서 준비한 GIT 서버 도메인과, 4번에서 준비한 GIT 서버를 연결.
- DNS 의 A 레코드에 서버의 IP 설정.(네임서버 제공업체에 문의)

## 6. GIT 서버 소프트웨어 설치
- 5번-도메인 연결 을 완료 한 후에 진행.

### 6-1. 서버 기본 세팅

- 서버 기본 세팅은 다음의 글 참조 : https://blog.lael.be/post/2600
- 0번(root 권한으로 변경) 부터 9번(Hostname 설정)까지 진행.
- 9번(Hostname 설정)을 진행할 때 hostname에 연결된 도메인 주소를 제대로 설정.

### 6-2. Gitlab 설치

- 공식 설치가이드 페이지 주소 : https://about.gitlab.com/install/#ubuntu

- 기본 폴더 이동

```bash
  #cd /root
```

- 필수 기초 소프트웨어 설치

```bash
  #sudo apt-get install curl openssh-server ca-certificates postfix
```

- 선택옵션이 표시시는 Internet Site 를 선택(또는 기본값을 사용).

- gitlab 설치 프로그램 저장소 추가

```bash
  #curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
```

- 저장소 목록 업데이트

```bash
  #apt-get update
```

- Gitlab Community Edition설치

```bash
  #sudo apt-get install gitlab-ce
```

- Gitlab 초기설정

```bash
  #sudo gitlab-ctl reconfigure
```

  - 서버에 문제가 있거나 업데이트시 위의 명령어를 실행해주면 정상화.

- 최고관리자 암호 설정

  - 최고관리자 ID 는 root 이다.
  - 암호를 설정.
  - 인터넷 브라우저를 열고, 연결된 도메인을 입력
  - GITLAB root 관리자 비밀번호를 설정한 후에 로그인 후 프로그램 설정.

![inst gitlab](./../../images/tech/gitlab-170208-7-545x400.png)

  - 가입 기능을 아예 끄거나, 가입 기능을 허용하려면 특정 도메인만 허용하고, 인증메일을 사용하도록 설정.
  - 가입 기능을 끄면 관리자에 의해 수동으로 계정을 만들어서 사용.

#### GITLAB 에 보안 인증서 추가 설치

- 무료 보안인증서 발행업체인 Let’s Encrypt 를 사용하여 GITLAB 에 보안 인증서를 추가.

```bash
  #apt-get install letsencrypt
```

```bash
  #mkdir -p /var/www/letsencrypt
```

```bash
  #vi /etc/gitlab/gitlab.rb
```
 
  - 설정파일 중간 쯤에 nginx 관련된 부분이 있는데 그 끝부분에 다음 추가.
  - (빠른 탐색 방법 : 명령모드에서 /logging 입력 후 엔터)

```bash
  nginx['custom_gitlab_server_config'] = "location ^~ /.well-known { root /var/www/letsencrypt; }"
```

![inst gitlab](./../../images/tech/gitlab-170208-8-768x568.png)

- 변경사항 적용 & 업데이트 확인

```bash
  #sudo gitlab-ctl reconfigure
  #sudo letsencrypt certonly -a webroot -w /var/www/letsencrypt -d gitlab.xxxxx.xxx
``` 

![inst gitlab](./../../images/tech/gitlab-170208-5-610x205.png)

- 위의 메일주소로 인증서 만료 직전 갱신알림메일 발송.
  - 약관 동의를 물어보면 A 를 입력.
  - 이메일 주소 공유를 물어보면 N 을 입력.

```bash
  #vi /etc/gitlab/gitlab.rb
```

- 발급된 인증서 gitlab에 적용.
- (빠른 탐색 방법 : 명령모드에서 1G )

- 서비스 주소 및 인증서 경로를 지정한다.

![inst gitlab](./../../images/tech/gitlab-170208-9-610x263.png)

(빠른 탐색 방법 : 명령모드에서 980G )

![inst gitlab](./../../images/tech/gitlab-ssl-1-768x648.png)

< 위의 부분을 찾아서 아래와 같이 변경한다 >

![inst gitlab](./../../images/tech/gitlab-ssl-2-768x623.png)

```bash
  nginx['ssl_certificate'] = "/etc/letsencrypt/live/#{node['fqdn']}/fullchain.pem"
  nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/#{node['fqdn']}/privkey.pem"
```

- 위의 코드가 적용되지 않는다면 수동으로 도메인을 강제 지정해 주어야 한다.

```bash
  nginx['ssl_certificate'] = "/etc/letsencrypt/live/gitlab.lael.be/fullchain.pem"
  nginx['ssl_certificate_key'] = "/etc/letsencrypt/live/gitlab.lael.be/privkey.pem"
```

- 변경사항 적용 & 업데이트 확인

```bash
  #sudo gitlab-ctl reconfigure
```

- SSL 인증서 자동갱신 설정

```bash
  #crontab -e
  10 5 * * 1 /usr/bin/letsencrypt renew >> /var/log/le-renew.log
  15 5 * * 1 /usr/bin/gitlab-ctl restart nginx
```

- GIT서비스에 재 로그인, HTTPS 인증서가 적용확인.
 

### 6-3. 방화벽 설정 (가이드)

- GITLAB에서 사용하는 TCP 포트는 80, 443, 22.
- 클라우드 서버를 사용중이고, 클라우드 방화벽을 사용할 수 있다면 해당 메뉴에서 적절한 IP 접근제한을 설정.
- ip 제한을 걸 경우 LetsEncrypt(SSL) 인증에 문제가 발생가능. 
  - 잠시 방화벽을 풀고 인증받거나 아예 80,443 을 public open 하여서 해결 가능.
  - Ubuntu의 경우 fail2ban 을 설정하여 비밀번호 로그인 몇회 오류시 접속을 차단.
    - https://blog.lael.be/post/1209.

### 6-4. 사용자 추가 (가이드)

- 가입을 받지 않게 설정 시 사용자 추가는 관리자가 직접추가.
  - root 계정으로 로그인 후
  - 새 사용자를 추가
  - 유저목록 -> Edit 버튼을 클릭, 비밀번호를 설정.
  - 해당 사용자에게 아이디/비밀번호를 통보. 
  - 사용자는 로그인 후 스스로 비밀번호를 변경.

## 7. 운영 및 유지보수

- 가끔 gitlab 관리자 아이디로 로그인시, “업데이트가 있으며 업데이트 해달라“.
- 이하 명령어를 사용최신버전으로 업데이트 가능

```bash
# apt-get update  
# apt-get upgrade  
```
  - 자동으로 git서버 중지 -> 업데이트 다운로드 -> 업데이트 패치 -> git 재시작.

- 참고 : gitlab 공식 업데이트 가이드 (https://about.gitlab.com/update/#ubuntu)

## 8. gitlab 백업 및 목원

### 8-1. 백업

```bash
  # sudo gitlab-rake gitlab:backup:create
```

- /etc/gitlab/gitlab.rb 파일의 gitlab_rails[‘backup_path’]  에 지정된 위치에 백업된다. (기본값 : /var/opt/gitlab/backups )
- 사이트 환경 설정, 사용자 정보, 저장소 정보를 포함한 모든 gitlab 정보 백업.

```bash
  # cd /var/opt/gitlab/backups
  # ll -h
```

- timestamp_gitlab_backup.tar
- 이렇게 백업이 생성 후.  이 파일을 안전하게 보관.


### 8-2. 복원

- gitlab 복원을 진행하면 [모든] 사이트 정보가 해당 백업파일 값으로 변경.
- 사이트 설정, 사용자 설정, 저장소 데이터를 포함한 모든 gitlab 정보가 해당 백업파일 값으로 변경.

#### 암호화키 복사

- 원본 서버에서 /etc/gitlab/gitlab-secrets.json 파일을 복사해서 복원할 서버에 COPY.
- 이 작업을 하지 않으면 웹UI 상에서 기존의 복원된 사용자 저장소에 대해 정보변경이 되지 않음.
- 프로젝트 정보 수정시, Whoops, something went wrong on our end. 같은 메세지가 나올 수 있다. 

#### 백업 폴더에 백업 파일 복사

- 백업한 서버와 복원할 서버가 같다면 (이미 이 파일이, 이 위치에 있기 때문에) skip.

```bash
  # cd /var/opt/gitlab/backups/
  # ll -h
```

#### 파일 이동 명령어

```bash
  # mv 해당백업파일.tar /var/opt/gitlab/backups/
```
 
#### 파일이 올바로 준비되었는지 확인

```bash
  # ll -h
```
![inst gitlab](./../../images/tech/gitlab-768x132.png)

#### DB를 사용하는 서비스 중지

```bash
  # sudo gitlab-ctl stop unicorn
  # sudo gitlab-ctl stop sidekiq
  # gitlab-ctl status
```

#### 복원하기

- _gitlab_backup.tar 를 제외한 파일 이름을 입력한다.

```bash
  # gitlab-rake gitlab:backup:restore BACKUP=1553573272_2019_03_26_11.9.0
```

- 복원 진행 중간에 정말로 복원 할 것인지 물어본다. yes 를 입력. 

#### gitlab 서비스 다시 재시작

```bash
  # gitlab-ctl restart
```

#### gitlab 서비스 상태 확인 및 교정 (check and sanitize)

```bash
  # gitlab-rake gitlab:check SANITIZE=true
```

- 잘못된 부분(빨간색 글자)이 있다면, 위의 상태 확인 명령어를 다시 실행.

#### 재부팅
- # reboot

#### GITLAB 디버깅 로그 파일 위치
- /var/log/gitlab/gitlab-rails/production.log

## 오류대응

### push 시 이하 오류발생의 경우(큰용량 파일 push?)

> fatal: the remote end hung hup unexpectedly  
> send-pack: unexpected disconnect while reading sideband packet  

```bash
  git config --local http.postBuffer 1024M
  git config --local http.maxRequestBuffer 1024M
  git config --local pack.deltaCacheSize 1024M
  git config --local pack.packSizeLimit 1024M
  git config --local pack.windowMemory 1024M
  git config --local core.packedGitLimit 1024m
  git config --local core.packedGitWindowSize 1024m
  git config --local core.compression 9
  git config --local ssh.postBuffer 2048M
  git config --local ssh.maxRequestBuffer 2048M

  // 확인
  git config --local -e
```

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>