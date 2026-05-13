---
layout: single
title: "[KAFKA] Kafka & Zookeeper"
excerpt: "KAFKA - Zookeeper"

categories:
  - tech
tags:
  - [tech, kafka, zookeeper]

toc: false
toc_sticky: true

date: 2022-12-15
last_modified_at: 2022-12-15
---
# Zookeeper

- 분산 코디네이션 서비스를 제공하는 오픈소스 프로젝트로 직접 어플리케이션 작업을 조율하는 것을 쉽게 개발할 수 있도록 도와주는 도구. 
  - 분산 코디네이션 서비스 : 분산 시스템에서 시스템 간의 정보 공유, 상태 체크, 서버들 간의 동기화를 위한 락 등을 처리해주는 서비스
- API를 이용해 동기화나 마스터 선출 등의 작업을 쉽게 구현가능.
- zookeeper는 Apache의 오픈 소스 프로젝트 중 하나이며, 공개 분산형 구성 서비스, 동기 서비스 및 대용량 분산 시스템을 위한 NameSpace 레지스트리를 제공함.
- znode(주키퍼 노드)는 NameSpace 안에 데이터를 저장하며, 클라이언트는 znode를 통해 데이터를 읽거나 씀.
- Leader 노드가 존재하며, Leader를 제외한 노드들은 Leader로부터 데이터를 동기화함.
- Leader에 장애가 발생하게 되면 다른 노드가 Leader의 역할을 맡아 안정성이 뛰어남.
- 일반적인 파일 시스템과 달리 zookeeper 데이터는 메모리에 보관되므로 높은 처리량과 낮은 대기 시간을 갖음.

![kafka_zookeeper](./../../images/tech/inst_zookeeper_01.png)
(위의 그림에서 Server는 Zookeeper, Client는 Kafka)

- 다운로드 : https://kafka.apache.org/downloads
- tar -xzf kafka_2.13-2.6.0.tgz

## Zookeeper 설정

### zookeeper.properties

```powershell
  # snapshot 데이터를 저장할 경로를 지정
  dataDir=C:/daiso/kafka/kafka_2.13-3.3.1/data/zookeeper

  # 클라이언트가 connect할 port 번호 지정
  clientPort=2181

  # 하나의 클라이언트에서 동시 접속하는 개수 제한, 기본값은 60이며, 0은 무제한
  maxClientCnxns=0

  # port 충돌을 방지하려면 admin server 비활성화(false)
  admin.enableServer=false
  # admin.serverPort=8080

  # 멀티 서버 설정
  # server.id=host:port:port
  server.1=localhost:2888:3888
  # server.2=server_host_1:2888:3888
  # server.3=server_host_2:2888:3888
  
  # 멀티 서버 설정시 각 서버의 dataDir 밑에 myid 파일이 있어야함.
  # echo 1 > myid
  
  # 리더 서버에 연결해서 동기화하는 시간, [멀티서버옵션]
  #initLimit=5
  
  # 리더 서버를 제외한 노드 서버가 리더와 동기화하는 시간, [멀티서버옵션]
  #syncLimit=2
```
  
## Zookeeper 실행하기

```powershell
  $ cd C:\daiso\kafka\kafka_2.13-3.3.1
  # bin/zookeeper-server-start.sh config/zookeeper.properties
  $ ./bin/windows/zookeeper-server-start.bat ./config/zookeeper.properties
```

- zookeeper-server-start.bat
  - 윈도우에서 zookeeper 서버를 실행하는 파일
- zookeeper.properties
  - zookeeper 서버 설정 파일
  - 포트 바인딩 등 설정 가능 (default = 2181포트)
  - 추가적인 설정은 공식문서를 확인

- zookeeper-server-start.bat 파일은 인자 값으로 zookeeper.properties를 넘겨주어야 한다.
- zookeeper를 실행하면 위에서 설정한 dataDir 경로에 log, snapshot 데이터가 저장되는 것을 확인할 수 있다.
- zookeeper는 필요한 설정이 있을 경우 공식문서를 참고해 properties를 추가하는 방식으로 사용

