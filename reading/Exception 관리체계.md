아래는 Java Spring 환경과 클린 아키텍처 원칙을 기반으로 한 **예외 관리 모범 사례 규정**입니다. 이 규정은 `id`뿐만 아니라 데이터 예외 처리 전반에 적용할 수 있도록 설계되었으며, 일관성을 유지하고 유지보수성을 높이기 위해 구체적인 가이드라인과 예제를 포함합니다. 규정은 팀 내에서 공유하고 코드 리뷰 및 개발 과정에서 준수하도록 권장할 수 있습니다.

---

# 예외 관리 모범 사례 규정

## 1. 목적
- 예외 처리를 일관성 있게 관리하여 코드의 가독성, 유지보수성, 디버깅 용이성을 높인다.
- 클린 아키텍처와 Spring 프레임워크의 특성을 반영하여 도메인 비즈니스 로직과 프레젠테이션 레이어를 명확히 분리한다.
- 클라이언트가 API 응답을 통해 예외의 원인을 명확히 파악할 수 있도록 지원한다.

## 2. 범위
- 모든 도메인(예: User, Balance, Order 등)에서 발생할 수 있는 데이터 관련 예외 처리.
- 컨트롤러, 서비스(UseCase), 리포지토리 레이어에서의 예외 처리.

## 3. 기본 원칙
- **도메인 중심**: 예외는 해당 도메인에 특화된 클래스(예: `UserException`)로 정의한다.
- **세분화**: 동일한 예외 원인을 포괄적으로 처리하지 않고, 상황에 따라 구체적으로 분리한다.
- **HTTP 상태 코드 준수**: REST API 표준에 맞는 HTTP 상태 코드를 반환한다.
- **중앙 관리**: 메시지와 에러 코드는 중앙화된 방식으로 관리한다.
- **체이닝 지원**: 원인 예외를 포함하여 디버깅을 용이하게 한다.

## 4. 예외 분류 및 정의
### 4.1. 기본 예외 클래스
모든 도메인 예외는 `BaseBusinessException`을 상속받아야 하며, 공통 속성과 메서드를 제공한다.

```java
public abstract class BaseBusinessException extends RuntimeException {
    private final String errorCode;
    private final String domain;
    private final LocalDateTime occurredAt;

    protected BaseBusinessException(String domain, String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.domain = domain;
        this.errorCode = errorCode;
        this.occurredAt = LocalDateTime.now();
    }

    public String getErrorCode() { return errorCode; }
    public String getDomain() { return domain; }
    public LocalDateTime getOccurredAt() { return occurredAt; }
    public abstract HttpStatus getHttpStatus();
}
```

### 4.2. 데이터 예외 분류
데이터 관련 예외는 다음과 같이 세분화하여 정의한다:
- **입력 검증 실패**: 데이터가 누락되었거나 형식적으로 잘못된 경우.
- **비즈니스 규칙 위반**: 도메인 내 비즈니스 로직에 부합하지 않는 값.
- **데이터 없음**: 요청된 데이터가 존재하지 않는 경우.
- **기술적 오류**: 데이터베이스, 네트워크 등 외부 시스템 문제.

#### 4.2.1. 입력 검증 실패
- **상황**: `null`, 빈 문자열, 잘못된 형식(예: 문자열로 된 숫자).
- **예외 클래스**: `ValidationException` 또는 도메인별 하위 예외(예: `UserValidationException`).
- **HTTP 상태 코드**: `400 Bad Request`.

```java
public class UserException extends BaseBusinessException {
    public static class NullIdException extends UserException {
        public NullIdException() {
            super("USER", "ERR_USER_ID_NULL", "사용자 ID는 null일 수 없습니다.", null);
        }
        @Override public HttpStatus getHttpStatus() { return HttpStatus.BAD_REQUEST; }
    }

    public static class InvalidFormatException extends UserException {
        public InvalidFormatException(String field, String value) {
            super("USER", "ERR_USER_INVALID_FORMAT", String.format("%s 필드(%s)의 형식이 잘못되었습니다.", field, value), null);
        }
        @Override public HttpStatus getHttpStatus() { return HttpStatus.BAD_REQUEST; }
    }
}
```

