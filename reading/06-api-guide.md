# 🌐 API 개발 가이드

## 📋 목차

1. [API 설계 원칙](#api-설계-원칙)
2. [표준 응답 형식](#표준-응답-형식)
3. [컨트롤러 개발 패턴](#컨트롤러-개발-패턴)
4. [Request/Response DTO](#requestresponse-dto)
5. [예외 처리 및 에러 응답](#예외-처리-및-에러-응답)
6. [API 문서화](#api-문서화)
7. [검증 및 보안](#검증-및-보안)
8. [성능 최적화](#성능-최적화)
9. [API 버전 관리](#api-버전-관리)

## API 설계 원칙

### 🎯 RESTful 원칙

#### 1. 리소스 기반 URI 설계
```http
# ✅ 좋은 예: 명사형 리소스 중심
GET    /api/balance/{userId}          # 잔액 조회
POST   /api/balance/charge            # 잔액 충전
GET    /api/products                  # 상품 목록 조회
POST   /api/orders                    # 주문 생성
GET    /api/orders/{orderId}          # 주문 조회

# ❌ 나쁜 예: 동사형 액션 중심
GET    /api/getBalance/{userId}       # 동사 사용
POST   /api/chargeBalance             # 동사 사용
POST   /api/createOrder               # 동사 사용
```

#### 2. HTTP 메서드 활용
```http
# CRUD 표준 매핑
GET    /api/products                  # 조회 (Read)
POST   /api/products                  # 생성 (Create)
PUT    /api/products/{id}             # 전체 수정 (Update)
PATCH  /api/products/{id}             # 부분 수정 (Partial Update)
DELETE /api/products/{id}             # 삭제 (Delete)

# 비즈니스 액션
POST   /api/balance/charge            # 잔액 충전
POST   /api/orders/{orderId}/pay      # 주문 결제
POST   /api/coupons/issue             # 쿠폰 발급
```

#### 3. 상태 코드 활용
```java
// 성공 응답
@PostMapping("/charge")
@ResponseStatus(HttpStatus.OK)           // 200: 성공
public BalanceResponse chargeBalance() { }

@PostMapping
@ResponseStatus(HttpStatus.CREATED)      // 201: 생성 성공
public OrderResponse createOrder() { }

@GetMapping("/{id}")
@ResponseStatus(HttpStatus.OK)           // 200: 조회 성공
public ProductResponse getProduct() { }

// 에러 응답은 GlobalExceptionHandler에서 자동 처리
// 400 Bad Request, 404 Not Found, 500 Internal Server Error
```

### 📐 URL 네이밍 규칙

#### 1. 패키지별 API 그룹핑
```
/api/balance/           # 잔액 관리
  ├── /charge           # 잔액 충전
  └── /{userId}         # 잔액 조회

/api/products/          # 상품 관리
  ├── /list             # 상품 목록
  └── /popular          # 인기 상품

/api/orders/            # 주문 관리
  ├── /                 # 주문 생성
  ├── /{orderId}        # 주문 조회
  ├── /{orderId}/pay    # 주문 결제
  └── /user/{userId}    # 사용자별 주문 목록

/api/coupons/           # 쿠폰 관리
  ├── /issue            # 쿠폰 발급
  └── /{userId}         # 보유 쿠폰 조회
```

#### 2. 쿼리 파라미터 활용
```http
# 페이지네이션
GET /api/products?limit=20&offset=0

# 필터링
GET /api/orders?status=PAID&startDate=2024-01-01

# 정렬
GET /api/products?sort=price,desc&sort=name,asc

# 검색
GET /api/products?search=smartphone&category=electronics
```

## 표준 응답 형식

### 📦 CommonResponse 구조

#### 1. 기본 응답 형식
```json
{
  "code": "S001",
  "message": "요청이 성공적으로 처리되었습니다",
  "data": {
    "userId": 1,
    "amount": 50000,
    "updatedAt": "2024-01-01T12:00:00"
  },
  "timestamp": "2024-01-01T12:00:00.123"
}
```

#### 2. 성공 응답 패턴
```java
// 데이터가 있는 성공 응답
@PostMapping("/charge")
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
    Balance balance = chargeBalanceUseCase.execute(request.getUserId(), request.getAmount());
    return new BalanceResponse(
        balance.getUser().getId(),
        balance.getAmount(),
        balance.getUpdatedAt()
    );
    // SuccessResponseAdvice가 자동으로 CommonResponse.success()로 래핑
}

// 데이터가 없는 성공 응답
@DeleteMapping("/{id}")
public void deleteProduct(@PathVariable Long id) {
    deleteProductUseCase.execute(id);
    // void 반환 시 CommonResponse.success()로 자동 래핑
}
```

#### 3. 에러 응답 형식
```json
{
  "code": "U001",
  "message": "사용자를 찾을 수 없습니다",
  "data": null,
  "timestamp": "2024-01-01T12:00:00.123"
}
```

### 🔄 응답 자동화

#### 1. SuccessResponseAdvice (성공 응답 자동 래핑)
```java
@RestControllerAdvice
public class SuccessResponseAdvice implements ResponseBodyAdvice<Object> {
    
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return !returnType.getGenericParameterType().equals(CommonResponse.class);
    }
    
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, 
                                MediaType selectedContentType, 
                                Class<? extends HttpMessageConverter<?>> selectedConverterType, 
                                ServerHttpRequest request, 
                                ServerHttpResponse response) {
        
        if (body == null) {
            return CommonResponse.success();
        }
        return CommonResponse.success(body);
    }
}
```

## 컨트롤러 개발 패턴

### 🏗️ 기본 구조

#### 1. 컨트롤러 템플릿
```java
/**
 * 잔액 관리 Controller
 * 사용자 잔액 충전 및 조회 기능을 제공합니다.
 */
@Tag(name = "잔액 관리", description = "사용자 잔액 충전 및 조회 API")
@RestController
@RequestMapping("/api/balance")
@RequiredArgsConstructor
public class BalanceController {

    private final ChargeBalanceUseCase chargeBalanceUseCase;
    private final GetBalanceUseCase getBalanceUseCase;

    @BalanceApiDocs(summary = "잔액 충전", description = "사용자의 잔액을 충전합니다")
    @PostMapping("/charge")
    public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
        // 1. 요청 검증
        if (request == null) {
            throw new CommonException.InvalidRequest();
        }
        request.validate();
        
        // 2. 비즈니스 로직 호출
        Balance balance = chargeBalanceUseCase.execute(request.getUserId(), request.getAmount());
        
        // 3. 응답 DTO 변환
        return new BalanceResponse(
            balance.getUser().getId(),
            balance.getAmount(),
            balance.getUpdatedAt()
        );
    }
}
```

#### 2. 컨트롤러 레이어 책임
```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;
    private final PayOrderUseCase payOrderUseCase;
    private final GetOrderUseCase getOrderUseCase;

    @PostMapping
    public OrderResponse createOrder(@RequestBody OrderRequest request) {
        // ✅ 컨트롤러의 적절한 책임
        // 1. 요청 검증
        request.validate();
        
        // 2. DTO → 도메인 변환
        Map<Long, Integer> productQuantities = request.getProducts().stream()
            .collect(Collectors.toMap(
                OrderRequest.ProductQuantity::getProductId,
                OrderRequest.ProductQuantity::getQuantity
            ));
        
        // 3. 비즈니스 로직 호출 (위임)
        Order order = createOrderUseCase.execute(request.getUserId(), productQuantities);
        
        // 4. 도메인 → DTO 변환
        return OrderResponse.from(order);
        
        // ❌ 컨트롤러에서 하면 안 되는 것들
        // - 비즈니스 규칙 검증 (도메인 레이어 책임)
        // - 데이터베이스 직접 접근
        // - 복잡한 계산 로직
    }
}
```

### 🎨 HTTP 메서드별 패턴

#### 1. POST - 리소스 생성/액션
```java
// 리소스 생성
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public OrderResponse createOrder(@RequestBody OrderRequest request) {
    request.validate();
    Order order = createOrderUseCase.execute(request.getUserId(), request.getProductQuantities());
    return OrderResponse.from(order);
}

// 비즈니스 액션
@PostMapping("/charge")
@ResponseStatus(HttpStatus.OK)
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
    request.validate();
    Balance balance = chargeBalanceUseCase.execute(request.getUserId(), request.getAmount());
    return BalanceResponse.from(balance);
}
```

#### 2. GET - 리소스 조회
```java
// 단건 조회
@GetMapping("/{userId}")
public BalanceResponse getBalance(@PathVariable Long userId) {
    if (userId == null || userId <= 0) {
        throw new UserException.InvalidUser();
    }
    
    Balance balance = getBalanceUseCase.execute(userId);
    return BalanceResponse.from(balance);
}

// 목록 조회 (페이지네이션)
@GetMapping
public List<ProductResponse> getProducts(
        @RequestParam(defaultValue = "20") int limit,
        @RequestParam(defaultValue = "0") int offset) {
    
    validatePagination(limit, offset);
    
    List<Product> products = getProductUseCase.execute(limit, offset);
    return products.stream()
        .map(ProductResponse::from)
        .collect(Collectors.toList());
}

private void validatePagination(int limit, int offset) {
    if (limit <= 0 || limit > 100) {
        throw new IllegalArgumentException("Limit must be between 1 and 100");
    }
    if (offset < 0) {
        throw new IllegalArgumentException("Offset must not be negative");
    }
}
```

#### 3. PUT/PATCH - 리소스 수정
```java
// 전체 업데이트 (PUT)
@PutMapping("/{productId}")
public ProductResponse updateProduct(
        @PathVariable Long productId,
        @RequestBody ProductUpdateRequest request) {
    
    request.validate();
    Product product = updateProductUseCase.execute(productId, request.toUpdateInfo());
    return ProductResponse.from(product);
}

// 부분 업데이트 (PATCH)
@PatchMapping("/{productId}/stock")
public ProductResponse updateStock(
        @PathVariable Long productId,
        @RequestBody StockUpdateRequest request) {
    
    request.validate();
    Product product = updateStockUseCase.execute(productId, request.getNewStock());
    return ProductResponse.from(product);
}
```

## Request/Response DTO

### 📥 Request DTO 패턴

#### 1. 기본 Request 구조
```java
@Schema(description = "잔액 충전 요청")
public class BalanceRequest implements DocumentedDto {
    
    @Schema(description = "사용자 ID", example = "1", required = true)
    private Long userId;
    
    @Schema(description = "충전 금액", example = "10000", required = true)
    private BigDecimal amount;

    // 비즈니스 규칙 상수
    private static final BigDecimal MIN_CHARGE_AMOUNT = new BigDecimal("1000");
    private static final BigDecimal MAX_CHARGE_AMOUNT = new BigDecimal("1000000");

    // 생성자
    public BalanceRequest() {}
    public BalanceRequest(Long userId, BigDecimal amount) {
        this.userId = userId;
        this.amount = amount;
    }

    // Getters/Setters
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public BigDecimal getAmount() { return amount; }
    public void setAmount(BigDecimal amount) { this.amount = amount; }
    
    /**
     * 요청 데이터 검증
     * 기본적인 null 체크와 형식 검증만 수행
     * 비즈니스 규칙 검증은 UseCase에서 처리
     */
    public void validate() {
        if (userId == null || userId <= 0) {
            throw new UserException.InvalidUser();
        }
        if (amount == null) {
            throw new BalanceException.InvalidAmount();
        }
        if (amount.compareTo(MIN_CHARGE_AMOUNT) < 0 || amount.compareTo(MAX_CHARGE_AMOUNT) > 0) {
            throw new BalanceException.InvalidAmount();
        }
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new BalanceException.InvalidAmount();
        }
    }

    @Override
    public Map<String, SchemaInfo> getFieldDocumentation() {
        return Map.of(
                "userId", new SchemaInfo("사용자 ID", "1"),
                "amount", new SchemaInfo("충전 금액", "10000")
        );
    }
}
```

#### 2. 복합 Request 구조
```java
@Schema(description = "주문 생성 요청")
public class OrderRequest implements DocumentedDto {
    
    @Schema(description = "사용자 ID", example = "1", required = true)
    private Long userId;
    
    @Schema(description = "주문 상품 목록", required = true)
    private List<ProductQuantity> products;
    
    @Schema(description = "적용 쿠폰 ID 목록")
    private List<Long> couponIds = new ArrayList<>();

    public void validate() {
        if (userId == null || userId <= 0) {
            throw new UserException.InvalidUser();
        }
        if (products == null || products.isEmpty()) {
            throw new OrderException.EmptyItems();
        }
        
        // 각 상품 수량 검증
        for (ProductQuantity pq : products) {
            pq.validate();
        }
    }

    @Schema(description = "상품 수량 정보")
    public static class ProductQuantity {
        @Schema(description = "상품 ID", example = "1", required = true)
        private Long productId;
        
        @Schema(description = "주문 수량", example = "2", required = true)
        private Integer quantity;

        public void validate() {
            if (productId == null || productId <= 0) {
                throw new ProductException.InvalidProductId();
            }
            if (quantity == null || quantity <= 0) {
                throw new OrderException.InvalidQuantity();
            }
        }

        // Getters/Setters...
    }

    // Getters/Setters...
}
```

### 📤 Response DTO 패턴

#### 1. Record 기반 Response (권장)
```java
@Schema(description = "잔액 조회 응답")
public record BalanceResponse(
        @Schema(description = "사용자 ID", example = "1")
        Long userId,
        @Schema(description = "현재 잔액", example = "50000")
        BigDecimal amount,
        @Schema(description = "마지막 업데이트 시간", example = "2024-01-01T12:00:00")
        LocalDateTime updatedAt
) implements DocumentedDto {
    
    // 정적 팩토리 메서드
    public static BalanceResponse from(Balance balance) {
        return new BalanceResponse(
            balance.getUser().getId(),
            balance.getAmount(),
            balance.getUpdatedAt()
        );
    }

    @Override
    public Map<String, SchemaInfo> getFieldDocumentation() {
        return Map.of(
                "userId", new SchemaInfo("사용자 ID", "1"),
                "amount", new SchemaInfo("현재 잔액", "50000"),
                "updatedAt", new SchemaInfo("마지막 업데이트 시간", "2024-01-01T12:00:00")
        );
    }
}
```

#### 2. 복합 Response 구조
```java
@Schema(description = "주문 조회 응답")
public record OrderResponse(
        @Schema(description = "주문 ID", example = "1")
        Long orderId,
        @Schema(description = "사용자 ID", example = "1")
        Long userId,
        @Schema(description = "주문 상품 목록")
        List<OrderItemResponse> items,
        @Schema(description = "총 주문 금액", example = "25000")
        BigDecimal totalAmount,
        @Schema(description = "주문 상태", example = "PENDING")
        OrderStatus status,
        @Schema(description = "주문 생성 시간", example = "2024-01-01T12:00:00")
        LocalDateTime createdAt
) {
    
    public static OrderResponse from(Order order) {
        List<OrderItemResponse> itemResponses = order.getItems().stream()
            .map(OrderItemResponse::from)
            .collect(Collectors.toList());
            
        return new OrderResponse(
            order.getId(),
            order.getUser().getId(),
            itemResponses,
            order.getTotalAmount(),
            order.getStatus(),
            order.getCreatedAt()
        );
    }

    @Schema(description = "주문 상품 응답")
    public record OrderItemResponse(
            @Schema(description = "상품 ID", example = "1")
            Long productId,
            @Schema(description = "상품명", example = "스마트폰")
            String productName,
            @Schema(description = "주문 수량", example = "2")
            Integer quantity,
            @Schema(description = "단가", example = "10000")
            BigDecimal unitPrice,
            @Schema(description = "총 가격", example = "20000") 
            BigDecimal totalPrice
    ) {
        public static OrderItemResponse from(OrderItem item) {
            return new OrderItemResponse(
                item.getProduct().getId(),
                item.getProduct().getName(),
                item.getQuantity(),
                item.getUnitPrice(),
                item.getTotalPrice()
            );
        }
    }
}
```

### 🔄 DTO 변환 패턴

#### 1. 정적 팩토리 메서드
```java
// ✅ 좋은 예: 명확한 변환 의도
public record ProductResponse(
    Long productId,
    String name,
    BigDecimal price,
    Integer availableStock
) {
    public static ProductResponse from(Product product) {
        return new ProductResponse(
            product.getId(),
            product.getName(),
            product.getPrice(),
            product.getAvailableStock()  // 비즈니스 로직 메서드 활용
        );
    }
}

// ❌ 나쁜 예: 생성자에 모든 로직
public ProductResponse(Product product) {
    this.productId = product.getId();
    this.name = product.getName();
    this.price = product.getPrice();
    this.availableStock = product.getStock() - product.getReservedStock(); // 계산 로직
}
```

#### 2. 컬렉션 변환
```java
@GetMapping
public List<ProductResponse> getProducts() {
    List<Product> products = getProductUseCase.execute();
    
    return products.stream()
        .map(ProductResponse::from)
        .collect(Collectors.toList());
}

// 페이지네이션 응답
public record ProductListResponse(
    List<ProductResponse> products,
    int totalCount,
    int currentPage,
    int pageSize
) {
    public static ProductListResponse from(List<Product> products, int totalCount, int page, int size) {
        List<ProductResponse> productResponses = products.stream()
            .map(ProductResponse::from)
            .collect(Collectors.toList());
            
        return new ProductListResponse(productResponses, totalCount, page, size);
    }
}
```

## 예외 처리 및 에러 응답

### 🚨 GlobalExceptionHandler 활용

#### 1. 도메인 예외 처리
```java
@RestControllerAdvice(basePackages = "kr.hhplus.be.server.api.controller")
public class GlobalExceptionHandler {

    /**
     * 비즈니스 로직 예외 처리
     * 도메인에서 정의한 커스텀 예외들을 처리한다.
     */
    @ExceptionHandler({
        BalanceException.class, 
        CouponException.class, 
        OrderException.class, 
        PaymentException.class, 
        ProductException.class, 
        UserException.class, 
        CommonException.class
    })
    public ResponseEntity<CommonResponse<Object>> handleBusinessException(RuntimeException ex) {
        // ErrorCode 자동 매핑
        ErrorCode errorCode = ErrorCode.fromDomainException(ex);
        HttpStatus status = ErrorCode.getHttpStatusFromErrorCode(errorCode);
        
        return ResponseEntity.status(status).body(
            CommonResponse.failure(errorCode, ex.getMessage())
        );
    }

    /**
     * 입력값 검증 실패 예외 처리
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<CommonResponse<Object>> handleValidationExceptions(MethodArgumentNotValidException ex) {
        String errorMessage = ex.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(CommonResponse.failure(ErrorCode.INVALID_INPUT, errorMessage));
    }

    /**
     * 예상하지 못한 모든 예외 처리 (최후의 보루)
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<CommonResponse<Object>> handleAllUncaughtException(Exception ex) {
        // 로깅 처리
        log.error("Unexpected exception occurred", ex);
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(CommonResponse.failure(ErrorCode.INTERNAL_SERVER_ERROR));
    }
}
```

#### 2. ErrorCode와 HTTP 상태 매핑
```java
public enum ErrorCode {
    // 성공
    SUCCESS("S001", "요청이 성공적으로 처리되었습니다"),
    
    // 클라이언트 에러 (4xx)
    INVALID_INPUT("C001", "입력값이 유효하지 않습니다"),
    USER_NOT_FOUND("U001", "사용자를 찾을 수 없습니다"),
    PRODUCT_NOT_FOUND("P001", "상품을 찾을 수 없습니다"),
    INSUFFICIENT_BALANCE("B002", "잔액이 부족합니다"),
    PRODUCT_OUT_OF_STOCK("P002", "상품 재고가 부족합니다"),
    
    // 서버 에러 (5xx)
    INTERNAL_SERVER_ERROR("S500", "서버 내부 오류가 발생했습니다"),
    CONCURRENCY_ERROR("S503", "동시 처리 중 충돌이 발생했습니다");

    private static final Map<ErrorCode, HttpStatus> ERROR_CODE_HTTP_STATUS_MAP = Map.of(
        SUCCESS, HttpStatus.OK,
        INVALID_INPUT, HttpStatus.BAD_REQUEST,
        USER_NOT_FOUND, HttpStatus.NOT_FOUND,
        PRODUCT_NOT_FOUND, HttpStatus.NOT_FOUND,
        INSUFFICIENT_BALANCE, HttpStatus.BAD_REQUEST,
        PRODUCT_OUT_OF_STOCK, HttpStatus.BAD_REQUEST,
        INTERNAL_SERVER_ERROR, HttpStatus.INTERNAL_SERVER_ERROR,
        CONCURRENCY_ERROR, HttpStatus.SERVICE_UNAVAILABLE
    );

    public static HttpStatus getHttpStatusFromErrorCode(ErrorCode errorCode) {
        return ERROR_CODE_HTTP_STATUS_MAP.getOrDefault(errorCode, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

#### 3. 도메인 예외와 ErrorCode 매핑
```java
public enum ErrorCode {
    // ... 기존 코드

    /**
     * 도메인 예외에서 ErrorCode 자동 추출
     */
    public static ErrorCode fromDomainException(RuntimeException ex) {
        String className = ex.getClass().getSimpleName();
        
        // 예외 클래스명으로 ErrorCode 매핑
        switch (className) {
            case "NotFound":
                if (ex instanceof UserException.NotFound) return USER_NOT_FOUND;
                if (ex instanceof ProductException.NotFound) return PRODUCT_NOT_FOUND;
                if (ex instanceof BalanceException.NotFound) return BALANCE_NOT_FOUND;
                break;
            case "InvalidAmount":
                return INVALID_AMOUNT;
            case "InsufficientBalance":
                return INSUFFICIENT_BALANCE;
            case "OutOfStock":
                return PRODUCT_OUT_OF_STOCK;
            case "ConcurrencyConflict":
                return CONCURRENCY_ERROR;
            default:
                return INTERNAL_SERVER_ERROR;
        }
        
        return INTERNAL_SERVER_ERROR;
    }
}
```

## API 문서화

### 📚 Swagger/OpenAPI 3.0 활용

#### 1. 기본 API 문서 설정
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("이커머스 플랫폼 API")
                .version("1.0.0")
                .description("사용자 잔액, 상품, 주문, 쿠폰 관리 API")
                .contact(new Contact()
                    .name("개발팀")
                    .email("dev@hhplus.kr")
                )
            )
            .servers(List.of(
                new Server().url("http://localhost:8080").description("로컬 개발 서버"),
                new Server().url("https://api.hhplus.kr").description("운영 서버")
            ));
    }
}
```

#### 2. 컨트롤러 문서화
```java
@Tag(name = "잔액 관리", description = "사용자 잔액 충전 및 조회 API")
@RestController
@RequestMapping("/api/balance")
@RequiredArgsConstructor
public class BalanceController {

    @Operation(
        summary = "잔액 충전",
        description = "사용자의 잔액을 지정된 금액만큼 충전합니다. 충전 가능한 금액은 1,000원 이상 1,000,000원 이하입니다."
    )
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "충전 성공",
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = CommonResponse.class))),
        @ApiResponse(responseCode = "400", description = "잘못된 요청",
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = CommonResponse.class))),
        @ApiResponse(responseCode = "404", description = "사용자를 찾을 수 없음",
            content = @Content(mediaType = "application/json",
                schema = @Schema(implementation = CommonResponse.class)))
    })
    @PostMapping("/charge")
    public BalanceResponse chargeBalance(
            @Parameter(description = "잔액 충전 요청 정보", required = true)
            @RequestBody BalanceRequest request) {
        
        request.validate();
        Balance balance = chargeBalanceUseCase.execute(request.getUserId(), request.getAmount());
        return BalanceResponse.from(balance);
    }
}
```

#### 3. 커스텀 어노테이션 활용
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Operation
public @interface BalanceApiDocs {
    String summary();
    String description() default "";
}

// 사용 예시
@BalanceApiDocs(
    summary = "잔액 충전", 
    description = "사용자의 잔액을 충전합니다"
)
@PostMapping("/charge")
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
    // 구현...
}
```

#### 4. DTO 문서화
```java
@Schema(description = "잔액 충전 요청")
public class BalanceRequest implements DocumentedDto {
    
    @Schema(
        description = "사용자 ID", 
        example = "1", 
        required = true,
        minimum = "1"
    )
    private Long userId;
    
    @Schema(
        description = "충전 금액 (1,000원 이상 1,000,000원 이하)", 
        example = "10000", 
        required = true,
        minimum = "1000",
        maximum = "1000000"
    )
    private BigDecimal amount;

    // DocumentedDto 인터페이스 구현
    @Override
    public Map<String, SchemaInfo> getFieldDocumentation() {
        return Map.of(
            "userId", new SchemaInfo("사용자 ID", "1", "양의 정수"),
            "amount", new SchemaInfo("충전 금액", "10000", "1,000원 이상 1,000,000원 이하")
        );
    }
}
```

### 📖 API 문서 자동화

#### 1. 예제 데이터 자동 생성
```java
@Component
public class ApiExampleGenerator {
    
    public static BalanceRequest createBalanceRequestExample() {
        return new BalanceRequest(1L, new BigDecimal("50000"));
    }
    
    public static OrderRequest createOrderRequestExample() {
        OrderRequest request = new OrderRequest();
        request.setUserId(1L);
        request.setProducts(List.of(
            new OrderRequest.ProductQuantity(1L, 2),
            new OrderRequest.ProductQuantity(2L, 1)
        ));
        return request;
    }
}
```

#### 2. 에러 응답 문서화
```java
@Schema(description = "에러 응답")
public class ErrorResponseExample {
    
    @Schema(example = "U001")
    private String code;
    
    @Schema(example = "사용자를 찾을 수 없습니다")
    private String message;
    
    @Schema(example = "null")
    private Object data;
    
    @Schema(example = "2024-01-01T12:00:00.123")
    private LocalDateTime timestamp;
}
```

## 검증 및 보안

### 🔒 입력값 검증

#### 1. DTO 레벨 검증
```java
public class BalanceRequest {
    
    public void validate() {
        // null 검증
        if (userId == null || userId <= 0) {
            throw new UserException.InvalidUser();
        }
        
        // 비즈니스 규칙 검증
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new BalanceException.InvalidAmount();
        }
        
        // 범위 검증
        if (amount.compareTo(MIN_CHARGE_AMOUNT) < 0 || amount.compareTo(MAX_CHARGE_AMOUNT) > 0) {
            throw new BalanceException.InvalidAmount();
        }
    }
}
```

#### 2. 컨트롤러 레벨 검증
```java
@PostMapping("/charge")
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
    // 기본 null 검증
    if (request == null) {
        throw new CommonException.InvalidRequest();
    }
    
    // DTO 검증 위임
    request.validate();
    
    // 비즈니스 로직 호출
    Balance balance = chargeBalanceUseCase.execute(request.getUserId(), request.getAmount());
    return BalanceResponse.from(balance);
}

@GetMapping("/{userId}")
public BalanceResponse getBalance(@PathVariable Long userId) {
    // Path Variable 검증
    if (userId == null || userId <= 0) {
        throw new UserException.InvalidUser();
    }
    
    Balance balance = getBalanceUseCase.execute(userId);
    return BalanceResponse.from(balance);
}
```

### 🛡️ 보안 고려사항

#### 1. 민감한 정보 응답 제외
```java
// ✅ 좋은 예: 민감한 정보 제외
public record UserResponse(
    Long userId,
    String name,
    LocalDateTime createdAt
    // password, internalId 등 민감한 정보 제외
) {
    public static UserResponse from(User user) {
        return new UserResponse(
            user.getId(),
            user.getName(),
            user.getCreatedAt()
        );
    }
}

// ❌ 나쁜 예: 모든 정보 노출
public record UserResponse(
    Long userId,
    String name,
    String password,        // 민감한 정보 노출
    String internalId,      // 내부 ID 노출
    LocalDateTime createdAt
) { }
```

#### 2. 입력값 크기 제한
```java
public class OrderRequest {
    
    private static final int MAX_PRODUCTS_PER_ORDER = 50;
    private static final int MAX_QUANTITY_PER_PRODUCT = 100;
    
    public void validate() {
        // 주문 상품 수량 제한
        if (products.size() > MAX_PRODUCTS_PER_ORDER) {
            throw new OrderException.TooManyProducts();
        }
        
        // 개별 상품 수량 제한
        for (ProductQuantity pq : products) {
            if (pq.getQuantity() > MAX_QUANTITY_PER_PRODUCT) {
                throw new OrderException.InvalidQuantity();
            }
        }
    }
}
```

#### 3. 에러 메시지 보안
```java
// ✅ 좋은 예: 일반적인 에러 메시지
@ExceptionHandler(UserException.NotFound.class)
public ResponseEntity<CommonResponse<Object>> handleUserNotFound(UserException.NotFound ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
        .body(CommonResponse.failure(ErrorCode.USER_NOT_FOUND, "사용자를 찾을 수 없습니다"));
}

// ❌ 나쁜 예: 시스템 내부 정보 노출
@ExceptionHandler(Exception.class) 
public ResponseEntity<CommonResponse<Object>> handleException(Exception ex) {
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(CommonResponse.failure(ErrorCode.INTERNAL_SERVER_ERROR, ex.getMessage())); // 스택 트레이스 노출 위험
}
```

## 성능 최적화

### ⚡ 응답 시간 최적화

#### 1. 페이지네이션
```java
@GetMapping
public List<ProductResponse> getProducts(
        @RequestParam(defaultValue = "20") int limit,
        @RequestParam(defaultValue = "0") int offset,
        @RequestParam(required = false) String sortBy,
        @RequestParam(defaultValue = "asc") String sortDirection) {
    
    // 입력값 검증
    validatePagination(limit, offset);
    validateSortParameters(sortBy, sortDirection);
    
    // 페이지네이션된 결과 조회
    List<Product> products = getProductUseCase.execute(limit, offset, sortBy, sortDirection);
    
    return products.stream()
        .map(ProductResponse::from)
        .collect(Collectors.toList());
}

private void validatePagination(int limit, int offset) {
    if (limit <= 0 || limit > 100) {
        throw new IllegalArgumentException("Limit must be between 1 and 100");
    }
    if (offset < 0) {
        throw new IllegalArgumentException("Offset must not be negative");
    }
}
```

#### 2. 조건부 응답 (필드 선택)
```java
@GetMapping("/{userId}")
public BalanceResponse getBalance(
        @PathVariable Long userId,
        @RequestParam(required = false) String fields) {
    
    Balance balance = getBalanceUseCase.execute(userId);
    
    // 요청된 필드만 응답
    if (fields != null) {
        return BalanceResponse.withFields(balance, fields);
    }
    
    return BalanceResponse.from(balance);
}

// BalanceResponse에 필드 선택 로직 추가
public static BalanceResponse withFields(Balance balance, String fields) {
    Set<String> fieldSet = Set.of(fields.split(","));
    
    return new BalanceResponse(
        fieldSet.contains("userId") ? balance.getUser().getId() : null,
        fieldSet.contains("amount") ? balance.getAmount() : null,
        fieldSet.contains("updatedAt") ? balance.getUpdatedAt() : null
    );
}
```

#### 3. 캐시 활용
```java
@GetMapping("/popular")
public List<ProductResponse> getPopularProducts(
        @RequestParam(defaultValue = "7") int days) {
    
    // 캐시에서 조회 시도
    String cacheKey = "popular-products-" + days;
    List<Product> products = cacheService.get(cacheKey, () -> {
        // 캐시 미스 시 실제 로직 실행
        return getPopularProductListUseCase.execute(days);
    }, Duration.ofMinutes(30)); // 30분 TTL
    
    return products.stream()
        .map(ProductResponse::from)
        .collect(Collectors.toList());
}
```

### 📊 모니터링 및 로깅

#### 1. API 호출 로깅
```java
@RestControllerAdvice
public class ApiLoggingAdvice {
    
    private static final Logger log = LoggerFactory.getLogger(ApiLoggingAdvice.class);
    
    @Around("execution(* kr.hhplus.be.server.api.controller.*.*(..))")
    public Object logApiCall(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        
        try {
            Object result = joinPoint.proceed();
            stopWatch.stop();
            
            log.info("API 호출 성공: {}.{} - 응답시간: {}ms", 
                className, methodName, stopWatch.getTotalTimeMillis());
            
            return result;
        } catch (Exception e) {
            stopWatch.stop();
            
            log.error("API 호출 실패: {}.{} - 응답시간: {}ms, 에러: {}", 
                className, methodName, stopWatch.getTotalTimeMillis(), e.getMessage());
            
            throw e;
        }
    }
}
```

## API 버전 관리

### 🔄 버전 관리 전략

#### 1. URL 경로 버전 관리 (권장)
```java
// v1 API
@RestController
@RequestMapping("/api/v1/balance")
public class BalanceControllerV1 {
    
    @PostMapping("/charge")
    public BalanceResponseV1 chargeBalance(@RequestBody BalanceRequestV1 request) {
        // v1 구현
    }
}

// v2 API (새로운 필드 추가)
@RestController
@RequestMapping("/api/v2/balance")
public class BalanceControllerV2 {
    
    @PostMapping("/charge")
    public BalanceResponseV2 chargeBalance(@RequestBody BalanceRequestV2 request) {
        // v2 구현
    }
}
```

#### 2. 하위 호환성 유지
```java
// v2 응답 (v1 호환)
public record BalanceResponseV2(
    Long userId,
    BigDecimal amount,
    LocalDateTime updatedAt,
    // v2에서 추가된 필드
    String currency,
    BigDecimal availableCredit
) {
    // v1 응답으로 변환
    public BalanceResponseV1 toV1() {
        return new BalanceResponseV1(userId, amount, updatedAt);
    }
}
```

#### 3. 지원 중단 계획
```java
@RestController
@RequestMapping("/api/v1/balance")
@Deprecated(since = "2.0", forRemoval = true)
public class BalanceControllerV1 {
    
    @PostMapping("/charge")
    @Operation(description = "⚠️ Deprecated: v2 API를 사용해주세요. 2024년 12월 31일 지원 중단 예정")
    public BalanceResponseV1 chargeBalance(@RequestBody BalanceRequestV1 request) {
        // 경고 헤더 추가
        HttpServletResponse.addHeader("Deprecation", "true");
        HttpServletResponse.addHeader("Sunset", "2024-12-31");
        
        // 기존 로직 유지
    }
}
```

---

**다음 읽을 문서**: [07-troubleshooting-guide.md](07-troubleshooting-guide.md)