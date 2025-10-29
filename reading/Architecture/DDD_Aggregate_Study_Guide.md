# DDD의 Aggregate 학습 자료

이 자료는 DDD(Domain-Driven Design)에서 **Aggregate**의 개념, 역할, 동작 방식, 그리고 Spring에서의 구현 예시를 설명합니다. Aggregate는 DDD의 핵심 구성 요소로, 도메인 모델의 일관성과 데이터 무결성을 보장하는 데 중요한 역할을 합니다.

---

## 1. Aggregate란?

### 정의
- **Aggregate**는 DDD에서 관련된 객체들을 하나의 단위로 묶은 클러스터입니다. 이 클러스터는 **일관성 경계(consistency boundary)**를 정의하며, 데이터 무결성을 보장하고 복잡한 도메인 모델을 관리하기 쉽게 만듭니다.
- Aggregate는 **루트 엔티티(Aggregate Root)**와 그에 속한 **엔티티(Entity)** 및 **값 객체(Value Object)**들로 구성됩니다.
- **핵심 아이디어**: Aggregate는 외부에서 오직 루트 엔티티를 통해 접근할 수 있으며, 내부 객체들은 루트 엔티티를 통해 관리됩니다. 이를 통해 데이터 변경 시 일관성을 유지합니다.

### 주요 구성 요소
1. **Aggregate Root**:
   - Aggregate의 진입점(entry point) 역할을 하는 엔티티.
   - 외부에서 Aggregate에 접근할 때는 반드시 루트 엔티티를 통해야 함.
   - 루트 엔티티는 Aggregate 내부의 모든 객체를 관리하고 일관성을 책임짐.
2. **엔티티(Entity)**:
   - 고유 식별자(ID)를 가지며, 생명 주기를 관리하는 객체.
   - Aggregate 내부에서 루트 엔티티와 함께 동작.
3. **값 객체(Value Object)**:
   - 식별자가 없고, 불변(immutable)으로 설계되는 객체.
   - 주로 속성(예: 주소, 금액)을 표현.

### 예시
- **도메인**: 전자상거래 시스템
- **Aggregate**: `Order` (주문)
  - **Aggregate Root**: `Order` 엔티티 (주문 ID로 식별)
  - **내부 객체**:
    - `OrderItem` 엔티티 (주문 항목, ID로 식별)
    - `Address` 값 객체 (배송지 주소)
  - 외부는 `Order`를 통해 주문 항목이나 배송지를 수정하며, 직접 `OrderItem`이나 `Address`에 접근 불가.

### 질문으로 확인
- Aggregate를 처음 들어봤다면, "객체들을 하나의 단위로 묶는다"는 개념이 어떤 상황에서 유용할 것 같나요?

---

## 2. Aggregate의 동작 방식

### 동작 원리
- **일관성 경계**:
  - Aggregate는 하나의 트랜잭션 단위로 동작합니다. 즉, Aggregate 내부의 모든 변경은 단일 트랜잭션 내에서 일관성을 유지해야 합니다.
  - 예: 주문 상태를 변경할 때, 주문 항목과 배송지가 함께 일관된 상태로 업데이트되어야 함.
- **루트 엔티티의 역할**:
  - 외부에서 Aggregate 내부 객체에 직접 접근하지 못하도록 캡슐화.
  - 루트 엔티티가 내부 객체의 상태를 관리하고, 변경 로직을 중앙화.
  - 예: `Order`의 `addItem()` 메소드를 통해 주문 항목을 추가하면, `Order`가 재고 확인 및 총액 계산을 수행.
- **불변 규칙**:
  - Aggregate 내부의 변경은 루트 엔티티를 통해야 하며, 외부에서 내부 객체를 직접 수정하면 안 됨.
  - 이는 데이터 무결성을 보장하고, 복잡한 도메인 로직을 단순화.

