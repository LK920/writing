# Spring @ControllerAdvice 에러 핸들러 전략 (실전/실무/테스트 관점)

---

## 1. 왜 별도의 에러 핸들러가 필요한가?

### 1.1. 기본 Spring MVC 예외 처리의 한계
- Spring은 컨트롤러에서 발생한 예외(런타임 포함)를 잡지 않으면, HTTP 500(Internal Server Error)로 응답한다.
- 비즈니스 로직에서 발생하는 예외(예: IllegalArgumentException)는 실제로는 클라이언트 잘못(400 Bad Request)인 경우가 많다.
- 하지만 별도의 예외 매핑이 없으면, 모든 예외가 500으로 내려가고, 응답 body도 일관성이 없다.

### 1.2. 실무에서의 문제점
- 프론트/외부시스템은 HTTP 상태코드만으로는 어떤 에러인지 구분이 어렵다.
- 비즈니스 에러(잔액 부족, 유저 없음 등)는 명확한 code/message가 필요하다.
- 테스트에서도 "정확히 어떤 에러가 발생했는지"를 검증해야 한다.
- 유지보수/확장성 측면에서 에러 응답의 일관성이 매우 중요하다.

---

## 2. ErrorCode 기반 에러 응답의 필요성

### 2.1. ErrorCode란?
- 프로젝트 내에서 비즈니스 에러를 enum 등으로 정의한 것 (ex: `ErrorCode.INVALID_AMOUNT`)
- 각 에러는 code(식별자), message(설명)로 구성
- 예시:
  ```java
  public enum ErrorCode {
      INSUFFICIENT_POINT("P001", "포인트가 부족합니다."),
      INVALID_AMOUNT("P002", "포인트는 0보다 커야 합니다."),
      ...
  }
  ```

### 2.2. ErrorCode 기반 응답의 장점
- 프론트/외부시스템이 code로 분기 처리 가능 (ex: P001이면 잔액부족 안내)
- message는 사용자/개발자에게 명확한 피드백 제공
- 테스트에서 code/message로 정확한 검증 가능
- 에러 응답의 일관성 보장 (모든 에러가 동일한 구조로 내려감)

---

## 3. @ControllerAdvice + @ExceptionHandler 전략

### 3.1. 핵심 원리
- `@RestControllerAdvice`는 전역적으로 예외를 잡아 처리할 수 있는 Spring 기능
- `@ExceptionHandler(IllegalArgumentException.class)` 등으로 특정 예외를 잡아 커스텀 응답 반환 가능
- ErrorCode와 매핑하여 code/message를 내려주면, 비즈니스 에러에 대한 일관된 응답이 가능

### 3.2. 실제 적용 코드
```java
@RestControllerAdvice
class ApiControllerAdvice extends ResponseEntityExceptionHandler {
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(IllegalArgumentException ex) {
        ErrorCode matched = Arrays.stream(ErrorCode.values())
            .filter(e -> e.getMessage().equals(ex.getMessage()))
            .findFirst()
            .orElse(ErrorCode.SYSTEM_ERROR);
        return ResponseEntity.badRequest().body(
            new ErrorResponse(matched.getCode(), matched.getMessage())
        );
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        return ResponseEntity.status(500).body(new ErrorResponse("500", "에러가 발생했습니다."));
    }
}
```

### 3.3. 동작 방식
- 서비스/컨트롤러에서 IllegalArgumentException 발생 시 → ErrorCode 매핑 → 400 + code/message 반환
- 그 외 알 수 없는 예외는 500 + 기본 에러 메시지

---

## 4. 실전 적용 효과 (테스트/운영 관점)

### 4.1. 테스트 코드에서의 효과
- 실패 케이스에서 HTTP 상태코드, code, message를 모두 검증 가능
- 예시:
  ```java
  mockMvc.perform(patch("/point/{id}/use", userId)
      .contentType(MediaType.APPLICATION_JSON)
      .content(objectMapper.writeValueAsString(use)))
      .andExpect(status().isBadRequest())
      .andExpect(jsonPath("$.code").value(ErrorCode.INSUFFICIENT_POINT.getCode()))
      .andExpect(jsonPath("$.message").value(ErrorCode.INSUFFICIENT_POINT.getMessage()));
  ```
- 테스트가 명확해지고, 실무에서 요구하는 에러 응답 일관성을 강제할 수 있다.

### 4.2. 운영/실서비스에서의 효과
- 프론트/외부시스템이 code로 분기 처리 가능 (ex: P002면 금액 입력 안내)
- 사용자에게 명확한 메시지 제공
- 장애/버그 발생 시, code로 빠른 원인 파악 가능
- 신규 에러 추가/변경 시, ErrorCode만 추가/수정하면 됨

---

## 5. 실무적 BestPractice / Tip

- **비즈니스 예외는 반드시 ErrorCode 기반으로 관리**
- 서비스/컨트롤러에서 IllegalArgumentException 등으로 예외 발생 시, 반드시 ErrorCode 메시지와 일치시킬 것
- @ControllerAdvice에서 ErrorCode 매핑이 안 되면 SYSTEM_ERROR 등으로 fallback
- Exception 핸들러는 500만 처리 (알 수 없는 예외)
- 테스트는 반드시 code/message까지 검증 (상태코드만 검증 X)
- 프론트/외부시스템과 에러코드/메시지 약속을 문서화할 것
- 신규 에러 추가 시, ErrorCode/핸들러/테스트 모두 동시 반영

---

## 6. 실전 Q&A (나와의 대화 기반)

- **Q: 왜 500이 자꾸 뜨지?**
  - A: 서비스 계층에서 IllegalArgumentException 등 비즈니스 예외가 발생해도, @ControllerAdvice에서 400으로 매핑하지 않으면 500으로 내려감
- **Q: 테스트에서 400을 기대하는데 500이 내려온다?**
  - A: 위와 같은 이유. 핸들러 추가로 해결
- **Q: ErrorCode 기준으로 테스트해야 하는 이유?**
  - A: 실무에서 code/message가 더 중요. 상태코드는 부가적 참고일 뿐
- **Q: Exception 핸들러는 왜 남겨두나?**
  - A: 진짜 시스템 장애(NullPointerException 등)는 500으로 내려야 함

---

## 7. 결론

- 실무/테스트/운영 모두에서 에러 응답의 일관성, 명확성, 확장성을 위해 반드시 별도의 에러 핸들러(@ControllerAdvice + ErrorCode 매핑)를 구현해야 한다.
- 이 전략을 쓰면 프론트/백엔드 협업, 유지보수, 장애 대응, 테스트 자동화 등 모든 면에서 압도적으로 유리하다.
- "상태코드만 신경쓰지 말고, code/message까지 신경써라"가 실무의 정답.

---

> 이 문서는 실제 TDD/실무/테스트/운영 경험, 그리고 나와의 대화에서 나온 요구와 문제 상황을 바탕으로 작성됨. 추가로 궁금한 점이 있으면 언제든 질문해도 좋다. 