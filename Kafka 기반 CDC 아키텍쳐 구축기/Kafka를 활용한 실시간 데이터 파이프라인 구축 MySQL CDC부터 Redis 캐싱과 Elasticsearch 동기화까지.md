## **1. 개요**
이 글에서는 **Kafka, MySQL, Docker, Debezium, Elasticsearch, Kibana**를 활용해 **CDC(데이터 변경 사항을 실시간으로 감지하고 반영하는)를 구현**하고, 변경된 데이터를 **Elasticsearch에 저장하여 검색 및 시각화**하는 방법을 다룹니다.

## **2. Elasticsearch & Kibana 소개**
- **Elasticsearch**: 대용량 데이터 검색 및 분석을 위한 분산 검색 엔진
- **Kibana**: Elasticsearch 데이터를 시각화하는 도구
- **Kafka**: 실시간 데이터 스트리밍을 위한 메시지 브로커
- **Debezium**: 데이터베이스 변경 사항을 감지하는 CDC 프레임워크

## **3. 프로젝트 설정**
> [!NOTE]
> - Docker : Kafka, Zookeeper, Kafka Connect, MySQL, Debezium, Elasticsearch, Kibana 실행
> - Kafka & Zookeeper: 메시지 브로커 역할
> - Debezium: MySQL의 변경 사항을 Kafka로 전송
> - Java 기반 Kafka Consumer: CDC 이벤트 처리 후 Elasticsearch 저장
> - Kibana: Elasticsearch 데이터 시각화

### **1) build.gradle** 
```groovy
// Kafka
implementation 'org.springframework.kafka:spring-kafka'
// Elasticsearch
implementation 'org.springframework.boot:spring-boot-starter-data-elasticsearch'
//Redis  
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```
### **2) application.yml**
```yaml
kafka:
  bootstrap-servers: localhost:29092
  consumer:
    group-id: post-consumer-group
    auto-offset-reset: earliest
    key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```
### **3) 커스텀 도커 이미지를 사용하기 위한 설정**
```yml
#DockerFile. 프로젝트 루트경로에 위치시켰음
FROM confluentinc/cp-kafka-connect:latest  
  
# Confluent Hub CLI를 사용하여 Elasticsearch 및 Debezium 플러그인 설치  
RUN confluent-hub install --no-prompt confluentinc/kafka-connect-elasticsearch:latest \  
    && confluent-hub install --no-prompt debezium/debezium-connector-mysql:latest  
  
# Kafka Connect 실행  
CMD ["bash", "-c", "echo 'Starting Kafka Connect'; /etc/confluent/docker/run"]
```
##### **📌 Dockerfile을 사용하여 custom-kafka-connect라는 이미지를 생성**
```bash
docker build -t custom-kafka-connect .
```

>[!NOTE]
> confluentinc/cp-kafka-connect 이미지는 elastic sink connector 플러그인을 포함하지 않아 커스텀 이미지를 만들었습니다.
> 제가 찾아본 결과 debezium connector와 elastic search sink connector 모두 포함하는 이미지를 찾을 수 없어 커스텀이미지를 사용했는데 혹시 다른방법을 찾게되면 업데이트 하겠습니다.