```bash
  [2022-12-15 15:50:44,948] INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)
  [2022-12-15 15:50:44,969] INFO Using org.apache.zookeeper.server.watch.WatchManager as watch manager (org.apache.zookeeper.server.watch.WatchManagerFactory)
  [2022-12-15 15:50:44,970] INFO Using org.apache.zookeeper.server.watch.WatchManager as watch manager (org.apache.zookeeper.server.watch.WatchManagerFactory)
  [2022-12-15 15:50:44,971] INFO zookeeper.snapshotSizeFactor = 0.33 (org.apache.zookeeper.server.ZKDatabase)
  [2022-12-15 15:50:44,971] INFO zookeeper.commitLogCount=500 (org.apache.zookeeper.server.ZKDatabase)
  [2022-12-15 15:50:44,974] INFO zookeeper.snapshot.compression.method = CHECKED (org.apache.zookeeper.server.persistence.SnapStream)
  [2022-12-15 15:50:44,980] INFO Reading snapshot \tmp\zookeeper\version-2\snapshot.0 (org.apache.zookeeper.server.persistence.FileSnap)
  [2022-12-15 15:50:44,983] INFO The digest value is empty in snapshot (org.apache.zookeeper.server.DataTree)
  [2022-12-15 15:50:45,019] INFO ZooKeeper audit is disabled. (org.apache.zookeeper.audit.ZKAuditProvider)
  [2022-12-15 15:50:45,020] INFO 160 txns loaded in 28 ms (org.apache.zookeeper.server.persistence.FileTxnSnapLog)
  [2022-12-15 15:50:45,021] INFO Snapshot loaded in 50 ms, highest zxid is 0xa0, digest is 277975764531 (org.apache.zookeeper.server.ZKDatabase)
  [2022-12-15 15:50:45,021] INFO Snapshotting: 0xa0 to \tmp\zookeeper\version-2\snapshot.a0 (org.apache.zookeeper.server.persistence.FileTxnSnapLog)
  [2022-12-15 15:50:45,024] INFO Snapshot taken in 4 ms (org.apache.zookeeper.server.ZooKeeperServer)
  [2022-12-15 15:50:45,034] INFO PrepRequestProcessor (sid:0) started, reconfigEnabled=false (org.apache.zookeeper.server.PrepRequestProcessor)
  [2022-12-15 15:50:45,035] INFO zookeeper.request_throttler.shutdownTimeout = 10000 (org.apache.zookeeper.server.RequestThrottler)
  [2022-12-15 15:50:45,047] INFO Using checkIntervalMs=60000 maxPerMinute=10000 maxNeverUsedIntervalMs=0 (org.apache.zookeeper.server.ContainerManager)
```

## Kafka 설정

### server.properties

- Kafka 관련 설정 파일
- 자세한 내용은 공식문서를 확인

```powershell
  # config/server.properties
  ############################# Server Basics #############################
  
  # Broker의 ID로 Cluster내 Broker를 구분하기 위해 사용(Unique 값)
  broker.id=0
  
  ############################# Socket Server Settings #############################
  # Broker가 사용하는 호스트와 포트를 지정, 형식은 PLAINTEXT://your.host.name:port 을 사용
  listeners=PLAINTEXT://:9092
  
  # Producer와 Consumer가 접근할 호스트와 포트를 지정, 기본값은 listeners를 사용
  advertised.listeners=PLAINTEXT://localhost:9092
  
  # 네트워크 요청을 처리하는 Thread의 개수, 기본값 3
  num.network.threads=3
  
  # I/O가 생길때 마다 생성되는 Thread의 개수, 기본값 8
  num.io.threads=8
  
  # socket 서버가 사용하는 송수신 버퍼 (SO_SNDBUF, SO_RCVBUF) 사이즈, 기본값 102400
  socket.send.buffer.bytes=102400
  socket.receive.buffer.bytes=102400
  
  # 서버가 받을 수 있는 최대 요청 사이즈이며, 서버 메모리가 고갈 되는 것 방지
  # JAVA의 Heap 보다 작게 설정해야 함, 기본값 104857600
  socket.request.max.bytes=104857600
  
  ############################# Log Basics #############################
  
  # 로그 파일을 저장할 디렉터리의 쉼표로 구분할 수 있음
  log.dirs=C:/dev/kafka_2.13-2.6.0/logs
  
  # 토픽당 파티션의 수를 의미, 
  # 입력한 수만큼 병렬처리 가능, 데이터 파일도 그만큼 늘어남
  num.partitions=1
  
  # 시작 시 log 복구 및 종료 시 flushing에 사용할 데이터 directory당 Thread 개수
  # 이 값은 RAID 배열에 데이터 directory에 대해 증가하도록 권장 됨
  num.recovery.threads.per.data.dir=1
  
  ############################# Internal Topic Settings #############################
  # 내부 Topic인 "_consumer_offsets", "_transaction_state"에 대한 replication factor
  # 개발환경 : 1, 운영할 경우 가용성 보장을 위해 1 이상 권장(3 정도)
  offsets.topic.replication.factor=1
  transaction.state.log.replication.factor=1
  transaction.state.log.min.isr=1
  
  ############################# Log Retention Policy #############################
  
  # 세그먼트 파일의 삭제 주기, 기본값 hours, 168시간(7일)
  # 옵션 [ bytes, ms, minutes, hours ] 
  log.retention.hours=168
  
  # 토픽별로 수집한 데이터를 보관하는 파일
  # 세그먼트 파일의 최대 크기, 기본값 1GB
  # 세그먼트 파일의 용량이 차면 새로운 파일을 생성
  log.segment.bytes=1073741824
  
  # 세그먼트 파일의 삭제 여부를 체크하는 주기, 기본값 5분(보존 정책)
  log.retention.check.interval.ms=300000
  
  ############################# Zookeeper #############################
  
  # 주키퍼의 접속 정보
  # 쉼표(,)로 많은 연결 서버 포트 설정 가능
  # 모든 kafka znode의 Root directory
  zookeeper.connect=localhost:2181
  
  # 주키퍼 접속 시도 제한시간(time out)
  zookeeper.connection.timeout.ms=18000
  
  
  ############################# Group Coordinator Settings #############################
  
  # GroupCoordinator 설정 - 컨슈머 rebalance를 지연시키는 시간
  # 개발환경 : 테스트 편리를 위해 0으로 정의
  # 운영환경 : 3초의 기본값을 설정하는게 좋음
  group.initial.rebalance.delay.ms=0

```  

