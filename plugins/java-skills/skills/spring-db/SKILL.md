---
name: spring-db
description: MSA 기반 DB 설계. 서비스별 독립 스키마, 테이블/인덱스 설계, Flyway 마이그레이션, 데이터 동기화, 성능 최적화, 감사/이력 패턴.
argument-hint: [도메인명 또는 요구사항]
allowed-tools: Read Grep Glob Write Edit Bash
effort: high
---

# MSA 기반 DB 설계 스킬

사용자 요구사항: `$ARGUMENTS`

## 역할

MSA 환경에서 **서비스별 독립 DB 스키마**를 설계하고, Flyway 마이그레이션 파일을 작성한다.
`spring-design`의 도메인 모델을 기반으로 물리적 DB 구조를 최적화한다.

---

## MSA DB 설계 원칙

### Database per Service

각 마이크로서비스는 **자체 데이터베이스**를 소유한다. 다른 서비스의 DB에 직접 접근하지 않는다.

```
[주문 서비스]          [결제 서비스]          [상품 서비스]
     │                     │                     │
 order_db              payment_db             product_db
 ┌──────────┐         ┌──────────┐          ┌──────────┐
 │ orders   │         │ payments │          │ items    │
 │ order_   │         │ payment_ │          │ item_    │
 │  items   │         │  logs    │          │  options │
 └──────────┘         └──────────┘          └──────────┘
```

### 서비스 간 데이터 참조 규칙

```
같은 서비스 내:
  orders.id ──FK──→ order_items.order_id   ✅ FK 사용

다른 서비스 참조:
  order_items.item_id = 123                ✅ 값만 저장 (FK 없음)
  order_items.item_token = "itm_xxx"       ✅ 토큰으로 참조
  order_items.item_id ──FK──→ items.id     ❌ 크로스 서비스 FK 금지
```

**이유:**
- 서비스 독립 배포/스케일링 보장
- 다른 서비스 DB 장애가 전파되지 않음
- 서비스별 DB 기술 선택 자유 (MySQL, PostgreSQL, MongoDB 등)

### 크로스 서비스 데이터 조회 방법

| 방법 | 설명 | 사용 시점 |
|------|------|----------|
| API 호출 | REST/gRPC로 다른 서비스에 요청 | 실시간 최신 데이터 필요 |
| 이벤트 동기화 | 이벤트로 데이터 복제본 유지 | 조회 빈도 높고 약간의 지연 허용 |
| CQRS 읽기 모델 | 여러 서비스 데이터를 조합한 읽기 전용 DB | 복잡한 조회/리포팅 |

---

## 설계 프로세스

### 1단계: 서비스 경계 확인

```
서비스별 소유 도메인 매핑
═══════════════════════════════

[order-service]
  소유: orders, order_items, order_item_option_groups, order_item_options
  참조: user_id (user-service), item_id/item_token (product-service), partner_id (partner-service)

[product-service]
  소유: items, item_option_groups, item_options
  참조: partner_id (partner-service)

[partner-service]
  소유: partners
  참조: 없음

[payment-service]
  소유: payments, payment_logs
  참조: order_token (order-service), user_id (user-service)
```

### 2단계: 테이블 설계

#### 공통 컬럼 규칙

모든 테이블에 다음 컬럼을 포함한다:

```sql
-- 필수 공통 컬럼
id          BIGINT NOT NULL AUTO_INCREMENT,  -- PK
created_at  DATETIME(6) NOT NULL,            -- 생성 시각
updated_at  DATETIME(6),                     -- 수정 시각

PRIMARY KEY (id)
```

#### 비즈니스 식별자 (Token)

외부에 노출되는 식별자는 auto-increment ID 대신 **Token**을 사용한다.

```sql
-- 비즈니스 식별자
order_token VARCHAR(50) NOT NULL,  -- ord_ + 16자 랜덤

-- ID는 내부 FK 용도, Token은 외부 API 용도
CREATE UNIQUE INDEX uidx_orders_order_token ON orders (order_token);
```