### 동작 흐름
1. 클라이언트가 Aggregate Root(예: `Order`)의 메소드를 호출.
2. 루트 엔티티가 내부 객체(예: `OrderItem`, `Address`)를 관리하며 도메인 규칙 적용.
3. 변경 사항은 단일 트랜잭션 내에서 저장소(Repository)를 통해 데이터베이스에 반영.
4. 외부는 Aggregate Root를 통해 결과 조회.

### 코드 예시 (Spring/JPA)
```java
@Entity
public class Order {
    @Id
    private Long id;
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    @Embedded
    private Address address;

    // Aggregate Root의 메소드
    public void addItem(Product product, int quantity) {
        // 도메인 규칙 적용
        if (quantity <= 0) {
            throw new IllegalArgumentException("수량은 0보다 커야 합니다.");
        }
        items.add(new OrderItem(product, quantity));
        calculateTotal();
    }

    public void updateAddress(Address newAddress) {
        this.address = newAddress;
    }

    private void calculateTotal() {
        // 총액 계산 로직
    }
}

@Embeddable
public class Address {
    private String street;
    private String city;
    // 불변 객체로 설계
}

@Entity
public class OrderItem {
    @Id
    private Long id;
    private Product product;
    private int quantity;
}
```
- **설명**:
  - `Order`는 Aggregate Root로, `OrderItem`과 `Address`를 관리.
  - 외부는 `Order.addItem()`이나 `Order.updateAddress()`를 호출해야 하며, 직접 `OrderItem`을 수정 불가.
  - 트랜잭션 내에서 `Order`의 변경이 완료되면 일관성 유지.

### 질문으로 확인
- Aggregate Root가 내부 객체를 관리하는 게 왜 중요한 걸까?

---

## 3. Aggregate가 해결하는 문제들

Aggregate는 DDD에서 복잡한 도메인 모델을 관리하고, 데이터 무결성 및 시스템 성능을 보장하는 데 기여합니다. 아래는 Aggregate가 해결하는 주요 문제들입니다.

### (1) 데이터 일관성 문제
- **문제**: 복잡한 도메인에서 여러 객체가 관련된 변경이 일관되지 않게 처리될 수 있음.
  - 예: 주문 상태를 변경했지만, 주문 항목이 업데이트되지 않아 총액이 잘못 계산됨.
- **해결**:
  - Aggregate는 단일 트랜잭션 내에서 모든 변경을 처리해 일관성을 보장.
  - 루트 엔티티가 도메인 규칙(예: 수량은 0보다 커야 함)을 강제.
- **실제 사례**:
  - 전자상거래 시스템에서 주문 취소 시, `Order`가 주문 항목과 결제 상태를 함께 업데이트해 데이터 불일치 방지.

### (2) 복잡한 도메인 로직 관리
- **문제**: 객체 간 관계가 복잡하면, 어디서 어떤 로직을 처리해야 할지 혼란스러움.
  - 예: 주문 항목 추가 시 재고 확인, 총액 계산, 배송비 계산이 여러 곳에서 분산 처리.
- **해결**:
  - Aggregate Root가 모든 로직을 중앙화해 관리.
  - 예: `Order.addItem()`이 재고 확인, 항목 추가, 총액 계산을 한 번에 처리.
- **실제 사례**:
  - 주문 처리 시스템에서 `Order`가 모든 변경 로직을 캡슐화해 코드 가독성과 유지보수성 향상.

### (3) 성능 문제 (트랜잭션 범위 축소)
- **문제**: 큰 트랜잭션 범위는 데이터베이스 락을 오래 유지해 성능 저하.
  - 예: 전체 주문 데이터를 한 번에 로드하면 메모리 사용량 증가.
- **해결**:
  - Aggregate는 명확한 경계를 정의해 트랜잭션 범위를 최소화.
  - 예: `Order`만 로드하고, 필요 시 `OrderItem`을 지연 로딩(lazy loading).
- **실제 사례**:
  - 대규모 주문 시스템에서 Aggregate 단위로 데이터를 로드해 메모리 사용량과 락 시간을 줄임.