## Kafka 실행

- 새로운 터미널을 열어서 Kafka 실행.
- kafka 압축을 푼 base 폴더로 이동한 후 다음 명령어 입력.

```powershell
  $ cd C:\daiso\kafka\kafka_2.13-3.3.1
  $ bin/windows/kafka-server-start.bat config/server.properties
```

- kafka-server-start.bat
  - 윈도우 OS에서 kafka 서버 실행 파일


```bash
  [2022-12-15 15:15:18,930] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
  [2022-12-15 15:15:18,993] INFO [BrokerToControllerChannelManager broker=0 name=forwarding]: Recorded new controller, from now on will use broker localhost:9092 (id: 0 rack: null) (kafka.server.BrokerToControllerRequestThread)
  [2022-12-15 15:15:19,009] INFO [BrokerToControllerChannelManager broker=0 name=alterPartition]: Recorded new controller, from now on will use broker localhost:9092 (id: 0 rack: null) (kafka.server.BrokerToControllerRequestThread)
```  

## 실행 확인

### 1. Topic 생성하기

> ## 하기전 해야하는 것  
> # /kafka/config/server.properties 에 들어가서   
> delete.topic.enable = true
>   
> ## topic 만들기  
> bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic dev-topic
>  
> ## 만들어졌는지 확인  
> bin/kafka-topics.sh --list --bootstrap-server localhost:9092  
>    
> ## 안에 머라고 궁시렁 궁시렁  
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic dev-topic  
>  
> ## 안에 머라 적었는지 확인  
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic dev-topic --from-beginning  
>  
> ## topic 제거하기  
> bin/kafka-topics.sh --delete --bootstrap-server localhost:9092  --topic dev-topic  
>   
> ## 없어졌는지 확인    
> bin/kafka-topics.sh --list --bootstrap-server localhost:9092
>    

- localhost:9092 카프카 서버에 dev-topic란 토픽을 생성.

```bash
  # $ bin/windows/kafka-topics.bat --create --topic dev-topic --bootstrap-server localhost:9092
  # bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic dev-topic
  $ bin/windows/kafka-topics.bat --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic dev-topic
```
  

```bash
  Created topic dev-topic.
```

- 현재 만들어져 있는 토픽 확인

```powershell
  # bin/kafka-topics.sh --list --bootstrap-server localhost:9092
  $ bin/windows/kafka-topics.bat --list --bootstrap-server localhost:9092
```
  

```bash
  dev-topic
```

- 특정 토픽의 설정 확인

```powershell
  # bin/kafka-topics.sh --describe --topic dev-topic --bootstrap-server localhost:9092
  $ bin/windows/kafka-topics.bat --describe --topic dev-topic --bootstrap-server localhost:9092
```
  

```bash
PS C:\daiso\kafka\kafka_2.13-3.3.1> bin/windows/kafka-topics.bat --describe --topic dev-topic --bootstrap-server localhost:9092
Topic: dev-topic        TopicId: qPKE8o4jTIip9406OJMVKA PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: dev-topic        Partition: 0    Leader: 0       Replicas: 0     Isr: 0
PS C:\daiso\kafka\kafka_2.13-3.3.1>
```

### 2. Producer, Consumer 실행하기

- 콘솔에서 Producer와 Consumer를 실행하여 실시간으로 토픽에 event를 추가하고 받을 수 있다.
- 터미널을 분할로 띄워서 진행.

#### Producer
- Producers: Topic에 메시지 보내기
- "생산자(Producer)"는 데이터를 Kafka 클러스터에 넣는 프로세스.
- bin 디렉토리의 명령어는 콘솔에 텍스트를 입력할 때마다 클러스터에 데이터를 입력하는 콘솔 생성자(Producer)를 제공.
- 콘솔 생산자(Producer)를 시작하려면 다음 명령을 실행.

