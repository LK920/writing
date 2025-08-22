# JPA `@OneToMany` vs `@Embeddable`: 비교 및 값 객체 사용 가이드

## 1. `@OneToMany`와 `@Embeddable`란?

JPA에서 `@OneToMany`와 `@Embeddable`은 엔티티 간 관계와 데이터 구조를 정의하는 두 가지 방식입니다. 이커머스 프로젝트에서 어떤 상황에 적합한지 비교하기 위해 각각의 개념을 먼저 설명합니다.

### 1.1. `@OneToMany`
- **정의**: 한 엔티티가 다른 엔티티의 컬렉션을 참조하는 관계. 예: 한 `User`가 여러 `Order`를 가짐.
- **특징**:
  - 별도의 테이블로 저장(예: `orders` 테이블).
  - 외래 키(FK)로 관계 정의(예: `orders.user_id` → `users.id`).
  - 양방향 매핑 가능(예: `Order`에서 `User` 참조).
- **비유**: `@OneToMany`는 한 명의 부모(`User`)가 여러 자식(`Order`)을 관리하는 가정과 같습니다. 자식은 독립적인 존재로, 부모와 별도로 생애 주기를 가질 수 있습니다.
- **예시**:
  ```java
  @Entity
  public class User {
      @Id
      private Long id;
      @OneToMany(mappedBy = "user")
      private List<Order> orders = new ArrayList<>();
  }

  @Entity
  public class Order {
      @Id
      private Long id;
      @ManyToOne
      @JoinColumn(name = "user_id")
      private User user;
      private BigDecimal totalAmount;
  }
  ```

### 1.2. `@Embeddable`과 `@Embedded`
- **정의**: 값 객체(Value Object)를 정의해 엔티티 내에 포함시키는 방식. 값 객체는 독립적인 생애 주기를 갖지 않고, 부모 엔티티에 완전히 종속됩니다.
- **특징**:
  - 별도 테이블 생성 없이 부모 엔티티의 테이블에 컬럼으로 저장.
  - 값 객체는 불변(immutable)하거나, 부모 엔티티의 생애 주기에 의존.
  - 주로 복합 데이터(예: 주소, 좌표)를 구조화.
- **비유**: `@Embeddable`은 사람의 신체 일부(예: 이름, 생년월일)와 같습니다. 이들은 독립적으로 존재하지 않고, 사람(`User`)이라는 엔티티에 포함됩니다.
- **예시**:
  ```java
  @Embeddable
  public class Address {
      private String street;
      private String city;
      private String zipCode;
  }

  @Entity
  public class User {
      @Id
      private Long id;
      @Embedded
      private Address address;
  }
  ```
  - **DB 테이블**:
    ```sql
    CREATE TABLE users (
        id BIGINT PRIMARY KEY,
        street VARCHAR(255),
        city VARCHAR(255),
        zip_code VARCHAR(10)
    );
    ```

---

## 2. `@OneToMany`와 `@Embeddable` 비교

| 항목 | `@OneToMany` | `@Embeddable` |
|------|--------------|---------------|
| **용도** | 엔티티 간 1:N 관계 정의(예: `User`와 `Order`) | 값 객체를 엔티티 내에 포함(예: `User`의 `Address`) |
| **저장 방식** | 별도 테이블에 저장, FK로 연결 | 부모 테이블의 컬럼으로 저장 |
| **생애 주기** | 자식 엔티티는 독립적 생애 주기 가능 | 값 객체는 부모 엔티티에 완전히 종속 |
| **쿼리** | 조인 쿼리 필요(예: `SELECT * FROM orders WHERE user_id = ?`) | 단일 테이블 쿼리(예: `SELECT street, city FROM users`) |
| **복잡성** | 양방향 매핑 시 동기화 관리 필요 | 동기화 불필요, 간단한 관리 |
| **성능** | N+1 문제, 조인 오버헤드 가능 | 조인 없음, 성능 우수 |
| **유연성** | 관계 수정/확장 용이 | 값 객체는 구조 변경 시 제약 |
| **이커머스 예시** | `User`와 `Order` 관계 | `Order`의 배송지 주소 |

