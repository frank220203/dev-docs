# [Docker compose] Kafka 설치

### Docker Compose 파일 만들기
```bash
# kafka 폴더 생성
mkdir ~/kafka
cd ~/kafka
```
```bash
# docker-compose.yaml 파일 생성성
vi docker-compose.yaml
```
```yaml
# docker-compose.yaml
services:
  # kafka의 메타데이터 및 클러스터 관리, 최소 3개 권장 (홀수 권장)
  zookeeper-1:
    image: wurstmeister/zookeeper
    container_name: zookeeper-1
    ports:
      - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: "server.1=zookeeper-1:2888:3888 server.2=zookeeper-2:2888:3888 server.3=zookeeper-3:2888:3888"

  zookeeper-2:
    image: wurstmeister/zookeeper
    container_name: zookeeper-2
    ports:
      - "2182:2181"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: "server.1=zookeeper-1:2888:3888 server.2=zookeeper-2:2888:3888 server.3=zookeeper-3:2888:3888"

  zookeeper-3:
    image: wurstmeister/zookeeper
    container_name: zookeeper-3
    ports:
      - "2183:2181"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: "server.1=zookeeper-1:2888:3888 server.2=zookeeper-2:2888:3888 server.3=zookeeper-3:2888:3888"

  # kafka 메시지 분배, 컨트롤러 - 브로커 1:1 대응
  kafka-controller-1:
    image: apache/kafka:latest
    container_name: kafka-controller-1
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: controller
      KAFKA_LISTENERS: CONTROLLER://:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      PATH: "/opt/kafka/bin:$PATH"
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3

  kafka-controller-2:
    image: apache/kafka:latest
    container_name: kafka-controller-2
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: controller
      KAFKA_LISTENERS: CONTROLLER://:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      PATH: "/opt/kafka/bin:$PATH"
    depends_on:
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3

  # kafka 메시지 저장, 최소 3개 권장 (유사시 컨트롤러 역할을 하기에 홀수 권장)
  kafka-broker-1:
    image: apache/kafka:latest
    container_name: kafka-broker-1
    ports:
      - 29092:9092
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: broker
      KAFKA_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka-broker-1:19092,PLAINTEXT_HOST://localhost:29092'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      PATH: "/opt/kafka/bin:$PATH"
    depends_on:
      - kafka-controller-1
      - kafka-controller-2
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3

  kafka-broker-2:
    image: apache/kafka:latest
    container_name: kafka-broker-2
    ports:
      - 39092:9092
    environment:
      KAFKA_NODE_ID: 4
      KAFKA_PROCESS_ROLES: broker
      KAFKA_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka-broker-2:19092,PLAINTEXT_HOST://localhost:39092'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      PATH: "/opt/kafka/bin:$PATH"
    depends_on:
      - kafka-controller-1
      - kafka-controller-2
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3

  kafka-broker-3:
    image: apache/kafka:latest
    container_name: kafka-broker-3
    ports:
      - 49092:9092
    environment:
      KAFKA_NODE_ID: 5
      KAFKA_PROCESS_ROLES: broker
      KAFKA_LISTENERS: 'PLAINTEXT://:19092,PLAINTEXT_HOST://:9092'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka-broker-3:19092,PLAINTEXT_HOST://localhost:49092'
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-controller-1:9093,2@kafka-controller-2:9093
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      PATH: "/opt/kafka/bin:$PATH"
    depends_on:
      - kafka-controller-1
      - kafka-controller-2
      - zookeeper-1
      - zookeeper-2
      - zookeeper-3
```

### 도커 컴포즈로 Kafka 실행
```bash
# -d은 백그라운드 실행
sudo docker compose up -d
```

### 토픽 생성
```bash
# 브로커 컨테이너 접속
sudo docker exec -it kafka-broker-1 bash
# 토픽 생성
kafka-topics.sh --create --bootstrap-server kafka-broker-1:19092 --replication-factor 3 --partitions 6 --topic topic-name
```

### 토픽 확인
```bash
# 도커 컨테이너 내에서 브로커에 접근하는 경우 kafka-broker-1:19092를 사용하고, 도커 외부에서 브로커에 접근하는 경우 localhost:29092를 쓴다.
kafka-topics.sh --list --bootstrap-server kafka-broker-1:19092
```

### 메시지 발행
```bash
kafka-console-producer.sh --bootstrap-server kafka-broker-1:19092 --topic topic-name
>> Hello, Kafka~
>> (탈출) Ctrl + c
```

### 메시지 확인
```bash
kafka-console-consumer.sh --bootstrap-server kafka-broker-1:19092 --topic topic-name --from-beginning
```