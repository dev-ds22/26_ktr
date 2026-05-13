---
layout: single
title: "[SERVER] Dev. Server 세팅"
excerpt: "CentOS7 설치 후 설정 및 JDK11, 기타 SW설치 및 설정"

categories:
  - tech
tags:
  - [centos7]

toc: false
toc_sticky: true

date: 2022-12-29
last_modified_at: 2022-12-29
---
## 1. Linux

    CentOS-7-x86_64-Minimal-2009.iso

- x86_64 
  - AMD의 라이선스를 받아서 구현한 Intel64의 차세대 아키텍처. 
  - 주로 개인용 컴퓨터에서 사용되는 프로세서(CPU), 64bit 운영체제를 지원. 
  - 32bit 운영체제에서는 애뮬레이터가 구동, 속도 저하 발생.
- i386
  - Intel에서 설계한 개인 컴퓨터용 마이크로 프로세서. 32bit 아키텍처로 설계.
- IBM Power BE (ppc64)
  - 서버, 마이크로컴퓨터, 워크스테이션, 슈퍼컴퓨터에서 사용, 64bit 운영체제를 지원.
- IBM Power (ppc64le)	
  - IBM Power BE와 동일, 32bit 운영체제를 지원.
- ARM64 (aarch64)
  - 주로 임베디드 기기에 사용되는 프로세서, 모바일 및 싱글보드 기기에서 사용되는 프로세서. 
  - 64bit 운영체제 지원.
- ARM32 (armhfp)
  - ARM64와 동일, 32bit 운영체제를 지원

### 1-1. CentOS-7-x86_64-Minimal-2009.iso 다운로드
### 1-2. Rufus 로 부팅USB 제작
  - 파티션 : GPT 로 작성
  - BitLocker 적용해제 필요
### 1-3. USB 부팅 후 CentOS 설치
### 1-4. USB Mount
  - su
  - fdisk -l
  - mount /usb/sda1 /tmp/usb
  - cp /tmp/usb /tml/setfile
### 1-5. 네트워크 설정





## 2. JDK

    java-11-amazon-corretto-devel-11.0.17.8-1.x86_64.rpm

## 3. Nexus
## 4. Gradle
## 5. Jenkins
## 6. Gitlab
## 7. NginX
## 8. Oracle

    V17530-01_1of2.zip
    V17530-01_2of2.zip


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>