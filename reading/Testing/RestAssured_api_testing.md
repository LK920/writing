# RestAssured로 API 테스트 배우기: 초보자를 위한 상세 가이드

## 1. RestAssured란?

RestAssured는 Java 기반의 오픈소스 라이브러리로, RESTful API(웹 서비스)를 테스트하기 위해 설계되었습니다. RESTful API는 HTTP 프로토콜을 통해 데이터를 주고받는 인터페이스로, 현대 웹 애플리케이션의 핵심입니다. RestAssured는 간결한 문법(도메인 특화 언어, DSL)을 제공하여 API 요청을 보내고 응답을 검증하는 테스트 코드를 쉽게 작성할 수 있도록 합니다.

### 비유
- RestAssured는 마치 레스토랑에서 주문서를 작성하고 요리(응답)를 확인하는 웨이터와 같습니다. 주문서(HTTP 요청)를 작성해 서버에 전달하고, 나온 요리가 주문과 일치하는지(응답 검증) 확인하는 과정을 간단히 처리해줍니다.
- 예: 이커머스 프로젝트에서 `orders` API를 테스트하려면, RestAssured로 주문 생성 요청(POST)을 보내고, 응답 상태 코드(200 OK)와 주문 데이터가 올바른지 확인할 수 있습니다.

### RestAssured의 주요 목적
- **자동화 테스트**: API의 기능, 성능, 보안을 자동으로 검증.
- **가독성**: BDD(행위 주도 개발) 스타일의 문법(given-when-then)으로 읽기 쉬운 테스트 작성.
- **유연성**: JSON, XML 응답 처리 및 다양한 HTTP 메서드 지원.

---

## 2. RestAssured의 핵심 기능

RestAssured는 API 테스트를 위한 다양한 기능을 제공합니다. 초보자가 이해해야 할 핵심 기능은 다음과 같습니다:

### 2.1. HTTP 메서드 지원
- GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS 등 모든 HTTP 메서드를 지원.
- 예: `GET /orders/123`로 특정 주문 조회, `POST /orders`로 새 주문 생성.

### 2.2. BDD 스타일 문법
- `given()`: 요청 설정(헤더, 파라미터, 바디 등).
- `when()`: 요청 실행(HTTP 메서드 호출).
- `then()`: 응답 검증(상태 코드, 바디, 헤더 등).
- 예:
  ```java
  given().when().get("/orders/123").then().statusCode(200);
  ```

### 2.3. 응답 검증
- 상태 코드(예: 200 OK, 404 Not Found), 헤더, 바디(JSON/XML), 응답 시간 등을 검증.
- Hamcrest 매처를 사용해 직관적인 검증 가능.
- 예: `body("order.id", equalTo(123))`.

### 2.4. 통합성
- JUnit, TestNG와 같은 테스트 프레임워크와 통합.
- Maven, Gradle과 같은 빌드 도구 지원.
- Serenity, Cucumber 같은 BDD 프레임워크와도 통합 가능.

### 2.5. JSON/XML 파싱
- JSONPath, XMLPath를 사용해 복잡한 응답 데이터 파싱.
- 예: `response.jsonPath().getInt("order.id")`.

---

## 3. RestAssured의 장점

1. **간결한 문법**:
   - 직관적인 DSL로 코드가 간단하고 읽기 쉬움.
   - 예: `given().header("Authorization", "Bearer token").when().get("/api").then().statusCode(200);`

2. **강력한 검증**:
   - 상태 코드, 헤더, 바디, 응답 시간을 한 번에 검증.
   - Hamcrest 매처로 다양한 조건 검사 가능(예: `equalTo`, `hasItems`).

3. **통합 용이성**:
   - JUnit/TestNG와 통합하여 CI/CD 파이프라인에 쉽게 포함.
   - Maven/Gradle로 의존성 관리 간단.

