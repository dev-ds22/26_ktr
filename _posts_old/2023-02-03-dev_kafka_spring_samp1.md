---
layout: single
title: "[tech] Spring Boot Kafka Template 사용 방법"
excerpt: "kafka"

categories:
  - tech
tags:
  - [kafka]

toc: false
toc_sticky: true

date: 2023-02-03
last_modified_at: 2023-02-03
---

# 1. Sample 1

### Spring Boot Kafka Template 사용 방법 - Kafka Header & Message

- Spring Boot Kafka Template 을 이용 Kafka 에 Multi Topic 을 Produce(=Send) 하다고 Consume(=Receive) 하는 방법.

- 참조 : https://github.com/jjeaby/kafkaSample.
- Spring 버전에 맞는 kafka 라이브러리의 버전.

| Spring for Apache Kafka Version | Spring Integration for Apache Kafka Version | kafka-clients       | Spring Boot                           |
| ------------------------------- | ------------------------------------------- | ------------------- | ------------------------------------- |
| 2.6.0                           | 5.3.x or 5.4.0-SNAPSHOT (pre-release)       | 2.6.0               | 2.3.x or 2.4.0-SNAPSHOT (pre-release) |
| 2.5.x                           | 3.3.x                                       | 2.5.0               | 2.3.x                                 |
| 2.4.x                           | 3.2.x                                       | 2.4.1               | 2.2.x                                 |
| 2.3.x                           | 3.2.x                                       | 2.3.1               | 2.2.x                                 |
| 2.2.x                           | 3.1.x                                       | 2.0.1, 2.1.x, 2.2.x | 2.1.x                                 |
| 2.1.x                           | 3.0.x                                       | 1.0.2               | 2.0.x (End of Life)                   |
| 1.3.x                           | 2.3.x                                       | 0.11.0.x, 1.0.x     | 1.5.x (End of Life)                   |

## 1-1. Kafka 라이브러리 설정

- pom.xml

```xml
<!-- KAFKA -->
  <dependency>
     <groupId>org.apache.kafka</groupId>
     <artifactId>kafka-clients</artifactId>
     <version>2.1.1</version>
  </dependency>
  <dependency>
     <groupId>org.apache.kafka</groupId>
     <artifactId>kafka-streams</artifactId>
     <version>2.1.1</version>
  </dependency>
  <dependency>
     <groupId>org.apache.kafka</groupId>
     <artifactId>kafka_2.12</artifactId>
     <version>2.1.1</version>
  </dependency>
  <dependency>
     <groupId>org.springframework.kafka</groupId>
     <artifactId>spring-kafka</artifactId>
     <version>2.2.12.RELEASE</version>
  </dependency>
```

- gradle

```bash
  implementation 'org.springframework.kafka:spring-kafka'
  testImplementation 'org.springframework.kafka:spring-kafka-test'
```

## 1-2. Topic 정보 등록

- 일반적으로 사용할 API 별로 호출 하는 Produce(=Send), Consume(=Recv) 생성, API 가 10 개 라면 Topic 은 20개.
- topic 은 호출하는 from(시스템 이름)-to(시스템 이름)-topic 으로 정의.
- group-id 와 topic-id 가 동일하면 Consume(=Recv) 가 되지 않으니 다르게 설정해야 함.
- Topic 을 Produce 할때 Kafka Header 에 Message Key 를 넣어 이를 이용 API 호출 가능

```yaml
### application.yaml

# Kafka 관련설정 - START      
spring:
  kafka:
    producer:
      bootstrap-servers: pilot.daiso.com:9096,pilot.daiso.com:9097,pilot.daiso.com:9098
      acks: all
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      
    listener:
      # ack-mode: MANUAL_IMMEDIATE
      type: SINGLE
      
    consumer:
      bootstrap-servers: pilot.daiso.com:9096,pilot.daiso.com:9097,pilot.daiso.com:9098
      group-id: dev-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      max-poll-records: 1000      
      # kafka 에서 메세지를 받고 자동으로 ACK 를 전송 여부 설정(true = 자동으로, false = 별도로 코드 구성 필요)
      enable-auto-commit: true
      # kafka 에서 메세지를 가져오는 consumer의 offset정보가 존재하지 않는 경우의 처리 방법
      # - latest : 가장 마지막 offset부터, 
      # - earliest : 가장 처음 offset부터, 
      # - none : offset 없다면 에러 발생
      auto-offset-reset: latest
      # Sample에서 사용하는 Group Id      
      group-id-sample-a: smaple-group-a
      group-id-sample-b: smaple-group-b
      group-id-sample-c: smaple-group-c
      
    # Sample에서 사용하는 topic
    template:
      sample-topic-a: sample-topic-a
      sample-topic-b: sample-topic-b
      sample-topic-c: sample-topic-c   
```

## 1-3. 사용 Class 작성

