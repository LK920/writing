# JPA 어노테이션 학습 자료: 사용 빈도 및 중요도 순 정리

이 문서는 JPA(Java Persistence API)에서 사용되는 주요 어노테이션의 역할과 사용법을 사용 빈도와 중요도를 기준으로 정리한 학습 자료입니다. 인메모리 환경에서는 JPA 어노테이션이 동작하지 않지만, 코드 유지보수와 향후 JPA 사용 가능성을 위해 각 어노테이션의 역할과 사용법을 이해하는 것이 중요합니다. 아래는 사용 빈도와 중요도를 기준으로 분류된 어노테이션 목록입니다.

---

## 1. 필수적이고 자주 사용되는 어노테이션 (★★★★★)

이 어노테이션들은 JPA 엔티티를 정의하고 데이터베이스와 매핑하는 데 핵심적인 역할을 합니다. 거의 모든 JPA 프로젝트에서 사용됩니다.

### `@Entity`
- **역할**: 클래스를 데이터베이스 테이블에 매핑되는 엔티티로 지정.
- **중요도**: 필수. JPA가 이 클래스를 엔티티로 인식하도록 함.
- **사용법**:
  - 클래스 선언 위에 추가.
  - `@Entity(name = "customEntityName")`으로 테이블 이름을 명시적으로 지정 가능 (기본값은 클래스 이름).
- **예제**:
  ```java
  import jakarta.persistence.Entity;
  
  @Entity
  public class Product {
      // 필드 및 메서드
  }
  ```
- **주의사항**:
  - 기본 생성자가 반드시 있어야 함.
  - `final` 클래스, `enum`, `interface`에는 사용 불가.

### `@Id`
- **역할**: 엔티티의 기본 키(Primary Key)를 지정.
- **중요도**: 필수. 엔티티는 반드시 하나의 기본 키를 가져야 함.
- **사용법**:
  - 기본 키 필드에 추가.
  - `@GeneratedValue`와 함께 사용하여 자동 생성 전략 설정 가능.
- **예제**:
  ```java
  import jakarta.persistence.Id;
  import jakarta.persistence.GeneratedValue;
  import jakarta.persistence.GenerationType;
  
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  ```
- **주의사항**:
  - 기본 키는 `null`이 될 수 없으며, 고유해야 함.

### `@GeneratedValue`
- **역할**: 기본 키 값의 생성 전략을 지정.
- **중요도**: 매우 높음. 기본 키 자동 생성이 필요한 경우 필수.
- **사용법**:
  - `strategy` 속성으로 생성 전략 지정.
    - `GenerationType.AUTO`: JPA 구현체가 적절한 전략 선택 (기본값).
    - `GenerationType.IDENTITY`: 데이터베이스 자동 증가 (예: MySQL의 `AUTO_INCREMENT`).
    - `GenerationType.SEQUENCE`: 데이터베이스 시퀀스 사용 (예: Oracle).
    - `GenerationType.TABLE`: 별도 테이블로 키 생성 (드물게 사용).
- **예제**:
  ```java
  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  ```
- **주의사항**:
  - `IDENTITY`는 데이터베이스에 따라 동작이 달라질 수 있음.

### `@Column`
- **역할**: 필드를 데이터베이스 테이블의 특정 컬럼에 매핑.
- **중요도**: 높음. 필드와 컬럼 간 세부 매핑이 필요할 때 사용.
- **사용법**:
  - `name`: 컬럼 이름 지정 (기본값은 필드 이름).
  - `nullable`: `false`로 설정 시 `NOT NULL` 제약 조건 추가.
  - `length`: 문자열 길이 지정.
  - `unique`: 유니크 제약 조건 추가.
- **예제**:
  ```java
  @Column(name = "product_name", nullable = false, length = 100)
  private String name;
  ```
- **주의사항**:
  - 생략 시 필드 이름이 컬럼 이름으로 사용됨.

---

## 2. 관계 매핑에 자주 사용되는 어노테이션 (★★★★☆)

엔티티 간 관계(1:1, 1:N, N:M)를 정의할 때 필수적인 어노테이션들입니다. 관계형 데이터베이스 작업에서 매우 자주 사용됩니다.

