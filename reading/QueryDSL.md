# QueryDSL 학습 가이드

이 문서는 **QueryDSL**에 대한 상세한 학습 자료입니다. QueryDSL은 JPA 기반의 동적 쿼리 작성 도구로, N+1 문제를 해결하고 엔티티 간 관계 설정을 피하는 복잡한 시스템에서 유연하게 데이터를 조회하는 데 유용합니다. 순환 참조와 관리 복잡성 우려를 반영하여, 초보자도 이해할 수 있도록 개념, 설정, 코드 예제, 실무적 고려 사항을 단계적으로 정리했습니다.

## 1. QueryDSL이란?

**QueryDSL**은 JPA, SQL, MongoDB 등 다양한 백엔드에서 동적이고 타입 안전한 쿼리를 작성할 수 있는 오픈소스 프레임워크입니다. JPA 환경에서 N+1 문제를 해결하고, 엔티티 간 관계 설정 없이 외래 키 기반으로 데이터를 조회할 때 특히 유용합니다.

### 주요 특징
- **타입 안전성**: Q-클래스를 사용해 컴파일 타임에 쿼리 오류 감지.
- **동적 쿼리**: 조건에 따라 쿼리를 동적으로 생성.
- **관계 설정 불필요**: 외래 키 기반 조인으로 엔티티 관계 설정 없이 조회 가능.
- **실무적 장점**: 복잡한 시스템에서 순환 참조와 관리 복잡성을 피하면서 유연한 쿼리 작성 가능.

### 언제 사용하나?
- 복잡한 동적 쿼리가 필요한 경우.
- 엔티티 간 관계 설정을 피하고 외래 키 기반으로 조회하려는 경우.
- N+1 문제를 해결하면서 타입 안전한 쿼리를 작성하고 싶을 때.

**질문**: QueryDSL을 프로젝트에 도입하려는 구체적인 이유가 있나요? 예: "동적 쿼리가 필요해서" 또는 "N+1 문제를 타입 안전하게 해결하고 싶어." 알려주시면 예제를 맞춤화할게요!

## 2. QueryDSL 설정

QueryDSL을 사용하려면 프로젝트에 의존성을 추가하고 Q-클래스를 생성해야 합니다.

### 의존성 추가 (Maven)
```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
```