```bash
  # bin/kafka-console-producer.sh --topic dev-topic --bootstrap-server localhost:9092
  bin/windows/kafka-console-producer.bat --topic dev-topic --bootstrap-server localhost:9092
  # bin/windows/kafka-console-producer.bat --broker-list localhost:9093,localhost:9094,localhost:9095 --topic dev-topic
```

```bash
  > dev-topic test~~~~~~~~~~~~~~~~~
  >
```

- Broker-list는 생산자가 방금 프로비저닝한 브로커의 주소.
- topic은 데이터가 입력하려는 주제(topic)를 지정.
- 데이터를 입력하고 Enter 키를 치면, Kafka 클러스터에 텍스트를 입력할 수 있는 명령 프롬프트가 표시.

- 컨슈머가 데이터를 가져가도 Topic 데이터는 삭제되지 않음.

> first insert  
> 두번째 입력  

- 프로듀서가 데이터를 보낼 때 '파티션키'를 지정해서 전송가능.
- 파티션키를 지정하지 않으면, 라운드로빈 방식으로 파티션에 저장.
- 파티션키를 지정하면, 파티셔너가 파티션키의 HASH 값을 구해서 특정 파티션에 할당.
- 카프카에서 kafka-console-producer.sh로 Consumer에게 메세지를 보낼 때 기본적으로 key값이 null로 설정.

#### Consumer
- Kafka consumers 를 사용하여 클러스터에서 데이터를 취득
- consumer를  시작하려면 다음 명령을 실행합니다. 
  - pruducer가 입력한 데이터가 출력.

```bash
  $ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic dev-topic --from-beginning
```

- bootstrap-server는 클러스터의 브로커 중 하나.
- Topic은 생산자가 클러스터에 데이터를 입력한 Topic(주제).
- 0 from-beginning은 클러스터에 현재 가지고 있는 모든 메시지를 원한다고 클러스터에 알림.
  - 컨슈머 그룹이 다른 새로운 컨슈머가 auto.offset.reset=earliest 설정으로 데이터를 0번부터 취득. 
    - 설정하지 않으면 새롭게 토픽에 생성된 메세지만 취득.
- 위의 명령을 실행하면 콘솔에 로그온한 생산자가 입력한 모든 메시지가 즉시 표시.
- 소비자가 실행되는 동안 생산자가 메시지를 입력하면 실시간으로 콘솔에 출력.

- 카프카에서는 consumer 그룹이라는 개념존재 --consumer-property group.id=group-01 형식으로 consumer 그룹지정가능 
- 카프카 브로커는 컨슈머 그룹 중 하나의 컨슈머에게만 이벤트를 전달
- 동일한 이벤트 처리를 하는 컨슈머를 clustering 한 경우에 컨슈머 그룹으로 지정하면 클러스터링된 컨슈머 중 하나의 서버가 데이터를 수신.

- key와 value를 콘솔창에 표시하기 위해서는 --property print.key=true --property key.separator=: 를 설정.

```bash
  bin/kafka-console-consumer.sh --bootstrap-server localhost:9093 --topic my-kafka-topic --from-beginning \
    --property print.key=true --property key.separator=:

  "key":"second key value"
  "secone message":null
  key3:third
```

#### Consumer Group
기존의 Message Queue 솔루션에서는 컨슈머가 메시지를 가져가면, 해당 메세지를 큐에서 삭제된다. 즉, 하나의 큐에 대하여 여러 컨슈머가 붙어서 같은 메세지를 컨슈밍할 수 없다. 하지만 Kafka는, 컨슈머가 메세지를 가져가도 큐에서 즉시 삭제되지 않으며, 하나의 토픽에 여러 컨슈머 그룹이 붙어 메세지를 가져갈 수 있다.

또한 각 consumer group마다 해당 topic의 partition에 대한 별도의 offset을 관리하고, group에 컨슈머가 추가/제거 될 때마다 rebalancing을 하여 group 내의 consumer에 partition을 할당하게 된다. 이는 컨슈머의 확장/축소를 용이하게 하고, 하나의 MQ(Message Queue)를 컨슈머 별로 다른 용도로 사용할 수 있는 확장성을 제공한다.

아래 명령어로 컨슈머 그룹을 지정합니다.
- $ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic [토픽 이름] --group [그룹 이름]

```bash
  $ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic dev-topic --group dev-group
```
그룹에 속하는 컨슈머가 여러개이면 로드밸런싱을 통해 자동으로 메세지를 분배
2. 그룹 정보 확인

```bash
  $ bin/kafka-console-groups.sh --bootstrap-server localhost:9092 --describe --group dev-group
```

- 위에 대한 명령어로 아래의 정보들을 얻을 수 있다.
  - CURRENT-OFFSET : Consumer Group이 Kafka에서 읽은 offset
  - LOG-END-OFFSET : 해당 topic, partition의 마지막 offset
  - LAG : LOG-END-OFFSET 과 CURRENT-OFFSET의 차이
    - LAG의 경우 topic의 partition단위로 읽어야 할 남은 데이터 수를 의미한다
 
