---
layout: single
title: "[Kafka] Producer, Consumer, Listener 구현"
excerpt: "Spring에서 Kafka Producer 및 Consumer 구현"

categories:
  - tech
tags:
  - [kafka, topic]

toc: false
toc_sticky: true

date: 2023-01-12
last_modified_at: 2023-01-12
---

# Spring에서 Kafka Producer 및 Consumer 구현

## 1. Spring kafka Producer

- Kafka를 Spring 프레임워크에서 효과적으로 사용할 수 있도록 만들어진 라이브러리. 
- 기존 Kafka 클라이언트 라이브러리를 래핑하여 Kafka 클라이언트에서 사용하는 여러 패턴을 사전제공.
- spring-kafka 라이브러리를 추가하면 관련 디펜던시로 kafka Client 라이브러리가 추가. 
- spring-kafka 라이브러리는 Admin, Consumer, Producer, Streams 기능을 제공합니다.

>  
> Java 기반 어플리케이션으로 kafka를 적용할때 사용하는 라이브러리는  
> spring kafka vs kafka clients 2가지가 있다.  
> spring-kafka는 kafka-clients보다 레퍼런스가 풍부하며, 고수준의 라이브러리로   
> 사용자가 사용하기 편리한 장점이 있다.   
> kafka clients는 저수준의 라이브러리로 사용자가 직접 Configuration을 해줘야하지만  
> 사용자가 직접 환경설정을 지정해주기 때문에 Consumer, Producer 하나만 셋팅하고   
> 구현해도 문제 없이 구현된다는 점이 편리하다.   
>  

- build.gradle에 디펜던시 추가

```bash
  dependencies {
      compile 'org.springframework.kafka:spring-kafka:2.5.10.RELEASE'
  }
```

- spring-kafka Producer는 Kafka Template 클래스를 사용하여 데이터를 전송가능. 
- Kafka Template은 Producer 팩토리(ProducerFactory) 클래스를 통해 생성.

### 1-1. 기본 Kafka Template

- 기본 Kafka Template은 기본 Producer 팩토리를 통해 생성된 Kafka Template을 사용. 
- 기본 Kafka Template을 사용할 때는 application.yaml에 Producer 옵션 설정후 사용. 
- application.yaml에 설정한 Producer 옵션값은 애플리케이션이 실행될 때 자동으로 오버라이드되어 설정. 
- **Spring Kafka Producer를 사용할 경우에는 필수 옵션이 없음.** 
  - 옵션을 미설정 시 bootstrap-servers는 localhost:9092
  - key-serializer와 value-serializer는 StringSerializer로 자동 설정.

#### application.yaml Producer 옵션값

```bash
  spring.kafka.producer.acks
  spring.kafka.producer.batch-size
  spring.kafka.producer.bootstrap-servers
  spring.kafka.producer.buffer-memory
  spring.kafka.producer.client-id
  spring.kafka.producer.compression-type
  spring.kafka.producer.key-serializer
  spring.kafka.producer.properties.*
  spring.kafka.producer.retries
  spring.kafka.producer.transaction-id-prefix
  spring.kafka.producer.value-serializer
```

#### test0부터 test9까지 Message 값을 클러스터로 보내는 Producer.

```java
  package com.example;

  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.CommandLineRunner;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.kafka.core.KafkaTemplate;

  @SpringBootApplication
  public class SpringProducerApplication implements CommandLineRunner {

    private static String TOPIC_NAME = "test";
    
    @Autowired
    private KafkaTemplate<Integer, String> template;

    public static void main(String[] args) {
      SpringApplication application = new SpringApplication(SpringProducerApplication.class);
      application.run(args);
    }

    @Override
    public void run(String... args) {
      for (int i = 0; i < 10; i++) {
          template.send(TOPIC_NAME, "test" + i);
      }
      System.exit(0);
    }
  }
```

- KafkaTemplate은 send(String topic, V data) 이외에도 여러 가지 데이터 전송 메서드들을 오버로딩하여 제공.

#### 데이터 전송 메서드