### `@ManyToOne`
- **역할**: 다대일(N:1) 관계를 정의. 여러 엔티티가 하나의 엔티티를 참조.
- **중요도**: 높음. 다대일 관계는 JPA에서 가장 흔한 관계.
- **사용법**:
  - `fetch`: `FetchType.LAZY` (지연 로딩, 기본값) 또는 `FetchType.EAGER` (즉시 로딩).
  - `optional`: `false`로 설정 시 `NOT NULL` 제약 조건 추가.
  - `@JoinColumn`과 함께 사용하여 외래 키 컬럼 지정.
- **예제**:
  ```java
  import jakarta.persistence.ManyToOne;
  import jakarta.persistence.JoinColumn;
  
  @ManyToOne
  @JoinColumn(name = "category_id")
  private Category category;
  ```
- **주의사항**:
  - `LAZY` 로딩을 권장하여 성능 최적화.

### `@OneToMany`
- **역할**: 일대다(1:N) 관계를 정의. 하나의 엔티티가 여러 엔티티를 참조.
- **중요도**: 높음. 다대일의 반대 방향 관계.
- **사용법**:
  - `mappedBy`: 연관 관계의 주인이 아닌 쪽에서 사용 (외래 키를 소유하지 않음).
  - `fetch`: `FetchType.LAZY` 권장.
  - `cascade`: 연관된 엔티티의 생명 주기 관리 (예: `CascadeType.ALL`).
- **예제**:
  ```java
  import jakarta.persistence.OneToMany;
  import java.util.List;
  
  @OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
  private List<Product> products;
  ```
- **주의사항**:
  - `mappedBy`를 반드시 지정하여 양방향 관계 설정.

### `@OneToOne`
- **역할**: 일대일(1:1) 관계를 정의.
- **중요도**: 중간. 특정 상황에서 사용.
- **사용법**:
  - `mappedBy`: 연관 관계의 주인이 아닌 쪽에서 사용.
  - `@JoinColumn`으로 외래 키 지정.
- **예제**:
  ```java
  import jakarta.persistence.OneToOne;
  import jakarta.persistence.JoinColumn;
  
  @OneToOne
  @JoinColumn(name = "detail_id")
  private ProductDetail detail;
  ```
- **주의사항**:
  - 성능상 주의해서 사용. `LAZY` 로딩 권장.

### `@ManyToMany`
- **역할**: 다대다(N:M) 관계를 정의. 조인 테이블을 통해 매핑.
- **중요도**: 중간. 복잡한 관계에서 사용.
- **사용법**:
  - `@JoinTable`로 조인 테이블과 컬럼 지정.
  - `mappedBy`로 연관 관계 주인 지정.
- **예제**:
  ```java
  import jakarta.persistence.ManyToMany;
  import jakarta.persistence.JoinTable;
  import jakarta.persistence.JoinColumn;
  import java.util.List;
  
  @ManyToMany
  @JoinTable(
      name = "product_tag",
      joinColumns = @JoinColumn(name = "product_id"),
      inverseJoinColumns = @JoinColumn(name = "tag_id")
  )
  private List<Tag> tags;
  ```
- **주의사항**:
  - 조인 테이블 관리로 인해 복잡도 증가. 가능하면 중간 엔티티로 분리 권장.

### `@JoinColumn`
- **역할**: 외래 키(Foreign Key) 컬럼을 지정.
- **중요도**: 높음. 관계 매핑에서 필수적.
- **사용법**:
  - `name`: 외래 키 컬럼 이름.
  - `referencedColumnName`: 참조하는 대상 테이블의 컬럼 이름 (기본값은 기본 키).
- **예제**:
  ```java
  @ManyToOne
  @JoinColumn(name = "category_id", nullable = false)
  private Category category;
  ```
- **주의사항**:
  - 생략 시 기본적으로 필드 이름 기반으로 컬럼 이름 생성.

---

## 3. 동시성 제어와 버전 관리 (★★★★☆)

동시성 문제를 해결하거나 데이터 일관성을 유지하는 데 사용됩니다.