- 이하 4개 클래스 작성
- KafkaProducerConfig : Kafaka Topic(=Message) 를 Produce(=Send) 하는 설정 클래스
- KafkaProducer : Kafaka Topic(=Message) 를 Produce(=Send) 구현체
- KafkaConsumerConfig : Kafaka Topic(=Message) 를 Consume(=Recv) 하는 설정 클래스
- KafkaConsumer : Kafaka Topic(=Message) 를 Consume(=Recv) 하는 구현체

### 1-3-1. kafkaProducerConfig

- kafkaProducerConfig 클래스를 만들고 kafka 접속 주소, Serializer, Deserializer 설정.

```java
import java.util.HashMap;
import java.util.Map;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;

@EnableKafka
@Configuration
public class KafkaProducerConfig {
  @Value("${spring.kafka.producer.bootstrap-servers}")
  private String bootstrapServer;

  @Value("${spring.kafka.producer.key-serializer}")
  private String keyDeSerializer;

  @Value("${spring.kafka.producer.value-serializer}")
  private String valueDeSerializer;

  @Bean
  public ProducerFactory<String, String> producerFactory() {
    Map<String, Object> configProps = new HashMap<>();
    configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServer);
    configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, keyDeSerializer);
    configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, valueDeSerializer);
    return new DefaultKafkaProducerFactory<>(configProps);
  }

  @Bean
  public KafkaTemplate<String, String> kafkaTemplate() {
    return new KafkaTemplate<>(producerFactory());
  }
}
```

### 1-3-2. KafkaProducer

- KafkaProducer 를 생성하고 Kafka 로 Topic 을 Produce(=Send)
  - kafkaTemplate.send 를 호출 간단하게 구현.
- setHeader(KafkaHeaders.MESSAGE_KEY, messageKey) 코드를 이용 kafka Header 에 Message Key 를 설정
  - 해당방법 이용 Header 에 Custom Key 설정가능

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.kafka.support.SendResult;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureCallback;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@Component
public class KafkaProducer {
	@Autowired
	private KafkaTemplate<String, String> kafkaTemplate;

	public void sendMessage(String topicName, String messageKey, String message) {
		Message<String> messageBuilder = MessageBuilder.withPayload(message).setHeader(KafkaHeaders.TOPIC, topicName)
				.setHeader(KafkaHeaders.MESSAGE_KEY, messageKey).build();
		ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(messageBuilder);

		future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
			@Override
			public void onSuccess(SendResult<String, String> result) {
				log.info("Sent message=[" + message + "] with offset=[" + result.getRecordMetadata().offset() + "]");
			}

			@Override
			public void onFailure(Throwable ex) {
				log.info("Unable to send message=[" + message + "] due to : " + ex.getMessage());
			}
		});
	}
}
```

### 1-3-3. KafkaConsumerConfig

- Topic 을 Consume(=Recv) .
- kafkaConsumerConfig 클래스를 만들고 kafka 접속 주소, Serializer, Deserializer, offsetReset 등 설정.

- Multi Topic 을 지원을 위해 ConsumerFactory, ConcurrentKafkaListenerContainerFactory 를 지원하는 Muti Topic 의 갯수와 동일하게 생성필요. 여기서는 top-a-topic, top-b-topic 을 사용 하므로 각 Topic 마다 ConsumerFactory, ConcurrentKafkaListenerContainerFactor 생성.

```java
import java.util.HashMap;
import java.util.Map;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import lombok.extern.slf4j.Slf4j;

@Configuration
@EnableKafka
@Slf4j
public class KafkaConsumerConfig {

  @Value("${spring.kafka.producer.bootstrap-servers}")
  private String bootstrapServer;
  
  @Value("${spring.kafka.consumer.value-deserializer}")
  private String keyDeSerializer;
  
  @Value("${spring.kafka.consumer.value-deserializer}")
  private String valueDeSerializer;
  
  @Value("${spring.kafka.consumer.auto-offset-reset}")
  private String offsetReset;
  
  @Value("${spring.kafka.consumer.max-poll-records}")
  private String maxPollRecords;
  
  @Value("${spring.kafka.consumer.enable-auto-commit}")
  private String enableAutoCommit;

