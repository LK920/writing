# JPA N+1 문제와 최적화 방안 학습 자료

## 1. N+1 문제란?

JPA(Java Persistence API)에서 **N+1 문제**는 엔티티를 조회할 때, 초기 쿼리(1번)와 연관된 엔티티를 조회하기 위한 추가 쿼리(N번)가 실행되어 총 **N+1번의 쿼리**가 발생하는 성능 문제를 말합니다. 이는 주로 **지연 로딩(Lazy Loading)** 설정에서 발생하며, 불필요한 쿼리로 인해 데이터베이스 부하가 증가하고 응답 시간이 길어질 수 있습니다.

### 기본 메커니즘
- **1번 쿼리**: 부모 엔티티 목록을 조회하는 쿼리.
- **N번 쿼리**: 부모 엔티티 각각에 대해 연관된 자식 엔티티를 조회하는 쿼리.
- 결과적으로, 부모 엔티티가 N개일 경우 N+1번의 쿼리가 실행.

### 왜 문제인가?
- **성능 저하**: 쿼리 실행 횟수가 많아지면 데이터베이스와 네트워크 I/O 부하 증가.
- **응답 시간 증가**: 다중 쿼리로 인해 애플리케이션 응답이 느려짐.
- **스케일링 문제**: 대량 데이터 처리 시 성능 병목 현상 심화.

---

## 2. N+1 문제가 발생하는 주요 사례

N+1 문제는 주로 연관 관계(OneToMany, ManyToOne 등)에서 지연 로딩을 사용할 때 발생합니다. 아래는 대표적인 사례들입니다.

### 사례 1: OneToMany 관계에서 지연 로딩
**상황**: `User` 엔티티가 `Order` 엔티티와 1:N 관계를 가짐. 모든 사용자를 조회한 후 각 사용자의 주문 목록을 접근.
```java
@Entity
public class User {
    @Id private Long id;
    private String name;
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;
}

@Entity
public class Order {
    @Id private Long id;
    private String item;
    @ManyToOne(fetch = FetchType.LAZY)
    private User user;
}
```
**코드**:
```java
List<User> users = userRepository.findAll();
for (User user : users) {
    System.out.println(user.getOrders().size()); // orders 접근
}
```
- **문제**:
  - `findAll()`로 사용자 목록 조회 (1번 쿼리).
  - 각 사용자의 `orders` 접근 시마다 추가 쿼리 실행 (N번 쿼리).
  - 결과: 사용자 100명 → 1 + 100 = 101번 쿼리.

### 사례 2: ManyToOne 관계에서 반복 접근
**상황**: `Order` 엔티티가 `User`와 N:1 관계. 모든 주문을 조회한 후 각 주문의 사용자 정보 접근.
```java
List<Order> orders = orderRepository.findAll();
for (Order order : orders) {
    System.out.println(order.getUser().getName()); // user 접근
}
```
- **문제**:
  - `findAll()`로 주문 목록 조회 (1번 쿼리).
  - 각 주문의 `user` 접근 시마다 추가 쿼리 실행 (N번 쿼리).
  - 결과: 주문 1000개 → 1 + 1000 = 1001번 쿼리.

### 사례 3: 컬렉션 조회 후 순회
**상황**: 특정 조건으로 엔티티 목록을 조회한 후, 연관된 컬렉션을 순회하며 추가 데이터 접근.
```java
List<User> users = userRepository.findByAgeGreaterThan(20);
for (User user : users) {
    for (Order order : user.getOrders()) {
        System.out.println(order.getItem());
    }
}
```
- **문제**: `orders` 컬렉션 접근 시 각 사용자마다 쿼리 실행, 반복문 내에서 추가 쿼리 발생.

### 사례 4: API 응답 생성 시
**상황**: REST API에서 DTO로 변환하면서 연관 엔티티 접근.
```java
public List<UserDto> getAllUsers() {
    List<User> users = userRepository.findAll();
    return users.stream()
                .map(user -> new UserDto(user.getName(), user.getOrders().size()))
                .collect(Collectors.toList());
}
```
- **문제**: `user.getOrders()` 호출 시마다 추가 쿼리 실행.

---

## 3. N+1 문제의 최적화 방안

N+1 문제를 해결하기 위해 다양한 최적화 기법을 사용할 수 있습니다. 아래는 주요 방법과 각각의 장단점, 예제 코드입니다.

