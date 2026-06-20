---
name: spring-layered
description: Spring Boot 4계층 레이어드 아키텍처 코드 구현. interfaces→application→domain→infrastructure 계층 분리, Store/Reader 패턴, MapStruct 매핑.
argument-hint: [도메인명 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# Spring Boot 레이어드 아키텍처 구현 스킬

사용자 요구사항: `$ARGUMENTS`

## 역할

Spring Boot MSA 서비스를 **4계층 레이어드 아키텍처**로 구현한다.
`spring-design` 스킬의 설계 산출물을 기반으로 코드를 작성한다.

---

## 기술 스택

- Java 17+
- Spring Boot 3.x
- Spring Data JPA + Hibernate
- MapStruct (DTO/Command/Info 변환)
- Flyway (DB 마이그레이션)
- MySQL 8.0
- Gradle
- Lombok

---

## 패키지 구조

```
com.example.{서비스명}/
├── {Service}Application.java
├── interfaces/
│   └── {도메인}/
│       ├── {Domain}ApiController.java
│       ├── {Domain}Dto.java
│       └── {Domain}DtoMapper.java
├── application/
│   └── {도메인}/
│       └── {Domain}Facade.java
├── domain/
│   ├── AbstractEntity.java
│   └── {도메인}/
│       ├── {Domain}.java
│       ├── {Domain}Command.java
│       ├── {Domain}Info.java
│       ├── {Domain}Service.java
│       ├── {Domain}ServiceImpl.java
│       ├── {Domain}Reader.java
│       ├── {Domain}Store.java
│       └── {Domain}InfoMapper.java
├── infrastructure/
│   └── {도메인}/
│       ├── {Domain}Repository.java
│       ├── {Domain}ReaderImpl.java
│       └── {Domain}StoreImpl.java
├── common/
│   ├── exception/
│   │   ├── BaseException.java
│   │   ├── InvalidParamException.java
│   │   ├── EntityNotFoundException.java
│   │   └── IllegalStatusException.java
│   ├── response/
│   │   ├── CommonResponse.java
│   │   └── CommonControllerAdvice.java
│   ├── interceptor/
│   │   └── CommonHttpRequestInterceptor.java
│   └── util/
│       └── TokenGenerator.java
└── config/
    └── JpaAuditingConfiguration.java
```

---

## 계층별 구현 규칙

### 1. Domain 계층 (핵심)

비즈니스 로직의 중심. 다른 계층에 의존하지 않는다.

#### Entity

```java
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
@Table(name = "orders")
public class Order extends AbstractEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String orderToken;
    private Long userId;

    @Enumerated(EnumType.STRING)
    private Status status;

    @Embedded
    private DeliveryFragment deliveryFragment;

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "order", cascade = CascadeType.PERSIST)
    private List<OrderItem> orderItems = Lists.newArrayList();

    @Getter
    @RequiredArgsConstructor
    public enum Status {
        INIT("주문시작"),
        ORDER_COMPLETE("주문완료"),
        DELIVERY_PREPARE("배송준비"),
        IN_DELIVERY("배송중"),
        DELIVERY_COMPLETE("배송완료");

        private final String description;
    }

    @Builder
    public Order(Long userId, String payMethod, String receiverName, /* ... */) {
        if (userId == null) throw new InvalidParamException("Order.userId");
        // 필수 필드 검증 ...

        this.orderToken = TokenGenerator.randomCharacterWithPrefix("ord_");
        this.userId = userId;
        this.status = Status.INIT;
        // ...
    }

    // 비즈니스 메서드
    public void orderComplete() {
        if (this.status != Status.INIT) throw new IllegalStatusException();
        this.status = Status.ORDER_COMPLETE;
    }

    public Long calculateTotalAmount() {
        return orderItems.stream()
                .mapToLong(OrderItem::calculateTotalAmount)
                .sum();
    }

    public boolean isAlreadyPaymentComplete() {
        return this.status != Status.INIT;
    }
}
```

**Entity 규칙:**
- `@NoArgsConstructor(access = PROTECTED)` — JPA용, 외부 사용 금지
- `@Builder` 생성자에서 필수 필드 null 검증
- Setter 금지. 상태 변경은 비즈니스 메서드로만
- Token 생성: `TokenGenerator.randomCharacterWithPrefix("{prefix}_")`
- Status enum은 Entity 내부 inner class로 정의

#### Value Object

```java
@Embeddable
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class DeliveryFragment {
    private String receiverName;
    private String receiverPhone;
    private String receiverZipcode;
    private String receiverAddress1;
    private String receiverAddress2;
    private String etcMessage;

    @Builder
    public DeliveryFragment(String receiverName, String receiverPhone, /* ... */) {
        if (StringUtils.isEmpty(receiverName)) throw new InvalidParamException("receiverName");
        // 필수 필드 검증
        this.receiverName = receiverName;
        // ...
    }
}
```

#### Command (입력 객체)

```java
public class OrderCommand {

    @Getter
    @Builder
    @ToString
    public static class RegisterOrder {
        private final Long userId;
        private final String payMethod;
        private final String receiverName;
        private final String receiverPhone;
        private final String receiverZipcode;
        private final String receiverAddress1;
        private final String receiverAddress2;
        private final String etcMessage;
        private final List<RegisterOrderItem> orderItemList;

        public Order toEntity() {
            return Order.builder()
                    .userId(userId)
                    .payMethod(payMethod)
                    .receiverName(receiverName)
                    // ...
                    .build();
        }
    }

    @Getter
    @Builder
    @ToString
    public static class RegisterOrderItem {
        private final Integer orderCount;
        private final String itemToken;
        private final String itemName;
        private final Long itemPrice;
        private final List<RegisterOrderItemOptionGroup> orderItemOptionGroupList;

        public OrderItem toEntity(Order order, Item item) {
            return OrderItem.builder()
                    .order(order)
                    .orderCount(orderCount)
                    .partnerId(item.getPartnerId())
                    .itemId(item.getId())
                    .itemToken(itemToken)
                    .itemName(itemName)
                    .itemPrice(itemPrice)
                    .build();
        }
    }
}
```

**Command 규칙:**
- inner static class로 정의
- `@Getter @Builder @ToString`
- `toEntity()` 메서드로 Entity 변환 담당
- 불변 객체 (`final` 필드)

#### Info (출력 객체)

```java
public class OrderInfo {

    @Getter
    @Builder
    @ToString
    public static class Main {
        private final Long orderId;
        private final String orderToken;
        private final Long userId;
        private final String payMethod;
        private final Long totalAmount;
        private final DeliveryInfo deliveryInfo;
        private final ZonedDateTime orderedAt;
        private final String status;
        private final String statusDescription;
        private final List<OrderItem> orderItemList;
    }

    @Getter
    @Builder
    @ToString
    public static class DeliveryInfo {
        private final String receiverName;
        private final String receiverPhone;
        // ...
    }
}
```

#### Store/Reader 인터페이스

```java
// 저장 책임
public interface OrderStore {
    Order store(Order order);
}

// 조회 책임
public interface OrderReader {
    Order getOrder(String orderToken);
}
```

#### Service 인터페이스 + 구현

```java
// 인터페이스
public interface OrderService {
    String registerOrder(OrderCommand.RegisterOrder command);
    OrderInfo.Main retrieveOrder(String orderToken);
    void paymentOrder(OrderCommand.PaymentRequest command);
}

// 구현
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {
    private final OrderStore orderStore;
    private final OrderReader orderReader;
    private final OrderItemSeriesFactory orderItemSeriesFactory;
    private final OrderInfoMapper orderInfoMapper;

    @Override
    @Transactional
    public String registerOrder(OrderCommand.RegisterOrder command) {
        Order order = orderStore.store(command.toEntity());
        orderItemSeriesFactory.store(order, command);
        return order.getOrderToken();
    }

    @Override
    @Transactional(readOnly = true)
    public OrderInfo.Main retrieveOrder(String orderToken) {
        var order = orderReader.getOrder(orderToken);
        return orderInfoMapper.of(order, order.getOrderItems());
    }
}
```

**Service 규칙:**
- 인터페이스와 구현 분리
- `@Transactional` — 쓰기: 기본, 읽기: `readOnly = true`
- Store/Reader를 통해서만 데이터 접근 (Repository 직접 사용 금지)
- 반환: 등록 → token(String), 조회 → Info 객체

#### InfoMapper (MapStruct)

```java
@Mapper(
        componentModel = "spring",
        injectionStrategy = InjectionStrategy.CONSTRUCTOR,
        unmappedTargetPolicy = ReportingPolicy.ERROR
)
public interface OrderInfoMapper {
    OrderInfo.Main of(Order order, List<OrderItem> orderItems);
    OrderInfo.OrderItem of(OrderItem orderItem);
}
```

---

### 2. Application 계층 (Facade)

도메인 서비스를 조합하고, 외부 서비스(알림 등)를 호출한다.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderFacade {
    private final OrderService orderService;
    private final NotificationService notificationService;

    public String registerOrder(OrderCommand.RegisterOrder command) {
        var orderToken = orderService.registerOrder(command);
        notificationService.sendKakao("ORDER_COMPLETE", "주문 완료");
        return orderToken;
    }

    public OrderInfo.Main retrieveOrder(String orderToken) {
        return orderService.retrieveOrder(orderToken);
    }
}
```

**Facade 규칙:**
- `@Transactional` 붙이지 않음 (Service 레벨에서 관리)
- 여러 Service 조합 가능
- 외부 서비스 호출 (알림, 이벤트 발행 등)
- 비즈니스 로직 포함 금지

---

### 3. Interfaces 계층 (Controller + DTO)

#### DTO

```java
public class OrderDto {

    // === Request ===
    @Getter
    @Setter
    @ToString
    public static class RegisterOrderRequest {
        @NotNull(message = "userId 는 필수값입니다")
        private Long userId;

        @NotBlank(message = "payMethod 는 필수값입니다")
        private String payMethod;

        @NotBlank(message = "receiverName 는 필수값입니다")
        private String receiverName;
        // ...

        @NotEmpty(message = "orderItemList 는 필수값입니다")
        private List<RegisterOrderItem> orderItemList;
    }

    @Getter
    @Setter
    @ToString
    public static class RegisterOrderItem {
        @NotNull private Integer orderCount;
        @NotBlank private String itemToken;
        @NotBlank private String itemName;
        @NotNull private Long itemPrice;
    }

    // === Response ===
    @Getter
    @Builder
    @ToString
    public static class Main {
        private final String orderToken;
        private final Long userId;
        private final String payMethod;
        private final Long totalAmount;
        private final String status;
        private final String statusDescription;
        private final DeliveryInfo deliveryInfo;
        private final List<OrderItem> orderItemList;
    }

    @Getter
    @Builder
    @ToString
    public static class RegisterOrderResponse {
        private final String orderToken;
    }
}
```

**DTO 규칙:**
- inner static class로 Request/Response 구분
- Request: `@Getter @Setter` + Bean Validation
- Response: `@Getter @Builder` (불변)
- 도메인 객체(Entity, Command, Info) 노출 금지

#### DtoMapper (MapStruct)

```java
@Mapper(
        componentModel = "spring",
        injectionStrategy = InjectionStrategy.CONSTRUCTOR,
        unmappedTargetPolicy = ReportingPolicy.ERROR
)
public interface OrderDtoMapper {
    // DTO → Command
    OrderCommand.RegisterOrder of(OrderDto.RegisterOrderRequest request);
    OrderCommand.RegisterOrderItem of(OrderDto.RegisterOrderItem request);

    // Info → DTO
    @Mappings({
            @Mapping(source = "orderedAt", target = "orderedAt", dateFormat = "yyyy-MM-dd HH:mm:ss")
    })
    OrderDto.Main of(OrderInfo.Main mainResult);

    // 단순 래핑
    default OrderDto.RegisterOrderResponse of(String orderToken) {
        return OrderDto.RegisterOrderResponse.builder()
                .orderToken(orderToken)
                .build();
    }
}
```

#### Controller

```java
@Slf4j
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderApiController {
    private final OrderFacade orderFacade;
    private final OrderDtoMapper orderDtoMapper;

    @PostMapping("/init")
    public CommonResponse registerOrder(@RequestBody @Valid OrderDto.RegisterOrderRequest request) {
        var command = orderDtoMapper.of(request);
        var orderToken = orderFacade.registerOrder(command);
        var response = orderDtoMapper.of(orderToken);
        return CommonResponse.success(response);
    }

    @GetMapping("/{orderToken}")
    public CommonResponse retrieveOrder(@PathVariable String orderToken) {
        var orderInfo = orderFacade.retrieveOrder(orderToken);
        var response = orderDtoMapper.of(orderInfo);
        return CommonResponse.success(response);
    }
}
```

**Controller 규칙:**
- `@RequestMapping("/api/v1/{도메인s}")`
- Facade만 호출 (Service 직접 호출 금지)
- MapStruct로 DTO↔Command 변환
- 반환: `CommonResponse.success(data)`
- Validation: `@Valid` + `@RequestBody`

---

### 4. Infrastructure 계층 (구현체)

#### Repository

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    Optional<Order> findByOrderToken(String orderToken);
}
```

#### Store/Reader 구현

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderStoreImpl implements OrderStore {
    private final OrderRepository orderRepository;

    @Override
    public Order store(Order order) {
        return orderRepository.save(order);
    }
}

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderReaderImpl implements OrderReader {
    private final OrderRepository orderRepository;

    @Override
    public Order getOrder(String orderToken) {
        return orderRepository.findByOrderToken(orderToken)
                .orElseThrow(EntityNotFoundException::new);
    }
}
```

**Infrastructure 규칙:**
- Domain 계층의 인터페이스(Store, Reader)를 구현
- JPA Repository는 이 계층에서만 사용
- 외부 API 호출(PaymentApiCaller 등)도 이 계층

---

## 디자인 패턴

### Strategy 패턴 (결제 등 분기 처리)

```java
// Interface
public interface PaymentApiCaller {
    boolean support(PayMethod payMethod);
    void pay(OrderCommand.PaymentRequest request);
}

// 구현체들
@Component
public class KakaoPayApiCaller implements PaymentApiCaller {
    @Override
    public boolean support(PayMethod payMethod) {
        return PayMethod.KAKAO_PAY == payMethod;
    }
    @Override
    public void pay(OrderCommand.PaymentRequest request) { /* 카카오페이 */ }
}

// 라우팅
@Component
@RequiredArgsConstructor
public class PaymentProcessorImpl implements PaymentProcessor {
    private final List<PaymentApiCaller> paymentApiCallerList;

    private PaymentApiCaller routingApiCaller(OrderCommand.PaymentRequest request) {
        return paymentApiCallerList.stream()
                .filter(caller -> caller.support(request.getPayMethod()))
                .findFirst()
                .orElseThrow(InvalidParamException::new);
    }
}
```

### Validator 체인 (순차 검증)

```java
public interface PaymentValidator {
    void validate(Order order, OrderCommand.PaymentRequest request);
}

@Component
@Order(value = 1)
public class PayAmountValidator implements PaymentValidator {
    @Override
    public void validate(Order order, OrderCommand.PaymentRequest request) {
        if (!order.calculateTotalAmount().equals(request.getAmount()))
            throw new InvalidParamException("주문가격이 일치하지 않습니다");
    }
}

// 사용: paymentValidatorList.forEach(v -> v.validate(order, request));
```

### Factory 패턴 (중첩 객체 생성)

```java
// Interface (domain)
public interface OrderItemSeriesFactory {
    List<OrderItem> store(Order order, OrderCommand.RegisterOrder command);
}

// Impl (infrastructure)
@Component
@RequiredArgsConstructor
public class OrderItemSeriesFactoryImpl implements OrderItemSeriesFactory {
    private final ItemReader itemReader;
    private final OrderStore orderStore;

    @Override
    public List<OrderItem> store(Order order, OrderCommand.RegisterOrder command) {
        return command.getOrderItemList().stream()
                .map(itemRequest -> {
                    var item = itemReader.getItemBy(itemRequest.getItemToken());
                    var orderItem = orderStore.store(itemRequest.toEntity(order, item));
                    // 하위 옵션 그룹, 옵션 재귀 저장 ...
                    return orderItem;
                })
                .collect(Collectors.toList());
    }
}
```

---

## 공통 모듈 (common)

### CommonResponse

```java
@Getter
@Builder
public class CommonResponse<T> {
    private Result result;
    private T data;
    private String message;
    private String errorCode;

    public enum Result { SUCCESS, FAIL }

    public static <T> CommonResponse<T> success(T data) {
        return (CommonResponse<T>) CommonResponse.builder()
                .result(Result.SUCCESS)
                .data(data)
                .build();
    }

    public static CommonResponse fail(String message, String errorCode) {
        return CommonResponse.builder()
                .result(Result.FAIL)
                .message(message)
                .errorCode(errorCode)
                .build();
    }

    public static CommonResponse fail(ErrorCode errorCode) {
        return CommonResponse.builder()
                .result(Result.FAIL)
                .message(errorCode.getErrorMessage())
                .errorCode(errorCode.name())
                .build();
    }
}
```

### 예외 계층

```java
@Getter
public class BaseException extends RuntimeException {
    private CommonResponse.ErrorCode errorCode;

    public BaseException() {
        super(ErrorCode.COMMON_SYSTEM_ERROR.getErrorMessage());
        this.errorCode = ErrorCode.COMMON_SYSTEM_ERROR;
    }

    public BaseException(ErrorCode errorCode) {
        super(errorCode.getErrorMessage());
        this.errorCode = errorCode;
    }

    public BaseException(String message, ErrorCode errorCode) {
        super(message);
        this.errorCode = errorCode;
    }
}

// InvalidParamException → COMMON_INVALID_PARAMETER
// EntityNotFoundException → COMMON_ENTITY_NOT_FOUND
// IllegalStatusException → COMMON_ILLEGAL_STATUS
```

### TokenGenerator

```java
public class TokenGenerator {
    private static final int TOKEN_LENGTH = 20;

    public static String randomCharacterWithPrefix(String prefix) {
        return prefix + RandomStringUtils.randomAlphabetic(TOKEN_LENGTH - prefix.length());
    }
}
```

### AbstractEntity (JPA Auditing)

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class AbstractEntity {
    @CreationTimestamp
    private ZonedDateTime createdAt;

    @UpdateTimestamp
    private ZonedDateTime updatedAt;
}
```

---

## Flyway 마이그레이션

파일 위치: `src/main/resources/db/migration/V{번호}__{설명}.sql`

```sql
-- V1__init_ddl.sql
CREATE TABLE orders (
    id          BIGINT NOT NULL AUTO_INCREMENT,
    order_token VARCHAR(50) NOT NULL,
    user_id     BIGINT NOT NULL,
    pay_method  VARCHAR(50),
    total_amount BIGINT,
    status      VARCHAR(30) NOT NULL,
    ordered_at  DATETIME(6),
    -- delivery
    receiver_name     VARCHAR(100),
    receiver_phone    VARCHAR(20),
    receiver_zipcode  VARCHAR(10),
    receiver_address1 VARCHAR(200),
    receiver_address2 VARCHAR(200),
    etc_message       VARCHAR(500),
    -- audit
    created_at DATETIME(6) NOT NULL,
    updated_at DATETIME(6),
    PRIMARY KEY (id)
) ENGINE=InnoDB;

CREATE INDEX idx_orders_order_token ON orders (order_token);
CREATE INDEX idx_orders_user_id ON orders (user_id);
```

**네이밍 규칙:**
- 테이블명: 복수형 snake_case (`orders`, `order_items`)
- 컬럼명: snake_case (`order_token`, `user_id`)
- 인덱스: `idx_{테이블}_{컬럼}` (`idx_orders_order_token`)

---

## 계층 간 의존성 규칙 (필수)

```
interfaces  → application, domain (Command/Info만)
application → domain (Service 인터페이스만)
domain      → 없음 (독립)
infrastructure → domain (인터페이스 구현)
```

- **금지**: Controller → Service 직접 호출 (Facade 경유)
- **금지**: Service → Repository 직접 호출 (Store/Reader 경유)
- **금지**: Domain → Infrastructure 의존 (의존성 역전)
- **금지**: Entity를 계층 밖으로 노출 (Command/Info/DTO로 변환)

상세 패턴은 `${CLAUDE_SKILL_DIR}/reference.md`를 참조한다.
