# Spring DB 레퍼런스

## MySQL 8.0 기준 타입 레퍼런스

### 숫자
| 타입 | 범위 | 용도 |
|------|------|------|
| TINYINT | -128 ~ 127 | boolean, 작은 상태값 |
| INT | ~21억 | 일반 정수, 수량 |
| BIGINT | ~922경 | PK, 금액, 대용량 카운터 |
| DECIMAL(M,D) | 정밀 소수 | 외화 금액, 비율 |

### 문자열
| 타입 | 최대 | 용도 |
|------|------|------|
| VARCHAR(N) | 65,535 bytes | 가변 길이 문자열 |
| TEXT | 65,535 bytes | 긴 텍스트 (인덱스 불가) |
| MEDIUMTEXT | 16MB | 큰 본문 (게시글 등) |
| JSON | ~4GB | 비정형 데이터 |

### 날짜/시간
| 타입 | 정밀도 | 용도 |
|------|--------|------|
| DATETIME(6) | 마이크로초 | 일반 날짜시간 (타임존 없음) |
| TIMESTAMP(6) | 마이크로초 | 타임존 변환 필요 시 |
| DATE | 일 단위 | 날짜만 |

## application.yml Flyway 설정

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true         # 기존 DB에 Flyway 적용 시
    baseline-version: 0               # baseline 버전
    validate-on-migrate: true         # 마이그레이션 파일 변조 감지
    out-of-order: false               # 순서 보장 (운영 환경에서 true 금지)
    clean-disabled: true              # flyway clean 명령 비활성화 (운영 필수)
```

## 인덱스 상세 가이드

### B-Tree 인덱스 동작 원리

```
복합 인덱스: (status, ordered_at)

  status='INIT', ordered_at='2024-01-01'
  status='INIT', ordered_at='2024-01-02'
  status='INIT', ordered_at='2024-01-03'
  status='ORDER_COMPLETE', ordered_at='2024-01-01'
  status='ORDER_COMPLETE', ordered_at='2024-01-02'

→ WHERE status = 'INIT'                       ✅ 인덱스 사용
→ WHERE status = 'INIT' AND ordered_at > ?    ✅ 인덱스 사용
→ WHERE ordered_at > ?                        ❌ 첫 번째 컬럼 스킵 → 풀스캔
→ WHERE status IN ('INIT','ORDER_COMPLETE')   ✅ 인덱스 사용
```

### 커버링 인덱스

쿼리에 필요한 모든 컬럼이 인덱스에 포함되어 테이블 접근 없이 응답:

```sql
-- 인덱스: (user_id, status, order_token, ordered_at)
-- 아래 쿼리는 테이블 데이터 페이지 접근 없이 인덱스만으로 응답 (Extra: Using index)
SELECT order_token, ordered_at
FROM orders
WHERE user_id = 1 AND status = 'INIT';
```

### 인덱스 안티패턴

```sql
-- ❌ 함수 사용 → 인덱스 무효화
WHERE YEAR(created_at) = 2024
-- ✅ 범위로 변환
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- ❌ 타입 불일치 → 암묵적 변환으로 인덱스 무효화
WHERE order_token = 12345    -- order_token은 VARCHAR
-- ✅ 타입 일치
WHERE order_token = '12345'

-- ❌ LIKE 앞쪽 와일드카드 → 인덱스 무효화
WHERE item_name LIKE '%셔츠'
-- ✅ 뒤쪽 와일드카드는 인덱스 사용 가능
WHERE item_name LIKE '셔츠%'

-- ❌ OR 조건은 인덱스 최적화 어려움
WHERE user_id = 1 OR partner_id = 2
-- ✅ UNION으로 분리
SELECT * FROM orders WHERE user_id = 1
UNION ALL
SELECT * FROM orders WHERE partner_id = 2
```

## 정규화 vs 반정규화

### 정규화 (기본)

```sql
-- 정규화: 주문과 상품 분리
orders (id, order_token, user_id, ...)
order_items (id, order_id, item_id, order_count, ...)
items (id, item_token, item_name, item_price, ...)
```

### 반정규화 (성능 목적)

```sql
-- 반정규화: 주문상품에 상품 정보 복제
order_items (
    id, order_id, item_id,
    item_token,     -- items에서 복제
    item_name,      -- items에서 복제 (주문 시점 상품명 보존)
    item_price,     -- items에서 복제 (주문 시점 가격 보존)
    order_count
)
```

**반정규화 적용 기준:**
| 기준 | 정규화 유지 | 반정규화 |
|------|-----------|---------|
| 조인 빈도 | 가끔 | 매 조회마다 |
| 데이터 불변성 | 원본이 자주 변경 | 시점 데이터 보존 필요 |
| MSA | 같은 서비스 내 | 다른 서비스 데이터 |
| 일관성 | 강한 일관성 필요 | 최종적 일관성 허용 |

## 대용량 ALTER TABLE 전략

운영 중인 대용량 테이블에 ALTER TABLE은 락이 발생할 수 있다:

### 온라인 DDL (MySQL 8.0)

```sql
-- ALGORITHM=INPLACE: 테이블 복사 없이 변경 (락 최소화)
ALTER TABLE orders ADD COLUMN memo VARCHAR(500), ALGORITHM=INPLACE, LOCK=NONE;

-- 인덱스 추가도 온라인 가능
CREATE INDEX idx_orders_memo ON orders (memo) ALGORITHM=INPLACE LOCK=NONE;
```

### pt-online-schema-change (대안)

```bash
# Percona Toolkit으로 무중단 스키마 변경
pt-online-schema-change \
  --alter "ADD COLUMN memo VARCHAR(500)" \
  D=order_db,t=orders \
  --execute
```

## docker-compose 템플릿 (MySQL)

```yaml
version: "3.7"

services:
  {서비스명}-db:
    image: mysql:8.0
    ports:
      - "{포트}:3306"
    environment:
      - MYSQL_DATABASE={서비스명}
      - MYSQL_ROOT_PASSWORD=root-pass
      - MYSQL_USER={서비스명}-svc
      - MYSQL_PASSWORD={서비스명}-pass
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --lower_case_table_names=1
      - --max_connections=200
      - --innodb_buffer_pool_size=256M
    volumes:
      - ./{서비스명}-mysql:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
```

## MSA DB 포트 분리 예시

| 서비스 | DB | 포트 |
|--------|-----|------|
| order-service | order_db | 3306 |
| payment-service | payment_db | 3307 |
| product-service | product_db | 3308 |
| user-service | user_db | 3309 |
| partner-service | partner_db | 3310 |