### (4) 외부 접근으로 인한 무결성 위반
- **문제**: 외부에서 Aggregate 내부 객체를 직접 수정하면 도메인 규칙이 무시될 수 있음.
  - 예: `OrderItem`의 수량을 직접 변경하면 재고 확인 로직이 실행되지 않음.
- **해결**:
  - Aggregate는 루트 엔티티를 통한 접근만 허용해 캡슐화를 보장.
  - 예: `Order.addItem()`을 통해 수량 변경 시 재고 확인 로직 강제.
- **실제 사례**:
  - 결제 시스템에서 `Payment` Aggregate가 결제 상태 변경을 관리해, 외부에서 직접 결제 상태를 수정하지 못하도록 방지.

### (5) 분산 트랜잭션 문제
- **문제**: 여러 Aggregate를 하나의 트랜잭션에서 수정하면 분산 트랜잭션이 되어 복잡성 증가.
- **해결**:
  - Aggregate는 단일 트랜잭션 단위로 설계되어, 각 Aggregate를 독립적으로 처리.
  - 예: `Order`와 `Inventory`는 별도 Aggregate로, 각각의 트랜잭션에서 처리.
- **실제 사례**:
  - 주문과 재고 업데이트를 이벤트 기반 비동기 처리로 분리해 분산 트랜잭션 방지.

### 질문으로 확인
- Aggregate가 데이터 일관성을 보장하는 데 어떻게 기여한다고 생각하나요?

---

## 4. Spring에서의 Aggregate 구현

Spring과 JPA를 사용해 Aggregate를 구현할 때는, Aggregate Root와 내부 객체 간의 관계를 명확히 정의하고, 트랜잭션 및 리포지토리를 통해 관리합니다.

### 구현 예시
```java
@Entity
public class Order {
    @Id
    private Long id;
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, mappedBy = "order")
    private List<OrderItem> items = new ArrayList<>();
    @Embedded
    private Address address;

    // 생성자
    protected Order() {}

    public Order(Long id, Address address) {
        this.id = id;
        this.address = address;
    }

    // 도메인 로직
    public void addItem(Product product, int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("수량은 0보다 커야 합니다.");
        }
        items.add(new OrderItem(this, product, quantity));
        calculateTotal();
    }

    public void updateAddress(Address newAddress) {
        this.address = newAddress;
    }

    private void calculateTotal() {
        // 총액 계산 로직
    }

    // 조회 메소드 (캡슐화 유지)
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
}

@Entity
public class OrderItem {
    @Id
    @GeneratedValue
    private Long id;
    @ManyToOne(fetch = FetchType.LAZY)
    private Order order;
    private Product product;
    private int quantity;

    protected OrderItem() {}

    public OrderItem(Order order, Product product, int quantity) {
        this.order = order;
        this.product = product;
        this.quantity = quantity;
    }
}

@Embeddable
public class Address {
    private String street;
    private String city;

    protected Address() {}

    public Address(String street, String city) {
        this.street = street;
        this.city = city;
    }
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {}
```

### 서비스 계층
```java
@Service
@Transactional
public class OrderService {
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public void createOrder(Long id, Address address, Product product, int quantity) {
        Order order = new Order(id, address);
        order.addItem(product, quantity);
        orderRepository.save(order);
    }
}
```
- **설명**:
  - `Order`는 Aggregate Root로, `OrderItem`과 `Address`를 관리.
  - 외부는 `OrderService`를 통해 `Order`에 접근하며, 직접 `OrderItem` 수정 불가.
  - `@Transactional`을 사용해 Aggregate 단위로 트랜잭션 관리.

### LazyConnectionDataSourceProxy와의 연계
- `LazyConnectionDataSourceProxy`는 Aggregate의 트랜잭션 처리에서 연결 풀 효율성을 높이는 데 유용합니다.
- 예: `OrderService.createOrder()`에서 외부 API 호출 후 DB 저장 시, `LazyConnectionDataSourceProxy`가 쿼리 실행 직전에만 연결을 획득해 풀 고갈 방지.