### `@Version`
- **역할**: 낙관적 락(Optimistic Locking)을 위한 버전 관리 필드 지정.
- **중요도**: 높음. 동시성 제어가 필요한 환경에서 필수.
- **사용법**:
  - `int`, `Long`, 또는 `Timestamp` 타입 필드에 추가.
  - JPA가 자동으로 버전 값을 증가시키며 충돌 감지.
- **예제**:
  ```java
  import jakarta.persistence.Version;
  
  @Version
  private int version;
  ```
- **주의사항**:
  - 엔티티 수정 시 버전 불일치로 `OptimisticLockException` 발생 가능.
  - 인메모리 환경에서는 동작하지 않음.

---

## 4. 데이터 매핑 및 세부 설정 (★★★☆☆)

데이터의 저장 방식이나 제약 조건을 세밀하게 조정할 때 사용됩니다.

### `@Table`
- **역할**: 엔티티가 매핑될 데이터베이스 테이블을 지정.
- **중요도**: 중간. 기본 테이블 이름이 클래스 이름과 다를 때 사용.
- **사용법**:
  - `name`: 테이블 이름 지정.
  - `schema` 또는 `catalog`: 스키마나 카탈로그 지정.
- **예제**:
  ```java
  import jakarta.persistence.Table;
  
  @Entity
  @Table(name = "products", schema = "shop")
  public class Product {
      // ...
  }
  ```
- **주의사항**:
  - 생략 시 클래스 이름이 테이블 이름으로 사용됨.

### `@Transient`
- **역할**: 필드가 데이터베이스에 저장되지 않도록 지정.
- **중요도**: 중간. 엔티티의 일시적인 데이터를 처리할 때 유용.
- **사용법**:
  - 필드 또는 getter 메서드에 추가.
- **예제**:
  ```java
  import jakarta.persistence.Transient;
  
  @Transient
  private String tempData;
  ```
- **주의사항**:
  - 계산된 값이나 임시 데이터에 사용.

### `@Enumerated`
- **역할**: Java `Enum` 타입을 데이터베이스 컬럼에 매핑.
- **중요도**: 중간. 열거형 데이터를 사용할 때 필요.
- **사용법**:
  - `EnumType.ORDINAL`: 순서(숫자)로 저장 (기본값).
  - `EnumType.STRING`: 문자열로 저장 (권장).
- **예제**:
  ```java
  import jakarta.persistence.Enumerated;
  import jakarta.persistence.EnumType;
  
  @Enumerated(EnumType.STRING)
  private Status status;
  
  public enum Status {
      ACTIVE, INACTIVE
  }
  ```
- **주의사항**:
  - `STRING` 사용 권장. `ORDINAL`은 순서 변경 시 문제 발생 가능.

### `@Temporal`
- **역할**: `java.util.Date` 또는 `java.util.Calendar` 타입을 데이터베이스 날짜/시간 타입에 매핑.
- **중요도**: 중간. Java 8 이전 날짜 처리에 사용.
- **사용법**:
  - `TemporalType.DATE`: 날짜만 저장.
  - `TemporalType.TIME`: 시간만 저장.
  - `TemporalType.TIMESTAMP`: 날짜와 시간 모두 저장.
- **예제**:
  ```java
  import jakarta.persistence.Temporal;
  import jakarta.persistence.TemporalType;
  
  @Temporal(TemporalType.TIMESTAMP)
  private Date createdAt;
  ```
- **주의사항**:
  - Java 8 이상에서는 `LocalDate`, `LocalDateTime` 사용 권장 (`@Temporal` 불필요).

---

## 5. 고급 설정 및 드물게 사용되는 어노테이션 (★★☆☆☆)

특정 상황에서만 사용되거나 고급 기능을 제공합니다.

### `@Embedded` / `@Embeddable`
- **역할**: 복합 객체를 엔티티에 포함시켜 재사용 가능한 객체로 매핑.
- **중요도**: 낮음. 복잡한 객체 구조에서 사용.
- **사용법**:
  - `@Embeddable`: 포함 가능한 클래스에 추가.
  - `@Embedded`: 엔티티에서 포함 객체 필드에 추가.
- **예제**:
  ```java
  import jakarta.persistence.Embeddable;
  import jakarta.persistence.Embedded;
  
  @Embeddable
  public class Address {
      private String city;
      private String street;
      // Getter, Setter
  }
  
  @Entity
  public class User {
      @Embedded
      private Address address;
      // ...
  }
  ```