### 이커머스에서의 비교
- **@OneToMany**: `User`가 여러 `Order`를 가지는 관계. `orders` 테이블은 독립적으로 관리되며, 주문은 사용자 없이도 존재 가능(예: 삭제된 사용자에 대한 주문 보존).
- **@Embeddable**: `Order`의 `Address` (배송지 정보). 주소는 주문에 포함되어 별도 테이블 없이 `orders` 테이블에 컬럼으로 저장.

---

## 3. 코치의 조언: "값 객체면 `@Embeddable`이 낫다"

코치가 말한 "값 객체(Value Object)"는 DDD(Domain-Driven Design)에서 고유 식별자(ID)가 없고, 부모 엔티티에 종속적인 객체를 의미합니다. `@Embeddable`이 값 객체에 적합한 이유와 이커머스 적용 사례를 살펴보겠습니다.

### 3.1. 값 객체의 특징
- **고유 식별자 없음**: 예: 주소는 독립적인 ID가 없고, `Order`나 `User`에 속함.
- **불변성**: 값 객체는 보통 불변(immutable)하거나, 부모 엔티티의 생애 주기에 의존.
- **의미적 통합**: 관련 속성을 묶어 의미를 명확히 함(예: `street`, `city`, `zipCode`를 `Address`로 통합).
- **이커머스 예시**: `Order`의 배송지 주소, 결제 정보(`PaymentInfo`), 쿠폰 적용 정보(`CouponDetails`).

### 3.2. 왜 `@Embeddable`이 값 객체에 적합한가?
- **단순성**: 별도 테이블 관리 없이 부모 엔티티에 포함, 코드와 DB 스키마 간소화.
- **성능**: 조인 쿼리 불필요, 단일 테이블 조회로 성능 향상.
- **의미적 명확성**: 값 객체를 클래스로 구조화해 도메인 모델 표현력 강화.
- **예시**: `Order`의 `Address`를 `@Embeddable`로 정의하면, `orders` 테이블에 `street`, `city`, `zip_code` 컬럼으로 저장되어 조인 없이 조회 가능.

### 3.3. `@OneToMany`와의 비교 (값 객체 관점)
- **@OneToMany**: 값 객체가 아닌 독립적 엔티티(예: `Order`, `OrderItem`)에 적합. 예: `Order`는 고유 ID를 가지며, `User` 없이도 존재 가능.
- **@Embeddable**: 값 객체(예: `Address`, `PaymentInfo`)에 적합. 독립적이지 않고, 부모 엔티티(`Order`)에 종속.
- **이커머스 판단 기준**:
  - 독립적 생애 주기, 고유 ID가 필요하면 `@OneToMany` (예: `Order`, `OrderItem`).
  - 부모 엔티티에 종속적이고 ID가 필요 없으면 `@Embeddable` (예: `Address`).

---

## 4. 이커머스 프로젝트에서의 적용 사례

### 4.1. `@OneToMany`: `User`와 `Order`
- **상황**: 한 `User`가 여러 `Order`를 가짐.
- **설정**:
  ```java
  @Entity
  public class User {
      @Id
      private Long id;
      @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
      private List<Order> orders = new ArrayList<>();
      public void addOrder(Order order) {
          orders.add(order);
          order.setUser(this);
      }
  }

  @Entity
  public class Order {
      @Id
      private Long id;
      @ManyToOne
      @JoinColumn(name = "user_id")
      private User user;
      private BigDecimal totalAmount;
  }
  ```
- **이유**: `Order`는 고유 ID를 가지며, 독립적으로 관리 가능(예: 사용자 삭제 후 주문 보존).
- **DB 테이블**:
  ```sql
  CREATE TABLE orders (
      id BIGINT PRIMARY KEY,
      user_id BIGINT,
      total_amount DECIMAL(10,2),
      FOREIGN KEY (user_id) REFERENCES users(id)
  );
  ```

