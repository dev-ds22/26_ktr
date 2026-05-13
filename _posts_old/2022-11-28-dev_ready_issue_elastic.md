---
layout: single
title: "[Tech Stack] 02 - ELK Stack"
excerpt: "ELK stack 적용"

categories:
  - tech
tags:
  - [tech, elk, kafka]

toc: false
toc_sticky: true

date: 2022-11-28
last_modified_at: 2022-11-28
---
# ELK stack

- 엘라스틱 사용요청으로 ELK Stack 환경필요
- ELK Stack = ElasticSearch + Logstash + Kibana (+ Beats)

## ElasticSearch

- 프로그램으로서의 검색엔진. 
- 저장된 데이터 중 검색하는 특정한 데이터를 포함한 데이터를 취득. 
- 역색인과 형태소분석 지원을 통한 효과적이고 빠른 데이터 검색.
### DB만 있으면 되는데, 왜 굳이 검색엔진?

- 관계형 데이터베이스는 단순 텍스트매칭에 대한 검색만을 제공
- 텍스트를 여러 단어로 변형하거나 텍스트의 특질을 이용한 동의어나 유의어를 활용한 검색 가능
- ElasticSearch에서는 관계형 데이터베이스에서 불가능한 비정형 데이터의 색인과 검색이 가능
  - 빅데이터 처리에 중요.
- ElasticSearch에서는 형태소 분석을 통한 자연어 처리가 가능
  - ElasticSearch는 다양한 형태소 분석 플러그인을 제공.
- 역색인 지원으로 매우 빠른 검색이 가능

### 관계형 데이터베이스와의 개념 비교

- 개념비교

| ElasticSearch | RDBMS |
|---|---|
| 인덱스(Index) | 데이터베이스(Database) |
| 샤드(Shard) | 파티션(Partition) |
| 타입(Type) | 테이블(Table) |
| 문서(Document) | 행(Row) |
| 필드(Field) | 열(Column) |
| 매핑(Mapping) | 스키마(Schema) |
| Query DSL | SQL |

- 6.0이하 버전의 ElasticSearch에서는 하나의 인덱스 내부 기능에 따라 데이터 분류 후에 여러 개의 타입을 만들어 사용했지만 현재는 하나의 인덱스에 하나의 타입만을 구성해야 합니다.
- 매핑은 필드의 구조와 제약조건에 대한 명세를 말하며 관계형 DB의 스키마와 같습니다.
- 관계형 DB와 ElasticSearch는 인덱스라는 개념을 다르게 사용하는데, 관계형 DB에서 인덱스는 그저 Where절의 쿼리와 Join을 빠르게 만들어주는 보조데이터의 도구로 사용됩니다.

### RESTful API 사용

- CRUD를 방식이 RESTful API 사용.
- JDBC에서 관계형DB가 있는 아이피와 포트를 연결하여 SELECT, INSERT, DELETE.
- HTTP 통신에서 갖는 GET, POST, PUT, DELETE 등의 메소드를 적용.
  - HEAD 메소드는 특정 문서의 정보 유무를 확인에 이용.

- ElasticSearch의 POST는 스키마가 미리 정의되어 있지 않더라도, 자동으로 필드를 생성하고 저장.

### ElasticSearch 장점

- 데이터베이스 대용으로 사용 가능
  - NoSQL 데이터베이스처럼 사용가능. 거의 실시간(NRT)에 데이터 검색이 가능.

- 대량의 비정형 데이터 보관 및 검색 가능
  - 기존 데이터베이스로 처리하기 어려운 대량의 비정형 데이터 검색이 가능. 
  - 전문 검색(Full-Text Search)과 구조 검색 모두를 지원.

- 오픈소스 검색엔진
  - 아파치 루씬(Lucene)기반 오픈소스 검색엔진으로 무료로 사용 가능.

- 전문 검색(Full-text Search)
  - 내용 전체를 색인하여 특정 단어가 포함된 문서를 검색하는 것이 가능.

- 통계 분석
  - 비정형 로그 데이터를 수집하고 한 곳에 모아서 통계 분석이 가능. 
  - Kibana 를 이용 시각화 가능.

- 스키마리스(Schemaless)
  - 비정형의 다양한 형태의 문서도 자동으로 색인, 검색이 가능합니다.

- RESTful API
  - RESTful API를 사용 HTTP통신 기반으로 요청을 받아 JSON 형식으로 응답
  - 다양한 플랫폼에서 응용 가능.

- 멀티 테넌시(Multi-tenancy)
  - ElasticSearch에서 인덱스는 서로 다른 인덱스에서도 검색 필드명만 같으면 다수 인덱스를 동시조회.

- Document-Oriented
  - 여러 계층의 데이터를 JSON 형식의 구조화된 문서로 인덱스에 저장 가능. 
  - 계층 구조 문서도 한 번의 쿼리로 쉽게 조회 가능.

- 역색인(Inverted-Index) 지원.
- 확장성과 가용성
  - 분산 시스템 구성으로 병렬적인 처리가 가능. 
  - 분산 환경에서는 데이터가 샤드(Shard)라는 단위로 나누어 제공. 
  - 인덱스 생성 시마다 샤드의 수 조정이 가능. 
  - 데이터의 종류와 성격에 따라 데이터를 분산하여 빠르게 처리 가능.

### ElasticSearch 단점

- 진입 장벽이 있다.
- Document 간 조인을 수행할 수 없다. (두번 쿼리로 해결 가능)
- 트랜잭션 및 롤백이 제공되지 않음.
- 실시간(Real Time)' 처리 불가능. (색인된 데이터가 1초 뒤에나 검색이 가능하다.) NRT(Near Real Time)라는 표현사용
- 진정한 의미의 업데이트를 지원하지 않는다. (실제로는 삭제했다가 다시 만드는 형태)

## Logstash

- 데이터 수집 파이프라인
- 파이프라인으로 데이터를 수집하여 필터를 통해 변환 후 ES 로 전송

## Kibana

- ES 에서 색인된 데이터를 검색하여 분석 및 시각화

## Beats

- 경량 데이터 수집기
- 서버에 에이전트로 설치하여 다양한 유형의 데이터를 Logstash나 Elasticsearch에 전송

![ELK Stack](./../../images/tech/images_elasticsearch.png)

## ELK(Elasticsearch + Logstash + Kibana) + Filebeat


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조
 * [Elastic 가이드 북]( https://esbook.kimjmin.net/ )
 * [AWS EC2 ELK 설치]( https://angryfullstack.tistory.com/entry/AWS-EC2-ELK-Elasticsearch7x-Logstash7x-Kibana7x-%EC%B5%9C%EC%8B%A0%EB%B2%84%EC%A0%84-%EC%84%A4%EC%B9%98 )
 * [Elasticsearch 기본]( https://velog.io/@shinychan95/Elasticsearch-%EA%B8%B0%EB%B3%B8-%EA%B0%9C%EB%85%90-%EB%B0%8F-%ED%8A%B9%EC%A7%95-%EC%A0%95%EB%A6%AC )

  </pre>
</details>