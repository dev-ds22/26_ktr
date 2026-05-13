---
layout: single
title: "[Kafka] KafkaListener Multi Topic 사용하기"
excerpt: "KafkaListener topic 여러개 사용하기"

categories:
  - tech
tags:
  - [kafka, topic]

toc: false
toc_sticky: true

date: 2023-01-12
last_modified_at: 2023-01-12
---

# Spring - @KafkaListener topic 여러개 사용

## KafkaListener를 구현예제.

### 1. application.yaml에 토픽 이름 설정.

```yaml
kafka:
  topic:
    member: member-topic
```

- 설정에 있는 토픽명을 사용하기 위해 topics = "${kafka.topic.member}"를 입력.

```java
  @KafkaListener(id = "myGroupId", topics = "${kafka.topic.member}")
  public void kafkaListener(MessageDto messageDto) {
    switch (messageDto.getType()) {
      case REGISTER: {
          memberService.register(messageDto);
          break;
      }
      case WITHDRAWAL: {
          memberService.withdrawal(messageDto.getId());
          break;
      }
      default: {
          throw new RuntimeException("wrong type error. type=" + messageDto.getType());
      }
    }
  }
```

### 2. Topic 분리 또는 Topic명 변경의 경우

- 그냥 바꿔 버리면 일시적으로 Message를 못 받다가 유실가능.
- 기존 토픽도 사용하고 새 토픽도 사용하는 상태로 변경.

- application.yaml에 새로운 토픽 이름을 추가.

```yaml
kafka:
  topic:
    member-old: member-topic
    member: member-topic-sandbox
```

- KafkaListener 새성 시 토픽을 두 개 설정.

```java
  @KafkaListener(id = "myGroupId", topics = {"${kafka.topic.member-old}", "${kafka.topic.member}"})
  public void kafkaListener(MessageDto messageDto) {
    switch (messageDto.getType()) {
      case REGISTER: {
          memberService.register(messageDto);
          break;
      }
      case WITHDRAWAL: {
          memberService.withdrawal(messageDto.getId());
          break;
      }
      default: {
          throw new RuntimeException("wrong type error. type=" + messageDto.getType());
      }
    }
  }
```

### 3. 토픽의 개수 가변적으로 변경 시.

- application.yaml 설정에서 쉼표(,)로 토픽 리스트를 설정.

```yaml
kafka:
  topic:
    member: member-topic,member-topic-sandbox
```

- 가져다 사용할 때 SpEL (Spring Expression Language)를 사용.

```java
  @KafkaListener(id = "myGroupId", topics = "#{'${kafka.topic.member}'.split(',')}")
  public void kafkaListener(MessageDto messageDto) {
    switch (messageDto.getType()) {
      case REGISTER: {
          memberService.register(messageDto);
          break;
      }
      case WITHDRAWAL: {
          memberService.withdrawal(messageDto.getId());
          break;
      }
      default: {
          throw new RuntimeException("wrong type error. type=" + messageDto.getType());
      }
    }
  }
```

- kafka.topic.member의 설정을 쉼표(,)로 split 해서 리스트로 만든다는 의미.
- SpEL을 이용해서 설정은 하나인데 여러 개의 토픽을 쉽게 사용.

--- 
### kafkaListener 속성 
- autoStartup : 컨테이너 팩터리의 기본 설정을 재정의하려면 true 또는 false로 설정.
- batch : 컨테이너 팩터리의 batchListener속성을 재정의.
- beanRef : 이 리스너가 정의된 현재 bean을 참조하기 위해 이 주석 내의 SpEL 표현식에 사용되는 pseudo bean name.
- clientIdPrefix : 제공되면 consumer factory configuration의 client id 속성을 재정의.
- concurrency : container factory's concurrency setting의 concurrency listener을 재정의.
- containerFactory : The bean name of the KafkaListenerContainerFactory to use to create the message listener container responsible to serve this endpoint.
- containerGroup : If provided, the listener container for this listener will be added to a bean with this value as its name, of type Collection<MessageListenerContainer>.
- contentTypeConverter : Set the bean name of a SmartMessageConverter (such as the CompositeMessageConverter) to use in conjunction with the MessageHeaders.CONTENT_TYPE header to perform the conversion to the required type.
- errorHandler : KafkaListenerErrorHandler리스너 메서드가 예외를 던질 경우 호출할 빈 이름을 설정.
- filter : RecordFilterStrategy컨테이너 팩토리에 구성된 전략을 재정의하도록 빈 이름을 설정.
- groupId : group.id이 리스너에 대해서만 이 값으로 소비자 팩토리 의 속성을 재정의.
- id : listener에 대한 컨테이너의 고유 식별자.
- idIsGroup : When groupId is not provided, use the id (if provided) as the group.id property for the consumer.
- info : 키가 있는 헤더로 추가될 정적 정보입니다 KafkaHeaders.LISTENER_INFO.
- properties : Kafka consumer properties; they will supersede any properties with the same name defined in the consumer factory (if the consumer factory supports property overrides).
- splitIterables : When false and the return type is an Iterable return the result as the value of a single reply record instead of individual records for each element..
- topicPartitions : The topicPartitions for this listener when using manual topic/partition assignment.
- topicPattern : The topic pattern for this listener.
- topics : The topics for this listener.

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

  </pre>
</details>