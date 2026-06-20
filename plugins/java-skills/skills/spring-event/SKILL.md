---
name: spring-event
description: 이벤트 기반 비동기 처리 설계 및 구현. Saga, 보상 트랜잭션, Outbox 패턴, DLQ, 멱등성. Kafka/RabbitMQ 구현 가이드 포함.
argument-hint: [이벤트 시나리오 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# 이벤트 기반 비동기 처리 스킬

사용자 요구사항: `$ARGUMENTS`

## 역할

MSA 서비스 간 **이벤트 기반 비동기 통신**을 설계하고 구현한다.
`spring-layered` 아키텍처 위에 이벤트 레이어를 추가한다.

---

## 이벤트 아키텍처 개요

### MSA에서 이벤트가 필요한 이유

```
주문 서비스 ──[동기 호출]──→ 결제 서비스 ──[동기 호출]──→ 배송 서비스
     ↑                          ↑                          ↑
     └── 하나라도 장애나면 전체 실패 (강결합)
```

```
주문 서비스 ──[이벤트 발행]──→ 메시지 브로커 ──→ 결제 서비스
                                            ──→ 배송 서비스
                                            ──→ 알림 서비스
     ↑
     └── 각 서비스 독립 처리 (약결합)
```

### 적용 판단 기준
| 상황 | 방식 |
|------|------|
| 즉시 결과가 필요 | 동기 호출 (REST/gRPC) |
| 결과를 기다릴 필요 없음 | 이벤트 발행 |
| 여러 서비스에 알려야 함 | 이벤트 발행 (pub/sub) |
| 트랜잭션 보장 필요 | Saga 패턴 |
| 순서 보장 필요 | 파티션 키 기반 이벤트 |

---

## 이벤트 설계

### 1단계: 이벤트 식별

도메인 이벤트는 **"~가 ~되었다"** 형태로 정의한다:

```
주문이 생성되었다      → OrderCreatedEvent
결제가 완료되었다      → PaymentCompletedEvent
결제가 실패했다        → PaymentFailedEvent
배송이 시작되었다      → DeliveryStartedEvent
재고가 차감되었다      → StockDeductedEvent
재고 차감이 실패했다   → StockDeductionFailedEvent
```

### 2단계: 이벤트 흐름도

```
[주문 서비스]                    [결제 서비스]              [재고 서비스]
     │                               │                        │
     │── OrderCreatedEvent ─────────→│                        │
     │                               │── PaymentCompletedEvent──→│
     │                               │                        │── StockDeductedEvent
     │                               │                        │
     │                          (실패 시)                      │
     │                               │── PaymentFailedEvent──→│
     │←── OrderCancelledEvent ───────│                        │── StockRestoredEvent
```

### 3단계: 이벤트 클래스 설계

```java
// 이벤트 베이스
@Getter
@SuperBuilder
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public abstract class DomainEvent {
    private String eventId;
    private String eventType;
    private ZonedDateTime occurredAt;
    private String aggregateId;  // 비즈니스 식별자 (token)

    protected DomainEvent(String eventType, String aggregateId) {
        this.eventId = UUID.randomUUID().toString();
        this.eventType = eventType;
        this.occurredAt = ZonedDateTime.now();
        this.aggregateId = aggregateId;
    }
}

// 구체 이벤트
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderCreatedEvent extends DomainEvent {
    private String orderToken;
    private Long userId;
    private Long totalAmount;
    private String payMethod;

    @Builder
    public OrderCreatedEvent(String orderToken, Long userId, Long totalAmount, String payMethod) {
        super("ORDER_CREATED", orderToken);
        this.orderToken = orderToken;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.payMethod = payMethod;
    }
}
```

**이벤트 클래스 규칙:**
- 과거형 네이밍: `{Domain}{Action}Event` (`OrderCreatedEvent`)
- 불변 객체: 생성 후 변경 불가
- 필수 필드: eventId, eventType, occurredAt, aggregateId
- 직렬화 가능해야 함 (JSON)
- 소비자에게 필요한 최소한의 데이터만 포함

---

## 구현 방식

### 방식 1: Spring ApplicationEvent (서비스 내부)

같은 애플리케이션 내에서 계층 간 이벤트 전파에 사용한다.

```java
// 이벤트 발행 (Facade에서)
@Service
@RequiredArgsConstructor
public class OrderFacade {
    private final OrderService orderService;
    private final ApplicationEventPublisher eventPublisher;

    public String registerOrder(OrderCommand.RegisterOrder command) {
        var orderToken = orderService.registerOrder(command);

        eventPublisher.publishEvent(OrderCreatedEvent.builder()
                .orderToken(orderToken)
                .userId(command.getUserId())
                .totalAmount(command.getTotalAmount())
                .payMethod(command.getPayMethod())
                .build());

        return orderToken;
    }
}

// 이벤트 구독 (같은 서비스 내)
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventListener {
    private final NotificationService notificationService;

    @Async
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("주문 생성 이벤트 수신: {}", event.getOrderToken());
        notificationService.sendKakao(event.getUserId(), "주문이 완료되었습니다");
    }

    // 트랜잭션 커밋 후 실행 보장
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreatedAfterCommit(OrderCreatedEvent event) {
        log.info("트랜잭션 커밋 후 이벤트 처리: {}", event.getOrderToken());
        // 외부 메시지 브로커로 발행
    }
}
```

**ApplicationEvent 규칙:**
- `@EventListener`: 트랜잭션과 무관하게 즉시 실행
- `@TransactionalEventListener`: 트랜잭션 커밋 후 실행 (외부 발행에 적합)
- `@Async`: 비동기 실행 (별도 스레드)
- 같은 서비스 내부 이벤트에만 사용

### 방식 2: Kafka (서비스 간 통신)

#### 의존성

```gradle
implementation 'org.springframework.kafka:spring-kafka'
```

#### 설정

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
    consumer:
      group-id: ${spring.application.name}
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false
      properties:
        spring.json.trusted.packages: "com.example.*"
    listener:
      ack-mode: manual
```

#### Producer

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventProducer {
    private final KafkaTemplate<String, DomainEvent> kafkaTemplate;

    private static final String TOPIC_ORDER = "order-events";

    public void publishOrderCreated(OrderCreatedEvent event) {
        kafkaTemplate.send(TOPIC_ORDER, event.getAggregateId(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("이벤트 발행 실패: {}", event.getEventId(), ex);
                    } else {
                        log.info("이벤트 발행 성공: topic={}, key={}", 
                                TOPIC_ORDER, event.getAggregateId());
                    }
                });
    }
}
```

#### Consumer

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentEventConsumer {
    private final PaymentService paymentService;
    private final IdempotencyChecker idempotencyChecker;

    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderCreated(OrderCreatedEvent event, Acknowledgment ack) {
        try {
            // 멱등성 체크
            if (idempotencyChecker.isDuplicate(event.getEventId())) {
                log.warn("중복 이벤트 스킵: {}", event.getEventId());
                ack.acknowledge();
                return;
            }

            paymentService.processPayment(event);

            idempotencyChecker.markProcessed(event.getEventId());
            ack.acknowledge();
        } catch (Exception e) {
            log.error("이벤트 처리 실패: {}", event.getEventId(), e);
            // ack 하지 않으면 재시도
        }
    }
}
```

### 방식 3: RabbitMQ (서비스 간 통신)

#### 의존성

```gradle
implementation 'org.springframework.boot:spring-boot-starter-amqp'
```

#### 설정

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
          max-interval: 10000
          multiplier: 2.0
```

#### Exchange/Queue 설정

```java
@Configuration
public class RabbitMQConfig {

    public static final String EXCHANGE_ORDER = "order.exchange";
    public static final String QUEUE_ORDER_CREATED = "order.created.queue";
    public static final String ROUTING_KEY_ORDER_CREATED = "order.created";
    public static final String DLX_EXCHANGE = "dlx.exchange";
    public static final String DLQ_ORDER = "order.created.dlq";

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange(EXCHANGE_ORDER);
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable(QUEUE_ORDER_CREATED)
                .withArgument("x-dead-letter-exchange", DLX_EXCHANGE)
                .withArgument("x-dead-letter-routing-key", "order.created.dead")
                .build();
    }

    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder
                .bind(orderCreatedQueue())
                .to(orderExchange())
                .with(ROUTING_KEY_ORDER_CREATED);
    }

    // Dead Letter
    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange(DLX_EXCHANGE);
    }

    @Bean
    public Queue dlqOrder() {
        return QueueBuilder.durable(DLQ_ORDER).build();
    }

    @Bean
    public Binding dlqBinding() {
        return BindingBuilder
                .bind(dlqOrder())
                .to(dlxExchange())
                .with("order.created.dead");
    }
}
```

#### Producer

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderEventProducer {
    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        rabbitTemplate.convertAndSend(
                RabbitMQConfig.EXCHANGE_ORDER,
                RabbitMQConfig.ROUTING_KEY_ORDER_CREATED,
                event
        );
        log.info("이벤트 발행: {}", event.getEventId());
    }
}
```

#### Consumer

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentEventConsumer {
    private final PaymentService paymentService;

    @RabbitListener(queues = RabbitMQConfig.QUEUE_ORDER_CREATED)
    public void handleOrderCreated(OrderCreatedEvent event, Channel channel,
                                    @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        try {
            paymentService.processPayment(event);
            channel.basicAck(tag, false);
        } catch (Exception e) {
            log.error("이벤트 처리 실패: {}", event.getEventId(), e);
            channel.basicNack(tag, false, false);  // DLQ로 이동
        }
    }
}
```