```powershell
  # bin/kafka-console-consumer.sh --topic dev-topic --from-beginning --bootstrap-server localhost:9092
  bin/windows/kafka-console-consumer.bat --topic dev-topic --from-beginning --bootstrap-server localhost:9092
```

```bash
  send test~~~~~
```

- 스프링 구현관련해서 이하 사이트내용 정리필요  
  - https://sup2is.github.io/2020/06/03/spring-boot-with-kafka-cluster.html  
  - https://semtax.tistory.com/83


# CentOS에서 kafka 설치

## 1. 카프카 다운로드

```bash
  wget https://dlcdn.apache.org/kafka/3.0.0/kafka_2.13-3.0.0.tgz
  tar xzf kafka_2.13-3.0.0.tgz
  mv kafka_2.13-3.0.0 /usr/local/kafka
```

## 2.카프카 Systemd Unit Files 세팅

> # which java  
> /usr/bin/java​  
>  
> # javac의 심볼릭 링크 원본 찾기  
> readlink -f /usr/bin/javac  
>  
> # java의 심볼릭 링크 원본 찾기  
> readlink -f /usr/bin/java  
> 

```bash
vim /etc/systemd/system/zookeeper.service
```

```bash
  [Unit]
  Description=Apache Zookeeper server
  Documentation=http://zookeeper.apache.org
  Requires=network.target remote-fs.target
  After=network.target remote-fs.target

  [Service]
  Type=simple
  ExecStart=/usr/bin/bash /usr/local/kafka/bin/zookeeper-server-start.sh /usr/local/kafka/config/zookeeper.properties
  ExecStop=/usr/bin/bash /usr/local/kafka/bin/zookeeper-server-stop.sh
  Restart=on-abnormal

  [Install]
  WantedBy=multi-user.target
  

  vim /etc/systemd/system/kafka.service
  

  [Unit]
  Description=Apache Kafka Server
  Documentation=http://kafka.apache.org/documentation.html
  Requires=zookeeper.service

  [Service]
  Type=simple
  Environment="JAVA_HOME=/usr/lib/jvm/jre-11-openjdk"
  ExecStart=/usr/bin/bash /usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties
  ExecStop=/usr/bin/bash /usr/local/kafka/bin/kafka-server-stop.sh

  [Install]
  WantedBy=multi-user.target
```

```bash
  systemctl daemon-reload
```

## 3. Kafka 서버 시작

```bash
sudo systemctl start zookeeper
sudo systemctl start kafka
sudo systemctl status kafkanetst
```

## 4. 포트 오픈 (필수 아님)

```bash
firewall-cmd --permanent --zone=public --add-port=9092/tcp
firewall-cmd --reload
```

## 5. 외부 접속 오픈 (필수 아님)

```bash
  vi /usr/local/kafka/config/server.properties
  # Hostname and port the broker will advertise to producers and consumers. If not set,
  # it uses the value for "listeners" if configured.  Otherwise, it will use the value
  # returned from java.net.InetAddress.getCanonicalHostName().
  advertised.listeners=PLAINTEXT://192.168.0.1:9092
```

## 6. 외부 접속 오픈 (필수 아님)

```bash
sudo systemctl restart kafka
sudo systemctl status kafka
```

## 7. Setting up a multi-broker Cluster(클러스터 구성)

```bash
  $ cp ./config/server.properties ./config/server-1.properties 
  $ cp ./config/server.properties ./config/server-2.properties
```

```bash
  config/server-1.properties:
      broker.id=1
      listeners=PLAINTEXT://:9093
      log.dirs=/tmp/kafka-logs-1

  config/server-2.properties:
      broker.id=2
      listeners=PLAINTEXT://:9095
      log.dirs=/tmp/kafka-logs-2
```

- Broker 서버를 띄울 properties 파일을 추가적으로 생성해주고 포트와 로그 경로를 변경해준다.
- 이후에, 추가한 Broker 서버들을 기동해준다.

```bash
  $ bin/kafka-server-start.sh config/server-1.properties &
  $ bin/kafka-server-start.sh config/server-2.properties &
```

- 기동시 로그에 아래와 같이 클러스터가 정상적으로 연결됐다는 메세지를 확인할 수 있다.
- server-0

```bash
  $ INFO [Partition my-replicated-topic-0 broker=0] ISR updated to 0,1 and version updated to [6] (kafka.cluster.Partition)
  $ INFO [Partition my-replicated-topic-0 broker=0] ISR updated to 0,1,2 and version updated to [7] (kafka.cluster.Partition)
```

- server-1

