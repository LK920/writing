# JPA Lazy Loading: 개념, 사용, 및 이커머스 프로젝트 적용 가이드

## 1. Lazy Loading이란?

Lazy Loading(지연 로딩)은 JPA에서 엔티티의 연관 관계 데이터를 필요할 때까지 데이터베이스에서 조회하지 않고, 실제로 접근할 때 조회하는 전략입니다. 반대 개념은 Eager Loading(즉시 로딩)으로, 엔티티를 조회할 때 연관 데이터도 함께 로드합니다.

### 비유
- Lazy Loading은 마치 책을 빌릴 때 목차만 먼저 받고, 특정 장을 읽고 싶을 때만 그 장의 내용을 가져오는 것과 같습니다. 반면, Eager Loading은 책 전체를 한 번에 받아오는 방식입니다.
- **이커머스 예시**: `User`를 조회할 때, 연관된 `orders` 목록은 필요할 때만 로드(Lazy)하거나, 즉시 로드(Eager)할 수 있습니다.

### Lazy Loading의 특징
- **기본 동작**: `@OneToMany`, `@ManyToMany`는 기본적으로 Lazy Loading, `@OneToOne`, `@ManyToOne`은 Eager Loading.
- **성능 이점**: 불필요한 데이터 조회를 줄여 초기 쿼리 성능 향상.
- **구현**: Hibernate가 프록시 객체를 사용해 실제 데이터 접근 시 쿼리 실행.

### 예시 코드
```java
@Entity
public class User {
    @Id
    private Long id;
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
    // Getters and Setters
}

@Entity
public class Order {
    @Id
    private Long id;
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
    private BigDecimal totalAmount;
    // Getters and Setters
}
```
- **동작**: `User`를 조회하면 `orders`는 프록시 객체로 유지되며, `user.getOrders()` 호출 시 실제 쿼리 실행.

---

## 2. Lazy Loading의 동작 방식

### 2.1. 동작 과정
1. **엔티티 조회**: `User`를 조회하면, `orders`는 프록시 객체로 로드됨(실제 데이터는 로드되지 않음).
   ```java
   User user = userRepository.findById(1L).orElseThrow();
   // 쿼리: SELECT * FROM users WHERE id = 1;
   ```
2. **연관 데이터 접근**: `user.getOrders()` 호출 시, 프록시가 실제 데이터 조회.
   ```java
   List<Order> orders = user.getOrders(); // 프록시 초기화
   // 쿼리: SELECT * FROM orders WHERE user_id = 1;
   ```

### 2.2. 이커머스에서의 Lazy Loading
- **시나리오**: `GET /users/1` API로 사용자를 조회할 때, `orders`는 필요하지 않으면 로드하지 않음.
- **쿼리 로그**:
  ```sql
  -- User 조회
  SELECT * FROM users WHERE id = 1;
  -- orders 접근 시
  SELECT * FROM orders WHERE user_id = 1;
  ```

---

## 3. Lazy Loading의 장점과 단점

### 3.1. 장점
- **성능 최적화**: 초기 조회 시 불필요한 연관 데이터 로드를 방지, 메모리와 쿼리 비용 절감.
  - 예: `User` 조회 시 `orders`를 로드하지 않아 빠른 응답.
- **유연성**: 필요한 데이터만 선택적으로 로드 가능.
- **이커머스 적용**: `GET /users` API에서 모든 사용자를 조회할 때, `orders`를 로드하지 않아 성능 향상.

### 3.2. 단점
- **N+1 문제**: 연관 데이터를 반복 접근하면 추가 쿼리가 발생.
  - 예: 100명의 `User`를 조회 후 각 `orders`를 접근하면 1(`users`) + 100(`orders`) = 101개 쿼리.
- **LazyInitializationException**: 영속성 컨텍스트 외부에서 프록시 접근 시 예외 발생.
  - 예: `@Transactional` 없이 `user.getOrders()` 호출.
- **복잡성**: 프록시 관리와 트랜잭션 범위 이해 필요.

### 3.3. 이커머스에서의 문제 사례
- **N+1 문제**: `GET /users` API로 사용자 목록을 조회하고, 각 사용자의 `orders`를 접근.
  ```java
  List<User> users = userRepository.findAll();
  users.forEach(u -> u.getOrders().size()); // N+1 쿼리 발생
  ```
- **LazyInitializationException**: `GET /orders/100` API에서 `Order.user`를 비트랜잭션 환경에서 접근.
  ```java
  Order order = orderRepository.findById(100L).orElseThrow();
  order.getUser().getId(); // LazyInitializationException
  ```

---

