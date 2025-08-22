# POJO(Plain Old Java Object) 학습 자료

## 1. POJO란?

POJO는 **Plain Old Java Object**의 약자로, Java에서 특정 프레임워크나 기술에 종속되지 않은 순수한 Java 객체를 의미합니다. 이는 Java의 기본적인 객체 지향 원칙을 따르는 간단하고 독립적인 클래스를 가리킵니다. POJO는 복잡한 상속 구조나 프레임워크 특정 어노테이션 없이 Java의 기본 기능만을 사용하여 정의됩니다.

### 정의
- **"Plain"**: 복잡한 프레임워크나 라이브러리 의존성 없이 단순한 Java 코드로 작성됨.
- **"Old"**: Java의 전통적인 객체 지향 프로그래밍 방식(예: JavaBeans 스타일)을 따름.
- **"Java Object"**: Java 언어로 작성된 객체로, 기본적인 클래스와 인스턴스 멤버로 구성.

POJO는 특정 프레임워크(예: Spring, Hibernate, EJB)에 의존하지 않으며, Java의 표준 규약만 준수하는 객체입니다. 이는 코드의 독립성과 재사용성을 높이는 데 중요한 역할을 합니다.

---

## 2. POJO의 특징

POJO는 다음과 같은 특징을 가집니다:

1. **프레임워크 독립성**:
   - Spring의 `@Component`, Hibernate의 `@Entity`, 또는 EJB의 특정 인터페이스 구현 없이 작성.
   - 특정 프레임워크의 어노테이션이나 상속 요구사항이 없음.

2. **단순성**:
   - 복잡한 상속 구조(예: 특정 프레임워크의 기반 클래스 상속)를 피함.
   - 기본적인 Java 클래스(필드, 생성자, 메서드)로 구성.

3. **JavaBeans 규약 준수** (선택적):
   - private 필드와 public getter/setter 메서드를 가짐.
   - 기본 생성자(no-argument constructor)를 제공.
   - 직렬화(Serializable) 구현 가능(필수 아님).

4. **테스트 용이성**:
   - 외부 의존성이 없으므로 단위 테스트가 쉬움.
   - 프레임워크 컨텍스트 없이 독립적으로 인스턴스화 가능.

5. **재사용성**:
   - 다양한 환경과 프레임워크에서 재사용 가능.
   - 특정 기술 스택에 묶이지 않음.

---

## 3. POJO의 구조 예제

다음은 전형적인 POJO의 예제입니다:

```java
public class User {
    private String id;
    private String name;
    private int age;

    // 기본 생성자
    public User() {}

    // 매개변수 생성자
    public User(String id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    // Getter와 Setter
    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    // 비즈니스 메서드 (예시)
    public boolean isAdult() {
        return age >= 18;
    }

    @Override
    public String toString() {
        return "User{id='" + id + "', name='" + name + "', age=" + age + "}";
    }
}
```

### 특징 설명
- **프레임워크 독립**: 위 클래스는 Spring, Hibernate 등의 어노테이션 없이 순수 Java로 작성됨.
- **JavaBeans 규약**: private 필드와 public getter/setter 제공, 기본 생성자 포함.
- **비즈니스 로직 포함**: `isAdult()`와 같은 도메인 로직을 포함할 수 있음.
- **직렬화 가능**: 필요 시 `implements Serializable` 추가 가능.

---

## 4. POJO와 클린 아키텍처

POJO는 클린 아키텍처(Clean Architecture)나 헥사고날 아키텍처(Hexagonal Architecture)에서 핵심적인 역할을 합니다. 클린 아키텍처에서는 도메인 계층(Entities 또는 Use Cases)이 외부 프레임워크로부터 독립적이어야 하며, 이를 위해 POJO가 이상적입니다.

### 클린 아키텍처에서의 역할
- **도메인 객체**: 도메인 계층의 엔티티(예: `User`, `Order`)는 POJO로 작성되어 프레임워크 의존성을 제거.
- **비즈니스 로직 캡슐화**: POJO는 비즈니스 규칙을 포함하며, 외부 시스템(DB, API 등)과 독립적으로 동작.
- **테스트 용이성**: 프레임워크 없이 테스트 가능, 모킹이 간단.
- **유연성**: 데이터베이스, ORM, 또는 API 변경 시 도메인 로직 수정 없이 어댑터 계층만 변경.

