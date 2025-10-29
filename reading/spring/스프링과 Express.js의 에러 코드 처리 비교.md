# 스프링과 Express.js의 에러 코드 처리 비교 및 커스텀 에러 설계

Express.js에서 커스텀 에러를 처리할 때 임의의 숫자 코드(예: `10001`)를 사용하는 방식과, 스프링에서 HTTP 상태 코드를 기반으로 에러를 처리하는 방식은 각각의 프레임워크 철학과 사용 패턴에서 차이가 발생합니다. 이 자료에서는 두 접근법의 차이, 스프링의 HTTP 상태 코드 사용 이유, 커스텀 에러 코드 설계 가이드, 그리고 실제 예제를 다룹니다.

## 1. Express.js의 커스텀 에러 코드 (예: ErrorCode 10001)

Express.js는 Node.js 기반의 경량 프레임워크로, 에러 처리에 대한 엄격한 규약이 없어 개발자가 자유롭게 에러 코드를 정의할 수 있습니다. 일반적으로 임의의 숫자 코드(예: `10001`, `10002`)를 사용해 특정 에러를 구분합니다.

### 1.1 Express.js 에러 코드 특징
- **임의의 숫자 코드**:
  - 개발자가 도메인별로 고유한 에러 코드를 정의 (예: `10001`은 `UserNotFound`, `10002`는 `InvalidInput`).
  - HTTP 상태 코드와 별개로 동작, 클라이언트에 커스텀 응답 구조로 전달.
- **유연성**:
  - 프레임워크가 강제하는 규약이 없어 자유롭게 설계 가능.
  - 예: `{ errorCode: 10001, message: "User not found" }`.
- **구현 예제**:
  ```javascript
  class CustomError extends Error {
      constructor(errorCode, message) {
          super(message);
          this.errorCode = errorCode;
      }
  }

  app.get('/users/:id', (req, res) => {
      try {
          throw new CustomError(10001, 'User not found');
      } catch (err) {
          res.status(404).json({ errorCode: err.errorCode, message: err.message });
      }
  });
  ```
- **사용 사례**:
  - 특정 비즈니스 로직 오류를 세밀하게 구분 (예: `10001`은 사용자 없음, `10002`는 중복 이메일).
  - 클라이언트가 에러 코드를 기반으로 특정 동작 수행 (예: 특정 UI 표시).

### 1.2 장점
- **세밀한 에러 구분**: HTTP 상태 코드(예: 400)로는 표현하기 어려운 도메인별 오류를 구체적으로 정의.
- **클라이언트 친화적**: 클라이언트가 에러 코드로 특정 오류 처리 로직 구현 가능.
- **프레임워크 독립성**: HTTP 상태 코드에 얽매이지 않음.

### 1.3 단점
- **비표준화**: 팀마다 다른 코드 체계로 인해 혼란 가능.
- **관리 복잡성**: 에러 코드가 많아지면 문서화 및 유지보수 부담 증가.
- **HTTP 표준과의 불일치**: HTTP 상태 코드를 무시하면 RESTful 원칙 위배 가능.

---

## 2. 스프링의 HTTP 상태 코드 기반 에러 처리

스프링은 RESTful API 설계를 강조하며, HTTP 상태 코드를 에러 처리의 기본으로 사용합니다. 스프링의 예외는 HTTP 상태 코드(예: 400, 404, 500)와 매핑되어 클라이언트에 표준화된 응답을 제공합니다.

### 2.1 스프링 에러 처리 특징
- **HTTP 상태 코드 활용**:
  - 스프링은 `HttpStatus` 열거형을 사용해 표준 HTTP 상태 코드(예: `404 Not Found`, `400 Bad Request`)와 예외를 매핑.
  - 예: `HttpClientErrorException`은 4xx, `HttpServerErrorException`은 5xx와 연계.
- **프레임워크 통합**:
  - `@ControllerAdvice`와 `@ExceptionHandler`를 통해 예외를 HTTP 응답으로 변환.
  - 기본 예외(예: `MethodArgumentNotValidException`)는 자동으로 HTTP 상태 코드에 매핑.