### 질문으로 확인
- Spring에서 Aggregate를 구현할 때 `@Transactional`을 어디에 붙이는 게 적절할까?

---

## 5. Aggregate와 LazyConnectionDataSourceProxy의 시너지

- **문제**: Aggregate를 처리하는 트랜잭션에서 외부 작업(예: API 호출)이 포함되면, 연결 풀이 불필요하게 점유됨.
- **해결**: `LazyConnectionDataSourceProxy`를 사용하면, Aggregate의 트랜잭션 시작 시 연결을 지연시켜 풀 효율성을 높임.
- **예시**:
  ```java
  @Service
  @Transactional
  public class OrderService {
      private final OrderRepository orderRepository;
      private final ExternalApiClient apiClient;

      public OrderService(OrderRepository orderRepository, ExternalApiClient apiClient) {
          this.orderRepository = orderRepository;
          this.apiClient = apiClient;
      }

      public void createOrder(Long id, Address address, Product product, int quantity) {
          // 외부 API 호출 (500ms 소요)
          apiClient.validateProduct(product);
          // Aggregate 생성 및 저장
          Order order = new Order(id, address);
          order.addItem(product, quantity);
          orderRepository.save(order);
      }
  }
  ```
  - `LazyConnectionDataSourceProxy`는 `orderRepository.save()` 시점에만 연결을 획득해, API 호출 동안 연결 풀을 점유하지 않음.

---

## 6. 장점과 한계

### Aggregate의 장점
- **일관성 보장**: 단일 트랜잭션 내에서 데이터 무결성 유지.
- **캡슐화**: 루트 엔티티를 통해 도메인 로직 중앙화.
- **성능 최적화**: 트랜잭션 범위 축소로 락 시간 감소.
- **유지보수성**: 복잡한 도메인 로직을 체계적으로 관리.

### Aggregate의 한계
- **설계 복잡성**: Aggregate 경계를 잘못 정의하면 지나치게 크거나 작은 Aggregate가 만들어짐.
- **성능 고려**: 큰 Aggregate는 메모리 사용량 증가 가능.

### LazyConnectionDataSourceProxy의 장점
- **연결 풀 효율성**: 불필요한 연결 점유 감소.
- **응답 시간 단축**: 외부 작업 동안 연결 미점유.
- **Replication 호환**: 읽기/쓰기 분리 환경에서 라우팅 정확도 향상.

### LazyConnectionDataSourceProxy의 한계
- **설정 복잡성**: Spring Boot 자동 설정 비활성화 필요.
- **네이티브 연결 제한**: `ConnectionProxy`로 인해 네이티브 연결 캐스팅 불가.

---

## 7. 질문으로 이해도 확인
1. Aggregate와 Aggregate Root의 차이는 무엇이라고 생각하나요?
2. Aggregate가 데이터 일관성을 보장하는 데 어떻게 기여한다고 생각하나요?
3. `LazyConnectionDataSourceProxy`를 Aggregate와 함께 사용할 때 어떤 이점이 있을까?
4. 당신의 프로젝트에서 Aggregate를 어떻게 설계할 것 같나요?

---

## 8. 추가 학습 제안
- **DDD 문서**: Eric Evans의 *Domain-Driven Design* 책에서 Aggregate 관련 챕터 읽기.
- **Spring 문서**: `LazyConnectionDataSourceProxy` 공식 문서와 JPA 문서 읽기.
- **실습**: Spring Boot 프로젝트에서 `Order` Aggregate를 구현하고, `LazyConnectionDataSourceProxy`를 설정해 연결 풀 사용량 비교.
- **심화**: MySQL Replication 환경에서 `AbstractRoutingDataSource`와 `LazyConnectionDataSourceProxy`를 함께 사용해보기.