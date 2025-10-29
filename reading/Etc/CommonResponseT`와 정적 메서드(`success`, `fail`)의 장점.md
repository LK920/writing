---
aliases:
  - CommonResponse<T>`와 정적 메서드(`success`
  - "`fail`)의 장점"
---
# `CommonResponse<T>`와 정적 메서드(`success`, `fail`)의 장점

## 📌 개요

`CommonResponse<T>` 구조는 API 응답을 표준화하기 위해 자주 사용되는 패턴입니다. 이 구조에서 `success`와 `fail`을 정적 메서드로 사용하는 방식은 코드의 간결성, 일관성, 유지보수성을 크게 향상시킵니다. 본 문서에서는 성용님의 조언을 바탕으로 이 방식의 장점과, `errorCode` 및 `CustomException` 처리, 그리고 "왜 `fail` 메서드가 필요한지"에 대한 질문과 HTTP 상태 코드 처리(200/500) 방식에 대해 자세히 설명합니다.

---

## ✅ `CommonResponse<T>`란?

`CommonResponse<T>`는 API 응답을 일관된 포맷으로 반환하기 위한 클래스입니다. 일반적으로 다음과 같은 필드를 포함합니다:

- `status`: HTTP 상태 코드 또는 내부 상태 코드 (예: 200, 400, 404)
- `message`: 사용자에게 보여줄 메시지 (예: "success", "사용자를 찾을 수 없습니다.")
- `errorCode`: 에러 식별 코드 (예: "USER_NOT_FOUND")
- `data`: 성공 시 데이터, 실패 시 null 또는 추가 에러 정보

예시 구조:

```java
public class CommonResponse<T> {
    private int status;
    private String message;
    private String errorCode;
    private T data;

    public static <T> CommonResponse<T> success(T data) {
        return new CommonResponse<>(200, "success", null, data);
    }

    public static <T> CommonResponse<T> fail(int status, String errorCode, String message) {
        return new CommonResponse<>(status, message, errorCode, null);
    }

    public static <T> CommonResponse<T> fail(CustomException ex) {
        return new CommonResponse<>(ex.getStatus(), ex.getMessage(), ex.getErrorCode(), null);
    }

    public static <T> CommonResponse<T> internalError(String message) {
        return new CommonResponse<>(500, message, "INTERNAL_SERVER_ERROR", null);
    }

    private CommonResponse(int status, String message, String errorCode, T data) {
        this.status = status;
        this.message = message;
        this.errorCode = errorCode;
        this.data = data;
    }

    // Getter 메서드들
}
```

---

## ✅ 왜 `success`와 `fail` 정적 메서드를 사용하는가?

성용님의 조언에서 `CommonResponse`의 `success`와 `fail`을 정적 메서드로 사용하는 이유는 다음과 같습니다:

### 1. 코드 간결성

- **문제**: `new CommonResponse<>(200, "success", null, data)`처럼 매번 동일한 구조를 작성하면 코드가 장황해지고 반복적입니다.
- **해결**: `CommonResponse.success(data)` 또는 `CommonResponse.fail(status, errorCode, message)`로 간단히 호출.
- **예시**:
    
    ```java
    // 전
    return new CommonResponse<>(200, "success", null, data);
    
    // 후
    return CommonResponse.success(data);
    ```
    

### 2. 일관된 응답 포맷 보장

- 정적 메서드는 응답의 구조를 캡슐화하여 실수(예: 잘못된 상태 코드 입력, 필드 누락)를 방지합니다.
- 성공/실패 응답의 형식이 항상 동일하게 유지되어 클라이언트가 예측 가능한 응답을 받습니다.

### 3. 유지보수성 향상

- 응답 포맷이 변경될 때(예: `errorCode` 필드 이름 변경), `CommonResponse` 클래스만 수정하면 모든 호출 지점이 자동으로 반영됩니다.
- Controller는 로직 변경 없이 그대로 유지 가능.

### 4. 가독성 개선

- `success`와 `fail`은 메서드 이름 자체가 의도를 명확히 드러냅니다.
- `new CommonResponse<>(...)`보다 직관적이고 코드가 간결해져 읽기 쉬워집니다.

---

## ✅ `ResponseDto`의 `from` 정적 메서드와의 조합

성용님은 DTO에 `from` 같은 정적 메서드를 추가해 Entity → DTO 변환 책임을 DTO 클래스에 위임하라고 제안했습니다. 이는 Controller의 책임을 줄이고 코드 간결성을 유지합니다.

### 예시

```java
public class UserResponse {
    private Long id;
    private String name;

    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getName());
    }
}
```

### 장점

- **책임 분리**: DTO 변환 로직을 Controller에서 분리해 DTO 클래스에서 관리.
- **간결한 Controller**:
    
    ```java
    // 전
    UserResponse dto = new UserResponse(user.getId(), user.getName());
    return new CommonResponse<>(200, "success", null, dto);
    
    // 후
    return CommonResponse.success(UserResponse.from(user));
    ```
    
- **유지보수성**: Entity 구조가 변경되더라도 `UserResponse.from()`만 수정하면 Controller는 영향을 받지 않음.

---

## ✅ `fail` 메서드의 필요성

질문에서 제기된 "왜 `fail`을 호출할 일이 있냐?"는 의문은, Controller에서 직접 `fail()`을 호출하는 경우가 드물다고 느껴질 수 있기 때문입니다. 하지만 `fail()`은 다양한 상황에서 필수적이며, 아래에서 그 사용 사례를 정리합니다.

### 1. `CustomException` 처리

서비스 계층에서 던져진 `CustomException`을 Controller에서 받아 `CommonResponse.fail()`로 변환합니다.

```java
// 서비스 계층
public class UserService {
    public User getUser(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new CustomException(ErrorCode.USER_NOT_FOUND));
    }
}

// Controller
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public ResponseEntity<CommonResponse<UserResponse>> getUser(@PathVariable Long id) {
        User user = userService.getUser(id);
        return ResponseEntity.ok(CommonResponse.success(UserResponse.from(user)));
    }

    @ExceptionHandler(CustomException.class)
    public ResponseEntity<CommonResponse<Void>> handleCustomException(CustomException ex) {
        return ResponseEntity.ok(CommonResponse.fail(ex)); // fail() 호출
    }
}
```

- **왜 필요?**: 서비스 계층에서 발생한 예외를 일관된 `CommonResponse` 포맷으로 변환하기 위해 `fail()`을 사용.
- **장점**: `@ExceptionHandler`를 통해 모든 `CustomException`을 체계적으로 처리 가능.

### 2. Controller에서 직접 에러 조건 처리

입력값 검증 등 간단한 에러 상황에서 `fail()`을 직접 호출합니다.

```java
@GetMapping("/users/check")
public ResponseEntity<CommonResponse<Void>> checkUser(@RequestParam String username) {
    if (username == null || username.isEmpty()) {
        return ResponseEntity.ok(
            CommonResponse.fail(400, "INVALID_INPUT", "사용자 이름이 비어 있습니다.")
        ); // 직접 fail() 호출
    }
    return ResponseEntity.ok(CommonResponse.success(null));
}
```

- **왜 필요?**: 서비스 계층까지 가지 않고 Controller에서 간단히 처리할 수 있는 에러를 위해.
- **장점**: 에러 응답도 `CommonResponse` 포맷을 유지해 일관성 보장.

### 3. 글로벌 예외 처리

예상치 못한 예외(예: `NullPointerException`)를 전역적으로 처리할 때 `fail()` 또는 `internalError()`를 사용합니다.

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<CommonResponse<Void>> handleGeneralException(Exception ex) {
        return ResponseEntity
                .status(500)
                .body(CommonResponse.internalError("서버에서 예기치 않은 오류가 발생했습니다."));
    }
}
```

- **왜 필요?**: 예외 처리를 중앙화하여 코드 중복을 줄이고 일관된 에러 응답을 제공.
- **장점**: 모든 예외를 `CommonResponse` 포맷으로 처리 가능.

### `fail` 호출이 드물게 느껴지는 이유와 반박

- **의문**: Controller에서 성공 케이스는 `success()`로 자주 호출하지만, `fail()`은 `@ExceptionHandler`나 특정 조건에서만 사용되므로 호출 빈도가 낮다고 느껴질 수 있음.
- **반박**: `fail()`은 호출 빈도가 적더라도, 에러 응답의 일관성을 위해 필수적입니다. 직접 호출(예: 입력 검증)이나 `@ExceptionHandler`를 통해 간접 호출되며, 에러 처리 로직을 통일하는 데 핵심 역할을 합니다.

---

## ✅ `errorCode`와 `CustomException` 처리

단순히 `success`와 `fail`만 사용하면 다양한 에러 상황을 세밀히 처리하기 어렵다는 우려가 있었습니다. 이를 해결하기 위해 `errorCode`와 `CustomException`을 통합한 구조를 설계했습니다.

### 1. `CustomException` 설계

```java
public class CustomException extends RuntimeException {
    private final int status;
    private final String errorCode;

