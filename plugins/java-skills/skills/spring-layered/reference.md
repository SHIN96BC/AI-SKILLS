# Spring Layered 레퍼런스

## build.gradle 템플릿

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.0'
    id 'io.spring.dependency-management' version '1.1.5'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(17)
    }
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    // Spring
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // DB
    runtimeOnly 'com.mysql:mysql-connector-j'
    implementation 'org.flywaydb:flyway-core:9.10.0'

    // MapStruct
    implementation 'org.mapstruct:mapstruct:1.4.2.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.4.2.Final'

    // Utility
    implementation 'com.google.guava:guava:28.2-jre'
    implementation 'org.apache.commons:commons-lang3:3.9'

    // Lombok
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'

    // Test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

## application.yml 템플릿

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: none
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        default_batch_fetch_size: 100
  flyway:
    enabled: true
    locations: classpath:db/migration
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/{서비스명}?useSSL=false&autoReconnect=true&characterEncoding=UTF-8&serverTimezone=UTC
    username: {서비스명}-svc
    password: {서비스명}-pass
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5

server:
  port: 8080
  shutdown: graceful

spring.lifecycle:
  timeout-per-shutdown-phase: 20s
```

## docker-compose.yml 템플릿

```yaml
version: "3.7"

services:
  {서비스명}-db:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_DATABASE={서비스명}
      - MYSQL_ROOT_PASSWORD=root-pass
      - MYSQL_USER={서비스명}-svc
      - MYSQL_PASSWORD={서비스명}-pass
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --lower_case_table_names=1
    volumes:
      - ./mysql:/var/lib/mysql
```

## HTTP 테스트 파일 템플릿

IntelliJ IDEA HTTP Client 형식:

```http
### 등록
POST http://localhost:8080/api/v1/orders/init
Content-Type: application/json

{
  "userId": 12345,
  "payMethod": "CARD",
  "receiverName": "홍길동",
  "receiverPhone": "01012345678",
  "receiverZipcode": "12345",
  "receiverAddress1": "서울시 강남구",
  "receiverAddress2": "101호",
  "etcMessage": "부재시 문 앞",
  "orderItemList": [
    {
      "orderCount": 1,
      "itemToken": "itm_xxxxxxxxxxxxxxxx",
      "itemName": "상품A",
      "itemPrice": 10000
    }
  ]
}

### 조회
GET http://localhost:8080/api/v1/orders/ord_xxxxxxxxxxxxxxxx
Content-Type: application/json
```

## CommonHttpRequestInterceptor

```java
@Slf4j
public class CommonHttpRequestInterceptor implements HandlerInterceptor {
    public static final String HEADER_REQUEST_UUID_KEY = "x-request-id";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String requestEventId = request.getHeader(HEADER_REQUEST_UUID_KEY);
        if (StringUtils.isEmpty(requestEventId)) {
            requestEventId = UUID.randomUUID().toString();
        }
        MDC.put(HEADER_REQUEST_UUID_KEY, requestEventId);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        MDC.clear();
    }
}
```

## CommonControllerAdvice

```java
@Slf4j
@ControllerAdvice
public class CommonControllerAdvice {

    // 비즈니스 예외 → HTTP 200
    @ResponseBody
    @ResponseStatus(HttpStatus.OK)
    @ExceptionHandler(value = BaseException.class)
    public CommonResponse onBaseException(BaseException e) {
        String eventId = MDC.get(CommonHttpRequestInterceptor.HEADER_REQUEST_UUID_KEY);
        log.warn("[BaseException] eventId = {}, errorMsg = {}", eventId, e.getMessage());
        return CommonResponse.fail(e.getMessage(), e.getErrorCode().name());
    }

    // Validation 실패 → HTTP 400
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public CommonResponse methodArgumentNotValidException(MethodArgumentNotValidException e) {
        FieldError fe = e.getBindingResult().getFieldError();
        String message = "Request Error" + " " + fe.getField() + "=" + fe.getRejectedValue();
        return CommonResponse.fail(message, ErrorCode.COMMON_INVALID_PARAMETER.name());
    }

    // 그 외 → HTTP 500
    @ResponseBody
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(value = Exception.class)
    public CommonResponse onException(Exception e) {
        String eventId = MDC.get(CommonHttpRequestInterceptor.HEADER_REQUEST_UUID_KEY);
        log.error("[Exception] eventId = {}, errorMsg = {}", eventId, e.getMessage(), e);
        return CommonResponse.fail(ErrorCode.COMMON_SYSTEM_ERROR);
    }
}
```

## 네이밍 컨벤션

### 클래스명
| 타입 | 규칙 | 예시 |
|------|------|------|
| Entity | 도메인 단수 | `Order`, `Item` |
| DTO | Domain + Dto | `OrderDto` |
| Command | Domain + Command | `OrderCommand` |
| Info | Domain + Info | `OrderInfo` |
| Service | Domain + Service | `OrderService` |
| ServiceImpl | Domain + ServiceImpl | `OrderServiceImpl` |
| Facade | Domain + Facade | `OrderFacade` |
| Controller | Domain + ApiController | `OrderApiController` |
| Repository | Domain + Repository | `OrderRepository` |
| Store | Domain + Store / StoreImpl | `OrderStore` |
| Reader | Domain + Reader / ReaderImpl | `OrderReader` |
| DtoMapper | Domain + DtoMapper | `OrderDtoMapper` |
| InfoMapper | Domain + InfoMapper | `OrderInfoMapper` |
| Factory | {목적} + Factory | `OrderItemSeriesFactory` |
| Validator | {목적} + Validator | `PayAmountValidator` |
| Exception | {원인} + Exception | `InvalidParamException` |

### 메서드명
| 용도 | 규칙 | 예시 |
|------|------|------|
| 등록 | register{Domain} | `registerOrder()` |
| 조회 | retrieve{Domain}, get{Domain} | `retrieveOrder()` |
| 수정 | change{동작} | `changeOnSale()` |
| 삭제 | remove{Domain} | `removeItem()` |
| 저장 | store | `store(Order order)` |
| 검증 | validate | `validate()` |
| 계산 | calculate{대상} | `calculateTotalAmount()` |
| 상태확인 | is{상태} | `isAlreadyPaymentComplete()` |
| 변환 | of, toEntity | `of(request)`, `toEntity()` |
