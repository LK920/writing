

스프링 프레임워크는 다양한 예외 클래스를 제공하여 특정 오류 상황을 명확히 처리할 수 있도록 설계되었습니다. 이 자료에서는 스프링에서 자주 사용되는 예외 객체들을 빈도에 따라 분류하고, 커스텀 예외 적용 시 사용하지 않을 가능성이 높은 예외들을 설명합니다. 또한, 커스텀 예외 설계 가이드와 예제 코드를 포함합니다.

## 1. 스프링 예외의 특징

스프링의 예외는 주로 `RuntimeException`을 상속받아 비검사 예외(Unchecked Exception)로 설계되었습니다. 이는 개발자가 명시적으로 `try-catch`로 처리하지 않아도 되도록 하여 코드 간결성을 유지합니다. 예외는 계층별(웹, 데이터, 비즈니스 로직 등)로 세분화되어 있으며, 주요 패키지는 다음과 같습니다:

- `org.springframework.core`: 일반적인 예외 (예: `IllegalArgumentException`).
- `org.springframework.web`: HTTP 및 웹 관련 예외.
- `org.springframework.data`: 데이터 접근 관련 예외.
- `org.springframework.security`: 보안 관련 예외.

Express.js의 `throw Error()`와 달리, 스프링은 상황별로 구체적인 예외를 제공하여 오류 원인을 명확히 파악할 수 있도록 합니다.

---

## 2. 자주 사용되는 예외 객체 (빈도순)

아래는 스프링 애플리케이션에서 자주 발생하거나 사용되는 예외들을 빈도에 따라 분류한 목록입니다. 빈도는 일반적인 웹 애플리케이션(REST API, 데이터베이스 연동, 보안 포함) 기준입니다.

### 2.1 매우 자주 사용 (★★★★★)
이 예외들은 대부분의 스프링 프로젝트에서 필수적으로 다뤄집니다.

1. **`IllegalArgumentException`**
   - **설명**: 잘못된 인자 전달 시 발생. 스프링 내부 및 사용자 코드에서 자주 사용.
   - **사용 사례**: 유효하지 않은 입력값(예: null, 잘못된 형식) 처리.
   - **예제**:
     ```java
     if (userId == null) {
         throw new IllegalArgumentException("User ID cannot be null");
     }
     ```
   - **빈도 이유**: 입력 검증 로직에서 기본적으로 사용, 프레임워크 독립적.

2. **`HttpClientErrorException`**
   - **설명**: REST 클라이언트 호출 시 HTTP 4xx 오류(예: 400 Bad Request, 404 Not Found) 발생.
   - **사용 사례**: `RestTemplate` 또는 `WebClient`로 외부 API 호출 시.
   - **예제**:
     ```java
     try {
         restTemplate.getForObject("https://api.example.com/users/1", User.class);
     } catch (HttpClientErrorException e) {
         log.error("HTTP Error: {}", e.getStatusCode());
     }
     ```
   - **빈도 이유**: REST API 기반 프로젝트에서 외부 호출 빈번.

3. **`DataAccessException`**
   - **설명**: 데이터베이스 작업(예: JPA, JDBC) 중 발생하는 예외의 상위 클래스.
   - **하위 클래스**: `DataIntegrityViolationException`, `EmptyResultDataAccessException`.
   - **사용 사례**: JPA 쿼리 실패, 제약 조건 위반 등.
   - **예제**:
     ```java
     try {
         userRepository.findById(id).orElseThrow();
     } catch (DataAccessException e) {
         throw new RuntimeException("Database error", e);
     }
     ```
   - **빈도 이유**: 데이터베이스 연동 필수 프로젝트에서 자주 발생.

4. **`MethodArgumentNotValidException`**
   - **설명**: `@Valid` 또는 `@Validated`로 유효성 검사 실패 시 발생.
   - **사용 사례**: REST API에서 요청 DTO 유효성 검사.
   - **예제**:
     ```java
     @PostMapping("/users")
     public ResponseEntity<?> createUser(@Valid @RequestBody UserDto userDto) {
         // 자동으로 MethodArgumentNotValidException 발생
     }
     ```
   - **빈도 이유**: 입력 데이터 검증은 REST API의 기본 요구사항.

### 2.2 자주 사용 (★★★★☆)
특정 상황에서 빈번히 발생하며, 대부분의 프로젝트에서 다룰 가능성이 높습니다.