### **4) Docker로 Kafka, Zookeeper, MySQL, Elasticsearch, Kibana 설정**
```yaml
# Docker Compose 파일 작성 (docker-compose.yml)
services:  
  zookeeper:  
    image: quay.io/debezium/zookeeper:3.0  
    container_name: zookeeper  
    restart: always  
    ports:  
      - "2181:2181"  
    environment:  
      - ZOOKEEPER_CLIENT_PORT=2181  
  
  kafka:  
    image: confluentinc/cp-kafka:latest  
    container_name: kafka  
    restart: always  
    environment:  
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181  
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,PLAINTEXT_HOST://0.0.0.0:29092  
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092  
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT  
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT  
      KAFKA_BROKER_ID: 1  
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1  
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"  #  자동 토픽 생성 활성화  
    ports:  
      - "9092:9092"  
      - "29092:29092"  
    depends_on:  
      - zookeeper  
  
  mysql:  
    image: mysql:8.0  
    container_name: mysql  
    restart: always  
    ports:  
      - "3306:3306"  
    environment:  
      MYSQL_ROOT_PASSWORD: root  
      MYSQL_USER: user  
      MYSQL_PASSWORD: password  
      MYSQL_DATABASE: mydb  
    volumes:  
      - mysql_data:/var/lib/mysql  
  
  redis:  
    image: "redis:latest"  
    container_name: redis  
    ports:  
      - "6379:6379"  
    restart: always  
  
  kafka-connect:  
    image: custom-kafka-connect  
    container_name: kafka-connect  
    restart: always  
    depends_on:  
      - kafka  
      - elasticsearch  
    ports:  
      - "8083:8083"  
    environment:  
      - CONNECT_BOOTSTRAP_SERVERS=kafka:9092  
      - CONNECT_GROUP_ID=connect-cluster  
      - CONNECT_CONFIG_STORAGE_TOPIC=connect-configs  
      - CONNECT_OFFSET_STORAGE_TOPIC=connect-offsets  
      - CONNECT_STATUS_STORAGE_TOPIC=connect-statuses  
      - CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1  
      - CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1  
      - CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1  
      - CONNECT_REST_ADVERTISED_HOST_NAME=kafka-connect  
      - CONNECT_PLUGIN_PATH=/usr/share/confluent-hub-components  
      - CONNECT_KEY_CONVERTER=org.apache.kafka.connect.json.JsonConverter  CONNECT_VALUE_CONVERTER=org.apache.kafka.connect.json.JsonConverter  
      - CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE=false  #JSON Converter일 때 필요  
      - CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE=false  #JSON Converter일 때 필요  
  
  elasticsearch:  
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.3  
    container_name: elasticsearch  
    restart: always  
    environment:  
      - discovery.type=single-node  
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"  
      - xpack.security.enabled=false  
    ports:  
      - "9200:9200"  
  
  kibana:  
    image: docker.elastic.co/kibana/kibana:8.5.3  
    container_name: kibana  
    depends_on:  
      - elasticsearch  
    ports:  
      - "5601:5601"  
  
volumes:  
  mysql_data:
```
##### **📌 Docker 명령어 참고**
```bash
# 컨테이너 실행
docker-compose up -d

# 컨테이너 중지
docker-compose down

# 특정 컨테이너만 재시작
docker-compose restart <container-name>

# 도커 컨테이너 상태확인
docker ps 
```
### **5) MySQL CDC 설정**
```bash
 # mysql 컨테이너에 접속
 docker exec -it mysql bash  
```
```sql
-- 사용자 생성 및 권한 부여
CREATE USER 'debezium'@'%' IDENTIFIED BY 'dbz';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;
```