---

## 핵심 패턴

### Saga 패턴 (보상 트랜잭션)

분산 트랜잭션을 이벤트 체인으로 관리한다. 실패 시 보상 이벤트를 발행하여 롤백한다.

#### Choreography 방식 (각 서비스가 자율적으로 반응)

```
정상 흐름:
주문 생성 → OrderCreatedEvent
         → 결제 처리 → PaymentCompletedEvent
                     → 재고 차감 → StockDeductedEvent
                                 → 주문 확정 완료

실패 흐름 (재고 부족):
주문 생성 → OrderCreatedEvent
         → 결제 처리 → PaymentCompletedEvent
                     → 재고 차감 실패 → StockDeductionFailedEvent
                                      → 결제 취소 → PaymentCancelledEvent
                                                   → 주문 취소 → OrderCancelledEvent
```

```java
// 주문 서비스 - 보상 이벤트 수신
@Component
@RequiredArgsConstructor
public class OrderSagaListener {
    private final OrderService orderService;

    @KafkaListener(topics = "payment-events", groupId = "order-service")
    public void handlePaymentFailed(PaymentFailedEvent event, Acknowledgment ack) {
        log.warn("결제 실패 → 주문 취소: {}", event.getOrderToken());
        orderService.cancelOrder(event.getOrderToken());
        ack.acknowledge();
    }
}

// 결제 서비스 - 보상 이벤트 수신
@Component
@RequiredArgsConstructor
public class PaymentSagaListener {
    private final PaymentService paymentService;

    @KafkaListener(topics = "stock-events", groupId = "payment-service")
    public void handleStockDeductionFailed(StockDeductionFailedEvent event, Acknowledgment ack) {
        log.warn("재고 부족 → 결제 취소: {}", event.getOrderToken());
        paymentService.cancelPayment(event.getOrderToken());
        ack.acknowledge();
    }
}
```