**이유:**
- ID 노출 시 전체 건수 추측 가능 (보안)
- ID는 DB 종속적, Token은 서비스 간 이식 가능
- URL에 Token 사용: `/api/v1/orders/ord_xxx` (ID 대신)

#### 컬럼 타입 선정 기준

| 데이터 | 타입 | 이유 |
|--------|------|------|
| PK | `BIGINT` | INT 범위(21억) 초과 대비 |
| 금액 | `BIGINT` | 소수점 없는 원화 기준. 외화는 `DECIMAL(19,4)` |
| 상태 | `VARCHAR(30)` | enum 문자열 저장 (가독성, 확장성) |
| 날짜 | `DATETIME(6)` | 마이크로초 정밀도, 타임존은 앱에서 관리 |
| 토큰 | `VARCHAR(50)` | prefix(4자) + 랜덤(16자) + 여유분 |
| 이름/제목 | `VARCHAR(100~200)` | 용도에 맞게 조절 |
| 설명/메모 | `VARCHAR(500)` 또는 `TEXT` | 500자 이내면 VARCHAR, 초과면 TEXT |
| Boolean | `TINYINT(1)` | MySQL에서 BOOLEAN은 TINYINT alias |
| JSON 데이터 | `JSON` | 비정형 데이터, 인덱스 필요시 가상 컬럼 |

#### 제약조건

```sql
-- NOT NULL: 필수 필드
order_token VARCHAR(50) NOT NULL,

-- UNIQUE: 비즈니스 유일 식별자
CREATE UNIQUE INDEX uidx_orders_order_token ON orders (order_token);

-- DEFAULT: 기본값
status VARCHAR(30) NOT NULL DEFAULT 'INIT',
retry_count INT NOT NULL DEFAULT 0,

-- CHECK (MySQL 8.0.16+): 값 범위 제한
CONSTRAINT chk_order_items_order_count CHECK (order_count > 0),
CONSTRAINT chk_order_items_item_price CHECK (item_price >= 0),
```

### 3단계: 인덱스 설계

#### 인덱스 전략

```sql
-- 1. 비즈니스 식별자 → UNIQUE INDEX
CREATE UNIQUE INDEX uidx_orders_order_token ON orders (order_token);

-- 2. FK 참조 컬럼 → INDEX
CREATE INDEX idx_order_items_order_id ON order_items (order_id);

-- 3. 자주 조회되는 조건 → INDEX
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_ordered_at ON orders (ordered_at);

-- 4. 복합 인덱스 (카디널리티 높은 컬럼 우선)
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
CREATE INDEX idx_orders_status_ordered ON orders (status, ordered_at);

-- 5. 커버링 인덱스 (SELECT 대상 컬럼까지 포함)
CREATE INDEX idx_orders_covering ON orders (user_id, status, order_token, ordered_at);
```

#### 인덱스 판단 기준

| 기준 | 인덱스 추가 | 인덱스 불필요 |
|------|------------|-------------|
| WHERE 절 빈도 | 자주 조회됨 | 거의 조회 안됨 |
| 카디널리티 | 높음 (user_id) | 낮음 (gender) |
| 테이블 크기 | 10만 건 이상 | 수천 건 이하 |
| 쓰기 비율 | 읽기 > 쓰기 | 쓰기 >> 읽기 |
| 정렬 사용 | ORDER BY에 사용됨 | 정렬 없음 |

#### 인덱스 네이밍

```
idx_{테이블}_{컬럼}              -- 일반 인덱스
uidx_{테이블}_{컬럼}             -- UNIQUE 인덱스
idx_{테이블}_{컬럼1}_{컬럼2}     -- 복합 인덱스
```

### 4단계: 관계 설계

#### 1:N 관계

