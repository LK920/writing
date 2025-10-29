# Spring Triangle (DIP, AOP, PSA) 학습 자료

이 자료는 Spring 프레임워크의 **Spring Triangle**을 **DIP (Dependency Inversion Principle)**, **AOP (Aspect-Oriented Programming)**, **PSA (Portable Service Abstraction)** 중심으로 설명합니다. Spring Triangle은 Spring의 설계 철학을 나타내며, 유연하고 확장 가능한 애플리케이션 개발을 가능하게 합니다. 이전 대화에서 다룬 AOP, 프록시, 셀프 인보케이션, `LazyConnectionDataSourceProxy`, DDD Aggregate와의 연계를 포함해, 각 개념의 정의, 동작 방식, 사용 방법, 그리고 문제 해결 사례를 체계적으로 정리합니다.

---

## 1. Spring Triangle이란?

**Spring Triangle**은 Spring 프레임워크의 핵심 설계 원칙과 기능을 대표하는 세 가지 요소로, 애플리케이션의 유연성, 모듈성, 이식성을 보장합니다. 여기서는 **DIP**, **AOP**, **PSA**로 구성된 Spring Triangle을 다룹니다:

1. **DIP (Dependency Inversion Principle, 의존성 역전 원칙)**: 모듈 간 의존성을 낮추고, 추상화에 의존하도록 설계하는 원칙.
2. **AOP (Aspect-Oriented Programming, 관점 지향 프로그래밍)**: 공통 관심사를 비즈니스 로직과 분리해 모듈화하는 프로그래밍 패러다임.
3. **PSA (Portable Service Abstraction, 이식 가능한 서비스 추상화)**: Spring이 제공하는 추상화 계층으로, 특정 기술(예: JDBC, JPA)에 의존하지 않고 이식 가능한 코드를 작성할 수 있게 함.

이 세 요소는 Spring의 핵심 철학인 **IoC (Inversion of Control, 제어의 역전)**을 구현하며, 서로 긴밀히 연계되어 동작합니다.

### 질문으로 확인
- Spring Triangle의 세 요소 중 어떤 게 가장 익숙하거나 낯설게 느껴지나요?

---

## 2. DIP (Dependency Inversion Principle)

### 정의
- **DIP**는 SOLID 원칙 중 하나로, "고수준 모듈은 저수준 모듈에 의존해서는 안 되며, 둘 다 추상화에 의존해야 한다"는 설계 원칙입니다.
- **핵심 아이디어**: 구체적인 구현(저수준 모듈)에 의존하지 않고, 인터페이스나 추상 클래스(추상화)에 의존해 결합도를 낮춤.
- Spring에서는 **DI (Dependency Injection, 의존성 주입)**을 통해 DIP를 구현합니다. Spring 컨테이너가 객체의 의존성을 주입해, 코드가 구체적인 클래스에 의존하지 않도록 합니다.

### 동작 방식
- Spring 컨테이너(IoC 컨테이너)는 Bean의 생성과 의존성 주입을 관리.
- `@Autowired`, `@Inject`, 또는 XML 설정을 통해 의존성을 주입.
- 예: 서비스 클래스가 특정 구현체가 아닌 인터페이스에 의존하도록 설계.

### 코드 예시
```java
public interface ProductRepository {
    Product findById(Long id);
}

@Repository
public class JpaProductRepository implements ProductRepository {
    @Override
    public Product findById(Long id) {
        // JPA로 구현
        return new Product(id, "상품");
    }
}

@Service
public class ProductService {
    private final ProductRepository repository;

    @Autowired
    public ProductService(ProductRepository repository) {
        this.repository = repository; // 추상화에 의존
    }

    public Product getProduct(Long id) {
        return repository.findById(id);
    }
}
```
- **설명**:
  - `ProductService`는 구체적인 `JpaProductRepository`가 아닌 `ProductRepository` 인터페이스에 의존.
  - Spring 컨테이너가 `JpaProductRepository`를 주입해 DIP 준수.
  - 다른 구현체(예: `MyBatisProductRepository`)로 교체 가능.