## 4. Lazy Loading 문제와 해결책

### 4.1. 문제 1: N+1 쿼리 문제
- **설명**: 다수 엔티티의 연관 데이터를 Lazy Loading으로 접근 시 추가 쿼리 발생.
- **이커머스 예시**: `GET /users` API에서 모든 `User`의 `orders`를 로드하며 쿼리 폭증.

#### 해결책
- **방법 1: `JOIN FETCH` 사용**
  - 단일 쿼리로 연관 데이터 로드.
  ```java
  @Repository
  public interface UserRepository extends JpaRepository<User, Long> {
      @Query("SELECT u FROM User u JOIN FETCH u.orders")
      List<User> findAllWithOrders();
  }
  ```
  - **결과 쿼리**:
    ```sql
    SELECT u.*, o.* FROM users u INNER JOIN orders o ON u.id = o.user_id;
    ```
- **방법 2: `EntityGraph` 사용**
  ```java
  @EntityGraph(attributePaths = {"orders"})
  List<User> findAll();
  ```
- **이커머스 적용**: `GET /users` API에서 `findAllWithOrders()`로 단일 쿼리 실행.
- **실습 코드**:
  ```java
  @Test
  void testAvoidNPlusOne() {
      List<User> users = userRepository.findAllWithOrders();
      users.forEach(u -> assertThat(u.getOrders()).isNotNull());
      // 쿼리 로그: 단일 JOIN 쿼리 확인
  }
  ```

### 4.2. 문제 2: LazyInitializationException
- **설명**: 영속성 컨텍스트가 닫힌 후 프록시 객체 접근 시 예외 발생.
- **이커머스 예시**: `GET /orders/100` API에서 `Order.user`를 비트랜잭션 환경에서 접근.
  ```java
  @RestController
  public class OrderController {
      @GetMapping("/orders/{id}")
      public Order getOrder(@PathVariable Long id) {
          return orderRepository.findById(id).orElseThrow();
      }
  }
  // 호출: order.getUser().getId() -> LazyInitializationException
  ```

#### 해결책
- **방법 1: `@Transactional` 사용**
  - 트랜잭션 내에서 프록시 초기화.
  ```java
  @RestController
  public class OrderController {
      @Autowired
      private OrderRepository orderRepository;

      @GetMapping("/orders/{id}")
      @Transactional(readOnly = true)
      public Order getOrder(@PathVariable Long id) {
          return orderRepository.findById(id).orElseThrow();
      }
  }
  ```
- **방법 2: Eager Loading으로 전환**
  ```java
  public class Order {
      @ManyToOne(fetch = FetchType.EAGER)
      @JoinColumn(name = "user_id")
      private User user;
  }
  ```
  - **주의**: Eager Loading은 불필요한 데이터 로드로 성능 저하 가능.
- **방법 3: DTO 사용**
  ```java
  public class OrderDto {
      private Long id;
      private Long userId;
      private BigDecimal totalAmount;
      public OrderDto(Order order) {
          this.id = order.getId();
          this.userId = order.getUser().getId();
          this.totalAmount = order.getTotalAmount();
      }
  }

  @RestController
  public class OrderController {
      @GetMapping("/orders/{id}")
      public OrderDto getOrder(@PathVariable Long id) {
          Order order = orderRepository.findById(id).orElseThrow();
          return new OrderDto(order);
      }
  }
  ```
- **이커머스 적용**: `GET /orders/100` API에서 `OrderDto`로 `user_id`만 반환, 프록시 접근 방지.
- **실습 코드**:
  ```java
  @Test
  void testOrderApi() {
      given()
          .when()
          .get("/orders/100")
          .then()
          .statusCode(200)
          .body("userId", equalTo(1));
  }
  ```

### 4.3. 문제 3: 프록시 객체의 복잡성
- **설명**: Lazy Loading은 프록시 객체를 사용하므로, 객체 상태(초기화 여부)를 관리해야 함.
- **이커머스 예시**: `Order.user`가 프록시인지 실제 객체인지 확인 필요.

#### 해결책
- **방법 1: Hibernate 초기화 확인**
  ```java
  import org.hibernate.Hibernate;

  public boolean isUserInitialized(Order order) {
      return Hibernate.isInitialized(order.getUser());
  }
  ```
- **방법 2: 명시적 초기화**
  ```java
  @Repository
  public interface OrderRepository extends JpaRepository<Order, Long> {
      @Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.id = :id")
      Optional<Order> findByIdWithUser(@Param("id") Long id);
  }
  ```