```sql
-- 부모 테이블
CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT,
    order_token VARCHAR(50) NOT NULL,
    PRIMARY KEY (id)
);

-- 자식 테이블
CREATE TABLE order_items (
    id BIGINT NOT NULL AUTO_INCREMENT,
    order_id BIGINT NOT NULL,           -- FK
    PRIMARY KEY (id),
    CONSTRAINT fk_order_items_order
        FOREIGN KEY (order_id) REFERENCES orders (id)
);

CREATE INDEX idx_order_items_order_id ON order_items (order_id);
```

#### N:M 관계 (중간 테이블)

```sql
-- 중간 테이블로 풀어서 설계
CREATE TABLE item_categories (
    id BIGINT NOT NULL AUTO_INCREMENT,
    item_id BIGINT NOT NULL,
    category_id BIGINT NOT NULL,
    created_at DATETIME(6) NOT NULL,
    PRIMARY KEY (id),
    CONSTRAINT fk_item_categories_item FOREIGN KEY (item_id) REFERENCES items (id),
    CONSTRAINT fk_item_categories_category FOREIGN KEY (category_id) REFERENCES categories (id)
);

CREATE UNIQUE INDEX uidx_item_categories ON item_categories (item_id, category_id);
```

#### 자기 참조 (계층 구조)

```sql
-- 카테고리 트리
CREATE TABLE categories (
    id BIGINT NOT NULL AUTO_INCREMENT,
    parent_id BIGINT,                   -- 자기 참조 (NULL = 루트)
    name VARCHAR(100) NOT NULL,
    depth INT NOT NULL DEFAULT 0,
    sort_order INT NOT NULL DEFAULT 0,
    PRIMARY KEY (id),
    CONSTRAINT fk_categories_parent FOREIGN KEY (parent_id) REFERENCES categories (id)
);

CREATE INDEX idx_categories_parent_id ON categories (parent_id);
```

### 5단계: MSA 특화 패턴

#### 데이터 복제 테이블 (로컬 캐시)

다른 서비스의 자주 참조되는 데이터를 이벤트로 복제한다.

```sql
-- order-service에 product-service의 상품 데이터 복제
CREATE TABLE item_snapshots (
    id BIGINT NOT NULL AUTO_INCREMENT,
    item_id BIGINT NOT NULL,            -- product-service의 item.id
    item_token VARCHAR(50) NOT NULL,
    item_name VARCHAR(200) NOT NULL,
    item_price BIGINT NOT NULL,
    partner_id BIGINT NOT NULL,
    synced_at DATETIME(6) NOT NULL,     -- 마지막 동기화 시각
    created_at DATETIME(6) NOT NULL,
    updated_at DATETIME(6),
    PRIMARY KEY (id)
);

CREATE UNIQUE INDEX uidx_item_snapshots_item_id ON item_snapshots (item_id);
CREATE INDEX idx_item_snapshots_item_token ON item_snapshots (item_token);
```

**규칙:**
- `_snapshots` 접미사로 복제 테이블 구분
- `synced_at` 컬럼으로 데이터 신선도 추적
- 원본 서비스의 변경 이벤트 수신 시 갱신

#### Outbox 테이블

이벤트 유실 방지를 위한 Outbox 패턴 테이블.

```sql
CREATE TABLE outbox_events (
    id BIGINT NOT NULL AUTO_INCREMENT,
    event_id VARCHAR(50) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(50) NOT NULL,
    payload JSON NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',  -- PENDING, PUBLISHED, FAILED
    retry_count INT NOT NULL DEFAULT 0,
    max_retries INT NOT NULL DEFAULT 5,
    created_at DATETIME(6) NOT NULL,
    published_at DATETIME(6),
    failed_at DATETIME(6),
    error_message VARCHAR(500),
    PRIMARY KEY (id)
);

CREATE UNIQUE INDEX uidx_outbox_event_id ON outbox_events (event_id);
CREATE INDEX idx_outbox_status_created ON outbox_events (status, created_at);
```

#### 멱등성 테이블

이벤트 중복 처리 방지.

