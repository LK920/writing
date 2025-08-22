# API 설계 및 컨트롤러 구현 가이드

## 📋 목차
1. RESTful API 설계 원칙
2. API 엔드포인트 설계
3. 컨트롤러 구현 전략
4. Request/Response DTO 설계
5. Validation 전략

---

## RESTful API 설계 원칙

### 🎯 기본 원칙
1. **리소스 기반 URL**: 행위가 아닌 리소스 중심
2. **HTTP 메서드 활용**: GET, POST, PUT, DELETE 적절히 사용
3. **상태 코드 활용**: 의미에 맞는 HTTP 상태 코드 반환
4. **일관된 응답 형식**: 표준화된 응답 구조

### 📝 URL 명명 규칙
```
✅ 좋은 예시
GET  /api/balance/{userId}     # 잔액 조회
POST /api/balance/charge       # 잔액 충전
GET  /api/product/list         # 상품 목록
POST /api/order                # 주문 생성

❌ 나쁜 예시
GET  /api/getBalance           # 동사 사용
POST /api/chargeUserBalance    # 길고 복잡한 이름
GET  /api/products-list        # 일관성 없는 구분자
```

---

## API 엔드포인트 설계

### 💰 잔액 관리 API
```http
# 잔액 충전
POST /api/balance/charge
Content-Type: application/json
{
  "userId": 1,
  "amount": 10000
}

# 잔액 조회
GET /api/balance/{userId}
```

### 📦 상품 관리 API
```http
# 상품 목록 조회 (페이징)
GET /api/product/list?limit=10&offset=0

# 인기 상품 조회
GET /api/product/popular
```

### 🛒 주문 관리 API
```http
# 주문 생성
POST /api/order
Content-Type: application/json
{
  "userId": 1,
  "productIds": [1, 2, 3],
  "couponIds": [1]
}

# 주문 결제
POST /api/order/{orderId}/pay
```

### 🎫 쿠폰 관리 API
```http
# 쿠폰 발급
POST /api/coupon/issue?userId=1&couponId=1

# 보유 쿠폰 조회 (페이징)
GET /api/coupon/{userId}?limit=10&offset=0
```

---

## 컨트롤러 구현 전략

### 🏗️ 진화 과정

#### 1단계: 기본 컨트롤러 구조
```java
@RestController
@RequestMapping("/api/balance")
public class BalanceController {

    @PostMapping("/charge")
    public void chargeBalance(@RequestParam Long userId, 
                            @RequestParam BigDecimal amount) {
        // TODO: 구현
    }

    @GetMapping("/{userId}")
    public BalanceResponse getBalance(@PathVariable Long userId) {
        // TODO: 구현
        return null; // ❌ 초기에 null 반환
    }
}
```

#### 2단계: RequestDTO 도입
```java
@PostMapping("/charge")
public void chargeBalance(@Valid @RequestBody BalanceChargeRequest request) {
    // request.getUserId(), request.getAmount() 사용
}
```

#### 3단계: 모의 데이터 반환
```java
@GetMapping("/{userId}")
public BalanceResponse getBalance(@PathVariable Long userId) {
    return new BalanceResponse(userId, 
        new BigDecimal("50000"), 
        LocalDateTime.now());
}
```

### 📋 최종 컨트롤러 구조
```java
/**
 * 잔액 관리 Controller
 * 사용자 잔액 충전 및 조회 기능을 제공합니다.
 */
@Tag(name = "잔액 관리", description = "사용자 잔액 충전 및 조회 API")
@RestController
@RequestMapping("/api/balance")
public class BalanceController {

    @ApiSuccess(summary = "잔액 충전", description = "사용자의 잔액을 충전합니다.")
    @PostMapping("/charge")
    @ResponseStatus(HttpStatus.OK)
    public void chargeBalance(@Valid @RequestBody BalanceChargeRequest request) {
        // TODO: 잔액 충전 로직 구현
    }

    @ApiSuccess(summary = "잔액 조회", description = "사용자의 현재 잔액을 조회합니다.")
    @GetMapping("/{userId}")
    public BalanceResponse getBalance(@PathVariable Long userId) {
        // TODO: 잔액 조회 로직 구현
        return new BalanceResponse(userId, new BigDecimal("50000"), LocalDateTime.now());
    }
}
```

