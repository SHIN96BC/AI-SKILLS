---
name: spring-test
description: Spring Boot 테스트 코드 작성. JUnit5 + Mockito + MockMvc. 레이어별 단위/통합 테스트, Testcontainers DB 테스트.
argument-hint: [테스트 대상 클래스 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# Spring Boot 테스트 스킬

사용자 요구사항: `$ARGUMENTS`

## 역할

`spring-layered` 아키텍처에 맞는 **레이어별 테스트 코드**를 작성한다.

---

## 기술 스택

- JUnit 5
- Mockito (단위 테스트 Mock)
- MockMvc (Controller 테스트)
- Spring Boot Test (`@SpringBootTest`)
- Testcontainers (DB 통합 테스트, 선택)
- AssertJ (가독성 높은 Assertion)

### 의존성

```gradle
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'

    // Testcontainers (선택)
    testImplementation 'org.testcontainers:testcontainers:1.19.8'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.8'
    testImplementation 'org.testcontainers:mysql:1.19.8'
}
```

---

## 테스트 구조

```
src/test/java/com/example/{서비스명}/
├── domain/
│   └── {도메인}/
│       ├── {Domain}Test.java              # Entity 단위 테스트
│       ├── {Domain}ServiceTest.java       # Service 단위 테스트
│       └── {Domain}ValidatorTest.java     # Validator 테스트
├── application/
│   └── {도메인}/
│       └── {Domain}FacadeTest.java        # Facade 단위 테스트
├── interfaces/
│   └── {도메인}/
│       └── {Domain}ApiControllerTest.java # Controller 테스트 (MockMvc)
├── infrastructure/
│   └── {도메인}/
│       └── {Domain}RepositoryTest.java    # Repository 통합 테스트
└── fixture/
    └── {Domain}Fixture.java               # 테스트 데이터 팩토리
```

---

## 레이어별 테스트 패턴

### 1. Entity 테스트 (Domain)

Entity의 생성, 비즈니스 메서드, 상태 전이를 검증한다.

```java
class OrderTest {

    @Test
    @DisplayName("주문 생성 시 상태는 INIT이고 토큰이 생성된다")
    void createOrder() {
        // given
        var order = Order.builder()
                .userId(1L)
                .payMethod("CARD")
                .receiverName("홍길동")
                .receiverPhone("01012345678")
                .receiverZipcode("12345")
                .receiverAddress1("서울시")
                .receiverAddress2("101호")
                .etcMessage("부재시 문 앞")
                .build();

        // then
        assertThat(order.getStatus()).isEqualTo(Order.Status.INIT);
        assertThat(order.getOrderToken()).startsWith("ord_");
        assertThat(order.getUserId()).isEqualTo(1L);
    }

    @Test
    @DisplayName("필수 필드 누락 시 InvalidParamException 발생")
    void createOrder_withoutUserId_throwsException() {
        assertThatThrownBy(() -> Order.builder()
                .userId(null)
                .payMethod("CARD")
                .receiverName("홍길동")
                .build())
                .isInstanceOf(InvalidParamException.class);
    }

    @Test
    @DisplayName("주문 완료 처리")
    void orderComplete() {
        // given
        var order = OrderFixture.createOrder();

        // when
        order.orderComplete();

        // then
        assertThat(order.getStatus()).isEqualTo(Order.Status.ORDER_COMPLETE);
    }

    @Test
    @DisplayName("이미 완료된 주문은 다시 완료 처리할 수 없다")
    void orderComplete_alreadyComplete_throwsException() {
        // given
        var order = OrderFixture.createOrder();
        order.orderComplete();

        // when & then
        assertThatThrownBy(order::orderComplete)
                .isInstanceOf(IllegalStatusException.class);
    }

    @Test
    @DisplayName("주문 총액 계산")
    void calculateTotalAmount() {
        // given
        var order = OrderFixture.createOrderWithItems(
                OrderFixture.createOrderItem(10000L, 2),  // 20000
                OrderFixture.createOrderItem(5000L, 3)    // 15000
        );

        // when
        var totalAmount = order.calculateTotalAmount();

        // then
        assertThat(totalAmount).isEqualTo(35000L);
    }
}
```

**Entity 테스트 규칙:**
- 순수 Java 테스트 (Spring 컨텍스트 불필요)
- 생성 성공/실패, 비즈니스 메서드, 상태 전이 검증
- Edge case 필수: null, 빈값, 잘못된 상태에서의 호출

### 2. Service 테스트 (Domain)

Service 비즈니스 로직을 Mock 기반으로 테스트한다.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @InjectMocks
    private OrderServiceImpl orderService;

    @Mock
    private OrderStore orderStore;

    @Mock
    private OrderReader orderReader;

    @Mock
    private OrderItemSeriesFactory orderItemSeriesFactory;

    @Mock
    private OrderInfoMapper orderInfoMapper;

    @Test
    @DisplayName("주문 등록 성공")
    void registerOrder() {
        // given
        var command = OrderFixture.createRegisterOrderCommand();
        var order = OrderFixture.createOrder();

        given(orderStore.store(any(Order.class))).willReturn(order);
        given(orderItemSeriesFactory.store(any(), any())).willReturn(List.of());

        // when
        var orderToken = orderService.registerOrder(command);

        // then
        assertThat(orderToken).isEqualTo(order.getOrderToken());
        then(orderStore).should().store(any(Order.class));
        then(orderItemSeriesFactory).should().store(eq(order), eq(command));
    }

    @Test
    @DisplayName("주문 조회 - 존재하지 않는 주문")
    void retrieveOrder_notFound() {
        // given
        given(orderReader.getOrder("ord_invalid"))
                .willThrow(new EntityNotFoundException());

        // when & then
        assertThatThrownBy(() -> orderService.retrieveOrder("ord_invalid"))
                .isInstanceOf(EntityNotFoundException.class);
    }
}
```

**Service 테스트 규칙:**
- `@ExtendWith(MockitoExtension.class)` — Spring 컨텍스트 없이 Mock
- `@InjectMocks` 대상: ServiceImpl
- `@Mock` 대상: Store, Reader, Factory, Mapper
- `given()` → `when()` → `then()` BDD 스타일
- 성공/실패/예외 케이스 각각 작성

### 3. Facade 테스트 (Application)

```java
@ExtendWith(MockitoExtension.class)
class OrderFacadeTest {

