# Spring Design 레퍼런스

## 도메인 모델링 패턴

### Root Entity 패턴
- 도메인의 진입점이자 Aggregate Root
- 비즈니스 식별자(Token) 생성: `TokenGenerator.randomCharacterWithPrefix("ord_")`
- 생성자에서 필수 필드 null/empty 검증 → `InvalidParamException`
- 상태 변경은 반드시 비즈니스 메서드를 통해 (setter 사용 금지)

### Token Prefix 규칙
| 도메인 | Prefix | 예시 |
|--------|--------|------|
| Order | ord_ | ord_UeiDIbGeRBRlwUWH |
| Item | itm_ | itm_fOeGqZhNhjmnOAXm |
| Partner | ptn_ | ptn_nzWJzditognwAinq |
| Payment | pay_ | pay_xKjRmWnYqLpVzBhT |
| Delivery | dlv_ | dlv_mNoPqRsTuVwXyZaB |
| User | usr_ | usr_aBcDeFgHiJkLmNoP |
| Product | prd_ | prd_qRsTuVwXyZaBcDeF |

새 도메인 추가 시 3글자 prefix + 언더스코어 규칙을 따른다.

### Value Object (@Embeddable) 기준
다음 조건을 만족하면 Value Object로 분리:
- 2개 이상의 필드가 항상 함께 사용됨
- 독립적인 생명주기가 없음 (소유 Entity에 종속)
- 다른 Entity에서도 재사용 가능

예시: DeliveryFragment(receiverName, receiverPhone, receiverZipcode, receiverAddress1, receiverAddress2, etcMessage)

### 상태 머신 설계 원칙
1. **단방향**: 상태는 앞으로만 전이 (역방향 전이 불가)
2. **검증 우선**: 상태 변경 메서드 내에서 현재 상태 검증
3. **명시적 네이밍**: `orderComplete()`, `startDelivery()` 처럼 동사형

## API 설계 패턴

### URL 규칙
```
POST /api/v1/{도메인s}/init              # 등록
POST /api/v1/{도메인s}/{action}           # 상태 변경
GET  /api/v1/{도메인s}/{token}            # 단건 조회
GET  /api/v1/{도메인s}                    # 목록 조회
```

### CommonResponse 구조
```json
{
  "result": "SUCCESS | FAIL",
  "data": { ... },
  "message": null,
  "errorCode": null
}
```

- 비즈니스 예외: HTTP 200 + result: FAIL
- Validation 실패: HTTP 400 + result: FAIL
- 시스템 예외: HTTP 500 + result: FAIL

### 에러 코드 네이밍
```
COMMON_SYSTEM_ERROR          # 시스템 오류
COMMON_INVALID_PARAMETER     # 파라미터 검증 실패
COMMON_ENTITY_NOT_FOUND      # 엔티티 없음
COMMON_ILLEGAL_STATUS        # 상태 오류
{DOMAIN}_CUSTOM_ERROR        # 도메인 특화 오류
```

## ERD 설계 패턴

### 인덱스 전략
- 비즈니스 식별자(token): UNIQUE INDEX
- 외래 키: INDEX
- 자주 조회되는 조건: INDEX
- 복합 인덱스: 카디널리티 높은 컬럼 우선

### MSA 간 참조
- 같은 서비스: FK 사용 가능
- 다른 서비스: FK 없이 ID/Token 값만 저장 (조인 불가, API 호출로 조회)

## 설계 시 고려사항

### 트랜잭션 범위
- 하나의 도메인 서비스 내에서 트랜잭션 완결
- 여러 도메인을 넘나드는 트랜잭션 → 이벤트 기반 비동기 처리 고려
- Facade에서 여러 Service를 호출하되, 트랜잭션은 각 Service에서 관리

### 확장성 고려
- 전략 패턴 적용 가능한 영역 식별 (결제 수단, 알림 채널 등)
- Validator 체인 적용 가능한 검증 로직 식별
- Factory 패턴 적용 가능한 복잡한 객체 생성 식별
