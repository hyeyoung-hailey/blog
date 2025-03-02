
## **1.  개요**
 이 글에서는 **Kafka, MySQL, Docker, Debezium**을 활용해 CDC(데이터 변경 사항을 실시간으로 감지하고 반영하는)를 구현하고, **Java 기반 Kafka Consumer**를 통해 변경된 데이터를 **Redis에 저장하여 Cache** 처리하는 방법을 다룹니다.

## **2.  CDC란?**
- 데이터베이스 변경 사항을 이벤트로 추출하는 기술
- 실시간 데이터 동기화, 데이터 분석 등에 활용

## **3.  Debezium과 Kafka 소개**
- **Debezium**: 데이터베이스 변경 사항을 감지하는 오픈소스 프레임워크
- **Kafka**: 데이터를 실시간으로 스트리밍하는 메시지 브로커

## **4.  프로젝트 설정

>[!NOTE]
>- Docker : Kafka, Zookeeper, Kafka Connect, MySQL, Debezium 실행
>- Kafka & Zookeeper: 메시지 브로커 역할
>- Debezium: MySQL의 변경 사항을 Kafka로 전송
>- Java 기반 Kafka Consumer: CDC 이벤트 처리
### **1) biuld.gradle**
```groovy
//kafka  
implementation 'org.springframework.kafka:spring-kafka'
//Redis
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```
### **2) application.yml Kafka, Redis 설정 추가**
```yaml
kafka:  
  bootstrap-servers: localhost:29092  
  consumer:  
    group-id: post-consumer-group  
    auto-offset-reset: earliest  
    key-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
    value-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
  producer:  
    key-serializer: org.apache.kafka.common.serialization.StringSerializer  
    value-serializer: org.apache.kafka.common.serialization.StringSerializer

data:  
  redis:  
    host: redis  
    port: 6379
```
### **3) Docker로 Kafka, Zookeeper, MySQL 설정**
```bash
# Docker Compose 파일 작성 (docker-compose.yml)
services:  
  mysql:  
    image: mysql:8  
    container_name: mysql  
    restart: always  
    environment:  
      MYSQL_ROOT_PASSWORD: root  
      MYSQL_DATABASE: mydb  
      MYSQL_USER: user  
      MYSQL_PASSWORD: password  
    ports:  
      - "3306:3306"  
  redis:  
    image: "redis:latest"  
    container_name: redis  
    ports:  
      - "6379:6379"  
    restart: always  
  
  zookeeper:  
    image: quay.io/debezium/zookeeper:3.0  
    container_name: zookeeper  
    restart: always  
    ports:  
      - "2181:2181"  
  
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
    ports:  
      - "9092:9092"  
      - "29092:29092"  
    depends_on:  
      - zookeeper  
  
  debezium:  
    image: quay.io/debezium/connect:latest  
    container_name: debezium  
    restart: always  
    ports:  
      - "8083:8083"  
    environment:  
      BOOTSTRAP_SERVERS: kafka:9092  
      GROUP_ID: post-group  
      CONFIG_STORAGE_TOPIC: debezium_configs  
      OFFSET_STORAGE_TOPIC: debezium_offsets  
      STATUS_STORAGE_TOPIC: debezium_status  
    #      database.include.list: mydb  
    #      table.include.list: mydb.post    
    depends_on:  
      - kafka  
      - mysql
```
##### 📌 docker 명령어 참고
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
### **4) MySQL CDC 설정**
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