  @Bean
  public ConsumerFactory<String, String> topicAConsumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServer);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, keyDeSerializer);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, valueDeSerializer);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, offsetReset);
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, enableAutoCommit);
    
    log.info("Topic-A ConsumerFactory props : {}", props);  
    return new DefaultKafkaConsumerFactory<>(props);
  }

  @Bean(name = "topicAKafkaListenerContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, String>
  topicAKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(topicAConsumerFactory());
    return factory;
  }

  @Bean
  public ConsumerFactory<String, String> topicBConsumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServer);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, keyDeSerializer);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, valueDeSerializer);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, offsetReset);
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, enableAutoCommit);
    
    log.info("Topic-B ConsumerFactory props : {}", props);        
    return new DefaultKafkaConsumerFactory<>(props);
  }

  @Bean(name = "topicBKafkaListenerContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, String>
  topicBKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(topicBConsumerFactory());
    return factory;
  }
  
  @Bean
  public ConsumerFactory<String, String> topicCConsumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServer);
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, keyDeSerializer);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, valueDeSerializer);
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, offsetReset);
    props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, enableAutoCommit);
    
    log.info("Topic-C ConsumerFactory props : {}", props);        
    return new DefaultKafkaConsumerFactory<>(props);
  }

  @Bean(name = "topicCKafkaListenerContainerFactory")
  public ConcurrentKafkaListenerContainerFactory<String, String>
  topicCKafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(topicCConsumerFactory());
    return factory;
  }
}
```

### 1-3-4. KafkaListener

- Topic 을 Consum(=Recv) 하는 KafkaListener 작성.
- KafkaListener 는 @kafkaListner 어노테이션을 이용하여 작성.
- @kafkaListner Annotation으로 자동으로 Kafka 에서 Topic 을 Polling 하므로 별도 코드/설정 필요없음.
- setHeader(KafkaHeaders.MESSAGE_KEY, messageKey) 로 Produce 에 설정한 kafka Header 에 Message Key 를 @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey 를 수신

```java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Component
public class KafkaConsumer {
	@KafkaListener(topics = "${spring.kafka.template.sample-topic-a}", containerFactory = "topicAKafkaListenerContainerFactory", groupId = "${spring.kafka.consumer.group-id-sample-a}")
	public void listenSampleATopic(@Payload String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
			@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) throws Exception {
		log.info("Topic: [sample-topic-a] messageKey Message: [" + messageKey + "]");
		log.info("Topic: [sample-topic-a] Received Message: [" + message + "] from partition: [" + partition + "]");
	}

	@KafkaListener(topics = "${spring.kafka.template.sample-topic-b}", containerFactory = "topicBKafkaListenerContainerFactory", groupId = "${spring.kafka.consumer.group-id-sample-b}")
	public void listenSampleBTopic(@Payload String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
			@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) throws Exception {
		log.info("Topic: [sample-topic-b] messageKey Message: [" + messageKey + "]");
		log.info("Topic: [sample-topic-b] Received Message: [" + message + "] from partition: [" + partition + "]");
	}

	@KafkaListener(topics = "${spring.kafka.template.sample-topic-c}", containerFactory = "topicCKafkaListenerContainerFactory", groupId = "${spring.kafka.consumer.group-id-sample-c}")
	public void listenSampleCTopic(@Payload String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
			@Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey) throws Exception {
		log.info("Topic: [sample-topic-c] messageKey Message: [" + messageKey + "]");
		log.info("Topic: [sample-topic-c] Received Message: [" + message + "] from partition: [" + partition + "]");
	}
}
```

### 1-3-5. 테스트 코드 작성

- Topic 을 Producer(=Send) 하여 Consume(=Recv) 를 테스트 하기 위해 GET /send Rest API 에서 kafkaProducer.sendMessage 를 호출하도록 추가

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@RestController
@RequestMapping("/sample/kafka")
public class KafkaMessageController {

  @Autowired
  KafkaProducer kafkaProducer;

  @RequestMapping(method = RequestMethod.GET, path = "/send") String send() {
    kafkaProducer.sendMessage("sample-topic-a", "message key sample-a", "sample-a -> sample message");
    kafkaProducer.sendMessage("sample-topic-b", "message key sample-b", "sample-b -> sample message");
    kafkaProducer.sendMessage("sample-topic-c", "message key sample-c", "sample-c -> sample message");
    return "Kafka Produce!!!";
  }
}
```

- 서버가 실행, localhost:8080/send 를 호출.
- "medium-sendid-topic", "company-sendid-topic” Topic 이 Produce(=Send) 되고 Consume(=Recv) 서버 로그 확인.

---

---

---

# 2. Sample 2

## 2-1. pom.xml 에 의존성 등록

- gradle

```bash
  implementation 'org.springframework.kafka:spring-kafka'
  testImplementation 'org.springframework.kafka:spring-kafka-test'
```

## 2-2. application.properties 설정 추가

- kafka의 설정 파일을 application.properties 파일에 추가.

```yaml
spring:    
  kafka:
    producer:
      bootstrap-servers: pilot.daiso.com:9096,pilot.daiso.com:9097,pilot.daiso.com:9098
      acks: all
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      
    listener:
      # ack-mode: MANUAL_IMMEDIATE
      type: SINGLE
      
    consumer:
      bootstrap-servers: pilot.daiso.com:9096,pilot.daiso.com:9097,pilot.daiso.com:9098
      group-id: dev-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      max-poll-records: 1000      
      # kafka 에서 메세지를 받고 자동으로 ACK 를 전송 여부 설정(true = 자동으로, false = 별도로 코드 구성 필요)
      enable-auto-commit: true
      # kafka 에서 메세지를 가져오는 consumer의 offset정보가 존재하지 않는 경우의 처리 방법
      # - latest : 가장 마지막 offset부터, 
      # - earliest : 가장 처음 offset부터, 
      # - none : offset 없다면 에러 발생
      auto-offset-reset: latest
      # Sample에서 사용하는 Group Id      
      group-id-sample-a: smaple-group-a
      group-id-sample-b: smaple-group-b
      group-id-sample-c: smaple-group-c
      
    # Sample에서 사용하는 topic
    template:
      sample-topic-a: sample-topic-a
      sample-topic-b: sample-topic-b
      sample-topic-c: sample-topic-c
      
      message-topic: message
      filtered-topic: filtered
      partitioned-topic: partitioned
      custom-topic: custom   
```