>[!NOTE]
>Debezium MySQL 커넥터 설정 전에 커넥터가 변경사항을 캡쳐하는 데이터베이스의권한을 부여해야합니다.
### **6) Debezium MySQL 커넥터 설정**
##### 📌 커넥터 등록
- **Method**: `POST`
- **URL**: `http://localhost:8083/connectors/`
- **Content-Type**: `application/json`
##### 📝 요청 본문 (Request Body)
```json
{
  "name": "mydb-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "topic.prefix": "mingle",
    "database.include.list": "mydb",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schemahistory.mydb",
    "include.schema.changes": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "true"
  }
}
```
##### 📝 Response Body(정상상태)
```json
{
  "name": "mydb-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "topic.prefix": "mingle",
    "database.include.list": "mydb",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092",
    "schema.history.internal.kafka.topic": "schemahistory.mydb",
    "include.schema.changes": "true",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "true"
  },
  "tasks": [],
  "type": "source"
}
```
##### 📌  커넥터 등록 후 확인
- **Method**: `GET`
- **URL** : `http://localhost:8083/connectors`
##### 📝 Response Body(정상상태)
``` json
[
	"mydb-connector"
]
```
##### 📌  커넥터 상태 확인
- **Method**: `GET`
- **URL**: `http://localhost:8083/connectors/<커넥터명>/status`
##### 📝 Response Body(정상 상태)
```json
{
  "name": "mydb-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect:8083"
    }
  ],
  "type": "source"
}
```
##### 📌  커넥터 삭제
- Method: `DELETE`
- URL: `http://localhost:8083/connectors/<커넥터명>
### **7) Elasticsearch Sink 커넥터 설정**
##### 📌 커넥터 등록
- **Method**: `POST`
- **URL**: `http://localhost:8083/connectors/`
- **Content-Type**: `application/json`
##### 📝 요청 본문 (Request Body)
```json
{
  "name": "elasticsearch-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "topics.regex": "mysql\\.mydb\\..*",
    "connection.url": "http://elasticsearch:9200",
    "type.name": "_doc",
    "key.ignore": "true",a
    "schema.ignore": "true"
  }
}
```
##### 📝 Response Body(정상상태)
```json
{
    "name": "elasticsearch-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "topics.regex": "mysql\\.mydb\\..*",
        "connection.url": "http://elasticsearch:9200",
        "type.name": "_doc",
        "key.ignore": "true",
        "schema.ignore": "true",
        "name": "elasticsearch-sink-connector"
    },
    "tasks": [],
    "type": "sink"
}
```
##### 📌  커넥터 등록 후 확인
- **Method**: `GET`
- **URL** : `http://localhost:8083/connectors`
##### 📝 Response Body(정상상태)
``` json
[
	"mydb-connector",
	"elasticsearch-sink-connector"
]
```
##### 📌  커넥터 상태 확인
- **Method**: `GET`
- **URL**: `http://localhost:8083/connectors/<커넥터명>/status`
##### 📝 Response Body(정상 상태)
```json
{
  "name": "mydb-connector",
  "connector": {
    "state": "RUNNING",
    "worker_id": "kafka-connect:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "kafka-connect:8083"
    }
  ],
  "type": "source"
}
```
##### 📌  설치된 플러그인 확인
- Method: `GET`
- URL: `http://localhost:8083/connector-plugins`
##### 📝 Response Body(정상 상태)
```json
[
  {
    "class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "type": "sink",
    "version": "14.1.2"
  },
  {
    "class": "io.debezium.connector.mysql.MySqlConnector",
    "type": "source",
    "version": "2.4.2.Final"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "7.9.0-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "7.9.0-ccs"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "7.9.0-ccs"
  }
]
```
>[!NOTE]
> **ElasticsearchSinkConnector** 와 **debezium.connector** 가 존재해야합니다!