### 4.2. `@Embeddable`: `Order`의 `Address`
- **상황**: `Order`의 배송지 정보를 값 객체로 관리.
- **설정**:
  ```java
  @Embeddable
  public class Address {
      private String street;
      private String city;
      private String zipCode;
      // Getters and Setters
  }

  @Entity
  public class Order {
      @Id
      private Long id;
      @Embedded
      private Address deliveryAddress;
      private BigDecimal totalAmount;
  }
  ```
- **이유**: `Address`는 독립적 ID가 없고, `Order`에 완전히 종속. 별도 테이블 대신 `orders` 테이블에 컬럼으로 저장.
- **DB 테이블**:
  ```sql
  CREATE TABLE orders (
      id BIGINT PRIMARY KEY,
      street VARCHAR(255),
      city VARCHAR(255),
      zip_code VARCHAR(10),
      total_amount DECIMAL(10,2)
  );
  ```

### 4.3. 코치의 조언 적용: `CouponDetails` as 값 객체
- **상황**: `Order`에 적용된 쿠폰 정보(예: 할인율, 쿠폰 코드)를 값 객체로 관리.
- **왜 `@Embeddable`?**: 쿠폰 정보는 `Order`에 종속적이며, 독립적 ID가 불필요.
- **설정**:
  ```java
  @Embeddable
  public class CouponDetails {
      private String couponCode;
      private BigDecimal discountRate;
      // Getters and Setters
  }

  @Entity
  public class Order {
      @Id
      private Long id;
      @Embedded
      private CouponDetails couponDetails;
      private BigDecimal totalAmount;
  }
  ```
- **DB 테이블**:
  ```sql
  CREATE TABLE orders (
      id BIGINT PRIMARY KEY,
      coupon_code VARCHAR(50),
      discount_rate DECIMAL(5,2),
      total_amount DECIMAL(10,2)
  );
  ```
- **이점**: `CouponDetails`를 별도 테이블로 관리(`@OneToMany`)할 필요 없이, `orders` 테이블에 간단히 저장.

---

## 5. 문제와 해결책: `@OneToMany`와 `@Embeddable` 비교

### 5.1. 문제: 성능 (N+1 쿼리)
- **@OneToMany**:
  - **문제**: `User` 조회 시 `orders`를 로드하며 추가 쿼리 발생.
  - **해결책**: `JOIN FETCH` 또는 `EntityGraph` 사용.
    ```java
    @Query("SELECT u FROM User u JOIN FETCH u.orders")
    List<User> findAllWithOrders();
    ```
- **@Embeddable**:
  - **문제 없음**: `Address`는 `orders` 테이블에 포함, 조인 불필요.
  - **이커머스 적용**: `Order`의 `deliveryAddress` 조회 시 단일 쿼리.
    ```java
    Order order = orderRepository.findById(1L).orElseThrow();
    assertThat(order.getDeliveryAddress().getCity()).isEqualTo("Seoul");
    ```

### 5.2. 문제: 데이터 관리 복잡성
- **@OneToMany**:
  - **문제**: 양방향 매핑 시 동기화 필요(예: `User.orders`와 `Order.user`).
  - **해결책**: 동기화 메서드 추가.
    ```java
    public class User {
        public void addOrder(Order order) {
            orders.add(order);
            order.setUser(this);
        }
    }
    ```
- **@Embeddable**:
  - **문제 없음**: `Address`는 `Order`에 포함, 동기화 불필요.
  - **이커머스 적용**: `Order`에 `CouponDetails` 설정 시 간단히 저장.
    ```java
    Order order = new Order();
    CouponDetails coupon = new CouponDetails("TEST10", new BigDecimal("0.1"));
    order.setCouponDetails(coupon);
    orderRepository.save(order);
    ```

### 5.3. 문제: JSON 직렬화
- **@OneToMany**:
  - **문제**: 양방향 관계로 무한 루프(예: `User` → `Order` → `User`).
  - **해결책**: `@JsonManagedReference`와 DTO 사용.
    ```java
    public class UserDto {
        private Long id;
        private List<Long> orderIds;
    }
    ```