```bash
  $ INFO [BrokerToControllerChannelManager broker=1 name=alterIsr]: Recorded new controller, from now on will use broker sydev:9092 (id: 0 rack: null) (kafka.server.BrokerToControllerRequestThread)
```

- server-2

```bash
  $ INFO [BrokerToControllerChannelManager broker=2 name=forwarding]: Recorded new controller, from now on will use broker sydev:9092 (id: 0 rack: null) (kafka.server.BrokerToControllerRequestThread)
```

## 8. 클러스터 Fail-Over 테스트

### 8-1. 먼저 Replica Factor를 3개 추가해서 클러스터용 신규 토픽을 생성한다.

```bash
  $ bin/kafka-topics.sh --create --bootstrap-server 10.222.10.170:9092 --replication-factor 3 --partitions 1 --topic dev-replicated-topic
  
  ...
  Created topic my-replicated-topic.
```

- 생성된 토픽에 대한 자세한 Describe 설명 조회

```log
  $ bin/kafka-topics.sh --describe --bootstrap-server 10.222.10.170:9092 --topic dev-replicated-topic

  Topic: my-replicated-topic TopicId: qf1AFc-STwiZ6uOBX13Amw PartitionCount: 1 ReplicationFactor: 3 Configs: segment.bytes=1073741824
  Topic: my-replicated-topic Partition: 0 Leader: 1 Replicas: 0,2,1 Isr: 1,0,2
```
  - Leader : 지정된 파티션에 대한 모든 읽기/쓰기를 담당하는 노드이다. 현재 나의 예제에서는 Leader가 1번(*:9093)이다.

### 8-2. Fail-Over 테스트

#### 프로듀서 실행

```log
  $ bin/kafka-console-producer.sh --broker-list 10.222.10.170:9092 --topic dev-replicated-topic

  > my test message 1
  > my test message 2
```

#### 컨슈머 실행

```log
  $ bin/kafka-console-consumer.sh --bootstrap-server 10.222.10.170:9092 --from-beginning --topic dev-replicated-topic

  my test message 1
  my test message 2
```

- 현재 리더 역할을 하고 있는 server-1을 kill -9 로 Shutdown 시킨다.

```log
  [root@sydev kafka_2.12-3.0.0]# netstat -anp|grep 9093
  tcp6       0      0 :::9093                 :::*                    LISTEN      28867/java      


  [root@sydev kafka_2.12-3.0.0]# kill -9 28867
```

##### Leader인 브로커를 죽인 이후,,

#### 프로듀서 

```log
  # $ bin/kafka-topics.sh --delete --bootstrap-server 10.222.10.170:9092  --topic dev-replicated-topic 
  # $ bin/kafka-console-producer.sh --broker-list 10.222.10.170:9092 --topic dev-replicated-topic
  $ bin/kafka-console-producer.sh --broker-list 10.222.10.170:9092,10.222.10.170:9093,localhost:9095 --topic dev-replicated-topic

  > my test message 1
  > my test message 2
  >
```

```log
  $ WARN [Producer clientId=console-producer] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  $ WARN [Producer clientId=console-producer] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  $ WARN [Producer clientId=console-producer] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  $ WARN [Producer clientId=console-producer] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  $ WARN [Producer clientId=console-producer] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)

  > after failover
  > you ok?
```

#### 컨슈머

```log
  my test message 1
  my test message 2

  $ WARN [Consumer clientId=consumer-console-consumer-3714-1, groupId=console-consumer-3714] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  $ WARN [Consumer clientId=consumer-console-consumer-3714-1, groupId=console-consumer-3714] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  $ WARN [Consumer clientId=consumer-console-consumer-3714-1, groupId=console-consumer-3714] Connection to node 1 (sydev/192.168.56.106:9093) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
  ...(생략)

  after failover
  you ok?
```

- 리더 브로커를 죽였지만, 이후에도 after failover, you ok? 메세지가 정상적으로 Producer를 통해 Consumer로 전달.

<details>
  <summary>Exp.</summary>  
  <pre>

### 참조

# zookeeper.properties

```bash
Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# the directory where the snapshot is stored.
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
# disable the per-ip limit on the number of connections since this is a non-production config
maxClientCnxns=0
# Disable the adminserver by default to avoid port conflicts.
# Set the port to something non-conflicting if choosing to enable this
admin.enableServer=false
# admin.serverPort=808

# 클러스터 구조에서 사용하는 설정인데 server설정은 각 서버의 ID로 사용
# 분산 서버에서는 자신이 사용할 server는 127.0.0.1로 host를 설정하고, 나머지는 분산 host로 변경
# server.1=10.222.10.170:2887:3887
# server.2=10.222.10.170:2888:3888
# server.3=10.222.10.170:2889:3889


# 분산예시
# server.1=127.0.0.1:2887:3887 # 자신
# server.2=151.2.3.4:2887:3887 # 분산
# server.3=561.3.1.3:2887:3887 # 분산

# Zookeeper 노드, Kafka와의 연결 Timeout 시간 설정
# initLimit=10
# syncLimit=50
```