1. **`IllegalStateException`**
   - **설명**: 객체의 상태가 작업을 수행하기에 부적절할 때 발생.
   - **사용 사례**: 초기화되지 않은 빈 호출, 잘못된 상태 전이.
   - **예제**:
     ```java
     if (!initialized) {
         throw new IllegalStateException("Service not initialized");
     }
     ```

2. **`HttpServerErrorException`**
   - **설명**: REST 클라이언트 호출 시 HTTP 5xx 오류(예: 500 Internal Server Error) 발생.
   - **사용 사례**: 외부 API 호출 실패 처리.
   - **예제**:
     ```java
     try {
         restTemplate.postForObject("https://api.example.com/users", user, User.class);
     } catch (HttpServerErrorException e) {
         log.error("Server Error: {}", e.getStatusCode());
     }
     ```

3. **`EntityNotFoundException`**
   - **설명**: JPA에서 엔티티를 찾을 수 없을 때 발생(주로 `javax.persistence` 패키지).
   - **사용 사례**: ID로 엔티티 조회 실패.
   - **예제**:
     ```java
     userRepository.findById(id).orElseThrow(() -> new EntityNotFoundException("User not found"));
     ```

4. **`AccessDeniedException`**
   - **설명**: 스프링 시큐리티에서 권한 부족 시 발생.
   - **사용 사례**: 인증된 사용자가 권한 없는 리소스 접근 시.
   - **예제**:
     ```java
     @PreAuthorize("hasRole('ADMIN')")
     public void adminOnly() {
         // AccessDeniedException 자동 발생
     }
     ```

### 2.3 보통 사용 (★★★☆☆)
특정 모듈이나 기능을 사용할 때 주로 발생합니다.

1. **`DataIntegrityViolationException`**
   - **설명**: 데이터베이스 제약 조건(예: 고유 키, 외래 키) 위반 시 발생.
   - **사용 사례**: 중복된 이메일로 사용자 등록 시도.
   - **예제**:
     ```java
     try {
         userRepository.save(user);
     } catch (DataIntegrityViolationException e) {
         throw new RuntimeException("Email already exists", e);
     }
     ```

2. **`NoSuchElementException`**
   - **설명**: `Optional`에서 값을 찾을 수 없을 때 발생.
   - **사용 사례**: `orElseThrow()` 사용 시.
   - **예제**:
     ```java
     User user = userRepository.findById(id).orElseThrow(() -> new NoSuchElementException("User not found"));
     ```

3. **`OptimisticLockException`**
   - **설명**: JPA에서 낙관적 락(Optimistic Locking) 충돌 시 발생.
   - **사용 사례**: 동시 수정 시 버전 충돌.
   - **예제**:
     ```java
     @Entity
     public class User {
         @Version
         private Long version;
     }
     ```

4. **`TransactionException`**
   - **설명**: 트랜잭션 처리 중 오류 발생 시 (예: 롤백 실패).
   - **사용 사례**: `@Transactional` 사용 중 문제.
   - **예제**:
     ```java
     @Transactional
     public void updateUser() {
         // TransactionException 발생 가능
     }
     ```

### 2.4 드물게 사용 (★★☆☆☆)
특정 상황에서만 사용되거나, 특정 모듈에 국한됩니다.

1. **`BeanCreationException`**
   - **설명**: 스프링 빈 생성 실패 시 발생.
   - **사용 사례**: 잘못된 빈 설정.
   - **예제**: 빈 주입 순환 참조 시 발생.

2. **`NonUniqueResultException`**
   - **설명**: JPA 쿼리에서 단일 결과를 기대했으나 여러 결과 반환.
   - **사용 사례**: `getSingleResult()` 호출 시.
   - **예제**:
     ```java
     Query query = entityManager.createQuery("SELECT u FROM User u WHERE u.email = :email");
     query.setParameter("email", "test@example.com");
     query.getSingleResult(); // NonUniqueResultException 발생 가능
     ```

3. **`HttpMessageNotReadableException`**
   - **설명**: HTTP 요청 본문 파싱 실패 시 발생.
   - **사용 사례**: 잘못된 JSON 형식 수신.
   - **예제**:
     ```java
     @PostMapping("/users")
     public void createUser(@RequestBody UserDto userDto) {
         // 잘못된 JSON 입력 시 HttpMessageNotReadableException
     }
     ```

### 2.5 매우 드물게 사용 (★☆☆☆☆)
특화된 모듈이나 극히 드문 상황에서만 발생.

1. **`TransientDataAccessException`**
   - **설명**: 일시적인 데이터베이스 오류(예: 데드락).
   - **사용 사례**: 재시도 로직 구현 시.