- **이커머스 적용**: `Order` 조회 시 `user`를 명시적으로 로드.
- **실습 코드**:
  ```java
  @Test
  void testOrderWithUser() {
      Order order = orderRepository.findByIdWithUser(100L).orElseThrow();
      assertThat(Hibernate.isInitialized(order.getUser())).isTrue();
  }
  ```

---

## 5. 이커머스 프로젝트에서의 Lazy Loading 적용

### 5.1. 시나리오: `User`와 `Order` 조회
- **상황**: `GET /users/1` API로 사용자 조회, `orders`는 필요 시만 로드.
- **설정**:
  ```java
  public class User {
      @Id
      private Long id;
      @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
      private List<Order> orders;
  }

  @RestController
  public class UserController {
      @Autowired
      private UserRepository userRepository;

      @GetMapping("/users/{id}")
      @Transactional(readOnly = true)
      public User getUser(@PathVariable Long id) {
          return userRepository.findById(id).orElseThrow();
      }
  }
  ```
- **결과**: `orders`는 `getUser().getOrders()` 호출 시 로드, 초기 쿼리 최소화.

### 5.2. 시나리오: `Order`와 `OrderItem`
- **상황**: `GET /orders/100` API로 주문 조회, `orderItems`는 Lazy Loading.
- **설정**:
  ```java
  public class Order {
      @Id
      private Long id;
      @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
      private List<OrderItem> orderItems;
  }

  @Repository
  public interface OrderRepository extends JpaRepository<Order, Long> {
      @Query("SELECT o FROM Order o JOIN FETCH o.orderItems WHERE o.id = :id")
      Optional<Order> findByIdWithItems(@Param("id") Long id);
  }
  ```
- **결과**: `findByIdWithItems`로 `orderItems`를 단일 쿼리로 로드, N+1 방지.

---

## 6. 학습자 팁

- **시작하기**:
  - Spring Boot 프로젝트에서 `User`와 `Order`로 Lazy Loading 설정.
  - `logging.level.org.hibernate.SQL=DEBUG`로 쿼리 로그 확인.
- **실습 예제**:
  - N+1 문제 재현:
    ```java
    @Test
    void testNPlusOne() {
        List<User> users = userRepository.findAll();
        users.forEach(u -> u.getOrders().size()); // N+1 발생
        // 쿼리 로그: SELECT * FROM orders ... (여러 번)
    }
    ```
  - `JOIN FETCH`로 해결:
    ```java
    @Test
    void testJoinFetch() {
        List<User> users = userRepository.findAllWithOrders();
        users.forEach(u -> assertThat(u.getOrders()).isNotNull());
        // 쿼리 로그: 단일 JOIN 쿼리
    }
    ```
  - LazyInitializationException 테스트:
    ```java
    @Test
    void testLazyException() {
        Order order = orderRepository.findById(100L).orElseThrow();
        assertThatThrownBy(() -> order.getUser().getId())
            .isInstanceOf(LazyInitializationException.class);
    }
    ```
  - RestAssured로 API 테스트:
    ```java
    @Test
    void testOrderApi() {
        given()
            .when()
            .get("/orders/100")
            .then()
            .statusCode(200)
            .body("id", equalTo(100));
    }
    ```
- **도구**: Testcontainers로 PostgreSQL 테스트, IntelliJ 디버거로 프록시 분석.
- **추천 학습 경로**:
  1. Lazy Loading으로 `User`와 `Order` 설정, 쿼리 로그 확인.
  2. N+1 문제 재현 및 `JOIN FETCH`로 해결.
  3. `@Transactional`으로 LazyInitializationException 해결.
  4. DTO로 직렬화 문제 방지 실습.

---

## 7. 요약

Lazy Loading은 JPA에서 연관 데이터를 필요할 때만 로드하여 성능을 최적화하지만, N+1 문제와 LazyInitializationException을 유발할 수 있습니다. 이커머스 프로젝트에서는 `User`와 `Order`, `Order`와 `OrderItem` 관계에 Lazy Loading을 적용해 초기 쿼리를 줄이고, `JOIN FETCH`, `@Transactional`, DTO로 문제를 해결할 수 있습니다. 초보자는 쿼리 로그를 분석하며 Lazy Loading 동작을 이해하고, 실습으로 N+1 문제를 해결해보세요.

### 추가 참고 자료
- Spring Data JPA 문서: https://spring.io/projects/spring-data-jpa
- Hibernate ORM 문서: https://hibernate.org/orm/documentation/
- Baeldung Lazy Loading 가이드: https://www.baeldung.com/hibernate-lazy-eager-loading
- Vlad Mihalcea 블로그: https://vladmihalcea.com/jpa-hibernate-lazy-loading/