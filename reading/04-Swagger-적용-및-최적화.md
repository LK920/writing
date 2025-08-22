# Swagger 적용 및 최적화 가이드

## 📋 목차
1. Swagger 기본 설정
2. 문제점과 해결 과정
3. 커스텀 어노테이션 설계
4. DTO 기반 문서화
5. 최적화 결과

---

## Swagger 기본 설정

### 📦 의존성 추가
```kotlin
// build.gradle.kts
dependencies {
    // Swagger/OpenAPI
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
}
```

### ⚙️ 기본 설정 클래스
```java
/**
 * Swagger/OpenAPI 설정 클래스
 * 
 * API 문서 자동 생성을 위한 설정을 담당한다.
 * 접속 URL: http://localhost:8080/swagger-ui/index.html
 */
@Configuration
public class SwaggerConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("E-Commerce API")
                        .version("1.0.0")
                        .description("항해플러스 이커머스 서비스 API 문서"))
                .servers(List.of(
                        new Server().url("http://localhost:8080").description("로컬 서버"),
                        new Server().url("https://dummy.url").description("운영 서버")
                ));
    }
}
```

### 🌐 접속 방법
- **Swagger UI**: `http://localhost:8080/swagger-ui/index.html`
- **OpenAPI JSON**: `http://localhost:8080/v3/api-docs`

---

## 문제점과 해결 과정

### ❌ 1단계: 지저분한 초기 코드
```java
// 😵 어노테이션 지옥 - 가독성 최악
@Operation(summary = "잔액 충전", description = "사용자의 잔액을 충전합니다.")
@ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "충전 성공"),
        @ApiResponse(responseCode = "400", description = "잘못된 요청 (유효하지 않은 금액 등)"),
        @ApiResponse(responseCode = "500", description = "서버 오류")
})
@PostMapping("/charge")
@ResponseStatus(HttpStatus.OK)
public void chargeBalance(
        @Parameter(description = "사용자 ID", required = true, example = "1")
        @NotNull(message = "사용자 ID는 필수입니다") @RequestParam Long userId,
        @Parameter(description = "충전 금액", required = true, example = "10000")
        @NotNull(message = "충전 금액은 필수입니다") 
        @DecimalMin(value = "0.0", inclusive = false, message = "충전 금액은 0보다 커야 합니다") 
        @RequestParam BigDecimal amount) {
    // 실제 로직은 몇 줄 안 되는데 어노테이션이 더 많음
}
```

### 🤔 문제점 분석
1. **가독성 저하**: 어노테이션이 실제 로직보다 많음
2. **중복 코드**: 비슷한 응답 패턴이 계속 반복
3. **유지보수 어려움**: 응답 코드 변경 시 모든 메서드 수정 필요
4. **컨트롤러 오염**: 비즈니스 로직과 무관한 문서화 코드가 섞임

### ✅ 해결 전략 (4단계 최적화)
1. **커스텀 어노테이션**: 반복되는 패턴을 어노테이션으로 묶기
2. **DTO 기반 문서화**: 요청/응답 스펙을 DTO에 위임
3. **공통 응답 모델**: 표준화된 오류 응답
4. **역할 분리**: 문서화와 비즈니스 로직 분리

---

## 커스텀 어노테이션 설계

### 🎯 1. 성공 응답용 어노테이션
```java
/**
 * 성공 응답용 커스텀 어노테이션
 * @Operation과 기본 응답 코드들을 묶어서 제공
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Operation
@ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "요청 성공"),
        @ApiResponse(responseCode = "400", description = "잘못된 요청"),
        @ApiResponse(responseCode = "500", description = "서버 오류")
})
public @interface ApiSuccess {
    String summary();
    String description() default "";
}
```

### 🎯 2. 생성 응답용 어노테이션
```java
/**
 * 생성 응답용 커스텀 어노테이션
 * @Operation과 생성 관련 응답 코드들을 묶어서 제공
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Operation
@ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "생성 성공"),
        @ApiResponse(responseCode = "400", description = "잘못된 요청"),
        @ApiResponse(responseCode = "409", description = "리소스 충돌"),
        @ApiResponse(responseCode = "500", description = "서버 오류")
})
public @interface ApiCreate {
    String summary();
    String description() default "";
}
```

