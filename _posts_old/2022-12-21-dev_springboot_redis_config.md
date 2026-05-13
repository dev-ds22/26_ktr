---
layout: single
title: "[Spring Boot] Spring Redis Config"
excerpt: "Spring Boot Redis 설정"

categories:
  - tech
tags:
  - [tech, spring boot, redis]

toc: false
toc_sticky: true

date: 2022-12-21
last_modified_at: 2022-12-21
---
# [Spring Boot] Redis config

- 출처 : https://velog.io/@jungh00ns/Spring-Boot-Redis-Spring-Session-%EC%97%B0%EB%8F%99-Redis-%EB%A3%AC%EB%AC%B8%EC%9E%90-%ED%95%B4%EC%84%9D

## Redis 구성관련 이하 사이트 참조
- https://velog.io/@ililil9482/Redis-Master-Slave-%EA%B5%AC%EC%84%B1
- https://velog.io/@ililil9482/Redis-Spring-Boot-Master-Slave-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0
- https://velog.io/@ililil9482/Redis-Cluster-%EA%B5%AC%EC%84%B1
- https://velog.io/@ililil9482/Redis-Cluster-%EA%B5%AC%EC%84%B1-Spring-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0

## 1. Spring boot redis 설정

### 1.1. 의존성 추가
  
- build.gradle 에 이하 의존성 추가
  
```bash
  implementation 'org.springframework.boot:spring-boot-starter-data-redis'
  implementation 'org.springframework.session:spring-session-data-redis'
```
  
### 1.2 application.yml 설정

```yaml
spring:
  profiles: local
  redis:
    host: XX.XX.XXX.XX
    port: 6379
    password: XXXXXXXXXX
```
  
- host : 레디스 서버 주소, 같은 서버라면 localhost
- port : 레디스의 기본 포트는 6379, 실제 서버에 적용시 바꿔준다.
- password : 레디스에 설정한 비밀번호

### 1.3 Config 파일 생성
  
```java
  @NoArgsConstructor
  @Configuration
  @EnableRedisHttpSession()
  public class RedisConfig{

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Value("${spring.redis.password}")
    private String redisPassword;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
      RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
      redisStandaloneConfiguration.setHostName(redisHost);
      redisStandaloneConfiguration.setPort(redisPort);
      redisStandaloneConfiguration.setPassword(redisPassword);

      return new LettuceConnectionFactory(redisStandaloneConfiguration);
    }
    
    @Bean
    public StringRedisTemplate stringRedisTemplate() {
      StringRedisTemplate stringRedisTemplate = new StringRedisTemplate();
      stringRedisTemplate.setConnectionFactory(redisConnectionFactory());
      stringRedisTemplate.setKeySerializer(new StringRedisSerializer());
      stringRedisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer(objectMapper));
      stringRedisTemplate.setDefaultSerializer(new StringRedisSerializer());
      stringRedisTemplate.afterPropertiesSet();

      return stringRedisTemplate;
    }
  }
```
  
- @EnableRedisHttpSession()은 메인 Application에 해줘도 무방.

#### Redis-server 연결 설정
- public RedisConnectionFactory redisConnectionFactory()
  - application.yml에서 설정해준 redis-server에 관한 설정.
  - 이를 통해 Spring Session이 쓸 redis-server를 연결.
  - 어플리케이션 실행 시 redis연결 관련 에러라면 위설정 관련 가능성이 큼.

#### Redis Serializer 설정
- public StringRedisTemplate stringRedisTemplate()
  - Redis에 데이터를 작성할 때는 기본적으로 룬문자(\xac\xed\x00\x05t\x00\x03key) 사용
  - 이것을 일반적인 형태로 바꿔주려면 serializer 설정필요.

## 1.4 Redis 데이터 확인
  
- 실제 redis서버에 가서 redis-cli를 통해 데이터를 조회.
- 데이터 조회 방법
  - 1. redis 설치 서버 접속
  - 2. redis-cli
  - 3. auth "내가 설정한 비밀번호"
  - 4. keys *

### 데이터에 룬문자 포함시 해결법(Redis Session 설정시)
  
- \xac\xed\x00\x05sr\x00\x0ejava.lang.Long 등으로 표시시 해결법
- RedisConfig 파일에 이하 추가

```java
  @Bean
  public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
    return new GenericJackson2JsonRedisSerializer();
  }
```
- Spring Boot가 세션 생성할 때는 springSessionDefaultRedisSerializer가 자동으로 생성되는 Session의 Seriealizer를 담당


<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

- application.yml

```yaml
  redis:
    host: 192.168.111.22
    port: 6379
    sentinel:
      master: mymaster
      nodes: 192.168.111.22:5389, 192.168.111.22:5399, 192.168.111.22:5409
    ## password: qwer9753^&*
    lettuce:
      pool:
        max-active: 40
        max-idle: 40
        min-idle: 10
```
  

- MybatisOracleConfig.java
  
```java
  @Slf4j
  @Configuration
  public class RedisConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Value("${spring.server_type}")
    private String serverType;

    @Value("${spring.redis.sentinel.master}")
    private String sentinelMaster;

    @Value("${spring.redis.sentinel.nodes}")
    private String[] sentinelArray;

//    @Value("${spring.redis.password}")
    private String redisPwd;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
      switch(serverType){
        case "prd_cloud":
        case "stg":
        case "prd":
          RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration();
          redisStandaloneConfiguration.setHostName(redisHost);
          redisStandaloneConfiguration.setPort(redisPort);
        //  redisStandaloneConfiguration.setPassword(redisPwd);
          return new LettuceConnectionFactory(redisStandaloneConfiguration);
        case "local":
        case "dev":
        default:
          RedisSentinelConfiguration redisSentinelConfiguration = new RedisSentinelConfiguration()
            .master(sentinelMaster);
            // .sentinel("192.168.111.22",5389)
            // .sentinel("192.168.111.22",5399)
            // .sentinel("192.168.111.22",5409);
          for(String sentinelAddr : sentinelArray){
            String setinelHost = sentinelAddr.split(":")[0];
            String setinelPort = sentinelAddr.split(":")[1];
            redisSentinelConfiguration.sentinel(setinelHost,Integer.parseInt(setinelPort));
          }
          return new LettuceConnectionFactory(redisSentinelConfiguration);
      }
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
      RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
      redisTemplate.setConnectionFactory(redisConnectionFactory());
      redisTemplate.setKeySerializer(new StringRedisSerializer());
		  // redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());

      return redisTemplate;
    }
  }
```
  

  </pre>
</details>