## 2-3. KafkaTopicConfig 생성

- Topic 설정

```java
import java.util.HashMap;
import java.util.Map;
import org.apache.kafka.clients.admin.AdminClientConfig;
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.KafkaAdmin;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Configuration
public class SampleKafkaTopicConfig {
  @Value(value = "${spring.kafka.producer.bootstrap-servers}")
  private String bootstrapAddress;
  
  @Value(value = "${spring.kafka.template.message-topic}")
  private String messageTopicName;
  
  @Value(value = "${spring.kafka.template.filtered-topic}")
  private String filteredTopicName;
  
  @Value(value = "${spring.kafka.template.partitioned-topic}")
  private String partitionedTopicName;
  
  @Value(value = "${spring.kafka.template.custom-topic}")
  private String customTopicName;
  
  @Bean
  public KafkaAdmin kafkaAdmin() {
    Map<String, Object> configs = new HashMap<>();
    configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
    return new KafkaAdmin(configs);
  }
  
  @Bean
  public NewTopic messageTopic() {
    return new NewTopic(messageTopicName,	1,(short)1);
  }
  
  @Bean
  public NewTopic filteredTopic() {
    return new NewTopic(filteredTopicName, 1, (short) 1);
  }
  
  @Bean
  public NewTopic partitionedTopic() {
    return new NewTopic(partitionedTopicName, 6, (short) 1);
  }
  
  @Bean
  public NewTopic customTopic() {
    return new NewTopic(customTopicName, 1, (short) 1);
  }
}
```

- KafkaAdmin 타입의 생성자를 통해, Kafka 설정정보도 주입 가능.
- Kafka-Spring 에서는 위의 코드를 통해서, 코드를 이용해서 프로그래밍 적으로 메시지 큐의 토픽 생성가능.
- 스프링 부트(Kafka-Spring)에서는, 토픽을 생성해주는 함수를 만들고 Bean으로 등록해주면 자동으로 토픽을 생성해서 주입.

## 2-4. KafkaProducerConfig

- 실제로 메시지를 발행하는 Producer에 관한 설정.

```java
import java.util.HashMap;
import java.util.Map;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.serializer.JsonSerializer;

@EnableKafka
@Configuration
public class SampleKafkaProducerConfig {
  @Value(value = "${spring.kafka.producer.bootstrap-servers}")
  private String bootstrapAddress;

  @Bean
  public ProducerFactory<String, String> producerFactory() {
    Map<String, Object> configProps = new HashMap<>();
    configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
    configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    return new DefaultKafkaProducerFactory<>(configProps);
  }

  @Bean
  public KafkaTemplate<String, String> kafkaTemplate() {
    return new KafkaTemplate<String, String>(producerFactory());
  }

  public ProducerFactory<String, CustomMessage> customMessageProducerFactory() {
    Map<String, Object> configProps = new HashMap<>();
    configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
    configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    return new DefaultKafkaProducerFactory<>(configProps);
  }

  @Bean
  public KafkaTemplate<String, CustomMessage> customMessageKafkaTemplate() {
    return new KafkaTemplate<>(customMessageProducerFactory());
  }
}
```

- CustomMessage

```java
import lombok.Data;

@Data
public class CustomMessage {
	
  // 없을 경우 이하 에러발생
  // cannot deserialize from object value 
  // (no delegate- or property-based creator)
  // 원인 : jackson library는 빈 생성자가 없는 모델을 생성할수 없어 발생
  // 해결책 : 모델(Member)에 따로 빈 생성자를 추가
	public CustomMessage() {}
	
  public CustomMessage(String msg, String name) {
		this.msg = msg;
		this.name = name;
	}  
	private String msg;
  private String name;
}
```

- ProducerFactory 객체를 이용, 각 메시지 종류별로 메시지를 어디에 보내고, 어떠한 방식으로 처리할것인지 설정.
- 실제 메시지는 KafkaTemplate 이라는 객체에 담겨서 전송.
  ​- Socket 프로그래밍에 비유하면 Producer 객체는 Socket 디스크립터고, ProducerFactory는 Socket 디스크립터를 만들어주는 팩토리 메서드.
- 위의 예제에서는, 2가지 종류의 메시지 정의.

## 2-5. KafkaConsumerConfig

- 실제로 메시지 취득.

