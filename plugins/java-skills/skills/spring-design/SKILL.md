---
name: spring-design
description: Spring Boot MSA 도메인 설계. 도메인 모델링, API 명세, ERD, 4계층 패키지 구조, 상태 머신을 설계한다.
argument-hint: [도메인명 또는 요구사항]
allowed-tools: Read Grep Glob Agent
effort: high
---

# Spring Boot MSA 도메인 설계 스킬

사용자 요구사항: `$ARGUMENTS`

## 역할

코드 작성 전 단계. Spring Boot MSA 서비스의 **도메인 모델, API, 패키지 구조**를 설계한다.
이 스킬의 산출물을 `spring-layered` 스킬에 전달하여 코드를 구현한다.

---

## 설계 프로세스

### 1단계: 요구사항 분석

사용자 요구사항에서 다음을 파악한다:
- **도메인 목적**: 이 서비스가 해결하는 비즈니스 문제
- **핵심 유스케이스**: 주요 기능 목록
- **외부 연동**: 다른 MSA 서비스, 외부 API, 메시지 브로커
- **데이터 특성**: 트래픽 규모, 정합성 요구 수준

### 2단계: 도메인 모델링

#### Entity 설계 원칙
- **Root Entity**: 도메인의 진입점. 비즈니스 식별자(Token) 보유
- **하위 Entity**: Root에 종속. `@ManyToOne`/`@OneToMany` 관계
- **Value Object**: `@Embeddable`로 독립 생명주기 없는 값 객체
- **상태 관리**: `Status` enum으로 상태 머신 정의

#### 도메인 모델 산출물 형식

```
[도메인명] Domain Model
═══════════════════════════════

Root Entity: Order
├── id: Long (PK, auto)
├── orderToken: String (비즈니스 식별자, ord_ prefix)
├── userId: Long
├── status: Status (INIT → ORDER_COMPLETE → DELIVERY_PREPARE → ...)
├── deliveryFragment: DeliveryFragment (@Embeddable)
│   ├── receiverName: String
│   ├── receiverPhone: String
│   └── receiverAddress: String
└── orderItems: List<OrderItem> (1:N, cascade)
    ├── orderCount: Integer
    ├── itemPrice: Long
    └── orderItemOptionGroups: List<OrderItemOptionGroup> (1:N, cascade)

비즈니스 메서드:
- calculateTotalAmount(): 주문 총액 계산
- orderComplete(): 상태 변경 + 검증
```

#### 상태 머신 산출물 형식

```
상태 다이어그램: Order.Status
═══════════════════════════════

INIT ──[결제완료]──→ ORDER_COMPLETE
  │                      │
  │                 [배송준비]
  │                      │
  │                      ▼
  │              DELIVERY_PREPARE
  │                      │
  │                 [배송시작]
  │                      ▼
  │               IN_DELIVERY
  │                      │
  │                 [배송완료]
  │                      ▼
  └──[주문취소]──→  DELIVERY_COMPLETE
```

### 3단계: ERD 설계

#### ERD 산출물 형식

```
ERD
═══════════════════════════════

[orders] 1 ──── N [order_items] 1 ──── N [order_item_option_groups] 1 ──── N [order_item_options]

Table: orders
┌──────────────────┬──────────────┬─────────┐
│ Column           │ Type         │ Const   │
├──────────────────┼──────────────┼─────────┤
│ id               │ BIGINT       │ PK, AI  │
│ order_token      │ VARCHAR(50)  │ UQ, NN  │
│ user_id          │ BIGINT       │ IDX, NN │
│ pay_method       │ VARCHAR(50)  │ NN      │
│ total_amount     │ BIGINT       │         │
│ status           │ VARCHAR(30)  │ NN      │
│ ordered_at       │ DATETIME(6)  │ IDX     │
│ created_at       │ DATETIME(6)  │ NN      │
│ updated_at       │ DATETIME(6)  │         │
└──────────────────┴──────────────┴─────────┘

인덱스:
- idx_orders_order_token (order_token)
- idx_orders_user_id (user_id)
- idx_orders_ordered_at (ordered_at)
```

#### DB 설계 규칙
- 모든 테이블: `id` (PK, AUTO_INCREMENT), `created_at`, `updated_at` 필수
- 비즈니스 식별자: `{도메인}_token` 컬럼 + UNIQUE INDEX
- FK는 같은 서비스 내에서만 사용. 다른 MSA 서비스의 ID는 FK 없이 값만 저장
- 컬럼명: snake_case
- 테이블명: 복수형 (orders, items, partners)

### 4단계: API 설계

#### API 산출물 형식

