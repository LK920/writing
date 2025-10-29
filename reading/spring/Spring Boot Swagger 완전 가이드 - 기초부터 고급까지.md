# Spring Boot Swagger 완전 가이드 - 기초부터 고급까지

## 📚 목차
1. [Swagger란 무엇인가?](#swagger란-무엇인가)
2. [Spring Boot에서 Swagger 기본 설정](#spring-boot에서-swagger-기본-설정)
3. [기본 어노테이션 사용법](#기본-어노테이션-사용법)
4. [우리 프로젝트의 고급 구현](#우리-프로젝트의-고급-구현)
5. [단계별 적용 가이드](#단계별-적용-가이드)
6. [실제 결과 확인](#실제-결과-확인)

---

## Swagger란 무엇인가?

### 🎯 Swagger의 목적
```
개발자가 API를 만들면 → Swagger가 자동으로 문서를 생성
클라이언트 개발자가 → 웹 브라우저에서 API 문서를 보고 테스트 가능
```

### 📱 Swagger UI 예시
```
http://localhost:8080/api-docs 접속하면:

┌─────────────────────────────────────┐
│ E-Commerce API                      │
│ 항해플러스 이커머스 서비스 API 문서    │
├─────────────────────────────────────┤
│ 📁 잔액 관리                        │
│   POST /api/balance/charge          │
│   GET  /api/balance/{userId}        │
│                                     │
│ 📁 상품 관리                        │
│   GET  /api/product/list            │
│   GET  /api/product/popular         │
└─────────────────────────────────────┘
```

---

## Spring Boot에서 Swagger 기본 설정

### 1. 의존성 추가 (build.gradle.kts)

```kotlin
dependencies {
    // SpringDoc OpenAPI (Swagger) 의존성
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0")
    
    // 기존 Spring Boot 의존성들
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
}
```

**왜 springdoc를 사용하나?**
- 기존 `springfox`는 Spring Boot 3.x와 호환성 문제
- `springdoc`은 최신 Spring Boot와 완벽 호환
- 더 간단한 설정과 더 나은 성능

### 2. application.yml 설정

```yaml
# SpringDoc/Swagger 설정
springdoc:
  api-docs:
    enabled: true                    # API 문서 생성 활성화
    path: /v3/api-docs              # OpenAPI 스펙 JSON 경로
  swagger-ui:
    enabled: true                    # Swagger UI 활성화
    path: /api-docs                 # Swagger UI 접근 경로
    display-request-duration: true   # 요청 시간 표시
    display-operation-id: false      # 오퍼레이션 ID 숨기기
    tags-sorter: alpha              # 태그 알파벳 순 정렬
    operations-sorter: alpha        # 오퍼레이션 알파벳 순 정렬
  show-actuator: false              # Actuator 엔드포인트 숨기기
```

### 3. 최소 설정 클래스

```java
@Configuration
public class SwaggerConfig {
    // 별도 설정 없이도 동작하지만, 커스터마이징을 위해 설정 클래스 생성
}
```

**이것만으로도 기본 Swagger가 동작합니다!**
- `http://localhost:8080/api-docs` → Swagger UI
- `http://localhost:8080/v3/api-docs` → OpenAPI JSON

---

## 기본 어노테이션 사용법

### 1. 컨트롤러 레벨 어노테이션

```java
@Tag(name = "잔액 관리", description = "사용자 잔액 충전 및 조회 API")
@RestController
@RequestMapping("/api/balance")
public class BalanceController {
    // 메서드들...
}
```

**결과**: Swagger UI에서 "잔액 관리" 섹션으로 그룹화됨

### 2. 메서드 레벨 어노테이션

```java
@Operation(
    summary = "잔액 충전",                    // 짧은 제목
    description = "사용자의 잔액을 충전합니다"   // 상세 설명
)
@ApiResponses(value = {
    @ApiResponse(responseCode = "200", description = "충전 성공"),
    @ApiResponse(responseCode = "400", description = "잘못된 요청")
})
@PostMapping("/charge")
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
    // 구현...
}
```

### 3. DTO 어노테이션

```java
@Schema(description = "잔액 충전 요청")
public class BalanceRequest {
    
    @Schema(description = "사용자 ID", example = "1", required = true)
    private Long userId;
    
    @Schema(description = "충전 금액", example = "10000", required = true)
    private BigDecimal amount;
    
    // getter, setter...
}
```

### 4. 기본 사용법의 한계점

```java
// 😰 이런 식으로 모든 메서드마다 반복 작업
@Operation(summary = "잔액 충전")
@ApiResponses(value = {
    @ApiResponse(responseCode = "200", description = "성공"),
    @ApiResponse(responseCode = "400", description = "잘못된 요청"),
    @ApiResponse(responseCode = "402", description = "잔액 부족"),
    @ApiResponse(responseCode = "404", description = "사용자 없음"),
    @ApiResponse(responseCode = "500", description = "서버 오류")
})
@PostMapping("/charge")
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
    // 구현...
}

// 😰 다른 메서드에서도 똑같은 에러 응답들을 반복해서 추가...
@Operation(summary = "잔액 조회")
@ApiResponses(value = {
    @ApiResponse(responseCode = "200", description = "성공"),
    @ApiResponse(responseCode = "400", description = "잘못된 요청"),
    @ApiResponse(responseCode = "404", description = "사용자 없음"),
    @ApiResponse(responseCode = "500", description = "서버 오류")
})
@GetMapping("/{userId}")
public BalanceResponse getBalance(@PathVariable Long userId) {
    // 구현...
}
```

**문제점**:
1. 🔄 **중복 코드**: 같은 에러 응답을 계속 반복 작성
2. 🐛 **누락 위험**: 개발자가 실수로 에러 케이스를 빠뜨릴 수 있음
3. 🔧 **유지보수 어려움**: 에러 메시지 변경 시 모든 곳을 수정해야 함
4. 📝 **예시 데이터 수동 관리**: 실제 DTO와 다른 예시 데이터

---

## 우리 프로젝트의 고급 구현

### 🚀 문제 해결 접근법

우리는 위의 문제들을 다음과 같이 해결했습니다:

```
기본 Swagger 
    ↓
커스텀 어노테이션 시스템 추가
    ↓
자동 에러 응답 생성 시스템
    ↓
DTO 기반 자동 예시 생성
    ↓
완전 자동화된 API 문서
```

### 1. 커스텀 어노테이션 시스템

#### 기존 방식 (반복 코드)
```java
// 😰 매번 이런 긴 어노테이션 작성
@Operation(summary = "잔액 충전")
@ApiResponses(value = {
    @ApiResponse(responseCode = "200", description = "성공"),
    @ApiResponse(responseCode = "402", description = "잔액 부족"),
    @ApiResponse(responseCode = "404", description = "사용자 없음")
})
```

#### 우리 방식 (간단한 어노테이션)
```java
// 😊 간단한 한 줄로 해결
@BalanceApiDocs(summary = "잔액 충전", description = "사용자 잔액을 충전합니다")
```

#### 커스텀 어노테이션 구현
```java
// 1. 어노테이션 정의
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface BalanceApiDocs {
    String summary();
    String description() default "";
}

// 2. 도메인별 어노테이션 생성
@BalanceApiDocs    // 잔액 관련 API용
@CouponApiDocs     // 쿠폰 관련 API용
@OrderApiDocs      // 주문 관련 API용
@ProductApiDocs    // 상품 관련 API용
```

### 2. 자동 에러 응답 생성 시스템

#### 에러 정보 중앙 관리
```java
public class ErrorSchemas {
    // 잔액 도메인의 모든 에러를 한 곳에서 관리
    public static final Map<Class<? extends Exception>, ErrorInfo> BALANCE_ERRORS = Map.of(
        BalanceException.InsufficientBalance.class,
        new ErrorInfo(402, "INSUFFICIENT_BALANCE", "잔액이 부족합니다"),
        
        BalanceException.NotFound.class,
        new ErrorInfo(404, "BALANCE_NOT_FOUND", "잔액 정보를 찾을 수 없습니다")
    );
}
```

#### 자동 추가 시스템
```java
@Component
public class SwaggerResponseCustomizer implements OperationCustomizer {
    
    @Override
    public Operation customize(Operation operation, HandlerMethod handlerMethod) {
        // 1. 메서드가 어떤 도메인인지 자동 감지
        String domain = extractDomainFromMethod(handlerMethod);
        
        // 2. 해당 도메인의 에러들을 자동으로 Swagger에 추가
        if ("balance".equals(domain)) {
            addBalanceErrors(operation);
        } else if ("coupon".equals(domain)) {
            addCouponErrors(operation);
        }
        
        return operation;
    }
}
```

### 3. DTO 기반 자동 예시 생성

#### DocumentedDto 인터페이스
```java
public interface DocumentedDto {
    @JsonIgnore  // JSON 직렬화에서 제외
    Map<String, SchemaInfo> getFieldDocumentation();
}
```

#### DTO 구현 예시
```java
public record BalanceResponse(
    Long userId,
    BigDecimal amount,
    LocalDateTime updatedAt
) implements DocumentedDto {
    
    @Override
    public Map<String, SchemaInfo> getFieldDocumentation() {
        return Map.of(
            "userId", new SchemaInfo("사용자 ID", "1"),
            "amount", new SchemaInfo("현재 잔액", "50000"),
            "updatedAt", new SchemaInfo("업데이트 시간", "2024-01-01T12:00:00")
        );
    }
}
```

#### 자동 예시 생성 시스템
```java
@Component
public class SwaggerSuccessResponseCustomizer implements OperationCustomizer {
    
    @Override
    public Operation customize(Operation operation, HandlerMethod handlerMethod) {
        // 1. 메서드의 리턴 타입 확인
        Class<?> returnType = handlerMethod.getReturnType().getParameterType();
        
        // 2. DTO의 getFieldDocumentation() 정보를 활용해서 실제 예시 생성
        if (DocumentedDto.class.isAssignableFrom(returnType)) {
            DocumentedDto dto = createInstance(returnType);
            Map<String, Object> exampleData = generateFromDocumentation(dto);
            addSuccessResponse(operation, exampleData);
        }
        
        return operation;
    }
}
```

---

## 단계별 적용 가이드

### Phase 1: 기본 Swagger 설정 (5분)

#### 1. 의존성 추가
```kotlin
// build.gradle.kts
implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.2.0")
```

#### 2. application.yml 설정
```yaml
springdoc:
  api-docs:
    enabled: true
    path: /v3/api-docs
  swagger-ui:
    enabled: true
    path: /api-docs
```

#### 3. 확인
- 애플리케이션 실행
- `http://localhost:8080/api-docs` 접속
- 기본 API 문서 확인

### Phase 2: 기본 어노테이션 적용 (30분)

#### 1. 컨트롤러에 @Tag 추가
```java
@Tag(name = "잔액 관리", description = "사용자 잔액 관련 API")
@RestController
public class BalanceController {
```

#### 2. 메서드에 @Operation 추가
```java
@Operation(summary = "잔액 충전", description = "사용자 잔액을 충전합니다")
@PostMapping("/charge")
public BalanceResponse chargeBalance(@RequestBody BalanceRequest request) {
```

#### 3. DTO에 @Schema 추가
```java
@Schema(description = "잔액 충전 요청")
public class BalanceRequest {
    @Schema(description = "사용자 ID", example = "1")
    private Long userId;
```

### Phase 3: 고급 자동화 시스템 구현 (2-3시간)

#### 1. docs 패키지 구조 생성
```
src/main/java/kr/hhplus/be/server/api/docs/
├── annotation/     # 커스텀 어노테이션들
├── schema/        # 에러 매핑, DTO 인터페이스
└── config/        # 자동화 시스템
```

#### 2. 커스텀 어노테이션 생성
```java
// docs/annotation/BalanceApiDocs.java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface BalanceApiDocs {
    String summary();
    String description() default "";
}
```

#### 3. 에러 스키마 정의
```java
// docs/schema/ErrorSchemas.java
public class ErrorSchemas {
    public static final Map<Class<? extends Exception>, ErrorInfo> BALANCE_ERRORS = Map.of(
        // 에러 매핑 정의
    );
}
```

#### 4. 커스터마이저 구현
```java
// docs/config/SwaggerResponseCustomizer.java
@Component
public class SwaggerResponseCustomizer implements OperationCustomizer {
    // 자동 에러 응답 추가 로직
}
```

#### 5. Swagger 설정 통합
```java
// docs/config/SwaggerConfig.java
@Configuration
public class SwaggerConfig {
    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
                .addOperationCustomizer(responseCustomizer)
                .build();
    }
}
```

### Phase 4: DTO 표준화 (1시간)

#### 1. DocumentedDto 인터페이스 구현
```java
public record BalanceResponse(...) implements DocumentedDto {
    @Override
    public Map<String, SchemaInfo> getFieldDocumentation() {
        return Map.of(
            "userId", new SchemaInfo("사용자 ID", "1"),
            "amount", new SchemaInfo("현재 잔액", "50000")
        );
    }
}
```

#### 2. 모든 DTO에 적용
- Request DTO: 4개 적용
- Response DTO: 5개 적용

---

## 실제 결과 확인

### 적용 전 vs 적용 후

#### 적용 전 (기본 Swagger)
```yaml
# 부족한 문서
/api/balance/charge:
  post:
    summary: chargeBalance  # 메서드명 그대로
    responses:
      "200":
        description: OK     # 기본 설명
```

#### 적용 후 (우리 시스템)
```yaml
# 완전한 문서
/api/balance/charge:
  post:
    tags: ["잔액 관리"]
    summary: "잔액 충전"
    description: "사용자의 잔액을 충전합니다"
    responses:
      "200":
        description: "요청 성공"
        content:
          application/json:
            example:
              success: true
              message: "요청 성공"
              data:
                userId: 1
                amount: 50000
                updatedAt: "2024-01-01T12:00:00"
      "400":
        description: "잘못된 요청"
        content:
          application/json:
            example:
              success: false
              errorCode: "INVALID_REQUEST"
              message: "잘못된 요청입니다"
      "402":
        description: "잔액 부족"
        content:
          application/json:
            example:
              success: false
              errorCode: "INSUFFICIENT_BALANCE"
              message: "잔액이 부족합니다"
      "404":
        description: "잔액 정보 없음"
        content:
          application/json:
            example:
              success: false
              errorCode: "BALANCE_NOT_FOUND"
              message: "잔액 정보를 찾을 수 없습니다"
      "500":
        description: "서버 오류"
        content:
          application/json:
            example:
              success: false
              errorCode: "INTERNAL_SERVER_ERROR"
              message: "서버 내부 오류가 발생했습니다"
```

### Swagger UI에서 확인할 수 있는 것들

#### 1. 완전한 API 문서
- ✅ 모든 엔드포인트 자동 문서화
- ✅ 도메인별 그룹화 (잔액, 쿠폰, 주문, 상품)
- ✅ 상세한 설명과 예시

#### 2. 실제 사용 가능한 예시 데이터
- ✅ 실제 DTO 구조와 일치하는 요청/응답 예시
- ✅ 복사해서 바로 사용할 수 있는 JSON 데이터

#### 3. 완전한 에러 처리 문서
- ✅ 모든 HTTP 상태 코드 문서화
- ✅ 도메인별 특화된 에러 응답
- ✅ 실제 에러 응답 형식과 일치

#### 4. 테스트 기능
- ✅ Swagger UI에서 직접 API 테스트 가능
- ✅ 요청 파라미터 자동 완성
- ✅ 실시간 응답 확인

---

## 🎯 핵심 차별점

### 기본 Swagger vs 우리 시스템

| 항목 | 기본 Swagger | 우리 시스템 |
|------|-------------|------------|
| **설정 복잡도** | 간단 | 초기 설정 후 간단 |
| **문서화 완성도** | 부분적 | 100% 완성 |
| **개발자 작업량** | 많음 (반복 작업) | 최소 (어노테이션 1개) |
| **에러 문서화** | 수동 추가 | 자동 생성 |
| **예시 데이터** | 수동 관리 | 자동 생성 |
| **일관성** | 개발자마다 다름 | 항상 일관됨 |
| **유지보수** | 어려움 | 쉬움 |

### 실제 개발 워크플로우

#### 기본 Swagger 방식
```
1. 새 API 메서드 작성
2. @Operation 어노테이션 추가
3. @ApiResponses로 모든 에러 케이스 수동 추가
4. 각 응답마다 예시 데이터 수동 작성
5. DTO 변경 시 예시 데이터도 수동 수정
6. 실수로 누락된 에러 케이스 있을 수 있음
```

#### 우리 시스템 방식
```
1. 새 API 메서드 작성
2. @BalanceApiDocs(summary = "...") 추가
3. 끝! (나머지는 모두 자동)
```

---

## 🚀 확장 가능성

### 현재 구현된 기능
- ✅ 도메인별 자동 에러 응답 (Balance, Coupon, Order, Product)
- ✅ DTO 기반 자동 예시 생성
- ✅ HTTP 상태 코드별 체계적 문서화
- ✅ 중앙화된 에러 관리

### 향후 확장 가능한 기능
- 🔄 **API 버전 관리**: v1, v2 API 자동 분리
- 🔐 **인증/권한 문서화**: JWT, OAuth 등 보안 관련 자동 문서화
- 📊 **성능 메트릭**: 응답 시간, 처리량 등 자동 추가
- 🧪 **테스트 케이스 생성**: Swagger 문서에서 자동 테스트 코드 생성
- 🌐 **다국어 지원**: 영어, 한국어 문서 자동 생성

---

## 📝 정리

Spring Boot에서 Swagger를 사용하는 방법:

1. **기본 단계**: 의존성 추가 → 설정 → 기본 어노테이션
2. **고급 단계**: 커스텀 어노테이션 → 자동화 시스템 → 완전 자동화

우리 프로젝트는 **기본 Swagger의 한계를 극복하고 완전 자동화된 API 문서화 시스템**을 구축했습니다.

**결과**: 개발자는 비즈니스 로직에만 집중하고, API 문서화는 시스템이 자동으로 처리합니다! 🎉

---
*🤖 Generated with [Claude Code](https://claude.ai/code)*

*Co-Authored-By: Claude <noreply@anthropic.com>*