---

## Request/Response DTO 설계

### 📥 Request DTO 설계 원칙

#### 1. @Schema로 문서화
```java
@Schema(description = "잔액 충전 요청")
public class BalanceChargeRequest {
    
    @Schema(description = "사용자 ID", example = "1", required = true)
    @NotNull(message = "사용자 ID는 필수입니다")
    private Long userId;
    
    @Schema(description = "충전 금액", example = "10000", required = true)
    @NotNull(message = "충전 금액은 필수입니다")
    @DecimalMin(value = "0.0", inclusive = false, message = "충전 금액은 0보다 커야 합니다")
    private BigDecimal amount;

    // 생성자, Getter, Setter
}
```

#### 2. Validation 통합
- Bean Validation 어노테이션 사용
- 메시지는 한국어로 작성
- 비즈니스 규칙 반영

### 📤 Response DTO 설계 원칙

#### 1. Record 사용 (불변성)
```java
@Schema(description = "잔액 조회 응답")
public record BalanceResponse(
    @Schema(description = "사용자 ID", example = "1")
    Long userId,
    @Schema(description = "현재 잔액", example = "50000")
    BigDecimal amount,
    @Schema(description = "마지막 업데이트 시간", example = "2024-01-01T12:00:00")
    LocalDateTime updatedAt
) {}
```

#### 2. 중첩 구조 지원
```java
public record OrderResponse(
    Long orderId,
    Long userId,
    String status,
    BigDecimal totalAmount,
    LocalDateTime createdAt,
    List<OrderItemResponse> items  // 중첩된 구조
) {
    public record OrderItemResponse(
        Long productId,
        String name,
        int quantity,
        BigDecimal price
    ) {}
}
```

---

## Validation 전략

### 🛡️ 두 가지 접근 방식

#### 방식 1: @RequestParam + @Validated (초기)
```java
@RestController
@Validated  // 클래스 레벨에 필요
public class BalanceController {
    
    @PostMapping("/charge")
    public void chargeBalance(
        @NotNull(message = "사용자 ID는 필수입니다") 
        @RequestParam Long userId,
        @NotNull(message = "충전 금액은 필수입니다") 
        @DecimalMin(value = "0.0", inclusive = false) 
        @RequestParam BigDecimal amount) {
        // 구현
    }
}
```

#### 방식 2: @RequestBody + @Valid (최종)
```java
@RestController
public class BalanceController {
    
    @PostMapping("/charge")
    public void chargeBalance(@Valid @RequestBody BalanceChargeRequest request) {
        // 구현 - Validation은 DTO 내부에서 처리
    }
}
```

### 📋 Validation 어노테이션 활용
```java
// 필수 값 검증
@NotNull(message = "사용자 ID는 필수입니다")
private Long userId;

// 빈 컬렉션 검증
@NotEmpty(message = "상품 목록은 필수입니다")
private List<Long> productIds;

// 숫자 범위 검증
@DecimalMin(value = "0.0", inclusive = false, message = "충전 금액은 0보다 커야 합니다")
private BigDecimal amount;

// 중첩 객체 검증
@Valid
private List<OrderItemRequest> items;
```

---

## 📚 핵심 학습 포인트

### 1. 발전 과정
1. **기본 구조** → RequestParam 방식
2. **DTO 도입** → RequestBody 방식  
3. **Validation 통합** → Bean Validation
4. **문서화** → Swagger 통합

### 2. 설계 원칙
- **일관성**: 모든 API가 동일한 패턴
- **검증**: 요청 데이터의 무결성 보장
- **문서화**: 자동 생성되는 API 문서
- **타입 안정성**: 컴파일 타임 타입 체크

### 3. 실무 팁
- **null 반환 금지**: 항상 적절한 모의 데이터 반환
- **HTTP 상태 코드**: 의미에 맞는 상태 코드 사용
- **페이징 지원**: limit, offset 파라미터 제공
- **에러 처리**: 표준화된 오류 응답 