# Spring Event 레퍼런스

## 이벤트 네이밍 규칙

### 이벤트 클래스
| 타입 | 규칙 | 예시 |
|------|------|------|
| 생성 | {Domain}CreatedEvent | OrderCreatedEvent |
| 완료 | {Domain}{Action}CompletedEvent | PaymentCompletedEvent |
| 실패 | {Domain}{Action}FailedEvent | PaymentFailedEvent |
| 취소 | {Domain}CancelledEvent | OrderCancelledEvent |
| 변경 | {Domain}{Field}ChangedEvent | OrderStatusChangedEvent |

### 토픽/큐 네이밍
| 브로커 | 규칙 | 예시 |
|--------|------|------|
| Kafka Topic | {도메인}-events | order-events |
| Kafka DLQ Topic | {도메인}-events.dlq | order-events.dlq |
| RabbitMQ Exchange | {도메인}.exchange | order.exchange |
| RabbitMQ Queue | {도메인}.{이벤트}.queue | order.created.queue |
| RabbitMQ Routing Key | {도메인}.{이벤트} | order.created |
| RabbitMQ DLQ | {도메인}.{이벤트}.dlq | order.created.dlq |

### Consumer Group
- Kafka: `{서비스명}-service` (예: `payment-service`)
- 같은 서비스의 여러 인스턴스가 같은 group으로 묶임

## AsyncConfig 설정

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("async-event-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("비동기 처리 예외: method={}, error={}", method.getName(), ex.getMessage(), ex);
        };
    }
}
```

## Kafka docker-compose

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
```

## RabbitMQ docker-compose

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"   # 관리 콘솔
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
```

## 이벤트 직렬화/역직렬화

### Jackson 설정

```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
    }
}
```

### Kafka JsonSerializer 커스텀 설정

```java
@Bean
public ProducerFactory<String, DomainEvent> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    config.put(JsonSerializer.ADD_TYPE_INFO_HEADERS, true);  // 타입 정보 헤더 추가
    return new DefaultKafkaProducerFactory<>(config);
}
```

## Saga 상태 관리 테이블 (Orchestration 방식)

```sql
CREATE TABLE saga_instances (
    id              BIGINT NOT NULL AUTO_INCREMENT,
    saga_id         VARCHAR(50) NOT NULL,
    saga_type       VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(50) NOT NULL,
    current_step    VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL,   -- STARTED, COMPENSATING, COMPLETED, FAILED
    payload         TEXT,
    created_at      DATETIME(6) NOT NULL,
    updated_at      DATETIME(6),
    PRIMARY KEY (id)
) ENGINE=InnoDB;

CREATE UNIQUE INDEX idx_saga_saga_id ON saga_instances (saga_id);
CREATE INDEX idx_saga_status ON saga_instances (status);
```

## 모니터링 포인트

### 주요 메트릭
- 이벤트 발행 성공/실패 수
- 이벤트 처리 지연 시간 (발행 시각 → 처리 시각)
- Consumer lag (미처리 이벤트 수)
- DLQ 메시지 수
- Outbox pending 이벤트 수
- 재시도 횟수

### 로깅 패턴
```java
// 발행
log.info("[EVENT_PUBLISH] eventId={}, type={}, aggregateId={}", 
         event.getEventId(), event.getEventType(), event.getAggregateId());

// 수신
log.info("[EVENT_CONSUME] eventId={}, type={}, aggregateId={}", 
         event.getEventId(), event.getEventType(), event.getAggregateId());

// 실패
log.error("[EVENT_FAIL] eventId={}, type={}, error={}", 
          event.getEventId(), event.getEventType(), ex.getMessage(), ex);

// 보상
log.warn("[SAGA_COMPENSATE] sagaId={}, step={}, reason={}", 
         sagaId, currentStep, reason);
```

## 이벤트 버전 관리

이벤트 스키마가 변경될 때 하위 호환성을 유지한다:

```java
// v1
public class OrderCreatedEvent extends DomainEvent {
    private String orderToken;
    private Long userId;
    private Long totalAmount;
}

// v2 - 필드 추가 (하위 호환)
public class OrderCreatedEvent extends DomainEvent {
    private String orderToken;
    private Long userId;
    private Long totalAmount;
    private String currency;  // 새 필드 (null 허용)
}
```

**규칙:**
- 필드 추가: OK (Consumer에서 null 허용)
- 필드 삭제: 금지 (기존 Consumer 깨짐)
- 필드 타입 변경: 금지
- 호환 불가능한 변경: 새 이벤트 타입 생성 (`OrderCreatedV2Event`)