```sql
CREATE TABLE processed_events (
    id BIGINT NOT NULL AUTO_INCREMENT,
    event_id VARCHAR(50) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    processed_at DATETIME(6) NOT NULL,
    PRIMARY KEY (id)
);

CREATE UNIQUE INDEX uidx_processed_events_event_id ON processed_events (event_id);
```

#### Saga 상태 테이블

분산 트랜잭션 상태 추적.

```sql
CREATE TABLE saga_instances (
    id BIGINT NOT NULL AUTO_INCREMENT,
    saga_id VARCHAR(50) NOT NULL,
    saga_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(50) NOT NULL,
    current_step VARCHAR(100) NOT NULL,
    status VARCHAR(20) NOT NULL,         -- STARTED, COMPENSATING, COMPLETED, FAILED
    payload JSON,
    created_at DATETIME(6) NOT NULL,
    updated_at DATETIME(6),
    PRIMARY KEY (id)
);

CREATE UNIQUE INDEX uidx_saga_saga_id ON saga_instances (saga_id);
CREATE INDEX idx_saga_status ON saga_instances (status);
```

---

## Flyway 마이그레이션

### 파일 네이밍 규칙

```
src/main/resources/db/migration/
├── V1__init_ddl.sql                    # 초기 테이블 생성
├── V2__add_payment_tables.sql          # 결제 테이블 추가
├── V3__add_index_orders_status.sql     # 인덱스 추가
├── V4__alter_orders_add_column.sql     # 컬럼 추가
├── V5__insert_initial_data.sql         # 초기 데이터 (DML)
└── V6__add_outbox_table.sql            # Outbox 테이블 추가
```

**네이밍 패턴:**
- `V{번호}__init_{영역}_ddl.sql` — 초기 생성
- `V{번호}__add_{테이블/컬럼}.sql` — 추가
- `V{번호}__alter_{테이블}_{변경내용}.sql` — 수정
- `V{번호}__drop_{테이블/컬럼}.sql` — 삭제
- `V{번호}__insert_{데이터설명}.sql` — 초기 데이터
- `V{번호}__add_index_{테이블}_{컬럼}.sql` — 인덱스 추가

### 마이그레이션 작성 규칙

```sql
-- ✅ 각 DDL 문 끝에 세미콜론
CREATE TABLE orders ( ... );

-- ✅ IF NOT EXISTS / IF EXISTS 사용 (멱등성)
CREATE TABLE IF NOT EXISTS orders ( ... );
DROP TABLE IF EXISTS temp_orders;

-- ✅ 큰 테이블 ALTER는 별도 마이그레이션으로 분리
-- V3__alter_orders_add_memo.sql
ALTER TABLE orders ADD COLUMN memo VARCHAR(500) AFTER etc_message;

-- ✅ 인덱스 추가도 별도 마이그레이션
-- V4__add_index_orders_memo.sql
CREATE INDEX idx_orders_memo ON orders (memo);

-- ❌ 하나의 마이그레이션에 DDL + DML 혼합 금지
-- ❌ 데이터 마이그레이션에서 DELETE/TRUNCATE 사용 금지 (백업 후 진행)
```

### 롤백 전략

Flyway는 공식적으로 롤백을 지원하지 않으므로, 역방향 마이그레이션을 수동으로 작성한다:

```sql
-- V5__add_column_orders_memo.sql (정방향)
ALTER TABLE orders ADD COLUMN memo VARCHAR(500);

-- V6__rollback_v5_remove_memo.sql (역방향 - 필요 시)
ALTER TABLE orders DROP COLUMN memo;
```

---

## 감사/이력 패턴

### Soft Delete

물리 삭제 대신 논리 삭제.