### 3.1 FetchType.EAGER 사용
**설명**: 연관 관계를 `FetchType.EAGER`로 설정하여 초기 쿼리에서 모든 연관 데이터를 함께 조회.
```java
@OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
private List<Order> orders;
```
- **쿼리**: `JOIN`을 사용해 사용자와 주문 데이터를 한 번에 조회.
- **장점**:
  - 간단한 설정으로 N+1 문제 해결.
  - 단일 쿼리로 데이터 조회 완료.
- **단점**:
  - 불필요한 데이터까지 조회(과도한 데이터 로드).
  - 복잡한 관계에서 쿼리가 비효율적(예: 카르테시안 곱).
  - 메모리 사용량 증가.
- **적합 사례**: 연관 데이터가 항상 필요하고, 데이터량이 적은 경우.

### 3.2 Fetch Join
**설명**: JPQL 또는 Criteria API에서 `FETCH JOIN`을 사용하여 연관 엔티티를 한 번에 조회.
```java
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```
- **쿼리**: `SELECT u.*, o.* FROM user u INNER JOIN order o ON u.id = o.user_id`.
- **장점**:
  - 명시적으로 필요한 연관 데이터만 조회.
  - 단일 쿼리로 N+1 문제 해결.
- **단점**:
  - 복잡한 쿼리 작성 필요.
  - 페이징(`@Pageable`)과 함께 사용할 때 주의 필요(Hibernate에서 결과 중복 가능).
- **예제**:
  ```java
  List<User> users = userRepository.findAllWithOrders();
  for (User user : users) {
      System.out.println(user.getOrders().size()); // 추가 쿼리 없이 동작
  }
  ```

### 3.3 Batch Fetch (Batch Size)
**설명**: Hibernate의 `@BatchSize` 어노테이션을 사용하여 연관 엔티티를 배치 단위로 조회.
```java
@OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
@BatchSize(size = 100)
private List<Order> orders;
```
- **동작**: 100명의 사용자 조회 시, `IN` 절을 사용해 100개의 `orders`를 한 번에 조회.
  - 예: `SELECT * FROM order WHERE user_id IN (?, ?, ..., ?)` (100개 ID).
- **장점**:
  - 쿼리 수를 N에서 N/batch_size로 줄임.
  - 설정이 간단.
- **단점**:
  - 배치 크기 조절 필요(너무 크면 메모리 문제, 너무 작으면 쿼리 수 증가).
  - 복잡한 관계에서는 여전히 다중 쿼리 발생 가능.
- **예제**:
  ```java
  List<User> users = userRepository.findAll();
  for (User user : users) {
      user.getOrders().size(); // 배치 단위로 조회
  }
  ```

### 3.4 Entity Graph
**설명**: JPA의 `@EntityGraph`를 사용하여 연관 엔티티를 동적으로 로드.
```java
@Entity
@NamedEntityGraph(
    name = "userWithOrders",
    attributeNodes = @NamedAttributeNode("orders")
)
public class User { ... }

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph(value = "userWithOrders")
    List<User> findAll();
}
```
- **쿼리**: `JOIN FETCH`와 유사한 단일 쿼리 생성.
- **장점**:
  - 선언적으로 연관 데이터 로드 정의.
  - 재사용 가능한 엔티티 그래프 정의 가능.
- **단점**:
  - 복잡한 관계에서 설정이 번거로움.
  - 페이징과 함께 사용 시 주의 필요.
- **예제**:
  ```java
  List<User> users = userRepository.findAll();
  for (User user : users) {
      System.out.println(user.getOrders().size()); // 추가 쿼리 없음
  }
  ```

### 3.5 DTO Projection
**설명**: JPQL 또는 Spring Data JPA Projection을 사용하여 필요한 데이터만 DTO로 조회.
```java
public interface UserProjection {
    String getName();
    List<OrderProjection> getOrders();

    interface OrderProjection {
        String getItem();
    }
}

@Query("SELECT new com.example.UserDto(u.name, u.orders) FROM User u")
List<UserDto> findAllUsersWithOrders();
```
- **장점**:
  - 필요한 데이터만 조회하여 메모리 사용 최적화.
  - 단일 쿼리로 데이터 로드.
- **단점**:
  - DTO 클래스 작성 필요.
  - 복잡한 매핑 로직 추가 가능.
