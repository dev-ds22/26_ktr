---
layout: single
title: "[Tech Stack] 01 - Reactive vs MVC"
excerpt: "Reactive (Webflux, NonBlocking) 와 MVC (Blocking) 비교"

categories:
  - tech
tags:
  - [tech, Reactive, MVC]

toc: false
toc_sticky: true

date: 2022-11-27
last_modified_at: 2022-11-27
---
# Reactive vs MVC

## 1. Reactive (nio)와 MVC (bio) 속도 비교

- Reactive 프로젝트의 비지니스 로직들이 모두 Async + NonBlocking 으로 되어있다면 빠를 것. 
  (DB connector, 외부 API 호출 등)
- 하나라도 Sync or Blocking 된 부분이 있거나, CPU 를 많이 쓰는 코드가 들어있다면 MVC보다 느림

## 2. blocking 로직이 들어간 Reactive

- 느리다. blocking 이 하나라도 들어가면 MVC가 더 빠르다.

## 3. Reactive 의 장단점

### 장점

- MVC 보다 빠르다. (더 빠른 응답속도)
- MVC 보다 가볍다. (낮은 CPU, Memory 사용)
- MVC에 비해 튜닝포인트가 적다. (Heap Memory, GC등)

### 단점 - 개발조직이 Reactive Patterns에 얼마나 익숙한가?

- Reactive는 패러다임의 변화가 필요
- Reactive 개념이 충분히 학습되지 않으면 Blocking 요소 개입으로 낮은 성능
- 유지보수 어려움
- 코드 가독성이 매우 낮음
- 값을 즉시 반환하지 않는 코드 블록의 디버깅 및 테스트 어려움
- 안정성 : Full Reactive 스택이 아직 안정적이지 않음

### R2DBC

- **리엑티브 방식에서 R2DBC를 사용하지 않으면 MVC보다 낮은 성능**
- H2, MariaDB, MySQL, Postgres, Oracle, MS SQL 지원
- R2DBC의 단점

>  
> Spring Data R2DBC aims at being conceptually easy.  
> In order to achieve this, it does NOT offer caching,   
> lazy loading, write-behind, or many other features of ORM frameworks.  
> This makes Spring Data R2DBC a simple, limited, opinionated object mapper.  
>

- 개념적으로 쉬운 것을 목표, 기존 JDBC나 JPA에서 쉽게 사용했던 기능을 제공안함

## 결론

- 100ms 미만의 트래픽이 높은 서비스개발 시 webflux + reative DB 를 사용
- 400ms 미만의 높은 트래픽을 맡게 된다면 계속 MVC 를 사용


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

 * [Non-Blocking(WebFlux) vs. Blocking(MVC)]( https://wowyongs.tistory.com/16 )

- END

  </pre>
</details>