- **구현 예제**:
  ```java
  @RestController
  public class UserController {
      @GetMapping("/users/{id}")
      public User getUser(@PathVariable Long id) {
          return userRepository.findById(id)
              .orElseThrow(() -> new EntityNotFoundException("User not found"));
      }
  }

  @RestControllerAdvice
  public class GlobalExceptionHandler {
      @ExceptionHandler(EntityNotFoundException.class)
      public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
          ErrorResponse response = new ErrorResponse("USER_NOT_FOUND", ex.getMessage());
          return new ResponseEntity<>(response, HttpStatus.NOT_FOUND);
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
  - **응답 예시**:
    ```json
    {
      "errorCode": "USER_NOT_FOUND",
      "message": "User not found"
    }
    ```
    HTTP 상태 코드: `404 Not Found`.

- **사용 사례**:
  - REST API에서 표준 HTTP 상태 코드를 사용해 클라이언트와 통신.
  - 예: `404`는 리소스 없음, `400`는 잘못된 요청, `500`은 서버 오류.

### 2.2 왜 HTTP 상태 코드를 사용하는가?
1. **RESTful 원칙 준수**:
   - REST API는 HTTP 프로토콜의 상태 코드를 활용해 클라이언트와 서버 간 오류 상태를 명확히 전달.
   - 예: `404`는 클라이언트가 찾는 리소스가 없음을 명확히 표현.
2. **표준화**:
   - HTTP 상태 코드는 전 세계적으로 표준화된 오류 체계로, 클라이언트(브라우저, 모바일 앱 등)가 이해하기 쉬움.
   - 커스텀 숫자 코드(예: `10001`)는 문서화 없이는 의미 파악 어려움.
3. **프레임워크 통합**:
   - 스프링의 `ResponseEntity`, `@ResponseStatus`, `@ControllerAdvice`는 HTTP 상태 코드와 긴밀히 통합.
   - 예: `MethodArgumentNotValidException`은 자동으로 `400 Bad Request`로 매핑.
4. **클라이언트 친화성**:
   - HTTP 상태 코드는 클라이언트가 표준 방식으로 오류 처리 가능 (예: `axios`에서 `response.status` 확인).
5. **간결성**:
   - 별도의 에러 코드 체계 관리 없이 HTTP 상태 코드로 충분한 정보 전달 가능.

### 2.3 장점
- **표준화**: HTTP 상태 코드는 REST API에서 널리 인정됨.
- **간결한 설계**: 추가적인 커스텀 코드 체계 관리 불필요.
- **클라이언트 호환성**: 브라우저, API 클라이언트가 즉시 이해 가능.
- **프레임워크 지원**: 스프링의 예외 처리 메커니즘이 HTTP 상태 코드와 통합.

### 2.4 단점
- **세밀성 부족**: HTTP 상태 코드(예: 400)는 다양한 오류를 포괄하므로 구체적인 오류 구분 어려움.
  - 예: `400 Bad Request`는 입력 오류, 형식 오류, 유효성 검사 실패 등을 모두 포함.
- **도메인 특화 한계**: 비즈니스 로직별 세부 오류(예: `DuplicateEmailException`)는 추가 정의 필요.
- **복잡한 매핑**: 커스텀 예외와 HTTP 상태 코드를 매핑하는 로직 필요.

---

## 3. 스프링에서 커스텀 에러 코드와 HTTP 상태 코드 결합

Express.js처럼 임의의 숫자 코드를 사용하던 방식과 스프링의 HTTP 상태 코드 기반 방식을 결합하여, 도메인 특화된 커스텀 에러 코드를 유지하면서 RESTful 원칙을 준수할 수 있습니다.

### 3.1 설계 원칙
1. **도메인 중심 에러 코드**:
   - 비즈니스 로직에 맞는 에러 코드 정의 (예: `USER_NOT_FOUND`, `INVALID_ORDER`).
   - 숫자 코드(예: `10001`) 대신 문자열 코드 사용 권장 (가독성, 유지보수성 향상).
2. **HTTP 상태 코드 매핑**:
   - 각 커스텀 에러 코드를 적절한 HTTP 상태 코드에 매핑.
   - 예: `USER_NOT_FOUND` → `404 Not Found`, `INVALID_INPUT` → `400 Bad Request`.
3. **일관된 응답 구조**:
   - 클라이언트에 반환되는 응답은 `{ errorCode, message }` 형식으로 통일.
4. **중앙화된 처리**:
   - `@ControllerAdvice`로 모든 예외를 중앙에서 처리.
5. **문서화**:
   - 에러 코드와 HTTP 상태 코드를 API 문서(Swagger, OpenAPI)에 명시.

### 3.2 구현 예제
```java
// 커스텀 예외 상위 클래스
public abstract class DomainException extends RuntimeException {
    private final String errorCode;
    private final HttpStatus httpStatus;