```sql
CREATE TABLE items (
    id BIGINT NOT NULL AUTO_INCREMENT,
    item_token VARCHAR(50) NOT NULL,
    item_name VARCHAR(200) NOT NULL,
    status VARCHAR(30) NOT NULL,
    deleted_at DATETIME(6),              -- NULL = 활성, NOT NULL = 삭제됨
    created_at DATETIME(6) NOT NULL,
    updated_at DATETIME(6),
    PRIMARY KEY (id)
);

-- 활성 데이터만 조회하는 인덱스
CREATE INDEX idx_items_active ON items (status) WHERE deleted_at IS NULL;
-- MySQL은 부분 인덱스 미지원이므로 복합 인덱스로 대체
CREATE INDEX idx_items_status_deleted ON items (deleted_at, status);
```

**JPA 적용:**
```java
@SQLRestriction("deleted_at IS NULL")  // Hibernate 6.3+
@Entity
public class Item extends AbstractEntity {
    private ZonedDateTime deletedAt;

    public void softDelete() {
        this.deletedAt = ZonedDateTime.now();
    }
}
```

### 이력 테이블 (History/Audit Log)

변경 이력을 별도 테이블에 기록.

```sql
CREATE TABLE order_histories (
    id BIGINT NOT NULL AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    order_token VARCHAR(50) NOT NULL,
    previous_status VARCHAR(30),
    new_status VARCHAR(30) NOT NULL,
    changed_by VARCHAR(100),             -- 변경 주체 (user_id 또는 시스템)
    change_reason VARCHAR(500),
    snapshot JSON,                       -- 변경 시점의 전체 데이터 스냅샷
    created_at DATETIME(6) NOT NULL,
    PRIMARY KEY (id)
);

CREATE INDEX idx_order_histories_order_id ON order_histories (order_id);
CREATE INDEX idx_order_histories_created ON order_histories (created_at);
```

### 버전 관리 (Optimistic Locking)

동시 수정 충돌 방지.

```sql
CREATE TABLE items (
    id BIGINT NOT NULL AUTO_INCREMENT,
    item_token VARCHAR(50) NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,   -- 낙관적 락 버전
    -- ...
    PRIMARY KEY (id)
);
```

```java
@Entity
public class Item extends AbstractEntity {
    @Version
    private Long version;
}
```

---

## 성능 고려사항

### 대용량 테이블 설계

#### 파티셔닝 (데이터 분산)

```sql
-- RANGE 파티셔닝 (날짜 기준)
CREATE TABLE order_logs (
    id BIGINT NOT NULL AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    log_type VARCHAR(50) NOT NULL,
    message TEXT,
    created_at DATETIME(6) NOT NULL,
    PRIMARY KEY (id, created_at)         -- 파티션 키 포함 필수
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

**파티셔닝 적용 기준:**
- 테이블 크기 1000만 건 이상
- 날짜 기반 조회가 대부분
- 오래된 데이터 아카이빙/삭제 필요

#### 아카이빙 전략

```sql
-- 아카이브 테이블 (구조 동일)
CREATE TABLE orders_archive LIKE orders;

-- 오래된 데이터 이동 (배치)
INSERT INTO orders_archive SELECT * FROM orders WHERE ordered_at < '2024-01-01';
DELETE FROM orders WHERE ordered_at < '2024-01-01';
```

### 쿼리 최적화 힌트

설계 시 예상 쿼리를 기반으로 인덱스를 결정한다:

```sql
-- 예상 쿼리 1: 사용자별 주문 목록 (최신순)
SELECT * FROM orders WHERE user_id = ? ORDER BY ordered_at DESC LIMIT 20;
→ INDEX: (user_id, ordered_at DESC)

-- 예상 쿼리 2: 상태별 주문 수 집계
SELECT status, COUNT(*) FROM orders GROUP BY status;
→ INDEX: (status)

-- 예상 쿼리 3: 특정 기간 + 상태 조회
SELECT * FROM orders WHERE status = 'INIT' AND ordered_at BETWEEN ? AND ?;
→ INDEX: (status, ordered_at)

