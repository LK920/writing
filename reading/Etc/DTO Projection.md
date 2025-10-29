
이 문서는 JPA에서 **DTO Projection**에 대한 상세한 학습 자료입니다. DTO Projection은 JPA에서 N+1 문제를 해결하고, 엔티티 간 관계 설정을 피하는 복잡한 시스템에서 효율적으로 데이터를 조회하는 방법입니다. 실무적 맥락(순환 참조와 관리 복잡성 우려)을 반영하여, 초보자도 이해할 수 있도록 개념, 사용 방법, 코드 예제, 실무적 고려 사항을 단계적으로 정리했습니다.

## 1. DTO Projection이란?

**DTO Projection**은 JPA에서 JPQL(또는 Criteria API)을 사용해 엔티티 대신 **DTO(Data Transfer Object)** 클래스로 데이터를 직접 조회하는 방법입니다. 엔티티 전체를 로딩하지 않고, 필요한 데이터만 선택적으로 조회하여 성능을 최적화하고, N+1 문제를 해결합니다. 특히, 엔티티 간 관계 설정(`@OneToMany`, `@ManyToOne`) 없이 외래 키 기반으로 데이터를 조회할 때 유용합니다.

### 주요 특징
- **선택적 데이터 조회**: 필요한 필드만 조회해 메모리 사용량 감소.
- **N+1 문제 해결**: 단일 쿼리로 연관 데이터를 가져와 추가 쿼리 방지.
- **관계 설정 불필요**: 엔티티 간 `@OneToMany` 같은 매핑 없이 외래 키 기반 조인 가능.
- **실무적 장점**: 복잡한 시스템에서 엔티티 관계 설정으로 인한 순환 참조나 관리 복잡성을 피할 수 있음.

### 언제 사용하나?
- 엔티티 간 관계 설정을 최소화하려는 경우.
- 조회 전용 쿼리에서 특정 데이터만 필요한 경우.
- N+1 문제를 해결하면서 JPA의 이식성을 유지하고 싶을 때.

**질문**: 당신의 프로젝트에서 DTO Projection을 사용하려는 구체적인 상황이 있나요? 예: "User와 Order 데이터를 조회하는데, 관계 설정 없이 특정 필드만 필요해." 구체적인 사례를 알려주시면 예제를 맞춤화할게요!

## 2. DTO Projection의 동작 원리

DTO Projection은 JPQL에서 `new` 키워드를 사용해 DTO 클래스의 생성자를 호출하여 데이터를 매핑합니다. 엔티티 간 관계 설정 없이 외래 키를 기반으로 조인 쿼리를 작성할 수 있습니다.

### 기본 예시
- **엔티티** (관계 설정 없음):
  ```java
  @Entity
  public class User {
      @Id
      private Long id;
      private String name;
      // No @OneToMany for Orders
  }

  @Entity
  public class Order {
      @Id
      private Long id;
      private Long userId; // 외래 키
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
      // Getters
  }
  ```
- **JPQL 쿼리**:
  ```java
  @Query("SELECT new com.example.dto.UserOrderDto(u.id, u.name, o.id, o.amount) " +
         "FROM User u LEFT JOIN Order o ON u.id = o.userId")
  List<UserOrderDto> findUserOrderDtos();
  ```
- **생성된 SQL**:
  ```sql
  SELECT u.id, u.name, o.id, o.amount
  FROM User u
  LEFT JOIN Order o ON u.id = o.user_id
  ```
- **결과**: 단일 쿼리로 `User`와 `Order` 데이터를 DTO로 매핑, N+1 문제 해결.

### 동작 원리
1. JPQL에서 `new` 키워드로 DTO 생성자를 호출.
2. `ON` 절로 외래 키 기반 조인 정의(관계 설정 불필요).
3. 필요한 필드만 선택해 DTO로 매핑.
4. 단일 쿼리로 실행되어 추가 쿼리(N+1) 방지.

**질문**: 위 예시에서 어떤 부분이 명확하고, 어떤 부분이 더 궁금한가요? 예: "JPQL에서 `ON` 절을 처음 봤어, 더 자세히 설명해줘."

## 3. DTO Projection의 장점과 단점

### 장점
- **N+1 문제 해결**: 단일 쿼리로 연관 데이터 조회.
- **관계 설정 불필요**: 순환 참조나 관리 복잡성 우려 없이 외래 키 기반 조회 가능.
- **성능 최적화**: 필요한 필드만 선택해 메모리 사용량 감소.
- **이식성**: JPQL은 데이터베이스 독립적이어서 Native Query보다 이식성 좋음.
- **Spring Data JPA와 호환**: `@Query`로 간단히 구현 가능.