- send(String topic, K key, V data) : Message 키, 값을 포함하여 특정 토픽으로 전달
- send(String topic, Integer partition, K key, V data) : Message 키, 값이 포함된 레코드를 특정 토픽의 특정 파티션으로 전달
- send(String topic, Integer partition, Long timestamp, K key, V data) : Message 키, 값 타임스탬프가 포함된 레코드를 특정 토픽의 특정 파티션으로 전달
- send(ProducerRecord<K, V> record) : Producer 레코드(ProducerRecord) 객체를 전송

### 1-2. Custom Kafka Template

- Producer 팩토리를 통해 만든 Kafka Template 객체를 Bean으로 등록하여 사용. 
- Producer에 필요한 각종 옵션을 선언하여 사용가능. 
- 하나의 Spring Kafka applicaton 내부에 다양한 종류의 Kafka Producer 인스턴스를 생성하고 싶다면 이 방식을 사용. 
- A클러스터로 전송하는 Kafka Producer와 B클러스터로 전송하는 Kafka Producer를 동시에 사용하고 싶다면 Custom Kafka Template을 사용하여 2개의 Kafka Template을 Bean으로 등록하여 사용.

```java
  package com.example;

  import org.apache.kafka.clients.producer.ProducerConfig;
  import org.apache.kafka.common.serialization.StringSerializer;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.kafka.core.*;
  import java.util.HashMap;
  import java.util.Map;

  // KafkaTemplate 빈 객체를 등록하기 위해 @Configuration 선언
  @Configuration
  public class KafkaTemplateConfiguration {

    @Bean
    public KafkaTemplate<String, String> customKafkaTemplate() {

      Map<String, Object> props = new HashMap<>();
      props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
      props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
      props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
      props.put(ProducerConfig.ACKS_CONFIG, "all");

      // ProducerFactory를 사용하여 KafkaTemplate 객체를 만들 때는 Producer 옵션을 직접 넣음
      ProducerFactory<String, String> pf = new DefaultKafkaProducerFactory<>(props);

      return new KafkaTemplate<>(pf);
    }
  }
```

- Spring Kafka에서는 KafkaTemplate 외에도 아래 템플릿을 제공.
- ReplyingKafkaTemplate: Consumer가 특정 데이터를 전달받았는지 여부 확인 가능
- RoutingKafkaTemplate: 전송하는 토픽별 옵션을 다르게 설정 가능

```java
  package com.example;

  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.CommandLineRunner;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.WebApplicationType;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.kafka.core.KafkaProducerException;
  import org.springframework.kafka.core.KafkaSendCallback;
  import org.springframework.kafka.core.KafkaTemplate;
  import org.springframework.kafka.support.SendResult;
  import org.springframework.util.concurrent.ListenableFuture;

  @SpringBootApplication
  public class SpringProducerApplication implements CommandLineRunner {

    private static String TOPIC_NAME = "test";

    // Bean 객체로 등록한 customKafkaTemplate 주입받도록 메서드 이름과 동일한 변수명 선언
    @Autowired
    private KafkaTemplate<String, String> customKafkaTemplate;

    public static void main(String[] args) {
      SpringApplication application = new SpringApplication(SpringProducerApplication.class);
      application.run(args);
    }

    @Override
    public void run(String... args) {
      // ListenableFuture: 전송 이후 정상 적재됐는지 여부 확인
      ListenableFuture<SendResult<String, String>> future = customKafkaTemplate.send(TOPIC_NAME, "test");
      
      // 비동기 확인 방법
      future.addCallback(new KafkaSendCallback<String, String>() {
          @Override
          public void onSuccess(SendResult<String, String> result) {
            // 브로커에 정상 적재 시 수행
          }

          @Override
          public void onFailure(KafkaProducerException ex) {
            // 적재되지 않고 이슈 발생 시 수행
          }
      });
      System.exit(0);
    }
  }
```

## 2. Spring Kafka Consumer

- Spring Kafka Consumer는 기존 Consumer를 2개의 타입으로 나누고 커밋을 7가지로 나누어 세분화 했습니다.

### 2-1. Consumer(Listener) 타입

