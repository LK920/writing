# Spring MockMvc 주요 함수: 사용 빈도순 정리 및 꿀 함수 제안

Spring **MockMvc**는 Spring MVC 애플리케이션의 컨트롤러를 테스트하기 위한 도구로, HTTP 요청/응답을 모방해 서버 없이 테스트할 수 있다. 아래는 MockMvc에서 자주 사용되는 함수들을 사용 빈도순으로 정리하고, 기능과 사용법을 설명한다. 또한, 덜 알려졌지만 유용한 "꿀 함수"를 제안한다.

---

## 1. `perform(RequestBuilder)` - 가장 자주 사용
### 기능
- HTTP 요청을 실행하는 핵심 함수.
- `MockMvcRequestBuilders`의 `get`, `post`, `put`, `delete` 등으로 생성된 요청을 처리.
- 반환값은 `ResultActions`로, 응답 검증(`andExpect`)이나 추가 작업(`andDo`, `andReturn`)을 체이닝 가능.

### 사용법
```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk())
    .andExpect(content().string("Hello, User!"));
```
- **설명**: `/api/users/1`로 GET 요청을 보내고, 응답 상태가 200 OK이고 내용이 "Hello, User!"인지 확인.
- **빈도**: 거의 모든 MockMvc 테스트에서 사용. 테스트의 시작점.

### 꿀팁
- URL 템플릿 사용: `get("/api/users/{id}", 1)`로 동적 경로 변수 처리.
- 요청 커스터마이징: `contentType`, `content`, `header` 등을 추가해 실제 요청과 유사하게 설정.

---

## 2. `andExpect(ResultMatcher)` - 매우 자주 사용
### 기능
- 응답을 검증하는 핵심 메서드.
- `MockMvcResultMatchers`의 다양한 메서드(`status`, `content`, `jsonPath`, `view` 등)와 함께 사용.
- HTTP 상태, 응답 본문, 헤더, 뷰 이름 등을 확인.

### 사용법
```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk())
    .andExpect(content().contentType("application/json"))
    .andExpect(jsonPath("$.name").value("John"));
```
- **설명**: 응답 상태가 200 OK, 컨텐츠 타입이 JSON, JSON 본문의 `name` 필드가 "John"인지 검증.
- **빈도**: 응답 검증이 테스트의 핵심이므로 `perform` 다음으로 가장 많이 사용.

### 꿀팁
- `jsonPath`로 복잡한 JSON 구조 검증 가능 (예: `$.users[0].name`).
- `status().is(200)` 대신 `status().isOk()` 사용으로 가독성 향상.

---

## 3. `andDo(ResultHandler)` - 자주 사용
### 기능
- 응답에 대한 추가 작업을 수행 (예: 디버깅을 위해 응답 출력).
- `MockMvcResultHandlers.print()`가 가장 흔히 사용됨.
- 요청/응답의 상세 로그를 콘솔에 출력해 디버깅에 유용.

### 사용법
```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.*;

mockMvc.perform(get("/api/users/1"))
    .andDo(print())
    .andExpect(status().isOk());
```
- **설명**: `/api/users/1`로 GET 요청 후, 요청/응답 세부 정보를 콘솔에 출력.
- **빈도**: 디버깅이나 테스트 실패 원인 파악 시 자주 사용.

### 꿀팁
- `print()`는 요청/응답의 전체 로그(헤더, 본문 등)를 출력하므로, 복잡한 테스트에서 문제 진단에 필수.
- 커스텀 `ResultHandler`를 구현해 특정 로그만 출력 가능.

---

## 4. `andReturn()` - 보통 사용
### 기능
- 테스트 결과를 `MvcResult` 객체로 반환.
- `andExpect`로 처리할 수 없는 복잡한 검증이나 응답 데이터를 직접 조작할 때 유용.
- 응답 본문, 헤더, 상태 코드 등을 직접 접근 가능.

### 사용법
```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;

MvcResult result = mockMvc.perform(get("/api/users/1"))
    .andExpect(status().isOk())
    .andReturn();

String content = result.getResponse().getContentAsString();
assertEquals("Hello, User!", content);
```
- **설명**: GET 요청 후 응답 본문을 `MvcResult`로 받아 문자열로 검증.
- **빈도**: JSON이나 복잡한 응답을 직접 파싱하거나 추가 검증이 필요할 때 사용.

### 꿀팁
- JSON 응답을 객체로 변환하려면 `ObjectMapper`와 함께 사용:
  ```java
  String json = result.getResponse().getContentAsString();
  User user = new ObjectMapper().readValue(json, User.class);
  assertEquals("John", user.getName());
  ```

---

## 5. `MockMvcRequestBuilders.*` (get, post, put, delete 등) - 보통 사용
### 기능
- HTTP 요청을 생성하는 빌더 메서드들.
- `get`, `post`, `put`, `delete` 등 HTTP 메서드에 맞는 요청을 구성.
- 경로 변수, 쿼리 파라미터, 요청 본문, 헤더 등을 설정 가능.