### **5) Debezium MySQL 커넥터 설정**
##### 📌 커넥터 등록
- **Method**: `POST`
- **URL**: `http://localhost:8083/connectors/`
- **Content-Type**: `application/json`
##### 📝 요청 본문 (Request Body)
```json
{
  "name": "mydb-connector", //커넥터명
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium", //권한 부여받은 계정으로 등록
    "database.password": "dbz",
    "database.server.id": "184054", // 고유한 정수 값이어야 하며, 다른 MySQL 서버                                          또는 Debezium 인스턴스와 중복되면 안됨
    "topic.prefix": "mingle", //prefix 가 없으면 등록이안되었음
    "database.include.list": "mydb", //cdc 대상은 mydb 전체
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9092", 
    "schema.history.internal.kafka.topic": "schemahistory.mydb"
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
    "name": "mydb-connector"
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
- URL: `http://localhost:8083/connectors/<커넥터명>`
### **6) KafkaConsumer.java**
```java
@Slf4j  
@Service  
@RequiredArgsConstructor  
public class PostKafkaConsumer {  
  
    private final ObjectMapper objectMapper;  
    private final RedisService redisService;  
  
    private static final long FOLLOWER_ID = 100L; // 테스트용  
  
    @KafkaListener(topics = "mingle.mydb.post", groupId = "post-consumer-group")  
    public void listen(ConsumerRecord<String, String> record) {  
        try {  
            JsonNode payload = extractPayload(record.value());  
            if (payload == null) return;  
  
            String operation = payload.get("op").asText();  
            switch (operation) {  
                case "c" -> handleInsert(payload);  
                case "u" -> handleUpdate(payload);  
                case "d" -> handleDelete(payload);  
                default -> log.warn("알 수 없는 이벤트 타입: {}", operation);  
            }  
  
        } catch (Exception e) {  
            log.error("Kafka 메시지 처리 중 오류 발생: {}", e.getMessage(), e);  
        }  
    }  
  
    private JsonNode extractPayload(String message) throws Exception {  
        JsonNode jsonNode = objectMapper.readTree(message);  
        return jsonNode.get("payload");  
    }  
  
    private void handleInsert(JsonNode payload) {  
        JsonNode afterNode = payload.get("after");  
        if (afterNode == null) return;  
  
        long id = afterNode.get("id").asLong();  
        long authorId = afterNode.get("author_id").asLong();  
        String content = afterNode.get("content").asText();  
        long createdAt = afterNode.get("created_at").asLong();  
  
        log.info("새로운 게시글 추가됨: ID={}, 작성자 ID={}, 내용={}, 생성시간={}", id, authorId, content, createdAt);  
        redisService.updateCache(FOLLOWER_ID, id);  
    }  
  
    private void handleUpdate(JsonNode payload) {  
        JsonNode afterNode = payload.get("after");  
        if (afterNode == null) return;  
  
        long id = afterNode.get("id").asLong();  
        long authorId = afterNode.get("author_id").asLong();  
  
        log.info("게시글 업데이트: ID={}, 작성자 ID={}, 변경된 데이터={}", id, authorId, afterNode);  
        redisService.updateCache(FOLLOWER_ID, id);  
    }  
  
    private void handleDelete(JsonNode payload) {  
        JsonNode beforeNode = payload.get("before");  
        if (beforeNode == null) return;  
  
        long deletedId = beforeNode.get("id").asLong();  
        long authorId = beforeNode.get("author_id").asLong();  
  
        log.info("게시글 삭제됨: ID={}, 작성자 ID={}", deletedId, authorId);  
        redisService.deleteCache(FOLLOWER_ID, deletedId);  
    }  
}
```
### **6) Redis Config**
```java
@Configuration  
public class RedisConfig {  
    @Bean  
    public RedisConnectionFactory redisConnectionFactory() {  
        return new LettuceConnectionFactory();  
    }  
  
    @Bean  
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory connectionFactory) {  
        RedisTemplate<String, String> template = new RedisTemplate<>();  
        template.setConnectionFactory(connectionFactory);  
        template.setKeySerializer(new StringRedisSerializer());  
        template.setValueSerializer(new StringRedisSerializer());  
        return template;  
    }  
}
```
### **7) RedisService.java**
```java
@Service  
@RequiredArgsConstructor  
public class RedisService {  
  
    private final RedisTemplate<String, String> redisTemplate;  
  
    public void updateCache(Long followerId, Long postId) {  
        String key = "newsfeed:user:" + followerId;  
        // 최신 게시글 추가  
        redisTemplate.opsForList().leftPush(key, postId.toString());  
        // 뉴스피드 최대 개수 제한  
        redisTemplate.opsForList().trim(key, 0, 19);  
    }  
  
    public void deleteCache(Long followerId, Long postId) {  
        String key = "newsfeed:user:" + followerId;  
        // 특정 postId만 제거  
        redisTemplate.opsForList().remove(key, 1, postId.toString());  
    }  
}
```
## **5. 결과 확인**

##### **1) kafka 생성된 topics 확인**
```bash
docker exec -it kafka kafka-topics --bootstrap-server localhost:9092 --list
```
**✅ 토픽 생성 정상일 때 결과**
```bash
__consumer_offsets
debezium_configs
debezium_offsets
debezium_status
mingle
mingle.mydb.post
mingle.mydb.users
schemahistory.mydb
```
##### **2) MySQL에서 데이터 변경 테스트**
```sql
insert into post (created_at, content, author_id) values (now(), 'content test','author');

update post set content='update test' where id = 12;

delete from post where id = 6;
```
##### **3) Kafka Consumer를 통해 이벤트 확인**
```bash
 docker exec -it kafka bash -c 'kafka-console-consumer --bootstrap-server kafka:29092 --topic mingle.mydb.post --from-beginning'
```
>[!TIP]
>-- topic mingle.mydb.post
>• 소비할 **Kafka 토픽 이름** (mingle.mydb.post)

✅ **정상적인 경우 출력 예시**
```json
{
  "payload": {
    "op": "c",
    "after": {
      "id": 15,
      "content": "content test",
      "created_at": "2025-03-02T12:34:56Z"
    }
  }
}
```
```json
{
  "payload": {
    "op": "u",
    "after": {
      "id": 12,
      "content": "update test",
      "created_at": "2025-03-02T12:30:00Z"
    }
  }
}
```
##### **4) Java Kafka Consumer 로그 확인**
```bash
새로운 게시글 추가됨: ID=15, 내용=content test, 생성시간=2025-03-02T12:34:56Z
게시글 업데이트: {"id": 12, "content": "update test", "created_at": "2025-03-02T12:30:00Z"}
```
##### **5) Redis 저장 확인**
```bash
# redis cli 접속
docker exec -it redis redis-cli
# 키 확인
keys *
# 값 확인
rlange newsfeed:user:100
```

## **6. 결론**
 Kafka, Debezium, MySQL, Redis를 활용하여 **CDC 기반의 실시간 데이터 동기화 시스템**을 구축하는 방법을 다루었습니다.

✅ **CDC 구현이 성공적으로 이루어졌다면 다음과 같은 동작을 확인할 수 있습니다.**
1. **MySQL 데이터 변경 → Kafka 토픽으로 이벤트 생성**
2. **Kafka Consumer가 변경 사항을 감지하여 로그 출력**
3. **Kafka 메시지가 Redis에 캐시로 저장됨**

이제 CDC를 활용하여 실시간 데이터 반영이 필요한 다양한 서비스(예: 피드, 알림 시스템)에 적용할 수 있습니다.