- 레코드 Listener(MessageListener): **단 1개의 레코드를 처리.** (Spring Kafka Consumer의 기본 Listener 타입)
- 배치 Listener(BatchMessageListener): **한 번에 여러 개 레코드들을 처리.**
  - 매뉴얼 커밋(Spring Kafka에서는 커밋이라고 부르지 않고 'AckMode'라고 함. AcksMode: MANUAL, MANUAL_IMMEDIATE)을 사용할 경우 Acknowledging이 붙은 Listener 사용
  - KafkaConsumer 인스턴스에 직접 접근하여 컨트롤하고 싶다면 ConsumerAware가 붙은 Listener 사용.
- listener은 thread-safe 하지 않기 때문에 Bean으로 등록한 listener을 여러곳에서 주입받아서 사용하면 안됨. **listener은 한 곳에서 호출되고 처리되도록 구현필요.**

### 2-2. Message Listener 종류와 파라미터

#### RECORD Type

- 1. MessageListener : onMessage(ConsumerRecord<K, V> data)
  - onMessage(V data)	오토 커밋 또는 Consumer 컨테이너의 AckMode를 사용하는 경우
- 2. AcknowledgingMessageListener : onMessage(ConsumerRecord<K, V> data, Acknowledgment acknowledgment)
  - onMessage(V data, Acknowledgment acknowledgment)	매뉴얼 커밋을 사용하는 경우
- 3. ConsumerAwareMessageListener : onMessage(ConsumerRecord<K, V> data, Consumer<K, V> consumer)
  - onMessage(V data, Consumer<K, V> consumer)	Consumer 객체를 활용하고 싶은 경우
- 4. AcknowledgingConsumerAwareMessageListener : onMessage(ConsumerRecord<K, V> data, Acknowledgment acknowledgment, Consumer<?, ?> consumer)
  - onMessage(V data, Acknowledgment acknowledgment, Consumer<K, V> consumer)	매뉴얼 커밋을 사용하고 Consumer 객체를 활용하고 싶은 경우

#### BATCH Type

- 1. BatchMessageListener : onMessage(ConsumerRecords<K, V> data)
  - onMessage(List<V> data)	오토 커밋 ㄸ도는 Consumer 컨테이너의 AckMode를 사용하는 경우
- 2. BatchAcknowledgingMessageListener : onMessage(ConsumerRecords<K, V> data, Acknowledgment acknowledgment)
  - onMessage(List<V> data, Acknowledgment acknowledgment)	매뉴얼 커밋을 사용하는 경우
- 3. BatchConsumerAwareMessageListener	onMessage(ConsumerRecords<K, V> data, Consumer<K, V> consumer)
  - onMessage(List<V> data, Consumer<K, V> consumer)	Consumer 객체를 활용하고 싶은 경우
- 4. BatchAcknowledgingConsumerAwareMessageListener	onMessage(ConsumerRecords<K, V> data, Acknowledgment acknowledgment, Consumer<K, V> consumer)
  - onMessage(List<V> data, Acknowledgment acknowledgment, Consumer<K, V> consumer)	매뉴얼 커밋을 사용하고 Consumer 객체를 활용하고 싶은 경우
 

### 3. AcksMode

- Spring Kafka Consumer의 AckMode 기본값은 BATCH이고 Consumer의 enable.auto.commit 옵션은 false.
- 1. RECORD	: 레코드 단위로 프로세싱 이후 커밋
- 2. BATCH :	poll() 메서드로 호출된 레코드가 모두 처리된 이후 커밋
  - Spring Kafka Consumer의 AckMode 기본값
- 3. TIME :	특정 시간 이후에 커밋
  - 이 옵션을 사용할 경우 시간 간격을 선언하는 AckTime 옵션을 설정해야 한다.
- 4. COUNT :	특정 개수만큼 레코드가 처리된 이후 커밋
  - 이 옵션을 사용할 경우에는 레코드 개수를 선언하는 AckCount 옵션을 설정해야 한다.