#### Orchestration 방식 (Saga Orchestrator가 전체 조율)

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderSagaOrchestrator {
    private final OrderService orderService;
    private final EventProducer eventProducer;

    // 단계별 처리
    public void handlePaymentResult(PaymentResultEvent event) {
        if (event.isSuccess()) {
            // 다음 단계: 재고 차감 요청
            eventProducer.publish(new StockDeductCommand(event.getOrderToken(), event.getItems()));
        } else {
            // 보상: 주문 취소
            orderService.cancelOrder(event.getOrderToken());
        }
    }

    public void handleStockResult(StockResultEvent event) {
        if (event.isSuccess()) {
            // 최종 완료
            orderService.completeOrder(event.getOrderToken());
        } else {
            // 보상: 결제 취소 → 주문 취소
            eventProducer.publish(new PaymentCancelCommand(event.getOrderToken()));
        }
    }
}
```

**Saga 선택 기준:**
| 기준 | Choreography | Orchestration |
|------|-------------|---------------|
| 복잡도 | 3단계 이하 | 4단계 이상 |
| 가시성 | 흐름 추적 어려움 | 중앙에서 전체 흐름 관리 |
| 결합도 | 서비스 간 약결합 | Orchestrator에 의존 |
| 추천 | 단순한 이벤트 체인 | 복잡한 비즈니스 흐름 |

---

### Outbox 패턴 (이벤트 유실 방지)

DB 트랜잭션과 이벤트 발행의 원자성을 보장한다.

```
문제: Service에서 DB 저장 성공 → 이벤트 발행 실패 → 데이터 불일치
해결: DB에 이벤트를 함께 저장 → 별도 프로세스가 DB에서 읽어서 발행
```

#### Outbox 테이블

```sql
CREATE TABLE outbox_events (
    id            BIGINT NOT NULL AUTO_INCREMENT,
    event_id      VARCHAR(50) NOT NULL,
    event_type    VARCHAR(100) NOT NULL,
    aggregate_id  VARCHAR(50) NOT NULL,
    payload       TEXT NOT NULL,           -- JSON 직렬화된 이벤트
    status        VARCHAR(20) NOT NULL,    -- PENDING, PUBLISHED, FAILED
    created_at    DATETIME(6) NOT NULL,
    published_at  DATETIME(6),
    retry_count   INT DEFAULT 0,
    PRIMARY KEY (id)
) ENGINE=InnoDB;