#### 4.2.2. 비즈니스 규칙 위반
- **상황**: 음수/0, 허용 범위 초과, 도메인 고유 규칙 위반.
- **예외 클래스**: `BusinessRuleException` 또는 도메인별 하위 예외(예: `UserBusinessRuleException`).
- **HTTP 상태 코드**: `400 Bad Request`.

```java
public class UserException extends BaseBusinessException {
    public static class InvalidIdValueException extends UserException {
        public InvalidIdValueException(String reason) {
            super("USER", "ERR_USER_ID_INVALID", "사용자 ID가 유효하지 않습니다: " + reason, null);
        }
        @Override public HttpStatus getHttpStatus() { return HttpStatus.BAD_REQUEST; }
    }
}
```

#### 4.2.3. 데이터 없음
- **상황**: 데이터베이스에서 조회되지 않음, 만료된 데이터.
- **예외 클래스**: `EntityNotFoundException` 또는 도메인별 하위 예외(예: `UserNotFoundException`).
- **HTTP 상태 코드**: `404 Not Found`.

```java
public class UserException extends BaseBusinessException {
    public static class EntityNotFoundException extends UserException {
        public EntityNotFoundException() {
            super("USER", "ERR_USER_NOT_FOUND", "해당 사용자를 찾을 수 없습니다.", null);
        }
        @Override public HttpStatus getHttpStatus() { return HttpStatus.NOT_FOUND; }
    }
}
```

#### 4.2.4. 기술적 오류
- **상황**: 데이터베이스 연결 실패, 외부 API 호출 실패.
- **예외 클래스**: `TechnicalException` 또는 도메인별 하위 예외(예: `UserTechnicalException`).
- **HTTP 상태 코드**: `500 Internal Server Error`.

```java
public class UserException extends BaseBusinessException {
    public static class DatabaseConnectionException extends UserException {
        public DatabaseConnectionException(Throwable cause) {
            super("USER", "ERR_USER_DB_CONNECTION", "데이터베이스 연결에 실패했습니다.", cause);
        }
        @Override public HttpStatus getHttpStatus() { return HttpStatus.INTERNAL_SERVER_ERROR; }
    }
}
```

## 5. 예외 처리 과정
### 5.1. 컨트롤러 레이어
- Bean Validation(`@Valid`)을 사용하여 입력 검증을 수행.
- 유효성 검사 실패 시 `MethodArgumentNotValidException`을 처리.
- 도메인 로직으로 전달하기 전에 기본 검증(예: `null` 체크)을 수행.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final GetUserUseCase getUserUseCase;

    @Autowired
    public UserController(GetUserUseCase getUserUseCase) {
        this.getUserUseCase = getUserUseCase;
    }

    @GetMapping("/{id}")
    public ResponseEntity<CommonResponse<UserResponse>> getUser(@PathVariable @NotNull Long id) {
        UserResponse response = getUserUseCase.execute(id);
        return ResponseEntity.ok(CommonResponse.<UserResponse>builder()
            .success(true)
            .data(response)
            .build());
    }
}
```

### 5.2. UseCase/서비스 레이어
- 비즈니스 규칙을 검증하고, 위반 시 적절한 예외를 던짐.
- Repository 호출 후 데이터 없음을 확인하고 `EntityNotFoundException`을 던짐.

```java
public class GetUserUseCase {
    private final UserRepository repository;

    @Autowired
    public GetUserUseCase(UserRepository repository) {
        this.repository = repository;
    }