### 사용법
```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

mockMvc.perform(post("/api/users")
    .contentType(MediaType.APPLICATION_JSON)
    .content("{\"name\": \"John\", \"age\": 30}"))
    .andExpect(status().isCreated());
```
- **설명**: `/api/users`로 POST 요청을 보내고, JSON 본문을 포함하며 상태 코드 201 확인.
- **빈도**: 모든 테스트에서 HTTP 요청을 만들 때 사용.

### 꿀팁
- 쿼리 파라미터: `.param("key", "value")`로 추가.
- 경로 변수: `get("/api/users/{id}", 1)`로 간편히 처리.
- 헤더 추가: `.header("Authorization", "Bearer token")`로 인증 테스트 가능.

---

## 꿀 함수 제안 (덜 사용되지만 유용)
다음은 자주 사용되지는 않지만 특정 상황에서 매우 유용한 MockMvc 함수들이다.

### 1. `andExpectAll(ResultMatcher...)`
- **기능**: 여러 `ResultMatcher`를 한 번에 검증. 여러 조건을 간결히 작성 가능.
- **사용법**:
  ```java
  import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
  import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

  mockMvc.perform(get("/api/users/1"))
      .andExpectAll(
          status().isOk(),
          content().contentType("application/json"),
          jsonPath("$.name").value("John")
      );
  ```
- **왜 유용?**: `andExpect`를 여러 번 호출하는 대신 하나로 묶어 가독성 향상. 복잡한 검증 로직을 간소화.
- **언제 사용?**: 응답에 대해 여러 조건(상태, 헤더, 본문 등)을 한 번에 확인할 때.

### 2. `MockMvcBuilders.standaloneSetup`
- **기능**: 특정 컨트롤러만 테스트하도록 MockMvc를 설정. 전체 애플리케이션 컨텍스트 대신 가벼운 설정 제공.
- **사용법**:
  ```java
  import static org.springframework.test.web.servlet.setup.MockMvcBuilders.*;

  MockMvc mockMvc = standaloneSetup(new UserController()).build();
  mockMvc.perform(get("/api/users/1"))
      .andExpect(status().isOk());
  ```
- **왜 유용?**: `@WebMvcTest`보다 더 가볍고, 단일 컨트롤러 테스트에 최적화. 불필요한 빈 로딩 방지.
- **언제 사용?**: 특정 컨트롤러만 단위 테스트하고 싶을 때.

### 3. `MockMvcResultMatchers.jsonPath`
- **기능**: JSON 응답의 특정 필드를 검증. 복잡한 JSON 구조를 쉽게 테스트.
- **사용법**:
  ```java
  import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
  import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

  mockMvc.perform(get("/api/users"))
      .andExpect(jsonPath("$[0].name").value("John"))
      .andExpect(jsonPath("$.length()").value(2));
  ```
- **왜 유용?**: 배열이나 중첩 JSON을 정밀하게 검증 가능. JsonPath 표현식으로 유연한 테스트 작성.
- **언제 사용?**: REST API의 JSON 응답이 복잡할 때.

### 4. `with(RequestPostProcessor)`
- **기능**: 요청에 커스텀 설정(예: 인증 정보, 세션 속성)을 추가.
- **사용법** (Spring Security 테스트 예시):
  ```java
  import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.*;

  mockMvc.perform(get("/api/secure").with(user("john").roles("USER")))
      .andExpect(status().isOk());
  ```
- **왜 유용?**: Spring Security와 통합해 인증/인가 테스트를 쉽게 작성. `@WithMockUser`보다 요청 단위로 유연.
- **언제 사용?**: 보안이 적용된 엔드포인트 테스트 시.

---

## 결론
- **핵심 함수**: `perform`, `andExpect`, `andDo`, `andReturn`, `MockMvcRequestBuilders.*`는 MockMvc 테스트의 기본.
- **꿀 함수**: `andExpectAll`, `standaloneSetup`, `jsonPath`, `with`는 특정 상황에서 테스트 효율성과 가독성을 크게 높인다.
- **권장사항**:
  - 디버깅 시 `andDo(print())`는 필수.
  - 복잡한 JSON 응답은 `jsonPath`로 처리.
  - 보안 테스트에는 `with`와 Spring Security 통합 활용.
  - `@WebMvcTest`와 `@MockBean`을 사용해 서비스 레이어 의존성을 모킹하면 테스트가 가벼워진다.[](https://rieckpil.de/guide-to-testing-spring-boot-applications-with-mockmvc/)

MockMvc는 Spring MVC 테스트를 위한 강력한 도구로, 위 함수들을 적절히 조합하면 컨트롤러 테스트를 효과적으로 작성할 수 있다.