# server.properties
```bash
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#
# This configuration file is intended for use in ZK-based mode, where Apache ZooKeeper is required.
# See kafka.server.KafkaConfig for additional details and defaults
#

############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.
broker.id=0

############################# Socket Server Settings #############################

# The address the socket server listens on. If not configured, the host name will be equal to the value of
# java.net.InetAddress.getCanonicalHostName(), with PLAINTEXT listener name, and port 9092.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
listeners=PLAINTEXT://pilot.daiso.com:9096

# Listener name, hostname and port the broker will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=PLAINTEXT://pilot.daiso.com:9096

# Maps listener names to security protocols, the default is for them to be the same. See the config documentation for more details
#listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600


############################# Log Basics #############################

# A comma separated list of directories under which to store log files
log.dirs=/tmp/kafka-logs

# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=1

# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown.
# This value is recommended to be increased for installations with data dirs located in RAID array.
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings  #############################
# The replication factor for the group metadata internal topics "__consumer_offsets" and "__transaction_state"
# For anything other than development testing, a value greater than 1 is recommended to ensure availability such as 3.
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

############################# Log Flush Policy #############################

# Messages are immediately written to the filesystem but by default we only fsync() to sync
# the OS cache lazily. The following configurations control the flush of data to disk.
# There are a few important trade-offs here:
#    1. Durability: Unflushed data may be lost if you are not using replication.
#    2. Latency: Very large flush intervals may lead to latency spikes when the flush does occur as there will be a lot of data to flush.
#    3. Throughput: The flush is generally the most expensive operation, and a small flush interval may lead to excessive seeks.
# The settings below allow one to configure the flush policy to flush data after a period of time or
# every N messages (or both). This can be done globally and overridden on a per-topic basis.

# The number of messages to accept before forcing a flush of data to disk
#log.flush.interval.messages=10000

# The maximum amount of time a message can sit in a log before we force a flush
#log.flush.interval.ms=1000

############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=168

# A size-based retention policy for logs. Segments are pruned from the log unless the remaining
# segments drop below log.retention.bytes. Functions independently of log.retention.hours.
#log.retention.bytes=1073741824

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=107374182

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

############################# Zookeeper #############################

# Zookeeper connection string (see zookeeper docs for details).
# This is a comma separated host:port pairs, each corresponding to a zk
# server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
# You can also append an optional chroot string to the urls to specify the
# root directory for all kafka znodes.
zookeeper.connect=localhost:2181

# Timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=18000


############################# Group Coordinator Settings #############################

# The following configuration specifies the time, in milliseconds, that the GroupCoordinator will delay the initial consumer rebalance.
# The rebalance will be further delayed by the value of group.initial.rebalance.delay.ms as new members join the group, up to a maximum of max.poll.interval.ms.
# The default value for this is 3 seconds.
# We override this to 0 here as it makes for a better out-of-the-box experience for development and testing.
# However, in production environments the default value of 3 seconds is more suitable as this will help to avoid unnecessary, and potentially expensive, rebalances during application startup.
group.initial.rebalance.delay.ms=0

delete.topic.enable = true
```

- server1.properties