### 예제: 클린 아키텍처에서의 POJO
```java
// 도메인 계층의 POJO
public class Order {
    private String orderId;
    private List<Item> items;
    private double totalPrice;

    public Order(String orderId, List<Item> items) {
        this.orderId = orderId;
        this.items = items;
        this.totalPrice = calculateTotalPrice();
    }

    private double calculateTotalPrice() {
        return items.stream().mapToDouble(Item::getPrice).sum();
    }

    public String getOrderId() {
        return orderId;
    }

    public List<Item> getItems() {
        return items;
    }

    public double getTotalPrice() {
        return totalPrice;
    }
}
```

- **설명**: `Order` 클래스는 도메인 로직(예: `calculateTotalPrice`)을 포함하며, JPA의 `@Entity`나 Spring의 `@Component` 같은 어노테이션이 없음. 이는 다른 기술 스택에서도 재사용 가능.

---

## 5. POJO와 비-POJO의 차이

POJO와 비-POJO(프레임워크에 종속된 객체)의 차이를 비교해보겠습니다.

### 비-POJO 예제 (JPA 엔티티)
```java
import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class UserEntity {
    @Id
    private String id;
    private String name;

    // Getter, Setter
}
```

### 차이점
| 특징                 | POJO                              | 비-POJO (예: JPA 엔티티)             |
|----------------------|-----------------------------------|--------------------------------------|
| **프레임워크 의존성** | 없음                              | JPA 어노테이션(`@Entity`, `@Id`)에 의존 |
| **재사용성**         | 높음 (모든 Java 환경에서 사용 가능) | 특정 프레임워크(JPA) 환경에서만 사용 가능 |
| **테스트**           | 프레임워크 없이 테스트 가능         | ORM 컨텍스트 필요                    |
| **복잡성**           | 단순 (순수 Java)                  | 프록시, 지연 로딩 등 복잡성 추가      |

### 문제점 (비-POJO)
- **종속성**: `@Entity`는 JPA 프레임워크에 묶여 있어 다른 ORM으로 전환 시 수정 필요.
- **테스트 어려움**: JPA 컨텍스트를 띄우거나 H2 같은 인메모리 DB가 필요.
- **유연성 저하**: 데이터베이스 스키마 변경이 객체 구조에 직접 영향을 미침.

---

## 6. POJO의 장점과 단점

### 장점
1. **독립성**:
   - 특정 프레임워크에 종속되지 않아 다양한 환경에서 재사용 가능.
   - 기술 스택 변경 시 코드 수정 최소화.
2. **단순성**:
   - 복잡한 상속이나 프레임워크 규약 없이 간단한 구조.
   - 코드 가독성과 유지보수성이 높음.
3. **테스트 용이성**:
   - 외부 의존성이 없어 단위 테스트가 쉬움.
   - 모킹 라이브러리(예: Mockito)로 의존성 주입 간단히 처리.
4. **클린 아키텍처 적합**:
   - 도메인 중심 설계에 이상적, 비즈니스 로직 캡슐화.
5. **직렬화 가능**:
   - `Serializable` 구현 시 네트워크 전송, 파일 저장 등에 활용 가능.

### 단점
1. **보일러플레이트 코드**:
   - Getter, Setter, 생성자 등 반복적인 코드가 많아질 수 있음.
   - 해결: Lombok 같은 라이브러리 사용(단, 과도한 사용은 주의).
2. **추가 매핑 필요**:
   - 프레임워크와 연동 시 DTO 또는 매퍼 클래스를 통해 변환 필요.
   - 예: POJO와 JPA 엔티티 간 매핑 로직 추가.
3. **기능 제한**:
   - 프레임워크의 편리한 기능(예: `@Transactional`, 자동 영속성)을 직접 사용하지 않으므로 추가 설계 필요.

---

## 7. POJO 사용 사례

POJO는 다양한 상황에서 유용합니다:

1. **도메인 모델**:
   - 클린 아키텍처의 엔티티 또는 도메인 객체로 사용.
   - 예: `User`, `Order`, `Product` 등 비즈니스 객체.

2. **데이터 전송 객체(DTO)**:
   - API 요청/응답 데이터를 전달하는 객체.
   - 예: REST API에서 클라이언트와 서버 간 데이터 교환.

3. **테스트**:
   - 단위 테스트에서 프레임워크 없이 객체 생성 및 테스트.
   - 예: `Mockito.mock()`으로 의존성 주입 테스트.

4. **직렬화**:
   - JSON, XML, 또는 네트워크 전송을 위한 객체.
   - 예: `ObjectMapper`로 JSON 변환.

5. **크로스 플랫폼**:
   - Spring, Quarkus, 또는 프레임워크 없는 환경에서 동일한 객체 사용.