- **예제**:
  ```java
  public class UserDto {
      private String name;
      private List<Order> orders;
      public UserDto(String name, List<Order> orders) {
          this.name = name;
          this.orders = orders;
      }
  }
  ```

### 3.6 쿼리 최적화 (IN 절 사용)
**설명**: 연관 데이터를 `IN` 절로 수동 조회하여 쿼리 수 감소.
```java
List<User> users = userRepository.findAll();
List<Long> userIds = users.stream().map(User::getId).collect(Collectors.toList());
List<Order> orders = orderRepository.findByUserIdIn(userIds);
```
- **장점**:
  - 명시적 쿼리 제어로 성능 최적화.
  - 대량 데이터 처리에 적합.
- **단점**:
  - 수동 매핑 로직 필요.
  - 코드 복잡성 증가.
- **예제**:
  ```java
  Map<Long, List<Order>> orderMap = orders.stream()
      .collect(Collectors.groupingBy(order -> order.getUser().getId()));
  for (User user : users) {
      List<Order> userOrders = orderMap.getOrDefault(user.getId(), Collections.emptyList());
  }
  ```

---

## 4. 최적화 방안 선택 가이드

| 방법              | 적합 상황                              | 주의점                              |
|-------------------|---------------------------------------|-------------------------------------|
| **FetchType.EAGER** | 소규모 데이터, 항상 연관 데이터 필요   | 과도한 데이터 로드, 메모리 사용 증가 |
| **Fetch Join**      | 특정 쿼리에서 연관 데이터 필요         | 페이징과 함께 사용 시 결과 중복 가능 |
| **Batch Fetch**     | 대량 데이터, 동적 로딩 필요           | 배치 크기 조절 필요                |
| **Entity Graph**    | 재사용 가능한 연관 데이터 조회         | 복잡한 설정 가능                   |
| **DTO Projection**  | 필요한 데이터만 조회, API 응답 최적화 | DTO 작성 및 매핑 로직 필요         |
| **IN 절 사용**      | 대량 데이터, 명시적 쿼리 제어 필요    | 코드 복잡성 증가                   |

### 권장 사항
1. **소규모 데이터**: `Fetch Join` 또는 `Entity Graph` 사용.
2. **대규모 데이터**: `Batch Fetch` 또는 `IN 절`로 쿼리 수 최소화.
3. **API 응답**: `DTO Projection`으로 데이터 전송 최적화.
4. **복잡한 관계**: `Entity Graph`로 재사용성 높임.
5. **항상 주의**: 쿼리 실행 계획 확인(예: `EXPLAIN` 사용).

---

## 5. 추가 팁: 모니터링과 디버깅

- **쿼리 로그 활성화**: `spring.jpa.show-sql=true`와 `hibernate.format_sql=true`로 실행된 쿼리 확인.
- **쿼리 수 확인**: Hibernate의 `Statistics` 또는 외부 모니터링 도구(New Relic, DataDog) 사용.
- **쿼리 최적화 확인**: 데이터베이스의 `EXPLAIN` 또는 `ANALYZE`로 실행 계획 분석.
- **캐싱 적용**: JPA 2차 캐시 또는 Redis를 활용해 반복 쿼리 감소.

---

## 6. 결론

JPA의 N+1 문제는 지연 로딩으로 인해 발생하는 성능 병목 현상으로, 데이터베이스 쿼리 수를 줄이는 것이 핵심입니다. `Fetch Join`, `Entity Graph`, `Batch Fetch`, `DTO Projection`, `IN 절` 등 다양한 최적화 방안을 상황에 맞게 사용하면 성능을 크게 개선할 수 있습니다.

### 요약
- **N+1 문제**: 초기 쿼리(1) + 연관 엔티티 쿼리(N)로 성능 저하.
- **발생 사례**: OneToMany, ManyToOne, 컬렉션 순회, API 응답.
- **최적화 방안**: Eager Fetch, Fetch Join, Batch Fetch, Entity Graph, DTO Projection, IN 절.
- **권장**: 쿼리 로그와 실행 계획 분석으로 최적화 효과 확인.

이 자료를 통해 N+1 문제를 정확히 이해하고, 적절한 최적화 기법을 적용하여 JPA 기반 애플리케이션의 성능을 향상시킬 수 있습니다.