### 해결하는 문제
- **높은 결합도**: 구체적인 구현에 의존하면 코드 변경이 어려움. DIP는 인터페이스에 의존해 유연성 확보.
- **테스트 어려움**: 구체 클래스 의존 시 단위 테스트에서 Mocking이 어려움. 인터페이스 의존으로 테스트 용이.
- **확장성 부족**: 새로운 구현체로 교체가 어려움. DIP로 구현체 교체 용이.

### 질문으로 확인
- DIP를 적용하지 않고 구체 클래스에 의존하면 어떤 문제가 생길까?

---

## 3. AOP (Aspect-Oriented Programming)

### 정의
- **AOP**는 공통 관심사(예: 로깅, 트랜잭션 관리, 보안)를 비즈니스 로직과 분리해 모듈화하는 프로그래밍 패러다임입니다.
- Spring에서는 `@Transactional`, `@Cacheable`, `@Async` 같은 어노테이션이 AOP를 통해 구현됩니다.
- **핵심 아이디어**: 공통 로직을 Aspect로 정의해, 비즈니스 로직에 직접 코드를 삽입하지 않고 동적으로 적용.

### 동작 방식
- Spring AOP는 **프록시 패턴**을 사용해 구현됩니다.
  - 프록시는 타겟 객체를 감싸서 호출을 가로채고, AOP 로직(예: 트랜잭션 시작/커밋)을 실행.
  - **JDK 동적 프록시**: 인터페이스 기반.
  - **CGLIB 프록시**: 클래스 기반.
- **셀프 인보케이션 문제** (이전 대화 참조):
  - 클래스 내부에서 `this.method()`를 호출하면 프록시를 우회해 AOP 로직이 적용되지 않음.
  - 예: `@Transactional`이 붙은 메소드가 셀프 인보케이션으로 호출되면 트랜잭션 미적용.

### 코드 예시
```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(Long id) {
        System.out.println("주문 생성");
        this.updateStock(id); // 셀프 인보케이션
    }

    @Transactional
    public void updateStock(Long id) {
        System.out.println("재고 업데이트");
    }
}
```
- **문제**: `updateStock()`이 프록시를 거치지 않아 `@Transactional`이 적용되지 않음.
- **해결**:
  - 별도 서비스로 분리.
  - 자기 자신 주입(`@Autowired OrderService self`).
  - AspectJ 사용.

### 해결하는 문제
- **코드 중복**: 트랜잭션, 로깅 등 공통 로직이 여러 곳에 흩어짐. AOP로 중앙화.
- **유지보수 어려움**: 공통 로직 변경 시 모든 코드를 수정해야 함. AOP로 변경 용이.
- **셀프 인보케이션 문제**: 프록시 기반 AOP의 한계로, 내부 호출 시 AOP 로직 미적용 (이전 대화 참조).

### 질문으로 확인
- AOP가 없으면 트랜잭션 관리 코드를 어디에 넣어야 할까?

---

## 4. PSA (Portable Service Abstraction)

### 정의
- **PSA**는 Spring이 제공하는 이식 가능한 서비스 추상화 계층으로, 특정 기술(예: JDBC, JPA, JMS)에 직접 의존하지 않고 추상화된 인터페이스를 사용해 코드를 작성할 수 있게 합니다.
- **핵심 아이디어**: 구현 세부 사항을 숨기고, 표준화된 인터페이스를 통해 기술 교체가 용이하도록 함.
- 예: `JdbcTemplate`, `PlatformTransactionManager`, `CacheManager`.

### 동작 방식
- Spring은 특정 기술에 의존하지 않는 추상화된 API를 제공.
- 예: `PlatformTransactionManager`는 JDBC, JPA, Hibernate 등 다양한 구현체를 지원.
- 개발자는 추상화된 인터페이스에만 의존하며, Spring이 적절한 구현체를 주입.