```
API 명세
═══════════════════════════════

[POST] /api/v1/orders/init          - 주문 등록
[POST] /api/v1/orders/payment-order - 결제 처리
[GET]  /api/v1/orders/{orderToken}  - 주문 조회

───────────────────────────────
POST /api/v1/orders/init
───────────────────────────────
Request:
{
  "userId": Long (필수),
  "payMethod": String (필수, enum: CARD|NAVER_PAY|KAKAO_PAY|TOSS_PAY),
  "receiverName": String (필수),
  "orderItemList": [
    {
      "orderCount": Integer (필수, min: 1),
      "itemToken": String (필수)
    }
  ]
}

Response (성공):
{
  "result": "SUCCESS",
  "data": {
    "orderToken": "ord_xxxxxxxxxxxxxxxx"
  },
  "message": null,
  "errorCode": null
}

Response (실패):
{
  "result": "FAIL",
  "data": null,
  "message": "요청한 값이 올바르지 않습니다",
  "errorCode": "COMMON_INVALID_PARAMETER"
}
```

#### API 설계 규칙
- URL: `/api/v1/{도메인복수형}/{action}` (kebab-case)
- 응답: CommonResponse 통일 (`result`, `data`, `message`, `errorCode`)
- 등록: POST, 반환값은 token
- 조회: GET, Path Variable은 token 사용 (id 노출 금지)
- 상태변경: POST (PATCH 대신)
- Validation: `@Valid` + Bean Validation

### 5단계: 패키지 구조 설계

#### 4계층 레이어드 구조

```
com.example.{서비스명}/
├── interfaces/                  # 외부 입출력
│   └── {도메인}/
│       ├── {Domain}ApiController.java
│       ├── {Domain}Dto.java     # Request/Response DTO (inner class)
│       └── {Domain}DtoMapper.java  # MapStruct (DTO ↔ Command)
├── application/                 # 유스케이스 조율
│   └── {도메인}/
│       └── {Domain}Facade.java  # Service 조합 + 외부 서비스 호출
├── domain/                      # 비즈니스 핵심
│   ├── AbstractEntity.java      # created_at, updated_at
│   └── {도메인}/
│       ├── {Domain}.java        # @Entity (Root)
│       ├── {Domain}Command.java # 입력 명령 (inner class)
│       ├── {Domain}Info.java    # 출력 정보 (inner class)
│       ├── {Domain}Service.java # 인터페이스
│       ├── {Domain}ServiceImpl.java
│       ├── {Domain}Reader.java  # 조회 인터페이스
│       ├── {Domain}Store.java   # 저장 인터페이스
│       ├── {Domain}InfoMapper.java # MapStruct (Entity → Info)
│       └── {하위도메인}/         # 하위 Entity, 옵션 등
├── infrastructure/              # 기술 구현
│   └── {도메인}/
│       ├── {Domain}Repository.java    # JpaRepository
│       ├── {Domain}ReaderImpl.java
│       ├── {Domain}StoreImpl.java
│       └── {외부연동}/               # API Caller 등
├── common/                      # 횡단 관심사
│   ├── exception/               # BaseException, ErrorCode
│   ├── response/                # CommonResponse, ControllerAdvice
│   ├── interceptor/             # RequestId MDC
│   └── util/                    # TokenGenerator
└── config/                      # Spring 설정
```

#### 계층 간 데이터 흐름

```
Controller → (DTO) → DtoMapper → (Command) → Facade → Service → Entity
                                                                   ↓
Controller ← (DTO) ← DtoMapper ← (Info) ← InfoMapper ← ─ ─ ─ Entity
```

- **DTO**: interfaces 계층 전용. 외부 요청/응답
- **Command**: domain 계층 입력. Facade/Service가 사용
- **Info**: domain 계층 출력. Entity → Info 변환 후 반환
- **Entity**: domain 내부에서만 직접 접근

### 6단계: 검증 항목

#### 설계 체크리스트
- [ ] 모든 Entity에 비즈니스 식별자(Token) 있는가
- [ ] 상태 전이가 명확하고 역방향 전이가 불가능한가
- [ ] 다른 MSA 서비스의 데이터에 FK를 걸지 않았는가
- [ ] API URL이 RESTful하고 일관성 있는가
- [ ] 계층 간 데이터 객체가 분리되어 있는가 (DTO/Command/Info/Entity)
- [ ] Value Object로 분리할 수 있는 필드 그룹이 있는가
- [ ] 비즈니스 로직이 Entity 메서드에 포함되어 있는가 (빈약한 도메인 모델 방지)

---

## 산출물 요약

설계 완료 시 다음 산출물을 제공한다:

1. **도메인 모델** — Entity 계층 구조, 필드, 관계, 비즈니스 메서드
2. **상태 머신** — 각 Entity의 상태 전이도
3. **ERD** — 테이블 구조, 인덱스, 제약조건
4. **API 명세** — 엔드포인트, Request/Response, 상태코드
5. **패키지 구조** — 4계층 디렉토리 + 클래스 목록
6. **설계 체크리스트** — 검증 결과

이 산출물을 `/java-skills:spring-layered` 스킬에 전달하여 구현한다.