```java
import java.util.HashMap;
import java.util.Map;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@EnableKafka
@Configuration
public class SampleKafkaConsumerConfig {

	@Value(value = "${spring.kafka.producer.bootstrap-servers}")
	private String bootstrapAddress;

	public ConsumerFactory<String, String> consumerFactory(String groupId) {
		Map<String, Object> props = new HashMap<>();
		props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
		props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
		props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
		props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
		return new DefaultKafkaConsumerFactory<>(props);
	}

	public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(String groupId) {
		ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
		factory.setConsumerFactory(consumerFactory(groupId));
		return factory;
	}

	@Bean
	public ConcurrentKafkaListenerContainerFactory<String, String> typeAKafkaListenerContainerFactory() {
		return kafkaListenerContainerFactory("typeA");
	}

	@Bean
	public ConcurrentKafkaListenerContainerFactory<String, String> typeBKafkaListenerContainerFactory() {
		return kafkaListenerContainerFactory("typeB");
	}

	@Bean
	public ConcurrentKafkaListenerContainerFactory<String, String> headersKafkaListenerContainerFactory() {
		return kafkaListenerContainerFactory("headers");
	}

	@Bean
	public ConcurrentKafkaListenerContainerFactory<String, String> partitionsKafkaListenerContainerFactory() {
		return kafkaListenerContainerFactory("partitions");
	}

	@Bean
	public ConcurrentKafkaListenerContainerFactory<String, String> filterKafkaListenerContainerFactory() {
		ConcurrentKafkaListenerContainerFactory<String, String> factory = kafkaListenerContainerFactory("filter");
		factory.setRecordFilterStrategy(record -> record.value().contains("world"));
		return factory;
	}

	public ConsumerFactory<String, CustomMessage> CustomMessageConsumerFactory() {
		Map<String, Object> props = new HashMap<>();
		props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
		props.put(ConsumerConfig.GROUP_ID_CONFIG, "custom");
		return new DefaultKafkaConsumerFactory<>(props, new StringDeserializer(), new JsonDeserializer<>(CustomMessage.class));
	}

	@Bean
	public ConcurrentKafkaListenerContainerFactory<String, CustomMessage> customMessageKafkaListenerContainerFactory() {
		ConcurrentKafkaListenerContainerFactory<String, CustomMessage> factory = new ConcurrentKafkaListenerContainerFactory<String, CustomMessage>();
		factory.setConsumerFactory(CustomMessageConsumerFactory());
		return factory;
	}
}
```

- ConsumerFactory 를 이용, 각 메시지 종류별로 메시지를 어디에서 받고, 어떠한 방식으로 처리할것인지 설정.
- 설정한 각 Topic 별로 메시지를 어디서/어떻게 받을지를 설정.

## 2-6. Message Test.

- 위에서 작성한 설정을 기반으로 아래와 같이 메시지 큐 테스트.

### MessageProducer

- 메시지 생성부.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Component;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.util.concurrent.ListenableFutureCallback;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Component
public class SampleMessageProducer {
	@Autowired
	private KafkaTemplate<String, String> kafkaTemplate;

	@Autowired
	private KafkaTemplate<String, CustomMessage> customMessageKafkaTemplate;

	@Value(value = "${spring.kafka.template.message-topic}")
	private String messageTopicName;

	@Value(value = "${spring.kafka.template.filtered-topic}")
	private String filteredTopicName;

	@Value(value = "${spring.kafka.template.partitioned-topic}")
	private String partitionedTopicName;

	@Value(value = "${spring.kafka.template.custom-topic}")
	private String customTopicName;

	public void sendMessage(String message) {
		ListenableFuture<SendResult<String, String>> future = kafkaTemplate.send(messageTopicName, message);

		future.addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
			@Override
			public void onSuccess(SendResult<String, String> result) {
				log.info("Sent message=[{}] with offset=[{}]" + message, result.getRecordMetadata().offset());
			}

			@Override
			public void onFailure(Throwable ex) {
				log.info("Unable to send message=[{}] due to : {}", message, ex.getMessage());
			}
		});
	}

	public void sendMessageToPartion(String message, int partition) {
		kafkaTemplate.send(partitionedTopicName, partition, null, message);
	}

	public void sendMessageToFiltered(String message) {
		kafkaTemplate.send(filteredTopicName, message);
	}

	public void sendCustomMessage(CustomMessage custom) {
		customMessageKafkaTemplate.send(customTopicName, custom);
	}
}
```

- 2종류 KafkaTemplate을 정의.
- 해당 KafkaTemplate 들은 위에서 만든 KafkaProducerConfig에 있는 빈 객체가 대입.
- kafkaTemplate를 이용 메시지 전송. 
- 메시지 큐 방식으로 통신하는 경우 필연적으로 비동기 방식으로 통신하기 때문에 콜백 함수를 등록.
- rabbitMQ 등 여타 보통 메시지 큐에서는 요청-응답 을 쌍 수신을 위해 보통 수신 큐, 송신 큐 2개를 둬서 통신. 
- 이러한 임시 큐들을 카프카와 같은 메시지 브로커 서비스가 자동 생성

### MessageListener

- 메시지 수신부.

```java
import java.util.concurrent.CountDownLatch;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.annotation.TopicPartition;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Data
@Component
public class SampleMessageListener {