### 코드 예시
```java
@Service
public class ProductService {
    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public ProductService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Product getProduct(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM product WHERE id = ?", 
            new Object[]{id}, 
            (rs, rowNum) -> new Product(rs.getLong("id"), rs.getString("name"))
        );
    }
}
```
- **설명**:
  - `JdbcTemplate`은 JDBC의 복잡한 세부 사항(예: `Connection` 관리)을 추상화.
  - MySQL에서 PostgreSQL로 변경해도 `JdbcTemplate` 코드는 변경 불필요.

### 해결하는 문제
- **기술 종속성**: 특정 DB나 메시징 시스템에 의존하면 기술 교체가 어려움. PSA로 추상화.
- **복잡한 설정**: JDBC, JMS 등의 저수준 API는 설정과 에러 처리가 복잡. PSA로 단순화.
- **이식성 부족**: 다른 기술 스택으로 마이그레이션 시 코드 수정 필요. PSA로 코드 재사용 가능.

### 질문으로 확인
- `JdbcTemplate` 없이 직접 JDBC를 사용하면 어떤 어려움이 있을까?

---

## 5. Spring Triangle과 이전 개념의 연계

### (1) AOP와 셀프 인보케이션
- **문제**: AOP는 프록시로 구현되며, 셀프 인보케이션(`this.method()`) 시 프록시를 우회해 `@Transactional`, `@Cacheable` 등이 동작하지 않음.
- **해결**:
  - 별도 서비스로 분리:
    ```java
    @Service
    public class OrderService {
        @Autowired
        private StockService stockService;

        @Transactional
        public void createOrder(Long id) {
            stockService.updateStock(id);
        }
    }
    ```
  - AspectJ 사용으로 프록시 의존 제거.
- **Spring Triangle과의 연계**: AOP는 Spring Triangle의 핵심 요소로, 공통 관심사를 처리해 DIP(추상화)와 PSA(이식성)를 강화.

### (2) LazyConnectionDataSourceProxy와의 연계
- **문제**: 트랜잭션 시작 시 연결을 즉시 획득하면 연결 풀이 고갈되거나 응답 시간이 길어짐.
- **해결**: `LazyConnectionDataSourceProxy`는 연결을 쿼리 실행 시점까지 지연시켜 연결 풀 효율성을 높임.
- **Spring Triangle과의 연계**:
  - **DIP**: `LazyConnectionDataSourceProxy`는 `DataSource` 인터페이스를 구현해 추상화에 의존.
  - **AOP**: `@Transactional` 트랜잭션 관리와 결합해 연결 지연 효과 극대화.
  - **PSA**: `DataSourceTransactionManager`와 함께 사용해 이식 가능한 트랜잭션 관리 제공.
- **예시**:
  ```java
  @Configuration
  public class DataSourceConfig {
      @Bean
      public DataSource dataSource() {
          return new LazyConnectionDataSourceProxy(new HikariDataSource());
      }

      @Bean
      public PlatformTransactionManager transactionManager(DataSource dataSource) {
          return new DataSourceTransactionManager(dataSource);
      }
  }
  ```

### (3) DDD Aggregate와의 연계
- **문제**: 복잡한 도메인에서 데이터 일관성과 캡슐화가 필요.
- **해결**: Aggregate는 루트 엔티티를 통해 일관성 경계를 정의.
- **Spring Triangle과의 연계**:
  - **DIP**: Aggregate는 인터페이스(예: Repository)에 의존해 구현체 교체 가능.
  - **AOP**: `@Transactional`을 사용해 Aggregate 단위로 트랜잭션 관리.
  - **PSA**: `JpaRepository`와 같은 PSA를 사용해 DB 접근 추상화.