CREATE INDEX idx_outbox_status ON outbox_events (status, created_at);
```

#### 구현

```java
// 1. 비즈니스 로직 + 이벤트 저장 (같은 트랜잭션)
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    private final OrderStore orderStore;
    private final OutboxStore outboxStore;
    private final ObjectMapper objectMapper;

    @Transactional
    public String registerOrder(OrderCommand.RegisterOrder command) {
        Order order = orderStore.store(command.toEntity());

        // Outbox에 이벤트 저장 (같은 트랜잭션)
        var event = OrderCreatedEvent.builder()
                .orderToken(order.getOrderToken())
                .userId(command.getUserId())
                .build();

        outboxStore.store(OutboxEvent.builder()
                .eventId(event.getEventId())
                .eventType("ORDER_CREATED")
                .aggregateId(order.getOrderToken())
                .payload(objectMapper.writeValueAsString(event))
                .status(OutboxStatus.PENDING)
                .build());

        return order.getOrderToken();
    }
}

// 2. 스케줄러가 Outbox에서 읽어서 발행
@Slf4j
@Component
@RequiredArgsConstructor
public class OutboxPublisher {
    private final OutboxReader outboxReader;
    private final OutboxStore outboxStore;
    private final EventProducer eventProducer;

    @Scheduled(fixedDelay = 1000)  // 1초마다
    public void publishPendingEvents() {
        var pendingEvents = outboxReader.findPendingEvents(100);

        for (var event : pendingEvents) {
            try {
                eventProducer.publish(event.getEventType(), event.getPayload());
                event.markPublished();
                outboxStore.store(event);
            } catch (Exception e) {
                log.error("Outbox 발행 실패: {}", event.getEventId(), e);
                event.incrementRetryCount();
                outboxStore.store(event);
            }
        }
    }
}
```

---

### 멱등성 (Idempotency)

동일한 이벤트가 여러 번 처리되어도 결과가 같도록 보장한다.

```java
// 처리 이력 테이블
// CREATE TABLE processed_events (
//     event_id   VARCHAR(50) PRIMARY KEY,
//     processed_at DATETIME(6) NOT NULL
// );