	public static CountDownLatch latch = new CountDownLatch(3);
	public static CountDownLatch partitionLatch = new CountDownLatch(2);
	public static CountDownLatch filterLatch = new CountDownLatch(2);
	public static CountDownLatch customLatch = new CountDownLatch(1);

	@KafkaListener(topics = "${spring.kafka.template.message-topic}", groupId = "typeA", containerFactory = "typeAKafkaListenerContainerFactory")
	public static void listenGroupTypeA(String message) {
		log.info("Received Message in group 'type A': {}", message);
		latch.countDown();
	}

	@KafkaListener(topics = "${spring.kafka.template.message-topic}", groupId = "typeB", containerFactory = "typeBKafkaListenerContainerFactory")
	public static void listenGroupTypeB(String message) {
		log.info("Received Message in group 'type B': {}", message);
		latch.countDown();
	}

	@KafkaListener(topics = "${spring.kafka.template.message-topic}", containerFactory = "headersKafkaListenerContainerFactory")
	public static void listenWithHeaders(@Payload String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
		log.info("Received Message: " + message + " from partition: " + partition);
		latch.countDown();
	}

	@KafkaListener(topicPartitions = @TopicPartition(topic = "${spring.kafka.template.partitioned-topic}", partitions = {
			"0", "3" }), containerFactory = "partitionsKafkaListenerContainerFactory")
	public static void listenToParition(@Payload String message, @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
		log.info("Received Message: {}, from partition: {}", message, partition);
		partitionLatch.countDown();
	}

	@KafkaListener(topics = "${spring.kafka.template.filtered-topic}", containerFactory = "filterKafkaListenerContainerFactory")
	public static void listenWithFilter(String message) {
		log.info("Recieved Message in filtered listener: {}", message);
		filterLatch.countDown();
	}

	@KafkaListener(topics = "${spring.kafka.template.custom-topic}", containerFactory = "customMessageKafkaListenerContainerFactory")
	public static void customListener(CustomMessage greeting) {
		log.info("Recieved greeting message: {}", greeting);
		customLatch.countDown();
	}
}
```

- 어떠한 Topic의 메시지를 어떠한 방식으로 받을지를 @KafkaListener를 이용해서 설정.
- KafkaListener를 통해서 특정 파티션의 메시지를 받거나 특정 그룹의 메시지를 받거나 하는등의 설정도 가능.
- Consumer와 같은 경우, 병렬로 메시지를 처리하는 경우도 있기때문에, 동시접근으로 인한 Race Condition과 같은 경우를 막기 위해서 CountDownLatch 라는 함수를 이용해서 접근을 제한.(일종의 세마포어)

### Test용 Controller

- 수신/송신 부분을 실제적용.

```java
import java.util.concurrent.TimeUnit;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RestController
@RequestMapping("/sample2/kafka")
public class SampleKafkaMessageController {

	@Autowired
	SampleMessageProducer kafkaProducer;

	@Autowired
	SampleMessageListener kafkaListener;

	@RequestMapping(method = RequestMethod.GET, path = "/send")
	void send() throws InterruptedException {
		kafkaProducer.sendMessage("Hello, World!");
		SampleMessageListener.latch.await(10, TimeUnit.SECONDS);

		for (int i = 0; i < 20; i++) {
			kafkaProducer.sendMessageToPartion("Hello To Partioned Topic!", i);
		}
		SampleMessageListener.partitionLatch.await(10, TimeUnit.SECONDS);

		kafkaProducer.sendMessageToFiltered("Hello Baeldung!");
		kafkaProducer.sendMessageToFiltered("Hello World!");
		SampleMessageListener.filterLatch.await(10, TimeUnit.SECONDS);

		kafkaProducer.sendCustomMessage(new CustomMessage("Custom", "World!"));
		SampleMessageListener.customLatch.await(10, TimeUnit.SECONDS);
	}
}
```
​
#### 실행시 로그확인.

```
  Sent message=[Hello, World!] with offset=[1]
  Received Messasge in group 'bar': Hello, World!
  Received Messasge in group 'foo': Hello, World!
  Received Messasge: Hello, World! from partition: 0
  Received Message: Hello To Partioned Topic! from partition: 0
  Received Message: Hello To Partioned Topic! from partition: 3
  Recieved Message in filtered listener: Hello Baeldung!
  Recieved Message in filtered listener: Hello World!