2. **`InvalidDataAccessApiUsageException`**
   - **설명**: 데이터 접근 API 잘못 사용 시.
   - **사용 사례**: 잘못된 쿼리 파라미터 전달.

3. **`UncategorizedSQLException`**
   - **설명**: 분류되지 않은 SQL 오류.
   - **사용 사례**: 데이터베이스 특정 오류 처리.

---

## 3. 커스텀 예외(Custom Exception) 적용 시 고려사항

커스텀 예외를 도입하면 스프링의 기본 예외를 직접 사용하지 않고, 도메인에 특화된 예외를 통해 오류를 더 명확히 표현할 수 있습니다. 이는 클린 아키텍처와 같은 설계에서 도메인 계층을 프레임워크로부터 독립적으로 유지하는 데 유용합니다.

### 3.1 커스텀 예외 설계 가이드
1. **도메인 중심 설계**:
   - 비즈니스 로직에 맞는 예외 정의 (예: `UserNotFoundException`, `InvalidOrderException`).
   - 프레임워크 예외를 직접 노출하지 않음.
2. **계층화**:
   - 상위 예외 클래스(예: `DomainException`)를 정의하고, 하위 예외로 세분화.
3. **HTTP 상태 매핑**:
   - REST API에서 적절한 HTTP 상태 코드로 변환 (예: `404 Not Found`, `400 Bad Request`).
4. **명확한 메시지**:
   - 사용자 친화적인 오류 메시지 포함.
5. **재사용성**:
   - 다양한 서비스에서 재사용 가능한 예외 설계.

### 3.2 커스텀 예외 예제
```java
// 상위 커스텀 예외
public class DomainException extends RuntimeException {
    private final String errorCode;

    public DomainException(String message, String errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

    public String getErrorCode() {
        return errorCode;
    }
}

// 하위 예외
public class UserNotFoundException extends DomainException {
    public UserNotFoundException(String userId) {
        super("User not found with ID: " + userId, "USER_NOT_FOUND");
    }
}

public class InvalidOrderException extends DomainException {
    public InvalidOrderException(String message) {
        super(message, "INVALID_ORDER");
    }
}

// 예외 처리 (@ControllerAdvice)
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        ErrorResponse response = new ErrorResponse(ex.getErrorCode(), ex.getMessage());
        return new ResponseEntity<>(response, HttpStatus.NOT_FOUND);
    }

    @ExceptionHandler(InvalidOrderException.class)
    public ResponseEntity<ErrorResponse> handleInvalidOrder(InvalidOrderException ex) {
        ErrorResponse response = new ErrorResponse(ex.getErrorCode(), ex.getMessage());
        return new ResponseEntity<>(response, HttpStatus.BAD_REQUEST);
    }
}

public class ErrorResponse {
    private String errorCode;
    private String message;

    public ErrorResponse(String errorCode, String message) {
        this.errorCode = errorCode;
        this.message = message;
    }
}
```

- **설명**:
  - `DomainException`: 모든 도메인 예외의 상위 클래스.
  - `UserNotFoundException`, `InvalidOrderException`: 특정 도메인 오류에 맞춘 하위 예외.
  - `@ControllerAdvice`: REST API에서 예외를 HTTP 응답으로 변환.

### 3.3 커스텀 예외 적용 시 불필요한 스프링 예외
커스텀 예외를 사용하면 스프링의 세부적인 예외를 직접 노출할 필요가 없어집니다. 아래는 커스텀 예외로 대체 가능해 사용 빈도가 낮아질 수 있는 예외들입니다:

1. **`DataAccessException` 및 하위 클래스** (`DataIntegrityViolationException`, `EmptyResultDataAccessException` 등):
   - **이유**: 데이터베이스 예외를 도메인 예외(예: `UserNotFoundException`, `DuplicateResourceException`)로 변환하여 노출.
   - **대체 예제**:
     ```java
     try {
         userRepository.findById(id).orElseThrow(() -> new UserNotFoundException(id));
     } catch (DataAccessException e) {
         throw new DatabaseOperationException("Database error occurred");
     }
     ```

2. **`HttpClientErrorException`, `HttpServerErrorException`**:
   - **이유**: 외부 API 호출 오류를 도메인 예외(예: `ExternalApiException`)로 추상화.
   - **대체 예제**:
     ```java
     try {
         restTemplate.getForObject(url, User.class);
     } catch (HttpClientErrorException e) {
         throw new ExternalApiException("Failed to call external API: " + e.getStatusCode());
     }
     ```

