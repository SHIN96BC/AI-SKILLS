# Spring Test 레퍼런스

## 테스트 어노테이션 가이드

| 어노테이션 | 용도 | 컨텍스트 | 속도 |
|-----------|------|---------|------|
| 없음 (순수 JUnit) | Entity, Util 단위 테스트 | 없음 | 매우 빠름 |
| `@ExtendWith(MockitoExtension.class)` | Service, Facade Mock 테스트 | 없음 | 빠름 |
| `@WebMvcTest` | Controller 테스트 | 웹 계층만 | 보통 |
| `@DataJpaTest` | Repository 테스트 | JPA 계층만 | 보통 |
| `@SpringBootTest` | 통합 테스트 | 전체 | 느림 |

## Testcontainers 설정

### 베이스 클래스

```java
@Testcontainers
@SpringBootTest
public abstract class IntegrationTestBase {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("test")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
}
```

### 사용

```java
class OrderIntegrationTest extends IntegrationTestBase {

    @Autowired
    private OrderFacade orderFacade;

    @Test
    @DisplayName("주문 등록 → 조회 통합 테스트")
    void registerAndRetrieveOrder() {
        // given
        var command = OrderFixture.createRegisterOrderCommand();

        // when
        var orderToken = orderFacade.registerOrder(command);
        var result = orderFacade.retrieveOrder(orderToken);

        // then
        assertThat(result.getOrderToken()).isEqualTo(orderToken);
        assertThat(result.getStatus()).isEqualTo("INIT");
    }
}
```

## 이벤트 테스트

### ApplicationEvent 테스트

```java
@SpringBootTest
class OrderEventTest {

    @Autowired
    private OrderFacade orderFacade;

    @MockBean
    private NotificationService notificationService;

    @Test
    @DisplayName("주문 생성 시 알림 이벤트 발생")
    void orderCreated_triggersNotification() {
        // given
        var command = OrderFixture.createRegisterOrderCommand();

        // when
        orderFacade.registerOrder(command);

        // then
        then(notificationService).should().sendKakao(anyString(), anyString());
    }
}
```

### Kafka Consumer 테스트

```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"order-events"})
class OrderEventConsumerTest {

    @Autowired
    private KafkaTemplate<String, DomainEvent> kafkaTemplate;

    @Autowired
    private PaymentService paymentService;

    @Test
    @DisplayName("주문 생성 이벤트 수신 시 결제 처리")
    void consumeOrderCreatedEvent() throws Exception {
        // given
        var event = OrderCreatedEvent.builder()
                .orderToken("ord_test123")
                .userId(1L)
                .totalAmount(10000L)
                .payMethod("CARD")
                .build();

        // when
        kafkaTemplate.send("order-events", event.getAggregateId(), event).get();

        // then (비동기이므로 대기 필요)
        await().atMost(5, TimeUnit.SECONDS).untilAsserted(() -> {
            // 결제 처리 검증 ...
        });
    }
}
```

## Strategy 패턴 테스트

```java
@ExtendWith(MockitoExtension.class)
class PaymentProcessorTest {

    @InjectMocks
    private PaymentProcessorImpl paymentProcessor;

    @Mock
    private List<PaymentValidator> paymentValidatorList;

    @Mock
    private List<PaymentApiCaller> paymentApiCallerList;

    @Test
    @DisplayName("카카오페이 결제 라우팅")
    void routeToKakaoPay() {
        // given
        var kakaoPayCaller = mock(KakaoPayApiCaller.class);
        given(kakaoPayCaller.support(PayMethod.KAKAO_PAY)).willReturn(true);

        var callers = List.of(kakaoPayCaller);
        ReflectionTestUtils.setField(paymentProcessor, "paymentApiCallerList", callers);

        var request = OrderCommand.PaymentRequest.builder()
                .payMethod(PayMethod.KAKAO_PAY)
                .build();

        // when
        paymentProcessor.pay(OrderFixture.createOrder(), request);

        // then
        then(kakaoPayCaller).should().pay(request);
    }
}
```

## MapStruct Mapper 테스트

```java
@SpringBootTest(classes = {OrderDtoMapperImpl.class})
class OrderDtoMapperTest {

    @Autowired
    private OrderDtoMapper orderDtoMapper;

    @Test
    @DisplayName("RegisterOrderRequest → RegisterOrder 매핑")
    void mapRequestToCommand() {
        // given
        var request = OrderFixture.createRegisterOrderRequest();

        // when
        var command = orderDtoMapper.of(request);

        // then
        assertThat(command.getUserId()).isEqualTo(request.getUserId());
        assertThat(command.getPayMethod()).isEqualTo(request.getPayMethod());
        assertThat(command.getReceiverName()).isEqualTo(request.getReceiverName());
    }
}
```

## 테스트 실행 명령

```bash
# 전체 테스트
./gradlew test

# 특정 클래스
./gradlew test --tests "com.example.order.domain.order.OrderTest"

# 특정 메서드
./gradlew test --tests "com.example.order.domain.order.OrderTest.createOrder"

# 특정 패키지
./gradlew test --tests "com.example.order.domain.*"

# 커버리지 리포트 (jacoco)
./gradlew jacocoTestReport
```

## JaCoCo 설정 (build.gradle)

```gradle
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.11"
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }

    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/config/**',
                '**/common/**',
                '**/*Application*',
                '**/*Dto*',
                '**/*Command*',
                '**/*Info*'
            ])
        }))
    }
}

test {
    finalizedBy jacocoTestReport
}
```

## 테스트 격리

```java
// 트랜잭션 롤백 (DataJpaTest는 기본 적용)
@Transactional
@SpringBootTest
class OrderIntegrationTest { ... }

// 데이터 초기화 (롤백 불가능한 경우)
@AfterEach
void tearDown() {
    orderRepository.deleteAll();
}

// 테스트 순서 보장 (필요한 경우만)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderSequentialTest {
    @Test @org.junit.jupiter.api.Order(1)
    void step1() { ... }

    @Test @org.junit.jupiter.api.Order(2)
    void step2() { ... }
}
```