### 예제: DTO로 사용
```java
public class UserDto {
    private String id;
    private String name;

    public UserDto(String id, String name) {
        this.id = id;
        this.name = name;
    }

    // Getter, Setter
}
```

- **설명**: REST API 응답으로 사용되는 `UserDto`는 프레임워크 의존성 없이 클라이언트에 데이터를 전달.

---

## 8. POJO와 프레임워크 통합

POJO를 프레임워크와 통합할 때는 어댑터 계층에서 프레임워크 의존성을 처리합니다. 예를 들어, Spring과 JPA를 사용하는 클린 아키텍처 프로젝트에서:

1. **도메인 계층**:
   - POJO로 작성된 `User` 클래스(비즈니스 로직 포함).
2. **어댑터 계층**:
   - JPA 엔티티(`UserEntity`)와 매핑 로직.
   - Spring 컨트롤러에서 `User`를 `UserDto`로 변환.
3. **포트**:
   - `UserRepository` 인터페이스를 통해 도메인과 저장소 연결.

### 예제: Spring과 POJO 통합
```java
// 도메인 계층 (POJO)
public class User {
    private String id;
    private String name;

    public User(String id, String name) {
        this.id = id;
        this.name = name;
    }

    // Getter, 비즈니스 로직
}

// 포트
public interface UserRepositoryPort {
    User findById(String id);
    void save(User user);
}

// 어댑터 (JPA 구현)
@Repository
public class JpaUserRepositoryAdapter implements UserRepositoryPort {
    private final JpaUserRepository jpaRepository;

    public JpaUserRepositoryAdapter(JpaUserRepository jpaRepository) {
        this.jpaRepository = jpaRepository;
    }

    @Override
    public User findById(String id) {
        UserEntity entity = jpaRepository.findById(id).orElseThrow();
        return new User(entity.getId(), entity.getName());
    }

    @Override
    public void save(User user) {
        UserEntity entity = new UserEntity(user.getId(), user.getName());
        jpaRepository.save(entity);
    }
}

// JPA 엔티티
@Entity
public class UserEntity {
    @Id
    private String id;
    private String name;

    public UserEntity(String id, String name) {
        this.id = id;
        this.name = name;
    }

    // Getter, Setter
}
```

- **설명**: `User`는 POJO로 유지되며, `UserEntity`와 매핑은 어댑터에서 처리. 도메인 계층은 JPA나 Spring에 독립적.

---

## 9. POJO 작성 시 권장 사항

1. **단순성 유지**:
   - 불필요한 의존성 추가 금지.
   - 최소한의 필드와 메서드로 구성.
2. **JavaBeans 규약 준수**:
   - Getter/Setter, 기본 생성자 제공.
   - 명확한 명명 규칙 사용.
3. **불변성 고려**:
   - 가능하면 필드를 `final`로 설정해 불변 객체로 설계.
   - 예: `final String id`.
4. **Lombok 사용 주의**:
   - `@Data`, `@Getter`, `@Setter`로 보일러플레이트 코드 줄일 수 있음.
   - 하지만 과도한 사용은 코드 가독성을 해칠 수 있음.
5. **도메인 로직 포함**:
   - 비즈니스 규칙을 POJO에 캡슐화(예: `isAdult()`).
6. **문서화**:
   - Javadoc으로 필드와 메서드의 역할 명확히 설명.

---

## 10. 결론

POJO는 Java 개발에서 단순성과 독립성을 보장하는 핵심 요소입니다. 클린 아키텍처와 같은 설계 원칙에서는 도메인 계층의 핵심 구성 요소로 사용되며, 프레임워크 의존성을 제거해 유연성과 테스트 용이성을 높입니다. POJO를 효과적으로 사용하면 코드 유지보수성이 향상되고, 다양한 기술 스택에서 재사용 가능한 견고한 애플리케이션을 구축할 수 있습니다.

### 요약
- **POJO 정의**: 프레임워크 독립적인 순수 Java 객체.
- **장점**: 단순성, 테스트 용이성, 재사용성.
- **단점**: 보일러플레이트 코드, 추가 매핑 필요.
- **사용 사례**: 도메인 모델, DTO, 테스트, 직렬화.
- **클린 아키텍처**: 도메인 계층의 핵심, 프레임워크 의존성 제거.

POJO를 통해 도메인 중심 설계를 실천하고, 어댑터 계층에서 프레임워크 통합을 처리하면 견고하고 유지보수 가능한 소프트웨어를 개발할 수 있습니다.