3. **`NonUniqueResultException`, `InvalidDataAccessApiUsageException`**:
   - **이유**: JPA 쿼리 관련 예외는 도 Hawkins, Thomas J. (2011). "Java Persistence with JPA." O'Reilly Media.  
   - **대체**: `ResourceNotUniqueException`, `InvalidQueryException` 등으로 변환.

4. **`TransactionException`, `OptimisticLockException`**:
   - **이유**: 트랜잭션 관련 예외는 비즈니스 로직에서 직접 처리하지 않고, 어댑터 계층에서 도메인 예외로 변환.
   - **대체 예제**:
     ```java
     @Transactional
     public void updateUser(User user) {
         try {
             userRepository.save(user);
         } catch (OptimisticLockException e) {
             throw new ConcurrentModificationException("Data modified concurrently");
         }
     }
     ```

5. **`HttpMessageNotReadableException`**:
   - **이유**: JSON 파싱 오류를 일반적인 `BadRequestException`으로 대체.
   - **대체 예제**:
     ```java
     @PostMapping("/users")
     public ResponseEntity<?> createUser(@RequestBody UserDto userDto) {
         try {
             // 처리 로직
         } catch (HttpMessageNotReadableException e) {
             throw new BadRequestException("Invalid request format");
         }
     }
     ```

### 3.4 커스텀 예외 적용 시 권장 사항
- **프레임워크 예외 캡슐화**: 스프링 예외를 도메인 계층에서 직접 처리하지 말고, 어댑터 계층에서 커스텀 예외로 변환.
- **의미 있는 이름**: 예외 이름은 도메인 용어로 명확히 (예: `UserNotFoundException` vs `EntityNotFoundException`).
- **HTTP 상태 매핑**: REST API 응답에 적합한 HTTP 상태 코드 정의.
- **일관성 유지**: 모든 예외를 `DomainException` 하위 클래스로 통합 관리.
- **로깅**: 예외 발생 시 상세 로그 기록 (예: `errorCode`, 스택 트레이스).

---

## 4. 예외 처리 모범 사례

1. **중앙화된 예외 처리**:
   - `@ControllerAdvice`를 사용하여 전역 예외 처리.
   - 예:
     ```java
     @RestControllerAdvice
     public class GlobalExceptionHandler {
         @ExceptionHandler(DomainException.class)
         public ResponseEntity<ErrorResponse> handleDomainException(DomainException ex) {
             return new ResponseEntity<>(new ErrorResponse(ex.getErrorCode(), ex.getMessage()), HttpStatus.BAD_REQUEST);
         }
     }
     ```

2. **계층 분리**:
   - 도메인 계층: 커스텀 예외만 사용.
   - 어댑터 계층: 스프링 예외를 커스텀 예외로 변환.
   - 컨트롤러 계층: HTTP 응답 생성.

3. **로그 및 모니터링**:
   - SLF4J, Logback 등으로 예외 로깅.
   - 예:
     ```java
     log.error("Error occurred: {}", ex.getMessage(), ex);
     ```

4. **사용자 친화적 메시지**:
   - 클라이언트에게 노출되는 오류 메시지는 간결하고 명확하게.

---

## 5. 결론

스프링의 예외 객체는 특정 상황에 맞게 세분화되어 있지만, 커스텀 예외를 도입하면 도메인 중심의 명확한 오류 처리가 가능하며, 프레임워크 의존성을 줄일 수 있습니다. 자주 사용되는 예외(`IllegalArgumentException`, `HttpClientErrorException`, `DataAccessException` 등)는 필수적으로 다뤄야 하지만, 커스텀 예외로 대체하면 코드 가독성과 유지보수성이 향상됩니다.

### 요약
- **자주 사용 예외**: `IllegalArgumentException`, `HttpClientErrorException`, `DataAccessException`, `MethodArgumentNotValidException`.
- **커스텀 예외**: 도메인 중심, 프레임워크 예외 캡슐화, HTTP 상태 매핑.
- **불필요 예외**: `DataAccessException`, `HttpClientErrorException` 등은 어댑터 계층에서 변환.
- **모범 사례**: 중앙화된 처리, 계층 분리, 명확한 메시지, 로깅.

이 자료를 통해 스프링 예외를 체계적으로 이해하고, 커스텀 예외를 효과적으로 적용하여 견고한 애플리케이션을 구축할 수 있습니다.