---
## **4. KafkaConsumer.java (Redis Cache 저장)**
```java
@Slf4j  
@Service  
@RequiredArgsConstructor  
public class KafkaConsumer {  
    private final RedisService redisService;  
    private final PostService postService;  
    private final ObjectMapper objectMapper;  
  
    @KafkaListener(topics = "mingle.mydb.post", groupId = "post-consumer-group")  
    public void listen(ConsumerRecord<String, String> record) {  
        try {  
            JsonNode jsonNode = objectMapper.readTree(record.value());  
            JsonNode payload = jsonNode.get("payload");  
  
            if (payload == null || !payload.hasNonNull("op")) {  
                log.warn("Kafka 메시지에 유효한 payload가 없음: {}", record.value());  
                return;  
            }  
  
            String operation = payload.get("op").asText();  
            JsonNode afterNode = payload.get("after");  
            JsonNode beforeNode = payload.get("before");  
  
            switch (operation) {  
                case "c":  
                    handleInsertEvent(afterNode);  
                    break;  
                case "u":  
                    handleUpdateEvent(beforeNode, afterNode);  
                    break;  
                case "d":  
                    handleDeleteEvent(beforeNode);  
                    break;  
                default:  
                    log.warn("지원하지 않는 Kafka 이벤트 유형: {}", operation);  
            }  
  
        } catch (Exception e) {  
            log.error("Kafka 메시지 처리 중 오류 발생: {}", e.getMessage(), e);  
        }  
    }  
  
    /**  
     * 새 게시글이 생성될 때 처리  
     */  
    private void handleInsertEvent(JsonNode afterNode) {  
        if (afterNode == null || !afterNode.hasNonNull("post_id") || !afterNode.hasNonNull("author_id")) {  
            log.warn("INSERT 이벤트에서 post_id 또는 author_id가 누락됨: {}", afterNode);  
            return;  
        }  
  
        long postId = afterNode.get("post_id").asLong();  
        long authorId = afterNode.get("author_id").asLong();  
        updateRedisCache(authorId, postId);  
        log.info(" [INSERT] 게시글 추가됨 - post_id: {}, author_id: {}", postId, authorId);  
    }  
  
    /**  
     * 기존 게시글이 수정될 때 처리 (content가 변경된 경우만)  
     */    private void handleUpdateEvent(JsonNode beforeNode, JsonNode afterNode) {  
        if (beforeNode == null || afterNode == null ||  
                !beforeNode.hasNonNull("content") || !afterNode.hasNonNull("content")) {  
            log.warn("UPDATE 이벤트에서 content 필드가 누락됨: before={}, after={}", beforeNode, afterNode);  
            return;  
        }  
  
        boolean isContentChanged = !beforeNode.get("content").asText().equals(afterNode.get("content").asText());  
        if (!isContentChanged) {  
            return;  
        }  
  
        long postId = afterNode.get("post_id").asLong();  
        long authorId = afterNode.get("author_id").asLong();  
        updateRedisCache(authorId, postId);  
        log.info("🔄 [UPDATE] 게시글 내용 변경 - post_id: {}, author_id: {}", postId, authorId);  
    }  
  
    /**  
     * 게시글이 삭제될 때 처리  
     */  
    private void handleDeleteEvent(JsonNode beforeNode) {  
        if (beforeNode == null || !beforeNode.hasNonNull("post_id") || !beforeNode.hasNonNull("author_id")) {  
            log.warn("DELETE 이벤트에서 post_id 또는 author_id가 누락됨: {}", beforeNode);  
            return;  
        }  
  
        long deletedId = beforeNode.get("post_id").asLong();  
        long authorId = beforeNode.get("author_id").asLong();  
        deleteRedisCache(authorId, deletedId);  
        log.info(" [DELETE] 게시글 삭제됨 - post_id: {}, author_id: {}", deletedId, authorId);  
    }  
  
    /**  
     * Redis 캐시에 게시글 추가  
     */  
    private void updateRedisCache(Long authorId, Long postId) {  
        Set<String> followers = redisService.getFollowers(authorId);  
        if (followers.isEmpty()) {  
            log.info(" [Redis] authorId {} 에 대한 팔로워 없음, 업데이트 생략", authorId);  
            return;  
        }  
        for (String follower : followers) {  
            redisService.updateCache(Long.valueOf(follower), postId);  
        }  
    }  
  
    /**  
     * Redis 캐시에서 게시글 삭제  
     */  
    private void deleteRedisCache(Long authorId, Long postId) {  
        Set<String> followers = redisService.getFollowers(authorId);  
        if (followers.isEmpty()) {  
            log.info(" [Redis] authorId {} 에 대한 팔로워 없음, 삭제 생략", authorId);  
            return;  
        }  
        for (String follower : followers) {  
            redisService.deleteCache(Long.valueOf(follower), postId);  
        }  
    }  
}
```

## **5. Kibana에서 데이터 확인**

##### **1) Elasticsearch 인덱스 확인**
```bash
curl -X GET "http://localhost:9200/_cat/indices?v"
```

##### **2) Kibana 접속 후 확인**
1. `http://localhost:5601`로 접속
2. `Stack Management > Index Patterns`에서 `mingle.mydb.post` 인덱스 생성
3. `Discover`에서 데이터 조회 가능

---
## **6. 결론**
##### **✅ MySQL CDC 데이터 흐름**
1. **MySQL Binlog** → MySQL에서 변경 사항이 발생하면 **Binlog**에 기록됨
2. **Debezium CDC 감지** → Debezium이 Binlog를 모니터링하여 변경 사항을 감지
3. **Kafka 브로커로 전송** → 감지된 변경 데이터를 Kafka 토픽에 게시
4. **Kafka Consumer** → Kafka Consumer가 메시지를 읽어 **Redis에 캐싱**
5. **Elasticsearch Connector** → afka Connect의 **Elasticsearch Sink**가 비동기적으로 메시지를 소비하여 **Elasticsearch에 저장**