- 5. COUNT_TIME :	TIME, COUNT 옵션 중 맞는 조건이 하나라도 나올 경우 커밋
- 6. MANUAL :	Acknowledgement.acknowledge() 메서드가 호출되면 다음번 poll() 때 커밋. 
  - 매번 acknowledge() 메서드를 호출하면 BATCH 옵션과 동일하게 동작. 
  - 이 옵션을 사용할 경우에는 AcknowledgingMessageListener 또는 BatchAcknowledgingMessageListener를 Listener로 사용.
- 7. MANUAL_IMMEDIATE :	Acknowledgement.acknowledge() 메서드를 호출한 즉시 커밋. 
  - 이 옵션을 사용할 경우에는 AcknowledgingMessageListener 또는 BatchAcknowledgingMessageListener를 Listener로 사용.

### 4. 기본 Listener 컨테이너

- 기본 Listener 컨테이너는 기본 Listener 컨테이너 팩토리를 통해 생성된 Listener 컨테이너를 사용. 
- 기본 Listener 컨테이너를 사용할 때는 application.yaml에 Consumer와 Listener 옵션을 넣고 사용할 수 있으며 설정한 옵션값은 애플리케이션이 실행될 때 자동으로 override되어 설정. 

#### application.yaml Consumer와 Listener 옵션값

```bash
  spring.kafka.consumer.auto-commit-interval
  spring.kafka.consumer.auto-offset-reset
  spring.kafka.consumer.bootstrap-servers
  spring.kafka.consumer.client-id
  spring.kafka.consumer.enable-auto-commit
  spring.kafka.consumer.fetch-max-wait
  spring.kafka.consumer.fetch-min-size
  spring.kafka.consumer.group-id
  spring.kafka.consumer.heartbeat-interval
  spring.kafka.consumer.key-deserializer
  spring.kafka.consumer.max-poll-records
  spring.kafka.consumer.properties.*
  spring.kafka.consumer.value-deserializer

  spring.kafka.listener.ack-count
  spring.kafka.listener.ack-mode
  spring.kafka.listener.ack-time
  spring.kafka.listener.client-id
  spring.kafka.listener.concurrency
  spring.kafka.listener.idle-event-interval
  spring.kafka.listener.log-container-config
  spring.kafka.listener.monitor-interval
  spring.kafka.listener.no-poll-threshold
  spring.kafka.listener.poll-timeout
  spring.kafka.listener.type
``` 

#### 4-1. 레코드 Listener(MessageListener)

- application.yaml

```yaml
  spring:
    kafka:
      consumer:
        bootstrap-servers: localhost:9092
      listener:
      type: RECORD
```

- Listener를 사용하기 위해서는 KafkaListener 어노테이션을 포함한 메서드를 선언필요. 
- KafkaListener 어노테이션에 포함된 파라미터에 따라 메서드에 필요한 파라미터 종류가 상이.

```java
  package com.example;

  import org.apache.kafka.clients.consumer.ConsumerRecord;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.kafka.annotation.KafkaListener;
  import org.springframework.kafka.annotation.PartitionOffset;
  import org.springframework.kafka.annotation.TopicPartition;

  @SpringBootApplication
  public class SpringConsumerApplication {
    public static Logger logger = LoggerFactory.getLogger(SpringConsumerApplication.class);

    public static void main(String[] args) {
      SpringApplication application = new SpringApplication(SpringConsumerApplication.class);
      application.run(args);
    }
    
    // 가장 기본적인 Listener 선언
    @KafkaListener(topics = "test",
          groupId = "test-group-00")
    public void recordListener(ConsumerRecord<String,String> record) {
      // Message 키, Message 값에 대한 처리
      logger.info(record.toString());
    }

    // Message 값을 파라미터로 받는 Listener
    @KafkaListener(topics = "test",
          groupId = "test-group-01")
    public void singleTopicListener(String messageValue) {
        logger.info(messageValue);
    }

    // 개별 Listener에 Kafka Consumer 옵션값 부여 (properties 옵션)
    @KafkaListener(topics = "test",
          groupId = "test-group-02", properties = {
          "max.poll.interval.ms:60000",
          "auto.offset.reset:earliest"
    })
    public void singleTopicWithPropertiesListener(String messageValue) {
      logger.info(messageValue);
    }

    // concurrency 옵션값에 해당하는 만큼 Consumer 스레드 생성 (병렬처리)
    @KafkaListener(topics = "test",
          groupId = "test-group-03",
          concurrency = "3")
    public void concurrentTopicListener(String messageValue) {
      logger.info(messageValue);
    }

    // 특정 토픽의 특정 파티션 구독 (topicPartitions 파라미터 사용)
    // PartitionOffset: 특정 오프셋 지정, 그룹 아이디에 관계없이 항상 설정한 오프셋의 데이터부터 가져옴
    @KafkaListener(topicPartitions =
          {
            @TopicPartition(topic = "test01", partitions = {"0", "1"}),
            @TopicPartition(topic = "test02", partitionOffsets = @PartitionOffset(partition = "0", initialOffset = "3"))
          },
          groupId = "test-group-04")
    public void listenSpecificPartition(ConsumerRecord<String, String> record) {
      logger.info(record.toString());
    }
  }
``` 