    public CustomException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.status = errorCode.getStatus();
        this.errorCode = errorCode.getCode();
    }

    // Getter 메서드들
}
```

### 2. `ErrorCode` Enum

```java
public enum ErrorCode {
    USER_NOT_FOUND(404, "USER_NOT_FOUND", "사용자를 찾을 수 없습니다."),
    INVALID_INPUT(400, "INVALID_INPUT", "잘못된 입력값입니다."),
    CONFLICT(409, "CONFLICT", "리소스 충돌이 발생했습니다.");

    private final int status;
    private final String code;
    private final String message;

    ErrorCode(int status, String code, String message) {
        this.status = status;
        this.code = code;
        this.message = message;
    }

    // Getter 메서드들
}
```

### 장점

- **세밀한 에러 처리**: `errorCode`를 통해 클라이언트가 에러를 구체적으로 식별 가능.
- **체계적 관리**: `ErrorCode` Enum으로 에러 코드를 중앙 관리, 새로운 에러 추가 용이.
- **확장성**: `CustomException`을 확장해 추가 데이터 포함 가능.

---

## ✅ HTTP 상태 코드 200/500 방식에 대한 논의

모든 성공과 커스텀 에러를 HTTP 200으로, 내부 서버 에러만 HTTP 500으로 처리하는 방식도 제안되었습니다.

### 구현 예시

```java
@ExceptionHandler(CustomException.class)
public ResponseEntity<CommonResponse<Void>> handleCustomException(CustomException ex) {
    return ResponseEntity.ok(CommonResponse.fail(ex)); // HTTP 200
}