- **예시**:
  ```java
  @Entity
  public class Order {
      @Id
      private Long id;
      @OneToMany(cascade = CascadeType.ALL)
      private List<OrderItem> items;

      public void addItem(Product product, int quantity) {
          items.add(new OrderItem(product, quantity));
      }
  }

  @Service
  @Transactional
  public class OrderService {
      private final OrderRepository repository;

      @Autowired
      public OrderService(OrderRepository repository) {
          this.repository = repository;
      }

      public void createOrder(Long id, Product product, int quantity) {
          Order order = new Order(id);
          order.addItem(product, quantity);
          repository.save(order);
      }
  }
  ```

---

## 6. Spring Triangle이 해결하는 문제들

### (1) 높은 결합도와 낮은 유연성
- **문제**: 구체 클래스에 의존하면 코드 변경이 어려움.
- **해결**: DIP를 통해 인터페이스에 의존, DI로 구현체 주입.
- **사례**: MySQL에서 PostgreSQL로 전환 시 `JdbcTemplate` 코드 변경 없이 전환 가능.

### (2) 코드 중복과 유지보수 어려움
- **문제**: 트랜잭션, 로깅 등 공통 로직이 여러 곳에 흩어짐.
- **해결**: AOP로 공통 관심사를 모듈화.
- **사례**: `@Transactional`로 모든 서비스에 트랜잭션 관리 적용.

### (3) 기술 종속성과 이식성 부족
- **문제**: 특정 기술(JDBC, Hibernate)에 의존하면 마이그레이션 어려움.
- **해결**: PSA로 추상화된 인터페이스 제공.
- **사례**: `PlatformTransactionManager`로 JPA, JDBC, Hibernate 모두 지원.

### (4) 성능 문제 (연결 풀 고갈 등)
- **문제**: 트랜잭션에서 연결을 즉시 획득하면 풀 고갈.
- **해결**: `LazyConnectionDataSourceProxy`와 AOP 결합으로 연결 지연.
- **사례**: 고트래픽 환경에서 연결 풀 효율성 향상.

### (5) 복잡한 도메인 관리
- **문제**: 복잡한 도메인에서 데이터 일관성 유지 어려움.
- **해결**: DDD Aggregate와 AOP(@Transactional), PSA Paranthetical expression
System: `@Transactional`)를 결합해 Aggregate 단위로 일관성 보장.

---

## 7. 장점과 한계

### 장점
- **DIP**:
  - 결합도 감소로 유연한 코드 설계.
  - 테스트 용이성과 확장성 향상.
- **AOP**:
  - 공통 로직 중앙화로 코드 간결화.
  - 유지보수성 향상.
- **PSA**:
  - 기술 독립성으로 코드 이식성 증가.
  - 설정과 에러 처리 단순화.

### 한계
- **DIP**: 인터페이스 설계와 DI 설정에 추가 노력 필요.
- **AOP**: 프록시 기반의 셀프 인보케이션 문제, 성능 오버헤드.
- **PSA**: 추상화로 인해 일부 기술별 세부 설정 제약.

---

## 8. 질문으로 이해도 확인
1. DIP를 적용한 코드와 적용하지 않은 코드의 차이점은 무엇일까?
2. AOP로 트랜잭션 관리를 처리하는 대신, 서비스 코드에 직접 트랜잭션 로직을 넣으면 어떤 단점이 있을까?
3. PSA를 사용하지 않고 특정 기술(예: MySQL JDBC)에 직접 의존하면 어떤 문제가 생길까?
4. `LazyConnectionDataSourceProxy`와 Spring Triangle의 어떤 요소가 가장 밀접하게 연계된다고 생각하나요?

---

## 9. 추가 학습 제안
- **Spring 문서**: Spring 공식 문서에서 IoC, AOP, JDBC/Transaction 섹션 읽기.
- **실습**: Spring Boot 프로젝트에서 `LazyConnectionDataSourceProxy`와 `@Transactional`을 사용해 트랜잭션 지연 효과 확인.
- **심화**: DDD Aggregate를 Spring 프로젝트에 구현하고, AOP와 PSA를 적용해보기.
- **탐구**: Spring Triangle의 개념이 다른 프레임워크(예: Java EE)와 어떻게 다른지 비교.