#### 4-2. 배치 Listener(BatchMessageListener)

- application.yaml

```yaml
  spring:
    kafka:
      consumer:
        bootstrap-servers: localhost:9092
      listener:
        type: BATCH
```

- 배치 Listener는 레코드 Listener와 다르게 KafkaListener로 사용되는 메서드의 파라미터를 List 또는 ConsumerRecords로 받음.

```java
  package com.example;

  import org.apache.kafka.clients.consumer.ConsumerRecords;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.kafka.annotation.KafkaListener;
  import java.util.List;

  @SpringBootApplication
  public class SpringConsumerApplication {
    public static Logger logger = LoggerFactory.getLogger(SpringConsumerApplication.class);

    public static void main(String[] args) {
      SpringApplication application = new SpringApplication(SpringConsumerApplication.class);
      application.run(args);
    }

    // ConsumerRecords
    @KafkaListener(topics = "test",
          groupId = "test-group-01")
    public void batchListener(ConsumerRecords<String, String> records) {
      records.forEach(record -> logger.info(record.toString()));
    }

    // Message 값들을 List 자료구조로 받아서 처리
    @KafkaListener(topics = "test",
          groupId = "test-group-02")
    public void batchListener(List<String> list) {
      list.forEach(recordValue -> logger.info(recordValue));
    }

    // concurrency = "3": 3개의 Consumer 스레드 생성
    @KafkaListener(topics = "test",
          groupId = "test-group-03",
          concurrency = "3")
    public void concurrentBatchListener(ConsumerRecords<String, String> records) {
      records.forEach(record -> logger.info(record.toString()));
    }
  }
```

#### 4-3. 배치 커밋 Listener(BatchAcknowledgingMessageListener)와 배치 Consumer 리스터(BatchConsumerAwareMessageListener)

- 동기, 비동기 커밋이나 Consumer 인스턴스에서 제공하는 메서드들을 활용하고 싶다면 배치 Consumer Listener를 사용. 
- Consumer 컨테이너에서 관리하는 AckMode를 사용하여 커밋하고 싶다면 배치 커밋 Listener를 사용. 
- AckMode, Consumer 양쪽모두 사용시에는 배치 커밋 Consumer Listener(BatchAcknowledgingConsumerAwareMessageListener)를 사용.

- application.yaml

```yaml
  spring:
    kafka:
      consumer:
        bootstrap-servers: localhost:9092
      listener:
        type: BATCH
        ack-mode: MANUAL_IMMEDIATE
```

```java
  package com.example;

  import org.apache.kafka.clients.consumer.Consumer;
  import org.apache.kafka.clients.consumer.ConsumerRecords;
  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.kafka.annotation.KafkaListener;
  import org.springframework.kafka.support.Acknowledgment;

  @SpringBootApplication
  public class SpringConsumerApplication {
    public static Logger logger = LoggerFactory.getLogger(SpringConsumerApplication.class);


    public static void main(String[] args) {
      SpringApplication application = new SpringApplication(SpringConsumerApplication.class);
      application.run(args);
    }

    // BatchAcknowledgingMessageListener
    @KafkaListener(topics = "test", groupId = "test-group-01")
    public void commitListener(ConsumerRecords<String, String> records, Acknowledgment ack) {
      records.forEach(record -> logger.info(record.toString()));
      ack.acknowledge(); // 커밋 수행
    }

    // BatchConsumerAwareMessageListener: Listener가 커밋하지 않도록 AckMode: MANUAL or MANUAL_IMMEDIATE 설정
    @KafkaListener(topics = "test", groupId = "test-group-02")
    public void consumerCommitListener(ConsumerRecords<String, String> records, Consumer<String, String> consumer) {
      records.forEach(record -> logger.info(record.toString()));
      consumer.commitSync();
    }
  }
```