@Component
@RequiredArgsConstructor
public class IdempotencyChecker {
    private final ProcessedEventRepository repository;

    public boolean isDuplicate(String eventId) {
        return repository.existsByEventId(eventId);
    }

    @Transactional
    public void markProcessed(String eventId) {
        repository.save(ProcessedEvent.of(eventId));
    }
}

// Consumer에서 사용
@KafkaListener(topics = "order-events")
public void handle(OrderCreatedEvent event, Acknowledgment ack) {
    if (idempotencyChecker.isDuplicate(event.getEventId())) {
        ack.acknowledge();
        return;  // 중복 스킵
    }

    // 비즈니스 처리 ...

    idempotencyChecker.markProcessed(event.getEventId());
    ack.acknowledge();
}
```

---

### Dead Letter Queue (DLQ)

처리 실패한 이벤트를 별도 큐로 분리하여 나중에 처리한다.

```java
// Kafka DLQ 설정
@Bean
public ConcurrentKafkaListenerContainerFactory<String, DomainEvent> kafkaListenerContainerFactory(
        ConsumerFactory<String, DomainEvent> consumerFactory,
        KafkaTemplate<String, DomainEvent> kafkaTemplate) {

    var factory = new ConcurrentKafkaListenerContainerFactory<String, DomainEvent>();
    factory.setConsumerFactory(consumerFactory);

    // 3회 재시도 후 DLQ로 이동
    var errorHandler = new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                    (record, ex) -> new TopicPartition(record.topic() + ".dlq", record.partition())),
            new FixedBackOff(1000L, 3L)  // 1초 간격, 3회 재시도
    );
    factory.setCommonErrorHandler(errorHandler);

    return factory;
}
```

---

## 패키지 구조 (이벤트 추가)

기존 `spring-layered` 구조에 이벤트 관련 패키지를 추가한다:

```
com.example.{서비스명}/
├── domain/
│   └── {도메인}/
│       └── event/
│           ├── DomainEvent.java          # 이벤트 베이스
│           ├── {Domain}CreatedEvent.java  # 도메인 이벤트
│           └── {Domain}EventListener.java # 내부 이벤트 리스너
├── infrastructure/
│   └── event/
│       ├── EventProducer.java            # Kafka/RabbitMQ Producer
│       ├── EventConsumer.java            # Kafka/RabbitMQ Consumer
│       ├── OutboxPublisher.java          # Outbox 스케줄러
│       └── IdempotencyChecker.java       # 멱등성 체커
└── config/
    ├── KafkaConfig.java                  # 또는 RabbitMQConfig.java
    └── AsyncConfig.java                  # @Async 설정
```

---

## 설계 체크리스트

- [ ] 이벤트 흐름도에서 모든 실패 케이스에 보상 경로가 있는가
- [ ] 멱등성 처리가 모든 Consumer에 적용되었는가
- [ ] Outbox 패턴으로 이벤트 유실을 방지하고 있는가
- [ ] DLQ로 실패 이벤트를 관리하고 있는가
- [ ] 이벤트에 최소한의 데이터만 포함하고 있는가 (과도한 데이터 금지)
- [ ] 이벤트 순서가 중요한 경우 파티션 키를 설정했는가
- [ ] Saga 보상 트랜잭션이 모든 단계에서 정의되었는가

상세 패턴은 `${CLAUDE_SKILL_DIR}/reference.md`를 참조한다.