### 단점
- **DTO 클래스 정의**: 각 쿼리에 맞는 DTO 클래스를 별도로 작성해야 함.
- **관리 비용**: 복잡한 시스템에서 DTO가 많아지면 관리 부담 증가.
- **복잡한 쿼리 제한**: 매우 복잡한 조인이나 동적 쿼리는 작성 어려움.

**질문**: 프로젝트에서 DTO를 이미 사용 중인가요? DTO 추가가 부담스러운지, 아니면 익숙한지 알려주시면 실무적 조언을 드릴게요!

## 4. 실무적 사용 사례

### 사례 1: User와 Order 조회
- **상황**: `User`와 `Order` 데이터를 조회하는데, 관계 설정 없이 N+1 문제 해결 필요.
- **해결**:
  ```java
  @Repository
  public interface UserRepository extends JpaRepository<User, Long> {
      @Query("SELECT new com.example.dto.UserOrderDto(u.id, u.name, o.id, o.amount) " +
             "FROM User u LEFT JOIN Order o ON u.id = o.userId")
      List<UserOrderDto> findUserOrderDtos();
  }
  ```
- **결과**: 단일 쿼리로 데이터 조회, N+1 문제 해결.

### 사례 2: 다중 테이블 조인
- **상황**: `User`, `Order`, `OrderItem`을 조회.
- **DTO**:
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
  ```
- **JPQL**:
  ```java
  @Query("SELECT new com.example.dto.UserOrderItemDto(u.id, u.name, o.id, oi.id) " +
         "FROM User u LEFT JOIN Order o ON u.id = o.userId " +
         "LEFT JOIN OrderItem oi ON o.id = oi.orderId")
  List<UserOrderItemDto> findUserOrderItemDtos();
  ```

### 실무 팁
- **DTO 재사용**: 비슷한 쿼리에서 DTO를 재사용해 관리 부담 줄이기.
- **필드 최소화**: DTO에 필요한 필드만 포함해 성능 최적화.
- **페이징 처리**:
  ```java
  @Query("SELECT new com.example.dto.UserOrderDto(u.id, u.name, o.id, o.amount) " +
         "FROM User u LEFT JOIN Order o ON u.id = o.userId")
  Page<UserOrderDto> findUserOrderDtos(Pageable pageable);
  ```
- **복잡한 DTO 관리**: 패키지 구조를 활용해 DTO를 체계적으로 관리(예: `com.example.dto.user`, `com.example.dto.order`).

**질문**: 위 사례 중 프로젝트에 적용하고 싶은 사례가 있나요? 예: "페이징 처리된 DTO Projection이 필요해."

## 5. N+1 문제와 DTO Projection
DTO Projection은 N+1 문제를 해결하는 데 특히 유용합니다:
- **문제**: Lazy Loading으로 `User`를 조회한 뒤 `Order`를 개별적으로 조회하면 N+1 쿼리 발생.
- **해결**: DTO Projection으로 단일 쿼리로 모든 데이터 조회.
- **예시**: 위의 `UserOrderDto` 쿼리는 단일 쿼리로 `User`와 `Order` 데이터를 가져와 N+1 문제 방지.

## 6. 실무적 고려: 순환 참조와 관리 복잡성
- **순환 참조 방지**: DTO Projection은 엔티티 간 관계 설정 없이 외래 키 기반으로 조회하므로, 순환 참조(예: `User` ↔ `Order`) 문제 없음.
- **관리 복잡성 감소**: 엔티티 매핑 대신 DTO로 데이터를 명확히 정의, 도메인 모델 단순화.
- **실무 예**: 마이크로서비스 환경에서 도메인 경계를 명확히 하기 위해 DTO Projection 사용.

## 7. 학습 활동
1. **실습**: `User`와 `Order`를 조회하는 DTO Projection을 작성해보세요. 예:
   - DTO 클래스 정의.
   - JPQL 쿼리 작성.
   - Spring Data JPA Repository에 추가.
2. **분석**: 프로젝트에서 N+1 문제가 발생하는 쿼리를 하나 적고, DTO Projection으로 해결할 수 있을지 검토해보세요.
3. **질문**: DTO Projection을 적용할 때 가장 궁금하거나 어려운 부분은 무엇인가요? 예: "DTO가 많아지면 어떻게 관리하지?"

## 8. 추가 자료
- **Spring Data JPA 문서**: `@Query`와 DTO Projection 사용법.
- **Hibernate 문서**: JPQL과 DTO Projection의 성능 최적화.
- **실무 사례**: 복잡한 시스템에서 DTO Projection으로 조회 최적화 사례 검색.

**질문**: DTO Projection에 대해 더 궁금한 점이나 구체적인 코드 예제가 필요한 부분이 있나요?

---