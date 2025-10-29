---
tags:
  - study
  - test
  - TDD
---
네, 좋은 질문입니다! `MockMvc`, `ObjectMapper`, `jsonPath`는 스프링에서 컨트롤러를 테스트할 때 거의 항상 함께 사용되는 삼총사입니다. 각각의 역할에 대해 쉽고 명확하게 설명해 드릴게요.

### 1. `MockMvc` (가짜 웹 서버)

*   **정체**: `Mvc`는 Spring MVC 프레임워크를 의미하고, `Mock`은 가짜라는 뜻입니다. 즉, **실제 서버를 띄우지 않고 컨트롤러를 테스트할 수 있도록 만들어진 가짜(Mock) MVC 환경**입니다.
*   **역할**: 이 테스트에서 `MockMvc`는 마치 **테스트 코드 안에 내장된 가상 Postman이나 웹 브라우저**와 같은 역할을 합니다.
    *   `mockMvc.perform(...)` 메소드를 통해 실제 HTTP 요청(GET, POST, PATCH 등)을 시뮬레이션합니다.
    *   예를 들어, `mockMvc.perform(get("/point/{id}", userId))` 코드는 `/point/1` 이라는 URL로 GET 요청을 보내는 것과 똑같은 효과를 냅니다.
    *   이렇게 요청을 보내면 `MockMvc`는 스프링의 `DispatcherServlet`부터 시작하는 웹 요청 처리 과정을 그대로 재현하여 `PointController`의 해당 메소드를 실행시킵니다.
*   **왜 쓰는가?**: 실제 `tomcat` 같은 웹 서버를 띄우고 API를 테스트하는 것보다 훨씬 가볍고 빠릅니다. 오직 웹 계층(컨트롤러)의 로직만 분리해서 신속하게 테스트할 수 있게 해줍니다.

### 2. `ObjectMapper` (JSON 변환기)

*   **정체**: Jackson 라이브러리에 포함된 클래스로, **Java 객체와 JSON 데이터 간의 변환**을 책임지는 전문가입니다.
    *   **Java 객체 → JSON 문자열 (직렬화, Serialization)**
    *   **JSON 문자열 → Java 객체 (역직렬화, Deserialization)**
*   **역할**: 이 테스트에서는 **HTTP 요청의 본문(Body)에 들어갈 JSON 데이터를 만드는 역할**을 합니다.
    *   `chargePoint` 테스트의 ` .content(objectMapper.writeValueAsString(amountToCharge))` 코드를 보세요.
    *   `@RequestBody`는 클라이언트가 JSON 형태의 데이터를 보낼 것이라고 기대합니다. 우리 테스트 코드에서 `amountToCharge`라는 `long` 타입 변수(예: `500L`)를 HTTP 요청 본문에 담으려면 `"500"` 이라는 JSON 형식의 문자열로 바꿔주어야 합니다.
    *   `objectMapper.writeValueAsString()`가 바로 이 역할을 수행합니다. Java의 `long` 타입을 JSON 문자열로 변환해주는 거죠. 만약 복잡한 DTO 객체였다면, `{"field1":"value1", "field2":123}` 같은 JSON 문자열로 깔끔하게 변환해줬을 겁니다.
*   **왜 쓰는가?**: 우리가 직접 `"{\"amount\":500}"` 같은 JSON 문자열을 조립하는 수고를 덜어주고, Java 객체를 사용해 훨씬 안전하고 편리하게 요청 데이터를 만들 수 있게 해줍니다.