    @InjectMocks
    private OrderFacade orderFacade;

    @Mock
    private OrderService orderService;

    @Mock
    private NotificationService notificationService;

    @Test
    @DisplayName("주문 등록 시 알림 발송")
    void registerOrder_sendsNotification() {
        // given
        var command = OrderFixture.createRegisterOrderCommand();
        given(orderService.registerOrder(command)).willReturn("ord_test123");

        // when
        var result = orderFacade.registerOrder(command);

        // then
        assertThat(result).isEqualTo("ord_test123");
        then(notificationService).should().sendKakao(anyString(), anyString());
    }
}
```

### 4. Controller 테스트 (Interfaces)

MockMvc로 HTTP 요청/응답을 검증한다.

```java
@WebMvcTest(OrderApiController.class)
class OrderApiControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderFacade orderFacade;

    @MockBean
    private OrderDtoMapper orderDtoMapper;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    @DisplayName("POST /api/v1/orders/init - 주문 등록 성공")
    void registerOrder() throws Exception {
        // given
        var request = OrderFixture.createRegisterOrderRequest();
        var command = OrderFixture.createRegisterOrderCommand();
        var response = OrderDto.RegisterOrderResponse.builder()
                .orderToken("ord_test123")
                .build();

        given(orderDtoMapper.of(any(OrderDto.RegisterOrderRequest.class))).willReturn(command);
        given(orderFacade.registerOrder(command)).willReturn("ord_test123");
        given(orderDtoMapper.of("ord_test123")).willReturn(response);

        // when & then
        mockMvc.perform(post("/api/v1/orders/init")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.result").value("SUCCESS"))
                .andExpect(jsonPath("$.data.orderToken").value("ord_test123"));
    }

    @Test
    @DisplayName("POST /api/v1/orders/init - Validation 실패")
    void registerOrder_validationFail() throws Exception {
        // given
        var request = new OrderDto.RegisterOrderRequest();  // 필수값 누락

        // when & then
        mockMvc.perform(post("/api/v1/orders/init")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.result").value("FAIL"));
    }

    @Test
    @DisplayName("GET /api/v1/orders/{orderToken} - 주문 조회 성공")
    void retrieveOrder() throws Exception {
        // given
        var orderInfo = OrderFixture.createOrderInfo();
        var response = OrderFixture.createOrderDtoMain();

        given(orderFacade.retrieveOrder("ord_test123")).willReturn(orderInfo);
        given(orderDtoMapper.of(orderInfo)).willReturn(response);

        // when & then
        mockMvc.perform(get("/api/v1/orders/ord_test123"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.result").value("SUCCESS"))
                .andExpect(jsonPath("$.data.orderToken").value("ord_test123"));
    }
}
```

**Controller 테스트 규칙:**
- `@WebMvcTest(컨트롤러.class)` — 웹 계층만 로드
- `@MockBean` — Facade, DtoMapper
- 성공, Validation 실패, 비즈니스 예외 각각 테스트
- `jsonPath()`로 응답 구조 검증

### 5. Repository 테스트 (Infrastructure)

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("orderToken으로 주문 조회")
    void findByOrderToken() {
        // given
        var order = OrderFixture.createOrder();
        orderRepository.save(order);

        // when
        var result = orderRepository.findByOrderToken(order.getOrderToken());

        // then
        assertThat(result).isPresent();
        assertThat(result.get().getUserId()).isEqualTo(order.getUserId());
    }

    @Test
    @DisplayName("존재하지 않는 orderToken 조회 시 빈 Optional")
    void findByOrderToken_notFound() {
        var result = orderRepository.findByOrderToken("ord_nonexistent");
        assertThat(result).isEmpty();
    }
}
```

### 6. Validator 테스트

```java
class PayAmountValidatorTest {

    private final PayAmountValidator validator = new PayAmountValidator();

    @Test
    @DisplayName("주문 금액과 결제 금액이 일치하면 통과")
    void validate_success() {
        // given
        var order = OrderFixture.createOrderWithTotalAmount(10000L);
        var request = OrderCommand.PaymentRequest.builder()
                .amount(10000L)
                .build();

        // when & then
        assertThatCode(() -> validator.validate(order, request))
                .doesNotThrowAnyException();
    }

    @Test
    @DisplayName("주문 금액과 결제 금액 불일치 시 예외")
    void validate_amountMismatch() {
        // given
        var order = OrderFixture.createOrderWithTotalAmount(10000L);
        var request = OrderCommand.PaymentRequest.builder()
                .amount(99999L)
                .build();

        // when & then
        assertThatThrownBy(() -> validator.validate(order, request))
                .isInstanceOf(InvalidParamException.class);
    }
}
```

---

## Fixture (테스트 데이터 팩토리)

테스트에서 반복되는 객체 생성을 중앙화한다.

```java
public class OrderFixture {

    public static Order createOrder() {
        return Order.builder()
                .userId(1L)
                .payMethod("CARD")
                .receiverName("홍길동")
                .receiverPhone("01012345678")
                .receiverZipcode("12345")
                .receiverAddress1("서울시 강남구")
                .receiverAddress2("101호")
                .etcMessage("부재시 문 앞")
                .build();
    }

    public static OrderCommand.RegisterOrder createRegisterOrderCommand() {
        return OrderCommand.RegisterOrder.builder()
                .userId(1L)
                .payMethod("CARD")
                .receiverName("홍길동")
                .receiverPhone("01012345678")
                .receiverZipcode("12345")
                .receiverAddress1("서울시 강남구")
                .receiverAddress2("101호")
                .etcMessage("부재시 문 앞")
                .orderItemList(List.of(createRegisterOrderItemCommand()))
                .build();
    }

    public static OrderCommand.RegisterOrderItem createRegisterOrderItemCommand() {
        return OrderCommand.RegisterOrderItem.builder()
                .orderCount(1)
                .itemToken("itm_test123")
                .itemName("테스트 상품")
                .itemPrice(10000L)
                .build();
    }

    public static OrderDto.RegisterOrderRequest createRegisterOrderRequest() {
        var request = new OrderDto.RegisterOrderRequest();
        request.setUserId(1L);
        request.setPayMethod("CARD");
        request.setReceiverName("홍길동");
        request.setReceiverPhone("01012345678");
        request.setReceiverZipcode("12345");
        request.setReceiverAddress1("서울시 강남구");
        request.setReceiverAddress2("101호");
        return request;
    }

    public static OrderItem createOrderItem(Long itemPrice, Integer orderCount) {
        return OrderItem.builder()
                .orderCount(orderCount)
                .itemPrice(itemPrice)
                .itemToken("itm_test")
                .itemName("테스트 상품")
                .build();
    }

    public static OrderInfo.Main createOrderInfo() {
        return OrderInfo.Main.builder()
                .orderToken("ord_test123")
                .userId(1L)
                .payMethod("CARD")
                .totalAmount(10000L)
                .status("INIT")
                .statusDescription("주문시작")
                .build();
    }
}
```

**Fixture 규칙:**
- 도메인별 `{Domain}Fixture.java` 생성
- `fixture/` 패키지에 모아둠
- 기본값이 유효한 객체를 생성 (테스트에서 필요한 값만 오버라이드)
- Entity, Command, DTO, Info 각각에 대한 팩토리 메서드 제공

---

## 테스트 작성 원칙

### 네이밍
- 클래스: `{대상}Test` (`OrderServiceTest`)
- 메서드: `@DisplayName`으로 한글 설명 (메서드명은 영문)
- 메서드명 패턴: `{메서드}_{시나리오}` (`registerOrder_success`, `registerOrder_invalidParam`)

### 구조: Given-When-Then
```java
@Test
@DisplayName("설명")
void methodName() {
    // given - 준비
    // when - 실행
    // then - 검증
}
```

### Assertion 스타일 (AssertJ)
```java
// 값 비교
assertThat(result).isEqualTo(expected);
assertThat(result).isNotNull();
assertThat(list).hasSize(3);
assertThat(string).startsWith("ord_");
assertThat(string).contains("test");

// 예외 검증
assertThatThrownBy(() -> service.doSomething())
        .isInstanceOf(InvalidParamException.class)
        .hasMessageContaining("필수값");

// 예외 없음 검증
assertThatCode(() -> service.doSomething())
        .doesNotThrowAnyException();
```

### Mock 스타일 (BDD Mockito)
```java
// Mock 설정
given(mock.method(any())).willReturn(value);
given(mock.method(any())).willThrow(new Exception());

// 호출 검증
then(mock).should().method(any());
then(mock).should(times(2)).method(any());
then(mock).should(never()).method(any());
then(mock).shouldHaveNoMoreInteractions();
```

### 테스트 커버리지 우선순위
1. **필수**: Entity 비즈니스 메서드, Service 핵심 로직
2. **권장**: Controller 요청/응답, Validator
3. **선택**: Repository 커스텀 쿼리, Facade

상세 패턴은 `${CLAUDE_SKILL_DIR}/reference.md`를 참조한다.
