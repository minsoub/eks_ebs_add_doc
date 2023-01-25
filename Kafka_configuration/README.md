# KAFKA 구성

## Local에서 Kafka 구성하기
- 로컬에서 Kafka를 구성하기 위해서 docker-compose를 통해서 생성한다.
- Kafka의 구성은 Zookeeper와 Kafka 이미지로 이루어진다.
```yaml
version: '2'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_INIT_LIMIT: 5
      ZOOKEEPER_SYNC_LIMIT: 2
    ports:
      - "22181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
```
- ZOOKEEPER_TICK_TIME
  - zookeeper이 클러스터를 구성할 때 동기화를 위한 기본 틱 타임을 지정
  - millisecond로 지정할 수 있으며 2000은 2second
- ZOOKEEPER_INIT_LIMIT
  - 주키퍼를 초기화를 위한 제한 시간을 설정
  - 주키퍼 클러스터는 쿼럼이라는 과정을 통해서 마스터를 선출한다. 이때 주키퍼들이 리더에게 커넥션을 맺을 때 지정할 초기 타임아웃 시간
  - 타임아웃 시간은 ZOOKEEPER_TICK_TIME 단위로 설정
  - ZOOKEEPER_TICK_TIME을 2000, ZOOKEEPER_INIT_LIMIT를 5로 설정했으므로 2000 * 5 = 10000 밀리세컨드. 즉 10초
  - 멀티 브로커에서 유효한 속성이다.
- ZOOKEEPER_SYNC_LIMIT
  - 주키퍼 리더와 나머지 서버들의 싱크 타임.
  - 이 시간내 싱크응답이 들어오는 경우 클러스터가 정상으로 구성되어 있음을 확인하는 시간.
  - 2로 설정했으므로 2000*2=4000으로 4초가 된다.
  - 멀티 브로커에서 유효한 속성.
- KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
  - single 브러커인경우에 지정하여 1로 설정
  - 멀티 브로커는 기본값을 사용하므로 설정이 필요 없다.
- KAFKA_GROUP_INITIAL_REBALANCE_DEPLAY_MS
  - 카프카 그룹이 초기 리밸런싱할 때 컨슈머들이 컨슈머 그룹에 조인할 때 대기 시간

## 로컬 실행
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker-compose -f kafka-configuration.yaml up -d

```

## 로컬 카프카 테스트
### docker 상태 로그 확인
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker ps
CONTAINER ID   IMAGE                              COMMAND                  CREATED         STATUS         PORTS                                         NAMES
8872accabb65   confluentinc/cp-kafka:latest       "/etc/confluent/dock…"   2 minutes ago   Up 2 minutes   9092/tcp, 0.0.0.0:29092->29092/tcp            kafka_configuration_kafka_1
4b1af0b11eb8   confluentinc/cp-zookeeper:latest   "/etc/confluent/dock…"   2 minutes ago   Up 2 minutes   2888/tcp, 3888/tcp, 0.0.0.0:22181->2181/tcp   kafka_configuration_zookeeper_1

```
### topic 생성
```shell
 ✘ jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker ps
CONTAINER ID   IMAGE                              COMMAND                  CREATED       STATUS       PORTS                                         NAMES
8872accabb65   confluentinc/cp-kafka:latest       "/etc/confluent/dock…"   6 hours ago   Up 6 hours   9092/tcp, 0.0.0.0:29092->29092/tcp            kafka_configuration_kafka_1
4b1af0b11eb8   confluentinc/cp-zookeeper:latest   "/etc/confluent/dock…"   6 hours ago   Up 6 hours   2888/tcp, 3888/tcp, 0.0.0.0:22181->2181/tcp   kafka_configuration_zookeeper_1

jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker exec 8872accabb65 kafka-topics --create --topic my-test-topic --bootstrap-server kafka:9092 --replication-factor 1 --partitions 1
Created topic my-test-topic.
```
- kafka-topics
  - 카프카 토픽에 대한 명령을 실행한다.
- create
  - 토픽을 생성
- --topic
  - 생성할 토픽 이름 지정
- --bootstrap-server service:port
  - bootstrap-server는 kafka 브로커 서비스를 나타낸다. 서비스:포트로 지정하여 접근할 수 있다.
- --replication-factor 1
  - 복제 개수를 지정한다.
- --partitions
  - 토픽내에 파티션 개수를 지정한다.
### topic 확인
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker exec 8872accabb65 kafka-topics --describe --topic my-test-topic --bootstrap-server kafka:9092
Topic: my-test-topic    TopicId: G_fjVTHGRZ2o5XiQ70gsnw PartitionCount: 1       ReplicationFactor: 1    Configs: 
        Topic: my-test-topic    Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```
### 컨슈머 실행
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker exec -it 8872accabb65 bash
[appuser@8872accabb65 ~]$ kafka-console-consumer --topic my-test-topic --bootstrap-server kafka:9092

```
### 프로듀서 실행
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc   master ±✚  docker exec -it 8872accabb65 bash
[appuser@8872accabb65 ~]$ kafka-console-producer --topic my-test-topic --broker-list kafka:9092
>Test
>Test is producer
>Go~~~
>Test
>Go
>Go
```
### 정리
```shell
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  docker-compose -f kafka-configuration.yaml down
Stopping kafka_configuration_kafka_1     ... done
Stopping kafka_configuration_zookeeper_1 ... done
Removing kafka_configuration_kafka_1     ... done
Removing kafka_configuration_zookeeper_1 ... done
Removing network kafka_configuration_default
 jms@jeongminseob-Mac-Pro  ~/Documents/eks_ebs_add_doc/Kafka_configuration   master ±✚  
```