### 5. Custom Listener 컨테이너

- 서로 다른 설정을 가진 2개 이상의 Listener를 구현하거나 리밸런스 Listener를 구현하기 위해서 Custom Listener 컨테이너를 사용. 
- Custom Listener 컨테이너를 만들기 위해서 Spring Kafka에서 Kafka Listener 컨테이너 팩토리(KafkaListenerContainerFactory) 인스턴스를 생성필요. 
- Kafka Listener 컨테이너 팩토리를 빈으로 등록하고 KafkaListener 어노테이션에서 Custom Listener 컨테이너 팩토리를 등록하면 Custom Listener 컨테이너를 사용가능.

```java
  package com.example;

  import org.apache.kafka.clients.consumer.Consumer;
  import org.apache.kafka.clients.consumer.ConsumerConfig;
  import org.apache.kafka.common.serialization.StringDeserializer;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
  import org.springframework.kafka.config.KafkaListenerContainerFactory;
  import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
  import org.springframework.kafka.listener.*;
  import org.apache.kafka.common.TopicPartition;

  import java.util.Collection;
  import java.util.HashMap;
  import java.util.Map;

  @Configuration
  public class ListenerContainerConfiguration {

    // customContainerFactory메서드명: KafkaListener 어노테이션 컨테이너 팩토리 등록 시 사용
    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> customContainerFactory() {

      Map<String, Object> props = new HashMap<>();
      props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
      props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
      props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

      // DefaultKafkaConsumerFactory: Consumer 기본 옵션을 설정하는 용도
      DefaultKafkaConsumerFactory cf = new DefaultKafkaConsumerFactory<>(props);

      // Listener 컨테이너를 만들기 위해 사용 (2개 이상의 Consumer Listener를 만들 때 사용)
      ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
      
      // 리밸런스 Listener를 선언하기 위해 setConsumerRebalanceListener 메서드 호출
      factory.getContainerProperties().setConsumerRebalanceListener(new ConsumerAwareRebalanceListener() {
        @Override
        public void onPartitionsRevokedBeforeCommit(Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {
          // 커밋이 되기 전, 리밸런스 발생 시 호출된다.
        }

        @Override
        public void onPartitionsRevokedAfterCommit(Consumer<?, ?> consumer, Collection<TopicPartition> partitions) {
          // 커밋이 일어난 이후, 리밸런스 발생 시 호출된다.
        }

        @Override
        public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        }

        @Override
        public void onPartitionsLost(Collection<TopicPartition> partitions) {
        }
      });
      factory.setBatchListener(false);
      // AckMode 설정
      factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.RECORD);
      
      // Consumer 설정값을 가지고 있는 DefaultKafkaConsumerFactory 인스턴스를 팩토리에 설정
      factory.setConsumerFactory(cf);
      return factory;
    }
  }
```

```java
  package com.example;

  import org.slf4j.Logger;
  import org.slf4j.LoggerFactory;
  import org.springframework.boot.SpringApplication;
  import org.springframework.boot.autoconfigure.SpringBootApplication;
  import org.springframework.kafka.annotation.KafkaListener;

  @SpringBootApplication
  public class SpringConsumerApplication {
    
    public static Logger logger = LoggerFactory.getLogger(SpringConsumerApplication.class);

    public static void main(String[] args) {
      SpringApplication application = new SpringApplication(SpringConsumerApplication.class);
      application.run(args);
    }

    @KafkaListener(topics = "test",
          groupId = "test-group",
          containerFactory = "customContainerFactory")
    public void customListener(String data) {
      logger.info(data);
    }
  }
```




<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>