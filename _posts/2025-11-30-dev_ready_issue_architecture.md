---
layout: single
title: "[Tech Stack] ETC - Technical Stack"
excerpt: "아키텍처 고려사항"

categories:
  - tech
tags:
  - [tech]

toc: false
toc_sticky: true

date: 2022-11-30
last_modified_at: 2022-11-30
---
# 아키텍처

## Technical Stack  
### 1. User  

- PC, MO, App(hybrid)  
- View Template Engine : Thymeleaf  
- Front End Framework : Vue.js  
- BO : Toast UI Grid(NHN, MIT License)  
    
### 2. Transmission Tier  

- Router s3
- CloudFront
  - HTTP → HTTPS : 쉬운 설정으로 http 로의 접속을 https로 리디렉션 가능.  
  - CDN을 통한 더 빠른 페이지 응답속도 : S3으로만 배포하는 경우, 선택한 리전 내에서만 생성, 해당 리전에서 멀어질수록 접속속도 저하.    
  - CloudFront 를 이용하면 전 세계에 분포된 엣지 로케이션이라고 하는 데이터 센터의 엣지 서버를 사용해 콘텐츠를 캐싱.
  - 사용자가 위치한 곳에서 가장 가까운 엣지 로케이션에서 콘텐츠를 제공.  
    
### 3. Web Tier  

- Web Cache(STON) : Web Cache, 이미지 Cache  
    
### 4. Interface Tice  

- Redis - Cache 및 서비스 현황 기록  
- API Developer Tool : Swagger  
    
### 5. Service Tier  

- FO, BO/API, MO, Batch, Elastic Search

### 기타 Solutions  

- FO  
  - 개인화추천 (?): 개인화 추천검색, 상품/행사/이벤트 추천  
  - 검색 사전작업(STT, QR, Barcode)  
  - 검색시스템(ElasticSearch) : 통합검색, 수정/인덱싱  
          
- BO  
  - 메시지관리, 이미지관리. Short URL관리  
  - 통합메시징(TMS - 통합 메시지 채널 최적화 솔루션)  
  - 이미지 리사이즈 시스템(DIMS, Dynamic Image Management System)  
    - 원래의 이미지를 다양한 형태로 가공하는 기능, STON에서 제공  
    - 라이브러리 imgscalr로 대체?  

- 기타 - JIRA-JENKINS : 요청관리, 형상관리, 배포관리, 장애관리  
  - 웹로그분석 : Google Analytics  
  - 이미지캐시 : STON  
  - 로그관리 : ELS(ElssticSearch, Logstash, Kibana)  
    
### 6. Data Tier  

- S3(Static, Logs)  
- RDBMS  
- Redis - 캐싱 서비스(Session, Object, Sequence Cache)  
    
### 7. Internal / External  

- RestApi  
- OXM(Object to XML) Framwork : Jackson Framework  
- Batch : Spring Batch  

### 그 외  

- IDE : SpringToolSuite
- JDK : 11
- Nexus 
- Swagger ( API 개발 )
- Codecommit
- Lombok
- MyBatis

## MQ  

- Kafka 
  - 고용량 데이타 및 높은 처리량이 요구될 때 사용
  - 동일한 메시지에 대해 복수개의 소비자가 필요시, 로깅, 스트리밍 서비스  

- RabbitMQ   
  - 비동기 서비스, 적은 데이타 트래픽의 간단한 유스케이스에 적합
  - 우선순위 큐와 유연한 라우팅 옵션같은 이점. 

- Spring Quartz 
- SonarQube(?)
- 보안(암호화) - Jasypt(Java Simplified Encryption)  
  - PBEWithMD5AndDES : MD5와 DES를 이용한 패스워드 기반 암호화
  - AES256(Advanced Encryption Standard) : 대칭 키 암호화, 블록 암호

- Repository : Codecommit  
- 배포(CICD) : Jenkins, CodeDeploy  
- 배포 스크립트 : Dockering  


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조


  </pre>
</details>