@ExceptionHandler(Exception.class)
public ResponseEntity<CommonResponse<Void>> handleGeneralException(Exception ex) {
    return ResponseEntity
        .status(500)
        .body(CommonResponse.internalError("서버에서 예기치 않은 오류가 발생했습니다."));
}
```

### 장점

- **클라이언트 단순화**: 클라이언트가 HTTP 상태 코드(200/500)만 확인하고, 세부 에러는 `status`, `errorCode`, `message`로 판단.
- **프록시 호환성**: 일부 환경에서 4xx/5xx 코드를 비표준적으로 처리할 때 유용.
- **일관된 응답**: 성공과 에러 모두 `CommonResponse` 포맷 유지.

### 단점

- **RESTful 원칙 위배 가능성**: HTTP 상태 코드(400, 404 등)는 에러 구분에 표준적으로 사용되므로, 200으로 통일하면 클라이언트가 응답 본문을 추가로 파싱해야 함.
- **디버깅 어려움**: 로그나 모니터링 도구에서 에러를 빠르게 식별하기 어려울 수 있음.
- **클라이언트 협업**: 클라이언트 팀이 HTTP 상태 코드를 기대한다면 추가 문서화 필요.

### 결론

- HTTP 200/500 방식은 클라이언트 환경에 따라 유용할 수 있지만, RESTful 원칙과 디버깅 편의성을 위해 표준 HTTP 상태 코드를 사용하는 것이 일반적입니다.
- `CommonResponse`의 `fail` 메서드는 이 두 접근법 모두를 지원하도록 유연하게 설계되었습니다.

---

## 🔚 요약

`CommonResponse<T>`와 `success`/`fail` 정적 메서드를 사용하는 방식은 다음과 같은 이유로 좋은 패턴입니다:

|요소|장점|예시|
|---|---|---|
|`success`/`fail` 정적 메서드|코드 간결성, 일관성, 실수 방지|`CommonResponse.success(data)`|
|`ResponseDto.from()`|DTO 변환 책임 분리, Controller 간결화|`UserResponse.from(user)`|
|`errorCode`와 `CustomException`|세밀한 에러 처리, 클라이언트 친화적 응답|`CommonResponse.fail(ex)`|
|HTTP 200/500 방식|클라이언트 단순화, 프록시 호환성|`ResponseEntity.ok(CommonResponse.fail(ex))`|
|유지보수성|로직 변경 시 Controller 수정 최소화|`ErrorCode` Enum 수정으로 새 에러 추가|

### 질문에 대한 답변

- **왜 `fail`이 필요?**: Controller에서 직접 에러 처리(예: 입력 검증)하거나 `@ExceptionHandler`를 통해 `CustomException`을 일관되게 처리하기 위해. 호출 빈도가 적더라도 에러 응답의 일관성을 보장.
- **왜 좋은 방식?**: 코드 중복 제거, 가독성 향상, 유지보수성 강화, 클라이언트와의 명확한 에러 커뮤니케이션.

이 패턴은 실무에서 널리 사용되며, Controller는 "요청-응답 조율"에 집중하고, 나머지 로직은 `CommonResponse`와 DTO로 분리해 이상적인 구조를 만듭니다. 추가적인 요구사항이나 수정이 필요하다면 언제든 말씀해주세요!