    public UserResponse execute(Long id) {
        if (id <= 0) {
            throw new UserException.InvalidIdValueException("ID는 0 또는 음수가 될 수 없습니다.");
        }
        User user = repository.findById(id)
            .orElseThrow(UserException.EntityNotFoundException::new);
        return new UserResponse(user.getId(), user.getName());
    }
}
```

### 5.3. Repository 레이어
- 데이터 조회 실패 시 `EntityNotFoundException`을 던짐.
- 기술적 오류(예: SQL 예외)는 래핑하여 상위 레이어로 전달.

```java
public interface UserRepository {
    Optional<User> findById(Long id) throws DataAccessException;
}
```

### 5.4. 전역 예외 처리
- `@RestControllerAdvice`를 사용하여 모든 예외를 중앙에서 처리.
- `errorCode`와 메시지를 `CommonResponse`에 포함.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BaseBusinessException.class)
    public ResponseEntity<CommonResponse<Object>> handleBusinessException(BaseBusinessException ex) {
        return ResponseEntity
            .status(ex.getHttpStatus())
            .body(CommonResponse.builder()
                .success(false)
                .message(ex.getMessage())
                .errorCode(ex.getErrorCode())
                .timestamp(LocalDateTime.now())
                .build());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<CommonResponse<Object>> handleValidationException(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(CommonResponse.builder()
                .success(false)
                .message("유효성 검사 실패: " + message)
                .errorCode("ERR_VALIDATION")
                .timestamp(LocalDateTime.now())
                .build());
    }
}
```

## 6. 메시지 관리
- 에러 메시지는 `MessageSource`를 사용해 중앙화하고, 국제화(i18n)를 지원.
- `messages.properties`에 정의.

```properties
# messages.properties
ERR_USER_ID_NULL=사용자 ID는 null일 수 없습니다.
ERR_USER_ID_INVALID=사용자 ID가 유효하지 않습니다: {0}
ERR_USER_NOT_FOUND=해당 사용자를 찾을 수 없습니다.
```

## 7. 모니터링 및 로깅
- 예외 발생 시 `ExceptionMonitor`를 통해 메트릭 수집.
- 로깅 프레임워크(SLF4J 등)를 사용해 디버깅 정보 기록.

```java
@Component
public class ExceptionMonitor {
    private final MeterRegistry meterRegistry;

    @Autowired
    public ExceptionMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordException(String domain, String errorCode) {
        Counter.builder("application.exception")
            .tag("domain", domain)
            .tag("error_code", errorCode)
            .register(meterRegistry)
            .increment();
    }
}
```

## 8. 테스트 가이드라인
- 단위 테스트에서 각 예외 상황을 커버.
- `@ParameterizedTest`와 `MethodSource`를 사용해 다양한 입력값 테스트.

```java
@ExtendWith(MockitoExtension.class)
class UserControllerTest {
    @Mock
    private GetUserUseCase getUserUseCase;
    private UserController userController;

    @BeforeEach
    void setUp() {
        userController = new UserController(getUserUseCase);
    }

    public static Stream<Arguments> provideInvalidIds() {
        return Stream.of(
            Arguments.of(null, UserException.NullIdException.class),
            Arguments.of(-1L, UserException.InvalidIdValueException.class),
            Arguments.of(0L, UserException.InvalidIdValueException.class)
        );
    }

    @ParameterizedTest
    @MethodSource("provideInvalidIds")
    void getUser_WithInvalidIds(Long id, Class<? extends Exception> expectedException) {
        assertThatThrownBy(() -> userController.getUser(id))
            .isInstanceOf(expectedException);
    }
}
```

## 9. 준수 사항
- 모든 개발자는 이 규정을 준수하며 코드를 작성.
- 새로운 예외가 필요할 경우, 기존 분류에 맞춰 정의하고 팀 리뷰를 거침.
- 예외 메시지와 `errorCode`는 중복되지 않도록 관리.

## 10. 업데이트 및 피드백
- 이 규정은 프로젝트 진행 상황에 따라 정기적으로 검토(분기별)하며 업데이트.
- 팀원 피드백은 `issue` 트래커에 제출하여 반영.

---

### 추가 설명
- **유연성**: 이 규정은 도메인별로 세부 예외를 확장할 수 있도록 설계되었으며, 필요에 따라 하위 클래스를 추가 가능.
- **실행 계획**: 처음에는 핵심 도메인(예: `User`, `Balance`)에 적용하고, 점진적으로 확장.
- **도구 지원**: Spring Boot Actuator, Micrometer, SLF4J를 활용해 모니터링과 로깅을 강화.

이 규정을 기반으로 프로젝트에 맞게 조정하고, 팀과 함께 논의하며 적용해 보세요. 추가적인 세부 사항이나 특정 도메인에 대한 예제가 필요하면 말씀해 주세요!