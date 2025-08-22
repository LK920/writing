# Lombok 최소화 및 IntelliJ 자동 생성 가이드

이 문서는 Lombok 어노테이션(`@Getter`, `@Setter`, `@ToString`, `@Builder`, `@NoArgsConstructor` 등)의 사용을 최소화하고, IntelliJ IDEA의 자동 생성 기능을 활용해 보일러플레이트 코드(getter, setter, toString, 빌더 패턴 등)를 생성하는 접근법을 상세히 설명한다. 사용자가 언급한 코치의 피드백(즉, Lombok을 최소화하고 IntelliJ의 자동 생성을 권장하며, 특히 `@Builder`를 직접 구현하는 방향)이 기술적 근거인지 개인적 선호인지 평가한다. 이커머스 플랫폼의 DTO와 엔티티 설계 맥락에서 예시, 비유, 설계 원칙, 개선 가이드를 제공하며, 빌드/테스트 및 코틀린 전환 시의 영향을 분석한다.

## 목차

- [1. 배경: Lombok과 IntelliJ 자동 생성](#1-배경-lombok과-intellij-자동-생성)
- [2. Lombok 어노테이션과 IntelliJ 자동 생성의 비교](#2-lombok-어노테이션과-intellij-자동-생성의-비교)
- [3. 비유로 이해하기](#3-비유로-이해하기)
- [4. Lombok 최소화의 장단점](#4-lombok-최소화의-장단점)
- [5. 빌드 및 테스트 시 영향](#5-빌드-및-테스트-시-영향)
- [6. 코틀린 전환 시 고려사항](#6-코틀린-전환-시-고려사항)
- [7. 이커머스 플랫폼에서의 예시](#7-이커머스-플랫폼에서의-예시)
- [8. Lombok 사용 검토 및 개선](#8-lombok-사용-검토-및-개선)
- [9. 코치의 선호: 기술적 근거 vs 개인 취향](#9-코치의-선호-기술적-근거-vs-개인-취향)
- [10. 참고 자료](#10-참고-자료)

## 1. 배경: Lombok과 IntelliJ 자동 생성

**Lombok**:
- Java의 보일러플레이트 코드를 줄이기 위한 라이브러리로, 어노테이션(`@Getter`, `@Setter`, `@ToString`, `@Builder`, `@NoArgsConstructor` 등)을 통해 getter, setter, toString, 빌더 패턴, 생성자 등을 자동 생성.
- 컴파일 타임에 코드를 생성하여 런타임 성능에는 영향 없음.
- DTO, 엔티티, VO에서 자주 사용.

**IntelliJ 자동 생성**:
- IntelliJ IDEA는 getter, setter, toString, 생성자, 빌더 패턴 등을 자동 생성하는 기능을 제공(단축키: `Alt+Insert` 또는 `Code > Generate`).
- 예: 클래스에서 `Alt+Insert`로 getter/setter, toString, 생성자를 즉시 생성 가능.
- 빌더 패턴도 IntelliJ의 플러그인(예: InnerBuilder) 또는 수동 템플릿으로 생성 가능.

**사용자의 질문**:
- 코치가 Lombok 어노테이션을 최소화하고 IntelliJ의 자동 생성을 권장하며, 특히 `@Builder`를 직접 구현하라고 한 이유는 무엇인가?
- 이는 빌드/테스트 및 코틀린 전환 시의 이점 때문인가, 아니면 코치의 개인적 취향인가?

**답변 요약**:
- Lombok 최소화와 IntelliJ 자동 생성은 가독성, 디버깅 용이성, 의존성 감소, 코틀린 호환성을 높이는 기술적 근거가 있다. 특히 `@Builder`를 직접 구현하면 코드의 의도를 명확히 하고, 커스터마이징이 쉬워진다.
- 그러나 IntelliJ 자동 생성은 반복 작업이 많아질 수 있고, 팀의 일관성 유지가 필요하다. 코치의 선호는 기술적 근거(가독성, 유지보수성)와 개인적 취향(명시적 코드 선호)의 혼합일 가능성이 높다.
- 코틀린 전환 시 `data class`가 Lombok의 기능을 대체하므로, Lombok 최소화는 전환 준비에 유리하다.

## 2. Lombok 어노테이션과 IntelliJ 자동 생성의 비교

### 2.1. 주요 Lombok 어노테이션
| 어노테이션              | 역할                                                                 | 사용 사례                     |
|-------------------------|----------------------------------------------------------------------|-------------------------------|
| `@Getter`               | 모든 필드에 getter 메서드 생성                                        | DTO, 읽기 전용 객체           |
| `@Setter`               | 모든 필드에 setter 메서드 생성                                        | DTO, 상태 변경 필요 시        |
| `@ToString`             | `toString()` 메서드 생성                                              | 디버깅, 로깅                  |
| `@NoArgsConstructor`    | 매개변수 없는 기본 생성자 생성, `access = AccessLevel.PROTECTED`로 제한 가능 | JPA 엔티티                   |
| `@Builder`              | 빌더 패턴 구현, 객체 생성 간소화                                     | 복잡한 객체 초기화(DTO, 엔티티) |

**예시** (Lombok 사용):
```java
@Builder
@Getter
@Setter
@ToString
public class CreateOrderRequest {
    private Long userId;
    private List<Long> productIds;
}
```

### 2.2. IntelliJ 자동 생성
- **기능**:
  - Getter/Setter: 모든 필드 또는 선택한 필드에 대해 생성.
  - toString: 필드 선택 가능, 커스텀 형식 지원.
  - 생성자: 기본 생성자, 모든 필드 생성자 등.
  - 빌더 패턴: InnerBuilder 플러그인 또는 수동 템플릿으로 생성.
- **예시** (IntelliJ로 생성):
  ```java
  public class CreateOrderRequest {
      private Long userId;
      private List<Long> productIds;

      // IntelliJ로 생성한 getter/setter
      public Long getUserId() { return userId; }
      public void setUserId(Long userId) { this.userId = userId; }
      public List<Long> getProductIds() { return productIds; }
      public void setProductIds(List<Long> productIds) { this.productIds = productIds; }

      // IntelliJ로 생성한 toString
      @Override
      public String toString() {
          return "CreateOrderRequest{" +
                 "userId=" + userId +
                 ", productIds=" + productIds +
                 '}';
      }

      // IntelliJ로 생성한 빌더 (InnerBuilder 플러그인 사용)
      public static class Builder {
          private Long userId;
          private List<Long> productIds;

          public Builder userId(Long userId) {
              this.userId = userId;
              return this;
          }

          public Builder productIds(List<Long> productIds) {
              this.productIds = productIds;
              return this;
          }

          public CreateOrderRequest build() {
              return new CreateOrderRequest(this);
          }
      }

      private CreateOrderRequest(Builder builder) {
          this.userId = builder.userId;
          this.productIds = builder.productIds;
      }
  }
  ```

### 2.3. 비교
| 항목                | Lombok                                              | IntelliJ 자동 생성                                   |
|---------------------|----------------------------------------------------|-------------------------------------------------|
| **코드 간소화**     | 어노테이션으로 소스 코드 최소화                     | 명시적 코드 생성, 소스 코드 길어짐               |
| **가독성**          | 소스 코드에 메서드 안 보임, 초보자에게 혼란 가능    | 메서드가 명시적으로 보여 가독성 높음             |
| **유지보수**        | 어노테이션 변경 쉬움, 하지만 의존성 관리 필요       | 코드 수정 쉬움, 의존성 없음                     |
| **커스터마이징**    | `@Builder` 등 기본 설정 제한적                     | 빌더 패턴, toString 등 커스텀 가능              |
| **의존성**          | Lombok 라이브러리 및 빌드 설정 필요                | 의존성 없음, IntelliJ 기본 기능 사용            |
| **코틀린 호환성**   | 코틀린에서 불필요, `data class`로 대체 가능         | 코틀린 전환 시 수동 코드 리팩토링 필요          |
| **디버깅**          | 생성된 메서드 추적 어려움                          | 명시적 코드로 디버깅 용이                       |

## 3. 비유로 이해하기

Lombok과 IntelliJ 자동 생성을 **요리 도구**로 비유하면:
- **상황**: 요리사(개발자)가 재료(필드)로 요리(클래스)를 만든다. 요리 과정에서 재료를 자르고 섞는 도구(getter, setter, toString, 빌더 등) 필요.
- **Lombok**:
  - 전자동 주방 기기: 버튼(`@Getter`, `@Builder`)만 누르면 재료가 자동으로 준비됨.
  - 장점: 빠르고 간편.
  - 단점: 기계 내부 동작(생성된 코드)이 안 보여 디버깅 어려움. 기계(Lombok 라이브러리) 없으면 동작 안 함.
- **IntelliJ 자동 생성**:
  - 수동 칼과 믹서: 요리사가 직접 도구를 선택해 재료를 준비.
  - 장점: 어떤 도구로 어떻게 자르는지 명확히 보임. 커스텀 가능(예: 특정 재료만 자름).
  - 단점: 수동 작업이 많아 시간이 더 걸림.
- **코틀린**:
  - 스마트 주방: 재료 준비 도구(getter, setter)가 기본 내장(`data class`). 별도 기계(Lombok)나 수동 도구(IntelliJ) 없이 간단히 요리 가능.
- **비유의 교훈**:
  - Lombok은 빠르지만 내부 동작이 안 보여 혼란 가능.
  - IntelliJ 자동 생성은 명시적이지만 작업량 증가.
  - 코틀린은 Lombok의 기능을 내장해 의존성을 줄임.

## 4. Lombok 최소화의 장단점

### 4.1. 장점
- **가독성 향상**:
  - IntelliJ로 생성한 코드는 메서드가 소스 코드에 명시되어 초보자나 비Lombok 사용자도 이해 쉬움.
  - 예: `getUserId()`가 코드에 직접 보임.
- **의존성 감소**:
  - Lombok 라이브러리와 어노테이션 프로세서 제거로 빌드 설정 단순화.
  - 예: `build.gradle`에서 Lombok 의존성 제거.
- **디버깅 용이**:
  - 생성된 메서드가 소스 코드에 있어 디버깅 시 오류 추적 쉬움.
- **커스터마이징**:
  - IntelliJ로 특정 필드만 getter/setter 생성하거나, `@Builder` 대신 커스텀 빌더 구현 가능.
- **코틀린 호환성**:
  - Lombok 없이 코드 작성 시 코틀린 전환 시 리팩토링 부담 감소.

### 4.2. 단점
- **보일러플레이트 증가**:
  - IntelliJ로 수동 생성 시 코드 길어짐.
  - 예: 10개 필드의 getter/setter는 수십 줄 추가.
- **작업량 증가**:
  - 필드 추가/변경 시 매번 IntelliJ로 메서드 재생성 필요.
- **팀 일관성 문제**:
  - 팀원마다 생성 방식(예: 빌더 패턴 구현)이 달라질 수 있음.
- **JPA 제약**:
  - 엔티티에 기본 생성자가 필요(JPA 요구사항)하므로, 수동으로 `protected` 생성자 작성해야 함.

### 4.3. `@Builder` 직접 구현의 장점
- **명시성**: IntelliJ로 생성한 빌더는 코드에 명시되어 동작 이해 쉬움.
- **커스터마이징**: 특정 필드만 빌더로 초기화하거나, 추가 검증 로직 추가 가능.
- **의존성 제거**: `@Builder`의 Lombok 의존성 제거.
- **예시** (IntelliJ 빌더):
  ```java
  public class CreateOrderRequest {
      private Long userId;
      private List<Long> productIds;

      public static class Builder {
          private Long userId;
          private List<Long> productIds;

          public Builder userId(Long userId) {
              if (userId <= 0) throw new IllegalArgumentException("Invalid userId");
              this.userId = userId;
              return this;
          }
          // ... (다른 필드)
          public CreateOrderRequest build() {
              return new CreateOrderRequest(this);
          }
      }
  }
  ```

## 5. 빌드 및 테스트 시 영향

### 5.1. 빌드 시 영향
- **Lombok**:
  - **장점**: 컴파일 타임에 코드 생성, 런타임 성능 영향 없음.
  - **단점**: Lombok 설정 오류(예: IDE 플러그인 누락, 어노테이션 프로세서 미설정)로 빌드 실패 가능.
  - 예: `@Builder`와 `@NoArgsConstructor` 충돌 시 컴파일 오류.
- **IntelliJ 자동 생성**:
  - **장점**: 의존성 없어 빌드 설정 간단. 코드가 명시적이므로 빌드 오류 추적 쉬움.
  - **단점**: 수동 생성으로 코드량 증가, 빌드 후 클래스 파일 크기 약간 증가(미미).
- **결론**: Lombok 최소화는 빌드 설정 단순화와 오류 감소에 기여. IntelliJ 자동 생성은 설정 오류 위험이 없지만, 코드 관리 부담 증가.

### 5.2. 테스트 시 영향
- **Lombok**:
  - **장점**: `@Getter`/`@Setter`로 DTO 테스트 간단.
    - 예: `request.setUserId(1L)`로 빠르게 테스트 객체 설정.
  - **단점**: `@ToString`으로 민감 데이터 노출 가능. `@Builder`는 디버깅 시 생성된 코드 추적 어려움.
- **IntelliJ 자동 생성**:
  - **장점**: 명시적 메서드로 디버깅 용이. 커스텀 `toString`으로 민감 데이터 제외 가능.
  - **단점**: 수동 생성한 getter/setter로 테스트 코드 작성 시 약간의 추가 작업 필요.
- **결론**: IntelliJ 자동 생성은 테스트 코드의 명시성과 디버깅 용이성을 높이지만, 초기 설정 시간이 늘어날 수 있다. `@Builder` 직접 구현은 테스트에서 객체 생성의 유연성을 높인다.

## 6. 코틀린 전환 시 고려사항

### 6.1. 코틀린의 데이터 클래스
- 코틀린의 `data class`는 getter, setter, `toString`, `equals`, `hashCode`를 기본 제공.
- **예시**:
  ```kotlin
  data class CreateOrderRequest(
      @field:NotNull val userId: Long,
      @field:NotEmpty val productIds: List<Long>
  )
  ```
  - 자동으로 getter, setter, `toString` 생성. 빌더 패턴은 별도 구현 가능.
- **Lombok 대비 장점**:
  - 의존성 제거: Lombok 라이브러리 불필요.
  - 가독성: 메서드가 암시적이지 않고 코틀린 문법으로 명확.
  - 불변성: `val`로 기본 불변 필드 제공.
- **단점**: 기존 Java 프로젝트를 코틀린으로 전환 시 리팩토링 비용 발생.

### 6.2. JPA와 코틀린
- **문제**: JPA는 기본 생성자를 요구, 코틀린 `data class`는 기본 생성자 제공 안 함.
- **해결**:
  - `noArg` 플러그인 사용:
    ```kotlin
    // build.gradle.kts
    plugins {
        id("org.jetbrains.kotlin.plugin.noarg") version "2.0.20"
    }
    noArg {
        annotation("javax.persistence.Entity")
    }
    ```
  - 수동 기본 생성자:
    ```kotlin
    @Entity
    class Order private constructor() {
        var id: Long? = null
        var userId: Long = 0
        var products: List<Product> = emptyList()
    }
    ```

### 6.3. 빌더 패턴과 코틀린
- 코틀린은 `data class`로 빌더 패턴을 기본 제공하지 않지만, 명시적 빌더 구현 가능.
- **예시**:
  ```kotlin
  data class CreateOrderRequest private constructor(
      val userId: Long,
      val productIds: List<Long>
  ) {
      companion object {
          class Builder {
              private var userId: Long? = null
              private var productIds: List<Long> = emptyList()

              fun userId(userId: Long) = apply { this.userId = userId }
              fun productIds(productIds: List<Long>) = apply { this.productIds = productIds }

              fun build(): CreateOrderRequest {
                  return CreateOrderRequest(
                      userId ?: throw IllegalStateException("userId is required"),
                      productIds
                  )
              }
          }
      }
  }
  ```
- **장점**: IntelliJ로 생성한 Java 빌더와 유사한 명시성, Lombok 의존성 제거.

### 6.4. 사용자의 의견 평가
- **의견**: Lombok 어노테이션을 최소화하고 IntelliJ 자동 생성 사용.
  - **평가**: 타당. IntelliJ 자동 생성은 의존성을 줄이고 코드 명시성을 높인다. 코틀린 전환 시 `data class`가 Lombok의 기능을 대체하므로, Lombok 최소화는 전환 준비에 유리.
- **의견**: `@Builder`를 직접 구현.
  - **평가**: 타당. 직접 구현한 빌더는 커스터마이징과 디버깅이 쉬우며, 코틀린에서도 유사한 방식으로 구현 가능.

## 7. 이커머스 플랫폼에서의 예시

이커머스 플랫폼의 주문 생성 시나리오를 통해 Lombok 최소화와 IntelliJ 자동 생성을 설명한다.

### 7.1. 시나리오: 주문 생성
- **요구사항**: 클라이언트가 주문 요청을 보내고, 서버는 주문을 생성하여 응답.
- **구조**: 클라이언트 → 컨트롤러 → 유스케이스 → 레포지토리 어댑터.

### 7.2. Lombok 사용 설계
```java
@Builder
@Getter
@Setter
@ToString
public class CreateOrderRequest {
    @NotNull
    private Long userId;
    @NotEmpty
    private List<Long> productIds;
}

@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Order {
    private Long id;
    private Long userId;
    private List<Product> products;
    private OrderStatus status;

    public Order(Long userId, List<Product> products) {
        this.userId = userId;
        this.products = products;
    }
}
```

### 7.3. IntelliJ 자동 생성 설계
```java
public class CreateOrderRequest {
    private Long userId;
    private List<Long> productIds;

    public CreateOrderRequest(Long userId, List<Long> productIds) {
        this.userId = userId;
        this.productIds = productIds;
    }

    // IntelliJ로 생성한 getter/setter
    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }
    public List<Long> getProductIds() { return productIds; }
    public void setProductIds(List<Long> productIds) { this.productIds = productIds; }

    // IntelliJ로 생성한 toString
    @Override
    public String toString() {
        return "CreateOrderRequest{userId=" + userId + ", productIds=" + productIds + '}';
    }

    // IntelliJ로 생성한 빌더
    public static class Builder {
        private Long userId;
        private List<Long> productIds;

        public Builder userId(Long userId) {
            this.userId = userId;
            return this;
        }

        public Builder productIds(List<Long> productIds) {
            this.productIds = productIds;
            return this;
        }

        public CreateOrderRequest build() {
            return new CreateOrderRequest(this.userId, this.productIds);
        }
    }
}

@Entity
public class Order {
    private Long id;
    private Long userId;
    private List<Product> products;
    private OrderStatus status;

    protected Order() {} // JPA 요구사항

    public Order(Long userId, List<Product> products) {
        this.userId = userId;
        this.products = products;
    }

    // IntelliJ로 생성한 getter
    public Long getId() { return id; }
    public Long getUserId() { return userId; }
    public List<Product> getProducts() { return products; }
    public OrderStatus getStatus() { return status; }

    // 비즈니스 메서드
    public void create() { this.status = OrderStatus.CREATED; }
}
```

### 7.4. 코틀린 대안
```kotlin
data class CreateOrderRequest(
    @field:NotNull val userId: Long,
    @field:NotEmpty val productIds: List<Long>
) {
    // 수동 빌더
    companion object {
        class Builder {
            private var userId: Long? = null
            private var productIds: List<Long> = emptyList()

            fun userId(userId: Long) = apply { this.userId = userId }
            fun productIds(productIds: List<Long>) = apply { this.productIds = productIds }

            fun build(): CreateOrderRequest {
                return CreateOrderRequest(
                    userId ?: throw IllegalStateException("userId is required"),
                    productIds
                )
            }
        }
    }
}

@Entity
class Order private constructor() {
    var id: Long? = null
    var userId: Long = 0
    var products: List<Product> = emptyList()
    var status: OrderStatus = OrderStatus.CREATED

    constructor(userId: Long, products: List<Product>) : this() {
        this.userId = userId
        this.products = products
    }

    fun create() {
        status = OrderStatus.CREATED
    }
}
```

### 7.5. 분석
- **Lombok**:
  - **장점**: 코드 간소화, 빠른 구현.
  - **단점**: `@ToString`으로 민감 데이터 노출 가능, `@Builder` 디버깅 어려움.
- **IntelliJ 자동 생성**:
  - **장점**: 코드 명시성, 디버깅 용이, 커스텀 빌더 구현 가능.
  - **단점**: 코드량 증가, 수동 관리 필요.
- **코틀린**:
  - **장점**: Lombok 없이 간소화, 불변성 지원.
  - **단점**: JPA 엔티티 구현 시 추가 작업 필요.
- **사용자의 코치 피드백 적용**:
  - Lombok 최소화는 IntelliJ 자동 생성으로 대체 가능, 특히 `@Builder` 직접 구현은 유연성과 명시성을 높임.
  - 코틀린 전환 시 Lombok 제거가 더 자연스럽다.

## 8. Lombok 사용 검토 및 개선

### 8.1. 검토 기준
1. **불필요한 어노테이션**:
   - DTO/엔티티에 불필요한 `@NoArgsConstructor`, `@ToString` 사용 여부.
   - **해결**: IntelliJ로 필요한 메서드만 생성.
2. **캡슐화 약화**:
   - `@Setter`로 엔티티의 상태가 임의로 변경되는지 확인.
   - **해결**: 비즈니스 메서드 사용.
3. **가독성**:
   - Lombok으로 메서드가 소스 코드에 안 보여 혼란 여부.
   - **해결**: IntelliJ로 명시적 코드 생성.
4. **빌더 패턴**:
   - `@Builder`로 커스터마이징 제한 여부.
   - **해결**: IntelliJ로 커스텀 빌더 구현.
5. **코틀린 준비**:
   - Lombok 의존성이 코틀린 전환에 방해 여부.
   - **해결**: `data class`로 전환.

### 8.2. 개선 예시
**문제**: Lombok 과도 사용.
```java
@Builder
@Getter
@Setter
@ToString
@NoArgsConstructor
@AllArgsConstructor
public class CreateOrderRequest {
    private Long userId;
    private List<Long> productIds;
}
```

**개선** (IntelliJ 자동 생성):
```java
public class CreateOrderRequest {
    private Long userId;
    private List<Long> productIds;

    public CreateOrderRequest(Long userId, List<Long> productIds) {
        this.userId = userId;
        this.productIds = productIds;
    }

    public Long getUserId() { return userId; }
    public List<Long> getProductIds() { return productIds; }

    @Override
    public String toString() {
        return "CreateOrderRequest{userId=" + userId + ", productIds=...}";
    }

    public static class Builder {
        private Long userId;
        private List<Long> productIds;

        public Builder userId(Long userId) {
            this.userId = userId;
            return this;
        }

        public Builder productIds(List<Long> productIds) {
            this.productIds = productIds;
            return this;
        }

        public CreateOrderRequest build() {
            return new CreateOrderRequest(userId, productIds);
        }
    }
}
```

**개선 결과**:
- `@Setter` 제거로 불변성 강화.
- `@ToString` 커스터마이징으로 민감 데이터 제외.
- `@Builder` 대신 IntelliJ로 커스텀 빌더 구현.
- 코틀린 전환 시 `data class`로 쉽게 변환 가능.

### 8.3. 개선 프로세스
1. **코드 리뷰**: Lombok 어노테이션 점검, 불필요한 `@NoArgsConstructor`, `@ToString` 식별.
2. **리팩토링**:
   - DTO: IntelliJ로 getter, toString, 빌더 생성.
   - 엔티티: `@NoArgsConstructor` 유지(JPA용), `@Setter` 대신 비즈니스 메서드.
3. **테스트**: 생성된 메서드 및 빌더 테스트.
4. **코틀린 전환 준비**: `data class`로 DTO/엔티티 변환 계획.

## 9. 코치 피드백 근거

- **가독성**: IntelliJ 자동 생성은 코드에 메서드가 명시되어 초보자 및 비Lombok 사용자에게 이해 쉬움.
- **의존성 감소**: Lombok 제거로 빌드 설정 단순화, 의존성 관리 부담 감소.
- **디버깅 용이성**: 명시적 코드로 오류 추적 쉬움.
- **코틀린 호환성**: Lombok 없이 작성된 코드는 코틀린 `data class`로 전환 용이.
- **커스터마이징**: `@Builder` 직접 구현으로 검증 로직 추가 가능.
- **팀 협업**: Lombok에 익숙하지 않은 팀원이 코드 이해 쉬움.
- **교육적 목적 가능성**: 초보자에게 코드 동작을 명확히 이해시키기 위해 Lombok 최소화 권장.


## 10. 참고 자료

- **도서**:
  - *Java Persistence with Hibernate* (Christian Bauer): JPA와 생성자.
  - *Kotlin in Action* (Dmitry Jemerov): 코틀린 데이터 클래스.
- **온라인 자료**:
  - [Lombok Documentation](https://projectlombok.org): 어노테이션 사용법.
  - [Baeldung: Lombok Pros and Cons](https://www.baeldung.com/lombok): Lombok 장단점.
  - [IntelliJ IDEA Code Generation](https://www.jetbrains.com/help/idea/generating-code.html): IntelliJ 자동 생성.
  - [Kotlin JPA](https://kotlinlang.org/docs/reference/data-classes.html): 코틀린과 JPA.
- **코드 예시**:
  - [Spring Boot with Lombok](https://github.com/spring-projects/spring-boot): Lombok 사용 예.
  - [Kotlin Spring Boot](https://github.com/JetBrains/kotlin): 코틀린 데이터 클래스 예.
- **팀 자료**:
  - `api/dto/` 디렉토리: 기존 DTO 코드.
  - 팀 Wiki: 프로젝트 아키텍처 문서.

---

## 결론

Lombok 어노테이션(`@Getter`, `@Setter`, `@ToString`, `@Builder`)을 최소화하고 IntelliJ IDEA의 자동 생성 기능을 사용하는 것은 가독성, 디버깅 용이성, 의존성 감소, 코틀린 전환 준비에 기술적 이점이 있다. 특히 `@Builder`를 직접 구현하면 커스터마이징과 명시성이 강화된다. 그러나 IntelliJ 자동 생성은 코드량 증가와 수동 관리 부담이 단점이다. 코틀린의 `data class`는 Lombok의 기능을 대체하므로, 전환 시 Lombok 제거가 유리하다. 코치의 선호는 기술적 근거와 개인 취향의 혼합으로 보이며, 팀의 협업 환경과 프로젝트 목표에 따라 Lombok 최소화 전략을 조정해야 한다. 이커머스 플랫폼에서는 DTO에 IntelliJ 자동 생성 또는 Java Record를, 엔티티에는 최소한의 Lombok(`@NoArgsConstructor`)을 사용하는 것을 권장한다.