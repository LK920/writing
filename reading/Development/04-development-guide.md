# 🛠️ 개발 가이드

## 📋 목차

1. [코딩 컨벤션](#코딩-컨벤션)
2. [네이밍 규칙](#네이밍-규칙)
3. [패키지 구조 규칙](#패키지-구조-규칙)
4. [개발 베스트 프랙티스](#개발-베스트-프랙티스)
5. [예외 처리 가이드](#예외-처리-가이드)
6. [동시성 처리 가이드](#동시성-처리-가이드)
7. [성능 최적화 가이드](#성능-최적화-가이드)
8. [코드 리뷰 가이드](#코드-리뷰-가이드)

## 코딩 컨벤션

### 🎨 Java 스타일 가이드

#### 1. 클래스 및 인터페이스
```java
// ✅ 좋은 예: PascalCase 사용
public class ChargeBalanceUseCase {
    // 구현...
}

public interface BalanceRepositoryPort {
    // 인터페이스 정의...
}

// ❌ 나쁜 예
public class chargeBalanceUseCase { }
public interface balanceRepository { }
```

#### 2. 메서드 및 변수
```java
// ✅ 좋은 예: camelCase 사용
public Balance chargeBalance(Long userId, BigDecimal amount) {
    BigDecimal currentAmount = balance.getAmount();
    LocalDateTime lastUpdatedAt = balance.getUpdatedAt();
    
    return balance;
}

// ❌ 나쁜 예
public Balance charge_balance(Long UserId, BigDecimal Amount) { }
```

#### 3. 상수
```java
// ✅ 좋은 예: UPPER_SNAKE_CASE
public class BalanceConstants {
    public static final BigDecimal MIN_CHARGE_AMOUNT = BigDecimal.valueOf(1000);
    public static final BigDecimal MAX_CHARGE_AMOUNT = BigDecimal.valueOf(1_000_000);
    public static final int DEFAULT_PAGE_SIZE = 20;
}

// ❌ 나쁜 예
public static final BigDecimal minChargeAmount = BigDecimal.valueOf(1000);
```

#### 4. 패키지명
```java
// ✅ 좋은 예: 소문자, 도트 구분
package kr.hhplus.be.server.domain.usecase.balance;
package kr.hhplus.be.server.adapter.storage.inmemory;

// ❌ 나쁜 예
package kr.hhplus.be.server.Domain.UseCase.Balance;
```

### 📝 주석 스타일

#### 1. 클래스 주석
```java
/**
 * 사용자 잔액 충전을 처리하는 유스케이스
 * 
 * <p>충전 가능한 금액 범위를 검증하고, 사용자의 현재 잔액에 충전 금액을 추가한다.
 * 동시성 제어를 위해 사용자별 락을 사용한다.</p>
 * 
 * @author 개발팀
 * @since 1.0
 */
@Component
public class ChargeBalanceUseCase {
    // 구현...
}
```

#### 2. 메서드 주석
```java
/**
 * 사용자 잔액을 충전한다
 * 
 * @param userId 충전할 사용자 ID
 * @param amount 충전 금액 (1,000원 이상 1,000,000원 이하)
 * @return 충전된 잔액 정보
 * @throws UserException.NotFound 사용자를 찾을 수 없는 경우
 * @throws BalanceException.InvalidAmount 충전 금액이 유효하지 않은 경우
 */
@Transactional
public Balance execute(Long userId, BigDecimal amount) {
    // 구현...
}
```

#### 3. 복잡한 로직 설명
```java
public void reserveStock(int quantity) {
    validateQuantity(quantity);
    
    // 이용 가능한 재고 = 전체 재고 - 예약된 재고
    if (!hasAvailableStock(quantity)) {
        throw new ProductException.OutOfStock();
    }
    
    // 재고를 즉시 차감하지 않고 예약 처리
    // 결제 완료 시 confirmReservation()에서 실제 차감
    this.reservedStock += quantity;
}
```

## 네이밍 규칙

### 🏷️ 클래스 네이밍

#### 1. UseCase 클래스
```java
// ✅ 좋은 예: 동사 + 명사 + UseCase
public class ChargeBalanceUseCase { }
public class CreateOrderUseCase { }
public class IssueCouponUseCase { }

// ❌ 나쁜 예
public class BalanceCharger { }
public class OrderCreator { }
```

#### 2. Repository 클래스
```java
// ✅ 좋은 예: InMemory + 도메인명 + Repository
public class InMemoryBalanceRepository { }
public class InMemoryOrderRepository { }

// ❌ 나쁜 예
public class BalanceRepo { }
public class MemoryOrderStorage { }
```

#### 3. Exception 클래스
```java
// ✅ 좋은 예: 도메인명 + Exception, 내부 클래스로 구체적 예외
public class BalanceException extends RuntimeException {
    
    public static class NotFound extends BalanceException {
        public NotFound() {
            super(ErrorCode.BALANCE_NOT_FOUND.getMessage());
        }
    }
    
    public static class InsufficientBalance extends BalanceException {
        public InsufficientBalance(BigDecimal required, BigDecimal current) {
            super(String.format("잔액 부족: 필요 금액 %s, 현재 잔액 %s", required, current));
        }
    }
}

// ❌ 나쁜 예
public class BalanceNotFoundException { }
public class NotEnoughBalanceException { }
```

### 🔤 메서드 네이밍

#### 1. CRUD 메서드
```java
// ✅ 좋은 예: 명확한 의도 표현
public Optional<Balance> findByUserId(Long userId) { }
public Balance save(Balance balance) { }
public void deleteByUserId(Long userId) { }
public List<Order> findByUserIdAndStatus(Long userId, OrderStatus status) { }

// ❌ 나쁜 예
public Balance get(Long userId) { }
public void remove(Long userId) { }
```

#### 2. 비즈니스 로직 메서드
```java
// ✅ 좋은 예: 비즈니스 의미 명확
public void charge(BigDecimal amount) { }
public void deduct(BigDecimal amount) { }
public boolean hasEnoughBalance(BigDecimal requiredAmount) { }
public void reserveStock(int quantity) { }
public void confirmReservation(int quantity) { }

// ❌ 나쁜 예
public void add(BigDecimal amount) { }
public void minus(BigDecimal amount) { }
public boolean check(BigDecimal amount) { }
```

#### 3. 검증 메서드
```java
// ✅ 좋은 예: validate 접두사, 구체적 목적
private void validateChargeAmount(BigDecimal amount) {
    if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
        throw new BalanceException.InvalidAmount();
    }
}

private void validateUserExists(Long userId) {
    // 검증 로직
}

// ❌ 나쁜 예
private void checkAmount(BigDecimal amount) { }
private boolean isValid(Long userId) { }
```

### 📦 변수 네이밍

#### 1. 컬렉션 변수
```java
// ✅ 좋은 예: 복수형 사용
List<Product> products = new ArrayList<>();
Map<Long, Balance> balances = new ConcurrentHashMap<>();
Set<Long> userIds = new HashSet<>();

// ❌ 나쁜 예
List<Product> productList = new ArrayList<>();
Map<Long, Balance> balanceMap = new ConcurrentHashMap<>();
```

#### 2. 불리언 변수
```java
// ✅ 좋은 예: is, has, can, should 접두사
boolean isActive = coupon.getStatus() == CouponStatus.ACTIVE;
boolean hasEnoughStock = product.hasAvailableStock(quantity);
boolean canIssue = coupon.canIssue();
boolean shouldExpire = LocalDateTime.now().isAfter(coupon.getEndDate());

// ❌ 나쁜 예
boolean active = true;
boolean stock = false;
```

## 패키지 구조 규칙

### 📁 표준 패키지 구조

```
kr.hhplus.be.server/
├── api/                    # API 레이어
│   ├── controller/         # REST 컨트롤러
│   │   ├── BalanceController.java
│   │   ├── OrderController.java
│   │   └── ...
│   ├── dto/               # 데이터 전송 객체
│   │   ├── request/       # 요청 DTO
│   │   │   ├── BalanceRequest.java
│   │   │   └── ...
│   │   └── response/      # 응답 DTO
│   │       ├── BalanceResponse.java
│   │       └── ...
│   ├── docs/              # API 문서 설정
│   ├── scheduler/         # 스케줄 작업
│   ├── CommonResponse.java
│   ├── ErrorCode.java
│   └── GlobalExceptionHandler.java
├── domain/                # 도메인 레이어
│   ├── entity/           # 도메인 엔티티
│   │   ├── Balance.java
│   │   ├── Order.java
│   │   └── ...
│   ├── enums/            # 도메인 열거형
│   │   ├── OrderStatus.java
│   │   ├── CouponStatus.java
│   │   └── ...
│   ├── exception/        # 도메인 예외
│   │   ├── BalanceException.java
│   │   ├── OrderException.java
│   │   └── ...
│   ├── port/             # 포트 인터페이스
│   │   ├── storage/      # 저장소 포트
│   │   ├── cache/        # 캐시 포트
│   │   └── locking/      # 락 포트
│   └── usecase/          # 유스케이스
│       ├── balance/
│       ├── order/
│       └── ...
├── adapter/              # 어댑터 레이어
│   ├── storage/inmemory/ # 인메모리 저장소
│   ├── cache/           # 캐시 구현
│   └── locking/         # 락 구현
└── config/              # 설정
    ├── JpaConfig.java
    └── ...
```

### 📋 패키지 명명 규칙

#### 1. 도메인별 그룹핑
```java
// ✅ 좋은 예: 도메인별로 명확히 분리
kr.hhplus.be.server.domain.usecase.balance.ChargeBalanceUseCase
kr.hhplus.be.server.domain.usecase.balance.GetBalanceUseCase
kr.hhplus.be.server.domain.usecase.order.CreateOrderUseCase
kr.hhplus.be.server.domain.usecase.order.PayOrderUseCase

// ❌ 나쁜 예: 도메인 구분 없음
kr.hhplus.be.server.domain.usecase.ChargeBalanceUseCase
kr.hhplus.be.server.domain.usecase.CreateOrderUseCase
```

#### 2. 레이어별 분리
```java
// ✅ 좋은 예: 명확한 레이어 구분
kr.hhplus.be.server.api.controller.BalanceController
kr.hhplus.be.server.domain.entity.Balance  
kr.hhplus.be.server.adapter.storage.inmemory.InMemoryBalanceRepository

// ❌ 나쁜 예: 레이어 혼재
kr.hhplus.be.server.balance.BalanceController
kr.hhplus.be.server.balance.Balance
kr.hhplus.be.server.balance.BalanceRepository
```

## 개발 베스트 프랙티스

### 🎯 도메인 주도 설계 원칙

#### 1. 엔티티는 자기 자신을 보호한다
```java
// ✅ 좋은 예: 엔티티 내부에서 비즈니스 규칙 검증
@Getter
@SuperBuilder
public class Product extends BaseEntity {
    private int stock;
    private int reservedStock;
    
    public void reserveStock(int quantity) {
        validateQuantity(quantity);
        
        if (!hasAvailableStock(quantity)) {
            throw new ProductException.OutOfStock();
        }
        
        this.reservedStock += quantity;
    }
    
    private void validateQuantity(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("수량은 0보다 커야 합니다");
        }
    }
    
    private boolean hasAvailableStock(int quantity) {
        return (this.stock - this.reservedStock) >= quantity;
    }
}

// ❌ 나쁜 예: 외부에서 엔티티 상태 직접 변경
// UseCase에서
product.setReservedStock(product.getReservedStock() + quantity); // 검증 없음
```

#### 2. UseCase는 비즈니스 플로우를 조율한다
```java
// ✅ 좋은 예: UseCase에서 전체 플로우 관리
@Component
@RequiredArgsConstructor
public class CreateOrderUseCase {
    
    private final UserRepositoryPort userRepositoryPort;
    private final ProductRepositoryPort productRepositoryPort;
    private final OrderRepositoryPort orderRepositoryPort;
    private final LockingPort lockingPort;
    
    @Transactional
    public Order execute(Long userId, Map<Long, Integer> productQuantities) {
        // 1. 동시성 제어
        String lockKey = "order-creation-" + userId;
        if (!lockingPort.acquireLock(lockKey)) {
            throw new ConcurrencyException.LockAcquisitionFailed();
        }
        
        try {
            // 2. 사용자 검증
            User user = findUserOrThrow(userId);
            
            // 3. 주문 아이템 생성 및 재고 예약
            List<OrderItem> orderItems = createOrderItemsAndReserveStock(productQuantities);
            
            // 4. 주문 생성
            Order order = createOrder(user, orderItems);
            
            return orderRepositoryPort.save(order);
            
        } finally {
            lockingPort.releaseLock(lockKey);
        }
    }
    
    // 복잡한 로직은 private 메서드로 분리
    private User findUserOrThrow(Long userId) {
        return userRepositoryPort.findById(userId)
            .orElseThrow(() -> new UserException.NotFound());
    }
}

// ❌ 나쁜 예: UseCase에서 세부 구현까지 모두 처리
@Component
public class CreateOrderUseCase {
    public Order execute(Long userId, Map<Long, Integer> productQuantities) {
        // 모든 로직이 하나의 메서드에 집중되어 가독성 저하
        // 100줄 이상의 긴 메서드...
    }
}
```

### 🔄 의존성 주입 패턴

#### 1. 생성자 주입 (권장)
```java
// ✅ 좋은 예: 생성자 주입으로 불변성 보장
@Component
@RequiredArgsConstructor  // Lombok으로 생성자 자동 생성
public class ChargeBalanceUseCase {
    
    private final UserRepositoryPort userRepositoryPort;
    private final BalanceRepositoryPort balanceRepositoryPort;
    
    // 필드는 final로 불변성 보장
    // 테스트에서 Mock 주입 용이
}

// ❌ 나쁜 예: 필드 주입
@Component
public class ChargeBalanceUseCase {
    
    @Autowired
    private UserRepositoryPort userRepositoryPort;  // 테스트 어려움
    
    @Autowired
    private BalanceRepositoryPort balanceRepositoryPort;  // 불변성 보장 안됨
}
```

#### 2. 인터페이스 의존
```java
// ✅ 좋은 예: 포트 인터페이스에 의존
@Component
@RequiredArgsConstructor
public class ChargeBalanceUseCase {
    
    private final BalanceRepositoryPort balanceRepositoryPort;  // 인터페이스
    
    public Balance execute(Long userId, BigDecimal amount) {
        // 구현체가 InMemory든 JPA든 상관없이 동일한 코드
        return balanceRepositoryPort.save(balance);
    }
}

// ❌ 나쁜 예: 구체 클래스에 의존
@Component
@RequiredArgsConstructor
public class ChargeBalanceUseCase {
    
    private final InMemoryBalanceRepository balanceRepository;  // 구체 클래스
    
    // 저장소 변경 시 이 코드도 수정 필요
}
```

### 📊 데이터 검증 패턴

#### 1. Request DTO에서 기본 검증
```java
// ✅ 좋은 예: DTO에서 형식 검증, 도메인에서 비즈니스 검증
public record BalanceRequest(
    Long userId,
    BigDecimal amount
) {
    public void validate() {
        // 기본적인 null, 형식 검증
        if (userId == null || userId <= 0) {
            throw new UserException.InvalidUser();
        }
        if (amount == null) {
            throw new BalanceException.InvalidAmount();
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new BalanceException.InvalidAmount();
        }
    }
}

// UseCase에서 비즈니스 검증
public class ChargeBalanceUseCase {
    public Balance execute(Long userId, BigDecimal amount) {
        // 비즈니스 규칙 검증
        validateChargeAmount(amount);
        // ...
    }
    
    private void validateChargeAmount(BigDecimal amount) {
        if (amount.compareTo(MAX_CHARGE_AMOUNT) > 0) {
            throw new BalanceException.InvalidAmount();
        }
    }
}
```

#### 2. 도메인 엔티티에서 무결성 보장
```java
// ✅ 좋은 예: 엔티티 생성 시점에 검증
@Getter
@SuperBuilder
public class Coupon extends BaseEntity {
    private String code;
    private BigDecimal discountRate;
    private int maxIssuance;
    private int issuedCount;
    
    @Override
    public void onCreate() {
        super.onCreate();
        validateCouponData();
    }
    
    @Override
    public void onUpdate() {
        super.onUpdate();
        validateCouponData();
    }
    
    private void validateCouponData() {
        if (code == null || code.trim().isEmpty()) {
            throw new CouponException.InvalidCouponData("쿠폰 코드는 필수입니다");
        }
        if (discountRate.compareTo(BigDecimal.ZERO) < 0 
            || discountRate.compareTo(BigDecimal.valueOf(100)) > 0) {
            throw new CouponException.InvalidCouponData("할인율은 0-100% 범위여야 합니다");
        }
    }
}
```

## 예외 처리 가이드

### 🚨 예외 계층 구조

```java
// 최상위 도메인 예외
public class BalanceException extends RuntimeException {
    protected BalanceException(String message) {
        super(message);
    }
    
    protected BalanceException(String message, Throwable cause) {
        super(message, cause);
    }
    
    // 구체적인 예외들을 내부 클래스로 정의
    public static class NotFound extends BalanceException {
        public NotFound() {
            super(ErrorCode.BALANCE_NOT_FOUND.getMessage());
        }
    }
    
    public static class InsufficientBalance extends BalanceException {
        public InsufficientBalance(BigDecimal required, BigDecimal current) {
            super(String.format("잔액 부족: 필요 금액 %s, 현재 잔액 %s", required, current));
        }
    }
    
    public static class InvalidAmount extends BalanceException {
        public InvalidAmount() {
            super(ErrorCode.INVALID_AMOUNT.getMessage());
        }
    }
}
```

### 🎯 ErrorCode 활용

```java
// ✅ 좋은 예: 중앙집중식 에러 코드 관리
public enum ErrorCode {
    // 사용자 관련
    USER_NOT_FOUND("U001", "사용자를 찾을 수 없습니다"),
    INVALID_USER_ID("U002", "유효하지 않은 사용자 ID입니다"),
    
    // 잔액 관련  
    BALANCE_NOT_FOUND("B001", "잔액 정보를 찾을 수 없습니다"),
    INSUFFICIENT_BALANCE("B002", "잔액이 부족합니다"),
    INVALID_AMOUNT("B003", "유효하지 않은 금액입니다"),
    
    // 상품 관련
    PRODUCT_NOT_FOUND("P001", "상품을 찾을 수 없습니다"),
    PRODUCT_OUT_OF_STOCK("P002", "상품 재고가 부족합니다"),
    
    // 공통
    INVALID_INPUT("C001", "입력값이 유효하지 않습니다"),
    CONCURRENCY_CONFLICT("C002", "동시 처리 중 충돌이 발생했습니다");
    
    private final String code;
    private final String message;
    
    ErrorCode(String code, String message) {
        this.code = code;
        this.message = message;
    }
    
    public String getMessage() {
        return message;
    }
    
    public String getCode() {
        return code;
    }
}

// 예외에서 ErrorCode 사용
public static class NotFound extends BalanceException {
    public NotFound() {
        super(ErrorCode.BALANCE_NOT_FOUND.getMessage());
    }
}
```

### 🌐 전역 예외 처리

```java
// ✅ 좋은 예: @ControllerAdvice로 중앙 집중 처리
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserException.class)
    public ResponseEntity<CommonResponse<Void>> handleUserException(UserException e) {
        log.warn("User exception occurred: {}", e.getMessage());
        
        ErrorCode errorCode = getErrorCodeFromException(e);
        HttpStatus status = ErrorCode.getHttpStatusFromErrorCode(errorCode);
        
        return ResponseEntity.status(status)
            .body(CommonResponse.error(errorCode.getCode(), e.getMessage()));
    }
    
    @ExceptionHandler(BalanceException.class)
    public ResponseEntity<CommonResponse<Void>> handleBalanceException(BalanceException e) {
        log.warn("Balance exception occurred: {}", e.getMessage());
        
        ErrorCode errorCode = getErrorCodeFromException(e);
        HttpStatus status = ErrorCode.getHttpStatusFromErrorCode(errorCode);
        
        return ResponseEntity.status(status)
            .body(CommonResponse.error(errorCode.getCode(), e.getMessage()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<CommonResponse<Void>> handleGeneralException(Exception e) {
        log.error("Unexpected exception occurred", e);
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(CommonResponse.error("INTERNAL_ERROR", "서버 내부 오류가 발생했습니다"));
    }
}
```

## 동시성 처리 가이드

### 🔒 락 사용 패턴

#### 1. 사용자별 락
```java
// ✅ 좋은 예: 세밀한 락 범위로 성능 최적화
@Component
@RequiredArgsConstructor
public class ChargeBalanceUseCase {
    
    private final LockingPort lockingPort;
    
    @Transactional
    public Balance execute(Long userId, BigDecimal amount) {
        String lockKey = "balance-charge-" + userId;  // 사용자별 락
        
        if (!lockingPort.acquireLock(lockKey)) {
            throw new ConcurrencyException.LockAcquisitionFailed();
        }
        
        try {
            // 잔액 충전 로직
            return processBalanceCharge(userId, amount);
            
        } finally {
            lockingPort.releaseLock(lockKey);  // 반드시 해제
        }
    }
}

// ❌ 나쁜 예: 전역 락으로 성능 저하
String lockKey = "balance-charge";  // 모든 사용자가 대기
```

#### 2. 리소스별 락
```java
// ✅ 좋은 예: 리소스별 세분화된 락
@Component
@RequiredArgsConstructor  
public class IssueCouponUseCase {
    
    @Transactional
    public CouponHistory execute(Long userId, Long couponId) {
        String lockKey = "coupon-issuance-" + couponId;  // 쿠폰별 락
        
        if (!lockingPort.acquireLock(lockKey)) {
            throw new ConcurrencyException.LockAcquisitionFailed();
        }
        
        try {
            // 쿠폰 발급 로직
            return processCouponIssuance(userId, couponId);
            
        } finally {
            lockingPort.releaseLock(lockKey);
        }
    }
}
```

### ⚡ ConcurrentHashMap 활용

```java
// ✅ 좋은 예: ConcurrentHashMap의 원자적 연산 활용
@Repository
public class InMemoryBalanceRepository implements BalanceRepositoryPort {
    
    private final Map<Long, Balance> balances = new ConcurrentHashMap<>();
    private final AtomicLong nextId = new AtomicLong(1L);
    
    @Override
    public Balance save(Balance balance) {
        Long balanceId = balance.getId() != null ? balance.getId() : nextId.getAndIncrement();
        
        // compute() 메서드로 원자적 업데이트
        return balances.compute(balanceId, (key, existing) -> {
            if (existing != null) {
                // 업데이트
                balance.onUpdate();
                balance.setId(existing.getId());
                balance.setCreatedAt(existing.getCreatedAt());
                return balance;
            } else {
                // 생성
                balance.onCreate();
                if (balance.getId() == null) {
                    balance.setId(balanceId);
                }
                return balance;
            }
        });
    }
}

// ❌ 나쁜 예: 비원자적 연산
public Balance save(Balance balance) {
    if (balances.containsKey(balance.getId())) {  // 비원자적
        balances.put(balance.getId(), balance);   // 사이에 다른 스레드가 개입 가능
    }
    return balance;
}
```

## 성능 최적화 가이드

### 🚀 캐싱 전략

#### 1. 읽기 전용 데이터 캐싱
```java
// ✅ 좋은 예: 자주 조회되는 데이터 캐싱
@Component
@RequiredArgsConstructor
public class GetBalanceUseCase {
    
    private final BalanceRepositoryPort balanceRepositoryPort;
    private final BalanceCachePort balanceCachePort;
    
    public Optional<Balance> execute(Long userId) {
        // 1. 캐시에서 먼저 조회
        Optional<Balance> cachedBalance = balanceCachePort.getBalance(userId);
        if (cachedBalance.isPresent()) {
            return cachedBalance;
        }
        
        // 2. 저장소에서 조회
        Optional<Balance> balance = balanceRepositoryPort.findByUserId(userId);
        
        // 3. 캐시에 저장 (TTL 설정)
        balance.ifPresent(b -> balanceCachePort.setBalance(userId, b, Duration.ofMinutes(10)));
        
        return balance;
    }
}
```

#### 2. 쓰기 시 캐시 무효화
```java
// ✅ 좋은 예: 데이터 변경 시 캐시 무효화
@Component
@RequiredArgsConstructor
public class ChargeBalanceUseCase {
    
    private final BalanceRepositoryPort balanceRepositoryPort;
    private final BalanceCachePort balanceCachePort;
    
    @Transactional
    public Balance execute(Long userId, BigDecimal amount) {
        // 잔액 변경 로직
        Balance updatedBalance = processBalanceCharge(userId, amount);
        
        // 캐시 무효화
        balanceCachePort.evictBalance(userId);
        
        return updatedBalance;
    }
}
```

### 📊 페이지네이션

```java
// ✅ 좋은 예: 페이지네이션으로 메모리 사용량 제어
@Component
@RequiredArgsConstructor
public class GetProductUseCase {
    
    private final ProductRepositoryPort productRepositoryPort;
    
    public List<Product> execute(int limit, int offset) {
        // 입력값 검증
        validatePagination(limit, offset);
        
        return productRepositoryPort.findAll(limit, offset);
    }
    
    private void validatePagination(int limit, int offset) {
        if (limit <= 0 || limit > 100) {  // 최대 100개로 제한
            throw new IllegalArgumentException("Limit must be between 1 and 100");
        }
        if (offset < 0) {
            throw new IllegalArgumentException("Offset must not be negative");
        }
    }
}
```

## 코드 리뷰 가이드

### ✅ 리뷰 체크리스트

#### 1. 아키텍처 준수
- [ ] 의존성 방향이 올바른가? (외부 → 내부)
- [ ] 도메인 로직이 적절한 위치에 있는가?
- [ ] 포트와 어댑터 패턴이 올바르게 적용되었는가?

#### 2. 코드 품질
- [ ] 메서드가 하나의 책임만 가지는가?
- [ ] 네이밍이 의도를 명확히 표현하는가?
- [ ] 매직 넘버나 하드코딩된 값이 없는가?

#### 3. 예외 처리
- [ ] 적절한 도메인 예외를 사용하는가?
- [ ] ErrorCode를 올바르게 사용하는가?
- [ ] 예외 메시지가 충분히 설명적인가?

#### 4. 테스트
- [ ] 단위 테스트가 작성되었는가?
- [ ] 경계값 테스트가 포함되었는가?
- [ ] 예외 상황 테스트가 있는가?

#### 5. 성능
- [ ] N+1 쿼리 문제가 없는가?
- [ ] 불필요한 객체 생성이 없는가?
- [ ] 적절한 캐싱이 적용되었는가?

### 🔍 리뷰 예시

```java
// ❌ 리뷰 포인트: 책임이 너무 많음
@Component
public class OrderService {
    public void createOrderAndSendEmail(OrderRequest request) {
        // 1. 주문 생성
        Order order = createOrder(request);
        
        // 2. 재고 차감
        reduceStock(order);
        
        // 3. 결제 처리
        processPayment(order);
        
        // 4. 이메일 발송 (다른 책임!)
        sendOrderConfirmationEmail(order);
    }
}

// ✅ 개선: 책임 분리
@Component
@RequiredArgsConstructor
public class CreateOrderUseCase {
    
    private final OrderRepositoryPort orderRepositoryPort;
    private final ProductRepositoryPort productRepositoryPort;
    private final OrderEventPort orderEventPort;  // 이벤트로 분리
    
    @Transactional
    public Order execute(OrderRequest request) {
        // 주문 생성 로직에만 집중
        Order order = createOrder(request);
        reserveStock(order);
        
        Order savedOrder = orderRepositoryPort.save(order);
        
        // 이벤트 발행으로 부가 작업 처리
        orderEventPort.publishOrderCreated(savedOrder);
        
        return savedOrder;
    }
}
```

---

**다음 읽을 문서**: [05-testing-guide.md](05-testing-guide.md)