### 플러그인 설정 (Q-클래스 생성)
```xml
<plugin>
    <groupId>com.mysema.maven</groupId>
    <artifactId>apt-maven-plugin</artifactId>
    <version>1.1.3</version>
    <executions>
        <execution>
            <goals>
                <goal>process</goal>
            </goals>
            <configuration>
                <outputDirectory>target/generated-sources/java</outputDirectory>
                <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Spring Boot 설정
- `JPAQueryFactory` 빈 설정:
  ```java
  @Configuration
  public class QueryDSLConfig {
      @PersistenceContext
      private EntityManager entityManager;

      @Bean
      public JPAQueryFactory jpaQueryFactory() {
          return new JPAQueryFactory(entityManager);
      }
  }
  ```

**질문**: QueryDSL 설정을 프로젝트에 적용해본 적 있나요? 설정 과정에서 궁금하거나 어려운 부분이 있으면 알려주세요!

## 3. QueryDSL의 동작 원리

QueryDSL은 Q-클래스(예: `QUser`, `QOrder`)를 사용해 타입 안전한 쿼리를 작성합니다. 엔티티 간 관계 설정 없이 외래 키 기반으로 조인 쿼리를 작성할 수 있습니다.

### 기본 예시
- **엔티티** (관계 설정 없음):
  ```java
  @Entity
  public class User {
      @Id
      private Long id;
      private String name;
  }

  @Entity
  public class Order {
      @Id
      private Long id;
      private Long userId;
      private BigDecimal amount;
  }
  ```
- **DTO 클래스**:
  ```java
  public class UserOrderDto {
      private Long userId;
      private String userName;
      private Long orderId;
      private BigDecimal orderAmount;

      public UserOrderDto(Long userId, String userName, Long orderId, BigDecimal orderAmount) {
          this.userId = userId;
          this.userName = userName;
          this.orderId = orderId;
          this.orderAmount = orderAmount;
      }
  }
  ```
- **QueryDSL 쿼리**:
  ```java
  @Service
  public class UserService {
      private final JPAQueryFactory queryFactory;

      public UserService(JPAQueryFactory queryFactory) {
          this.queryFactory = queryFactory;
      }

      public List<UserOrderDto> findUserOrderDtos() {
          QUser user = QUser.user;
          QOrder order = QOrder.order;

          return queryFactory
              .select(Projections.constructor(UserOrderDto.class, user.id, user.name, order.id, order.amount))
              .from(user)
              .leftJoin(order).on(user.id.eq(order.userId))
              .fetch();
      }
  }
  ```
- **생성된 SQL**:
  ```sql
  SELECT u.id, u.name, o.id, o.amount
  FROM User u
  LEFT JOIN Order o ON u.id = o.user_id
  ```
- **결과**: 단일 쿼리로 `User`와 `Order` 데이터를 DTO로 매핑, N+1 문제 해결.

### 동작 원리
1. Q-클래스(예: `QUser`, `QOrder`)를 사용해 엔티티 필드에 타입 안전하게 접근.
2. `Projections.constructor`로 DTO 생성.
3. `on` 절로 외래 키 기반 조인 정의.
4. 단일 쿼리로 실행되어 N+1 문제 방지.

**질문**: QueryDSL 쿼리 작성에서 어떤 부분이 궁금한가요? 예: "Q-클래스 생성이 이해 안 돼" 또는 "`on` 절 사용법이 더 필요해."

## 4. QueryDSL의 장점과 단점

### 장점
- **타입 안전성**: 컴파일 타임에 쿼리 오류 감지.
- **동적 쿼리**: 조건에 따라 쿼리 동적 생성 가능.
  ```java
  public List<UserOrderDto> findUserOrderDtos(String nameFilter) {
      QUser user = QUser.user;
      QOrder order = QOrder.order;

      return queryFactory
          .select(Projections.constructor(UserOrderDto.class, user.id, user.name, order.id, order.amount))
          .from(user)
          .leftJoin(order).on(user.id.eq(order.userId))
          .where(nameFilter != null ? user.name.containsIgnoreCase(nameFilter) : null)
          .fetch();
  }
  ```
- **관계 설정 불필요**: 외래 키 기반 조인으로 순환 참조와 관리 복잡성 회피.
- **JPA와 혼용**: Spring Data JPA와 함께 사용 가능.

### 단점
- **의존성 추가**: QueryDSL 라이브러리와 설정 필요.
- **학습 비용**: Q-클래스와 쿼리 문법 학습 필요.
- **빌드 설정**: Q-클래스 생성을 위한 빌드 플러그인 설정.

**질문**: QueryDSL 도입 시 가장 걱정되는 부분은 무엇인가요? 예: "설정 과정이 복잡할까 봐" 또는 "팀원들이 익숙하지 않을 것 같아."

## 5. 실무적 사용 사례

### 사례 1: 동적 조회
- **상황**: 사용자 이름 필터링과 함께 `User`와 `Order` 조회.
- **QueryDSL**:
  ```java
  public List<UserOrderDto> findUserOrderDtos(String nameFilter, BigDecimal minAmount) {
      QUser user = QUser.user;
      QOrder order = QOrder.order;

      return queryFactory
          .select(Projections.constructor(UserOrderDto.class, user.id, user.name, order.id, order.amount))
          .from(user)
          .leftJoin(order).on(user.id.eq(order.userId))
          .where(
              nameFilter != null ? user.name.containsIgnoreCase(nameFilter) : null,
              minAmount != null ? order.amount.goe(minAmount) : null
          )
          .fetch();
  }
  ```

### 사례 2: 다중 테이블 조인
- **상황**: `User`, `Order`, `OrderItem` 조회.
- **QueryDSL**:
  ```java
  public class UserOrderItemDto {
      private Long userId;
      private String userName;
      private Long orderId;
      private Long itemId;

      public UserOrderItemDto(Long userId, String userName, Long orderId, Long itemId) {
          this.userId = userId;
          this.userName = userName;
          this.orderId = orderId;
          this.itemId = itemId;
      }
  }

  public List<UserOrderItemDto> findUserOrderItemDtos() {
      QUser user = QUser.user;
      QOrder order = QOrder.order;
      QOrderItem orderItem = QOrderItem.orderItem;

      return queryFactory
          .select(Projections.constructor(UserOrderItemDto.class, user.id, user.name, order.id, orderItem.id))
          .from(user)
          .leftJoin(order).on(user.id.eq(order.userId))
          .leftJoin(orderItem).on(order.id.eq(orderItem.orderId))
          .fetch();
  }
  ```

### 실무 팁
- **Q-클래스 관리**: Q-클래스를 `target/generated-sources`에 생성해 소스 관리에서 제외.
- **동적 조건 처리**: `BooleanBuilder`로 복잡한 동적 조건 처리:
  ```java
  BooleanBuilder builder = new BooleanBuilder();
  if (nameFilter != null) {
      builder.and(user.name.containsIgnoreCase(nameFilter));
  }
  if (minAmount != null) {
      builder.and(order.amount.goe(minAmount));
  }
  queryFactory.select(...).where(builder).fetch();
  ```
- **페이징 처리**:
  ```java
  queryFactory
      .select(Projections.constructor(UserOrderDto.class, user.id, user.name, order.id, order.amount))
      .from(user)
      .leftJoin(order).on(user.id.eq(order.userId))
      .offset(pageable.getOffset())
      .limit(pageable.getPageSize())
      .fetch();
  ```

**질문**: 위 사례 중 프로젝트에 적용하고 싶은 사례가 있나요? 예: "동적 쿼리가 필요해."

## 6. N+1 문제와 QueryDSL
QueryDSL은 N+1 문제를 해결하는 데 효과적입니다:
- **문제**: Lazy Loading으로 `User`를 조회한 뒤 `Order`를 개별적으로 조회하면 N+1 쿼리 발생.
- **해결**: QueryDSL로 단일 쿼리로 모든 데이터 조회.
- **예시**: 위의 `UserOrderDto` 쿼리는 단일 쿼리로 `User`와 `Order` 데이터를 가져와 N+1 문제 방지.

## 7. 실무적 고려: 순환 참조와 관리 복잡성
- **순환 참조 방지**: QueryDSL은 외래 키 기반 조인으로 관계 설정 없이 조회, 순환 참조 문제 없음.
- **관리 복잡성 감소**: 동적 쿼리로 다양한 요구사항 처리, 엔티티 모델 단순화.
- **실무 예**: 마이크로서비스 환경에서 도메인 경계를 명확히 하기 위해 QueryDSL 사용.

## 8. 학습 활동
1. **실습**: QueryDSL로 `User`와 `Order`를 조회하는 쿼리를 작성해보세요. 예:
   - Q-클래스 확인.
   - DTO와 QueryDSL 쿼리 작성.
   - Service에 추가.
2. **분석**: 프로젝트에서 N+1 문제가 발생하는 쿼리를 하나 적고, QueryDSL로 해결할 수 있을지 검토해보세요.
3. **질문**: QueryDSL 적용 시 가장 궁금하거나 어려운 부분은 무엇인가요? 예: "동적 쿼리 작성법이 궁금해."

## 9. 추가 자료
- **QueryDSL 문서**: 공식 사이트에서 JPA 쿼리 작성법.
- **Spring Data JPA와 QueryDSL 통합**: Spring Boot에서 QueryDSL 설정 가이드.
- **실무 사례**: QueryDSL로 동적 쿼리와 N+1 문제 해결 사례 검색.

**질문**: QueryDSL에 대해 더 궁금한 점이나 구체적인 코드 예제가 필요한 부분이 있나요?

---