### 📊 어노테이션 활용 효과
```java
// Before: 15줄의 어노테이션
@Operation(summary = "잔액 충전", description = "사용자의 잔액을 충전합니다.")
@ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "충전 성공"),
        @ApiResponse(responseCode = "400", description = "잘못된 요청"),
        @ApiResponse(responseCode = "500", description = "서버 오류")
})

// After: 1줄의 어노테이션
@ApiSuccess(summary = "잔액 충전", description = "사용자의 잔액을 충전합니다.")
```

---

## DTO 기반 문서화

### 📥 Request DTO 문서화
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

### 📤 Response DTO 문서화
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

### 🔄 RequestParam → RequestBody 전환
```java
// Before: 파라미터 개별 문서화
public void chargeBalance(
    @Parameter(description = "사용자 ID", required = true, example = "1")
    @NotNull @RequestParam Long userId,
    @Parameter(description = "충전 금액", required = true, example = "10000") 
    @NotNull @DecimalMin(value = "0.0", inclusive = false) @RequestParam BigDecimal amount)

// After: DTO 기반 문서화
public void chargeBalance(@Valid @RequestBody BalanceChargeRequest request)
```

### 📋 장점
1. **문서화 중앙화**: DTO에서 한 번만 정의
2. **타입 안정성**: 컴파일 타임 검증
3. **재사용성**: 같은 DTO를 여러 곳에서 활용
4. **Validation 통합**: Bean Validation과 Swagger 문서화 동시 적용

---

## 최적화 결과

### 🎨 최종 깔끔한 컨트롤러
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

### 📊 비교 결과

| 항목 | Before (초기) | After (최적화) | 개선율 |
|------|---------------|----------------|--------|
| 어노테이션 줄 수 | ~15줄/메서드 | ~1줄/메서드 | **93% 감소** |
| 코드 가독성 | 매우 나쁨 | 매우 좋음 | **대폭 개선** |
| 유지보수성 | 어려움 | 쉬움 | **대폭 개선** |
| 문서 품질 | 분산됨 | 중앙화됨 | **향상** |

### 🎯 각 컨트롤러별 최적화

#### 1. BalanceController
```java
// 2개 메서드 → @ApiSuccess 사용
// RequestParam → RequestBody 전환 (BalanceChargeRequest)
```

#### 2. ProductController
```java
// 2개 메서드 → @ApiSuccess 사용
// 파라미터 문서화 → 메서드 수준에서만 관리
```

#### 3. OrderController
```java
// 주문 생성: @ApiCreate 사용 (201 Created)
// 결제: @ApiSuccess 사용 (200 OK)
// RequestParam → RequestBody 전환 (CreateOrderRequest)
```

#### 4. CouponController
```java
// 2개 메서드 → @ApiSuccess 사용
// RequestParam 방식 유지 (단순한 파라미터)
```

---

## 📚 핵심 학습 포인트

### 1. 최적화 전략
1. **어노테이션 추상화**: 반복 패턴을 커스텀 어노테이션으로
2. **DTO 중심 설계**: 문서화를 DTO에 위임
3. **관심사 분리**: 비즈니스 로직과 문서화 분리
4. **일관성 유지**: 모든 API가 동일한 패턴

### 2. 실무 적용 가이드
- **커스텀 어노테이션**: 팀 표준 응답 패턴 정의
- **DTO 활용**: 요청/응답 스펙을 DTO에 집중
- **예시 데이터**: 실제 사용 가능한 예시 제공
- **그룹화**: @Tag로 API 그룹 관리

### 3. 추가 고려사항
- **보안**: 민감한 정보는 문서화에서 제외
- **버전 관리**: API 버전별 문서 분리
- **환경별 설정**: 개발/운영 환경별 서버 URL 관리
- **인증**: JWT 토큰 기반 인증 문서화

### 4. 성과 측정
- **개발 생산성**: 문서화 시간 단축
- **코드 품질**: 가독성 및 유지보수성 향상
- **팀 협업**: 표준화된 API 문서로 소통 효율성 증대
- **사용자 경험**: 개발자가 API를 쉽게 이해하고 사용 가능

---

## 🎯 결론

Swagger 적용 과정에서 학습한 핵심은 **"도구가 코드를 지배하지 않게 하는 것"**입니다. 

초기에는 Swagger 어노테이션이 컨트롤러를 오염시켰지만, 적절한 추상화와 관심사 분리를 통해 깔끔하고 유지보수 가능한 코드를 만들 수 있었습니다.

이는 실무에서 외부 라이브러리나 프레임워크를 도입할 때 항상 고려해야 할 중요한 원칙입니다. 