    public DomainException(String errorCode, String message, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
    }

    public String getErrorCode() {
        return errorCode;
    }

    public HttpStatus getHttpStatus() {
        return httpStatus;
    }
}

// 구체적인 커스텀 예외
public class UserNotFoundException extends DomainException {
    public UserNotFoundException(String userId) {
        super("USER_NOT_FOUND", "User not found with ID: " + userId, HttpStatus.NOT_FOUND);
    }
}

public class InvalidOrderException extends DomainException {
    public InvalidOrderException(String message) {
        super("INVALID_ORDER", message, HttpStatus.BAD_REQUEST);
    }
}

// 전역 예외 처리
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ErrorResponse> handleDomainException(DomainException ex) {
        ErrorResponse response = new ErrorResponse(ex.getErrorCode(), ex.getMessage());
        return new ResponseEntity<>(response, ex.getHttpStatus());
    }

    // 스프링 기본 예외 처리
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getAllErrors().get(0).getDefaultMessage();
        return new ResponseEntity<>(new ErrorResponse("INVALID_INPUT", message), HttpStatus.BAD_REQUEST);
    }
}

public class ErrorResponse {
    private String errorCode;
    private String message;

    public ErrorResponse(String errorCode, String message) {
        this.errorCode = errorCode;
        this.message = message;
    }

    // Getters
}

// 서비스 계층
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public User getUser(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id.toString()));
    }
}

// 컨트롤러
@RestController
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.getUser(id));
    }
}
```

- **응답 예시**:
  ```json
  {
    "errorCode": "USER_NOT_FOUND",
    "message": "User not found with ID: 123"
  }
  ```
  HTTP 상태 코드: `404 Not Found`.

- **설명**:
  - `DomainException`: 에러 코드와 HTTP 상태 코드를 포함.
  - `UserNotFoundException`, `InvalidOrderException`: 도메인 특화 예외.
  - `@ControllerAdvice`: 중앙화된 예외 처리로 HTTP 상태 코드와 통합.
  - 클라이언트는 `errorCode`로 세부 오류 구분, HTTP 상태 코드로 RESTful 응답 처리.

### 3.3 Express.js 스타일 숫자 코드 사용
스프링에서도 Express.js처럼 숫자 코드를 사용할 수 있습니다. 단, HTTP 상태 코드와의 매핑을 명확히 정의해야 합니다.

```java
public class UserNotFoundException extends DomainException {
    public UserNotFoundException(String userId) {
        super("10001", "User not found with ID: " + userId, HttpStatus.NOT_FOUND);
    }
}