```

---

## Producer Option

| Producer Option                       | Description                                                                                                                                                                                                                                                |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| bootstrap.servers                     | 카프카 클러스터는 클러스터 마스터라는 개념이 없으므로, 클러스터 내 모든 서버가 클라이언트의 요청을 받을 수 있음. 클라이언트가 카프카 클러스터에 처음 연결하기 위한 호스트와 포트정보를 나타 냄.                                                                                                                                              |
| client.dns.lookup                     | 하나의 호스트에 여러 IP를 매핑해 사용하는 일부 환경에서 클라이언트가 하나의 IP와 연결하지 못할 경우에 다른 IP로 시도하는 설정. use_all_dns_ips가 기본값으로, DNS에 할당된 호스트의 모든 IP를 쿼리하고 저장. 첫 번째 IP로 접근 실패 시, 종료하지 않고 다음 IP로 접근 시도. resolve_canonical_bootstrap_servers_only 옵션은 Kerberos 환경에서 FQDN을 얻기 위한 용도로 사용 됨. |
| acks                                  | 프로듀서가 카프카 토픽의 리더 측에 메시지를 전송한 후 요청을 완료하기를 결정하는 옵션. 0, 1, all(-1)로 표현하며, 0은 빠른 전송을 의미하지만, 일부 메시지 손실 가능성 있음. 1은 리더가 메시지를 받았는지 확인하지만, 모든 팔로워를 전부 확인하지 않음. 대부분 기본값으로 1을 사용. all은 팔로워가 메시지를 받았는지 여부 확인. 다소 느릴 수 있으나 하나의 팔로우가 있는 한 메시지 손실은 없음.                    |
| buffer.memory                         | 프로듀서가 카프카 서버로 데이터를 보내기 위해 잠시 대기(배치 전송이나 딜레이 등)할 수 있는 전체 메모리 바이트(byte)                                                                                                                                                                                      |
| compression.type                      | 프로듀서가 메시지 전송 시 선택할 수 있는 압축 타입. none, gzip, snappy, lz4, zstd 중 선택                                                                                                                                                                                          |
| enbale.idempotence                    | 설정을 true로 하는 경우 중복 없는 전송이 가능하며, 이와 동시에 max.in.flight.requests.per.connection은 5 이하, retries는 0 이상, acks는 all로 설정해야 함.                                                                                                                                      |
| max.in.flight.requests.per.connection | 하나의 커넥션에서 프로듀서가 최대한 ACK 없이 전송할 수 있는 요청 수. 메시지의 순서가 중요하다면 1로 설정할 것을 권장하지만 성능은 다소 떨어짐.                                                                                                                                                                       |
| retries                               | 일시적인 오류로 인해 전송에 실패한 데이터를 다시 보내주는 횟수                                                                                                                                                                                                                        |
| batch.size                            | 프로듀서는 동일한 파티션으로 보내는 여러 데이터를 함께 배치로 보내려고 시도. 적절한 배치 크기 설정은 성능에 도움을 줌.                                                                                                                                                                                       |
| linger.ms                             | 배치 형태의 메시지를 보내기 전 추가적인 메시지를 위해 기다리는 시간을 조정하고, 배치 크기에 도달하지 못한 상황에서 linger.ms 제한 시간에 도달 했을 때 메시지를 전송.                                                                                                                                                        |
| transactional.id                      | ‘정확히 한 번 전송’을 위해 사용하는 옵션. 동일한 TransactionalId에 한해 정확히 한 번을 보장. 옵션을 사용하기 전 enable.idempotence를 true로 설정해야 함.                                                                                                                                                |

### 참고사항

##### acks
- 전송된 ProducerRecord를 수신하는 파티션 리플리카(복제 서버로 동작하는 브로커)의 수를 제어
- 프로듀서가 서버에 메시지를 보낸 후 요청을 완료하기 전 승인의 수
- 메시지 유실 가능성에 큰 영향을 주며, 다음 세 가지로 설정 가능 
  - acks = 0 : 프로듀서는 브로커의 응답을 기다리지 않는다
  - acks = 1 : 리더는 데이터를 기록, 팔로워는 신경쓰지 않음
  - acks = all : 무손실, 동기화된 모든 리플리카가 메시지를 받으면 프로듀서가 브로커의 성공 응답을 받음

##### buffer.memory
- 브로커에게 전송될 메시지의 버퍼로 사용할 메모리 양(byte)

##### compression.type
- 기본적인 메시지는 압축되지 않은 상태로 전송되지만, 이 값을 설정하면 압축되어 전송 - snappy, gzip, lz4

##### retries
- 최대 재전송 회수. (default=2147483647) retry.backoff.ms(100ms)로 재전송간에 시간을 조정할 수 있음.

##### batch.size
- 같은 파티션에 쓰는 다수의 레코드를 배치 단위로 관리하는데, 이 배치에 사용될 메모리양을 말함.(byte)
- 너무 작게 설정할 경우, 프로듀서가 자주 메시지를 전송해야 하므로 성능 저하 가능성이 있음.

##### linger.ms
- 현재의 배치를 전송하기 전까지 기다리는 시간(default=0)
- batch.size 가 가득 차서 전송이 되거나, linger.ms 시간이 다 되어 전송이 되거나.

##### client.id
- 어떤 클라이언트에서 전송된 메시지인지 식별하기 위해 브로커가 사용

##### max.in.flight.requests.per.connection
- blocking 되기 전까지 응답이 오지 않은 메시지들을 몇 개까지 허용할 것인지. 이값을 1로 설정하면 메시지의 전송 순서대로 브로커가 쓰게 된다. (default = 5)

##### request.timeout.ms
- 데이터 전송시(request.timeout.ms), 메타데이터 요청할 때(metadata.fetch.timeout.ms) 프로듀서가 서버의 응답을 기다리는 제한 시간
- timeout.ms는 동기화된 리플리카들이 메시지를 인지하는동안 브로커가 대기하는 시간, acks 매개변수의 설정에 따라 달라진다
- https://kafka.apache.org/documentation/#upgrade_1100_notable
- 만약 서버의 응답 없이 제한 시간이 경과되면 프로듀서는 재전송을 하거나 예외, 콜백을 전달하여 에러 응답. 

##### max.block.ms
- send() 메서드 호출시, 프로듀서의 전송 버퍼가 가득 차거나 메타데이터를 요청했지만 사용할 수 없을 때 이 시간동안 일시 중단됨 
- 그 다음에 max.block.ms의 시간이 되면 시간 경과 예외가 발생

##### max.request.size
- 전송될 수 있는 가장 큰 메시지의 크기, 프로듀서가 하나의 요청으로 전송할 수 있는 메시지의 최대 개수 모두를 이 매개변수로 제한함
- 브로커에 message.max.bytes 메시지의 최대 크기 제한값과 max.request.size의 값이 일치되도록 설정하는 것이 좋음
- 브로커가 거부하는 크기의 메시지를 프로듀서가 전송하지 않을 것이기 때문

##### receive.buffer.bytes와 send.buffer.bytes
- TCP 소켓이 사용하는 (송수신) 버퍼 크기
- -1로 사용하면 운영체제 기본값 사용
- 프로듀서, 컨슈머가 서로 다른 데이터센터의 브로커들과 통신할 때 이 값을 증가시키는 것이 좋음

---

## Consumer Option

| Consumer Option               | Description                                                                                                                                 |
|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| bootstrap.servers             | 프로듀서와 동일하게 브로커의 정보를 입력.                                                                                                                     |
| fetch.min.bytes               | 한 번에 가져올 수 있는 최소 데이터 크기. 지정한 크기보다 작은 경우, 요청에 응답하지 않고 데이터가 누적될 때까지 대기.                                                                       |
| group.id                      | 컨슈머가 속한 컨슈머 그룹을 식별하는 식별자. 동일한 그룹 내의 컨슈머 정보는 모두 공유 됨.                                                                                        |
| heartbeat.interval.ms         | 하트 비트가 있다는 것은 컨슈머의 상태가 active임을 의미. session.timeout.ms와 밀접한 관계가 있으며, session.timeout.ms보다 낮은 값으로 설정해야 함. 일반적으로 session.timeout.ms의 1/3로 설정. |
| max.partition.fetch.bytes     | 파티션 당 가져올 수 있는 최대 크기                                                                                                                        |
| session.timeout.ms            | 이 옵션을 이용해, 컨슈머가 종료된 것인지를 판단. 컨슈머는 주기적으로 하트 비트를 보내야 하고, 만약 이 시간 전까지 하트 비트를 보내지 않았다면 해당 컨슈머는 종료된 것으로 간주하고 컨슈머 그룹에서 제외 후, 리밸런싱을 시작 함.          |
| enable.auto.commit            | 백그라운드로 주기적으로 오프셋을 커밋                                                                                                                        |
| auto.offset.reset             | 카프카에서 초기 오프셋이 없거나 현재 오프셋이 더 이상 존재하지 않는 경우에 다음 옵션으로 reset 함.                                                                                 |
|                               | earliest: 가장 초기의 오프셋값으로 설정                                                                                                                  |
|                               | latest: 가장 마지막 오프셋값으로 설정                                                                                                                    |
|                               | none: 이전 오프셋값을 찾지 못하면 에러를 나타냄.                                                                                                              |
|                               |                                                                                                                                             |
| fetch.max.bytes               | 한 번의 가져오기 요청으로 가져올 수 있는 최대 크기                                                                                                               |
| group.instance.id             | 컨슈머의 고유 식별자. 만약 설정한다면 static 멤버로 간주되어, 불필요한 리밸런싱을 하지 않음.                                                                                    |
| isolation.level               | 트랜잭션 컨슈머에서 사용되는 옵션.                                                                                                                         |
|                               | read_uncommited(Default) : 모든 메시지를 읽음                                                                                                       |
|                               | read_committed : 트랜잭션이 완료된 메시지만 읽음                                                                                                          |
|                               |                                                                                                                                             |
| max.poll.records              | 한 번의 poll() 요청으로 가져오는 최대 메시지 수.                                                                                                             |
| partition.assignment.strategy | 파티션 할당 전략.(Default : range)                                                                                                                 |
| fetch.max.wait.ms             | fetch.min.bytes에 의해 설정된 데이터보다 적은 경우 요청에 대한 응답을 기다리는 최대 시간.                                                                                  |


<details>
  <summary>Exp.</summary>  
  <pre>

  </pre>
</details>