4. **커뮤니티 지원**:
   - 오픈소스로 활발한 커뮤니티와 풍부한 문서 제공.
   - 최신 버전(2025년 기준 5.5.x)에서 지속적인 업데이트.[](https://rest-assured.io/)

---

## 4. RestAssured의 단점

1. **Java 기반 한계**:
   - Java에 익숙하지 않은 사용자는 학습 곡선이 있음.
   - Postman 같은 GUI 도구에 비해 비기술적 사용자에게 불편.

2. **SOAP 지원 제한**:
   - 주로 RESTful API 테스트에 특화. SOAP 테스트는 추가 설정 필요.[](https://www.frugaltesting.com/blog/how-to-perform-data-driven-api-testing-with-rest-assured)

3. **복잡한 테스트의 오버헤드**:
   - 대규모 테스트 스위트에서는 성능 오버헤드 가능.
   - 간단한 테스트에는 과도한 추상화로 느껴질 수 있음.[](https://www.linkedin.com/pulse/pros-cons-different-api-test-tools-restassured-craig-risi)

4. **GUI 부재**:
   - Postman처럼 직관적인 UI가 없어 수동 테스트에 비해 디버깅이 어려울 수 있음.

---

## 5. RestAssured 설정 및 설치

RestAssured를 사용하려면 Java 프로젝트에 라이브러리를 추가해야 합니다. 아래는 Maven을 사용한 설정 예제입니다.

### 5.1. 전제 조건
- **Java**: JDK 8 이상 설치.
- **IDE**: IntelliJ, Eclipse 등.
- **빌드 도구**: Maven 또는 Gradle.
- **테스트 프레임워크**: JUnit 5 또는 TestNG.

### 5.2. Maven 의존성 추가
`pom.xml`에 다음 의존성을 추가합니다:
```xml
<dependencies>
    <!-- RestAssured -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>5.5.5</version>
        <scope>test</scope>
    </dependency>
    <!-- TestNG -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.9.0</version>
        <scope>test</scope>
    </dependency>
    <!-- JSON Simple (선택적, JSON 파싱용) -->
    <dependency>
        <groupId>com.googlecode.json-simple</groupId>
        <artifactId>json-simple</artifactId>
        <version>1.1.1</version>
    </dependency>
</dependencies>
```
- 최신 버전은 Maven Repository에서 확인.[](https://jignect.tech/rest-assured-basics-a-beginners-guide-to-automated-api-testing-in-java/)

### 5.3. 프로젝트 구조
```
src
└── test
    └── java
        └── com.example.tests
            └── OrderApiTest.java
```

---

## 6. 이커머스 프로젝트에서의 RestAssured 적용 예시

이커머스 프로젝트의 ERD(예: `orders`, `users`, `products`)를 기반으로 RestAssured를 사용한 API 테스트 예제를 작성해 보겠습니다.

### 6.1. 시나리오: 주문 조회 API 테스트
- **API**: `GET /orders/{orderId}`
- **목표**: 주문 ID로 주문을 조회하고, 상태 코드(200)와 주문 데이터(주문 ID, 사용자 ID, 총액)를 검증.
- **예상 응답**:
  ```json
  {
      "order": {
          "id": 123,
          "user_id": 456,
          "total_amount": 99.99,
          "status": "PENDING"
      }
  }
  ```

### 6.2. 테스트 코드
```java
import io.restassured.RestAssured;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

public class OrderApiTest {
    @BeforeClass
    public void setup() {
        RestAssured.baseURI = "https://api.example.com";
        RestAssured.basePath = "/v1";
    }

    @Test
    public void testGetOrder() {
        given()
            .pathParam("orderId", 123)
            .header("Authorization", "Bearer your_token")
        .when()
            .get("/orders/{orderId}")
        .then()
            .statusCode(200)
            .body("order.id", equalTo(123))
            .body("order.user_id", equalTo(456))
            .body("order.total_amount", equalTo(99.99f))
            .body("order.status", equalTo("PENDING"));
    }
}
```
- **설명**:
  - `@BeforeClass`: 모든 테스트 전에 기본 URI와 경로 설정.
  - `given()`: 요청에 필요한 파라미터(`orderId`)와 헤더(인증 토큰) 설정.
  - `when()`: GET 요청 실행.
  - `then()`: 응답 상태 코드와 JSON 바디 검증.

### 6.3. 시나리오: 주문 생성 API 테스트
- **API**: `POST /orders`
- **목표**: 새 주문을 생성하고, 응답 상태 코드(201)와 생성된 주문 ID를 검증.
- **요청 바디**:
  ```json
  {
      "user_id": 456,
      "items": [
          {"product_id": 789, "quantity": 2}
      ],
      "total_amount": 199.98
  }
  ```

### 6.4. 테스트 코드
```java
import io.restassured.RestAssured;
import io.restassured.http.ContentType;
import org.testng.annotations.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

public class OrderApiTest {
    @Test
    public void testCreateOrder() {
        String requestBody = "{\"user_id\": 456, \"items\": [{\"product_id\": 789, \"quantity\": 2}], \"total_amount\": 199.98}";

        given()
            .contentType(ContentType.JSON)
            .body(requestBody)
        .when()
            .post("/orders")
        .then()
            .statusCode(201)
            .body("order.id", notNullValue())
            .body("order.user_id", equalTo(456))
            .body("order.total_amount", equalTo(199.98f));
    }
}
```
- **설명**:
  - `contentType()`: 요청 바디가 JSON임을 지정.
  - `body()`: JSON 문자열로 요청 바디 전송.
  - `notNullValue()`: 생성된 주문 ID가 null이 아닌지 확인.

---

## 7. 데이터 주도 테스트 (Data-Driven Testing)

이커머스 프로젝트에서 다양한 사용자 ID로 주문 조회를 테스트하려면 데이터 주도 테스트가 유용합니다. TestNG의 `@DataProvider`를 사용한 예제입니다.

### 7.1. 데이터 주도 테스트 코드
```java
import io.restassured.RestAssured;
import org.testng.annotations.DataProvider;
import org.testng.annotations.Test;
import static io.restassured.RestAssured.*;
import static org.hamcrest.Matchers.*;

public class OrderApiTest {
    @DataProvider(name = "orderData")
    public Object[][] orderData() {
        return new Object[][] {
            {123, 456, 99.99f},
            {124, 457, 149.99f}
        };
    }

    @Test(dataProvider = "orderData")
    public void testGetOrderWithData(int orderId, int userId, float totalAmount) {
        given()
            .pathParam("orderId", orderId)
        .when()
            .get("/orders/{orderId}")
        .then()
            .statusCode(200)
            .body("order.id", equalTo(orderId))
            .body("order.user_id", equalTo(userId))
            .body("order.total_amount", equalTo(totalAmount));
    }
}
```
- **설명**:
  - `@DataProvider`: 테스트 데이터를 배열로 제공.
  - 여러 주문 ID, 사용자 ID, 총액 조합을 한 번에 테스트하여 테스트 커버리지 증가.

---

## 8. 이커머스 프로젝트에 RestAssured 적용 시 고려사항

1. **인증 처리**:
   - 이커머스 API는 보통 JWT 토큰이나 API 키를 요구. `header("Authorization", "Bearer token")`로 처리.
   - 예: `coupons` API 테스트 시 인증 헤더 포함.

2. **재고 관리 테스트**:
   - `products` 테이블의 `stock`과 `reserved_stock`을 테스트하려면, 재고 감소 API(`POST /orders`)를 호출하고 응답으로 재고 변화를 검증.
   - 예: `body("product.stock", lessThanOrEqualTo(100))`.

3. **동시성 테스트**:
   - 낙관적 락(`version` 필드)이나 비관적 락을 사용하는 API는 여러 요청을 병렬로 보내어 동시성 문제 테스트.
   - RestAssured는 `time(lessThan(2000L))`로 응답 시간 검증 가능.

4. **에러 처리**:
   - 잘못된 입력(예: 유효하지 않은 `user_id`)으로 400/404 상태 코드 테스트.
   - 예: `given().body("{\"user_id\": -1}").when().post("/orders").then().statusCode(400);`

---

## 9. 학습자 팁

- **시작하기**:
  - 간단한 GET 요청 테스트부터 작성(예: `GET /orders/123`).
  - 공식 문서(https://rest-assured.io)와 GitHub 예제를 참고.[](https://rest-assured.io/)

- **실습 예제**:
  - 이커머스 API(예: https://reqres.in/api/users)로 연습.
  - JSONPlaceholder(https://jsonplaceholder.typicode.com)로 무료 테스트 가능.[](https://www.frugaltesting.com/blog/rest-assured-api-testing-tutorial-with-testng-and-junit)

- **디버깅**:
  - `log().all()`로 요청/응답 상세 로그 출력.
  - 예: `given().log().all().when().get("/orders").then().log().all();`

- **도구 통합**:
  - Allure 또는 Extent Report로 테스트 보고서 생성.
  - Jenkins로 CI/CD 파이프라인에 통합.

- **추천 학습 경로**:
  1. 기본 GET/POST 테스트 작성.
  2. JSONPath로 복잡한 응답 파싱 연습.
  3. 데이터 주도 테스트 구현.
  4. 인증 및 에러 처리 테스트 추가.

---

## 10. 요약

RestAssured는 RESTful API 테스트를 위한 강력하고 직관적인 Java 라이브러리로, 이커머스 프로젝트의 `orders`, `products`, `coupons` 같은 API를 효과적으로 테스트할 수 있습니다. 간결한 문법, 강력한 검증 기능, JUnit/TestNG 통합으로 초보자도 쉽게 시작할 수 있습니다. 이 자료를 통해 기본 설정, 테스트 작성, 데이터 주도 테스트를 연습하고, 실제 이커머스 API에 적용해보세요.

### 추가 참고 자료
- 공식 문서: https://rest-assured.io[](https://rest-assured.io/)
- GitHub: https://github.com/rest-assured/rest-assured
- 튜토리얼: Baeldung REST-assured 가이드[](https://www.baeldung.com/rest-assured-tutorial)
- 무료 API: https://jsonplaceholder.typicode.com[](https://www.frugaltesting.com/blog/rest-assured-api-testing-tutorial-with-testng-and-junit)