- **주의사항**:
  - 중복 코드를 줄이는 데 유용.

### `@ElementCollection`
- **역할**: 기본 타입이나 임베디드 객체의 컬렉션을 매핑.
- **중요도**: 낮음. 간단한 컬렉션 매핑에 사용.
- **사용법**:
  - `@CollectionTable`로 별도 테이블 지정.
- **예제**:
  ```java
  import jakarta.persistence.ElementCollection;
  import jakarta.persistence.CollectionTable;
  import jakarta.persistence.Column;
  import java.util.Set;
  
  @ElementCollection
  @CollectionTable(name = "product_images")
  @Column(name = "image_url")
  private Set<String> images;
  ```
- **주의사항**:
  - 관계형 엔티티 대신 간단한 데이터 컬렉션에 적합.

### `@MappedSuperclass`
- **역할**: 공통 필드와 매핑 정보를 상속받는 슈퍼클래스에 사용.
- **중요도**: 낮음. 엔티티 간 공통 속성을 관리할 때 유용.
- **사용법**:
  - 슈퍼클래스에 추가. 자체로는 엔티티가 아님.
- **예제**:
  ```java
  import jakarta.persistence.MappedSuperclass;
  import jakarta.persistence.Temporal;
  import jakarta.persistence.TemporalType;
  import java.util.Date;
  
  @MappedSuperclass
  public abstract class BaseEntity {
      @Temporal(TemporalType.TIMESTAMP)
      private Date createdAt;
      // Getter, Setter
  }
  
  @Entity
  public class Product extends BaseEntity {
      @Id
      private Long id;
      // ...
  }
  ```
- **주의사항**:
  - 독립적으로 쿼리 불가.

---

## 6. 인메모리 환경에서의 JPA 어노테이션
인메모리 데이터 저장소(예: `HashMap`, 인메모리 DB)를 사용할 경우, JPA 어노테이션은 동작하지 않습니다. 이는 JPA가 데이터베이스와의 매핑을 처리하는 ORM(Object-Relational Mapping) 프레임워크이기 때문입니다. 하지만 다음과 같은 이유로 어노테이션을 유지하거나 이해하는 것이 유용할 수 있습니다:
- **미래 확장성**: 나중에 JPA 기반 데이터베이스로 전환할 가능성을 대비.
- **코드 가독성**: 어노테이션이 의도를 명확히 전달 (예: `@Id`는 기본 키 역할 명시).
- **유지보수**: 팀원이나 미래의 개발자가 JPA를 사용할 것으로 예상 가능.

**권장 사항**:
- 인메모리 환경에서 JPA 어노테이션이 불필요하다면, `@Entity`, `@Id` 등을 제거하고 일반 POJO로 변환.
- JPA 어노테이션을 유지하려면, 코드 주석을 통해 "현재 인메모리 사용 중, JPA 어노테이션은 향후 사용 대비"라고 명시.

---

## 7. 실제 사용 시 고려사항
1. **어노테이션 최소화**: 불필요한 어노테이션은 코드 복잡도를 높임. 기본값으로 충분한 경우 생략.
2. **성능 고려**: `FetchType.EAGER`는 성능 저하를 유발할 수 있으므로 `LAZY` 권장.
3. **충돌 방지**: `@Version`을 사용해 동시성 문제 관리 (JPA 환경에서만 동작).
4. **테스트**: 어노테이션 설정 후 실제 데이터베이스에서 동작 확인.
5. **문서화**: 어노테이션의 목적과 설정 이유를 주석으로 명확히 기록.

---

## 8. 결론
JPA 어노테이션은 엔티티와 데이터베이스 간 매핑을 간편하게 처리하며, 동시성 제어와 관계 설정을 효율적으로 관리할 수 있게 합니다. 인메모리 환경에서는 동작하지 않지만, 코드의 의도를 명확히 하고 향후 JPA 도입 가능성을 대비하기 위해 유지할 수 있습니다. 이 자료를 통해 각 어노테이션의 역할과 사용법을 이해하고, 프로젝트에 적합한 방식으로 적용할 수 있기를 바랍니다.