```
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#
# This configuration file is intended for use in ZK-based mode, where Apache ZooKeeper is required.
# See kafka.server.KafkaConfig for additional details and defaults
#

############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.
broker.id=1

############################# Socket Server Settings #############################

# The address the socket server listens on. If not configured, the host name will be equal to the value of
# java.net.InetAddress.getCanonicalHostName(), with PLAINTEXT listener name, and port 9092.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092
listeners=PLAINTEXT://pilot.daiso.com:9097

# Listener name, hostname and port the broker will advertise to clients.
# If not set, it uses the value for "listeners".
advertised.listeners=PLAINTEXT://pilot.daiso.com:9097

# Maps listener names to security protocols, the default is for them to be the same. See the config documentation for more details
#listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600


############################# Log Basics #############################

# A comma separated list of directories under which to store log files
log.dirs=/tmp/kafka-logs-1

# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=1

# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown.
# This value is recommended to be increased for installations with data dirs located in RAID array.
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings  #############################
# The replication factor for the group metadata internal topics "__consumer_offsets" and "__transaction_state"
# For anything other than development testing, a value greater than 1 is recommended to ensure availability such as 3.
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

############################# Log Flush Policy #############################

# Messages are immediately written to the filesystem but by default we only fsync() to sync
# the OS cache lazily. The following configurations control the flush of data to disk.
# There are a few important trade-offs here:
#    1. Durability: Unflushed data may be lost if you are not using replication.
#    2. Latency: Very large flush intervals may lead to latency spikes when the flush does occur as there will be a lot of data to flush.
#    3. Throughput: The flush is generally the most expensive operation, and a small flush interval may lead to excessive seeks.
# The settings below allow one to configure the flush policy to flush data after a period of time or
# every N messages (or both). This can be done globally and overridden on a per-topic basis.

# The number of messages to accept before forcing a flush of data to disk
#log.flush.interval.messages=10000

# The maximum amount of time a message can sit in a log before we force a flush
#log.flush.interval.ms=1000

############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=168

# A size-based retention policy for logs. Segments are pruned from the log unless the remaining
# segments drop below log.retention.bytes. Functions independently of log.retention.hours.
#log.retention.bytes=1073741824

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=107374182

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

############################# Zookeeper #############################

# Zookeeper connection string (see zookeeper docs for details).
# This is a comma separated host:port pairs, each corresponding to a zk
# server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
# You can also append an optional chroot string to the urls to specify the
# root directory for all kafka znodes.
zookeeper.connect=localhost:2181

# Timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=18000


############################# Group Coordinator Settings #############################

# The following configuration specifies the time, in milliseconds, that the GroupCoordinator will delay the initial consumer rebalance.
# The rebalance will be further delayed by the value of group.initial.rebalance.delay.ms as new members join the group, up to a maximum of max.poll.interval.ms.
# The default value for this is 3 seconds.
# We override this to 0 here as it makes for a better out-of-the-box experience for development and testing.
# However, in production environments the default value of 3 seconds is more suitable as this will help to avoid unnecessary, and potentially expensive, rebalances during application startup.
group.initial.rebalance.delay.ms=0

delete.topic.enable = true


```

# application.yaml
```yaml
spring:
  kafka:
    bootstrap-servers:
      - 10.222.10.170:9092
    listener:
      ack-mode: MANUAL_IMMEDIATE
      type: SINGLE      
    consumer:
      # consumer bootstrap servers가 따로 존재하면 설정
      # bootstrap-servers: 192.168.0.4:9092
      # 식별 가능한 Consumer Group Id
      group-id: dev-group
      # Kafka 서버에 초기 offset이 없거나, 서버에 현재 offset이 더 이상 존재하지 않을 경우 수행할 작업을 설정
      # latest: 가장 최근에 생산된 메시지로 offeset reset
      # earliest: 가장 오래된 메시지로 offeset reset
      # none: offset 정보가 없으면 Exception 발생
      auto-offset-reset: earliest
      # 데이터를 받아올 때, key/value를 역직렬화
      # JSON 데이터를 받아올 것이라면 JsonDeserializer
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    producer:
      acks: all
      # producer bootstrap servers가 따로 존재하면 설정
      # bootstrap-servers: 3.34.97.97:9092

      # 데이터를 보낼 때, key/value를 직렬화
      # JSON 데이터를 보낼 것이라면 JsonDeserializer
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

```bash
  - zookeeper 실행확인
  $ netstat -anp|grep 2181

  - zookeeper 정지
  $ ./bin/zookeeper-server-stop.sh

  $ systemctl daemon-reload
  $ systemctl enable zookeeper
  $ systemctl start zookeeper  

  - zookeeper 실행
  $ bin/zookeeper-server-start.sh config/zookeeper.properties &

  - kafka #1 실행
  $ bin/kafka-server-start.sh config/server.properties &

  - kafka #2 실행
  $ bin/kafka-server-start.sh config/server-1.properties &

  - kafka #3 실행
  $ bin/kafka-server-start.sh config/server-2.properties &

  - Producer 삭제
  # $ bin/kafka-topics.sh --delete --bootstrap-server pilot.daiso.com:9092  --topic dev-replicated-topic 

  - Producer 실행  
  $ bin/kafka-console-producer.sh --broker-list pilot.daiso.com:9096,pilot.daiso.com:9097,pilot.daiso.com:9098 --topic dev-replicated-topic

  - Consumer 실행
  $ bin/kafka-console-consumer.sh --bootstrap-server pilot.daiso.com:9096 --from-beginning --topic dev-replicated-topic

  $ bin/kafka-console-consumer.sh --bootstrap-server pilot.daiso.com:9097 --from-beginning --topic dev-replicated-topic

  $ bin/kafka-console-consumer.sh --bootstrap-server pilot.daiso.com:9098 --from-beginning --topic dev-replicated-topic
  
  - kafka # 토픽생성
  $ bin/kafka-topics.sh --create --bootstrap-server localhost:9096 --replication-factor 1 --partitions 1 --topic sample-topic-a

  $ bin/kafka-topics.sh --create --bootstrap-server localhost:9096 --replication-factor 1 --partitions 1 --topic sample-topic-b

  $ bin/kafka-topics.sh --create --bootstrap-server localhost:9096 --replication-factor 1 --partitions 1 --topic sample-topic-c    
```

  </pre>
</details>