- **@Embeddable**:
  - **문제 없음**: `Address`는 엔티티가 아니므로 직렬화 간단.
  - **이커머스 적용**: `Order`의 `deliveryAddress`를 JSON으로 직렬화.
    ```json
    {
      "id": 1,
      "deliveryAddress": {
        "street": "123 Main St",
        "city": "Seoul",
        "zipCode": "12345"
      }
    }
    ```

---

## 6. RestAssured로 테스트

`@OneToMany`와 `@Embeddable`의 동작을 검증하는 RestAssured 테스트입니다.

### 6.1. `@OneToMany` 테스트: `User`와 `Order`
```java
@Test
void testGetUserWithOrders() {
    given()
        .when()
        .get("/users/1")
        .then()
        .statusCode(200)
        .body("id", equalTo(1))
        .body("orderIds", hasItem(100));
}
```

### 6.2. `@Embeddable` 테스트: `Order`의 `Address`
```java
@Test
void testGetOrderWithAddress() {
    given()
        .when()
        .get("/orders/1")
        .then()
        .statusCode(200)
        .body("deliveryAddress.city", equalTo("Seoul"));
}
```

---

## 7. 학습자 팁

- **시작하기**:
  - Spring Boot 프로젝트에서 `User`와 `Order`로 `@OneToMany`, `Order`와 `Address`로 `@Embeddable` 설정.
  - `logging.level.org.hibernate.SQL=DEBUG`로 쿼리 확인.
- **실습 예제**:
  - `@OneToMany` N+1 문제 재현 및 해결:
    ```java
    @Test
    void testNPlusOne() {
        List<User> users = userRepository.findAll();
        users.forEach(u -> u.getOrders().size()); // N+1 발생
        List<User> usersWithFetch = userRepository.findAllWithOrders();
        assertThat(usersWithFetch.get(0).getOrders()).isNotEmpty();
    }
    ```
  - `@Embeddable`로 `Address` 저장 테스트:
    ```java
    @Test
    void testSaveOrderWithAddress() {
        Order order = new Order();
        order.setId(1L);
        order.setDeliveryAddress(new Address("123 Main St", "Seoul", "12345"));
        orderRepository.save(order);
        Order saved = orderRepository.findById(1L).orElseThrow();
        assertThat(saved.getDeliveryAddress().getCity()).isEqualTo("Seoul");
    }
    ```
- **도구**: Testcontainers로 PostgreSQL 테스트, IntelliJ 디버거로 쿼리 분석.
- **추천 학습 경로**:
  1. `@OneToMany`로 `User`와 `Order` 설정, N+1 문제 확인.
  2. `@Embeddable`로 `Address` 설정, 단일 쿼리 확인.
  3. RestAssured로 API 응답 검증.
  4. 값 객체와 엔티티의 차이 분석.

---

## 8. 요약

`@OneToMany`는 독립적 엔티티(예: `Order`)의 1:N 관계에 적합하며, 외래 키와 별도 테이블로 관리됩니다. 반면, `@Embeddable`은 값 객체(예: `Address`, `CouponDetails`)에 적합하며, 부모 엔티티 테이블에 포함되어 조인 없이 조회 가능합니다. 코치의 조언처럼, 값 객체(독립 ID 불필요, 부모에 종속)에는 `@Embeddable`이 더 간단하고 효율적입니다. 이커머스 프로젝트에서는 `User`와 `Order`에 `@OneToMany`, `Order`의 배송지나 쿠폰 정보에 `@Embeddable`을 사용하세요. 실습으로 쿼리와 직렬화 차이를 비교해보세요.

### 추가 참고 자료
- Spring Data JPA 문서: https://spring.io/projects/spring-data-jpa
- Hibernate ORM 문서: https://hibernate.org/orm/documentation/
- Baeldung JPA 가이드: https://www.baeldung.com/jpa-embedded-and-embeddable
- Vlad Mihalcea 블로그: https://vladmihalcea.com/jpa-embeddable/