public class InvalidOrderException extends DomainException {
    public InvalidOrderException(String message) {
        super("10002", message, HttpStatus.BAD_REQUEST);
    }
}
```

- **장점**:
  - Express.js와 유사한 코드 체계로 기존 개발자 친화적.
  - 숫자 코드로 세부 오류 구분 가능.
- **단점**:
  - 숫자 코드의 의미를 문서화하지 않으면 혼란 초래.
  - HTTP 상태 코드와 중복되는 정보로 복잡성 증가 가능.

### 3.4 권장: 문자열 코드 사용
숫자 코드(예: `10001`) 대신 문자열 코드(예: `USER_NOT_FOUND`)를 사용하는 이유:
- **가독성**: `10001`보다 `USER_NOT_FOUND`가 의미 명확.
- **유지보수성**: 코드 추가/변경 시 숫자 충돌 관리 불필요.
- **문서화 용이**: Swagger/OpenAPI에서 문자열 코드가 더 직관적.

---

## 4. 스프링에서 HTTP 상태 코드 사용의 이유 (상세)

스프링이 HTTP 상태 코드를 기본으로 사용하는 이유는 다음과 같습니다:

1. **RESTful API 표준 준수**:
   - REST는 HTTP 프로토콜을 기반으로 하며, 상태 코드(200, 404, 500 등)는 클라이언트와 서버 간 표준화된 오류 전달 수단.
   - 예: `404`는 리소스 없음, `403`은 권한 없음을 명확히 전달.
   - Express.js는 RESTful 원칙을 덜 강제하므로 커스텀 코드 사용이 일반적.

2. **프레임워크 통합**:
   - 스프링의 `ResponseEntity`, `@ResponseStatus`, `HttpStatus`는 HTTP 상태 코드와 직접 연계되어 설계.
   - 예: `MethodArgumentNotValidException`은 자동으로 `400 Bad Request`로 매핑.
   - `@ControllerAdvice`로 예외를 HTTP 상태 코드와 통합 처리 가능.

3. **클라이언트 호환성**:
   - HTTP 상태 코드는 브라우저, 모바일 앱, API 클라이언트(예: `axios`, `fetch`)가 이해하는 표준.
   - 커스텀 숫자 코드는 클라이언트가 추가 문서 없이 이해하기 어려움.

4. **간결한 설계**:
   - HTTP 상태 코드(4xx, 5xx)는 대부분의 오류를 포괄하므로 별도의 코드 체계 관리 부담 감소.
   - 예: `400`은 입력 오류, `500`은 서버 오류로 충분히 표현 가능.

5. **오류 계층화**:
   - 스프링 예외는 계층적으로 설계되어 세부 오류를 HTTP 상태 코드로 매핑.
   - 예: `DataAccessException` → `500 Internal Server Error`.

---

## 5. Express.js와 스프링의 혼합 접근법

Express.js 스타일의 커스텀 에러 코드를 스프링에 도입하면서 HTTP 상태 코드를 유지하려면 다음과 같은 접근을 추천합니다:

1. **문자열 에러 코드 + HTTP 상태 코드**:
   - 에러 코드는 도메인별로 명확한 문자열 사용 (예: `USER_NOT_FOUND`).
   - 각 에러 코드에 HTTP 상태 코드 매핑.
2. **표준화된 응답 구조**:
   - `{ errorCode, message, status }` 형식으로 응답.
   - 예:
     ```json
     {
       "errorCode": "USER_NOT_FOUND",
       "message": "User not found with ID: 123",
       "status": 404
     }
     ```
3. **문서화**:
   - Swagger/OpenAPI로 에러 코드와 HTTP 상태 코드 문서화.
   - 예:
     ```yaml
     /users/{id}:
       get:
         responses:
           '404':
             description: User not found
             content:
               application/json:
                 schema:
                   type: object
                   properties:
                     errorCode:
                       type: string
                       example: USER_NOT_FOUND
                     message:
                       type: string
                       example: User not found with ID: 123
                     status:
                       type: integer
                       example: 404
     ```

### 구현 예제 (혼합 방식)
```java
// 커스텀 예외
public class UserNotFoundException extends DomainException {
    public UserNotFoundException(String userId) {
        super("USER_NOT_FOUND", "User not found with ID: " + userId, HttpStatus.NOT_FOUND);
    }
}

// 응답 DTO
public class ErrorResponse {
    private String errorCode;
    private String message;
    private int status;

    public ErrorResponse(String errorCode, String message, HttpStatus httpStatus) {
        this.errorCode = errorCode;
        this.message = message;
        this.status = httpStatus.value();
    }
}