-- 예상 쿼리 4: 주문 + 주문상품 조인
SELECT o.*, oi.* FROM orders o JOIN order_items oi ON o.id = oi.order_id WHERE o.order_token = ?;
→ INDEX: orders(order_token), order_items(order_id)
```

---

## 산출물 형식

### 테이블 정의서

```
Table: orders
═══════════════════════════════════════════════════════════════
Column            │ Type          │ Null │ Default │ 설명
──────────────────┼───────────────┼──────┼─────────┼──────────
id                │ BIGINT        │ NO   │ AI      │ PK
order_token       │ VARCHAR(50)   │ NO   │         │ 비즈니스 식별자
user_id           │ BIGINT        │ NO   │         │ 주문자 (user-service)
pay_method        │ VARCHAR(50)   │ YES  │         │ 결제 수단
total_amount      │ BIGINT        │ YES  │         │ 주문 총액 (원)
status            │ VARCHAR(30)   │ NO   │ 'INIT'  │ 주문 상태
ordered_at        │ DATETIME(6)   │ YES  │         │ 주문 확정 시각
receiver_name     │ VARCHAR(100)  │ YES  │         │ 수령인명
receiver_phone    │ VARCHAR(20)   │ YES  │         │ 수령인 연락처
receiver_zipcode  │ VARCHAR(10)   │ YES  │         │ 우편번호
receiver_address1 │ VARCHAR(200)  │ YES  │         │ 주소1
receiver_address2 │ VARCHAR(200)  │ YES  │         │ 주소2 (상세)
etc_message       │ VARCHAR(500)  │ YES  │         │ 배송 메모
created_at        │ DATETIME(6)   │ NO   │         │ 생성 시각
updated_at        │ DATETIME(6)   │ YES  │         │ 수정 시각

Indexes:
  uidx_orders_order_token  │ UNIQUE │ (order_token)
  idx_orders_user_id       │        │ (user_id)
  idx_orders_status        │        │ (status)
  idx_orders_ordered_at    │        │ (ordered_at)

FK: 없음 (Root Entity)
서비스 간 참조: user_id → user-service
```

### 서비스 간 데이터 참조 맵

```
데이터 참조 맵
═══════════════════════════════

order-service
  └── orders.user_id ──────→ user-service (API 호출)
  └── order_items.item_id ──→ product-service (스냅샷 + API)
  └── order_items.partner_id → partner-service (스냅샷)

payment-service
  └── payments.order_token ──→ order-service (이벤트 수신)
  └── payments.user_id ──────→ user-service (API 호출)

데이터 동기화:
  product-service ──[ItemUpdatedEvent]──→ order-service (item_snapshots 갱신)
  partner-service ──[PartnerUpdatedEvent]──→ order-service (partner_snapshots 갱신)
```

---

## 설계 체크리스트

### 테이블 설계
- [ ] 모든 테이블에 `id`, `created_at`, `updated_at` 있는가
- [ ] 비즈니스 식별자(Token)에 UNIQUE INDEX 있는가
- [ ] FK 컬럼에 INDEX 있는가
- [ ] 컬럼 타입이 데이터 특성에 맞는가 (금액=BIGINT, 상태=VARCHAR 등)
- [ ] NOT NULL 제약이 적절한가

### MSA 분리
- [ ] 크로스 서비스 FK가 없는가
- [ ] 다른 서비스 데이터 참조 방식이 정의되었는가 (API/이벤트/스냅샷)
- [ ] 데이터 복제 테이블(_snapshots)이 필요한 곳에 있는가
- [ ] Outbox 테이블이 이벤트 발행 서비스에 있는가

### 인덱스
- [ ] 예상 쿼리 기반으로 인덱스가 설계되었는가
- [ ] 복합 인덱스의 컬럼 순서가 카디널리티 순인가
- [ ] 불필요한 인덱스(쓰기 부하)가 없는가

### 마이그레이션
- [ ] Flyway 파일 네이밍이 규칙을 따르는가
- [ ] DDL과 DML이 분리되어 있는가
- [ ] 대용량 테이블 ALTER에 대한 영향도를 검토했는가

상세 패턴은 `${CLAUDE_SKILL_DIR}/reference.md`를 참조한다.