// 전역 예외 처리
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ErrorResponse> handleDomainException(DomainException ex) {
        ErrorResponse response = new ErrorResponse(ex.getErrorCode(), ex.getMessage(), ex.getHttpStatus());
        return new ResponseEntity<>(response, ex.getHttpStatus());
    }
}
```

- **응답 예시**:
  ```json
  {
    "errorCode": "USER_NOT_FOUND",
    "message": "User not found with ID: 123",
    "status": 404
  }
  ```

---

## 6. 커스텀 에러 코드 vs HTTP 상태 코드: 언제 무엇을?

### 6.1 커스텀 에러 코드 사용 시 (Express.js 스타일)
- **적합 상황**:
  - 비즈니스 로직별로 세밀한 오류 구분 필요 (예: `INVALID_EMAIL_FORMAT` vs `DUPLICATE_EMAIL`).
  - 클라이언트가 특정 에러 코드에 따라 다른 동작 수행 (예: UI에서 특정 메시지 표시).
  - 내부 시스템 간 통신에서 고유한 에러 코드 체계 필요.
- **예시**:
  - 사용자 인증 실패: `AUTH_INVALID_CREDENTIALS` (10001), `AUTH_EXPIRED_TOKEN` (10002).

### 6.2 HTTP 상태 코드만 사용 시
- **적합 상황**:
  - 간단한 REST API 설계, 표준 HTTP 상태 코드로 충분.
  - 클라이언트가 HTTP 상태 코드만으로 오류 처리 가능.
  - 복잡한 에러 코드 체계 관리 피하고 싶을 때.
- **예시**:
  - `404 Not Found`로 모든 "리소스 없음" 오류 처리.

### 6.3 혼합 접근
- **권장 상황**:
  - RESTful 원칙을 준수하면서 도메인 특화 에러 코드 필요.
  - 클라이언트가 세부 에러를 처리해야 하지만, HTTP 상태 코드로 표준화된 응답도 필요.
- **예시**:
  - `USER_NOT_FOUND` (404), `INVALID_ORDER` (400).

---

## 7. 추가 팁: 커스텀 에러 코드 관리

1. **문서화**:
   - 에러 코드 목록을 별도 문서(예: `error_codes.md`) 또는 Swagger에 유지.
   - 예:
     ```markdown
     | Error Code       | HTTP Status | Description                |
     |------------------|-------------|----------------------------|
     | USER_NOT_FOUND   | 404         | User not found by ID       |
     | INVALID_ORDER    | 400         | Invalid order parameters   |
     ```

2. **코드 충돌 방지**:
   - 문자열 코드는 고유성을 보장 (예: 접두사 사용: `USER_`, `ORDER_`).
   - 숫자 코드 사용 시 범위별로 구분 (예: 10000~19999는 사용자, 20000~29999는 주문).

3. **클라이언트 협업**:
   - 프론트엔드 개발자와 에러 코드 및 HTTP 상태 코드 매핑 협의.
   - 예: `USER_NOT_FOUND` 발생 시 특정 UI 표시.

4. **로깅**:
   - 에러 코드와 함께 상세 로그 기록.
   - 예:
     ```java
     log.error("Error [{}]: {}", ex.getErrorCode(), ex.getMessage(), ex);
     ```

---

## 8. 결론

Express.js의 임의 숫자 코드(예: `10001`) 방식은 유연하지만 비표준화와 관리 복잡성이 단점입니다. 반면, 스프링은 HTTP 상태 코드를 기본으로 사용하여 RESTful 원칙을 준수하고 클라이언트 호환성을 높입니다. 스프링에서 커스텀 에러 코드를 도입할 때는 문자열 코드(예: `USER_NOT_FOUND`)와 HTTP 상태 코드를 결합하여 도메인 특화 오류를 명확히 표현하면서 RESTful 표준을 유지하는 것이 이상적입니다.

### 요약
- **Express.js**: 임의 숫자 코드(예: `10001`)로 세밀한 오류 구분, 유연하지만 비표준.
- **스프링**: HTTP 상태 코드(예: 404) 기반, RESTful 원칙 준수, 표준화된 응답.
- **혼합 접근**: 문자열 에러 코드 + HTTP 상태 코드로 도메인 특화와 표준화 모두 달성.
- **권장**: `@ControllerAdvice`로 중앙화된 처리, 문자열 코드 사용, 문서화 철저.

이 자료를 통해 Express.js와 스프링의 에러 처리 차이를 이해하고, 스프링에서 효과적인 커스텀 에러 코드 설계를 구현할 수 있습니다.